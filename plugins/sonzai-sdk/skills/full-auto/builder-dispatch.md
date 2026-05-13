# Builder dispatch (Phase 2)

After masterplan-assembly writes the masterplan doc, dispatch a builder subagent to execute it. Use the `Agent` tool with `subagent_type: general-purpose`.

## When to dispatch

- **`full-auto` mode:** directly after `masterplan-assembly.md` finishes writing the file.
- **`cto-loop` mode:** after `../cto-loop/masterplan-gate.md` returns `approve`.

The dispatch logic below is identical in both modes.

## Inputs to provide

The subagent has no context from this conversation. Provide everything explicitly:

1. **Masterplan path** — absolute path to `docs/cto-review/<date>-masterplan.md`. Subagent must READ this file first.
2. **System prompt** — content of `subagent-prompts/builder.md` (the builder rules)
3. **Operator's CWD** — where the builder writes
4. **Always-search rule** — re-state it inline in the dispatch prompt (belt + suspenders — the system prompt also has it)

## Dispatch prompt template

```
You are a builder subagent for `cto-loop`. Implement the project described in:

  ${MASTERPLAN_PATH}

You MUST:
1. Read the masterplan file in full FIRST, before any edits.
2. Follow your system prompt rules (see `subagent-prompts/builder.md`).
3. CRITICAL: never write a package version, install command, or docker image tag from your training memory.
   Before writing ANY manifest (package.json, requirements.txt, go.mod, Dockerfile, docker-compose.yml),
   run the version-checker subagent (see `subagent-prompts/version-checker.md`) to get current values.
4. Commit each logical unit of work as you go (`git add` + `git commit -m`).
   - Conventional commit prefix: `feat:` / `fix:` / `chore:` / `docs:`
   - Co-author line: `Co-Authored-By: Claude Code (cto-loop) <noreply@anthropic.com>`
5. Do NOT `git push`. Do NOT deploy to remote infra. Local commits + local files only.
6. Return when done. Reply with:
   ```
   {
     "status": "done" | "blocked" | "needs_context",
     "commits": ["sha1", "sha2", ...],
     "entry_files": ["src/server.ts", "docker-compose.yml", ...],
     "smoke_command": "docker compose up -d --wait",
     "open_questions": [...],
     "blocker": "<if blocked, what stopped you>"
   }
   ```

Operator's CWD: ${OPERATOR_CWD}
Project type: ${PROJECT_TYPE}  <!-- greenfield | brownfield -->

If brownfield, ALSO read the audit context:
  ${BROWNFIELD_CONTEXT_PATH}
And follow Hard Rule 7: integrate, don't recreate.

Now: read the masterplan and begin.
```

## Handling builder return states

- **`done`** → proceed to `local-deploy.md` (Phase 3). Capture the JSON output to in-memory state.
- **`blocked`** → print the blocker to operator. Ask: "Builder blocked: <blocker>. Reply with `unblock <txt>` (I'll re-dispatch with this context) or `abort`." Bound at 3 unblock cycles.
- **`needs_context`** → builder hit something not specified in the masterplan. Print the missing-context message. Either auto-fill from in-memory state (preferred) or ask operator if it's a real ambiguity. Re-dispatch.

## Model choice for the builder

Use a capable model (Sonnet or Opus). The builder writes code across multiple files; this is not a haiku-class task.

```
Agent({
  description: "Builder for cto-loop project",
  subagent_type: "general-purpose",
  model: "sonnet",   // or "opus" for complex multi-service projects
  prompt: <the dispatch prompt above>
})
```

(Per the always-search rule, you may want to bump to opus if the masterplan involves a stack you're less confident about — the model upgrade reduces hallucination risk further, in addition to version-checker.)

## Brownfield handling

When `project_type == brownfield`, the dispatch prompt MUST include:

```
You are integrating into an existing repo. Read every file the masterplan's "file structure" section marks as MODIFY *before* changing it. Do NOT delete or rename files the operator didn't explicitly approve. If a section of the masterplan conflicts with what's actually in the repo (e.g., schema already exists), prefer the existing repo and flag in `open_questions`.
```

This prevents the builder from steamrolling existing patterns.

## Hard rules

1. **One builder subagent per Phase 2 invocation.** No parallel implementers — they conflict on file writes.
2. **Always-search rule propagates.** Builder must invoke version-checker before any manifest write — this is enforced via the system prompt AND repeated inline in the dispatch.
3. **No remote git ops.** Builder commits locally; never pushes.
4. **Builder cannot dispatch sub-subagents** for general work — only `version-checker` for version queries. Keeps the dispatch tree shallow.
5. **Builder return is JSON or escalation.** If it returns a freeform message instead of the JSON shape, re-dispatch with "your previous reply didn't follow the return format; please return JSON only."

## Handoff

On `status: done`, save the builder JSON to in-memory state and read `local-deploy.md` next.
