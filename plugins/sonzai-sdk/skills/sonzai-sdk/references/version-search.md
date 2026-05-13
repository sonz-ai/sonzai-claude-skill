# Always search for current package versions

Short, sharp rule for the wizard and quick-start references: **never recommend a specific package version or install command from training memory.** Verify at write-time.

## Why this matters

Training data lags real-world state by months. Recommending `pip install sonzai==0.5.0` when 1.x is current, or `next@14` when 16 is shipping, or a deprecated CLI flag, sends developers down stale paths. The wizard's whole point is to give them a current, correct starting line.

## What to verify

Whenever the wizard or any reference is about to print a specific version number, install command, or docker image tag, run the right command first:

| Recommending | Command |
|---|---|
| `pip install sonzai` (current version) | `pip index versions sonzai` OR `curl -s https://pypi.org/pypi/sonzai/json \| jq -r .info.version` |
| `npm install @sonzai-labs/agents` (current version) | `npm view @sonzai-labs/agents version` |
| `go get github.com/sonz-ai/sonzai-go@latest` | `go list -m -versions github.com/sonz-ai/sonzai-go` (take last non-rc) |
| Framework getting-started (Next.js install, FastAPI install, etc.) | `WebFetch` the framework's official docs page |
| Docker image tag (`postgres:17`, `redis:7`, etc.) | `curl -s "https://hub.docker.com/v2/repositories/library/<image>/tags?page_size=20"` |

## Where this applies in the wizard

- **`auth-and-setup.md`** — when telling a developer how to install the SDK in their language, verify the current pip/npm/go-get command first
- **`python.md` / `typescript.md` / `go.md` quick-starts** — every code block that shows an install command or a `package.json`/`pyproject.toml`/`go.mod` snippet must have a current version
- **`migration-from-http.md`** — install commands inside the migration recipes
- **Any answer the wizard gives about "what version should I use"** — always search, never recite

## What NOT to do

- ❌ `"sonzai": "^0.3.0"` written from memory because that's what you saw during training
- ❌ `pip install fastapi==0.100.0` because that version sticks in your head
- ❌ Recommending `npm i -g typescript` when the current advice is the package-local install
- ❌ Recommending `postgres:14` because you "remember" 14 is the LTS
- ❌ Skipping the search when you're "confident" — confidence ≠ recency

## Pin + date

Every version-pinned recommendation should carry an "as of <date>" annotation, so a developer reading the wizard output knows when it was last verified. E.g.:

```bash
# As of <today>, the current sonzai SDK is v<X.Y> (verified via `npm view`).
npm install @sonzai-labs/agents
```

If you're writing a long-lived reference file (this skill's `*.md` files), include the verification date in a comment so future readers / contributors know whether to re-verify.

## Hard rules

1. **No version from memory.** Period.
2. **Verify at write-time**, not "from training data".
3. **Pin + date** every version-specific recommendation.
4. **If the registry is unreachable**, halt that recommendation — do NOT fall back to memory.
5. **Prefer major.minor for docker tags**, not `latest` (which moves under you).

## See also

- The full version of this rule (with `version-checker` subagent definition and per-ecosystem details) lives in the autonomous-build skills: `../../full-auto/version-search.md` and `../../cto-loop/` (uses the same).
