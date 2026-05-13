# QA loop

Phase 5: outer session exercises the app the builder built, feeds failures back to the builder, repeats up to 5 cycles. Continuously, no operator prompts.

## Inputs (read these first)

- `.full-auto/build-summary.json` (from Phase 4)
- `.full-auto/wizard-answers.md` (acceptance checklist section)
- The current `cycle_n` counter (starts at 1)

## Step 1 — Decide what to start

From `build-summary.json`:

```jsonc
"entrypoints": {
  "backend": "go run ./cmd/server",  // or null if nothing to start
  "frontend": "npm run dev",          // or null
  "ports": {"backend": 8080, "frontend": 3000},
  "env_required": ["SONZAI_API_KEY"]
}
```

**Env preflight:**
- Check every var in `env_required` is set
- If any missing AND a real value isn't available to this session → halt the entire run with `.full-auto/BLOCKED.md` ("Cannot run QA — SONZAI_API_KEY missing and no test key available"). This is a hard limit; do NOT mock the API.
- If a `.env` file exists in `{{TARGET_REPO_PATH}}`, source it before starting servers

If `backend` is null AND `frontend` is null → builder reported nothing to start; halt with BLOCKED ("Builder reported no entrypoints, cannot exercise")

## Step 2 — Start servers

Use `Bash run_in_background=true`. Run commands FROM inside the target repo's directory.

```
# in target repo dir
backend_log=.full-auto/qa-cycle-{{cycle_n}}/backend.stdout.log
mkdir -p $(dirname $backend_log)
<backend_command> > $backend_log 2>&1 &
```

Same for frontend if present.

Capture the bash shell-ids the runtime returns. Save them — we'll need them to stop the servers between cycles.

## Step 3 — Wait for readiness

Use `Monitor` tool on the background bash shells to stream stdout until you see a readiness signal:

| Stack | Readiness regex |
|---|---|
| Go (net/http, Echo, Gin, Chi) | `listening` / `bind` / `:{{port}}` |
| Node (Express, Elysia, Hono, Fastify) | `listening` / `ready` / `Local:.*http://localhost` |
| Python (FastAPI, Flask, Django) | `Uvicorn running` / `Running on http` / `started server` |
| Next.js / Vite | `Local:.*http://localhost` / `ready in` |

If port is taken (`EADDRINUSE` / `address already in use`):
- Try once more with a free port range (10000-19999), updating the URL we test against
- If still failing → record as Cycle Failure: "server failed to start: port conflict"

Cap waits at 60 seconds. If no readiness signal in 60s → record as Cycle Failure: "server did not start within 60s; tail of stdout:" + last 20 lines.

## Step 4 — Exercise the backend

For each item in `wizard-answers.md`'s **Acceptance Checklist** that names an HTTP endpoint or SDK call:

### HTTP endpoint items
```bash
# example for a chat endpoint
curl -sS -w '\n%{http_code} %{time_total}s\n' \
  -X POST "http://localhost:{{port}}/chat" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${SONZAI_API_KEY}" \
  -d '{"agent_id":"qa-agent","user_id":"qa-user-1","message":"hello"}' \
  > .full-auto/qa-cycle-{{cycle_n}}/endpoint-chat.log 2>&1
```

Pass criteria per checklist item:
- HTTP status as specified (typically 2xx)
- Response body shape matches what the spec/template says
- Latency under the spec'd budget (Q6) — measure `%{time_total}` and compare

### Memory-persistence items
- POST a chat with user_id=`qa-user-1`, content "remember my favorite color is blue"
- POST a second chat with same user_id, content "what's my favorite color?"
- Assert: response contains `blue` (loose grep — memory may surface via different phrasings)

### Scheduled-reminder items
- Don't wait for cron to fire. Instead: assert the schedule was registered (`client.schedules.list` or the equivalent REST endpoint returns a row for this agent_id)

### Shared-memory items (enterprise only)
- Two different user_ids chat the same agent
- Fact stored by user A is reachable when user B asks (if sharedMemory=true)

### SDK-call items
- Run a tiny test script (`.full-auto/qa-cycle-{{cycle_n}}/probe.{lang}`) that does the SDK call and prints the result
- Compare to expected shape

Aggregate every probe's pass/fail into `.full-auto/qa-cycle-{{cycle_n}}/backend.md`.

## Step 5 — Exercise the frontend (if present)

**Detect browser MCP availability** by inspecting the loaded toolset for tool names starting with `mcp__chrome-devtools__` or `mcp__playwright__`. (You can check by attempting ToolSearch on those names; if no match, the MCP isn't installed.)

### If browser MCP available

For each user-flow in the acceptance checklist that describes a UI interaction:

1. Navigate to `http://localhost:{{frontend_port}}`
2. Screenshot → save under `.full-auto/qa-cycle-{{cycle_n}}/frontend/<step-name>.png`
3. Drive the flow:
   - Click selectors, type into inputs, wait for navigation
   - After each step: screenshot + capture console errors via DevTools MCP `getConsoleMessages` or equivalent
4. Final state assertion: text content / URL / state matches what the checklist item demands

Aggregate into `.full-auto/qa-cycle-{{cycle_n}}/frontend.md`.

### If browser MCP NOT available

Skip UI driving. Write into `frontend.md`:

```markdown
# Frontend (cycle {{cycle_n}})

Browser MCP not detected in this session (looked for: mcp__chrome-devtools__*, mcp__playwright__*).

UI was exercised via the API contract only. Add a browser MCP (e.g., chrome-devtools-mcp) to drive UI tests in future runs.

## Smoke test (curl-only)
- `GET http://localhost:{{frontend_port}}/` returned: {{HTTP_STATUS}}
- Body contains: {{key markers from spec, e.g., HTML <title>}}
```

This is a degraded run, NOT a failure — record it but proceed.

## Step 6 — Aggregate cycle results

Write `.full-auto/qa-cycle-{{cycle_n}}.md`:

```markdown
# QA cycle {{cycle_n}} of 5

**Built commit**: {{FINAL_BUILDER_SHA}}

## Backend probes
- ✅ {{N_PASS}} / {{N_TOTAL}}

### Passing
- POST /chat: 200 in 312ms
- (etc)

### Failing
- POST /chat (second user): 500 Internal Server Error
  - Stdout tail: "...panic: nil pointer dereference at user.go:42"
  - Hypothesis: missing nil check around user struct
- Schedule registration probe: 404 (endpoint not found)
  - Hypothesis: builder forgot to wire client.schedules.create

## Frontend
{{INLINE_OR_REFERENCE}}

## Decision
- Failures: {{N_FAIL}}
- Cycle: {{cycle_n}} of 5
- Next: {{ "Phase 6 (all pass)" | "re-dispatch builder for fix cycle" | "BLOCKED (cycle 5 reached)" }}
```

## Step 7 — Decide next

| Condition | Action |
|---|---|
| All checklist items pass | Stop servers (kill bash shells), proceed to Phase 6 |
| Failures exist AND cycle < 5 | Stop servers. Fill `subagent-prompts/fixer-prompt.md.template` with this cycle's report. `SendMessage` to `sonzai-builder`. Wait for return. Increment cycle counter. Re-enter Step 1. |
| Failures exist AND cycle == 5 | Stop servers. Write `.full-auto/BLOCKED.md` with last cycle report. Exit. |

**Always stop servers between cycles** — the builder may rebuild ports, change addresses, or break startup. Don't carry stale processes between cycles.

To stop: send a kill signal or `pkill -f <command-keyword>` via Bash, OR rely on Monitor's session lifecycle. Confirm port is free before re-starting.

## Failure hypothesis hints

When recording failures, include a short "Hypothesis" line per failure — this saves the builder time during the fix cycle. Hypothesize, don't diagnose deeply; the builder can dive in.

| Error class | Hint to include |
|---|---|
| HTTP 500 with stack trace | "stack points at {{file}}:{{line}} — likely {{reading from the trace}}" |
| HTTP 401/403 | "auth header missing or env not loaded" |
| HTTP 404 | "endpoint not registered — likely wiring missed in router" |
| Timeout / no response | "blocking call without timeout, or server not bound to listen address" |
| Schema mismatch | "response missing field X — likely struct tag or marshal config" |
| Console error in browser | "uncaught at {{file}}:{{line}} — likely null deref or missing await" |

The fixer prompt asks the builder to address each failure; clear hypotheses cut fix latency.

## When NOT to re-dispatch

- Builder previously returned BLOCKED → don't re-dispatch (exit with BLOCKED)
- All failures are due to missing env vars / external services the operator must wire (e.g., "Stripe webhook never fires because Stripe is not configured") → record as "operator-action-needed", do NOT ask the builder to fix
- Failures only on flows the transcript explicitly de-scoped → record as "deferred per transcript scope", do NOT ask the builder to fix
