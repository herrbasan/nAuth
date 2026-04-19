# nAuth Specification

## Architecture Topology
Centralized state authority over decentralized mapped verifiers. 
Optimized for <10ns validation latency at the edge.

### Components
1. **State Server (Authority):** Manages lifecycle of users and tokens. Persists to `nDB`. Broadcasts state mutations.
2. **SDK (Verifier):** Runs in target services. Replicates token state to local native V8 `Map`. Blocks requests synchronously via O(1) memory lookup.

## Tech Stack Constraints
- Runtime: Node.js (Vanilla).
- Network: Native `http`. No Express/frameworks.
- DB: `ndb` (napi bindings).
- Crypto: Native `crypto` (`crypto.scrypt` for passwords, `crypto.randomBytes` for tokens).
- Sync: Native Server-Sent Events (SSE).
- Dependencies: `ndb` only. Zero other external backend packages.
- Frontend: `nui_wc2` (Native Web Components). Served as static ES modules. No bundlers.

## Management UI
- Served directly by the State Server.
- **Security Boundary:** Hardcoded network filtering. Endpoints serving the UI and its API (`/admin/*`) immediately drop connections not originating from `127.0.0.1`, `::1`, or the configured trusted local subnet.
- **Tech:** Built strictly with `nui_wc2`. UI interacts with the State Server API to create/revoke tokens and manage users.

## Data Schema (nDB JSONL)
```json
// User Record
{
  "id": "usr_<random>",
  "type": "user",
  "username": "string",
  "passwordHash": "string",
  "roles": ["string"]
}

// Token Record (Session or API Key)
{
  "id": "tok_<random>", // Opaque 64-byte hex string
  "type": "token",
  "kind": "session|apikey",
  "sub": "usr_<random>", // Maps to User.id
  "roles": ["string"], // Snapshot of granted roles
  "exp": "number|null" // Unix timestamp or null for indefinite
}
```

## State Server API
Port bound. Exposed to trusted local network only.

### REST Endpoints
- `POST /login`: Validates password -> generates `tok_` -> stores in `nDB` -> returns token.
- `POST /keys`: Generates persistent `apikey` -> stores in `nDB` -> returns token.
- `DELETE /tokens/:id`: Removes token from `nDB` -> broadcasts `DEL` offset.
- `GET /state`: Returns complete snapshot of active tokens (for SDK cold start).

### Sync Stream
- `GET /sync`: Keep-alive SSE stream. 
- Payload format: `{"action": "SET|DEL", "token": {...}}`
- Emits instantly on `ndb.insert()`, `ndb.delete()`, or token expiration.

## SDK Mechanics

### Initialization (Block Until Truth)
1. HTTP `GET /state`.
2. Populate local `const tokenCache = new Map()`. Key: `tok.id`, Value: `{ sub, roles }`.
3. Open `GET /sync` SSE connection. 
   - On `SET`: `tokenCache.set(id, data)`
   - On `DEL`: `tokenCache.delete(id)`
4. If SSE drops, reconnect. If state diverges, block all requests until `GET /state` resolves. 

### Validation Interceptor
Target service HTTP handler wrapper.
```javascript
function intercept(secretTokenKey, req, res, next) {
  // 1. Extract Bearer
  const token = req.headers.authorization?.substring(7);
  if (!token) return fail(res);
  
  // 2. O(1) nanosecond validation
  const session = tokenCache.get(token);
  if (!session) return fail(res); // Fail Fast. Drop immediately.
  
  // 3. Attach authoritative state
  req.auth = session;
  return next(req, res);
}
```

## Security & Philosophy Assertions
- Tokens are non-mathematical (opaque). Verification relies entirely on State Server truth via Map presence.
- No DB I/O during request validation.
- No JS Event Loop blocking during request validation.
- Exact exact header payload matching required. No fallback paths.
- SDK has zero knowledge of passwords or `User` objects, only `Token` identities and roles.