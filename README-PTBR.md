# FastAPI Nested JWT Authentication

Versao em portugues (Brasil). For English, see `README.md`.

Projeto backend em Python com FastAPI focado em autenticacao segura usando **Nested JWT (JWS + JWE)**, cookies HttpOnly, rotacao de tokens, revogacao em logout e organizacao por modulos com abordagem inspirada em **Clean Architecture + DDD**.

O objetivo deste repositorio e servir como base real para autenticacao moderna em APIs, mantendo separacao de responsabilidades e facilidade de evolucao.

## Sumario

- [Visao Geral](#visao-geral)
- [Arquitetura](#arquitetura)
- [Estrutura do Projeto](#estrutura-do-projeto)
- [Stack e Dependencias](#stack-e-dependencias)
- [Configuracao de Ambiente](#configuracao-de-ambiente)
- [Execucao Local (UV)](#execucao-local-uv)
- [Execucao com Docker](#execucao-com-docker)
- [Comandos Makefile](#comandos-makefile)
- [Migracoes de Banco](#migracoes-de-banco)
- [Fluxo de Autenticacao](#fluxo-de-autenticacao)
- [Endpoints Principais](#endpoints-principais)
- [Boas Praticas para Evolucao](#boas-praticas-para-evolucao)
- [Testes](#testes)

## Visao Geral

Este projeto implementa um fluxo de autenticacao com foco em seguranca de producao:

- **Access token e refresh token em formato Nested JWT** (token assinado e depois criptografado).
- **Cookies HttpOnly** para transporte dos tokens.
- **Vinculo de sessao por device + user-agent**, com validacao contra banco.
- **Rotacao de token** no endpoint de refresh.
- **Revogacao de refresh/access token** no logout.
- **Tratamento padronizado de resposta e erros** via middlewares/handlers globais.

## Arquitetura

O codigo esta organizado por modulos em `app/modules`, cada um com subcamadas:

- `domain/`: entidades, value objects e regras de negocio puras.
- `application/`: casos de uso e interfaces (ports).
- `infrastructure/`: repositorios e modelos SQLAlchemy.
- `presentation/`: routers FastAPI, schemas e docs OpenAPI.

No `app/core/` ficam componentes transversais:

- `settings.py`: configuracoes via `pydantic-settings`.
- `database.py`: engines sync/async e sessoes SQLAlchemy.
- `security.py`: hash de senha, emissao/validacao de JWT, autorizacao por papel.
- `middleware.py`: logging de request, envelope de resposta e cookie de device.
- `migrations.py`: verificacao/aplicacao automatica de migracoes no startup.
- `key_management.py`: geracao automatica de chaves RSA caso nao existam.
- `resources.py`: ciclo de vida da aplicacao (startup/shutdown).

## Estrutura do Projeto

Estrutura resumida:

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

## Stack e Dependencias

Dependencias principais declaradas em `pyproject.toml`:

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

Notas importantes do estado atual do repositorio:

- O ambiente local pode ser gerenciado por **uv** (`uv.lock` presente).
- O `Dockerfile` atual instala dependencias por `requirements.txt`.
- `.python-version` e `Dockerfile` apontam para **Python 3.14**.

## Configuracao de Ambiente

1. Copie o arquivo de exemplo:

```bash
cp .env.example .env
```

2. Preencha os campos obrigatorios no `.env`, principalmente:

- Banco: `POSTGRESQL_DATABASE`, `POSTGRESQL_USERNAME`, `POSTGRESQL_PASSWORD`, `POSTGRESQL_HOST`, `POSTGRESQL_PORT`
- JWT/chaves: `JWT_ISSUER`, `JWT_AUDIENCE`, `JWT_SIGNING_KEY_PASSWORD`, `JWT_ENCRYPTION_KEY_PASSWORD`, `JWT_HASH_FINGERPRINT`
- Cookies: `COOKIES_TOKEN_TYPE_KEY`, `COOKIES_ACCESS_TOKEN_KEY`, `COOKIES_ACCESS_TOKEN_PATH`, `COOKIES_REFRESH_TOKEN_KEY`, `COOKIES_REFRESH_TOKEN_PATH`, `COOKIES_DEVICE_KEY`, `COOKIES_DOMAIN`
- Admin seed: `SECURITY_ADMIN_EMAIL`, `SECURITY_ADMIN_PASSWORD`
- CORS/seguranca: `SECURITY_ALLOW_ORIGINS`, `SECURITY_ALLOW_HEADERS`, `SECURITY_ALLOW_METHODS`, `SECURITY_EMAIL_ALLOWED_DOMAINS`

3. Chaves RSA:

- O startup da API tenta gerar automaticamente as chaves em `secrets/keys/` se nao existirem.
- Se preferir, voce pode gerar manualmente seguindo `secrets/keys/README.md`.

## Execucao Local (UV)

Com o `uv` instalado:

```bash
uv sync
uv run -- uvicorn app.app:app --reload --host 0.0.0.0 --port 8000
```

Documentacao interativa:

- `http://localhost:8000/docs`
- `http://localhost:8000/redoc`

Observacao: em ambiente `production`, o projeto desabilita OpenAPI/Swagger.

## Execucao com Docker

Subindo API + PostgreSQL + pgAdmin com compose:

```bash
docker compose up --build
```

Ou usando os alvos do `Makefile`:

```bash
make start
```

## Comandos Makefile

Comandos disponiveis em `Makefile`:

- `make start`: sobe stack com rebuild.
- `make start-silent`: sobe stack em background.
- `make view-processes`: lista containers.
- `make delete`: derruba stack e remove volumes/containers.
- `make dependencies-up`: sobe apenas `database` e `database-admin`.
- `make dependencies-up-silent`: igual ao anterior, em background.
- `make dependencies-down`: derruba somente dependencias.

## Migracoes de Banco

O projeto usa Alembic e, no startup, tenta aplicar migracoes pendentes automaticamente.

Comandos manuais uteis:

```bash
alembic revision --autogenerate -m "descricao_da_migracao"
alembic upgrade head
alembic current
alembic downgrade -1
```

Referencias:

- `migrations/README.md`
- `app/core/migrations.py`

## Fluxo de Autenticacao

### Resumo do fluxo

1. Usuario faz login com `username/password` (OAuth2 form).
2. API valida credenciais, gera access/refresh token nested JWT e salva hashes de JTI no banco.
3. Tokens sao enviados em cookies HttpOnly.
4. Requests autenticadas validam token + device cookie + user-agent + estado de revogacao no banco.
5. Refresh rotaciona tokens e atualiza hashes.
6. Logout revoga refresh/access token da sessao.

### Cookies

As chaves dos cookies sao definidas por variaveis de ambiente (`COOKIES_*`).
Um middleware tambem garante o cookie de dispositivo (`COOKIES_DEVICE_KEY`) para amarrar a sessao ao cliente.

### Claims e seguranca

- Validacao de `iss`, `aud`, `exp`, `nbf`, `jti`, `scope`.
- Assinatura RSA (`RS256`) + criptografia (`RSA-OAEP-256` + `A256GCM`).
- Hash de senha com Argon2 (`pwdlib`).
- Hash de identificadores de token com HMAC-SHA256 (`JWT_HASH_FINGERPRINT`).

## Endpoints Principais

Rotas publicas:

- `GET /` -> redireciona para `/docs`
- `GET /health`
- `POST /api/v1/user` -> cria usuario
- `POST /api/v1/authentication/login` -> login
- `POST /api/v1/example`

Rotas autenticadas:

- `GET /api/v1/user/me` -> usuario logado
- `PATCH /api/v1/authentication/refresh` -> renova tokens
- `DELETE /api/v1/authentication/logout` -> logout (revogacao de sessao)
- `GET /api/v1/alembic-version` -> apenas admin

Exemplo de criacao de usuario:

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

Exemplo de login (form-urlencoded):

```bash
curl -X POST "http://localhost:8000/api/v1/authentication/login" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "username=john@example.com&password=StrongP@ssw0rd!" \
  -c cookies.txt
```

Exemplo de endpoint autenticado com cookies:

```bash
curl -X GET "http://localhost:8000/api/v1/user/me" \
  -b cookies.txt
```

## Boas Praticas para Evolucao

- Mantenha separacao de camadas (Presentation -> Application -> Domain, Infra implementa portas).
- Evite logica de negocio em `routers.py`; concentre em `use_cases.py` e dominio.
- Sempre tipar funcoes e modelos.
- Ao criar modulo novo, replique estrutura `domain/application/infrastructure/presentation`.
- Documente novos endpoints no arquivo `docs.py` do modulo.

## Testes

A pasta `test/` ja existe com estrutura inicial por modulo (`test/modules/...`).
Se for evoluir a base, recomendável cobrir ao menos:

- casos de uso de `authentication` e `user`;
- validacao de refresh/logout e revogacao;
- fluxos de autorizacao por papel (`user`, `manager`, `admin`);
- endpoints principais com cliente HTTP de teste.

