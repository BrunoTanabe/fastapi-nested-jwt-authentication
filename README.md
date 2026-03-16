# FastAPI Nested JWT Authentication

English version. For Portuguese (Brazil), see `README-PTBR.md`.

Python backend project built with FastAPI, focused on secure authentication using **Nested JWT (JWS + JWE)**, HttpOnly cookies, token rotation, logout revocation, and a modular structure inspired by **Clean Architecture + DDD**.

This repository is intended to be a practical base for modern API authentication with clear separation of concerns and easy long-term maintenance.

## Summary

- [Overview](#overview)
- [Architecture](#architecture)
- [Project Structure](#project-structure)
- [Stack and Dependencies](#stack-and-dependencies)
- [Environment Configuration](#environment-configuration)
- [Local Run (UV)](#local-run-uv)
- [Run with Docker](#run-with-docker)
- [Makefile Commands](#makefile-commands)
- [Database Migrations](#database-migrations)
- [Authentication Flow](#authentication-flow)
- [Main Endpoints](#main-endpoints)
- [Best Practices for Evolution](#best-practices-for-evolution)
- [Tests](#tests)

## Overview

This project implements an authentication flow with production-oriented security:

- **Access token and refresh token as Nested JWT** (signed then encrypted).
- **HttpOnly cookies** for token transport.
- **Session binding by device + user-agent**, validated against database state.
- **Token rotation** in refresh endpoint.
- **Refresh/access token revocation** on logout.
- **Standardized response and exception handling** through global middleware/handlers.

## Architecture

Code is organized by modules under `app/modules`, each with sublayers:

- `domain/`: entities, value objects, and pure business rules.
- `application/`: use cases and interfaces (ports).
- `infrastructure/`: repositories and SQLAlchemy models.
- `presentation/`: FastAPI routers, schemas, and OpenAPI docs.

Cross-cutting components live in `app/core/`:

- `settings.py`: configuration via `pydantic-settings`.
- `database.py`: sync/async SQLAlchemy engines and sessions.
- `security.py`: password hashing, JWT generation/validation, role authorization.
- `middleware.py`: request logging, response envelope, and device cookie middleware.
- `migrations.py`: migration status check and auto-upgrade on startup.
- `key_management.py`: automatic RSA key generation if missing.
- `resources.py`: application lifespan (startup/shutdown).

## Project Structure

Short structure overview:

```text
fastapi-nested-jwt-authentication/
├── app/
│   ├── app.py
│   ├── core/
│   └── modules/
│       ├── authentication/
│       ├── user/
│       ├── health/
│       ├── example/
│       ├── shared/
│       └── blank/
├── migrations/
├── scripts/
├── secrets/keys/
├── test/
├── docker-compose.yaml
├── Dockerfile
├── pyproject.toml
├── requirements.txt
└── alembic.ini
```

## Stack and Dependencies

Main dependencies declared in `pyproject.toml`:

- `fastapi[standard]`
- `sqlalchemy` + `alembic`
- `asyncpg` / `psycopg`
- `pydantic` + `pydantic-settings`
- `jwcrypto`
- `pwdlib[argon2]`
- `cryptography`
- `orjson`
- `loguru`
- `hypercorn`

Current repository notes:

- Local environment can be managed with **uv** (`uv.lock` is present).
- Current `Dockerfile` installs dependencies via `requirements.txt`.
- `.python-version` and `Dockerfile` currently target **Python 3.14**.

## Environment Configuration

1. Copy the example file:

```bash
cp .env.example .env
```

2. Fill required variables in `.env`, especially:

- Database: `POSTGRESQL_DATABASE`, `POSTGRESQL_USERNAME`, `POSTGRESQL_PASSWORD`, `POSTGRESQL_HOST`, `POSTGRESQL_PORT`
- JWT/keys: `JWT_ISSUER`, `JWT_AUDIENCE`, `JWT_SIGNING_KEY_PASSWORD`, `JWT_ENCRYPTION_KEY_PASSWORD`, `JWT_HASH_FINGERPRINT`
- Cookies: `COOKIES_TOKEN_TYPE_KEY`, `COOKIES_ACCESS_TOKEN_KEY`, `COOKIES_ACCESS_TOKEN_PATH`, `COOKIES_REFRESH_TOKEN_KEY`, `COOKIES_REFRESH_TOKEN_PATH`, `COOKIES_DEVICE_KEY`, `COOKIES_DOMAIN`
- Admin seed: `SECURITY_ADMIN_EMAIL`, `SECURITY_ADMIN_PASSWORD`
- CORS/security: `SECURITY_ALLOW_ORIGINS`, `SECURITY_ALLOW_HEADERS`, `SECURITY_ALLOW_METHODS`, `SECURITY_EMAIL_ALLOWED_DOMAINS`

3. RSA keys:

- API startup tries to auto-generate keys under `secrets/keys/` when they do not exist.
- If preferred, generate them manually following `secrets/keys/README.md`.

## Local Run (UV)

With `uv` installed:

```bash
uv sync
uv run -- uvicorn app.app:app --reload --host 0.0.0.0 --port 8000
```

Interactive docs:

- `http://localhost:8000/docs`
- `http://localhost:8000/redoc`

Note: in `production` environment, OpenAPI/Swagger endpoints are disabled.

## Run with Docker

Start API + PostgreSQL + pgAdmin via compose:

```bash
docker compose up --build
```

Or using `Makefile` targets:

```bash
make start
```

## Makefile Commands

Available commands in `Makefile`:

- `make start`: start stack with rebuild.
- `make start-silent`: start stack in background.
- `make view-processes`: list containers.
- `make delete`: stop stack and remove volumes/containers.
- `make dependencies-up`: start only `database` and `database-admin`.
- `make dependencies-up-silent`: same as above, in background.
- `make dependencies-down`: stop only dependency services.

## Database Migrations

Project uses Alembic and, on startup, attempts to auto-apply pending migrations.

Useful manual commands:

```bash
alembic revision --autogenerate -m "migration_description"
alembic upgrade head
alembic current
alembic downgrade -1
```

References:

- `migrations/README.md`
- `app/core/migrations.py`

## Authentication Flow

### Flow summary

1. User logs in with `username/password` (OAuth2 form).
2. API validates credentials, generates nested JWT access/refresh tokens, and stores JTI hashes in database.
3. Tokens are set in HttpOnly cookies.
4. Authenticated requests validate token + device cookie + user-agent + revocation state in database.
5. Refresh rotates tokens and updates hashes.
6. Logout revokes refresh/access token state for the current session.

### Cookies

Cookie keys are configured via `COOKIES_*` environment variables.
A middleware also guarantees a device cookie (`COOKIES_DEVICE_KEY`) to bind session to client context.

### Claims and security

- Validation of `iss`, `aud`, `exp`, `nbf`, `jti`, `scope`.
- RSA signature (`RS256`) + encryption (`RSA-OAEP-256` + `A256GCM`).
- Password hashing with Argon2 (`pwdlib`).
- Token identifier hashing using HMAC-SHA256 (`JWT_HASH_FINGERPRINT`).

## Main Endpoints

Public routes:

- `GET /` -> redirects to `/docs`
- `GET /health`
- `POST /api/v1/user` -> create user
- `POST /api/v1/authentication/login` -> login
- `POST /api/v1/example`

Authenticated routes:

- `GET /api/v1/user/me` -> current user
- `PATCH /api/v1/authentication/refresh` -> rotate/renew tokens
- `DELETE /api/v1/authentication/logout` -> logout (session revocation)
- `GET /api/v1/alembic-version` -> admin only

Example: create user

```bash
curl -X POST "http://localhost:8000/api/v1/user" \
  -H "Content-Type: application/json" \
  -d '{
    "first_name": "John",
    "last_name": "Doe",
    "preferred_name": "Joe",
    "gender": "male",
    "birthdate": "1995-01-01",
    "email": "john@example.com",
    "phone": "+5511999999999",
    "password": "StrongP@ssw0rd!"
  }'
```

Example: login (form-urlencoded)

```bash
curl -X POST "http://localhost:8000/api/v1/authentication/login" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=john@example.com&password=StrongP@ssw0rd!" \
  -c cookies.txt
```

Example: authenticated endpoint using cookies

```bash
curl -X GET "http://localhost:8000/api/v1/user/me" \
  -b cookies.txt
```

## Best Practices for Evolution

- Keep strict layer separation (Presentation -> Application -> Domain, Infra implements ports).
- Avoid business logic in `routers.py`; keep it in `use_cases.py` and domain layer.
- Keep strong typing in functions and models.
- When creating a new module, replicate `domain/application/infrastructure/presentation`.
- Document new endpoints in each module `docs.py`.

## Tests

The `test/` folder already exists with per-module scaffolding (`test/modules/...`).
As the project evolves, prioritize tests for:

- `authentication` and `user` use cases;
- refresh/logout and revocation flows;
- role-based authorization (`user`, `manager`, `admin`);
- main API endpoints with an HTTP test client.

