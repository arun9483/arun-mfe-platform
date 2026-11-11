# MVP-2: Single sign-on (POC)

MVP-2 adds users to the shop: one sign-in with an identity provider (IdP), every unit protected, and the
user's identity proven on every hop, with no session held by any server. It builds on [MVP-1](mvp.md),
frozen at tag `mvp-1`, and on the [vision](vision.md)'s "Users and access" requirements.

- **Additive and off by default.** With `AUTH_ENABLED=false` (the default) every unit behaves exactly as in
  MVP-1, and MVP-1's success criteria 1–14 still pass.
- **Docker first.** MVP-2 runs on Docker Desktop, from `shop-mfe-platform`. Staging comes later.
- **IdP agnostic.** Keycloak is the IdP. Only one adapter in each unit and the IdP's own settings know it;
  everything else is standard OpenID Connect (OIDC) and OAuth 2.0.

Decisions and the alternatives weighed are in the ADRs:
[0002 authentication architecture](adr/0002-authentication-architecture.md),
[0003 session cookies](adr/0003-session-cookies.md),
[0004 IdP adapter](adr/0004-idp-adapter.md),
[0005 backend-to-backend access](adr/0005-backend-to-backend-access.md).

## Scope

| In MVP-2 | Not in MVP-2 |
|---|---|
| Sign-in, refresh and sign-out, owned by the Shell | A second app on another origin to prove single sign-on across apps |
| Every Shell and MFE route protected | Public pages |
| Checks in every FES and backend, failing closed | Entitlement (which MFEs a user may see): later |
| `/api/<unit>/me` in every unit | Staging and production |
| Backend-to-backend calls with the caller's own identity | A branded sign-in page (Keycloak's default theme) |
| `/health/units*` and `/version/units*` behind an operator token | |
| Standalone MFEs with a development-only bypass | |

## Terms

| Term | What it is | Issued to | Kept in | Used for |
|---|---|---|---|---|
| **Lightweight access token** | The signed-in user's access token: a signed JWT with only `sub`, `sid`, `iss`, `exp`, `iat`, `jti`, `azp` and `aud`; no profile, no roles | `shell-fes` | Cookie `shop_at` | Browser → FES → its backend: which user the request is for |
| **Refresh token** | Gets a new lightweight access token while the IdP's session lives | `shell-fes` | Cookie `shop_rt` | Shell FES → IdP only |
| **ID token** | Tells the Shell FES who signed in. Never a bearer for an API | `shell-fes` | Cookie `shop_id` | Checked at sign-in; `id_token_hint` at sign-out |
| **Service access token** | A backend's, or the operator caller's, own token from the client-credentials flow; a full JWT with its roles | Each backend's client, `shop-ops` | The caller's memory, reused until shortly before expiry | Backend → IdP, backend → backend, operator → `/health/units*` |

`sub` is the user's permanent ID and the only user key units store. `sid` names one sign-in session; a user
signed in on two devices has two.

## Architecture

```
Browser (Shell + MFE clients; no token readable by page code)
  │ cookies: shop_at (Path=/) · shop_rt (Path=/auth) · shop_id (Path=/auth/logout)
  ├──── sign-in / sign-out redirects ─────────────────► IdP: Keycloak, https://shop.test:8443, realm `shop`
  ▼                                                      users, roles, the sign-in session, its admin API
nginx (HTTPS, cache)
  ▼
shop-router   routing; checks the operator token on /health/units*, /version/units*
  ├─► Shell FES    /auth/login · /auth/callback · /auth/refresh · /auth/logout   (client shell-fes)
  │                + the checks every FES makes
  ├─► Catalog FES  check shop_at · /userinfo gate · forward
  └─► Cart FES     check shop_at · /userinfo gate · forward
         │ Authorization: Bearer <lightweight access token>
         ▼
  Shell, Catalog and Cart backends
         check the lightweight access token → sub
         │ Authorization: Bearer <service access token>
         ├─► IdP, through the IdP adapter: user and this unit's roles by sub
         └─► another unit's backend: + sub; the provider checks the caller and its API role

Standalone: browser → Shell stub → one MFE's FES → its backend, with AUTH_DEV_BYPASS (a fixed dev user, no IdP)
New Relic: setUserId(sub) in the Shell client; setUserID(sub) in every FES and backend
```

The router stays unaware of users: `/auth/*` reaches the Shell FES as "everything else" does in MVP-1. The
IdP is not behind the router; nginx publishes it on its own port, so it is a separate origin.

## Ownership

| Piece | Repository | Owns |
|---|---|---|
| Keycloak, its Postgres database, realm basics (realm, lifetimes, test users), the `shop-ops` client, the settings importer | `shop-mfe-platform` (`docker/`, `keycloak/`) | Running the IdP and applying every repository's settings |
| `shell-fes` and `shell-backend` clients, Shell roles | `shop-shell` (`keycloak/`) | The sign-in client and Shell roles |
| `catalog-backend` client, Catalog roles and API roles, its grants to consumers | `shop-catalog` (`keycloak/`) | Who may use Catalog, and for what |
| `cart-backend` client, Cart roles and API roles, its grants | `shop-cart` (`keycloak/`) | The same for Cart |
| `shop-router` client: the audience and holder of the `ops` role | `shop-router` (`keycloak/`) | Who may read unit health and versions |
| Sign-in lifecycle and all three cookies | `shop-shell` (Shell FES) | The only place tokens are received, refreshed or cleared |
| FES and backend checks, `/me`, the IdP adapter | `server-common/` in each unit, copied as in MVP-1 | Checks, lookups, the only IdP-specific code |
| Vision, this document, ADRs | `arun-mfe-platform` | Requirements and decisions |

**Settings per repository.** Each repository keeps `keycloak/<repository>.yaml` (see the appendix).
`shop-mfe-platform` runs [`keycloak-config-cli`](https://github.com/adorsys/keycloak-config-cli) at start: the
realm basics first, then every checked-out repository's file, found the way the Docker stack already finds
sibling repositories. A unit adding a role changes only its own file. **The provider grants its own
roles**: a Catalog role given to a user or to another unit's service account is listed in `shop-catalog`'s
file, so each file is the full list of who may use that unit.

## Identity provider settings

Realm `shop` on Keycloak 26 (exact version pinned in phase 1).

| Client | Owner | Type | Settings |
|---|---|---|---|
| `shell-fes` | `shop-shell` | Confidential; authorization code with PKCE | Redirect `https://shop.test/auth/callback` (and `http://localhost:3000/auth/callback` for local e2e); "Always use lightweight access token" on |
| `shell-backend`, `catalog-backend`, `cart-backend` | Each unit | Confidential; service account only, never signs a user in | Service account holds `realm-management` → `view-users`. Defines the unit's user roles (`catalog-viewer`, …) and API roles (`catalog-api:read-products`, …) |
| `shop-router` | `shop-router` | Bearer-only: no secret, never signs in | Defines the `ops` role |
| `shop-ops` | `shop-mfe-platform` | Confidential; service account only | Granted `shop-router` → `ops` |

FESes have no client of their own: only the Shell FES signs users in, with `shell-fes`.

**Audience without editing other repositories.** Keycloak's *audience resolve* mapper adds a client to `aud`
when the user holds one of that client's roles, so a user with a Catalog role gets `aud: catalog-backend`
without `shell-fes` naming Catalog. Its "Add to lightweight access token" switch is on.

| Setting | Value | Why |
|---|---|---|
| Access token lifespan | 5 min | Bounds how long a signed-out user's token still passes a local check |
| Session idle / maximum | 30 min / 10 h | Refresh stops working when the session ends |
| Refresh token rotation | Off | Several tabs may refresh at once |
| Test users | `alice` (every unit's roles), `bob` (no Catalog role, to test `403`); passwords in gitignored `.env` | Repeatable e2e |

## Tokens and cookies

| Cookie | Holds | Attributes | Sent to | Read by |
|---|---|---|---|---|
| `shop_at` | Lightweight access token, not encrypted | `HttpOnly; Secure; SameSite=Lax; Path=/` | Every request | Every FES, which checks it with `jose`; no shared secret |
| `shop_rt` | Refresh token, encrypted | `HttpOnly; Secure; SameSite=Lax; Path=/auth` | `/auth/*` only | Shell FES |
| `shop_id` | ID token, encrypted | `HttpOnly; Secure; SameSite=Lax; Path=/auth/logout` | `/auth/logout` only | Shell FES |

- **`Path` decides which server gets a cookie; `HttpOnly` hides it from page code.** Page loads, `entry.js`,
  chunks and API calls carry only `shop_at`. The refresh and ID tokens reach only the Shell FES, and only on
  its `/auth/*` routes.
- **Set together, cleared together.** Sign-in and refresh set all three; sign-out and a failed refresh clear
  all three. A cookie is cleared only by a response naming the same `Path`, so each is cleared with its own
  `Path` and `Max-Age=0`.
- **`shop_at` names `Path=/` explicitly.** Cookies set by `/auth/*` responses otherwise default to `Path=/auth`.
- **Encryption.** The Shell FES encrypts `shop_rt` and `shop_id` with `AUTH_COOKIE_KEY`, which no other unit
  holds. `shop_at` is not encrypted: it is a signed token any FES can check, carrying no personal data beyond
  `sub`.
- **An exception to MVP-1.** MVP-1 says no server sets a cookie. These three cookies are the exception. They
  hold no server state, so any pod still serves any request.

## Flows

### Sign-in
1. A protected request without a valid `shop_at`: a page is redirected to `/auth/login?returnTo=<path>`; an
   API call or `entry.js` gets `401` problem+json.
2. The Shell FES redirects to the IdP's authorization endpoint (code with PKCE, `state`, `nonce`), keeping
   `state`, `nonce`, the PKCE verifier and `returnTo` in a short-lived encrypted cookie named for this attempt,
   `shop_tx_<state>`, on `Path=/auth/callback`. Each attempt has its own, so sign-ins started in several tabs
   do not overwrite each other.
3. The user signs in on the IdP's page; the IdP starts its session.
4. The IdP redirects to `/auth/callback?code=…`. The Shell FES exchanges the code with `shell-fes`'s
   credentials and checks the ID token (signature, `iss`, `aud`, `nonce`).
5. It sets the three cookies, clears `shop_tx_<state>` and redirects to `returnTo`, which must be a path on the
   shop's origin.
6. A callback whose attempt cookie is missing or does not match: if the browser already holds a valid
   `shop_at`, because another tab signed in, it redirects to `returnTo`; otherwise it starts a new sign-in.

### Page and MFE load
1. The Shell's HTML, every `/mfe/<name>/entry.js` and every `/api/*` need a valid `shop_at`. Hashed files
   hold no data and stay public and edge-cacheable.
2. The Shell client calls `/api/shell/me`, shows the user, and calls `newrelic.setUserId(sub)`.
3. On `/catalog`, `import('/mfe/catalog/entry.js')` carries `shop_at`; Catalog's FES checks it.

### API call
For example `GET /api/catalog/products`:
1. **FES.** Checks `shop_at` with `jose`: signature, `iss`, `exp`, `azp = shell-fes`, and that it is an access
   token (`idp.isAccessToken`). Then the `/userinfo` gate, cached per `sid` for about 30 s: the session is
   live and the user holds one of this unit's roles (`idp.rolesFromClaims`). Then `newrelic.setUserID(sub)`.
2. The FES calls its backend with `Authorization: Bearer <lightweight access token>`.
3. **Backend.** Checks the token with `jose`: signature, `iss`, `exp`, its own name in `aud`, and the token
   type. Takes `sub`, calls `newrelic.setUserID(sub)`, then `idp.getUser(sub)` and
   `idp.getUnitRoles(sub, 'catalog')` with its own service access token, cached per `sub` for 30–60 s. A
   disabled user or a missing role is refused. The request is handled with `sub` as the user key.
4. The backend never sends the lightweight access token anywhere.

### Refresh
1. A FES finds `shop_at` expired and answers `401` problem+json.
2. The client takes the lock `shop-auth-refresh` with the [Web Locks API](https://w3c.github.io/web-locks/)
   (`navigator.locks.request`), which every tab of the shop shares. Inside the lock it first retries the
   call: another tab may have refreshed already, and cookies are shared by every tab.
3. Only if the retry still gets `401` does it call `POST /auth/refresh`. The Shell FES refreshes with
   `shell-fes`'s credentials (`scope=openid`, so a new ID token comes too) and sets the three cookies. The
   client releases the lock and retries its call.
4. If refresh fails, because the IdP's session has ended, the three cookies are cleared and the client goes
   to `/auth/login?returnTo=<path>`.

Each client carries this small `401` → lock → retry → refresh → retry wrapper itself. It shares only a URL
and a lock name with the Shell, never code, as the vision requires. No tab refreshes on a timer: refresh
happens only when a call is refused, so hidden tabs do nothing.

### Sign-out
1. The Shell client calls `newrelic.setUserId(null)`, then submits a `POST` form to `/auth/logout`, so the
   browser follows the redirects that come back.
2. The Shell FES reads the ID token from `shop_id`, clears the three cookies and redirects to the IdP's
   end-session endpoint with `id_token_hint` and `post_logout_redirect_uri`, so no confirmation page is shown.
   Without `shop_id` it sends `client_id` instead, and the IdP asks the user to confirm.
3. The IdP ends its session.
4. Elsewhere: refresh fails at once; a FES's `/userinfo` gate fails within its cache time (about 30 s); a
   backend's local check passes until the lightweight access token expires (at most 5 min). This is the
   accepted price of checks that need no call per request.

### Several tabs
Every tab of one browser profile shares the cookies, so all tabs always hold the same tokens. The Shell keeps
the tabs in step with a [`BroadcastChannel`](https://html.spec.whatwg.org/multipage/web-messaging.html#broadcasting-to-other-browsing-contexts)
named `shop-auth`, and remembers the `sub` it shows.

| Event in one tab | Message | Every other tab's Shell |
|---|---|---|
| Sign-out | `{ type: 'signed-out' }` | Navigates to `/auth/login?returnTo=<its path>` at once, so no tab keeps showing the user's data |
| Sign-in | `{ type: 'signed-in', sub }` | On the sign-in page: reloads to its `returnTo`. Showing another `sub`: reloads (see below) |
| Refresh | None: the Web Lock above coordinates it | — |

- **Another user in another tab.** Tab A shows alice; in tab B the user signs out and signs in as bob. Tab A's
  next request would run as bob while its screen still shows alice. So a Shell that learns of another `sub`,
  from a `signed-in` message or from `/api/shell/me`, reloads the page. MFEs live in the Shell's page, so the
  reload covers them; they need no code for it.
- **When a tab becomes visible** (`visibilitychange`), the Shell calls `/api/shell/me`. This catches what no
  message announces: the IdP's session expiring, an administrator ending it, or a sign-out from another app or
  browser. A `401` leads to the usual refresh, and a failed refresh to sign-in.
- **Sign-ins started in two tabs** each keep their own attempt cookie (see Sign-in), and a callback that finds
  the user already signed in goes to its `returnTo`.
- **A refresh in flight during a sign-out** in another tab may write new cookies for the ended session. Their
  lightweight access token lasts at most 5 min, the next refresh fails, and the `signed-out` message has
  already sent that tab to sign-in. Accepted.

Messages carry only a type and `sub`, never a token. `BroadcastChannel` and Web Locks are standard and in every
current browser (Chrome 69, Edge 79, Firefox 96, Safari 15.4 and later); Web Locks needs a secure context, which
`https://shop.test` and `http://localhost` are. Without them the design stays correct, only slower to react:
without Web Locks each tab refreshes on its own; without `BroadcastChannel` other tabs notice a sign-out when
they become visible or make their next call.

### `/api/<unit>/me`
Answered by the unit's backend from `idp.getUser(sub)` and `idp.getUnitRoles(sub, unit)`. Field names follow
[OpenID Connect Core §5.1](https://openid.net/specs/openid-connect-core-1_0.html#StandardClaims), plus
`roles`, which holds **only this unit's roles**:

```json
{
  "sub": "3f2a9c1e-…",
  "preferred_username": "alice",
  "name": "Alice Test",
  "given_name": "Alice",
  "family_name": "Test",
  "email": "alice@example.com",
  "email_verified": true,
  "locale": "en",
  "roles": ["catalog-viewer", "catalog-editor"]
}
```

`email` and `locale` appear only where the unit's UI needs them. Never returned: tokens, `sid`, other units'
roles, raw IdP attributes. A user who is disabled gets `401`, never `200` with `enabled: false`.

Each client calls only its own unit's `/me`, and only when its UI needs the user. Only the Shell calls
`newrelic.setUserId`.

### Backend to backend
For example, Cart's backend reading product names from Catalog's backend:
1. Cart's backend gets its service access token (client credentials, reused until shortly before expiry).
2. It calls Catalog's backend with `Authorization: Bearer <service access token>` and the user's `sub` in the
   request.
3. Catalog's backend checks the token with `jose`: its own name in `aud`, and the caller's API role
   (`catalog-api:read-products`). Anything else is refused.
4. Per endpoint, Catalog decides whether it trusts the caller for that `sub` or also checks the user's own
   Catalog roles with `idp.getUnitRoles`. Its API documentation says which.
5. The grant is in Catalog's `keycloak/` file, so Catalog always knows its consumers.

A client never calls another unit's FES. Data from another domain is combined on the backend, as the vision
requires. Links to another unit's pages stay plain URLs; the target unit checks its own access. Events between
MFEs (the vision's event service) carry only IDs.

### Unit health and version for operators
1. `shop-ops` gets a service access token holding `shop-router` → `ops`.
2. It calls `/health/units*` or `/version/units*` with `Authorization: Bearer <service access token>`.
3. The router checks it with `jose`: `shop-router` in `aud` and the `ops` role; otherwise `401` or `403`.
4. Each server's own `/health/live`, `/health/ready` and `/version` stay public; Docker's health checks use them.

`HEALTH_UNITS_AUTH` is off by default, so MVP-1's criterion 11 still passes.

## Checks per layer

| Layer | Checks | Decides | Fails with |
|---|---|---|---|
| Browser | Nothing enforced | What to show, from `/me` | — |
| Router | Only `/health/units*`, `/version/units*`: `aud`, `ops` role | Operator access | `401`, `403` |
| FES | `shop_at`: signature, `iss`, `exp`, `azp`, token type; `/userinfo` gate | Signed in? Allowed into this unit? | `401`, `403` |
| Backend | Lightweight access token: signature, `iss`, `exp`, `aud`, token type; then `enabled` and roles by `sub`. Service access token: `aud`, API role | Everything about the data: ownership, actions | `401`, `403` |

**Fail closed.** If the IdP cannot be reached, a check answers `503`, never "allow". Every refusal is
`application/problem+json` with `traceId` and `Cache-Control: no-store`.

## IdP adapter

Sign-in, tokens and cookies use only standard OIDC and OAuth through `openid-client` and `jose`, configured
from `OIDC_ISSUER_URL` by OIDC Discovery. The only IdP-specific code is one adapter in `server-common/`,
chosen by `IDP_ADAPTER`. See [ADR 0004](adr/0004-idp-adapter.md).

```ts
export interface IdpUser {
  sub: string;
  preferred_username: string;
  name?: string;
  given_name?: string;
  family_name?: string;
  email?: string;
  email_verified?: boolean;
  locale?: string;
  enabled: boolean;
}

export interface IdpAdapter {
  /** Backend: the user's profile by sub, using the backend's service access token. */
  getUser(sub: string): Promise<IdpUser | undefined>;
  /** Backend: the user's effective roles in one unit, by sub. */
  getUnitRoles(sub: string, unit: string): Promise<string[]>;
  /** FES: this unit's roles from /userinfo claims; each IdP shapes role claims its own way. */
  rolesFromClaims(claims: Record<string, unknown>, unit: string): string[];
  /** FES and backend: true only for an access token, never an ID or refresh token. */
  isAccessToken(header: Record<string, unknown>, claims: Record<string, unknown>): boolean;
}
```

| Concern | Standard, any IdP | Keycloak adapter |
|---|---|---|
| Sign-in, refresh, sign-out, discovery | OIDC Core, RP-Initiated Logout, Discovery | — |
| Token checks | JWT and JWKS: `iss`, `exp`, `aud`, `azp` | — |
| Service access tokens | OAuth 2.0 client credentials | — |
| Token type | RFC 9068 header `typ: at+jwt`, where the IdP uses it | Claim `typ` is `Bearer` (`ID` for an ID token, `Refresh` for a refresh token) |
| Roles from `/userinfo` | — | `resource_access.<unit>-backend.roles` |
| User by `sub` | — | `GET /admin/realms/shop/users/<sub>` |
| Roles by `sub` | — | `GET /admin/realms/shop/users/<sub>/role-mappings/clients/<client-uuid>/composite` |

- Nothing outside the adapter calls an IdP-specific endpoint or reads an IdP-specific claim.
- The adapter caches per `sub` and fails closed.
- Every adapter passes the same contract tests; the Keycloak adapter also runs against a real Keycloak.
- Another IdP needs a new adapter and its own settings files, with no change to flows, cookies or checks. It
  must allow refresh tokens to be used more than once (or rotate with a reuse window), support RP-initiated
  logout, and issue a small access token with `sub` and an `aud` per backend.

## Packages

All open source.

| Package | Used by | For |
|---|---|---|
| [`openid-client`](https://www.npmjs.com/package/openid-client) | Shell FES (sign-in, refresh, sign-out); every FES (`/userinfo`); every backend (client credentials) | OIDC and OAuth flows. OpenID-certified |
| [`jose`](https://www.npmjs.com/package/jose) | Every FES and backend, the router | Checking tokens against the IdP's cached public keys; encrypting `shop_rt` and `shop_id` |
| [`keycloak-config-cli`](https://github.com/adorsys/keycloak-config-cli) | `shop-mfe-platform` (a container) | Applying every repository's IdP settings |

IdP admin calls are plain `fetch`. Not used: `express-openid-connect` (one cookie only, see
[ADR 0003](adr/0003-session-cookies.md)), `keycloak-connect` (deprecated by Keycloak) and `keycloak-js`
(keeps tokens in the browser).

## API conventions

MVP-1's conventions still apply: trace context, `traceresponse`, problem+json with `traceId`, health and
version. MVP-2 adds:

- **`401`** when there is no valid credential; **`403`** when the credential is valid but lacks the role;
  **`503`** when the IdP cannot be reached. All are problem+json with `traceId` and `no-store`.
- **`/api/<unit>/me`**: as above, `200` with `Cache-Control: no-store` and `Vary: Cookie`.
- **`/auth/refresh` and `/auth/logout`** accept only `POST`, check that `Origin` is the shop's own origin, and
  rely on `SameSite=Lax` against cross-site requests, so another site cannot refresh or end a user's session.
- **`/auth/login`** and **`/auth/callback`** accept a `returnTo` that is a path on the shop's own origin, never a
  full URL.

## Standalone mode

Standalone MFEs keep the Shell stub as in MVP-1, with a **development-only bypass** instead of the IdP:

1. The MFE's FES and backend run with `AUTH_DEV_BYPASS=true`; no IdP, no sign-in, no cookies.
2. The FES skips the cookie and `/userinfo`, acting as the user from `AUTH_DEV_USER_SUB` and
   `AUTH_DEV_USER_ROLES`, and calls its backend without a token.
3. The backend skips its token check and the IdP; `/api/<unit>/me` returns that user with this unit's
   configured roles. Everything after the check runs as usual, with `sub` as the user key.
4. Changing `AUTH_DEV_USER_ROLES` exercises role-based UI and `403`s without an IdP.

**Guard rails.** A server refuses to start with the bypass on when `NODE_ENV=production` or on Cloud Run
(`K_SERVICE` is set). It logs a warning at start and adds `X-Auth-Dev-Bypass: true` to every response.
Deploy builds never set it. Sign-in, refresh, sign-out and real IdP lookups are tested only with the whole shop.

## Monitoring

| Where | Call | When |
|---|---|---|
| Shell client | `newrelic.setUserId(sub)`, `setUserId(null)` | After `/api/shell/me`; at sign-out |
| Every FES and backend | `newrelic.setUserID(sub)` | After its token check |

Browser events and server transactions then carry `enduser.id`, so one user can be followed from browser to
backend. Only `sub` is sent, never an email or username. This is the one call the Shell makes on its
`newrelic` object; MFEs still make none.

## Configuration

Every switch is off by default.

| Variable | Used by | Default |
|---|---|---|
| `AUTH_ENABLED` | every FES and backend | `false`: MVP-1 behaviour |
| `HEALTH_UNITS_AUTH` | router | `false` |
| `OIDC_ISSUER_URL` | every FES and backend, the router | `https://shop.test:8443/realms/shop` |
| `OIDC_CLIENT_ID`, `OIDC_CLIENT_SECRET` | Shell FES (`shell-fes`), each backend (its own client), the operator caller (`shop-ops`) | per developer, in `.env` |
| `AUTH_COOKIE_KEY` | Shell FES | per developer; encrypts `shop_rt` and `shop_id` |
| `IDP_ADAPTER` | every FES and backend | `keycloak` |
| `AUTH_DEV_BYPASS`, `AUTH_DEV_USER_SUB`, `AUTH_DEV_USER_ROLES` | an MFE's FES and backend, standalone only | unset |
| `KC_HOSTNAME`, `KC_HOSTNAME_BACKCHANNEL_DYNAMIC` | Keycloak | `https://shop.test:8443`, `true` |

Keycloak, its database and the importer run from a separate `docker/compose.sso.yaml` overlay in
`shop-mfe-platform`; the base stack does not change.

## Ports

| Port | Server | Purpose |
|---|---|---|
| `8443` | nginx | The IdP's public origin, `https://shop.test:8443`, with the existing `shop.test` certificate |
| `8080` | Keycloak | Internal only: nginx and the units' servers reach it as `keycloak:8080` |
| `5432` | Postgres | Internal only: Keycloak's database |

`KC_HOSTNAME` keeps the issuer `https://shop.test:8443/realms/shop` for the browser and for token checks,
while servers reach Keycloak on the internal network.

## Success criteria

Verified by Playwright against `https://shop.test` with `AUTH_ENABLED=true` and Keycloak running, unless
stated. Criterion 14 is checked by hand in New Relic.

1. With every switch off, MVP-1's criteria 1–14 pass unchanged.
2. Without a session, every Shell and MFE page redirects to sign-in with its `returnTo`, and every `/api/*`
   call and `/mfe/<name>/entry.js` gets `401` problem+json. Hashed files are still served.
3. After sign-in the user lands on `returnTo`. The browser holds `shop_at` (`Path=/`), `shop_rt`
   (`Path=/auth`) and `shop_id` (`Path=/auth/logout`), all `HttpOnly; Secure; SameSite=Lax`; page code sees
   none of them; page, MFE and API requests carry only `shop_at`.
4. A user without a unit's role gets `403` for that unit's `entry.js` and API, the MFE's no-access fallback
   shows, and the rest of the shop keeps working.
5. Every backend, called directly, refuses a missing token, an expired one, one signed by another key, one
   without its name in `aud`, and an ID token, each with `401` problem+json.
6. `/api/<unit>/me` returns the OIDC fields and only that unit's roles, with `no-store` and `Vary: Cookie`,
   and never a token or `sid`.
7. When the lightweight access token expires, the next call gets `401`, the client refreshes once and retries
   successfully; concurrent `401`s in several tabs lead to one refresh.
8. Sign-out shows no confirmation page, clears all three cookies, and ends access: refresh fails at once, every
   FES refuses within 30 s and every backend within 5 min.
9. With the IdP stopped, FES and backend checks answer `503` problem+json and never allow a request.
10. A backend calling another without the needed API role gets `403`; with the role, the call succeeds and
    carries the user's `sub`.
11. With `HEALTH_UNITS_AUTH=true`, `/health/units*` and `/version/units*` answer `401` without a token, `403`
    without the `ops` role and as in MVP-1 with it; each server's own health and version stay public.
12. Each MFE runs standalone with `AUTH_DEV_BYPASS=true` as the configured user and roles, every response
    carries `X-Auth-Dev-Bypass: true`, and a server with the bypass refuses to start when `NODE_ENV=production`
    or `K_SERVICE` is set.
13. The IdP adapter's contract tests pass, and no code outside the adapter calls a Keycloak endpoint or reads a
    Keycloak-specific claim.
14. Browser events and server transactions carry `enduser.id` equal to `sub`, and no email or username.
15. No axe violations at WCAG 2.2 AA on the no-access fallback and the Shell's user display, in both themes.
16. With two tabs open, signing out in one sends the other to sign-in at once; signing in in one takes the other
    from the sign-in page to its `returnTo`; sign-ins started in both tabs both succeed.
17. With two tabs open, signing in as another user in one makes the other reload and show that user before it
    sends any request as them.

MVP-1's criterion 6 now reads: the browser talks only to the public origin, the IdP's origin during sign-in
and sign-out redirects, and the New Relic browser agent's own endpoints.

## To confirm in phase 1

These depend on the IdP's exact behaviour and are checked on a running Keycloak before any unit code:

- The claims in Keycloak's lightweight access token: `sub` through the `basic` scope, `aud` through audience
  resolve with its lightweight switch, and whether `scope` remains.
- Sign-out with an **expired** `id_token_hint`: the ID token lives 5 min, sign-out may come hours later. If
  Keycloak refuses it, sign-out refreshes first, or signs out server-side with the refresh token inside the
  adapter.
- The admin API's response fields for the pinned version.
- `keycloak-config-cli` with several files for one realm: no file may remove another's clients, roles or
  grants, and grants that need another repository's service account must apply after it exists.
- Docker networking: the issuer stays `https://shop.test:8443/…` while servers reach `keycloak:8080`.
- Real cookie sizes, each expected well under 4 KB.
- What Keycloak does when a second tab submits its sign-in form after the first tab has signed in.
- `shop_at` received by pages and APIs after `/auth/callback` and `/auth/refresh`.

## Phases

**Provisional.** Re-planned after this document is approved; review may change them. One pull request each,
and the next starts only when asked.

1. Keycloak, its database, the importer, realm basics and every repository's `keycloak/` file, in the compose
   overlay.
2. The Shell FES's sign-in lifecycle and the three cookies, `/api/shell/me`, the user in the Shell, keeping
   tabs in step, New Relic.
3. The IdP adapter, FES and backend checks in every unit, `/api/<unit>/me`, the clients' refresh wrapper with
   its Web Lock, the standalone bypass.
4. Backend-to-backend access with API roles; the operator token on the router.
5. End-to-end tests for MVP-2, with MVP-1's still passing with every switch off.

## Appendix: IdP settings per repository

`keycloak-config-cli` YAML, Keycloak's own realm format. Secrets are never committed: `$(env:…)` is filled at
import (`import.var-substitution.enabled=true`) from each repository's gitignored `.env`. Attribute and mapper
names are confirmed in phase 1.

### `shop-mfe-platform/keycloak/realm.yaml` (applied first)

```yaml
realm: shop
enabled: true
sslRequired: external
accessTokenLifespan: 300
ssoSessionIdleTimeout: 1800
ssoSessionMaxLifespan: 36000
revokeRefreshToken: false

clientScopes:
  - name: roles                        # built-in; audience resolve must reach lightweight tokens
    protocol: openid-connect
    protocolMappers:
      - name: audience resolve
        protocol: openid-connect
        protocolMapper: oidc-audience-resolve-mapper
        config:
          access.token.claim: "true"
          lightweight.claim: "true"

users:                                 # test users; each unit's file grants its own roles
  - username: alice
    enabled: true
    email: alice@example.com
    emailVerified: true
    firstName: Alice
    lastName: Test
    credentials:
      - type: password
        value: $(env:TEST_USER_PASSWORD)
  - username: bob                      # no Catalog role: tests 403
    enabled: true
    email: bob@example.com
    emailVerified: true
    firstName: Bob
    lastName: Test
    credentials:
      - type: password
        value: $(env:TEST_USER_PASSWORD)

clients:
  - clientId: shop-ops                 # the operator's caller for /health/units*
    protocol: openid-connect
    publicClient: false
    secret: $(env:SHOP_OPS_SECRET)
    standardFlowEnabled: false
    directAccessGrantsEnabled: false
    serviceAccountsEnabled: true
```

### `shop-shell/keycloak/shop-shell.yaml`

```yaml
realm: shop

clients:
  - clientId: shell-fes                # the one sign-in client
    protocol: openid-connect
    publicClient: false
    secret: $(env:SHELL_FES_SECRET)
    standardFlowEnabled: true
    implicitFlowEnabled: false
    directAccessGrantsEnabled: false
    serviceAccountsEnabled: false
    redirectUris: ["https://shop.test/auth/callback", "http://localhost:3000/auth/callback"]
    webOrigins: ["https://shop.test", "http://localhost:3000"]
    attributes:
      pkce.code.challenge.method: S256
      client.use.lightweight.access.token.enabled: "true"
      post.logout.redirect.uris: "https://shop.test/##http://localhost:3000/"

  - clientId: shell-backend
    protocol: openid-connect
    publicClient: false
    secret: $(env:SHELL_BACKEND_SECRET)
    standardFlowEnabled: false
    directAccessGrantsEnabled: false
    serviceAccountsEnabled: true

roles:
  client:
    shell-backend:
      - name: shell-user

users:
  - username: service-account-shell-backend
    clientRoles:
      realm-management: [view-users]
  - username: alice
    clientRoles:
      shell-backend: [shell-user]
  - username: bob
    clientRoles:
      shell-backend: [shell-user]
```

### `shop-catalog/keycloak/shop-catalog.yaml`

```yaml
realm: shop

clients:
  - clientId: catalog-backend
    protocol: openid-connect
    publicClient: false
    secret: $(env:CATALOG_BACKEND_SECRET)
    standardFlowEnabled: false
    directAccessGrantsEnabled: false
    serviceAccountsEnabled: true

roles:
  client:
    catalog-backend:
      - name: catalog-viewer           # user roles
      - name: catalog-editor
      - name: catalog-api:read-products  # API role for other backends

users:
  - username: service-account-catalog-backend
    clientRoles:
      realm-management: [view-users]
  - username: service-account-cart-backend   # Catalog grants Cart read access
    clientRoles:
      catalog-backend: [catalog-api:read-products]
  - username: alice
    clientRoles:
      catalog-backend: [catalog-viewer, catalog-editor]
```

### `shop-cart/keycloak/shop-cart.yaml`

```yaml
realm: shop

clients:
  - clientId: cart-backend
    protocol: openid-connect
    publicClient: false
    secret: $(env:CART_BACKEND_SECRET)
    standardFlowEnabled: false
    directAccessGrantsEnabled: false
    serviceAccountsEnabled: true

roles:
  client:
    cart-backend:
      - name: cart-user

users:
  - username: service-account-cart-backend
    clientRoles:
      realm-management: [view-users]
  - username: alice
    clientRoles:
      cart-backend: [cart-user]
  - username: bob
    clientRoles:
      cart-backend: [cart-user]
```

### `shop-router/keycloak/shop-router.yaml`

```yaml
realm: shop

clients:
  - clientId: shop-router              # audience and holder of the ops role; never signs in
    protocol: openid-connect
    bearerOnly: true

roles:
  client:
    shop-router:
      - name: ops

users:
  - username: service-account-shop-ops
    clientRoles:
      shop-router: [ops]
```
