# QA loop (Phase 3 — after deploy)

After `docker compose up` is healthy and migrations applied, exercise the running app. This is **functional** QA — not unit tests; those are the builder's responsibility.

## Smoke checks (always run)

1. **Root URL responds 2xx:**
   ```bash
   curl -sf -o /dev/null -w "%{http_code}" http://localhost:${APP_PORT}/
   ```
   Expect: 200 (or 301/302 if root redirects). Anything 4xx/5xx = fail.

2. **Health endpoint** (if app exposes one — most templates here do):
   ```bash
   curl -sf http://localhost:${APP_PORT}/health
   ```
   Expect: 200 with JSON body containing `ok: true` or similar.

3. **DB reachable** (if DB in stack):
   ```bash
   docker compose exec -T db pg_isready -U ${POSTGRES_USER}
   ```
   Expect: `accepting connections`.

4. **App ↔ DB connectivity** (if DB in stack): exercise an endpoint that hits the DB (auth signup, conversation list, etc.). Use `curl` with the appropriate route.

If any smoke check fails: dispatch fixer subagent. Bounded 5 retries before forcing Gate B with "QA failing".

## Archetype-specific functional checks

Based on the masterplan's archetype (Phase 0d Q1), run additional checks. These exercise the Sonzai SDK integration end-to-end.

### companion

1. Create a user via the auth flow (signup endpoint with test email)
2. Send a chat message via `/api/chat` (or equivalent) with a fixture: `{"message": "hello"}`
3. Verify response shape: `{"reply": <non-empty string>, "session_id": <uuid>, ...}`
4. Verify the SDK actually called Sonzai by checking the response is contextual (not a canned message) — if the test response is `"hello"` echoed back, fail
5. Send a follow-up message in the same session: `{"message": "what did I just say?"}`
6. Verify the response references the previous message — confirms memory is wired

### guide-router

1. Send an initial message: `{"message": "I need help with X"}`
2. Verify response includes a routing decision (which specialist to engage)
3. Verify subsequent messages go to that specialist

### enterprise-assistant

1. Upload a fixture document to the KB endpoint (if KB is in scope)
2. Query: `{"message": "what does the document say about Y?"}`
3. Verify response cites the document

### customer-support

1. Send a fixture ticket: `{"message": "my order is broken"}`
2. Verify response either answers from KB or escalates with `{"escalate": true, "reason": ...}`

### game-npc

1. Send a dialogue event: `{"event": "player_approaches", "context": {...}}`
2. Verify NPC response includes appropriate state changes

### coach-therapist

1. Send a session-open message
2. Send 3 follow-up messages
3. Verify mood / diary state is being tracked (check the appropriate endpoint)

### hybrid-custom

1. Use the smoke checks only — archetype is by definition unknown
2. Flag in QA report: "hybrid-custom — functional checks must be specified by operator at Gate B"

## QA report

After all checks (smoke + archetype-specific), assemble a QA report in-memory:

```yaml
qa_report:
  smoke:
    root_url:        pass
    health:          pass
    db:              pass
    db_connectivity: pass
  archetype_companion:
    auth_signup:     pass
    chat_basic:      pass
    chat_memory:     fail (response did not reference previous message)
  overall:           fail
  failing_checks:    [archetype_companion.chat_memory]
  logs_tail: |
    <last 50 lines of docker compose logs --tail 50>
```

## On failure

If `overall: fail`:

1. Dispatch fixer subagent (see `subagent-prompts/fixer.md`) with:
   - The QA report
   - The masterplan path
   - The failing check description
   - The relevant logs
2. Fixer commits its changes
3. `docker compose up -d --build` (rebuild + restart)
4. Re-run QA loop from the top (smoke + archetype)
5. Bounded 5 retries. After that:

### Notify build done with failing QA (best-effort)

If `notify_enabled` is true: dispatch `subagent-prompts/notifier.md`:

```yaml
event:        full_auto_failed_qa
run_id:       <run-id>
recipient:    <from state>
subject:      "[full-auto] Build done with failing QA — run <run-id>"
body: |
  ⚠️  full-auto build done with failing QA (5 fixer cycles exhausted).

  Run:        <run-id>
  Live URL:   <APP_URL>
  Report:     docs/cto-review/<date>-final-report.md
  QA status:  FAILING on: <comma-list of failing checks>
  Time taken: <ELAPSED>

  App is up; some flows broken. See report for what's red.
interactive:  false
```

Record IDs in state. Continue to final-report (which records QA as failed).

   Surface to operator: "QA still failing after 5 fix attempts. Forcing Gate B for your decision." Then proceed to Gate B WITH `qa: failing` in the report.

## On success

`overall: pass` → save QA report to in-memory state, then:

### Notify build complete (best-effort)

If `notify_enabled` is true (from Phase 0-pre): dispatch `subagent-prompts/notifier.md`:

```yaml
event:        full_auto_complete
run_id:       <run-id from state>
recipient:    <from state>
subject:      "[full-auto] Build complete (healthy) — run <run-id>"
body: |
  ✅ full-auto build complete (healthy).

  Run:        <run-id>
  Live URL:   <APP_URL>
  Report:     docs/cto-review/<date>-final-report.md
  QA status:  passing
  Time taken: <ELAPSED>

  App is still running (docker compose). Push when ready.
interactive:  false
```

Record returned `message_id` / `message_ts` in run state for the final report's notifications table. Best-effort; if notifier returns errors for both channels, log one line and continue to final-report.

- **`full-auto` mode:** write the final report (`final-report.md.template`) and exit. App keeps running locally; operator owns next steps (push, deploy to prod, tear down).
- **`cto-loop` mode:** read `../cto-loop/cto-review-gate.md` next (Gate B).

## Hard rules

1. **QA exercises the running app**, not static analysis. `curl` and HTTP requests, not lint.
2. **Use fixtures, never real user data.** Test emails: `cto-loop-qa-<uuid>@example.com`. Test messages: ephemeral.
3. **Clean up fixtures.** After QA, delete the test users / messages. If the DB schema doesn't support cascade-delete, leave a note in the QA report.
4. **Bounded retries.** 5 fixer cycles, then to Gate B with `qa: failing`.
5. **Don't gate on QA pass.** Even with failing QA, you go to Gate B — but you tell the operator "QA failing" so they can decide.
6. **No tenant-specific fixtures.** Generic data only.
