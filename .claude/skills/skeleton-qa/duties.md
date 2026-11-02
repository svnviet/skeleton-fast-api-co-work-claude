# QA duties

Add a row to extend QA. `Rule` = the CLAUDE.md section it enforces.

## Layers & architecture

| ID | Rule | Verify | Severity |
|---|---|---|---|
| L1 | Project layout | Each concern is a package with `__init__.py`; no flat `auth.py`/`config.py` at `app/` root | Major |
| L2 | Project layout | `main.py` only creates the app, registers routers/middleware; no logic or config | Major |
| L3 | Project layout | Route handlers are 1-2 lines and call a service method | Major |
| L4 | Classes & OOP | Logic lives in `*Service`/`*Repository`/`*Client`/`*Builder` classes with `__init__` injection; no module-level mutable state | Major |
| L5 | Classes & OOP | Layers pass Pydantic/SQLAlchemy models, not raw dicts; no business logic on schemas | Major |
| L6 | Prompt-building | Wording only in `app/prompts/`, assembly in `prompt_builder/`, HTTP in `<provider>_client/` | Major |
| L7 | Middleware | Each middleware in its own file, registered explicitly in `main.py` | Minor |
| L8 | API versioning | No `/v1/` prefix without a coexisting second version | Minor |

## Security

| ID | Rule | Verify | Severity |
|---|---|---|---|
| S1 | Auth | `app/auth/api_key.py` reads the key at import time and raises if unset | Blocker |
| S2 | Auth | Key compared with `secrets.compare_digest`, never `==`/`!=` | Blocker |
| S3 | Auth | Only `GET /health` is unauthenticated; all other routes covered by router-level `Depends(require_api_key)` | Blocker |
| S4 | Docker | No secret in `ENV`/`ARG`/`COPY` in the Dockerfile; no `.env` committed or copied into image | Blocker |
| S5 | Database | No Alembic migration for tables the service doesn't own; no migration applied by scaffolding | Blocker |
| S6 | HTTP calls | Each outbound call states fail-hard vs fail-soft in a comment; no silent swallow of a required call | Major |
| S7 | Contracts | Mirrored upstream shapes cite their source file; caller re-validates responses | Minor |

## Config, tests, tooling

| ID | Rule | Verify | Severity |
|---|---|---|---|
| C1 | Configuration | Every env var read only in `app/core/config.py`; each has an `.env.example` line with a why-comment | Minor |
| C2 | Testing | `tests/conftest.py` uses `os.environ.setdefault` before importing `app.main`; httpx against the real app; provider stubbed at HTTP layer | Major |
| C3 | Docker | `python:3.11-slim`; `requirements.txt` installed before `COPY app` | Minor |
| C4 | Linting | `ruff` pinned and configured in `pyproject.toml`; `ruff check` and `ruff format --check` pass | Minor |
| C5 | Git commits | Commits authored by the human's identity; no AI `Co-authored-by` trailer | Minor |

## Added later

<!-- New duties go here. Keep: ID, Rule, Verify, Severity. -->
