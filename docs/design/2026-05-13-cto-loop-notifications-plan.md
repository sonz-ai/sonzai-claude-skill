# cto-loop + full-auto Notifications Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add outbound Slack DM + Gmail notifications for `cto-loop` (Gate A, Gate B, final report) and `full-auto` (build complete healthy, build done with failing QA). For cto-loop gates, operator can reply via Slack DM or Gmail to drive the gate flow asynchronously. v1.7.0 release.

**Architecture:** Three new subagent prompts (`notifier.md`, `reply-poller.md`) and orchestration files (`notify-setup.md` autonomous + interactive variants) live in `full-auto/` per Rule 6 of repo `CLAUDE.md`. Cto-loop's gate files get an async path that wraps existing terminal-sync behavior. Plugin's `.mcp.json` ships community Gmail + Slack MCPs as fallbacks; Claude Code users get Anthropic's first-party MCPs via `/mcp`; runtime detection chooses what to use.

**Tech Stack:** Markdown skill files (TDD-as-documentation discipline). `.mcp.json` at plugin root. JSON for `plugin.json` / `marketplace.json` / `package.json`. No code dependencies — skills are pure prompt artifacts. Subagents introspect their own tool list to detect MCPs.

**Reference docs:**
- Spec: `docs/design/2026-05-13-cto-loop-notifications-design.md`
- Prior design (v1.6.0): `docs/design/2026-05-13-cto-loop-design.md`
- Repo conventions: `CLAUDE.md` (especially Rules 1, 5, 6)

---

## File Structure (locked-in)

### New files (5)

| Path | Purpose |
|---|---|
| `plugins/sonzai-sdk/.mcp.json` | Community Gmail + Slack MCP fallbacks for non-Claude-Code platforms |
| `plugins/sonzai-sdk/skills/full-auto/subagent-prompts/notifier.md` | Single-shot notification dispatcher (Slack DM + Gmail) |
| `plugins/sonzai-sdk/skills/full-auto/subagent-prompts/reply-poller.md` | Polls Slack DMs + Gmail unread for cto-loop gate replies |
| `plugins/sonzai-sdk/skills/full-auto/notify-setup.md` | Autonomous variant: load config, detect MCPs, no asking |
| `plugins/sonzai-sdk/skills/cto-loop/notify-setup.md` | Interactive variant: same detection + asks for Gmail/Slack if not cached |

### Modified files (17)

| Path | Why |
|---|---|
| `plugins/sonzai-sdk/skills/full-auto/SKILL.md` | Add Phase 0-pre to pipeline overview |
| `plugins/sonzai-sdk/skills/full-auto/pipeline.md` | Insert Phase 0-pre + notify hooks |
| `plugins/sonzai-sdk/skills/full-auto/qa-loop.md` | Dispatch notifier on QA pass / fail-exhaustion |
| `plugins/sonzai-sdk/skills/full-auto/final-report.md.template` | Note which notifications fired |
| `plugins/sonzai-sdk/skills/cto-loop/SKILL.md` | Add Phase 0-pre to overlay table, async behavior |
| `plugins/sonzai-sdk/skills/cto-loop/pipeline.md` | Overlay map shows Phase 0-pre + async gates |
| `plugins/sonzai-sdk/skills/cto-loop/masterplan-gate.md` | Async path: notify + ScheduleWakeup + poll |
| `plugins/sonzai-sdk/skills/cto-loop/cto-review-gate.md` | Same async pattern |
| `plugins/sonzai-sdk/skills/sonzai-sdk/SKILL.md` | Skill-matrix mention of notifications |
| `plugins/sonzai-sdk/.claude-plugin/plugin.json` | v1.7.0 + description |
| `plugins/sonzai-sdk/.codex-plugin/plugin.json` | v1.7.0 + description |
| `plugins/sonzai-internal-staff/.claude-plugin/plugin.json` | v1.7.0 lockstep |
| `plugins/sonzai-internal-staff/.codex-plugin/plugin.json` | v1.7.0 lockstep |
| `.claude-plugin/marketplace.json` | Plugin description update |
| `package.json` | v1.7.0 |
| `CHANGELOG.md` | v1.7.0 entry |
| `README.md` | Notification feature mention |
| `CLAUDE.md` | Rule 7 (notification PII security), Rule 8 (.mcp.json + always-search) |

---

## Task graph

```
Task 1 (research) ─────┐
                       ├──► Task 2 (.mcp.json)
                       │
                       ├──► Task 3 (notifier.md)
                       │
                       └──► Task 4 (reply-poller.md)
                              │
                              ├──► Task 5 (notify-setup.md × 2)
                              │       │
                              │       ├──► Task 6 (full-auto pipeline wiring)
                              │       │
                              │       ├──► Task 7 (cto-loop gate A async)
                              │       │
                              │       ├──► Task 8 (cto-loop gate B async)
                              │       │
                              │       └──► Task 9 (cto-loop overlay docs)
                              │
                              └──► Task 10 (wizard SKILL.md skill-matrix)

Task 11 (manifests v1.7.0) ──► Task 12 (CHANGELOG + README) ──► Task 13 (CLAUDE.md)
                                          │
                                          └──► Task 14 (sanity check) ──► Task 15 (tag + release)
```

Tasks 2/3/4 may proceed in parallel after Task 1.
Tasks 6/7/8/9/10 may proceed in parallel after Task 5.
Tasks 11/12/13 are independent and may proceed in parallel.
Task 14 + 15 are terminal.

---

### Task 1: Verify package names + MCP tool surfaces (research)

**Type:** Research subagent. NO file writes, NO commits. Output is a markdown blob this controller pastes into subsequent tasks.

**Files:**
- Read: `docs/design/2026-05-13-cto-loop-notifications-design.md` (the spec, all sections)

- [ ] **Step 1: Verify Gmail MCP package**

Run:
```bash
npm view @gongrzhe/server-gmail-autoauth-mcp version description repository 2>&1 | head -20
```
Expected: a valid package with version + description matching "Gmail MCP server with auto authentication".

If the package does NOT resolve, also try:
```bash
npm view @gongrzhe/gmail-mcp-server version 2>&1 | head -5
```

Record the EXACT package name that resolves.

- [ ] **Step 2: Verify Slack MCP package (community fallback)**

Run:
```bash
npm view slack-mcp-server version description repository 2>&1 | head -20
```
Expected: valid package, korotovsky's slack-mcp-server.

Record the EXACT package name.

- [ ] **Step 3: Pull README for both, identify exact tool names exposed**

Use WebFetch to read the README.md from each package's GitHub repository (URLs from `npm view`'s output above). For each MCP server, extract the list of tool names it exposes. Specifically check for:

- Gmail: `send_email`, `read_email`, `search_emails`, `list_messages`, `mark_as_read`, or equivalents
- Slack: `slack_post_message`, `slack_post_dm`, `slack_get_dm_history`, `slack_get_unread_messages`, `slack_users_lookupByEmail`, `slack_mark_as_read`, or equivalents

- [ ] **Step 4: Verify Anthropic-shipped tool prefixes**

For Claude Code, Anthropic's built-in MCPs use a `mcp__<server-slug>__<tool>` naming convention. The exact slug for `claude.ai Gmail` and `plugin:slack:slack` is platform-dependent. Output the BEST-GUESS slugs as:

- Gmail (Anthropic-hosted): `mcp__claude_ai_gmail__*` (educated guess; subagents at runtime introspect their actual tool list)
- Slack (Anthropic plugin): `mcp__plugin_slack_slack__*` (same)
- Community Gmail: derived from npm package name (e.g. `mcp__gmail__*` if `.mcp.json` server key is `"gmail"`)
- Community Slack: `mcp__slack__*` if `.mcp.json` server key is `"slack"`

Document this in the output. Detection at runtime is the source of truth; these guesses are seed values for documentation.

- [ ] **Step 5: Report findings**

Return a markdown block in this exact structure (the controller pastes it into Task 2, 3, 4 prompts):

```markdown
## Verified findings

### Gmail community MCP
- Package: `<exact-npm-name>`
- Latest version: <x.y.z>
- Install: `npx -y <package>`
- Tool names: <comma-separated list of actual tools>
- Auth flow: <one-line summary>

### Slack community MCP
- Package: `<exact-npm-name>`
- Latest version: <x.y.z>
- Install: `npx -y <package>`
- Env vars needed: <comma-separated; e.g. SLACK_MCP_XOXP_TOKEN>
- Tool names: <comma-separated list>
- Auth flow: <one-line summary>

### Best-guess Anthropic-shipped tool prefixes
- Gmail: `mcp__claude_ai_gmail__*` (verify at runtime via tool-list introspection)
- Slack: `mcp__plugin_slack_slack__*` (same)

### .mcp.json keys
- Gmail key: `gmail` → tool prefix `mcp__gmail__*`
- Slack key: `slack` → tool prefix `mcp__slack__*`
```

NO commits. This research feeds Tasks 2, 3, 4.

---

### Task 2: Write `plugins/sonzai-sdk/.mcp.json`

**Files:**
- Create: `plugins/sonzai-sdk/.mcp.json`
- Read for context: `docs/design/2026-05-13-cto-loop-notifications-design.md` §`.mcp.json for non-Claude-Code platforms`
- Read Task 1 output for: exact package names + env vars

- [ ] **Step 1: Write `.mcp.json` content**

Replace `<gmail-package>` and `<slack-package>` with the verified package names from Task 1. Replace `<env-var>` with the verified env var name for Slack auth (e.g. `SLACK_MCP_XOXP_TOKEN`).

```json
{
  "mcpServers": {
    "gmail": {
      "command": "npx",
      "args": ["-y", "<gmail-package>"]
    },
    "slack": {
      "command": "npx",
      "args": ["-y", "<slack-package>"],
      "env": {
        "<env-var>": "${<env-var>}"
      }
    }
  }
}
```

- [ ] **Step 2: Verify JSON is valid**

Run:
```bash
cd /Volumes/CORSAIR/code/sonzai/sonzai-claude-skill && jq . plugins/sonzai-sdk/.mcp.json > /dev/null && echo "OK"
```
Expected: `OK`. If jq fails: re-edit until valid.

- [ ] **Step 3: Verify no version pinning**

Run:
```bash
cd /Volumes/CORSAIR/code/sonzai/sonzai-claude-skill && grep -E '@[0-9]+\.[0-9]+\.[0-9]+' plugins/sonzai-sdk/.mcp.json
```
Expected: empty output (no version pins, per Rule 5).

- [ ] **Step 4: Commit**

```bash
cd /Volumes/CORSAIR/code/sonzai/sonzai-claude-skill && git add plugins/sonzai-sdk/.mcp.json && git commit -m "feat(skill): add .mcp.json with community Gmail+Slack fallbacks"
```

---

### Task 3: Write `notifier.md` subagent prompt

**Files:**
- Create: `plugins/sonzai-sdk/skills/full-auto/subagent-prompts/notifier.md`
- Read for context: `docs/design/2026-05-13-cto-loop-notifications-design.md` §`Notification body formats`, §`Security / PII rules`, §`MCP detection`
- Read Task 1 output for: actual MCP tool names + best-guess Anthropic prefixes

- [ ] **Step 1: Write the file**

File content (entire body of `notifier.md`):

````markdown
# Notifier subagent prompt

Dispatched by `notify-setup.md` (during pipeline events) and by `cto-loop` gate flows. Sends ONE notification to operator's configured Gmail + Slack handle. Best-effort.

## Inputs (the dispatching skill passes these in the prompt)

```yaml
event:        gate_a | gate_b | final_report | full_auto_complete | full_auto_failed_qa
run_id:       2026-05-13-1442-a8c1
recipient:
  gmail:      operator@example.com
  slack_user: U07ABC123
body:         |
  <pre-rendered notification body — multi-line, plain text>
subject:      <one-line subject for Gmail>
interactive:  true | false   # if true, body includes reply grammar
```

## What you do (single shot, no loop)

1. Introspect your tool list. Find the FIRST available Gmail send tool, in this precedence:
   - Anthropic-hosted: any `mcp__claude_ai_gmail__send_*` or similar (look for tools containing both `gmail` and `send`)
   - Codex first-party: `mcp__codex_gmail__send_*`
   - Community: `mcp__gmail__send_email` (or similar from the `.mcp.json` fallback)

2. Same for Slack DM:
   - Anthropic plugin: `mcp__plugin_slack_slack__*` looking for `post_dm` or `send_message` with a user-id parameter
   - Codex first-party: `mcp__codex_slack__*`
   - Community: `mcp__slack__*` looking for `slack_post_dm` or `chat_postMessage` with `channel: <user-id>`

3. **Send Gmail** if any Gmail tool was found:
   - `to`: recipient.gmail
   - `subject`: subject (input)
   - `body`: body (input)
   - If the tool requires plain-text vs HTML, prefer plain-text.

4. **Send Slack DM** if any Slack tool was found:
   - `channel`: recipient.slack_user (DM works with user-id as channel)
   - `text`: body (input)
   - If the Slack tool supports it, post the DM in a thread keyed on `run_id` (search prior DMs for `[cto-loop run <run_id>]` token and reply to that thread; else top-level).

5. **Report back**. Return a JSON object:

```json
{
  "gmail":   { "sent": true, "message_id": "<...>" } | { "sent": false, "reason": "<one-line>" } | { "sent": false, "reason": "no_tool_available" },
  "slack":   { "sent": true, "message_ts": "<...>" } | { "sent": false, "reason": "<one-line>" } | { "sent": false, "reason": "no_tool_available" }
}
```

## Behavior on partial failure

- Gmail tool errors → record reason, continue, try Slack.
- Slack tool errors → record reason, continue.
- Both fail → return both reasons; controller decides whether to retry or fall back to terminal-only.

## Hard rules

1. **No PII in body** beyond the recipient header. Body is whatever the dispatching skill passed in (already pre-redacted).
2. **No retries inside this subagent.** Single shot. Controller retries if it wants.
3. **No state writes.** Controller writes message_id / message_ts to `~/.config/sonzai/cto-runs/<run-id>.json`.
4. **No tenant names, no transcript content** in any tool call. If you see either in the body, treat as a controller bug and return error.
````

- [ ] **Step 2: Verify file contains required sections**

Run:
```bash
cd /Volumes/CORSAIR/code/sonzai/sonzai-claude-skill && \
  grep -c "^## " plugins/sonzai-sdk/skills/full-auto/subagent-prompts/notifier.md
```
Expected: ≥5 (Inputs, What you do, Behavior on partial failure, Hard rules, plus the H1)

Run:
```bash
cd /Volumes/CORSAIR/code/sonzai/sonzai-claude-skill && \
  grep -E "mcp__claude_ai_gmail|mcp__plugin_slack_slack|mcp__gmail|mcp__slack" \
    plugins/sonzai-sdk/skills/full-auto/subagent-prompts/notifier.md | wc -l
```
Expected: ≥4 (mentions all four detection paths).

- [ ] **Step 3: Commit**

```bash
cd /Volumes/CORSAIR/code/sonzai/sonzai-claude-skill && \
  git add plugins/sonzai-sdk/skills/full-auto/subagent-prompts/notifier.md && \
  git commit -m "feat(skill): add notifier subagent prompt"
```

---

### Task 4: Write `reply-poller.md` subagent prompt

**Files:**
- Create: `plugins/sonzai-sdk/skills/full-auto/subagent-prompts/reply-poller.md`
- Read for context: `docs/design/2026-05-13-cto-loop-notifications-design.md` §`Reply parsing grammar`, §`Idempotency`, §`Async gate flow`
- Read Task 1 output for: actual MCP tool names

- [ ] **Step 1: Write the file**

File content (entire body of `reply-poller.md`):

````markdown
# Reply-poller subagent prompt

Dispatched by `cto-loop` gate flows on each scheduled wake-up. Checks Slack DMs + Gmail unread for a reply to a specific gate. Returns parsed action or "none".

## Inputs

```yaml
run_id:       2026-05-13-1442-a8c1
gate:         gate_a | gate_b
recipient:
  gmail:      operator@example.com
  slack_user: U07ABC123
last_seen:
  slack_ts:   1715607735.123456   # only consider messages newer than this
  gmail_internal_date: 1715607735000
grammar:      gate_a | gate_b     # which keyword table to use
```

## What you do

### Step 1: Poll Slack DMs

Introspect for available Slack tools (same precedence as notifier.md). Find a tool that lists DM history for the channel = `recipient.slack_user`. Common names:
- `mcp__plugin_slack_slack__conversations_history` with `channel = <user-id>` (returns DM history with that user)
- `mcp__slack__slack_get_dm_history` (community: korotovsky)
- `mcp__claude_ai_gmail__*` does NOT cover Slack — skip if no Slack tool.

Filter: `ts > last_seen.slack_ts` AND `user == recipient.slack_user` (replies FROM the operator).

### Step 2: Poll Gmail

Find a Gmail search tool. Common names:
- `mcp__claude_ai_gmail__search_emails` with query
- `mcp__gmail__search_emails` (community: GongRzhe)

Search query: `is:unread from:<recipient.gmail> subject:"run <run_id>"`. The notifier puts `run <run_id>` in the subject line so replies thread back.

### Step 3: Parse the first new reply

For each new reply (Slack or Gmail), extract the body text. Strip quoted-reply blocks (lines starting with `>`, or after `On <date>, <name> wrote:`).

Apply the grammar table based on input `grammar`:

#### Gate A grammar

| Body starts with (case-insensitive) | Action |
|---|---|
| `approve` | `{ "action": "approve", "raw": "<body>" }` |
| `reject ` + ≥4 chars | `{ "action": "reject", "feedback": "<rest>" }` |
| `edit` | `{ "action": "edit" }` |
| `abort` | `{ "action": "abort" }` |
| anything else | `{ "action": "unparseable", "raw": "<body>" }` |

#### Gate B grammar

| Body starts with | Action |
|---|---|
| `approve` | `{ "action": "approve" }` |
| `feedback ` + ≥4 chars | `{ "action": "feedback", "text": "<rest>" }` |
| `one more ` + ≥4 chars | `{ "action": "one_more", "text": "<rest>" }` |
| `abort` | `{ "action": "abort" }` |
| anything else | `{ "action": "unparseable", "raw": "<body>" }` |

### Step 4: Mark as read

If the message was actioned (anything except `unparseable`):
- Slack: call the available `mark_as_read` tool if exposed, else just record the ts.
- Gmail: call `mark_as_read` or `modify_message` to remove the `UNREAD` label.

If `unparseable`: do NOT mark as read. Skill controller decides whether to send a clarification.

### Step 5: Report back

Return JSON:

```json
{
  "result":      "matched" | "none",
  "source":      "slack" | "gmail" | null,
  "action":      "approve" | "reject" | "edit" | "abort" | "feedback" | "one_more" | "unparseable" | null,
  "text":        "<feedback text or rejection reason or raw body>",
  "new_last_seen": {
    "slack_ts":  "<latest considered Slack ts, even if no match>",
    "gmail_internal_date": <latest considered Gmail internalDate>
  }
}
```

If multiple new replies are present (e.g. operator replied twice via Slack and Gmail), pick the EARLIEST by absolute timestamp across both channels. First reply wins.

## Hard rules

1. **Strict keyword.** No fuzzy match. `lgtm` / `looks good` / `ship it` are ALL `unparseable`. Deliberate intent matches the existing terminal grammar.
2. **No retries on tool error.** Return `result: "none"` with `error` field. Controller schedules next wakeup.
3. **No PII in returned text.** `text` is the operator's reply only, never tool output or system metadata.
4. **No tool side-effects beyond mark-as-read.** Don't post messages, don't move emails to labels other than removing UNREAD.
````

- [ ] **Step 2: Verify required sections**

Run:
```bash
cd /Volumes/CORSAIR/code/sonzai/sonzai-claude-skill && \
  grep -E "^### Step [1-5]:" plugins/sonzai-sdk/skills/full-auto/subagent-prompts/reply-poller.md | wc -l
```
Expected: 5

Run:
```bash
cd /Volumes/CORSAIR/code/sonzai/sonzai-claude-skill && \
  grep -E "Gate [AB] grammar" plugins/sonzai-sdk/skills/full-auto/subagent-prompts/reply-poller.md | wc -l
```
Expected: 2

- [ ] **Step 3: Commit**

```bash
cd /Volumes/CORSAIR/code/sonzai/sonzai-claude-skill && \
  git add plugins/sonzai-sdk/skills/full-auto/subagent-prompts/reply-poller.md && \
  git commit -m "feat(skill): add reply-poller subagent prompt"
```

---

### Task 5: Write `notify-setup.md` (both variants)

**Files:**
- Create: `plugins/sonzai-sdk/skills/full-auto/notify-setup.md` (autonomous)
- Create: `plugins/sonzai-sdk/skills/cto-loop/notify-setup.md` (interactive)
- Read for context: `docs/design/2026-05-13-cto-loop-notifications-design.md` §`Recipient config`, §`MCP detection`, §`Failure modes`

- [ ] **Step 1: Write `full-auto/notify-setup.md`**

File content:

````markdown
# Notify-setup (Phase 0-pre — autonomous variant)

Runs at the very start of `full-auto`, before Phase 0a transcript analysis. Detects whether outbound notifications are possible. Does NOT ask the operator anything (full-auto is unattended).

## When to use

First step of `pipeline.md` for full-auto. Before any transcript work.

## What you do

### Step 1: Load recipient config

Check in this precedence:

1. Env vars: `SONZAI_NOTIFY_GMAIL`, `SONZAI_NOTIFY_SLACK_HANDLE`, `SONZAI_NOTIFY_SLACK_USER_ID`. If all three are set, use them.
2. File: `~/.config/sonzai/cto.json`. If exists, load.
3. Neither: notifications DISABLED for this run. Print one line:

```
Notifications disabled for this run — no recipient configured.
Set SONZAI_NOTIFY_GMAIL + SONZAI_NOTIFY_SLACK_USER_ID, or run /cto-loop once to set ~/.config/sonzai/cto.json.
Proceeding in unattended mode (no notify on completion).
```

Save `notify_enabled = false` to in-memory state, return.

### Step 2: Detect Gmail MCP

Introspect available tools. Check in precedence:

1. `mcp__claude_ai_gmail__*` (Anthropic-hosted)
2. `mcp__codex_gmail__*` (Codex first-party)
3. `mcp__gmail__*` (community fallback from `.mcp.json`)

Record which prefix matched (or null). Call ONE no-op tool (e.g. a list/profile tool) to verify auth. If it returns "needs authentication" / auth error: treat as not-available, record reason.

### Step 3: Detect Slack MCP

Same pattern:

1. `mcp__plugin_slack_slack__*` (Anthropic plugin)
2. `mcp__codex_slack__*` (Codex first-party)
3. `mcp__slack__*` (community fallback)

Verify auth via a profile/lookup call. Record which prefix matched.

### Step 4: Resolve slack_user_id if missing

If recipient config has `slack_handle` but not `slack_user_id`, AND Slack MCP is available: call the available `users_lookupByEmail` or `users_lookup` tool to convert. Cache result back to `~/.config/sonzai/cto.json`.

### Step 5: Summarize state

Save to in-memory state for the rest of the pipeline:

```yaml
notify_enabled:    true | false
gmail_tool_prefix: mcp__claude_ai_gmail__ | mcp__codex_gmail__ | mcp__gmail__ | null
slack_tool_prefix: mcp__plugin_slack_slack__ | mcp__codex_slack__ | mcp__slack__ | null
recipient:
  gmail:         <email-or-null>
  slack_user_id: <id-or-null>
```

Print one line back to terminal:

```
Notifications: Slack ✓ (mcp__plugin_slack_slack__), Gmail ✓ (mcp__claude_ai_gmail__). Will notify on build complete / failing QA.
```

OR if disabled:

```
Notifications disabled — no MCP available or no recipient configured. Build proceeds silently.
```

Continue to Phase 0a.

## Hard rules

1. **Never block on missing MCPs.** This is autonomous mode; degrade silently.
2. **Never ask the operator.** That's cto-loop's job.
3. **No retries on detection.** One pass, record state, move on.
````

- [ ] **Step 2: Write `cto-loop/notify-setup.md`**

File content:

````markdown
# Notify-setup (Phase 0-pre — interactive variant for cto-loop)

Runs at the very start of `/cto-loop`, before Phase 0a transcript analysis. Overlay on `../full-auto/notify-setup.md` — uses the same detection logic, adds an interactive ask if recipient config is missing AND no env vars are set.

## When to use

First step of `cto-loop/pipeline.md`. Before any transcript work.

## What you do

### Step 1: Run the autonomous detection from `../full-auto/notify-setup.md`

Follow Steps 1-3 from `../full-auto/notify-setup.md` exactly — load env vars, check `~/.config/sonzai/cto.json`, detect Gmail + Slack MCP tool prefixes.

### Step 2: If recipient config is missing, ASK

This is the cto-loop-specific behavior. If Step 1 reached the "notifications disabled" branch BUT both Gmail AND Slack MCPs are detected as available, prompt the operator:

```
─────────────────────────────────────────────────────
NOTIFY-SETUP

I detected Gmail and Slack MCPs are available. For async gate replies
(approve / feedback / abort via your inbox or Slack DM), I need:

  Gmail address (where to email you): _______________________
  Slack handle (or user ID if known): _______________________

Reply with both on one line, comma-separated. Examples:
  nas@example.com, @nas
  nas@example.com, U07ABC123

Or type `skip` to run in terminal-only mode (no notifications, you
must come back to this terminal to reply at gates).
─────────────────────────────────────────────────────
```

### Step 3: Capture + persist

If operator replied with values:

1. Validate Gmail looks like an email (`grep -E '^[^@]+@[^@]+\.[^@]+$'`).
2. If slack input starts with `@`, treat as handle and resolve to user_id via Slack `users_lookupByEmail` or `users_lookup`. If it starts with `U` followed by alphanumerics, treat as user_id directly.
3. Save to `~/.config/sonzai/cto.json`:

```bash
mkdir -p ~/.config/sonzai && cat > ~/.config/sonzai/cto.json <<EOF
{
  "gmail": "<email>",
  "slack_handle": "<handle>",
  "slack_user_id": "<resolved-id>"
}
EOF
chmod 600 ~/.config/sonzai/cto.json
```

4. Print confirmation:

```
Saved to ~/.config/sonzai/cto.json. Future cto-loop / full-auto runs will reuse this.
Override with env vars SONZAI_NOTIFY_GMAIL / SONZAI_NOTIFY_SLACK_USER_ID if needed.
```

If operator replied `skip`:

1. Set `notify_enabled = false` in in-memory state.
2. Print: `Skipped. Gates will only accept terminal replies.`
3. Continue.

If invalid input (e.g. one field missing): re-ask once. Second invalid → set `notify_enabled = false`, print one-liner, continue.

### Step 4: Resume `../full-auto/notify-setup.md` from Step 4

Continue with slack_user_id resolution (if needed) and Step 5 (summarize state). The rest of the autonomous flow handles it.

## Hard rules

1. **Ask at most once per run.** Don't re-ask if config exists.
2. **`skip` is explicit.** Empty / whitespace / "no" → re-ask. Only literal `skip` short-circuits.
3. **chmod 600 on `~/.config/sonzai/cto.json`.** Per-user secret file (well, sort-of — contains email + slack handle, not a credential, but treat as sensitive).
4. **Detection MUST pass before asking.** If no Gmail MCP AND no Slack MCP are detected, do NOT ask — there's no notification channel, just print an enablement guide and skip:

```
No Gmail or Slack MCP enabled. Skip notify-setup.
To enable in Claude Code: /mcp → enable claude.ai Gmail + plugin:slack:slack.
To enable in Codex:        codex mcp enable codex_gmail codex_slack.
Notifications disabled. Gates will accept terminal replies only.
```
````

- [ ] **Step 3: Verify both files**

Run:
```bash
cd /Volumes/CORSAIR/code/sonzai/sonzai-claude-skill && \
  test -f plugins/sonzai-sdk/skills/full-auto/notify-setup.md && \
  test -f plugins/sonzai-sdk/skills/cto-loop/notify-setup.md && echo OK
```
Expected: `OK`

Run cross-ref check:
```bash
cd /Volumes/CORSAIR/code/sonzai/sonzai-claude-skill && \
  grep -E "\.\./full-auto/notify-setup\.md" plugins/sonzai-sdk/skills/cto-loop/notify-setup.md | wc -l
```
Expected: ≥1 (cto-loop variant references the autonomous version)

- [ ] **Step 4: Commit**

```bash
cd /Volumes/CORSAIR/code/sonzai/sonzai-claude-skill && \
  git add plugins/sonzai-sdk/skills/full-auto/notify-setup.md \
          plugins/sonzai-sdk/skills/cto-loop/notify-setup.md && \
  git commit -m "feat(skill): add notify-setup.md (autonomous + interactive variants)"
```

---

### Task 6: Wire full-auto pipeline to dispatch notifier on QA terminal events

**Files:**
- Modify: `plugins/sonzai-sdk/skills/full-auto/SKILL.md`
- Modify: `plugins/sonzai-sdk/skills/full-auto/pipeline.md`
- Modify: `plugins/sonzai-sdk/skills/full-auto/qa-loop.md`
- Modify: `plugins/sonzai-sdk/skills/full-auto/final-report.md.template`
- Read for context: `docs/design/2026-05-13-cto-loop-notifications-design.md` §`Notification body formats`, §`Events`

- [ ] **Step 1: Add Phase 0-pre to `SKILL.md`**

Read the current `SKILL.md`. In the pipeline overview section (look for `## Pipeline` or `## Phases` heading and the existing phase list 0a → 5):

- Add a new row/entry for `Phase 0-pre: Notify-setup` at the top, BEFORE Phase 0a.
- Body: "Detects Gmail + Slack MCP availability and loads recipient config. See `notify-setup.md`."
- Add an entry in the file list: `| notify-setup.md | Phase 0-pre: detect notify capability |`

Add to the pipeline summary line near the top of SKILL.md (the one that lists phases): change `0a → 0b → ... → 5` to `0-pre → 0a → 0b → ... → 5`.

- [ ] **Step 2: Add Phase 0-pre to `pipeline.md`**

Read `pipeline.md`. At the very top of the phase list, before `## Phase 0a`, add:

````markdown
## Phase 0-pre: Notify-setup

→ Read `notify-setup.md`

Detect Gmail + Slack MCP availability. Load recipient config from env vars or `~/.config/sonzai/cto.json`. If neither config nor MCPs available, set `notify_enabled = false` and proceed silently. This phase NEVER blocks — degraded notify is OK.

State produced for downstream:
- `notify_enabled`: bool
- `gmail_tool_prefix`, `slack_tool_prefix`: strings or null
- `recipient`: { gmail, slack_user_id }
````

- [ ] **Step 3: Wire notifier in `qa-loop.md`**

Read `qa-loop.md`. Find the "QA passing → final report" branch (search for `final-report.md.template` or similar). Add BEFORE the handoff to final-report:

````markdown
### Notify build complete (best-effort)

If `notify_enabled` is true: dispatch `subagent-prompts/notifier.md` with:

```yaml
event:        full_auto_complete
run_id:       <run-id from state>
recipient:    <from state>
subject:      "[full-auto] Build complete (healthy) — run <run-id>"
body: |
  ✅ full-auto build complete (healthy).

  Run:        <run-id>
  Live URL:   <APP_URL from state>
  Report:     docs/cto-review/<date>-final-report.md
  QA status:  passing
  Time taken: <ELAPSED>

  App is still running (docker compose). Push when ready.
interactive:  false
```

Record returned message IDs in run state. Best-effort; if notifier returns errors for both channels, log one line and continue to final-report.
````

Find the "5 fixer cycles exhausted, give up" branch (search for `5 cycles` or `cycle limit`). Add BEFORE the give-up exit:

````markdown
### Notify build done with failing QA (best-effort)

If `notify_enabled` is true: dispatch `subagent-prompts/notifier.md` with:

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

Record IDs in state. Continue to final-report (which marks QA as failed).
````

- [ ] **Step 4: Update `final-report.md.template`**

Read the file. Add a new section at the bottom, AFTER any existing sections:

````markdown
## Notifications sent

<!-- Filled at end of run. -->
| Event | Channel | Status |
|---|---|---|
{{NOTIFICATIONS_TABLE}}

(Filled with rows from run state's `notifications_sent[]`. Status is `sent` / `failed: <reason>` / `disabled`.)
````

The `{{NOTIFICATIONS_TABLE}}` placeholder gets filled by the final-report writer. Format:

```markdown
| full_auto_complete | Gmail | sent |
| full_auto_complete | Slack | sent |
```

If `notify_enabled` is false, the table reads:
```markdown
| (none) | — | disabled — no recipient/MCP |
```

- [ ] **Step 5: Verify**

Run:
```bash
cd /Volumes/CORSAIR/code/sonzai/sonzai-claude-skill && \
  grep -c "Phase 0-pre" plugins/sonzai-sdk/skills/full-auto/SKILL.md
```
Expected: ≥1

Run:
```bash
cd /Volumes/CORSAIR/code/sonzai/sonzai-claude-skill && \
  grep -c "notifier.md" plugins/sonzai-sdk/skills/full-auto/qa-loop.md
```
Expected: ≥2 (two dispatch points)

Run:
```bash
cd /Volumes/CORSAIR/code/sonzai/sonzai-claude-skill && \
  grep -c "NOTIFICATIONS_TABLE" plugins/sonzai-sdk/skills/full-auto/final-report.md.template
```
Expected: ≥1

- [ ] **Step 6: Commit**

```bash
cd /Volumes/CORSAIR/code/sonzai/sonzai-claude-skill && \
  git add plugins/sonzai-sdk/skills/full-auto/SKILL.md \
          plugins/sonzai-sdk/skills/full-auto/pipeline.md \
          plugins/sonzai-sdk/skills/full-auto/qa-loop.md \
          plugins/sonzai-sdk/skills/full-auto/final-report.md.template && \
  git commit -m "feat(skill): wire full-auto pipeline to notifier on QA terminal events"
```

---

### Task 7: Wire cto-loop Gate A (`masterplan-gate.md`) for async path

**Files:**
- Modify: `plugins/sonzai-sdk/skills/cto-loop/masterplan-gate.md`
- Read for context: `docs/design/2026-05-13-cto-loop-notifications-design.md` §`Async gate flow`, §`Reply parsing grammar`, §`State file`

- [ ] **Step 1: Read current `masterplan-gate.md`**

It's the terminal-sync gate. Keep the existing terminal flow as a section, then layer the async path on top.

- [ ] **Step 2: Restructure the file**

Replace the current content with:

````markdown
# Masterplan gate (Gate A)

Operator approval of the masterplan doc before any build work. **This gate is non-optional.**

The gate has TWO modes depending on whether async notify is set up:

- **Async mode** (default if `notify_enabled` from Phase 0-pre is true AND `ScheduleWakeup` tool is available): print banner, dispatch notifier, save state, ScheduleWakeup, exit turn. On wakeup, dispatch reply-poller; resume on match or reschedule.
- **Sync mode** (fallback): print banner, wait for terminal input only. Original behavior.

## When to use

Reached after `../full-auto/masterplan-assembly.md` has written the masterplan file.

## Print (both modes)

```
─────────────────────────────────────────────────────
🚪 GATE A — MASTERPLAN APPROVAL

File:    docs/cto-review/<YYYY-MM-DD>-masterplan.md
Mode:    <greenfield | brownfield>
Sonzai:  <archetype>, <runtime-mode>, [<capabilities>]
Stack:   <backend> + <frontend> + <db> + <auth>
Scope:   <comma-list>
Risks:   <count> identified
Verified: <date>

Reply with one of:
  approve         → I proceed to build (Phase 2)
  edit            → I pause; you edit the file; type 'done'
  reject <txt>    → I re-derive with your feedback and re-emit
  abort           → stop, leave masterplan in place

[Async note (only printed in async mode):]
  Also notified via Slack DM + Gmail. Reply through any channel.
  Polling every 20min for up to 24h.
─────────────────────────────────────────────────────
```

## Mode selection

At Gate A entry:

1. Read in-memory state: is `notify_enabled` true?
2. Check tool list: is `ScheduleWakeup` available?
3. If BOTH true → **async mode**. Else → **sync mode**.

## Async mode

### Step 1: Notify

Dispatch `../full-auto/subagent-prompts/notifier.md`:

```yaml
event:        gate_a
run_id:       <run-id>
recipient:    <from state>
subject:      "[cto-loop] Gate A — Masterplan ready — run <run-id>"
body: |
  🚪 cto-loop Gate A — Masterplan ready for your review.

  Run:     <run-id>
  File:    <masterplan path>
  Mode:    <greenfield|brownfield>
  Stack:   <stack summary>
  Scope:   <one-line>

  Reply with:
    approve
    reject <reason>
    edit             (edit the file directly, then reply 'approved')
    abort

  (Waiting up to 24h. Polling every 20min.)
interactive:  true
```

Record returned IDs in state.

### Step 2: Save state + schedule wakeup

Save run state to `~/.config/sonzai/cto-runs/<run-id>.json`:

```json
{
  "run_id": "<run-id>",
  "started_at": "<iso8601>",
  "base_sha": "<sha>",
  "mode": "cto-loop",
  "masterplan_path": "<path>",
  "current_phase": "gate_a",
  "recipients": { "gmail": "<...>", "slack_user_id": "<...>" },
  "notifications_sent": [ /* from notifier */ ],
  "first_poll_at": "<iso8601>",
  "last_poll_at": "<iso8601>",
  "poll_count": 0,
  "timeout_at": "<iso8601 + 24h>",
  "last_seen": { "slack_ts": "0", "gmail_internal_date": 0 },
  "paused": false
}
```

Call `ScheduleWakeup`:

```yaml
delaySeconds: 1200
reason:       "cto-loop Gate A waiting for reply"
prompt:       "<original /loop prompt — re-fires cto-loop, which detects existing state file and resumes>"
```

Exit this turn.

### Step 3 (on wakeup): Dispatch reply-poller

When the agent re-enters via the wakeup, it should detect the existing state file at `~/.config/sonzai/cto-runs/<run-id>.json` with `current_phase = gate_a`, and dispatch `../full-auto/subagent-prompts/reply-poller.md`:

```yaml
run_id:    <run-id>
gate:      gate_a
recipient: <from state>
last_seen: <from state>
grammar:   gate_a
```

### Step 4: Process result

Based on poller's return:

- `result: "matched", action: "approve"` → process as "approve" action below. Clear `current_phase`, advance to Phase 2.
- `result: "matched", action: "reject", text: "<reason>"` → process as "reject <reason>" action below (re-derive).
- `result: "matched", action: "edit"` → print banner "Operator chose to edit. Pause until file is approved." Re-schedule wakeup at 600s (10min, faster cadence for edit mode); on next wake, just re-poll for `approve`.
- `result: "matched", action: "abort"` → process as "abort".
- `result: "matched", action: "unparseable"` → if state.unparseable_count >= 2 (third unparseable within run): dispatch notifier with a clarification body (see design §`Reply parsing grammar` → Ambiguous section). Increment count, re-schedule. Else: silently re-schedule.
- `result: "none"` → check timeout. If `now >= timeout_at`: pause (Step 5). Else: re-schedule wakeup.

### Step 5: Timeout / pause

If `now >= timeout_at`:

1. Dispatch notifier:
   ```yaml
   event:    gate_a_paused
   subject:  "[cto-loop] Gate A timed out — run <run-id>"
   body: |
     ⏸ cto-loop Gate A paused after 24h with no reply.
     Resume with /cto-loop resume <run-id> when ready.
   interactive: false
   ```
2. Set `state.paused = true`, save state.
3. Print to terminal: `Gate A timed out at 24h. Resume with /cto-loop resume <run-id>.`
4. Exit. Do NOT delete state.

## Sync mode

(Existing terminal-sync behavior — unchanged.)

[Keep the existing terminal-only "approve / edit / reject / abort" action handling section exactly as it was. Below.]

## Action handling (used by BOTH modes)

### `approve`

1. Mark the masterplan's approval checkbox: replace `- [ ] approved` with `- [x] approved`.
2. Save the masterplan path to in-memory state.
3. Clear async state file's `current_phase` (move on).
4. Proceed to `../full-auto/builder-dispatch.md`.

### `edit`

1. (Sync mode) Print: "Paused. Edit `<file>` directly. Type `done` when ready."
   (Async mode) Print: "Pause for edit; re-poll cadence reduced to 10min until you reply 'approve'."
2. Wait / re-poll until `approve`.
3. On `approve`: re-read file, re-print banner, re-ask. Loop until `approve` or `reject`.

### `reject <txt>`

1. Capture rejection text into state.
2. Go back to `../full-auto/answer-derivation.md` with rejection in inputs.
3. Re-run `../full-auto/masterplan-assembly.md` — overwrite same file.
4. Return to Gate A for re-approval.
5. Bounded: 3 rejection cycles. After 3, escalate (per existing rule).

### `abort`

1. Print: "Aborting. Masterplan left at <path>. No build performed."
2. Clear state file (or set `paused: true, aborted: true`).
3. Exit.

## Hard rules

1. **Mode auto-selected, not user-selected.** Async is on whenever the prerequisites are met. No flag.
2. **`approve` is explicit.** No "lgtm" / "looks good" auto-approve. Same for async replies.
3. **Bounded reject cycles.** 3 max, same as terminal mode.
4. **Terminal input is always honored in BOTH modes.** Even in async mode, if the operator happens to be at the terminal and types `approve`, that path works. Async polling is additive.
5. **State file is local-only.** `~/.config/sonzai/cto-runs/` is per-user. Never committed.
````

- [ ] **Step 3: Verify**

Run:
```bash
cd /Volumes/CORSAIR/code/sonzai/sonzai-claude-skill && \
  grep -cE "Async mode|Sync mode" plugins/sonzai-sdk/skills/cto-loop/masterplan-gate.md
```
Expected: ≥3

Run:
```bash
cd /Volumes/CORSAIR/code/sonzai/sonzai-claude-skill && \
  grep -cE "ScheduleWakeup|reply-poller|notifier\.md" plugins/sonzai-sdk/skills/cto-loop/masterplan-gate.md
```
Expected: ≥4

- [ ] **Step 4: Commit**

```bash
cd /Volumes/CORSAIR/code/sonzai/sonzai-claude-skill && \
  git add plugins/sonzai-sdk/skills/cto-loop/masterplan-gate.md && \
  git commit -m "feat(skill): wire cto-loop Gate A async notify + poll path"
```

---

### Task 8: Wire cto-loop Gate B (`cto-review-gate.md`) for async path + final-report notify

**Files:**
- Modify: `plugins/sonzai-sdk/skills/cto-loop/cto-review-gate.md`
- Read for context: `docs/design/2026-05-13-cto-loop-notifications-design.md` §`Async gate flow`, §`Notification body formats` Gate B + final report

- [ ] **Step 1: Read current `cto-review-gate.md`**

- [ ] **Step 2: Restructure file**

The transformation mirrors Task 7 but with Gate B-specific grammar and the final-report notification.

At the top, after `# CTO review gate (Gate B)` heading, add the same `## Mode selection` section as Task 7. Then add `## Async mode` section with these specifics:

### Step 1 (async): Notify

```yaml
event:        gate_b
run_id:       <run-id>
recipient:    <from state>
subject:      "[cto-loop] Gate B — Live app ready (cycle <N>) — run <run-id>"
body: |
  🚪 cto-loop Gate B — Live app ready (cycle <N>/5).

  Run:        <run-id>
  Live URL:   <APP_URL>
  Commits:    <BASE_SHA>..HEAD (<N> commits)
  Built files: <FILE_COUNT>
  QA status:  <QA_STATUS>
  Time taken: <ELAPSED>

  What was built:
  <THREE_PARA_SUMMARY>

  Reply with:
    approve              → final report, done
    feedback <text>      → I dispatch fixer, re-deploy, return here
    one more <text>      → only valid after cycle 5; force-extra cycle
    abort                → stop, leave docker compose running

  (Waiting up to 24h. Polling every 20min.)
interactive:  true
```

### Step 4 (process result):

- `action: "approve"` → process as approve below. ALSO dispatch notifier with `event: final_report`:

  ```yaml
  event:        final_report
  subject:      "[cto-loop] Complete (approved) — run <run-id>"
  body: |
    ✅ cto-loop complete (approved).

    Run:          <run-id>
    Live URL:     <APP_URL>
    Report:       docs/cto-review/<date>-final-report.md
    Total cycles: <CYCLE_N>

    App is still running (docker compose). Push when ready.
  interactive:  false
  ```

- `action: "feedback", text: "<txt>"` → append to `docs/cto-review/<date>-feedback-log.md`, dispatch `feedback-iteration.md`, re-enter Gate B on next deploy. Reset `last_seen` and re-schedule wakeup.

- `action: "one_more", text: "<txt>"` → only valid if `cycle_n >= 5`. Bypass cycle limit once. Process as feedback.

- `action: "abort"` → notify with abort event:
  ```yaml
  event:    gate_b_aborted
  subject:  "[cto-loop] Aborted at Gate B — run <run-id>"
  body: |
    🛑 cto-loop aborted at Gate B.
    docker compose still running. Inspect or `docker compose down`.
  ```
  Set state to aborted. Exit. No final report.

The existing terminal-sync `approve / feedback / abort / one more` handling stays unchanged in the `## Action handling` section. The cycle counter logic + 5-cycle warning + 6-cycle hard-stop also stays.

- [ ] **Step 3: Verify**

Run:
```bash
cd /Volumes/CORSAIR/code/sonzai/sonzai-claude-skill && \
  grep -cE "Async mode|notifier|reply-poller" plugins/sonzai-sdk/skills/cto-loop/cto-review-gate.md
```
Expected: ≥4

Run:
```bash
cd /Volumes/CORSAIR/code/sonzai/sonzai-claude-skill && \
  grep -cE "final_report|gate_b" plugins/sonzai-sdk/skills/cto-loop/cto-review-gate.md
```
Expected: ≥3

- [ ] **Step 4: Commit**

```bash
cd /Volumes/CORSAIR/code/sonzai/sonzai-claude-skill && \
  git add plugins/sonzai-sdk/skills/cto-loop/cto-review-gate.md && \
  git commit -m "feat(skill): wire cto-loop Gate B async path + final-report notify"
```

---

### Task 9: Update cto-loop pipeline + SKILL.md to reflect Phase 0-pre + async

**Files:**
- Modify: `plugins/sonzai-sdk/skills/cto-loop/SKILL.md`
- Modify: `plugins/sonzai-sdk/skills/cto-loop/pipeline.md`

- [ ] **Step 1: Update `cto-loop/SKILL.md`**

Read the existing SKILL.md. In the `## Pipeline overlay` table (or wherever phases are listed), add a NEW row at the top:

```markdown
| Phase | full-auto behavior | cto-loop overlay |
|---|---|---|
| **0-pre Notify-setup** | `../full-auto/notify-setup.md` (autonomous) | **`notify-setup.md` (interactive — asks if missing)** |
```

Also update the "## Files (cto-loop-specific only)" table to add:

```markdown
| `notify-setup.md` | Phase 0-pre: interactive ask for Gmail/Slack if not cached |
```

Update the description in YAML frontmatter to mention "async notification + reply via Slack/Gmail" as a feature:

The current frontmatter description ends with something like "Runs full-auto's pipeline with two operator gates inserted and an interactive tech-stack intake (7 questions) instead of autonomous defaults." Append: " Operator can ALSO reply to gates via Slack DM or Gmail when those MCPs are enabled — skill polls every 20min for up to 24h."

- [ ] **Step 2: Update `cto-loop/pipeline.md`**

Read pipeline.md. At the top of the phase overlay map, add a row for Phase 0-pre:

```markdown
| **0-pre Notify-setup** | `../full-auto/notify-setup.md` (autonomous) | **`notify-setup.md`** (interactive 1Q ask if no cache) |
```

Add a section header `## Phase 0-pre` near the top, before the existing phases, with:

```markdown
## Phase 0-pre: Notify-setup

→ Read `notify-setup.md`

Interactive overlay on `../full-auto/notify-setup.md`. Detects Gmail + Slack MCP availability. If recipient config missing AND at least one MCP available, asks operator once for Gmail address + Slack handle; caches to `~/.config/sonzai/cto.json`. Used by Gate A and Gate B for async reply support.

If no MCPs available OR operator says `skip`: cto-loop runs in terminal-only sync mode (same as full-auto with notify disabled).
```

Also add a `## Async gates` section near the bottom, describing the wakeup loop briefly:

```markdown
## Async gates

When `notify_enabled` is true AND `ScheduleWakeup` is available, Gates A and B run in async mode:

1. Print gate banner (same as terminal).
2. Dispatch `../full-auto/subagent-prompts/notifier.md` to send Slack DM + Gmail.
3. Save run state to `~/.config/sonzai/cto-runs/<run-id>.json`.
4. `ScheduleWakeup(1200s)`, exit turn.
5. On wakeup: dispatch `../full-auto/subagent-prompts/reply-poller.md`, process result.
6. Loop steps 4-5 until reply OR 24h timeout (then pause + resumable).

Terminal input remains an override path even in async mode.
```

- [ ] **Step 3: Verify**

Run:
```bash
cd /Volumes/CORSAIR/code/sonzai/sonzai-claude-skill && \
  grep -c "Phase 0-pre\|0-pre Notify-setup" plugins/sonzai-sdk/skills/cto-loop/SKILL.md
```
Expected: ≥1

Run:
```bash
cd /Volumes/CORSAIR/code/sonzai/sonzai-claude-skill && \
  grep -c "Async gates\|async mode" plugins/sonzai-sdk/skills/cto-loop/pipeline.md
```
Expected: ≥1

- [ ] **Step 4: Commit**

```bash
cd /Volumes/CORSAIR/code/sonzai/sonzai-claude-skill && \
  git add plugins/sonzai-sdk/skills/cto-loop/SKILL.md plugins/sonzai-sdk/skills/cto-loop/pipeline.md && \
  git commit -m "feat(skill): document Phase 0-pre + async gates in cto-loop pipeline"
```

---

### Task 10: Update wizard SKILL.md skill-matrix

**Files:**
- Modify: `plugins/sonzai-sdk/skills/sonzai-sdk/SKILL.md`

- [ ] **Step 1: Update the "Going beyond the wizard" table**

Read the SKILL.md. Find the existing skill-matrix:

```markdown
| A meeting transcript + no human in loop | Unattended autonomous build → running app | `full-auto` (sibling) |
| A meeting transcript + tech-lead in loop | Supervised build with 2 gates (masterplan + live-app review) | `cto-loop` (sibling) |
```

Update those two rows:

```markdown
| A meeting transcript + no human in loop | Unattended autonomous build → running app. Sends Slack/Gmail notification on completion (build done healthy / build done failing QA) when MCPs are enabled. | `full-auto` (sibling) |
| A meeting transcript + tech-lead in loop | Supervised build with 2 gates (masterplan + live-app review). Operator can reply to gates from Slack DM or Gmail (with MCPs enabled) — terminal isn't required. | `cto-loop` (sibling) |
```

Update the paragraph below the table that starts with "Both `full-auto` and `cto-loop`...":

Append: " Both auto-dispatch a `notifier` subagent at terminal events when Gmail + Slack MCPs are enabled. `cto-loop` additionally polls for replies via Slack DM + Gmail unread (24h budget, paused-resumable)."

- [ ] **Step 2: Verify**

Run:
```bash
cd /Volumes/CORSAIR/code/sonzai/sonzai-claude-skill && \
  grep -c "Slack/Gmail\|Slack DM\|Gmail" plugins/sonzai-sdk/skills/sonzai-sdk/SKILL.md
```
Expected: ≥2

- [ ] **Step 3: Commit**

```bash
cd /Volumes/CORSAIR/code/sonzai/sonzai-claude-skill && \
  git add plugins/sonzai-sdk/skills/sonzai-sdk/SKILL.md && \
  git commit -m "feat(skill): mention notifications in wizard skill-matrix"
```

---

### Task 11: Bump manifests to v1.7.0

**Files:**
- Modify: `package.json`
- Modify: `plugins/sonzai-sdk/.claude-plugin/plugin.json`
- Modify: `plugins/sonzai-sdk/.codex-plugin/plugin.json`
- Modify: `plugins/sonzai-internal-staff/.claude-plugin/plugin.json`
- Modify: `plugins/sonzai-internal-staff/.codex-plugin/plugin.json`
- Modify: `.claude-plugin/marketplace.json`

- [ ] **Step 1: Bump `package.json` to 1.7.0**

```bash
cd /Volumes/CORSAIR/code/sonzai/sonzai-claude-skill && \
  jq '.version = "1.7.0"' package.json > package.json.tmp && \
  mv package.json.tmp package.json
```

Verify:
```bash
cd /Volumes/CORSAIR/code/sonzai/sonzai-claude-skill && jq -r .version package.json
```
Expected: `1.7.0`

- [ ] **Step 2: Bump all four plugin.json files to 1.7.0**

For each plugin.json, update the version AND extend the description to mention notifications:

```bash
cd /Volumes/CORSAIR/code/sonzai/sonzai-claude-skill && \
  for f in plugins/sonzai-sdk/.claude-plugin/plugin.json \
           plugins/sonzai-sdk/.codex-plugin/plugin.json \
           plugins/sonzai-internal-staff/.claude-plugin/plugin.json \
           plugins/sonzai-internal-staff/.codex-plugin/plugin.json; do
    jq '.version = "1.7.0"' "$f" > "$f.tmp" && mv "$f.tmp" "$f"
    echo "Bumped $f"
  done
```

Then manually edit the two `sonzai-sdk/plugin.json` descriptions (Claude + Codex) to append after the existing v1.6.0 text:

"v1.7.0 adds outbound Slack DM + Gmail notifications on pipeline events for both full-auto and cto-loop, and async reply support for cto-loop gates (operator can approve/feedback/abort via Slack DM or Gmail; skill polls 20min cadence, 24h timeout, resumable)."

Use Edit tool to append to the `.description` field for each.

- [ ] **Step 3: Update `marketplace.json`**

Edit `.claude-plugin/marketplace.json`. Append the same notifications text to the `sonzai-sdk` plugin's `description` field.

Verify all six JSON files are valid:
```bash
cd /Volumes/CORSAIR/code/sonzai/sonzai-claude-skill && \
  for f in package.json .claude-plugin/marketplace.json \
           plugins/sonzai-sdk/.claude-plugin/plugin.json \
           plugins/sonzai-sdk/.codex-plugin/plugin.json \
           plugins/sonzai-internal-staff/.claude-plugin/plugin.json \
           plugins/sonzai-internal-staff/.codex-plugin/plugin.json; do
    jq . "$f" > /dev/null && echo "OK $f" || echo "FAIL $f"
  done
```
Expected: all six `OK`.

- [ ] **Step 4: Verify version consistency**

Run:
```bash
cd /Volumes/CORSAIR/code/sonzai/sonzai-claude-skill && \
  grep -E '"version"' package.json plugins/*/.claude-plugin/plugin.json plugins/*/.codex-plugin/plugin.json
```
Expected: all five show `"version": "1.7.0"`.

- [ ] **Step 5: Commit**

```bash
cd /Volumes/CORSAIR/code/sonzai/sonzai-claude-skill && \
  git add package.json .claude-plugin/marketplace.json plugins/*/.claude-plugin/plugin.json plugins/*/.codex-plugin/plugin.json && \
  git commit -m "chore: bump to v1.7.0 (notifications + async reply)"
```

---

### Task 12: Update CHANGELOG.md + README.md

**Files:**
- Modify: `CHANGELOG.md`
- Modify: `README.md`

- [ ] **Step 1: Prepend v1.7.0 entry to CHANGELOG.md**

Read CHANGELOG.md. Add a new entry at the TOP (above v1.6.0):

```markdown
## v1.7.0 — 2026-05-13

### Added — Notifications + async reply

**Both `full-auto` and `cto-loop` now dispatch outbound notifications** (Slack DM + Gmail) at pipeline terminal events:
- `full-auto`: Build complete healthy, build done with failing QA
- `cto-loop`: Gate A reached, Gate B reached, final report written, abort

**`cto-loop` adds async reply support**: when Gmail + Slack MCPs are enabled, operator can reply to Gate A and Gate B from their inbox or Slack DM. First reply (terminal, Slack, or Gmail) wins. Polling cadence 20min, timeout 24h, then paused-resumable.

### New files
- `plugins/sonzai-sdk/.mcp.json` — community Gmail + Slack MCP fallbacks for non-Claude-Code platforms
- `plugins/sonzai-sdk/skills/full-auto/notify-setup.md` — autonomous variant
- `plugins/sonzai-sdk/skills/full-auto/subagent-prompts/notifier.md` — notification dispatcher
- `plugins/sonzai-sdk/skills/full-auto/subagent-prompts/reply-poller.md` — reply poller
- `plugins/sonzai-sdk/skills/cto-loop/notify-setup.md` — interactive variant (asks if not cached)

### Modified
- Full-auto pipeline + qa-loop wire notifier on terminal events
- Cto-loop masterplan-gate + cto-review-gate add async ScheduleWakeup branches
- Wizard SKILL.md mentions notifications in skill-matrix
- Repo CLAUDE.md adds Rule 7 (notification PII) + Rule 8 (.mcp.json always-search)

### MCP detection
Skill detects available MCP tools at runtime. Precedence:
1. Anthropic-shipped: `claude.ai Gmail` (via `/mcp`) + `plugin:slack:slack` (via `/plugin install slack`)
2. Codex first-party: `codex_gmail` + `codex_slack` (v0.117.0+)
3. Community fallback: `.mcp.json` ships GongRzhe Gmail MCP + korotovsky Slack MCP

If neither, falls back to terminal-only sync mode (v1.6.0 behavior).

### Hard rules
- Notifications are best-effort — failed send never blocks pipeline
- No transcript / customer / tenant content in any notification body
- Strict keyword reply grammar (no fuzzy match — same as terminal)
- State file `~/.config/sonzai/cto-runs/<run-id>.json` is local-only
```

- [ ] **Step 2: Update README.md**

Read README.md. Find the section that lists the three public skills (likely "Skills" or "What's in this repo"). Add a bullet under cto-loop:

"- **Async replies via Slack/Gmail (v1.7.0+):** when Gmail + Slack MCPs are enabled, operator can approve/feedback/abort gates from their inbox or Slack DM instead of returning to the terminal. 20min poll cadence, 24h timeout, paused-resumable."

Add a bullet under full-auto:

"- **Build-complete notifications (v1.7.0+):** sends a Slack DM + Gmail when the build finishes (healthy or with failing QA). Useful for long unattended runs."

Add a short "Notification setup" section before "Install":

```markdown
## Notification setup (optional)

For Slack DM + Gmail notifications + async reply support in cto-loop:

**Claude Code:**
1. `/mcp` → enable `claude.ai Gmail` (authenticate)
2. `/mcp` → enable `plugin:slack:slack` (authenticate)

**Codex:**
1. `codex mcp enable codex_gmail`
2. `codex mcp enable codex_slack`

**Other platforms (Gemini CLI, Cursor):** the plugin's `.mcp.json` ships community fallbacks (`@gongrzhe/...` for Gmail, `slack-mcp-server` for Slack). Set `SLACK_MCP_XOXP_TOKEN` env var for Slack; Gmail OAuths on first use.

If no notifications are wanted, do nothing — both skills fall back to terminal-only mode.
```

- [ ] **Step 3: Verify**

Run:
```bash
cd /Volumes/CORSAIR/code/sonzai/sonzai-claude-skill && \
  grep -c "v1.7.0" CHANGELOG.md
```
Expected: ≥1

Run:
```bash
cd /Volumes/CORSAIR/code/sonzai/sonzai-claude-skill && \
  grep -c "Notification setup\|Slack DM" README.md
```
Expected: ≥1

- [ ] **Step 4: Commit**

```bash
cd /Volumes/CORSAIR/code/sonzai/sonzai-claude-skill && \
  git add CHANGELOG.md README.md && \
  git commit -m "docs: v1.7.0 changelog + README notification setup section"
```

---

### Task 13: Update repo CLAUDE.md with new hard rules

**Files:**
- Modify: `CLAUDE.md`

- [ ] **Step 1: Add Rule 7 + Rule 8**

Read CLAUDE.md. After existing Rule 6 (shared core in full-auto/), add:

```markdown
### 7. Notification bodies obey Phase 0a's PII / tenant-name rules

Notifications from `notifier.md` are derived from masterplan + run state, never from raw transcript. Same redaction as Phase 0a applies: no tenant / client / customer names, no PII (operator name beyond recipient header), no transcript content, no code snippets, no API keys, no remote URLs (Live URLs are localhost only).

The notifier subagent is forbidden from creative composition — it sends exactly what the dispatching skill puts in `body`. If a contributor adds dynamic body-generation logic that reaches outside the run state, reject.

### 8. `.mcp.json` follows always-search rule too

The plugin's `.mcp.json` MUST NOT pin specific package versions. Use `@latest` or unpinned. If you're adding a new MCP server to `.mcp.json`, verify the package name + invocation flags via `npm view` at write-time — same rule as Rule 5.
```

- [ ] **Step 2: Verify**

Run:
```bash
cd /Volumes/CORSAIR/code/sonzai/sonzai-claude-skill && \
  grep -cE "^### 7\.|^### 8\." CLAUDE.md
```
Expected: 2

- [ ] **Step 3: Commit**

```bash
cd /Volumes/CORSAIR/code/sonzai/sonzai-claude-skill && \
  git add CLAUDE.md && \
  git commit -m "docs: add Rule 7 (notification PII) + Rule 8 (.mcp.json always-search) to CLAUDE.md"
```

---

### Task 14: Sanity check (cross-refs + forbidden patterns + JSON validity)

**Type:** Validation-only subagent. NO file writes. Reports issues; controller fixes via additional tasks if any are found.

- [ ] **Step 1: Cross-reference resolution**

Run:
```bash
cd /Volumes/CORSAIR/code/sonzai/sonzai-claude-skill && \
  grep -rEoh '\.\./full-auto/[a-zA-Z0-9._/-]+\.(md|template|json)' plugins/sonzai-sdk/skills/cto-loop/ | sort -u | \
  while IFS= read -r ref; do
    target="plugins/sonzai-sdk/skills/cto-loop/$ref"
    if [ -f "$target" ]; then
      echo "OK   $ref"
    else
      echo "MISS $ref"
    fi
  done
```
Expected: every line starts with `OK`. Any `MISS` → controller fixes the cross-ref.

- [ ] **Step 2: Forbidden patterns in public plugin**

Run:
```bash
cd /Volumes/CORSAIR/code/sonzai/sonzai-claude-skill && \
  grep -rnE "Razer|Eragon|PocketSouls|sonzai-ai-monolith-ts|services/contextengine|services/ai-service|platform/api/internal|DuckLake|BigQuery" plugins/sonzai-sdk/ | grep -v "\.git"
```
Expected: empty output. Any hits → controller fixes.

- [ ] **Step 3: Always-search compliance in `.mcp.json`**

Run:
```bash
cd /Volumes/CORSAIR/code/sonzai/sonzai-claude-skill && \
  grep -E '@[0-9]+\.[0-9]+\.[0-9]+|version.*[0-9]+\.[0-9]+\.[0-9]+' plugins/sonzai-sdk/.mcp.json
```
Expected: empty output.

- [ ] **Step 4: All JSON files valid**

Run:
```bash
cd /Volumes/CORSAIR/code/sonzai/sonzai-claude-skill && \
  for f in package.json .claude-plugin/marketplace.json \
           plugins/sonzai-sdk/.mcp.json \
           plugins/sonzai-sdk/.claude-plugin/plugin.json \
           plugins/sonzai-sdk/.codex-plugin/plugin.json \
           plugins/sonzai-internal-staff/.claude-plugin/plugin.json \
           plugins/sonzai-internal-staff/.codex-plugin/plugin.json; do
    jq . "$f" > /dev/null && echo "OK $f" || echo "FAIL $f"
  done
```
Expected: all `OK`.

- [ ] **Step 5: Version consistency**

Run:
```bash
cd /Volumes/CORSAIR/code/sonzai/sonzai-claude-skill && \
  jq -r .version package.json plugins/*/.claude-plugin/plugin.json plugins/*/.codex-plugin/plugin.json
```
Expected: all lines say `1.7.0`.

- [ ] **Step 6: New files exist**

Run:
```bash
cd /Volumes/CORSAIR/code/sonzai/sonzai-claude-skill && \
  for f in plugins/sonzai-sdk/.mcp.json \
           plugins/sonzai-sdk/skills/full-auto/notify-setup.md \
           plugins/sonzai-sdk/skills/full-auto/subagent-prompts/notifier.md \
           plugins/sonzai-sdk/skills/full-auto/subagent-prompts/reply-poller.md \
           plugins/sonzai-sdk/skills/cto-loop/notify-setup.md; do
    test -f "$f" && echo "OK $f" || echo "MISS $f"
  done
```
Expected: all `OK`.

- [ ] **Step 7: Report**

Return a markdown block listing any FAIL / MISS results. If all clean: report "All checks pass." Controller acts on findings.

---

### Task 15: Tag + GitHub release v1.7.0

**Files:** Git operations only.

- [ ] **Step 1: Verify all commits on main**

```bash
cd /Volumes/CORSAIR/code/sonzai/sonzai-claude-skill && \
  git status && git log --oneline -20
```
Expected: working tree clean. Latest commits include all Task 2-13 commits.

- [ ] **Step 2: Push to remote**

```bash
cd /Volumes/CORSAIR/code/sonzai/sonzai-claude-skill && git push
```
Expected: pushes Task 2-13 commits to origin/main.

- [ ] **Step 3: Create annotated tag**

```bash
cd /Volumes/CORSAIR/code/sonzai/sonzai-claude-skill && \
  git tag -a v1.7.0 -m "v1.7.0 — notifications + async reply

Adds outbound Slack DM + Gmail notifications on pipeline terminal events
for both full-auto and cto-loop. cto-loop additionally polls for replies
to gates via Slack DM + Gmail unread (20min cadence, 24h timeout).

New files (5):
- .mcp.json (community Gmail+Slack fallbacks)
- full-auto/notify-setup.md (autonomous)
- cto-loop/notify-setup.md (interactive)
- full-auto/subagent-prompts/notifier.md
- full-auto/subagent-prompts/reply-poller.md

Modified (17): pipeline files, gate files, manifests, docs, CLAUDE.md
"
```

- [ ] **Step 4: Push tag**

```bash
cd /Volumes/CORSAIR/code/sonzai/sonzai-claude-skill && git push origin v1.7.0
```

- [ ] **Step 5: Create GitHub release**

```bash
cd /Volumes/CORSAIR/code/sonzai/sonzai-claude-skill && \
  gh release create v1.7.0 \
    --title "v1.7.0 — notifications + async reply" \
    --notes "$(cat <<'EOF'
Adds outbound Slack DM + Gmail notifications on pipeline terminal events for both `full-auto` and `cto-loop`. `cto-loop` additionally polls for replies via Slack DM + Gmail unread (20 min cadence, 24 h timeout, paused-resumable).

## What's new

**Both skills:** dispatch outbound notification at terminal events. full-auto on build complete or failing QA. cto-loop on Gate A, Gate B, final report.

**cto-loop only:** async reply support. Operator can reply to a gate via Slack DM or Gmail reply (or terminal — first reply wins). Strict keyword grammar (approve / reject `<txt>` / feedback `<txt>` / abort), same as terminal.

**MCP detection:** runtime-introspect available Gmail/Slack MCPs. Anthropic's first-party (`claude.ai Gmail`, `plugin:slack:slack`) is preferred; Codex first-party second; community fallbacks (shipped in plugin's `.mcp.json`) third. If neither, degrades to v1.6.0 terminal-sync.

**Resumability:** 24 h cumulative wait then state-pause with `/cto-loop resume <run-id>`.

## Notification setup

**Claude Code:** `/mcp` → enable Gmail + Slack, authenticate each.
**Codex:** `codex mcp enable codex_gmail codex_slack`.
**Other:** plugin's `.mcp.json` auto-loads community Gmail + Slack on plugin enable.

## File-level changes

5 new files, 17 modified. See CHANGELOG.md and `docs/design/2026-05-13-cto-loop-notifications-design.md` for the full spec.

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

- [ ] **Step 6: Verify release**

```bash
cd /Volumes/CORSAIR/code/sonzai/sonzai-claude-skill && \
  gh release view v1.7.0 --json url -q .url
```
Expected: prints a GitHub release URL.

---

## Self-Review

**1. Spec coverage:**

| Spec section | Implementing task(s) |
|---|---|
| Summary + goals + non-goals | All tasks |
| Events (cto-loop + full-auto) | Tasks 6 (full-auto), 7+8 (cto-loop) |
| State file schema | Task 7 (Gate A) writes it; Task 8 (Gate B) reads/extends |
| Recipient config | Task 5 (notify-setup variants) |
| MCP detection | Tasks 3+4+5 (subagent prompts + notify-setup) |
| `.mcp.json` for fallbacks | Task 2 |
| Notification body formats | Tasks 6 (full-auto bodies) + 7+8 (cto-loop bodies) |
| Async gate flow | Tasks 7 (Gate A) + 8 (Gate B) |
| Reply parsing grammar | Task 4 (reply-poller) |
| Idempotency | Task 4 (reply-poller mark-as-read + last_seen) |
| Resumability | Tasks 7+8 (24h timeout, state save) |
| Failure modes | Tasks 3+4+5 (all dispatch with graceful degradation) |
| Codex parity | Tasks 11 (.codex-plugin/plugin.json bumps) + 5 (Codex tool prefix detection) |
| Hard rules | Tasks 13 (CLAUDE.md rules 7+8) + per-task internal hard rule sections |
| Out of scope | not implemented (correct) |

All sections covered. No gaps.

**2. Placeholder scan:**

- All "Run X, expected Y" steps have explicit commands.
- All file content is given inline (no "fill in details").
- No "TBD" / "TODO" / "implement later" in the plan body.
- File paths are absolute and exact.

**3. Type consistency:**

- Subagent input shape `{event, run_id, recipient, body, subject, interactive}` consistent across Task 3 (notifier definition) and Tasks 6/7/8 (callers).
- Subagent output shape consistent (notifier returns `{gmail, slack}`; reply-poller returns `{result, source, action, text, new_last_seen}`).
- State file schema same in Task 7 + 8 + 14 (sanity check) + spec.
- Tool prefix patterns (`mcp__claude_ai_gmail__*` etc.) consistent across Tasks 3 / 4 / 5.
- Grammar keywords (`approve` / `reject` / `feedback` / `abort` / `edit` / `one more`) consistent across Task 4 (parser) + Tasks 7/8 (gate handlers).

Plan is internally consistent.

---

## Execution

User pre-decided subagent-driven on main (no worktree). Skip the execution-choice offer; controller dispatches Task 1 first (research), then Tasks 2-5 in parallel where possible, then sequential per the task graph dependencies.
