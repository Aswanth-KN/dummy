# Session Storage Architecture Recommendation

Date: 2026-08-10

## Executive summary

The current application already stores the session cookie in the browser. The
browser receives an opaque `HttpOnly` cookie, while PostgreSQL stores only a
SHA-256 hash of its random session identifier, encrypted Keycloak tokens, and
session expiry metadata.

The recommended architecture is:

> Keep the opaque cookie in the browser and replace persistent PostgreSQL
> session storage with an ephemeral Redis session store.

This provides the best balance of security, performance, horizontal scaling,
logout/revocation support, and refresh-token concurrency control.

If the requirement strictly prohibits all server-side session state, use an
encrypted and authenticated `HttpOnly` client-side session cookie. Do not put
access tokens, refresh tokens, ID tokens, JWTs, or session identifiers in
`localStorage` or `sessionStorage`.

## Requirement clarification

The decision should be based on the following question:

> Does the requirement prohibit persistent relational database storage, or
> does it prohibit all server-side session state?

These are different requirements:

- If persistent PostgreSQL storage is prohibited, use Redis with TTL and, when
  required, disable disk persistence.
- If all server-side state is prohibited, use an encrypted client-side session
  cookie and accept its operational and security trade-offs.
- If the frontend must be able to read the tokens, the proposed design is not
  recommended for this business application.

## Current implementation

The current flow is a Backend for Frontend (BFF) using Authorization Code with
PKCE:

1. FastAPI completes the OAuth/OIDC flow with Keycloak.
2. FastAPI generates a random opaque session identifier.
3. The browser receives the identifier in an `HttpOnly; SameSite=Strict`
   cookie.
4. PostgreSQL stores only the SHA-256 hash of that identifier.
5. Access, refresh, and ID tokens are encrypted with Fernet before being stored
   in PostgreSQL.
6. React sends the cookie automatically using `credentials: 'include'` and
   never reads OAuth tokens.

Relevant files:

- `HW-Backend/unified-portal-shell-api/app/auth/router.py`
- `HW-Backend/unified-portal-shell-api/app/auth/session_store.py`
- `HW-Backend/unified-portal-shell-api/app/auth/login_flow.py`
- `HW_SOW2/host/src/auth/api.js`

Therefore, the application is not currently storing a plaintext browser cookie
in the database. It is storing protected server-side session state.

## Options comparison

| Approach | Token exposure to JavaScript | Revocation | Multi-instance support | Operational cost | Recommendation |
| --- | --- | --- | --- | --- | --- |
| PostgreSQL server-side session | None | Strong | Strong | Moderate, with reads/writes on requests | Safe but heavier than needed |
| Redis server-side session | None | Strong | Strong | Low latency and TTL-native | **Recommended** |
| Encrypted client-side cookie | None | Limited | Stateless | Low server state, higher cookie complexity | Fallback when server state is forbidden |
| Tokens in `localStorage` or `sessionStorage` | Exposed to JavaScript/XSS | Token-dependent | Stateless | Simple but high risk | **Do not use** |
| Tokens in memory or a Web Worker | Partially isolated | Token-dependent | Stateless | More frontend complexity | Only when the BFF is rejected |

## Recommended architecture: opaque cookie and Redis

```text
Browser
  └── opaque random HttpOnly cookie
          |
          v
Shell BFF / FastAPI
  └── Redis session
        ├── encrypted access token
        ├── encrypted refresh token
        ├── encrypted ID token
        ├── idle TTL
        ├── absolute expiry
        └── refresh lock/version
          |
          v
Keycloak and protected APIs
```

### Browser cookie

Use a cookie similar to:

```text
__Host-Http-hw_session=<cryptographically-random-value>
Secure
HttpOnly
SameSite=Strict
Path=/
No Domain attribute
```

The cookie must contain only a meaningless random identifier. It must not
contain user information or plaintext OAuth tokens.

### Redis record

Use a hash of the random identifier as the Redis key. The value should contain:

- encrypted Keycloak token bundle
- Keycloak subject/user identifier
- creation time
- last-used time when needed
- absolute expiration time
- refresh version or locking information

Apply the idle timeout using Redis TTL. Validate the absolute expiration stored
inside the encrypted record so extending the Redis TTL cannot extend the
maximum session lifetime.

### Token refresh

Refresh-token rotation must be serialized. Use an atomic Redis operation,
distributed lock, or compare-and-set version so two simultaneous requests do
not use the same refresh token.

The lock must have a short TTL so a failed FastAPI process cannot leave the
session permanently locked.

### Persistence and availability

If the policy requires sessions to disappear after infrastructure restart,
disable Redis RDB and AOF persistence. If availability across Redis restarts is
required, use replication or a managed Redis service and document that this is
server-side session storage even though it is not the application database.

## Browser-only fallback: encrypted client-side session

Current OAuth browser guidance permits a BFF to use signed and encrypted
client-side session state. In this design, FastAPI encrypts the Keycloak token
bundle and places the ciphertext in an `HttpOnly` cookie. React still cannot
read it.

The cookie must use authenticated encryption, such as AES-GCM, JWE, PASETO
local tokens, or a carefully managed Fernet implementation. A signed-only JWT
is insufficient because signing prevents modification but does not provide
confidentiality.

### Required controls

- `Secure`, `HttpOnly`, and `SameSite=Strict`
- `Path=/` and no `Domain` attribute
- `__Host-Http-` cookie-name prefix
- issued-at and absolute-expiration values inside the encrypted payload
- current and previous encryption keys during controlled key rotation
- Keycloak token revocation during logout
- maximum encrypted-cookie-size validation in automated tests
- no plaintext token logging
- `Cache-Control: no-store` on responses that set or refresh the cookie

### Limitations

#### Cookie size

General-purpose browsers are expected to support approximately 4096 bytes per
cookie, including the name, value, and attributes. Access, refresh, and ID
tokens together may exceed this after encryption and Base64 encoding.

The real Keycloak token bundle must be measured before selecting this design.
Use a conservative application limit of approximately 3.5 KB and fail safely
when it is exceeded. Splitting the session across several cookies increases
complexity and makes atomic updates harder.

#### Refresh concurrency

Two tabs can submit the same encrypted cookie while the access token is
expiring. Both requests may attempt to exchange the same refresh token. When
Keycloak rotates refresh tokens, one request can succeed while the other fails
or overwrites the newer cookie.

A completely stateless backend cannot provide the same reliable refresh lock
as PostgreSQL or Redis.

#### Revocation

FastAPI cannot centrally invalidate one client-side cookie without maintaining
a denylist, which reintroduces server-side state. Logout must revoke the
Keycloak session or tokens and delete the browser cookie. A copied cookie may
remain replayable until Keycloak rejects its tokens or they expire.

#### Request overhead

The complete encrypted token bundle is transmitted with every applicable HTTP
request. The current opaque session identifier is much smaller.

## Approaches that should not be used

### Local storage or session storage

Do not store any of these in `localStorage` or `sessionStorage`:

- access token
- refresh token
- ID token
- JWT
- session identifier
- client secret

Any JavaScript executing in the application origin can read these storage
areas. One XSS vulnerability can therefore extract credentials and use them
outside the user's browser.

### JavaScript-readable authentication cookie

Do not remove `HttpOnly` to let React read the session. React does not need the
credential value; it only needs an authenticated `/auth/me` response.

### Plain signed token bundle

Do not put OAuth tokens inside a signed-only JWT cookie. Its payload is encoded,
not encrypted, and can be decoded by anyone who obtains the cookie.

## Additional improvements for either accepted design

### Centralized CSRF protection

The application currently performs an exact-origin check for login and logout,
but it also contains state-changing role and permission endpoints. Apply a
central CSRF policy to every cookie-authenticated `POST`, `PUT`, `PATCH`, and
`DELETE` request.

Use one of:

- framework-supported anti-forgery tokens
- synchronizer token pattern
- correctly implemented double-submit cookie
- a required custom request header combined with strict CORS and exact origin
  validation

SameSite cookies are defense in depth and should not be the only centralized
control when same-site sibling applications exist.

### Same-origin deployment

Serve the frontend and BFF through one HTTPS browser origin where possible:

```text
https://portal.example.com/
https://portal.example.com/api/v1/...
```

This simplifies cookies and CORS and avoids cross-site browser behavior.

### Cookie lifecycle

- Rotate the session identifier after authentication and privilege changes.
- Keep idle and absolute timeouts.
- Delete the cookie during logout.
- Revoke the Keycloak session or refresh token during logout.
- Clear invalid cookies after decryption, validation, or refresh failure.

## Decision

For this Honeywell business portal, choose:

> **Opaque secure browser cookie plus an ephemeral Redis session store.**

This removes persistent PostgreSQL session storage while preserving the
security benefits of the existing BFF: tokens remain unavailable to frontend
JavaScript, sessions can be revoked, refresh-token rotation is serialized, and
multiple FastAPI instances can serve the same user.

Use the encrypted client-side cookie only if the TL confirms that every form of
server-side session storage is prohibited and explicitly accepts the cookie
size, revocation, and concurrency limitations.

## Acceptance criteria

Before considering the change complete:

1. Confirm the exact meaning of "not stored in DB" with the TL/security owner.
2. Confirm that OAuth tokens never enter JavaScript-accessible storage.
3. Test login, session restoration, expiration, refresh, logout, and revoked
   Keycloak sessions.
4. Test simultaneous refresh requests from multiple tabs.
5. Test multiple FastAPI replicas.
6. Verify `Secure`, `HttpOnly`, `SameSite=Strict`, `Path=/`, and the cookie-name
   prefix in a real browser.
7. Add centralized CSRF tests for every state-changing endpoint.
8. For client-side sessions, test the real encrypted Keycloak token size and
   reject oversized cookies safely.
9. Confirm encryption-key rotation and incident-driven mass session
   invalidation behavior.
10. Perform security review before production deployment.

## Primary references

- [RFC 10017: OAuth 2.0 for Browser-Based Applications](https://auth48-transition.rfc-editor.org/authors/rfc10017.html)
- [RFC 9700: Best Current Practice for OAuth 2.0 Security](https://www.rfc-editor.org/rfc/rfc9700.html)
- [RFC 6265: HTTP State Management Mechanism](https://www.rfc-editor.org/rfc/rfc6265.html)
- [OWASP Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)
