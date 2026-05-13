# Project type detection (Phase 0b)

After transcript analysis, decide: **greenfield** (operator's CWD will become a brand-new project) or **brownfield** (operator's CWD already contains a codebase that the new functionality must integrate into).

## Detection

Run all of these checks on the operator's CWD. Each "yes" is a brownfield signal.

| Signal | Command / check |
|---|---|
| Git repo? | `test -d .git` or `git rev-parse --is-inside-work-tree` |
| Has commits? | `git log -1 --oneline 2>/dev/null` returns a line |
| Has source files? | `find . -maxdepth 3 -type f \( -name '*.ts' -o -name '*.tsx' -o -name '*.js' -o -name '*.py' -o -name '*.go' -o -name '*.rs' -o -name '*.java' -o -name '*.rb' \) | head -1` returns anything |
| Has manifest? | `test -f package.json -o -f requirements.txt -o -f pyproject.toml -o -f go.mod -o -f Cargo.toml -o -f Gemfile -o -f pom.xml` |
| Has compose? | `test -f docker-compose.yml -o -f compose.yml -o -f Dockerfile` |

**Decision:**
- **All signals fail** → greenfield (CWD is empty or `git init`-only)
- **One or more signals pass** → brownfield (CWD has prior work)

Edge cases:
- Empty `git init`-only dir → greenfield
- Dir with only `README.md` and `LICENSE` → greenfield (counts as "scaffolded but no code")
- Monorepo with multiple packages → brownfield (proceed with full audit, the audit handles workspace detection)
- Dir with `.git/` from a prior failed clone but no other files → greenfield (warn operator we'll write over)

## Handoff

- **Greenfield** → read `tech-stack-intake.md` next (Phase 0c-G)
- **Brownfield** → read `brownfield-audit.md` next (Phase 0c-B)

## Pre-empting the audit when transcript explicitly says greenfield

If the transcript explicitly states "greenfield project", "new repo", "from scratch", and the CWD has prior signals, **ask the operator** before deciding:

> "I detected existing files in this directory (`package.json`, `src/`, etc.) but the transcript says greenfield. Which is right?
> - `brownfield` — integrate into what's here (recommended)
> - `greenfield` — wipe and start fresh (DANGER: I will not auto-delete; you do it first)
> - `abort` — let me re-orient"

Never auto-wipe a directory. If operator wants greenfield in a non-empty dir, ask them to run `cd ..` to a fresh path and re-invoke.
