# Gateway Auth Endpoints — v1.0 Commission
**Project:** ZenSci Portal
**Date:** 2026-02-19
**Target repo:** `AgenticGatewayByDojoGenesis/`
**Spec source:** `ZenithScience/docs/specs/portal/zen-sci-web-v1.0-spec.md` §R1 Auth section
**Status:** Ready to commission

---

## 1. Context & Grounding

### Why this commission exists

The AgenticGateway already validates JWTs via `AuthMiddleware`, `AdminAuthMiddleware`, and `OptionalAuthMiddleware` in `server/middleware/auth.go`. It uses HMAC HS256 signing with `JWT_SECRET` from the environment. What it does **not have** is any endpoint to issue tokens — no `/auth/register`, `/auth/login`, `/auth/refresh`.

The zen-sci-portal desktop app and web portal both authenticate against the Gateway. The `JWT_SECRET` is shared between Gateway and portal, so tokens issued here are trusted everywhere. This commission adds the three auth endpoints to AgenticGateway.

### Files you must read before writing any code

```
server/middleware/auth.go          -- validateToken(), GatewayClaims, jwtSecret, signing method
server/server.go                   -- Server struct, setupMiddleware(), OptionalAuthMiddleware usage
server/router.go                   -- Route registration pattern (v1 group, admin group)
server/handle_health.go            -- Canonical handler style (s.errorResponse, c.JSON, struct binding)
server/handle_gateway.go           -- POST handler pattern (ShouldBindJSON, gin.H responses)
server/database/                   -- Existing DB adapter structure
server/migrations/20260207_v0.0.30_local_auth.sql  -- local_users table schema
go.mod                             -- Current dependencies
```

### What already exists that you MUST use

| Existing | How to use it |
|---|---|
| `jwtSecret` in `middleware/auth.go` | **Re-export or relocate** so `handle_auth.go` can call `issueToken()` using the same secret |
| `GatewayClaims{Role string}` | Embed in issued tokens; use `Role: "user"` for registered users |
| `local_users` SQLite table | **Extend** with a new migration — add `email`, `password_hash`, `display_name` |
| `s.errorResponse(c, status, code, message)` | Use for all error responses |
| `OptionalAuthMiddleware()` | Auth endpoints must be registered **without** this middleware |
| Existing Gin request ID + rate limit middleware | Already applied globally — do not add extra |

### Critical constraints

- **DO NOT** modify `middleware/auth.go` — only add a new `issueToken()` function in `handle_auth.go` using the same `jwtSecret` variable via a package-level accessor.
- **DO NOT** add a `/auth` group to the `/v1` prefix — register it at root: `s.router.Group("/auth")`.
- **DO NOT** use PostgreSQL — auth user store lives in the same SQLite (`.dojo/dojo.db`) as the rest of the Gateway's local state.
- **DO NOT** implement email verification or password reset in v1 — stub-safe responses only.
- **DO NOT** implement refresh token rotation in v1 — return a new access token signed from the same credentials.
- Access token TTL: **24 hours**. Refresh token TTL: **7 days**.

---

## 2. Implementation Groups

Execute groups in sequence. Each group builds on the previous.

---

### Group 1 — Migration: extend local_users

**File to create:**
```
server/migrations/20260219_v1.0.0_portal_auth.sql
```

**Content specification:**

```sql
-- Migration: v1.0.0 Portal Auth — extend local_users for portal credential storage
-- Date: 2026-02-19
-- Requires: v0.0.30 (local_users table must exist)
-- Database: .dojo/dojo.db (SQLite)

PRAGMA foreign_keys = ON;
BEGIN TRANSACTION;

INSERT OR IGNORE INTO schema_migrations (version, applied_at, description)
VALUES ('20260219_v1.0.0_portal_auth', datetime('now'),
        'Portal auth: add email, password_hash, display_name to local_users');

-- Extend local_users with portal credential fields
-- ALTER TABLE in SQLite only supports ADD COLUMN
ALTER TABLE local_users ADD COLUMN email TEXT;
ALTER TABLE local_users ADD COLUMN password_hash TEXT;
ALTER TABLE local_users ADD COLUMN display_name TEXT;

-- Unique constraint on email (non-null emails must be unique)
-- SQLite does not support ADD CONSTRAINT on existing tables.
-- Enforce uniqueness via unique index with WHERE filter.
CREATE UNIQUE INDEX IF NOT EXISTS idx_local_users_email
    ON local_users(email) WHERE email IS NOT NULL;

COMMIT;
```

**Apply logic:** The migration system must be updated to apply this migration on startup if not yet applied. Follow the same pattern as `migration.go` in `server/database/`.

---

### Group 2 — Database layer: auth queries

**File to create:**
```
server/database/auth.go
```

**Package:** `package database`

**Functions to implement:**

```go
// CreatePortalUser inserts a new user record with hashed credentials.
// Returns the generated user ID (UUID v4).
func CreatePortalUser(db *sql.DB, email, passwordHash, displayName string) (string, error)

// GetPortalUserByEmail retrieves a user by email for login.
// Returns (userID, passwordHash, displayName, error).
// Returns sql.ErrNoRows if not found.
func GetPortalUserByEmail(db *sql.DB, email string) (id, passwordHash, displayName string, err error)

// GetPortalUserByID retrieves a user by ID for token refresh.
// Returns (email, displayName, error).
func GetPortalUserByID(db *sql.DB, userID string) (email, displayName string, err error)
```

**Implementation notes:**
- Use `"database/sql"` and `"github.com/google/uuid"`.
- `CreatePortalUser`: INSERT INTO local_users with `user_type = 'authenticated'`, generated UUID as id, `created_at = NOW()`, `last_accessed_at = NOW()`, plus the new `email`, `password_hash`, `display_name` fields.
- `GetPortalUserByEmail`: SELECT id, password_hash, display_name FROM local_users WHERE email = ? AND user_type = 'authenticated'.
- `GetPortalUserByID`: SELECT email, display_name FROM local_users WHERE id = ? AND user_type = 'authenticated'.
- All errors propagate — no swallowing.

---

### Group 3 — Token issuance function

**Location:** At the bottom of `server/handle_auth.go` (same file as the handlers, keeps it private to the server package).

```go
// issueToken creates a signed JWT for the given user ID and role.
// ttl controls token lifetime (e.g. 24*time.Hour for access, 7*24*time.Hour for refresh).
func issueToken(userID, role string, ttl time.Duration) (string, error) {
    claims := middleware.GatewayClaims{
        RegisteredClaims: jwt.RegisteredClaims{
            Subject:   userID,
            IssuedAt:  jwt.NewNumericDate(time.Now()),
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(ttl)),
        },
        Role: role,
    }
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    // jwtSecret is package-level in middleware — re-export via GetJWTSecret() accessor
    return token.SignedString(middleware.GetJWTSecret())
}
```

**Add to `server/middleware/auth.go`:**

```go
// GetJWTSecret returns the configured JWT secret for use by token-issuing handlers.
// This is the only function in this file that should be exported for issuance.
func GetJWTSecret() []byte {
    return jwtSecret
}
```

---

### Group 4 — Handlers: POST /auth/register, /auth/login, /auth/refresh

**File to create:**
```
server/handle_auth.go
```

**Package:** `package server`

**Imports needed:**
```go
import (
    "net/http"

    "github.com/gin-gonic/gin"
    "golang.org/x/crypto/bcrypt"

    "github.com/TresPies-source/AgenticGatewayByDojoGenesis/server/database"
    "github.com/TresPies-source/AgenticGatewayByDojoGenesis/server/middleware"
)
```

**Add `golang.org/x/crypto` to go.mod** (run `go get golang.org/x/crypto@latest`).

---

#### POST /auth/register

```
Request body:
{
    "email": "researcher@lab.edu",
    "password": "...",        // 8–72 chars, validated server-side
    "display_name": "Jane Doe"
}

Success 201:
{
    "user_id": "<uuid>",
    "display_name": "Jane Doe",
    "access_token": "<jwt>",
    "refresh_token": "<jwt>",
    "expires_in": 86400
}

Error 400: invalid_request   (missing/malformed fields)
Error 409: email_taken       (unique constraint violation)
Error 500: server_error
```

**Handler logic:**
1. `ShouldBindJSON` → validate email non-empty, password 8–72 chars, display_name non-empty. Return 400 on failure.
2. `bcrypt.GenerateFromPassword([]byte(req.Password), 12)` → return 500 on bcrypt error.
3. `database.CreatePortalUser(s.authDB, req.Email, string(hash), req.DisplayName)` → on `UNIQUE constraint failed`, return 409 `email_taken`.
4. `issueToken(userID, "user", 24*time.Hour)` → access token.
5. `issueToken(userID, "refresh", 7*24*time.Hour)` → refresh token.
6. Return 201 with both tokens.

---

#### POST /auth/login

```
Request body:
{
    "email": "researcher@lab.edu",
    "password": "..."
}

Success 200:
{
    "user_id": "<uuid>",
    "display_name": "Jane Doe",
    "access_token": "<jwt>",
    "refresh_token": "<jwt>",
    "expires_in": 86400
}

Error 400: invalid_request
Error 401: invalid_credentials   (wrong email or password — same message for both)
Error 500: server_error
```

**Handler logic:**
1. Bind + validate fields. Return 400 on failure.
2. `database.GetPortalUserByEmail(s.authDB, req.Email)` → on `sql.ErrNoRows`, return 401 `invalid_credentials`.
3. `bcrypt.CompareHashAndPassword([]byte(storedHash), []byte(req.Password))` → on error, return 401 `invalid_credentials`. **Never distinguish between wrong email and wrong password.**
4. Issue access + refresh tokens.
5. Return 200.

---

#### POST /auth/refresh

```
Request body:
{
    "refresh_token": "<jwt>"
}

Success 200:
{
    "access_token": "<jwt>",
    "expires_in": 86400
}

Error 400: invalid_request
Error 401: invalid_token   (expired, malformed, or not a refresh token)
Error 500: server_error
```

**Handler logic:**
1. Bind `refresh_token` field. Return 400 if empty.
2. Call `middleware.ValidateRefreshToken(req.RefreshToken)` → returns `(userID string, err error)`.
3. Add to `server/middleware/auth.go`:
   ```go
   // ValidateRefreshToken validates a refresh token and returns the user ID.
   // Returns error if the token is expired, malformed, or not a refresh token.
   func ValidateRefreshToken(tokenString string) (string, error) {
       claims := &GatewayClaims{}
       token, err := jwt.ParseWithClaims(tokenString, claims, func(t *jwt.Token) (interface{}, error) {
           if _, ok := t.Method.(*jwt.SigningMethodHMAC); !ok {
               return nil, jwt.ErrSignatureInvalid
           }
           return jwtSecret, nil
       })
       if err != nil || !token.Valid {
           return "", jwt.ErrSignatureInvalid
       }
       if claims.Role != "refresh" {
           return "", jwt.ErrTokenInvalidClaims
       }
       subject, err := claims.GetSubject()
       if err != nil || subject == "" {
           return "", jwt.ErrTokenInvalidSubject
       }
       return subject, nil
   }
   ```
4. On error: return 401 `invalid_token`.
5. `database.GetPortalUserByID(s.authDB, userID)` → verify user still exists. On error: return 401 `invalid_token`.
6. `issueToken(userID, "user", 24*time.Hour)` → new access token only.
7. Return 200 with new access token.

---

### Group 5 — Server struct: add authDB

The `Server` struct in `server/server.go` needs a `*sql.DB` field for the auth database.

**Add to Server struct:**
```go
authDB *sql.DB  // SQLite connection to .dojo/dojo.db for auth operations
```

**Add to ServerDeps struct** (wherever it is defined):
```go
AuthDB *sql.DB
```

**Wire in New():**
```go
s.authDB = deps.AuthDB
```

**In `main.go`** (or wherever `New(ServerDeps{...})` is called): open the SQLite connection to `.dojo/dojo.db` and pass it as `AuthDB`. Apply the migration `20260219_v1.0.0_portal_auth.sql` on startup if not yet applied via the existing migration runner.

---

### Group 6 — Route registration

**In `server/router.go`**, add the auth group **before** the v1 group, at the top of `setupRoutes()`:

```go
// ─── Auth (Portal v1.0) ──────────────────────────────────────────────────────
// Routes are public — no AuthMiddleware applied.
// Rate limiting is provided by the global RateLimitMiddleware.
auth := s.router.Group("/auth")
{
    auth.POST("/register", s.handleAuthRegister)
    auth.POST("/login", s.handleAuthLogin)
    auth.POST("/refresh", s.handleAuthRefresh)
}
```

The handler method names must match exactly: `s.handleAuthRegister`, `s.handleAuthLogin`, `s.handleAuthRefresh`.

---

### Group 7 — Tests

**File to create:**
```
server/handle_auth_test.go
```

**Tests to write (use Go's `testing` + `net/http/httptest` + `encoding/json`):**

| Test name | What it verifies |
|---|---|
| `TestRegister_Success` | 201 with both tokens in response body |
| `TestRegister_DuplicateEmail` | 409 `email_taken` on second register with same email |
| `TestRegister_InvalidPassword` | 400 when password < 8 chars |
| `TestRegister_MissingFields` | 400 when email or display_name absent |
| `TestLogin_Success` | 200 with tokens after successful register+login |
| `TestLogin_WrongPassword` | 401 `invalid_credentials` |
| `TestLogin_UnknownEmail` | 401 `invalid_credentials` |
| `TestRefresh_Success` | 200 with new access_token from valid refresh token |
| `TestRefresh_ExpiredToken` | 401 `invalid_token` |
| `TestRefresh_AccessTokenNotRefresh` | 401 `invalid_token` when access token used as refresh |

**Setup:** Use an in-memory SQLite database (`:memory:`) for each test. Apply the migrations programmatically in `TestMain`.

---

## 3. File Manifest

```
AgenticGatewayByDojoGenesis/
├── go.mod                                   MODIFIED — add golang.org/x/crypto
├── server/
│   ├── handle_auth.go                       NEW — 3 handlers + issueToken()
│   ├── handle_auth_test.go                  NEW — 10 tests
│   ├── server.go                            MODIFIED — add authDB field + ServerDeps.AuthDB
│   ├── router.go                            MODIFIED — register /auth group
│   ├── middleware/
│   │   └── auth.go                          MODIFIED — add GetJWTSecret() + ValidateRefreshToken()
│   ├── database/
│   │   └── auth.go                          NEW — CreatePortalUser, GetPortalUserByEmail, GetPortalUserByID
│   └── migrations/
│       └── 20260219_v1.0.0_portal_auth.sql  NEW — ALTER TABLE local_users + unique email index
```

**Total:** 2 new files, 3 modified files, 1 new migration.

---

## 4. Success Criteria

All of the following must be true before the commission is considered complete:

- [ ] `go build ./...` passes with zero errors
- [ ] `go test ./server/...` passes — all 10 new auth tests green
- [ ] `POST /auth/register` with valid payload returns 201 + two JWTs
- [ ] `POST /auth/register` with duplicate email returns 409 `email_taken`
- [ ] `POST /auth/login` with correct credentials returns 200 + two JWTs
- [ ] `POST /auth/login` with wrong password returns 401 `invalid_credentials` (not 403, not 404)
- [ ] `POST /auth/refresh` with valid refresh token returns 200 + new access token
- [ ] `POST /auth/refresh` with an access token (not refresh) returns 401
- [ ] `POST /auth/refresh` with expired token returns 401
- [ ] Issued access tokens pass `AuthMiddleware()` on protected routes
- [ ] Refresh token has `role: "refresh"` in claims, access token has `role: "user"`
- [ ] Existing tests (`go test ./...`) are unaffected — no regressions

---

## 5. Explicit Constraints

1. **DO NOT** rotate refresh tokens. On refresh, issue only a new access token; return the same refresh token or require re-login after 7 days.
2. **DO NOT** add email verification (no email sending in v1).
3. **DO NOT** add rate limiting beyond what the global `RateLimitMiddleware` already provides.
4. **DO NOT** store plaintext passwords anywhere — bcrypt cost 12 minimum.
5. **DO NOT** distinguish wrong-email from wrong-password in error messages or codes — always return `invalid_credentials`.
6. **DO NOT** use PostgreSQL — SQLite only, same `.dojo/dojo.db`.
7. **DO NOT** add a GET /auth/me endpoint — not needed until portal v1.1.
8. **DO NOT** modify any existing route groups (`/v1`, `/admin`) — `/auth` is a new top-level group.
9. `/auth` routes must work WITHOUT a JWT header present — they are fully public endpoints.
10. The `jwtSecret` variable in `middleware/auth.go` must remain unexported. Only `GetJWTSecret()` is exported.

---

## 6. Integration Note for the Portal Commission

When the zen-sci-web portal commission is written, the SvelteKit app will:
- Call `POST /auth/register` or `POST /auth/login` on the Gateway URL
- Store the returned `access_token` in a secure httpOnly cookie (portal-side)
- Send `Authorization: Bearer <token>` on all authenticated Gateway requests
- Call `POST /auth/refresh` when the access token expires

The Gateway URL in Docker Compose is `http://gateway:8080` (internal network). The portal's `GATEWAY_URL` env var points to this.
