# Code conventions for FastAPI services built with Claude

This file is the reusable rule set for any FastAPI microservice scaffolded from this skeleton.
It was distilled from two real services built this way: `subtitle-service` and
`news-script-generation-service` (both siblings of this repo's parent monorepo). Keep it short
and concrete — a rule Claude can actually check itself against, not a style essay.

## Project layout

Modular from the start — a **package per concern**, not a flat file per concern, even while a
concern is still small. A one-function `auth.py` today is how you end up with a 400-line
`auth.py` in six months because there was never a natural place to split it. Start with the
package; let it hold one file until it genuinely needs more.

```
app/
  __init__.py
  main.py              # FastAPI() instance, router registration, middleware registration — nothing else
  core/
    __init__.py
    config.py          # every env var read, in one place (see Configuration below)
  auth/
    __init__.py
    api_key.py         # the APIKeyHeader dependency (see Auth below)
  middleware/
    __init__.py
    <name>.py          # one ASGI/HTTP middleware per file (request logging, error normalization, …)
  schemas/
    __init__.py
    <domain>.py        # Pydantic request/response/domain models, grouped by domain not dumped in one file
  db/
    __init__.py
    session.py         # SQLAlchemy engine/session setup
    models.py          # SQLAlchemy table definitions (see Database below)
  <feature>/            # one package per real unit of logic (e.g. prompts/, gemini_client/) —
    __init__.py         # never a god file; split wording/config from assembly logic from I/O
alembic/
  env.py, versions/
alembic.ini
requirements.txt     # pinned ranges with a comment explaining any non-obvious pin
pyproject.toml       # just [tool.pytest.ini_options] unless the project needs more
Dockerfile
.env.example
tests/
  conftest.py    # set required env vars BEFORE anything imports app.main (see Testing)
  test_*.py
```

A route handler in `main.py` should be a thin wrapper that calls into a named function imported
from the relevant package — never grow `main.py` into a dump of logic, config, or middleware
definitions. Within any package, split further by file the moment a second distinct thing shows
up (e.g. `auth/api_key.py` plus a later `auth/jwt.py`, not both crammed into `auth/api_key.py`).

## Auth

Every service is a private, same-host/trusted-network dependency (one app's backend calling
another over localhost or an internal network) — not a general-purpose auth system. Use a single
shared-secret header (`X-API-Key` via `fastapi.security.APIKeyHeader`), checked with
`secrets.compare_digest` (constant-time — a plain `!=` leaks timing information), defined in
`app/auth/api_key.py`.

- Read the expected key **at module import time**, not per-request, and **raise if unset** — a
  missing secret must fail the process at boot, never silently serve unauthenticated.
- `GET /health` is the one open, unauthenticated route — infra liveness probes shouldn't need a
  secret just to ask "is the process up."
- Every other route requires the header via a router-level `Depends(require_api_key)`, not a
  per-route decorator easy to forget on a new endpoint.
- If a second auth scheme is ever needed (e.g. a JWT path for a different caller), it gets its own
  file in `app/auth/` (`app/auth/jwt.py`) — never branches inside `api_key.py`.

## Middleware

`app/middleware/` holds cross-cutting ASGI/HTTP middleware — request/response logging, error
normalization into a consistent JSON error shape, request-ID propagation, etc. One file per
distinct middleware, registered explicitly in `main.py` (`app.add_middleware(...)`) rather than
auto-discovered — an explicit, visible list of what wraps every request. Don't add a middleware
for something a single route's own `Depends` can handle just as well (e.g. auth stays in
`app/auth/`, not middleware, since it's not cross-cutting to every route — `/health` skips it).

## Configuration

One `app/core/config.py`, one function per env var (not a module-level constant grabbed at import
time, unless `auth/api_key.py`'s "fail at boot" reasoning specifically applies). Every var gets:
- A sane, explicit default where one exists, or a loud `RuntimeError` where it doesn't.
- A line in `.env.example` with a one-sentence comment explaining *why* it exists or what happens
  if it's left at its default — not just the var name.
- A naming convention of `<SERVICE_NAME>_<THING>` for anything the *calling* service configures
  about reaching this one (base URL, API key, timeout) — e.g. `SCRIPT_GENERATION_SERVICE_BASE_URL`
  — so a caller's own `.env.example` reads clearly next to its other sibling-service vars.

## Contracts with a sibling service in another language

When this service's request/response shape mirrors a type defined in another language (e.g. a
TypeScript zod schema in a Node monolith this service talks to), there is no shared-type system
across that boundary — hand-mirror the fields as Pydantic models, read from the actual source
schema, not from memory or a stale copy. Keep both sides independently defensive:
- This service's Pydantic models reject a malformed request at its own boundary.
- The **calling** side should still re-validate the response it gets back against its own schema,
  rather than trusting this service's output blindly — a contract drift should fail loudly at the
  caller, not propagate a malformed object downstream.
- Document in a comment exactly which upstream file/line the mirrored shape was copied from, so a
  future change to the source is traceable instead of silently drifting.

## Prompt-building services (LLM wording)

If this service's job is to build a prompt and call an LLM, split **wording** from **assembly**
from **transport**, in three separate packages (each following the "one file until it needs more"
rule from Project layout above):
- `app/prompts/` — plain string constants, the actual instruction text. Nothing else lives here.
  This is the one package a non-engineer or a prompt-focused reviewer should be able to read top
  to bottom and understand the model's full instructions.
- `app/prompt_builder/` — pure functions that assemble those constants plus request-specific data
  into the final prompt string. No I/O, no network calls, trivially unit-testable.
- `app/<provider>_client/` (e.g. `app/gemini_client/`) — the actual HTTP call to the LLM provider,
  response parsing, and any defensive clamping of the output (e.g. a provider's structured-output
  feature enforcing shape but not string-length caps).
A wording change should be a diff inside `app/prompts/` alone. If the project later needs multiple
prompt versions live at once, version by filename inside that package (`prompts/grounded_v2.py`)
once that need is real — don't build a version-switching abstraction speculatively before anything
needs to switch.

## HTTP calls to other services

Decide **fail-hard vs. fail-soft** deliberately per call, and say so in a comment:
- **Fail-hard** (raise/propagate) when the result is not optional — the caller's own job can't
  complete meaningfully without it. Don't swallow the error into a default value.
- **Fail-soft** (catch, log, return `None`/a fallback) only for a genuinely best-effort, secondary
  feature where degrading gracefully is the correct behavior, not a shortcut to avoid handling an
  error path.
Never mix the two in the same function without a comment explaining which call is which and why.

## Database

If this service shares a physical database with another service that already owns tables there
via a different migration tool (e.g. Prisma on a Node side), **never** let this service's Alembic
chain manage a table it doesn't own — two migration tools both believing they control one table's
DDL is a guaranteed eventual conflict. Default to:
- This service's own tables only, under their own Alembic chain, ideally in their own dedicated
  Postgres schema (not the other system's default schema) so ownership is visible just from the
  namespace.
- If this service genuinely must read or write a table owned elsewhere, do it via a plain
  SQLAlchemy Core `Table()` reflecting that exact DDL (case-sensitive identifiers included — check
  whether the owning ORM uses `@map`/snake_case or writes exact-case identifiers before assuming
  either), and never generate an Alembic migration against it.
- Don't apply a migration against a shared/production database as a side effect of scaffolding a
  new service — write and review the migration file first; running it against anything real is a
  separate, deliberate step.

## Testing

`tests/conftest.py` sets every required env var (`os.environ.setdefault(...)`, never overwrite a
real value a developer set) **before** any test module imports `app.main` — auth.py-style
import-time validation means import order matters; conftest.py is collected first by pytest, which
is what makes this safe. Use `httpx`'s test client against the real FastAPI app, not a hand-rolled
mock of it. Stub the outbound LLM/provider call at the HTTP layer (so the test exercises real
request/response parsing) rather than mocking your own wrapper function.

Default to the minimum test suite that actually proves the service works end to end — don't add a
parity test against another implementation, a full unit-test matrix, or multi-version regression
tests speculatively. Add those when a real need appears (e.g. porting logic from another language
and needing to prove behavioral equivalence before cutover).

## Docker

`python:3.11-slim` base. `COPY requirements.txt .` and `pip install` **before** `COPY app ./app` —
preserves Docker's layer cache so a code-only change doesn't reinstall dependencies. Never bake a
secret into the image via `ENV` in the Dockerfile; pass it at `docker run -e ...` time instead, and
say so in a comment next to any `ENV` block that sets non-secret defaults.

## API versioning

Don't add a `/v1/` prefix or any version-negotiation scheme until there's a second version that
actually needs to coexist with the first. A single current contract, documented in `schemas.py`
and this service's own README, is simpler to reason about and faster to iterate on — the explicit
reason this skeleton exists.

## Git commits

Every commit's author/committer is the repo owner's own personal git identity (whatever
`user.name`/`user.email` is already configured locally) — never Claude, never an AI identity, and
never a `Co-authored-by: Claude`/similar trailer. Don't change git config to add or switch to a
bot/AI identity for any commit in this project. Claude may stage changes and write the commit
message, but the human's own account is the sole author of record.

## General style

- No comments that restate what the code already says. A comment earns its place only by
  explaining a non-obvious *why* — a constraint, a past incident, a deliberate trade-off.
- No speculative abstraction for a need that doesn't exist yet (a plugin system, a config format,
  a generic base class) — three similar lines beat a premature abstraction.
- Keep `main.py` route handlers one or two lines: parse/validate input, call the real function,
  return its result. Business logic lives in the named feature files, not inline in a route.
