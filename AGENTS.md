# AGENTS.md

Entry context for an AI agent working on repo-radar. Read this before changing
anything.

## What this project is

A multi-agent code auditing service. A user submits a public Git repository;
specialised auditors read it in parallel and report typed findings, and each run
is compared against the previous run of the same repository.

Full detail: [README.md](README.md). Architecture and stack:
[DOCS.md](DOCS.md). Where it is going: [ROADMAP.md](ROADMAP.md).

## Before you start

```bash
make install     # dependencies, database, migrations
```

`make` on its own lists every target.

You do **not** need an API key. Tests replay recorded cassettes.

## Definition of done

A change is not finished until all of these pass:

```bash
cd backend
.venv/bin/ruff check . && .venv/bin/ruff format --check .
.venv/bin/mypy app            # strict
.venv/bin/pytest -q --cov=app # 197 passing, 0 skipped, 90% minimum

cd ../frontend
npm run lint && npm run typecheck && npm run test && npm run build
```

Or simply `make check`, which runs all of it.

**Zero skips is the bar.** A skipped evaluation is not a passing evaluation. If
`tests/eval/` starts skipping, a prompt or tool schema changed and cassettes need
re-recording.

## Rules specific to this codebase

**Never make a run look cleaner than it was.** A run where every auditor failed is
`FAILED`, not `COMPLETED` with an empty list. An empty findings list must never be
displayed as "nothing found" unless auditors actually ran. This defect has
appeared twice, in two different layers.

**Money is `Decimal`, never `float`.** Costs are summed across thousands of calls
and compared against a ceiling.

**An unpriced model raises.** Do not add a zero-cost fallback: it would let a run
bypass its budget.

**The repository URL is untrusted input.** Do not relax anything in `cloner.py`
without understanding what it blocks. Each limit has a test explaining the attack.

**The API key is the caller's.** It must never reach the database, a log line, an
error message, or a response. `tests/test_byok.py` and
`tests/test_settings_secrets.py` assert this; do not weaken them.

**Findings are matched on category and file path, never wording.** The model
rephrases everything on every run. This applies to both the evaluation and the
diff, and they must agree.

**Fixtures under `tests/fixtures/` contain deliberately bad code.** They are input
data, not tests. They are excluded from lint and from pytest collection. Do not
"fix" them.

## Adding things

| To add | Do this |
| --- | --- |
| An auditor | A file in `app/auditors/` with a name, system prompt and tool description. Then a fixture, a `golden.json` entry, a `CASES` entry, and recorded cassettes. |
| A provider | A file in `app/llm/providers/` satisfying `LLMClient`, a prefix in `_KEY_PREFIXES`, a default model, and prices in `usage.py`. Copy an existing adapter. |
| An endpoint | Router stays thin. Business logic goes in `services/`, queries in `repositories/`. |

## Where things live

```
app/routers/      HTTP and WebSocket handlers, no business logic
app/services/     business logic
app/repositories/ database queries
app/llm/          provider-agnostic client, cassettes, cost
app/auditors/     one module per auditor
```

## Traps that have already cost time

- **`params` in Next.js 15 is a promise**, unwrapped with `use()`. It was a plain
  object in 14. Getting this wrong is invisible to typecheck, which simply
  believes the annotation, and then fails on every request.
- **Tailwind strips interpolated class names.** `text-severity-${x}` renders
  unstyled. Use an explicit map.
- **Do not run `next build` while `next dev` is running.** Both write to `.next/`
  and the dev server breaks with "Cannot find module './958.js'". Fix:
  `rm -rf .next`.
- **Pinned Gemini model names 404 on free-tier keys** even though ListModels
  returns them. Use the `-latest` aliases.
- **A freshly created SQLAlchemy row has no loaded relationships.** Serialising
  one triggers lazy IO from sync code and raises `MissingGreenlet`. Use
  `set_committed_value`.

## Commit style

Conventional Commits, one logical change per commit, per `rules/RULES_GIT.md`.
Explain **why** in the body, not what — the diff already says what.
