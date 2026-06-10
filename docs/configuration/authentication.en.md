# Authentication

Caelix includes a multi-user authentication system based on **SQLite** and **JWT**. No external database is required: everything is stored in a single file within the container.

## Overview

| Component | Technology |
|-----------|-----------|
| User storage | SQLite (`/workspace/.caelix/auth.db`), schema versioned via `PRAGMA user_version` |
| Password hashing | bcrypt |
| Access tokens | JWT HS256 (PyJWT) |
| SPA session | `caelix_session` cookie, **httpOnly**, `SameSite=strict` |
| Signing key | Auto-generated and persisted in `/workspace/.caelix/jwt_secret.key` |

## Web console session (httpOnly cookie)

To harden the interface against XSS, the session JWT is **never stored in `localStorage`**. On login, the backend sets an httpOnly cookie:

| Attribute | Value |
|-----------|-------|
| Name | `caelix_session` |
| `HttpOnly` | Yes — not readable from JavaScript |
| `SameSite` | `strict` — blocks cross-site requests (CSRF hardening) |
| `Secure` | Enabled automatically when the request arrives over HTTPS (off for plain HTTP / LAN deployments) |
| `Max-Age` | Same as the JWT lifetime (`CAELIX_JWT_EXPIRE_MINUTES`, 8 h by default) |
| `Path` | `/` |

The SPA keeps the token in memory for the tab's lifetime and relies on the httpOnly cookie to re-authenticate after a page refresh (call to `/api/auth/me`). Logging out (`POST /api/auth/logout`) clears the cookie.

The backend resolves the token in this priority order: `Authorization: Bearer` header, then `X-Caelix-Token`, then the `caelix_session` cookie, then (legacy) the `?token=` query parameter. CLI/API clients therefore use the `Bearer` header while the SPA uses the cookie.

## Roles

Caelix defines two roles:

### Administrator (`admin`)

Full access to all features:

- Create, delete, modify containers, images, volumes, networks, stacks
- Execute commands in containers (`exec`)
- Manifest management (edit, validate, doctor)
- Backup / restore configuration
- Prune Docker resources
- User management (CRUD)
- Webhooks, registries, templates, proxy, notifications management

### Technician (`technicien`)

Read access and limited operational actions:

- View all dashboards, logs, stats, inspections
- Start / stop / restart services and containers
- Acknowledge alerts and incidents
- Reconcile a single application
- Read stack environment variables
- Browse volumes (read-only)

Destructive actions (delete, kill, prune, exec, deploy, backup/restore, manifest editing) are **forbidden** for technicians — the backend returns `403 Forbidden`.

## Default Account

On first startup, if no users exist in the database, Caelix automatically creates:

- **Username**: `admin`
- **Password**: `admin` (or the value of `CAELIX_ADMIN_PASSWORD`)
- **Role**: `admin`

!!! warning "Mandatory password change"
    After the first login with `admin/admin`, the interface forces a password change before allowing access to the dashboard.

## Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `CAELIX_ADMIN_PASSWORD` | Initial admin account password | `admin` |
| `CAELIX_JWT_SECRET` | Secret key for signing JWT tokens. If not set, a key is auto-generated and persisted | Auto-generated |
| `CAELIX_JWT_EXPIRE_MINUTES` | JWT token validity duration in minutes | `480` (8 hours) |
| `CAELIX_UI_TOKEN` | Legacy token (backward compatibility). **Deprecated** — migrate to user accounts | - |

## Security

### Brute Force Protection

The login endpoint includes a rate limiter:

- **5 failed attempts** within a **5-minute** window trigger a lockout
- The lockout lasts **10 minutes** for the concerned IP address
- Failed attempts are logged in the audit trail

### Token Invalidation

JWT tokens include a timestamp (`pw_ts`) corresponding to the user's `updated_at`. After a password change:

- All old tokens are automatically invalidated
- A new token is issued at the time of the change
- Active sessions on other devices are disconnected

### Timing Attack Protection

The login endpoint always executes a bcrypt verification, even if the user doesn't exist. This prevents username enumeration through response time measurement.

### Audit Trail

All authentication events are tracked in the audit log:

- `login` — successful login (with IP)
- `login_failed` — failed attempt (with IP and username)
- `login_disabled` — attempt on a disabled account
- `password_change` — password change
- `user_create` / `user_update` / `user_delete` — account management

## REST API

### Public Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/auth/login` | Login (returns a JWT) |

### Authenticated Endpoints

| Method | Path | Required Role | Description |
|--------|------|---------------|-------------|
| `GET` | `/api/auth/me` | Any role | Current user information |
| `POST` | `/api/auth/change-password` | Any role | Change own password (issues a new token + cookie) |
| `POST` | `/api/auth/logout` | — | Clears the session cookie (idempotent) |
| `POST` | `/api/auth/sse-ticket` | Any role | Issues a single-use ticket to open an SSE stream |

### Admin Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/auth/users` | List users |
| `POST` | `/api/auth/users` | Create a user |
| `PUT` | `/api/auth/users/{id}` | Update a user (role, active, password) |
| `DELETE` | `/api/auth/users/{id}` | Delete a user |

### Login Example

```bash
# Login
curl -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username": "admin", "password": "your_password"}'

# Response: the token is returned in the body (for CLI/API clients)
# AND set in an httpOnly `caelix_session` cookie (used by the SPA).
# {"ok": true, "token": "eyJ...", "user": {"username": "admin", "role": "admin"}}

# Using the token (CLI/API clients)
curl http://localhost:8080/api/state \
  -H "Authorization: Bearer eyJ..."
```

## SSE stream authentication (single-use tickets)

`EventSource` cannot send custom headers. To avoid passing the long-lived JWT in the URL (where it would leak into proxy logs and browser history), SSE streams use **single-use tickets**:

1. The authenticated client calls `POST /api/auth/sse-ticket` → receives `{"ok": true, "ticket": "..."}`
2. It opens the stream with `?ticket=<ticket>` (not `?token=`)
3. The backend consumes the ticket atomically: it is valid **once** and expires after **30 seconds**

```javascript
// 1. Get a ticket
const { ticket } = await (await fetch('/api/auth/sse-ticket', { method: 'POST' })).json()
// 2. Open the stream
const es = new EventSource(`/api/stream?ticket=${encodeURIComponent(ticket)}`)
```

The number of concurrent SSE streams is capped at 5 per IP address.

## User Management (UI)

Administrators can manage users from **Settings > Users**:

- Create new accounts (admin or technician)
- Modify role or active/inactive status
- Reset a user's password
- Delete a user (except yourself)

## Backward Compatibility

The old `CAELIX_UI_TOKEN` mechanism (single shared token) remains functional as a fallback. If a token cannot be decoded as a valid JWT, Caelix checks if it matches `CAELIX_UI_TOKEN` and grants temporary admin access.

!!! warning "Deprecation"
    This mechanism is **deprecated**. A warning is logged on each use. It will be removed in a future version. Migrate to user accounts.
