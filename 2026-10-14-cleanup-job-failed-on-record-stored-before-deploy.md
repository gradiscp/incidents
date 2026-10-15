# 2026-10-14 - Cleanup job failed every 20 minutes on a record stored before a deploy

**Status:** Mitigated - fix written and tested, not yet deployed
**Area:** Pydantic models with `extra="forbid"`, stored JSON records (Azure Cosmos DB), scheduled
Container Apps job, missing alert

## Summary

In a company project's staging environment, the scheduled cleanup job that files and closes expired
sessions failed on every run for more than ten hours, always on the same session record. The record had
been written before a deploy that removed three fields from a strict Pydantic model; the new version
could no longer read it. Nothing alerted on the failure. Fix: stored records are read tolerantly at the
one place they are loaded, while request bodies and signature data stay strict.

## Impact

- Staging only. One test session was neither filed in the document system nor closed: 32 failed job
  runs from 10:40 to 21:00 local time, then still failing until the fix is deployed.
- Session records carry a 24-hour database TTL as a backstop. If the fix is not deployed in time, the
  record is deleted without being filed. With real data that is the failure the cleanup job exists to
  prevent: a signed document deleted without being filed.
- In production, every session open during such a deploy would fail the same way.

## Environment

| Component | Detail |
|---|---|
| API | Python, FastAPI, Pydantic v2; request models use `extra="forbid"` |
| Storage | Azure Cosmos DB; the session record is the same Pydantic model, stored as JSON |
| Lifetime | a session lives 1 hour; the database TTL deletes the record 24 hours after creation |
| Cleanup | Container Apps job every 20 minutes: loads expired sessions, files signed documents, closes them |
| Logs | Log Analytics, table `ContainerAppConsoleLogs`, 30 days |

## Timeline

Local time (UTC+2). Times from the job history and the logs unless marked as estimated.

- 2026-10-14 about 09:30 (estimated) - a test session is created by the previous version.
- 10:12-10:16 - deploy of a version that removes three answer fields from the session model.
- 10:20 - cleanup run: nothing expired yet, `failed: 0`.
- about 10:30 (estimated) - the session expires (1 hour).
- 10:40 - first failed run: `session.sweep_failed`, `error: ValidationError`, job execution "Failed".
- Every 20 minutes after that - the same failure on the same record; no alert.
- about 20:10 (estimated) - noticed as failed runs in the job's execution history during other work on
  staging.
- about 21:00 - fix written and tested locally (unit tests and the database emulator); pending deploy.

## Root cause

One Pydantic model served two purposes: validating request bodies, where unknown keys must be refused,
and reading records the app itself had stored. With `extra="forbid"`, removing a field from the model
makes every record still holding that key invalid. The records live up to 24 hours, so every deploy
reads records the previous version wrote. The cleanup job loads each expired record through that model,
fails, logs the exception type and exits non-zero, run after run.

Contributing: the alert on the job's failure event was planned but not yet built, and a failed job run
shows only in the job's execution history.

## Evidence

Job history (read-only `az containerapp job execution list`): 100 executions "Succeeded" up to
10:20, then 32 "Failed" from 10:40 to 21:00, one every 20 minutes.

Log query over the staging workspace:

```kusto
ContainerAppConsoleLogs
| where Log has "sweep"
| extend action = extract("\"action\": \"([^\"]+)\"", 1, Log)
| summarize n=count(), first_seen=min(TimeGenerated), last_seen=max(TimeGenerated) by action
```

Result: `session.sweep_failed` 31 lines, 10:40 to 20:40 local (the 21:00 run's line had not arrived
yet). Shape of each (id replaced):
`{"action": "session.sweep_failed", "actorId": "expiry-sweep", "details": {"caseId": "<id>",
"error": "ValidationError"}}`, followed by `session.sweep` with `"failed": 1`.

Reproduction: a record in the old shape, validated against the new model as a request body, fails
with `extra_forbidden` on exactly the three removed keys (now a test, see Prevention).

## What didn't work

Nothing was tried on the live system; a re-run would fail the same way. Approaches considered and
rejected:

- **`extra="ignore"` for stored records only:** covers a removed field, but not a rule that became
  stricter (a stored value the new version refuses), which fails the same way.
- **Reading everything leniently:** would also accept a damaged signature record and silently drop
  it; signature data must stay strict.
- **Separate storage models** mirroring the API models: duplicates every field.
- **Schema versions with migrate-on-read:** more machinery than records that live at most 24 hours
  need.
- **Waiting for the TTL:** removes the stuck record, but without filing it.

## Resolution

- The one function that turns a stored item into a session validates with `extra="ignore"` per call
  and with a validation context that leaves an answer the current rules refuse unanswered (its
  default), instead of failing. Signature records and request bodies stay strict.
- A compatibility fixture holds a signed session exactly as the previous version wrote it (the removed
  fields, a value a newer rule refuses, the database's own keys). Tests: it loads, the cleanup job files
  and closes it, a damaged signature record still fails, and the same answers as a request body are
  still refused. One more test reads it through the database emulator.
- The failure event was added to the planned alert, which is a release blocker.

## Unverified

- Which key made the real record fail: the log names only the exception type, and the record was not
  read. The removed fields are the cause inferred from the timing and the deploy's diff.
- That the first cleanup run after the deploy files and closes the stuck record. Check: the job's
  execution history shows "Succeeded" again, and `session.sweep` reports `closed` 1 without a
  `session.sweep_failed` line.
- Whether the advisor's own requests on that session also failed after the deploy (not checked).

## Prevention

- **Test:** the compatibility fixture. Its docstring says: when a change breaks it, keep the stored
  shape compatible (a default for a new field, accept the old form of a removed or stricter one until
  no record can hold it, 24 hours) instead of editing the fixture.
- **Rule** in the API's contributor guide, next to the storage code: stored records are read
  tolerantly; request bodies stay strict.
- **Alert** on the cleanup job's failure event: planned, not live yet.
- **Habit:** when a field is removed, renamed or its rule tightened on a model that is also stored,
  ask what happens to the records already written.
