# nAuth Specification

## Architecture Topology
Centralized state authority over decentralized mapped verifiers. 
Optimized for <10ns validation latency at the edge.

### Components
1. **State Server (Authority):** Manages lifecycle of users and tokens. Persists to `nDB`. Broadcasts state mutations via SSE.
2. **SDK:** Runs in target services. Three layers: Verifier (token replication + validation), User Management (CRUD via API), Session Management (login/logout flows).

### Directory Structure
```
server.js              — Entry point
src/
  server/              — State Server (routes, SSE, admin UI serving)
  sdk/                 — SDK (exportable module for target services)
  utils/               — Logger wrapper, crypto helpers
```

## Tech Stack Constraints
- Runtime: Node.js (Vanilla). *Note: Runtime choice is not final — may be re-evaluated for performance before implementation.*
- Network: Native `http`. No Express/frameworks.
- DB: `nDB` (napi bindings). Used directly via `const { Database } = require('ndb')` for O(1) synchronous JSONL storage.
- Crypto: Native `crypto` (`crypto.scrypt` for passwords, `crypto.randomBytes` for tokens). Parameters decided at implementation time.
- Sync: Native Server-Sent Events (SSE).
- Logger: `nLogger` submodule used for structured JSON session and error tracking.
- Frontend: `nui_wc2` (Native Web Components). Served as static ES modules. No bundlers.

## Component Integrations

### 1. nDB (Storage)
- Initialized on startup: `const db = new Database('./data/nauth.jsonl')`.
- API is mostly synchronous map operations like `db.insert()`, `db.get()`, `db.query()`.
- **Usage**: Only used by the State Server for authoritative permanent record keeping. Target SDKs never interact with `nDB` directly, relying solely on their isolated V8 `Map` synced via SSE.

### 2. nLogger (Telemetry)
- Loaded via `modules/nLogger`. Wrapped via `src/utils/logger.js`.
- Used to trace boundary events: login attempts, token creations, verification failures (401/403 blocks with source IPs), and SSE sync metrics.
- Structured session logging format (`[LEVEL] [nAuth] message {JSON}`) guarantees parsability by `jq` or local diagnostics.

### 3. nui_wc2 (Management UI)
- Served directly by the State Server out of the `modules/nui_wc2/` static files.
- **Security Boundary:** Admin UI static files served only to `127.0.0.1` / `::1`. Not configurable. Remote admin access requires SSH tunnel or direct machine access.
- **Tech:** Built strictly with `nui_wc2` standalone browser initialization (`<script type="module" src="/nui/nui.js">`). Utilizes native `data-action` mapping to trigger HTTP JSON REST calls to the State Server API to manage tokens, users, and roles.

## Configuration
Environment variables only. Mandatory. Missing config crashes immediately (fail-fast).

| Variable | Description | Example |
|---|---|---|
| `NAUTH_HOST` | Bind address for service endpoints | `0.0.0.0` |
| `NAUTH_PORT` | Listen port | `3100` |
| `NAUTH_SUBNET` | Allowed CIDR for service endpoint access | `192.168.0.0/16` |

Admin UI static files are always `127.0.0.1` / `::1` only — hardcoded, not configurable.

## Data Schema (nDB JSONL)
```json
// User Record
{
  "id": "usr_<random>",
  "type": "user",
  "username": "string",
  "passwordHash": "string",
  "salt": "string",
  "roles": ["string"]
}

// Token Record (Session or API Key)
{
  "id": "tok_<random>",
  "type": "token",
  "kind": "session|apikey",
  "sub": "usr_<random>",
  "roles": ["string"],
  "exp": "number|null"
}
```

## Token Roles & Authorization

Roles on tokens determine what the bearer can do **at the nAuth API level**. The SDK's `intercept()` does not enforce roles — that's the target app's job. But nAuth's own endpoints check the bearer token's roles for authorization.

### Token Kind & Role Matrix

| Token kind | Role | Capabilities |
|---|---|---|
| `session` | *(none)* | Change own password, view own profile |
| `session` | `admin` | Full user CRUD, token management, API key creation |
| `apikey` | `register` | Create users (for app registration flows) |
| `apikey` | `admin` | Full programmatic access |

### Endpoint Authorization

| Endpoint | Auth Required | Authorized For |
|---|---|---|
| `POST /register` | API key (`Authorization: Bearer`) | API key with `register` role |
| `POST /login` | None (credentials in body) | Anyone with valid username + password |
| `POST /change-password` | Session token | Token owner (changes own password) |
| `POST /keys` | Session token | `admin` role required |
| `DELETE /tokens/:id` | Session token | Token owner or `admin` role |
| `GET /state` | None | Network-restricted (`NAUTH_SUBNET`) |
| `GET /sync` | None | Network-restricted (`NAUTH_SUBNET`) |
| `GET /users` | Session or API key | `admin` role required |
| `POST /users` | Session or API key | `admin` role required |
| `PUT /users/:id` | Session or API key | `admin` role required |
| `DELETE /users/:id` | Session or API key | `admin` role required |
| `GET /tokens` | Session or API key | `admin` role required |
| Admin UI static files | None | IP-restricted (`127.0.0.1` / `::1`) |

## State Server API
Bound to `NAUTH_HOST:NAUTH_PORT`. Service endpoints filtered by `NAUTH_SUBNET`.

### Error Response Format
All errors return a flat JSON structure:
```json
{
  "error": "unauthorized",
  "message": "Invalid or missing token"
}
```
- `error`: Machine-readable code (`unauthorized`, `forbidden`, `bad_request`, `not_found`, `conflict`)
- `message`: Human-readable description

### Public Endpoints
- `POST /login`: Validates username + password → generates session token → stores in `nDB` → returns token in response body. Response: `{ "token": "tok_...", "sub": "usr_...", "roles": [...] }`
- `POST /register`: Creates a new user. Requires API key with `register` role in `Authorization` header. Body: `{ "username", "password", "roles": [] }`. Response: `{ "id": "usr_...", "username": "...", "roles": [...] }`. Does NOT issue a session token — the app calls `/login` separately.
- `POST /change-password`: Changes password for the authenticated user. Requires session token. Body: `{ "oldPassword", "newPassword" }`. Verifies old password before updating.

### Token Endpoints
- `POST /keys`: Generates persistent `apikey`. Requires session token with `admin` role. Body: `{ "roles": ["register"] }`. Response: `{ "token": "tok_...", "roles": [...] }`
- `DELETE /tokens/:id`: Removes token from `nDB` → broadcasts `DEL` via SSE. Token owner can delete own tokens. `admin` role can delete any token.
- `GET /state`: Returns complete snapshot of active tokens (for SDK cold start). Network-restricted, no auth.

### User Management Endpoints
All require `admin` role (session or API key).

- `GET /users` — List all users.
- `POST /users` — Create user (username, password, roles).
- `PUT /users/:id` — Update user (password, roles).
- `DELETE /users/:id` — Delete user. **Cascades:** all tokens for this user are deleted and `DEL` events are broadcast via SSE.

### Token Management Endpoints
- `GET /tokens` — List all tokens. Requires `admin` role.

Roles are plain strings managed as part of the user form. No dedicated roles endpoint. No role definitions table.

### Sync Stream
- `GET /sync`: Keep-alive SSE stream. Accessible within `NAUTH_SUBNET`.
- Payload format: `{"action": "SET|DEL", "token": {...}}`
- Emits instantly on `ndb.insert()`, `ndb.delete()`, or token expiration.

### Admin UI
- Static files served from `modules/nui_wc2/` to `127.0.0.1` / `::1` only.
- The admin UI calls the same REST endpoints (users, tokens, keys) using an admin session token obtained via `/login`.

## SDK Architecture

The SDK is a single import that provides three layers. All layers share the same connection to the State Server.

### Layer 1: Verifier (Token Replication & Validation)

#### Initialization (Block Until Truth)
1. HTTP `GET /state`.
2. Populate local `const tokenCache = new Map()`. Key: `tok.id`, Value: `{ sub, roles, exp }`.
3. Open `GET /sync` SSE connection. 
   - On `SET`: `tokenCache.set(id, data)`
   - On `DEL`: `tokenCache.delete(id)`
4. On SSE disconnect, reconnect and perform full state replacement via `GET /state`. No delta sync. No sequence offsets. The full snapshot is always truth.

#### Connection State & Reconnection Policy
The SDK exposes connection state so the target application can decide its own policy:

- `sdk.isConnected` — boolean, SSE connection alive.
- `sdk.onDisconnect` / `sdk.onReconnect` — event hooks for the application.
- `sdk.revoke(tokenId)` — convenience method, calls `DELETE /tokens/:id` on the State Server.
- `sdk.isExpired(session)` — convenience method, checks `session.exp < Date.now()`.

The SDK ships with a **default interceptor** that uses a configurable grace period (default ~30s) — serves from cache while disconnected, blocks after timeout. Applications can replace this with their own policy.

#### Validation Interceptor (Default)
```javascript
function intercept(req, res, next) {
  const token = req.headers.authorization?.substring(7);
  if (!token) return fail(res);

  const session = tokenCache.get(token);
  if (!session) return fail(res);

  req.auth = session;
  return next(req, res);
}
```
The interceptor is a passthrough. `req.auth` contains `{ sub, roles, exp }`. Authorization (role checking, expiration enforcement) is the target application's responsibility.

### Layer 2: User Management

Convenience methods wrapping the State Server's user management API. Requires a token with `admin` role.

- `sdk.users.create(username, password, roles)` → `POST /users`
- `sdk.users.list()` → `GET /users`
- `sdk.users.update(id, { password, roles })` → `PUT /users/:id`
- `sdk.users.delete(id)` → `DELETE /users/:id`

### Layer 3: Session Management

Manages the current user's session. Used by apps for login/logout flows.

- `sdk.auth.login(username, password)` → `POST /login` → stores token internally
- `sdk.auth.logout()` → `DELETE /tokens/:id` → clears local token
- `sdk.auth.register(username, password, roles)` → `POST /register` → requires app API key
- `sdk.auth.changePassword(oldPassword, newPassword)` → `POST /change-password`
- `sdk.auth.createApiKey(roles)` → `POST /keys` → requires `admin` role
- `sdk.auth.getSession()` → returns current `{ sub, roles, exp }` or null
- `sdk.auth.isAuthenticated` → boolean

## Application Integration Pattern

### Setup (Admin)
1. Admin creates an API key via admin UI or `POST /keys` with `register` role.
2. API key is stored in the app's config (`NAUTH_APP_KEY` env var).

### User Registration Flow
```
User → App UI (register form)
         ↓
     App calls sdk.auth.register(username, password)
         ↓
     SDK → POST /register with API key → nAuth creates user → returns { id, username, roles }
         ↓
     App creates user record in its own data store, keyed by usr_id
         ↓
     App calls sdk.auth.login(username, password)
         ↓
     SDK → POST /login → nAuth returns token → SDK stores internally
         ↓
     User is authenticated. All requests pass through sdk.intercept()
         ↓
     App uses req.auth.sub to scope all data queries
```

### Data Ownership Split

| Concern | nAuth | App |
|---|---|---|
| User identity (id, username, password, roles) | **Owns** | Reads `req.auth.sub` |
| Token issuance & validation | **Owns** | Uses SDK |
| App-specific data (chat history, preferences, etc.) | **Doesn't know about** | **Owns** — keyed by `usr_id` |
| Login/register UI | Admin UI (admin only) | **App UI** (user-facing) |
| Password management | **Provides API** | **Provides UI** |

## Roles Model
- Roles are plain strings on user and token records.
- nAuth uses roles **only for its own endpoint authorization** (see Token Roles & Authorization table).
- The SDK exposes `roles` as metadata on `req.auth`. What the target service does with them is outside nAuth's boundary.
- No role definitions table. No permissions matrix.

## Security & Philosophy Assertions
- Tokens are non-mathematical (opaque). Verification relies entirely on State Server truth via Map presence.
- No DB I/O during request validation.
- No JS Event Loop blocking during request validation.
- Exact header payload matching required. No fallback paths.
- SDK has zero knowledge of passwords or `User` objects, only `Token` identities and roles.
- No rate limiting. nAuth operates in trusted, network-isolated, on-premise environments. If rate limiting becomes necessary, the broader architecture would need re-evaluation.
- Admin UI static files served only to `127.0.0.1` / `::1`. No remote admin UI access.
- Token `exp` is metadata. nAuth does not enforce expiration. Target applications own their session lifecycle.
- Registration endpoint gated by API key — no open account creation.
- nAuth is the identity provider (who you are). Apps are the data owners (what you did). Linked by `usr_id`.

## Design Decisions Log
1. **SDK is a passthrough for verification** — roles and expiration are metadata, not enforced by nAuth's `intercept()`.
2. **nAuth enforces roles on its own endpoints** — API-level authorization uses token roles.
3. **Cascade delete on user removal** — all user tokens are revoked and broadcast immediately.
4. **Full state replacement on SSE reconnect** — no delta sync, no offsets. Simplicity over efficiency.
5. **No cookies** — tokens returned in response body only. Works for browser and non-browser consumers.
6. **Env vars only for config** — no config files. Missing config crashes immediately.
7. **Admin UI static files hardcoded to localhost** — on-premise security requirement. SSH tunnel for remote admin.
8. **No rate limiting** — trusted environment assumption.
9. **`salt` field on user record** — stored alongside `passwordHash` for clarity.
10. **Three-layer SDK** — Verifier (token sync), User Management (CRUD), Session Management (login/logout). Single import, cohesive API.
11. **Registration via API key** — apps register users programmatically using a pre-issued API key with `register` role. No open registration.
12. **Separation of identity and data** — nAuth owns user identity, apps own app-specific data. Linked by `usr_id`.
