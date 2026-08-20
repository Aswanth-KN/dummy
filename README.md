# IPS FusionHub Authentication

This repository contains several FastAPI services and a local Keycloak installation. Authentication is centralized around the shell API, which acts as a backend-for-frontend (BFF) for browser sign-in. The admin API can validate bearer access tokens directly and can also resolve the shell session. The agent API uses machine-to-machine credentials for Databricks.

## Architecture

```text
Browser / SPA
    |
    | browser redirects and credentialed requests
    v
Shell API (:8000)
    |  Authorization Code + PKCE
    |  confidential client: session
    |  server-side SQLite session store
    v
Keycloak (:8080, realm: honeywell)

Shell session or bearer access token
    |
    +--> Admin API
    |       verifies JWT with Keycloak JWKS
    |       checks live realm roles through Keycloak Admin API
    |
    +--> Other resource APIs

Agent API
    |
    +--> Databricks SDK OAuth client credentials
```

The browser does not receive the Keycloak authorization code, access token, refresh token, ID token, or client secret. It receives an opaque session cookie and a separate CSRF cookie.

## Browser Login Flow

1. The SPA calls `GET /api/v1/auth/config` to obtain public OIDC settings such as the issuer, authorization endpoint, client id, scopes, and callback URLs.
2. The SPA navigates to `GET /api/v1/auth/login`. This is a top-level redirect, not an XHR.
3. The shell API creates a one-time transaction containing:
   - a PKCE verifier;
   - an OIDC nonce;
   - an opaque state value;
   - a browser-binding transaction secret;
   - a validated return path.
4. The shell stores the transaction in SQLite and sends the browser to Keycloak with Authorization Code + PKCE (`S256`).
5. Keycloak authenticates the user and redirects to `/api/v1/auth/callback` with `code` and `state`.
6. The shell consumes the transaction exactly once, verifies the transaction cookie, exchanges the code server-to-server, and validates the ID token signature, issuer, audience, required claims, and nonce.
7. The shell stores the access, refresh, and ID tokens in the server-side session database. It returns only cookies and redirects the browser back to the SPA.

### Session cookies

| Cookie | Purpose | Browser-readable |
| --- | --- | --- |
| `__Host-session` | Opaque session id. The server uses it to find tokens in SQLite. | No (`HttpOnly`) |
| `__Host-txn` | Short-lived binding for an in-progress login transaction. | No (`HttpOnly`) |
| `__Host-csrf` | Per-session double-submit CSRF token. | Yes, so the SPA can echo it in `X-CSRF-Token` |

Cookie flags and names are configured centrally. In production, use secure cookies and an appropriate same-site policy.

## Session Requests and Refresh

- `GET /api/v1/auth/token` returns the current session state. Signed out is a normal `200` response with `authenticated: false`.
- `GET /api/v1/auth/me` returns the authenticated identity and current realm roles.
- `POST /api/v1/auth/refresh` rotates the token set immediately.
- `POST /api/v1/auth/logout` revokes the refresh token when possible, deletes the local session, clears cookies, and returns a Keycloak logout URL for the SPA to navigate to.

For authenticated requests, the shell API:

1. Reads the opaque session id from the cookie.
2. Rejects missing, idle-expired, or absolute-expired sessions.
3. Enforces origin and double-submit CSRF checks on unsafe methods.
4. Refreshes the access token ahead of expiry.
5. Verifies the access token before creating the current user.
6. Slides the idle timeout by reissuing the cookies.

Refresh-token rotation is serialized per session. This is required because Keycloak is configured with zero refresh-token reuse; two concurrent refreshes must not submit the same refresh token.

## Authorization and Roles

Authentication proves the identity. Authorization is performed separately.

- `CurrentUser` in the shell API is defined in `ips_fusionhub_shell_api/app/auth/deps.py`.
- `CurrentUser` in the admin API is defined in `ips_fusionhub_admin_api/app/auth/deps.py`.
- `require_realm_role("role-name")` creates a FastAPI dependency for role-protected routes.
- Effective realm roles are read from Keycloak's Admin REST API and cached briefly. They are not trusted solely from the token because a role can be removed after the token was minted.
- Role changes made through the service invalidate the relevant local role cache. Changes made directly in Keycloak become visible after the cache TTL.

The admin API's domain routers attach these dependencies to protected operations. The API's own `admin_role` setting defines the realm role allowed to administer users, roles, modules, and permissions.

## Admin API Authentication Paths

The admin API supports two request styles in `ips_fusionhub_admin_api/app/auth/deps.py`:

### Bearer access token

A caller sends:

```http
Authorization: Bearer <access-token>
```

`HTTPBearer` extracts the credential and `app/auth/token.py` verifies it. The verifier checks:

- the signing algorithm;
- the `kid` and Keycloak public signing key;
- required `exp`, `iat`, `iss`, `sub`, and `aud` claims;
- the configured issuer and audience;
- the access-token type;
- optional allowed client ids.

### Shell session cookie

When no bearer header is present, the admin API forwards the caller's `Cookie` header to the shell API's `/api/v1/auth/me` endpoint. The returned profile becomes the admin API's `TokenUser` representation. This keeps browser session credentials in the shell while allowing admin routes to use the same `CurrentUser` and role dependencies.

## Keycloak and Token Verification Code Map

### Shell API: browser-facing BFF

| Responsibility | Code |
| --- | --- |
| Route registration and `/api/v1` prefix | `ips_fusionhub_shell_api/app/main.py`, `ips_fusionhub_shell_api/app/api.py` |
| Login, callback, session state, refresh, logout | `ips_fusionhub_shell_api/app/auth/router.py` |
| Authorization URL, PKCE, code exchange, refresh, revocation, ID-token validation | `ips_fusionhub_shell_api/app/auth/oidc.py` |
| Session lookup, transaction storage, expiry, refresh locking | `ips_fusionhub_shell_api/app/auth/session_store.py` |
| Cookie names and security flags | `ips_fusionhub_shell_api/app/auth/cookies.py` |
| Origin and double-submit CSRF checks | `ips_fusionhub_shell_api/app/auth/csrf.py` |
| Session-to-user dependency and automatic refresh | `ips_fusionhub_shell_api/app/auth/deps.py` |
| JWT claims-to-user mapping and access-token verification | `ips_fusionhub_shell_api/app/auth/token.py` |
| Live Keycloak realm-role lookup | `ips_fusionhub_shell_api/app/auth/live_roles.py` |
| OIDC URLs, cookie settings, lifetimes, issuer, audience | `ips_fusionhub_shell_api/app/core/config.py` |
| Startup session database and JWKS initialization | `ips_fusionhub_shell_api/app/main.py` |

### Admin API: resource server and admin operations

| Responsibility | Code |
| --- | --- |
| Bearer or shell-cookie authentication dependency | `ips_fusionhub_admin_api/app/auth/deps.py` |
| JWT signature and claim validation | `ips_fusionhub_admin_api/app/auth/token.py` |
| Cached Keycloak public signing keys | `ips_fusionhub_admin_api/app/auth/jwks.py` |
| Live realm-role lookup and cache invalidation | `ips_fusionhub_admin_api/app/auth/live_roles.py` |
| Keycloak Admin REST client and service-account token handling | `ips_fusionhub_admin_api/app/core/keycloak_admin.py` |
| Protected route composition | `ips_fusionhub_admin_api/app/api.py` |
| Admin API settings, including shell URL and token validation | `ips_fusionhub_admin_api/app/core/config.py` |

## Keycloak Realm Setup

The local realm setup is defined by `ips_fusionhub_shell_api/keycloak/setup_local_dev.py`. It is idempotent and provisions the application clients, login flow, local developer user, role assignments, redirect URIs, token settings, and service-account permissions.

The intended clients are:

- `session`: the confidential shell/BFF client. The shell owns the code exchange and keeps its secret server-side. PKCE is also required.
- `users`: a confidential service-account client used by backend code for Keycloak Admin REST operations and development login-token creation. It is not a browser login client.

Redirect URI configuration must match the shell settings exactly. The SPA callback is derived from `FRONTEND_URL` and `AUTH_CALLBACK_PATH`; the API URL is not the browser callback.

On Windows, `scripts/windows/env.cmd` supplies local defaults and discovers the Keycloak installation. `scripts/windows/start.cmd` starts the local services and runs the optional realm setup when the setup script is available.

## Development-Only Email Login

When enabled by `ENABLE_DEV_LOGIN`, the shell exposes `POST /api/v1/auth/login-hint`. The backend uses the `users` service account to obtain a one-time Keycloak login token for a registered email address. The browser then enters the normal authorization-code flow with that hint.

This is not a separate authentication mechanism or a token bypass: it still creates a normal Keycloak SSO session and passes through the same callback, ID-token validation, server-side session creation, CSRF protection, refresh, and logout logic. It must remain disabled outside local development.

## Agent API Authentication

The agent API does not implement browser user authentication. Its downstream Databricks client is configured with OAuth machine-to-machine credentials:

- settings: `ips_fusionhub_agent_api/app/core/config.py`;
- client construction: `ips_fusionhub_agent_api/app/services/databricks_client.py`;
- token refresh: handled by the Databricks SDK.

If the Databricks configuration is incomplete, agent routes return service-unavailable errors rather than attempting to authenticate with a user session.

## Important Configuration

Authentication settings are loaded from each service's `.env` file through its `app/core/config.py`. Keep secrets out of source control. The important relationships are:

- `KEYCLOAK_SERVER_URL` + `KEYCLOAK_REALM` define the issuer and OIDC endpoints.
- `KEYCLOAK_CLIENT_ID=session` and `KEYCLOAK_CLIENT_SECRET` identify the shell's confidential client.
- `KEYCLOAK_AUDIENCE` must match the audience mapper configured in Keycloak.
- `FRONTEND_BASE_URL` and `AUTH_CALLBACK_PATH` must match the Keycloak redirect URI exactly.
- `FRONTEND_POST_LOGOUT_PATH` must match the registered post-logout redirect URI.
- `CORS_ORIGINS` must contain every frontend origin that sends credentialed requests.
- `SESSION_DB_PATH`, idle timeout, absolute timeout, and refresh leeway control local session behavior.
- `ALLOWED_CLIENTS`, when set, restricts which `azp` values may use bearer tokens.

For troubleshooting, compare the effective service settings, Keycloak client redirect URIs/web origins, issuer, audience mapper, and the browser's actual frontend port. A frontend fallback from port `3000` to `3001`, `3002`, or `3003` must be reflected in the configured CORS and Keycloak web-origin lists.
