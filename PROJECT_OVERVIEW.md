# NestJS Boilerplate — Complete Project Overview

This document explains the full architecture, flow, and structure of this NestJS boilerplate. Paste it into ChatGPT for context.

---

## What This Project Is

A production-ready NestJS REST API boilerplate that supports **two database backends** switchable via an environment variable:

- **Relational** — PostgreSQL via TypeORM
- **Document** — MongoDB via Mongoose

The same business logic, controllers, and services work with both. The database choice only affects the infrastructure layer.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Framework | NestJS 11 (Express platform) |
| Language | TypeScript 5 |
| Relational DB | PostgreSQL + TypeORM |
| Document DB | MongoDB + Mongoose |
| Auth | JWT (access + refresh tokens) via Passport |
| Password hashing | bcryptjs |
| Validation | class-validator + class-transformer |
| API docs | Swagger (OpenAPI) at `/docs` |
| Code generation | Hygen templates |

---

## Project Structure

```
src/
├── main.ts                    # Bootstrap: Swagger, global pipes, versioning
├── app.module.ts              # Root module; picks DB based on env var
│
├── config/                    # App-level config (port, prefix, language)
│
├── auth/                      # Authentication module
│   ├── auth.controller.ts     # HTTP routes for auth
│   ├── auth.service.ts        # Business logic: login, register, refresh, logout
│   ├── auth.module.ts
│   ├── config/                # JWT secrets, expiry config
│   ├── dto/                   # Request/response shapes
│   └── strategies/            # Passport JWT + JWT-refresh + anonymous strategies
│
├── users/                     # Users module
│   ├── users.controller.ts    # CRUD endpoints (admin only)
│   ├── users.service.ts       # Business logic: create, update, delete, find
│   ├── users.module.ts
│   ├── domain/user.ts         # Pure domain entity (no DB annotations)
│   ├── dto/                   # create-user, update-user, query-user DTOs
│   └── infrastructure/
│       └── persistence/
│           ├── user.repository.ts              # Abstract repository interface
│           ├── relational/repositories/        # TypeORM implementation
│           └── document/repositories/          # Mongoose implementation
│
├── session/                   # Session management module
│   ├── session.service.ts
│   ├── domain/session.ts
│   └── infrastructure/persistence/            # Same relational/document split
│
├── roles/                     # Role definitions (admin=1, user=2)
├── statuses/                  # Status definitions (active=1, inactive=2)
│
├── database/
│   ├── typeorm-config.service.ts   # TypeORM connection setup
│   ├── mongoose-config.service.ts  # Mongoose connection setup
│   ├── data-source.ts              # TypeORM DataSource for CLI migrations
│   ├── migrations/                 # TypeORM SQL migration files
│   └── seeds/                      # Seed data for both DB types
│
└── utils/                     # Shared helpers
    ├── serializer.interceptor.ts   # Resolves promises in responses
    ├── infinity-pagination.ts      # Cursor-style pagination helper
    ├── validate-config.ts          # Config validation utility
    └── types/                      # Shared TypeScript types
```

---

## How the Database Switch Works

In `app.module.ts`, at module load time:

```ts
const infrastructureDatabaseModule = (databaseConfig() as DatabaseConfig).isDocumentDatabase
  ? MongooseModule.forRootAsync({ useClass: MongooseConfigService })
  : TypeOrmModule.forRootAsync({ useClass: TypeOrmConfigService, ... });
```

The `isDocumentDatabase` flag is set by the `DATABASE_TYPE` env var. If it equals `"mongodb"` it is `true`; otherwise (e.g., `"postgres"`) it is `false`.

Each feature module (users, sessions) provides **two repository implementations** behind a single abstract class:

```
UserRepository (abstract)
├── RelationalUserRepository   ← registered when DATABASE_TYPE=postgres
└── DocumentUserRepository     ← registered when DATABASE_TYPE=mongodb
```

Services only depend on the abstract `UserRepository` — they never know which DB is running.

---

## Startup Flow (`main.ts`)

1. `NestFactory.create(AppModule)` — creates NestJS app with CORS enabled
2. Global API prefix set from `APP_PREFIX` env var (default `api`), root `/` excluded
3. URI-based versioning enabled (routes use `/v1/`, `/v2/`, etc.)
4. `ValidationPipe` — validates all incoming request bodies via class-validator decorators
5. `ClassSerializerInterceptor` — serializes responses, respects `@Exclude` / `@Expose` on domain classes
6. `ResolvePromisesInterceptor` — resolves lazy/promise fields that class-transformer can't handle
7. Swagger UI mounted at `/docs`
8. App listens on `APP_PORT` (default `3001`)

---

## Auth Flow

### Registration

```
POST /api/v1/auth/email/register
Body: { firstName, lastName, email, password }

→ AuthService.register()
→ UsersService.create()
    → hashes password with bcrypt
    → assigns role: user (id=2), status: active (id=1)
    → saves to DB
← 204 No Content
```

### Login

```
POST /api/v1/auth/email/login
Body: { email, password }

→ AuthService.validateLogin()
    → finds user by email
    → verifies password with bcrypt.compare()
    → creates a new Session record with a random SHA-256 hash
    → signs two JWTs:
        - Access token  (payload: { id, role, sessionId })  expires: AUTH_JWT_TOKEN_EXPIRES_IN (e.g. 15m)
        - Refresh token (payload: { sessionId, hash })       expires: AUTH_REFRESH_TOKEN_EXPIRES_IN (e.g. 3650d)
← 200 { token, refreshToken, tokenExpires, user }
```

### Authenticated Request

```
GET /api/v1/auth/me
Header: Authorization: Bearer <access_token>

→ JwtStrategy.validate(payload)
    → verifies signature using AUTH_JWT_SECRET
    → returns payload { id, role, sessionId } (no DB lookup — stateless)
→ AuthService.me()
    → UsersService.findById(payload.id)
← 200 User object
```

### Token Refresh

```
POST /api/v1/auth/refresh
Header: Authorization: Bearer <refresh_token>

→ JwtRefreshStrategy validates refresh token using AUTH_REFRESH_SECRET
→ AuthService.refreshToken()
    → finds session by { sessionId, hash } — validates the hash matches DB
    → generates a new hash, updates the session record (hash rotation)
    → signs new access + refresh tokens
← 200 { token, refreshToken, tokenExpires }
```

### Logout

```
POST /api/v1/auth/logout
Header: Authorization: Bearer <access_token>

→ AuthService.logout()
    → SessionService.deleteById(sessionId)
    → session removed from DB — refresh token is now invalid
← 204 No Content
```

### Profile Update

```
PATCH /api/v1/auth/me
Header: Authorization: Bearer <access_token>
Body: { firstName?, lastName?, password?, oldPassword? }

→ If changing password:
    → verifies oldPassword matches current hash
    → invalidates ALL other sessions for this user (forces re-login on other devices)
    → updates password hash
← 200 Updated User
```

### Account Delete (Soft)

```
DELETE /api/v1/auth/me
Header: Authorization: Bearer <access_token>

→ UsersService.remove(id)  — soft delete (sets deletedAt, not a hard delete)
← 204 No Content
```

---

## Users Module (Admin CRUD)

All routes require `Authorization: Bearer <access_token>` **and** `role = admin (1)`.

| Method | Route | Description |
|---|---|---|
| POST | `/api/v1/users` | Create user |
| GET | `/api/v1/users` | List users (paginated) |
| GET | `/api/v1/users/:id` | Get one user |
| PATCH | `/api/v1/users/:id` | Update user |
| DELETE | `/api/v1/users/:id` | Soft-delete user |

### Pagination

`GET /api/v1/users?page=1&limit=10&filters[role][id]=2&sort[0][orderBy]=createdAt&sort[0][order]=DESC`

Returns:

```json
{
  "data": [ ...users ],
  "hasNextPage": true
}
```

---

## Session System

On every login a `Session` record is created:

```
Session {
  id
  user      → FK to User
  hash      → random SHA-256 string (stored in DB, also embedded in refresh token)
  createdAt
  updatedAt
  deletedAt
}
```

On refresh, the server checks `sessionId + hash` match in DB, then **rotates the hash** (generates a new one, updates DB, embeds new hash in new refresh token). This means each refresh token is single-use and old refresh tokens become invalid — a rolling refresh token pattern.

---

## Role & Status System

### Roles (enum)
| ID | Name |
|----|------|
| 1 | admin |
| 2 | user |

### Statuses (enum)
| ID | Name |
|----|------|
| 1 | active |
| 2 | inactive |

These are stored as IDs in the database. They are plain enums in code — no separate DB table lookup at runtime.

---

## Domain / Infrastructure Separation Pattern

Each feature follows this layered pattern:

```
domain/user.ts              ← Plain class, no DB decorators. Used by services and controllers.
    ↑ mapped by
infrastructure/
  persistence/
    user.repository.ts      ← Abstract class defining the interface
    relational/
      entities/user.entity.ts    ← TypeORM @Entity with @Column decorators
      mappers/user.mapper.ts     ← Converts entity ↔ domain
      repositories/user.repository.ts  ← Implements abstract, uses TypeORM
    document/
      entities/user.schema.ts    ← Mongoose @Schema/@Prop decorators
      mappers/user.mapper.ts     ← Converts document ↔ domain
      repositories/user.repository.ts  ← Implements abstract, uses Mongoose
```

Services always import and use the abstract `UserRepository`. The concrete implementation is injected by the module based on which DB is configured.

---

## Serialization / Response Shaping

The `User` domain class uses class-transformer groups:

```ts
@Expose({ groups: ['me', 'admin'] })
email: string | null;   // only shown to the user themselves or admins

@Exclude({ toPlainOnly: true })
password?: string;      // never sent in any response
```

Controllers set the active group:

```ts
@SerializeOptions({ groups: ['me'] })   // on /auth/* routes
@SerializeOptions({ groups: ['admin'] }) // on /users/* routes
```

---

## Environment Variables

### Relational (PostgreSQL)

```env
NODE_ENV=development
APP_PORT=3001
APP_NAME="NestJS API"
API_PREFIX=api
APP_FALLBACK_LANGUAGE=en
APP_HEADER_LANGUAGE=x-custom-lang

DATABASE_TYPE=postgres
DATABASE_HOST=localhost
DATABASE_PORT=5433
DATABASE_USERNAME=root
DATABASE_PASSWORD=secret
DATABASE_NAME=api
DATABASE_SYNCHRONIZE=false
DATABASE_MAX_CONNECTIONS=100
DATABASE_SSL_ENABLED=false

AUTH_JWT_SECRET=secret
AUTH_JWT_TOKEN_EXPIRES_IN=15m
AUTH_REFRESH_SECRET=secret_for_refresh
AUTH_REFRESH_TOKEN_EXPIRES_IN=3650d
```

### Document (MongoDB)

Same as above but:

```env
DATABASE_TYPE=mongodb
DATABASE_URL=mongodb://localhost:27017/api
```

---

## Code Generation (Hygen)

New resources and properties are scaffolded with CLI commands, not hand-written:

```bash
# Generate a new resource (both DB variants)
npm run generate:resource:all-db

# Generate only relational
npm run generate:resource:relational

# Generate only document
npm run generate:resource:document

# Add a property to an existing entity (both DBs)
npm run add:property:to-all-db
```

These generators create: controller, service, module, domain class, DTOs, both repository implementations, both entity/schema files, mappers, and the TypeORM migration — all wired together.

---

## Running the App

```bash
# Relational (PostgreSQL)
npm run migration:run:relational    # run DB migrations
npm run seed:run:relational:local   # seed initial roles, statuses, admin user
npm run start:relational            # start with hot-reload

# Document (MongoDB)
npm run seed:run:document:local
npm run start:document
```

API is available at `http://localhost:3001/api/v1/`
Swagger docs at `http://localhost:3001/docs`

---

## Key Design Decisions

1. **No email confirmation** — registration immediately creates an active user. The original boilerplate had confirm-email flow but it has been removed in this version.
2. **No social auth** — Google/Facebook/Apple OAuth modules have been removed.
3. **Stateless access tokens** — the JWT strategy does NOT look up the user in DB on every request. It only verifies the token signature. The session (and thus revocation capability) only matters at refresh time.
4. **Soft deletes** — users are never hard-deleted; `deletedAt` is set. TypeORM/Mongoose queries automatically exclude soft-deleted records.
5. **Rolling refresh tokens** — each use of a refresh token generates a new one and invalidates the old hash, preventing refresh token reuse attacks.
