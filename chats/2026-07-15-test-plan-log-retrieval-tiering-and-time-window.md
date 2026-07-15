# Test Plan — Error-First Log Retrieval Tiering + Free-Text Relative Time-Window Parsing

**Commit under test**: `a038604` — "feat(slice-15): error-first log retrieval tiering + free-text relative time-window parsing"
**Branch**: `015-adk-model-provider-wiring`
**Files changed**:
- [LocalCloudLoggingSdkAdapter.java](../src/main/java/com/equifax/tsa3/agentassist/tools/logging/LocalCloudLoggingSdkAdapter.java)
- [LogAnalysisBranchAdapter.java](../src/main/java/com/equifax/tsa3/agentassist/orchestration/delegation/LogAnalysisBranchAdapter.java)

## What changed (context)

1. **Correlation-ID field-scoped filter** — `buildFilter()` now matches `jsonPayload."Correlation-Id"="<id>"` directly for `IdentifierType.CORRELATION_ID`, instead of a broad free-text search across all string fields. Other identifier types keep the old free-text match.
2. **Error-first retrieval tiering** — `retrieve()` tries a `severity="ERROR"`-scoped filter first. It falls back to the full (unscoped-by-severity) filter only when the error-tier pass returns a genuine `EMPTY_RESULT`, not on transport/auth failures.
3. **Free-text relative time-window parsing** — `LogAnalysisBranchAdapter.extractRelativeWindow(String)` parses phrases like `"last 3 days"`, `"past 2 hours"`, `"last 30 minutes"` out of the turn's `inputText` and uses that as the log retrieval window **only when** the caller didn't supply an explicit `logWindowStart`/`logWindowEnd` on the invocation context.

## Pre-requisites

- Java 21, Maven (`/Users/sxc739/Applications/apache-maven-3.9.11/bin/mvn`), `jq`, `curl`, `gcloud` CLI authenticated.
- ADC configured for Vertex AI Gemini calls: `gcloud auth application-default login`.
- `TSA3_ADC_PROJECT_ID` set to a project with Vertex AI + Cloud Logging access (e.g. `ews-vs-ceir-qa-npe-1941`).
- Local profile config already has `tsa3.orchestration.adk-hosted.*.enabled=true` and `model-wired.*.enabled=true` (see [application-local.yml](../src/main/resources/application-local.yml)).
- A known-good correlation ID with log data in the **correct time window** (the previous session found a mismatch: default retrieval window is "last 15 hours," but the reference log data is dated `2026-07-07`/`2026-07-08`). Use `correlationId=09afb86b-6fb8-46d4-9dcf-11166d485d13` with an explicit "last N days" phrase wide enough to cover it, or pull a fresh ID from today's window.

---

## Part 1 — Unit / compile-level checks

These don't require the live app; they confirm nothing is broken at the code level.

### 1.1 Compile

```bash
cd /Users/sxc739/workspace/tsa/15462_US_VS_TSA3
/Users/sxc739/Applications/apache-maven-3.9.11/bin/mvn -s .mvn/settings-dev.xml -q compile
```
**Expect**: exits 0, no output.

### 1.2 Run existing regression suites that exercise these two classes

There is no dedicated `LogAnalysisBranchAdapterTest`; these classes are exercised indirectly by the Slice 9 log-retrieval test suites and log-analysis integration tests.

```bash
cd /Users/sxc739/workspace/tsa/15462_US_VS_TSA3
/Users/sxc739/Applications/apache-maven-3.9.11/bin/mvn -s .mvn/settings-dev.xml -q \
  -Dtest=LoggingProviderPortContractTest,Slice9PlanningExecutionSmokeTest,DeterministicEvidenceHandoffIntegrationTest,DeterministicSchemaSuccessRateGateTest,NormalizedOutcomeIntegrationTest,DuplicateRetrySuppressionTest,Slice9QuickstartScenariosTest,LocalCloudLoggingSdkAdapterIntegrationTest \
  test
```
**Expect**: `BUILD SUCCESS`, no failures.

> If you want a quick manual check of just the new parser logic without a full Spring context, you can add a throwaway `main()` or a JShell snippet exercising `LogAnalysisBranchAdapter.extractRelativeWindow(...)` — it's `static` and package-visible for exactly this purpose. Example inputs to try:
> - `"last 3 days"` → non-null `[start, end]`, `end - start ≈ 3 days`
> - `"past 2 hours"` → `≈ 2 hours`
> - `"last 30 minutes"` → `≈ 30 minutes`
> - `"summarize the outage"` (no phrase) → `null`
> - `"last 3 dys"` (typo/no unit match) → `null`

---

## Part 2 — Live end-to-end smoke test (primary validation)

### 2.1 Start the app

```bash
cd /Users/sxc739/workspace/tsa/15462_US_VS_TSA3
export TSA3_ADC_PROJECT_ID=ews-vs-ceir-qa-npe-1941
export GOOGLE_APPLICATION_CREDENTIALS=/path/to/service-account.json   # or rely on gcloud ADC
/Users/sxc739/Applications/apache-maven-3.9.11/bin/mvn -s .mvn/settings-dev.xml spring-boot:run -Dspring-boot.run.profiles=local,dev
```
Wait for `Started ...Application` in the log. Keep this terminal open (or background it and `tail -f` a log file).

**Important**: if the app was already running before you pulled/committed these changes, **kill and restart it** — a stale process will not have the new filter/parsing logic. Confirm no other instance is running first:
```bash
ps aux | grep spring-boot:run | grep -v grep
```

### 2.2 Run the smoke test script

In a second terminal:
```bash
cd /Users/sxc739/workspace/tsa/15462_US_VS_TSA3
export TSA3_ADC_PROJECT_ID=ews-vs-ceir-qa-npe-1941
export BASE_URL=http://localhost:8080
./scripts/smoke-test-gemini-turn.sh
```

This script submits the turn:
> `"check erros for correlation id in the last 3 days correlation-id: 09afb86b-6fb8-46d4-9dcf-11166d485d13"`

**What to check in the output**:
| Check | Expected |
|---|---|
| Session creation | `sessionId` returned, non-null |
| Turn submission | `turnId` returned, non-null |
| Polling loop | `lifecycleState` eventually leaves `PENDING`/`IN_PROGRESS` within ~20s |
| Final response | `outputText` (or equivalent) is populated with a Gemini-generated summary, not a generic degrade message |

### 2.3 Confirm the relative-window parse actually took effect (debug logs)

With `logging.level.com.equifax.tsa3.agentassist.runtime.adk=DEBUG` already set in [application-local.yml](../src/main/resources/application-local.yml), also watch for the `LogAnalysisBranchAdapter` debug line. If it's not visible, temporarily bump its logger too:
```bash
# optional, if you don't see LogAnalysisBranchAdapter debug lines
export TSA3_ADDITIONAL_LOGGING="logging.level.com.equifax.tsa3.agentassist.orchestration.delegation=DEBUG"
```

In the app's console/log output, look for:
```
LogAnalysisBranchAdapter.fetchBundle: parsed relative window from inputText start=... end=...
```
- **Expect**: this line appears, and `end - start ≈ 3 days` (matching "last 3 days" in the smoke-test's `inputText`).
- **If it's absent**: either the phrase wasn't matched (regex bug) or `context.logWindowStart()/logWindowEnd()` were already non-null from elsewhere (explicit values always win — check for that too).

### 2.4 Confirm the tightened correlation-ID filter and error-first tiering (debug logs)

Look for the retrieval debug line in `LocalCloudLoggingSdkAdapter`:
```
LocalCloudLoggingSdkAdapter.retrieve: project=... identifier=... window=[...] errorFilter=... fullFilter=...
```
- **Check `errorFilter`**: should contain both `jsonPayload."Correlation-Id"="09afb86b-6fb8-46d4-9dcf-11166d485d13"` (field-scoped, not free-text) **and** `severity="ERROR"`.
- **Check `fullFilter`**: should contain the field-scoped correlation-ID match but **without** the `severity="ERROR"` clause.
- **Tiering behavior**: if the correlation ID actually has ERROR-level log entries in the resolved window, the error-tier pass should succeed and the full-filter retry should **not** fire. If there are no ERROR entries but other levels exist, you should see:
  ```
  LocalCloudLoggingSdkAdapter: error-tier filter empty, retrying with full filter project=... identifier=...
  ```
  followed by a successful full-filter result.

### 2.5 Cross-check against the raw `gcloud` query

Run the same query directly against Cloud Logging to confirm the app's tightened filter matches reality:
```bash
gcloud logging read 'jsonPayload."Correlation-Id"="09afb86b-6fb8-46d4-9dcf-11166d485d13"' \
  --project=ews-sre-gke-us-qa-npe-417c --limit=200 --order=asc --format=json
```
- **Expect**: the record count and content roughly match what the app's evidence bundle produced (allowing for the app's own result cap/truncation).
- **If the app returns nothing but this raw query returns records**: check the resolved time window in the debug logs (Part 2.3) — the most likely cause is a window mismatch, not the filter itself (this was the exact regression diagnosed in the prior session: default 15-hour window vs. `2026-07-08`-dated data).

---

## Part 3 — Regression checks (make sure nothing else broke)

### 3.1 Non-correlation-ID identifier types still use free-text match

Submit a turn using a different identifier type your app supports (e.g. a numeric `transactionId` or `accountId`, per `NUMERIC_ID_PATTERN` in `LogAnalysisBranchAdapter`):
```bash
curl -s -X POST "${BASE_URL}/api/v1/runtime/sessions/${SESSION_ID}/turns" \
  -H 'Content-Type: application/json' \
  -d '{
    "apiVersion":"1.0",
    "requestId":"'"$(uuidgen)"'",
    "sessionId":"'"${SESSION_ID}"'",
    "inputText":"Summarize log for transactionId=123456789",
    "actor": {"actorId":"local-tester","actorKind":"USER","authorizationScope":"SAME_USER_HOST"},
    "hostContext": {"hostId":"local-smoke-test","environment":"local"}
  }'
```
**Expect**: `buildFilter()` debug line shows the old-style free-text `AND ("123456789")` clause, not a `jsonPayload."Correlation-Id"=...` clause. Confirms the field-scoped tightening didn't leak into other identifier types.

### 3.2 Explicit window still overrides free-text parsing

If you have a way to pass `logWindowStart`/`logWindowEnd` directly (currently only reachable via test harnesses / `BranchInvocationContext`, not the public REST DTO — see the prior session's investigation), verify explicit values are **not** overwritten by a "last N days" phrase in the same `inputText`. This is best done as a unit-level check on `LogAnalysisBranchAdapter.fetchBundle()` rather than through the REST API, since the REST DTO doesn't currently expose these fields.

### 3.3 No relative-window phrase present

Submit a turn with **no** time phrase (e.g. just `"correlationId=09afb86b-6fb8-46d4-9dcf-11166d485d13"`).
**Expect**: no `"parsed relative window from inputText"` debug line; retrieval falls back to the existing `BoundedRetrievalAssumptions` default (900 minutes / 15 hours), unchanged from pre-patch behavior.

### 3.4 Full regression suite (optional, broader confidence)

```bash
cd /Users/sxc739/workspace/tsa/15462_US_VS_TSA3
/Users/sxc739/Applications/apache-maven-3.9.11/bin/mvn -s .mvn/settings-dev.xml -q test
```
**Expect**: `BUILD SUCCESS`. (Note: per commit history, there was 1 pre-existing unrelated failure noted in earlier Slice 15 commits — if you see exactly that one and nothing new, it's not a regression from this patch.)

---

## Shutdown

```bash
ps aux | grep spring-boot:run | grep -v grep
kill <PID> 2>/dev/null; sleep 3; ps aux | grep spring-boot:run | grep -v grep; echo done
```

---

## Sign-off checklist

- [ ] 1.1 Compiles cleanly
- [ ] 1.2 Existing log-retrieval regression suites pass
- [ ] 2.2 Smoke test produces a real Gemini-generated `outputText`
- [ ] 2.3 Relative-window parse debug line confirms ~3-day window from "last 3 days" phrase
- [ ] 2.4 Error-first tiering behaves correctly (tries ERROR filter, falls back only on empty result)
- [ ] 2.5 App's retrieved records match the raw `gcloud logging read` cross-check
- [ ] 3.1 Non-correlation-ID identifiers still use free-text filter
- [ ] 3.3 No-phrase inputs still fall back to default window
- [ ] 3.4 (optional) Full suite green aside from known pre-existing failures