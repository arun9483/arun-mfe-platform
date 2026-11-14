# MVP-2: Single sign-on (POC)

MVP-2 adds users to the shop: one login with an identity provider (IdP), every unit protected, and the
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
| Login, refresh and logout, owned by the Shell | A second app on another origin to prove single sign-on across apps |
| Every Shell and MFE route protected | Public pages |
| Checks in every FES and backend, failing closed | Entitlement (which MFEs a user may see): later |
| `/api/<unit>/me` in every unit | Staging and production |
| Backend-to-backend calls with the caller's own identity | A branded login page (Keycloak's default theme) |
| `/health/units*` and `/version/units*` behind an operator token | |
| Standalone MFEs with a development-only bypass | |

## Terms

| Term | What it is | Issued to | Kept in | Used for |
|---|---|---|---|---|
| **Lightweight access token** | The logged-in user's access token: a signed JWT with only `sub`, `sid`, `iss`, `exp`, `iat`, `jti`, `azp` and `aud`; no profile, no roles | `shell-fes` | Cookie `shop_at` | Browser → FES → its backend: which user the request is for |
| **Refresh token** | Gets a new lightweight access token while the IdP's session lives | `shell-fes` | Cookie `shop_rt` | Shell FES → IdP only |
| **ID token** | Tells the Shell FES who logged in. Never a bearer for an API | `shell-fes` | Cookie `shop_id` | Checked at login; `id_token_hint` at logout |
| **Service access token** | A backend's, or the operator caller's, own token from the client-credentials flow; a full JWT with its roles | Each backend's client, `shop-ops` | The caller's memory, reused until shortly before expiry | Backend → IdP, backend → backend, operator → `/health/units*` |

`sub` is the user's permanent ID and the only user key units store. `sid` names one login session; a user
logged in on two devices has two.

## Architecture

```
Browser (Shell + MFE clients; no token readable by page code)
  │ cookies: shop_at (Path=/) · shop_rt (Path=/auth) · shop_id (Path=/auth/logout)
  ├──── login / logout redirects ─────────────────► IdP: Keycloak, https://shop.test:8443, realm `shop`
  ▼                                                      users, roles, the login session, its admin API
nginx (HTTPS, cache)
  ▼
shop-router   routing; checks the operator token on /health/units*, /version/units*
  ├─► Shell FES    /auth/login · /auth/callback · /auth/resume · /auth/refresh · /auth/logout   (client shell-fes)
  │                + the checks every FES makes
  ├─► Catalog FES  check shop_at · gate: its backend's /me · forward
  └─► Cart FES     check shop_at · gate: its backend's /me · forward
         │ Authorization: Bearer <lightweight access token>
         ▼
  Shell, Catalog and Cart backends
         check the lightweight access token → sub, sid; session live, enabled, unit roles
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
| `shell-fes` and `shell-backend` clients, Shell roles | `shop-shell` (`keycloak/`) | The login client and Shell roles |
| `catalog-backend` client, Catalog roles and API roles, its grants to consumers | `shop-catalog` (`keycloak/`) | Who may use Catalog, and for what |
| `cart-backend` client, Cart roles and API roles, its grants | `shop-cart` (`keycloak/`) | The same for Cart |
| `shop-router` client: the audience and holder of the `ops` role | `shop-router` (`keycloak/`) | Who may read unit health and versions |
| Login lifecycle and all three cookies | `shop-shell` (Shell FES) | The only place tokens are received, refreshed or cleared |
| FES and backend checks, `/me`, the IdP adapter | `server-common/` in each unit, copied as in MVP-1 | Checks, lookups, the only IdP-specific code |
| Vision, this document, ADRs | `arun-mfe-platform` | Requirements and decisions |

**Settings per repository.** Each repository keeps `keycloak/<repository>.yaml` (see the appendix).
`shop-mfe-platform` runs [`keycloak-config-cli`](https://github.com/adorsys/keycloak-config-cli) at start: the
realm basics first, then every checked-out repository's file, found the way the Docker stack already finds
sibling repositories. A unit adding a role changes only its own file. **The provider grants its own
roles**: a Catalog role given to a user or to another unit's service account is listed in `shop-catalog`'s
file, so each file is the full list of who may use that unit.

## Identity provider settings

Realm `shop` on Keycloak 26.8.2, with Postgres 18.6, applied by `keycloak-config-cli` 6.5.1. The files themselves
are in each repository's `keycloak/` folder; the appendix lists them.

| Client | Owner | Type | Settings |
|---|---|---|---|
| `shell-fes` | `shop-shell` | Confidential; authorization code with PKCE | Redirect `https://shop.test/auth/callback` (and `http://localhost:3000/auth/callback` for local e2e); "Always use lightweight access token" on; mappers putting `sub` and the units' audiences into lightweight tokens |
| `shell-backend`, `catalog-backend`, `cart-backend` | Each unit | Confidential; service account only, never signs a user in | Service account holds `realm-management` → `view-users`. Optional scope `kc-admin-api`, asked for only for admin API calls. Defines the unit's user roles (`catalog-viewer`, …) and API roles (`catalog-api:read-products`, …), granted through the unit's groups |
| `shop-router` | `shop-router` | Bearer-only: no secret, never logs in | Defines the `ops` role |
| `shop-ops` | `shop-mfe-platform` | Confidential; service account only | Granted `shop-router` → `ops` |

FESes have no client of their own: only the Shell FES signs users in, with `shell-fes`.

**Audience without editing other repositories.** Keycloak's *audience resolve* mapper adds a client to `aud`
when the user holds one of that client's roles, so a user with a Catalog role gets `aud: catalog-backend`
without `shell-fes` naming Catalog. `shell-fes` carries its own copy with "Add to lightweight access token"
on, and a `sub` mapper with the same switch: Keycloak leaves `sub` out of lightweight tokens otherwise.

**Roles are granted through groups.** Each unit defines its own groups (`/catalog/viewer`, `/catalog/editor`,
`/catalog/api-read-products`, …), each holding that unit's roles, and its file puts users and other units'
service accounts in them. A user's client roles listed in one settings file would replace those from every
other file; group memberships from every file are kept (`IMPORT_USERS_MERGEGROUPS`).

**Admin API scope.** Keycloak 26.8's admin API accepts only tokens carrying the `kc-admin-api` scope and
audience. It is optional on each backend's client, so a backend asks for it only for admin API calls, and a
token it sends to another backend never carries admin API rights.

**Applying.** Two passes: realm basics and each repository's `keycloak/shop-*.yaml`, then each repository's
`keycloak/grants.yaml`, which grants roles to other repositories' service accounts once those exist.

| Setting | Value | Why |
|---|---|---|
| Access token lifespan | 5 min | Bounds how long a logged-out user's token still passes a local check |
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
- **Set together, cleared together.** Login and refresh set all three; logout and a failed refresh clear
  all three. A cookie is cleared only by a response naming the same `Path`, so each is cleared with its own
  `Path` and `Max-Age=0`.
- **`shop_at` names `Path=/` explicitly.** Cookies set by `/auth/*` responses otherwise default to `Path=/auth`.
- **Encryption.** The Shell FES encrypts `shop_rt` and `shop_id` with `AUTH_COOKIE_KEY`, which no other unit
  holds. `shop_at` is not encrypted: it is a signed token any FES can check, carrying no personal data beyond
  `sub`.
- **An exception to MVP-1.** MVP-1 says no server sets a cookie. These three cookies are the exception. They
  hold no server state, so any pod still serves any request.

## Flows

### Login
1. A protected request without a valid `shop_at`: a page is redirected to `/auth/resume?returnTo=<path>`
   (see "Page load with an expired login"), which goes on to `/auth/login?returnTo=<path>` when there is no
   login to refresh; an API call or `entry.js` gets `401` problem+json.
2. `/auth/login` first checks that the IdP answers (its metadata, inside the network, 2 s timeout, answer
   reused for 5 s). If it does not, the browser gets the shop's HTML `503` with **Try again** linking to
   `returnTo`, never the IdP's address, where nginx could only show a bare `502` (verification finding F9).
   Otherwise the Shell FES redirects to the IdP's authorization endpoint (code with PKCE, `state`, `nonce`), keeping
   `state`, `nonce`, the PKCE verifier and `returnTo` in a short-lived encrypted cookie named for this attempt,
   `shop_tx_<state>`, on `Path=/auth/callback`. Each attempt has its own, so logins started in several tabs
   do not overwrite each other.
3. The user logs in on the IdP's page; the IdP starts its session.
4. The IdP redirects to `/auth/callback?code=…`. The Shell FES exchanges the code with `shell-fes`'s
   credentials and checks the ID token (signature, `iss`, `aud`, `nonce`).
5. It sets the three cookies, clears `shop_tx_<state>` and redirects to `returnTo`, which must be a path on the
   shop's origin.
6. A callback whose attempt cookie is missing or does not match: if the browser already holds a valid
   `shop_at`, because another tab logged in, it redirects to `returnTo`; otherwise it starts a new login.

### Page and MFE load
1. The Shell's HTML, every `/mfe/<name>/entry.js` and every `/api/*` need a valid `shop_at`. Hashed files
   hold no data and stay public and edge-cacheable. Pages have no file extension: a missing file (e.g.
   `/favicon.ico`) is a plain `404`, never the app and never through the gate, where an expired login would
   start a login round-trip in the background for an image request (verification finding F3). The Shell has
   its own favicon.
2. With login on, the Shell FES marks the page (`<meta name="shop-login" content="on">`). Only then does the
   Shell client call `/api/shell/me`, show the user and call `newrelic.setUserId(sub)`; with login off it makes
   no extra request, as in MVP-1.
3. On `/catalog`, `import('/mfe/catalog/entry.js')` carries `shop_at`; Catalog's FES checks it with its gate.
4. A failed `import()` does not say why, so with login on the Shell then reads the entry's status (`HEAD`):
   `401` → refresh under the shared lock and load once more (or log in); `403` → "You do not have access to
   Catalog" in Catalog's place, not reported as an error, the rest of the shop working; anything else → the
   "unavailable" fallback, as in MVP-1. MFEs need no code for this (decided 2026-11-11).

### API call
For example `GET /api/catalog/products`:
1. **FES.** Checks `shop_at` with `jose`: signature, `iss`, `exp`, `azp = shell-fes`, and that it is an access
   token (`idp.isAccessToken`). A token without this unit's backend in `aud` is a `403` at once: the IdP puts
   it there only when the user holds one of the unit's roles, and the backend would otherwise answer `401`,
   which the client takes for an expired login (found in phase 3). Then the **gate**: it asks its own
   backend's `/me` with the token, cached per `sid` for 15 s. `200` with at least one of this unit's roles lets the request through; `401` (session
   ended, user disabled) and `403` (no role in this unit) are passed back. Then `newrelic.setUserID(sub)`.
2. The FES calls its backend with `Authorization: Bearer <lightweight access token>`.
3. **Backend.** Checks the token with `jose`: signature, `iss`, `exp`, its own name in `aud`, and the token
   type. Takes `sub` and `sid`, calls `newrelic.setUserID(sub)`, then through the IdP adapter, with its own
   service access token: `idp.isSessionActive(sub, sid)`, `idp.getUser(sub)` and `idp.getUnitRoles(sub,
   'catalog')`, cached per `sid` for 15 s. An ended session or a disabled user is `401`; a missing role
   is `403`. The request is handled with `sub` as the user key.
4. The backend never sends the lightweight access token anywhere.

Keycloak refuses lightweight access tokens at `/userinfo`, so the FES asks its backend, which already holds
everything the decision needs. The FES needs no client of its own and makes no IdP call.

### Refresh
1. A FES finds `shop_at` expired and answers `401` problem+json.
2. The client takes the lock `shop-auth-refresh` with the [Web Locks API](https://w3c.github.io/web-locks/)
   (`navigator.locks.request`), which every tab of the shop shares. Inside the lock it first retries the
   call: another tab may have refreshed already, and cookies are shared by every tab.
3. Only if the retry still gets `401` does it call `POST /auth/refresh`. The Shell FES refreshes with
   `shell-fes`'s credentials (`scope=openid`, so a new ID token comes too) and sets the three cookies. The
   client releases the lock and retries its call.
4. If refresh fails, because the IdP's session has ended (`401`), the three cookies are cleared and the client
   goes to `/auth/login?returnTo=<path>`. If login cannot be checked right now (`503`, e.g. the IdP is down), the
   client stays on the page and shows that the data could not be loaded: login would only reach an IdP that is
   down (decided in phase 5).

### Page load with an expired login
A page load (a full navigation, not a call from the client) cannot use the client's refresh: `shop_rt` is on
`Path=/auth`, so the page's request does not carry it. So the gate sends the page to
`/auth/resume?returnTo=<path>`, which does (verification finding F9):

| `/auth/resume` finds | Then |
|---|---|
| `shop_rt`, refresh succeeds | Sets the three cookies and redirects to `returnTo`. The browser never visits the IdP. |
| No `shop_rt`, or the IdP's session has ended | Clears the cookies (if any) and redirects to `/auth/login?returnTo=<path>`. |
| `Sec-Fetch-Site` other than `same-origin` or `none`, or missing | No refresh: redirects to `/auth/login?returnTo=<path>`, as before F9. |
| The IdP cannot answer | The shop's HTML `503`, **Try again** linking to `returnTo`. |

**Try again** always links to the shop page asked for (`returnTo`), never an `/auth` address, so the gate
decides afresh: a login refreshed in another tab loads the page at once. Without a `returnTo`, or with one
that is not a shop path, it links to `/`. `/auth/resume` does not take part in the client's Web Locks lock;
two tabs loading pages at once refresh twice, which Keycloak allows (`revokeRefreshToken: false`).

Each client carries this small `401` → lock → retry → refresh → retry wrapper itself. It shares only a URL
and a lock name with the Shell, never code, as the vision requires. No tab refreshes on a timer: refresh
happens only when a call is refused, so hidden tabs do nothing.

### Logout
1. The Shell client calls `newrelic.setUserId(null)`, then submits a `POST` form to `/auth/logout`, so the
   browser follows the redirects that come back.
2. The Shell FES reads the ID token from `shop_id`, clears the three cookies and redirects to the IdP's
   end-session endpoint with `id_token_hint` and `post_logout_redirect_uri=<origin>/auth/logged-out`, so no
   confirmation page is shown. Without `shop_id` it sends `client_id` instead, and the IdP asks the user to
   confirm.
3. The IdP ends its session and sends the browser to `/auth/logged-out`, a small page from the Shell FES that
   tells the other tabs (`logged-out`) and goes on to the shop, which asks to log in. Telling them earlier, at
   submit, let a tab log in again on the session still being ended (found in phase 2).
4. Elsewhere: refresh fails at once, and every backend and FES gate refuses within 30 s: the token's `sid` is no
   longer a live session, and the FES and the backend each reuse an answer for 15 s.

### Several tabs
Every tab of one browser profile shares the cookies, so all tabs always hold the same tokens. The Shell keeps
the tabs in step with a [`BroadcastChannel`](https://html.spec.whatwg.org/multipage/web-messaging.html#broadcasting-to-other-browsing-contexts)
named `shop-auth`. The Shell client keeps the `sub` it shows in page memory, set from `/api/shell/me`; it is
not written to storage or a cookie, a reload clears it, and no server stores it, so servers stay stateless.

| Event in one tab | Message | Every other tab's Shell |
|---|---|---|
| Logout | `{ type: 'logged-out' }`, sent by `/auth/logged-out` once the IdP's session has ended | Navigates to `/auth/login?returnTo=<its path>` at once, so no tab keeps showing the user's data |
| Login | `{ type: 'logged-in', sub }`, sent whenever the Shell shows a user | Showing another `sub`: reloads (see below). A tab on the IdP's login page is on another origin and gets no message; Keycloak continues it by itself (see below) |
| Refresh | None: the Web Lock above coordinates it | — |

- **Another user in another tab.** Tab A shows alice; in tab B the user logs out and logs in as bob. Tab A's
  next request would run as bob while its screen still shows alice. So a Shell client that learns of another `sub`,
  from a `logged-in` message or from `/api/shell/me`, reloads the page. MFEs live in the Shell's page, so the
  reload covers them; they need no code for it.
- **When a tab becomes visible** (`visibilitychange`), the Shell calls `/api/shell/me`. This catches what no
  message announces: the IdP's session expiring, an administrator ending it, or a logout from another app or
  browser. A `401` leads to the usual refresh, and a failed refresh to login.
- **Logins started in two tabs** each keep their own attempt cookie (see Login), and a callback that finds
  the user already logged in goes to its `returnTo`.
- **A tab left on Keycloak's login page** continues by itself once the browser has logged in in another tab:
  Keycloak 24 and later watch for it (`authChecker.js`) and send that tab through
  `login-actions/restart` → `authenticate` → its own `/auth/callback`, with no second form. Each tab still
  completes its own attempt with its own `state` (found in the manual verification, F5).
- **A refresh in flight during a logout** in another tab may write new cookies for the ended session. Their
  lightweight access token lasts at most 5 min, the next refresh fails, and the `logged-out` message has
  already sent that tab to login. Accepted.

Messages carry only a type and `sub`, never a token. `BroadcastChannel` and Web Locks are standard and in every
current browser (Chrome 69, Edge 79, Firefox 96, Safari 15.4 and later); Web Locks needs a secure context, which
`https://shop.test` and `http://localhost` are. Without them the design stays correct, only slower to react:
without Web Locks each tab refreshes on its own; without `BroadcastChannel` other tabs notice a logout when
they become visible or make their next call.

### `/api/<unit>/me`
Answered by the unit's backend from the checks in "API call": `idp.isSessionActive(sub, sid)`,
`idp.getUser(sub)` and `idp.getUnitRoles(sub, unit)`. It is also the FES's gate. Field names follow
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
Built in phase 4: Cart's backend reads product names and prices from Catalog's backend.
1. Cart's backend gets its service access token (client credentials, no admin scope, reused until shortly
   before expiry; `serviceCaller`).
2. It calls `GET /service/products?ids=…` on Catalog's backend with `Authorization: Bearer <service access
   token>` and the user's `sub` in `X-Shop-User-Sub`. Never the user's token.
3. Catalog's backend checks the token with `jose` (`requireService`): its own name in `aud`, an access token
   with no `sid` (so never a user's token), and the caller's API role (`catalog-api:read-products`, read through
   `idp.serviceRoles`). Anything else is refused: `401`, or `403` for a token that names Catalog but lacks the
   role. A user's token on this route is a `401`, and a service token on a user route too (no `sid`).
4. Per endpoint, Catalog decides whether it trusts the caller for that `sub` or also checks the user's own
   Catalog roles. For `read-products` it trusts the caller (decided): product data is not the user's, and a user
   without a Catalog role, such as `bob`, still sees names in their cart. `X-Shop-User-Sub` is recorded for
   tracing only.
5. The grant is in Catalog's `keycloak/` file, so Catalog always knows its consumers. On Docker only Catalog's
   backend and the backends it grants share a network, `catalog-api`.
6. Cart composes from Catalog only with login on and `CATALOG_BACKEND_URL` set; otherwise it keeps its own copy
   (MVP-1). If Catalog cannot answer, the cart is a `503`.

A client never calls another unit's FES. Data from another domain is combined on the backend, as the vision
requires. Links to another unit's pages stay plain URLs; the target unit checks its own access. Events between
MFEs (the vision's event service) carry only IDs.

### Unit health and version for operators
1. `shop-ops` gets a service access token holding `shop-router` → `ops`.
2. It calls `/health/units*` or `/version/units*` with `Authorization: Bearer <service access token>`.
3. The router checks it with `requireService`: `shop-router` in `aud`, no `sid`, and the `ops` role; otherwise
   `401`, or `403` for a token that names the router but lacks the role. The router carries a copy of the units'
   `server-common/src/auth/` (decided: one implementation everywhere).
4. Each server's own `/health/live`, `/health/ready` and `/version` stay public; Docker's health checks use them.

`HEALTH_UNITS_AUTH` is off by default, so MVP-1's criterion 11 still passes.

## Checks per layer

| Layer | Checks | Decides | Fails with |
|---|---|---|---|
| Browser | Nothing enforced | What to show, from `/me` | — |
| Router | Only `/health/units*`, `/version/units*`: `aud`, `ops` role | Operator access | `401`, `403` |
| FES | `shop_at`: signature, `iss`, `exp`, `azp`, token type; its backend in `aud`; gate: its backend's `/me` | Logged in? Allowed into this unit? | `401`, `403` |
| Backend | Lightweight access token: signature, `iss`, `exp`, `aud`, token type; then live session by `sid`, `enabled` and roles by `sub`. Service access token: `aud`, API role | Everything about the data: ownership, actions | `401`, `403` |

**Fail closed.** If the IdP cannot be reached, a check answers `503`, never "allow". Every refusal is
`application/problem+json` with `traceId` and `Cache-Control: no-store`, except a page, which gets a short HTML
`503` ("Login cannot be checked right now. Try again shortly.") with a **Try again** link to the page asked for
(F6; it does not reload by itself). That holds for an expired login too, from `/auth/resume` and
`/auth/login` (F9). Two limits, by design:

- Answers the FES gate and the backend cached just before an outage stay valid for up to 15 s each, the same
  window as logout.
- Service routes and the router's operator token are checked offline with the IdP's cached keys: a service
  access token has no session to look up, so they keep working until the token expires.

## IdP adapter

Login, tokens and cookies use only standard OIDC and OAuth through `openid-client` and `jose`, configured
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
  /** Backend: whether the login session named by the token's sid is still live. */
  isSessionActive(sub: string, sid: string): Promise<boolean>;
  /** FES and backend: true only for an access token, never an ID or refresh token. */
  isAccessToken(header: Record<string, unknown>, claims: Record<string, unknown>): boolean;
  /** Provider backend and router: the roles a service access token holds on `client`. */
  serviceRoles(claims: Record<string, unknown>, client: string): string[];
}
```

| Concern | Standard, any IdP | Keycloak adapter |
|---|---|---|
| Login, refresh, logout, discovery | OIDC Core, RP-Initiated Logout, Discovery | — |
| Token checks | JWT and JWKS: `iss`, `exp`, `aud`, `azp` | — |
| Service access tokens | OAuth 2.0 client credentials | — |
| Token type | RFC 9068 header `typ: at+jwt`, where the IdP uses it | Claim `typ` is `Bearer` (`ID` for an ID token, `Refresh` for a refresh token) |
| Admin API token | — | Client credentials with `scope=kc-admin-api` |
| User by `sub` | — | `GET /admin/realms/shop/users/<sub>` |
| Roles by `sub` | — | `GET /admin/realms/shop/users/<sub>/groups?briefRepresentation=false`: each group's roles by client ID. `view-users` cannot look up a client's internal ID, which the role-mapping endpoints need |
| Live session | — | `GET /admin/realms/shop/users/<sub>/sessions` contains the token's `sid` |
| Roles in a service access token | — | `serviceRoles`: `resource_access.<client>.roles` |

- Nothing outside the adapter calls an IdP-specific endpoint or reads an IdP-specific claim.
- The adapter caches per `sid` and fails closed.
- Servers fetch the IdP's metadata inside Docker (`http://keycloak:8080/realms/shop/.well-known/openid-configuration`)
  and check that its `issuer` is `OIDC_ISSUER_URL`. `openid-client`'s own `discovery()` refuses a URL that differs
  from the issuer, so they build its `Configuration` from that checked metadata.
- Every adapter passes the same contract tests; the Keycloak adapter also runs against a real Keycloak.
- Another IdP needs a new adapter and its own settings files, with no change to flows, cookies or checks. It
  must allow refresh tokens to be used more than once (or rotate with a reuse window), support RP-initiated
  logout, and issue a small access token with `sub` and an `aud` per backend.

## Packages

All open source.

| Package | Used by | For |
|---|---|---|
| [`openid-client`](https://www.npmjs.com/package/openid-client) | Shell FES (login, refresh, logout); every backend (client credentials) | OIDC and OAuth flows. OpenID-certified |
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
  Opened in a browser (`GET`), they answer `405` with `Allow: POST` and a short page (for logout: "use the Log
  out button"); any other `/auth` address is a `404`, never the app (F1).
- **`/auth/resume`** is a `GET`, as pages are sent to it. It refreshes only when the browser's `Sec-Fetch-Site`
  is `same-origin` or `none` (the user's own navigation), which covers every step of a redirect chain. A
  navigation from another site, including a link to any shop page that then redirects here, or a request
  without the header, goes on to `/auth/login` instead, so another site cannot make it refresh (for example to
  keep a session from timing out). It never ends a session or logs in another user.
- **`/auth/login`**, **`/auth/resume`** and **`/auth/callback`** accept a `returnTo` that is a path on the shop's
  own origin, never a full URL.

## Standalone mode

Standalone MFEs keep the Shell stub as in MVP-1, with a **development-only bypass** instead of the IdP:

1. The MFE's FES and backend run with `AUTH_DEV_BYPASS=true`; no IdP, no login, no cookies.
2. The FES skips the cookie and its gate, acting as the user from `AUTH_DEV_USER_SUB` and
   `AUTH_DEV_USER_ROLES`, and calls its backend without a token.
3. The backend skips its token check and the IdP; `/api/<unit>/me` returns that user with this unit's
   configured roles. Everything after the check runs as usual, with `sub` as the user key.
4. Changing `AUTH_DEV_USER_ROLES` exercises role-based UI and `403`s without an IdP.

**Guard rails.** A server refuses to start with the bypass on when `NODE_ENV=production` or on Cloud Run
(`K_SERVICE` is set). It logs a warning at start and adds `X-Auth-Dev-Bypass: true` to every response.
Deploy builds never set it. Login, refresh, logout and real IdP lookups are tested only with the whole shop.

## Monitoring

| Where | Call | When |
|---|---|---|
| Shell client | `newrelic.setUserId(sub)`, `setUserId(null)` | After `/api/shell/me`; at logout |
| Every FES and backend | `newrelic.setUserID(sub)` | After its token check |

Browser events and server transactions then carry `enduser.id`, so one user can be followed from browser to
backend. Only `sub` is sent, never an email or username. This is the one call the Shell makes on its
`newrelic` object; MFEs still make none.

## Configuration

Every switch is off by default.

| Variable | Used by | Default |
|---|---|---|
| `AUTH_ENABLED` | every FES and backend | `false`: MVP-1 behaviour |
| `HEALTH_UNITS_AUTH` | router | `false`; with it, the router also needs `OIDC_ISSUER_URL` and `OIDC_INTERNAL_URL` |
| `CATALOG_BACKEND_URL` | Cart's backend | unset: Cart keeps its own names and prices (MVP-1) |
| `OIDC_ISSUER_URL` | every FES and backend, the router | `https://shop.test:8443/realms/shop` |
| `OIDC_INTERNAL_URL` | every FES and backend, the router | `http://keycloak:8080/realms/shop`: where the server reaches the IdP inside Docker |
| `PUBLIC_ORIGIN` | Shell FES | `https://shop.test`: the callback, logout and `Origin` checks |
| `LOGIN_CLIENT_ID` | an MFE's FES | `shell-fes`: the only login client whose tokens it accepts (`azp`). It has no client of its own |
| `OIDC_CLIENT_ID`, `OIDC_CLIENT_SECRET` | Shell FES (`shell-fes`), each backend (its own client), the operator caller (`shop-ops`) | in each server's own `.env`: its copy of the secret in its repository's `keycloak/.env`, which must match (decided 2026-11-11) |
| `AUTH_COOKIE_KEY` | Shell FES | per developer; encrypts `shop_rt` and `shop_id` |
| `IDP_ADAPTER` | every FES and backend | `keycloak` |
| `AUTH_DEV_BYPASS`, `AUTH_DEV_USER_SUB`, `AUTH_DEV_USER_ROLES` | an MFE's FES and backend, standalone only | unset |
| `KC_HOSTNAME`, `KC_HOSTNAME_BACKCHANNEL_DYNAMIC` | Keycloak | `https://shop.test:8443`, `true` |
| `KC_HOSTNAME_ADMIN`, `KEYCLOAK_DEBUG_PORT` | Keycloak | `http://localhost:8081`, `8081`: the admin console, for debugging |
| `KEYCLOAK_ADMIN_USER`, `KEYCLOAK_ADMIN_PASSWORD`, `KEYCLOAK_DB_PASSWORD`, `TEST_USER_PASSWORD`, `SHOP_OPS_SECRET` | `shop-mfe-platform`'s `docker/.env` | per developer |

Keycloak, its database and the importer run from a separate `docker/compose.sso.yaml` overlay in
`shop-mfe-platform`; the base stack does not change.

## Ports

| Port | Server | Purpose |
|---|---|---|
| `8443` | nginx | The IdP's public origin, `https://shop.test:8443`, with the existing `shop.test` certificate |
| `8080` | Keycloak | Inside Docker: nginx and the units' servers reach it as `keycloak:8080` |
| `8081` | Keycloak | `127.0.0.1` only: the admin console, for debugging. `/admin` is refused on `8443` |
| `5432` | Postgres | Internal only: Keycloak's database |

`KC_HOSTNAME` keeps the issuer `https://shop.test:8443/realms/shop` for the browser and for token checks,
while servers reach Keycloak on the internal network.

**Networks.** Each unit reaches Keycloak on its own Docker network (`idp-edge`, `idp-router`, `idp-shell`,
`idp-catalog`, `idp-cart`; the importer and the check script on `idp-admin`), with Keycloak alone on all of them.
Phase 4 found that the first overlay's single shared `idp` network let every container reach every backend,
undoing MVP-1's private backends while the overlay was on. `catalog-api` joins Catalog's backend and the
backends it grants; a network is two-way, so Catalog's backend can reach Cart's too, which refuses any call
without a valid token for it. A one-way path would need a proxy.

**Header size.** Logging in sets three cookies at once, about 4 KB of response headers. nginx's default buffer
refused that ("upstream sent too big header"), so `shop.conf` raises it to 16 KB. Any CDN or proxy in front of
the shop needs the same headroom.

**Staging, later.** On Cloud Run an FES sends its Google identity token in `Authorization` to reach its private
backend. With login on, that header carries the lightweight access token, so the Google token moves to
`X-Serverless-Authorization`, which Cloud Run also accepts.

## Success criteria

Verified by Playwright against `https://shop.test` with `AUTH_ENABLED=true` and Keycloak running, unless
stated. Criterion 14 is checked by hand in New Relic.

1. With every switch off, MVP-1's criteria 1–14 pass unchanged.
2. Without a session, every Shell and MFE page redirects to login with its `returnTo`, and every `/api/*`
   call and `/mfe/<name>/entry.js` gets `401` problem+json. Hashed files are still served.
3. After login the user lands on `returnTo`. The browser holds `shop_at` (`Path=/`), `shop_rt`
   (`Path=/auth`) and `shop_id` (`Path=/auth/logout`), all `HttpOnly; Secure; SameSite=Lax`; page code sees
   none of them; page, MFE and API requests carry only `shop_at`.
4. A user without a unit's role gets `403` for that unit's `entry.js` and API, the Shell shows "You do not have
   access" in that MFE's place, and the rest of the shop keeps working.
5. Every backend, called directly, refuses a missing token, an expired one, one signed by another key, one
   without its name in `aud`, and an ID token, each with `401` problem+json.
6. `/api/<unit>/me` returns the OIDC fields and only that unit's roles, with `no-store` and `Vary: Cookie`,
   and never a token or `sid`.
7. When the lightweight access token expires, the next call gets `401`, the client refreshes once and retries
   successfully; concurrent `401`s in several tabs lead to one refresh.
8. Logout shows no confirmation page, clears all three cookies, and ends access: refresh fails at once, and
   every backend and FES refuses within 30 s.
9. With the IdP stopped, once answers cached before the outage have expired (15 s), FES and backend checks
   answer `503` and never allow a request; pages show the HTML `503`; an expired login stays in the shop,
   on a page load too, with or without a refresh token (F9). With the IdP back, the shop works again
   without restarting any server.
10. A backend calling another without the needed API role is refused (`401` when its token does not name the
    provider, which Keycloak leaves out without a role there; `403` when it does but lacks the role); with the
    role, the call succeeds and carries the user's `sub`.
11. With `HEALTH_UNITS_AUTH=true`, `/health/units*` and `/version/units*` answer `401` without a valid token
    for the router, `403` with one lacking the `ops` role, and as in MVP-1 with it; each server's own health and
    version stay public.
12. Each MFE runs standalone with `AUTH_DEV_BYPASS=true` as the configured user and roles, every response
    carries `X-Auth-Dev-Bypass: true`, and a server with the bypass refuses to start when `NODE_ENV=production`
    or `K_SERVICE` is set.
13. The IdP adapter's contract tests pass, and no code outside the adapter calls a Keycloak endpoint or reads a
    Keycloak-specific claim.
14. Browser events and server transactions carry `enduser.id` equal to `sub`, and no email or username.
15. No axe violations at WCAG 2.2 AA on the no-access fallback and the Shell's user display, in both themes.
16. With two tabs open, logging out in one sends the other to login at once, and logins started in both tabs
    both succeed.
17. With two tabs open, logging in as another user in one makes the other reload and show that user before it
    sends any request as them.

MVP-1's criterion 6 now reads: the browser talks only to the public origin, the IdP's origin during login
and logout redirects, and the New Relic browser agent's own endpoints.

## Confirmed in phase 1

Checked on Docker Desktop on 2026-11-11 by `shop-mfe-platform`'s `docker/sso-check.sh`, which logs in through
Keycloak's own page with Playwright as `shell-fes`. Every check passes; MVP-1's e2e suite still passes with
Keycloak running.

| Item | Result |
|---|---|
| Issuer and networking | Discovery inside Docker reports `https://shop.test:8443/realms/shop`; back-channel endpoints point to `keycloak:8080` |
| Lightweight access token | `exp`, `iat`, `jti`, `iss`, `aud`, `sub`, `typ`, `azp`, `sid`, `scope`; no profile or roles. **`sub` needed a mapper.** `aud` holds the backends of units the user has roles in, plus `account` |
| Token type | Claim `typ`: `Bearer`, `ID`, `Refresh`. The JWT header says `JWT`, not `at+jwt` |
| `/userinfo` | **Refuses lightweight access tokens.** The FES gate asks its backend's `/me` instead (decided 2026-11-11) |
| Admin API | **Needs the `kc-admin-api` scope** (Keycloak 26.8). With `view-users`: user by `sub`, roles through groups, sessions by `sub` |
| Settings in several files | **A user's client roles in one file replaced every other file's.** Roles are granted through groups; re-running every file, and one file alone, changes nothing |
| Refresh token used twice | Both succeed |
| Two tabs | Logins started in both tabs both complete, each with its own `state` |
| Logout, expired `id_token_hint` (5+ min old) | Accepted: no confirmation page; refresh then fails, and the `sid` is gone from the user's sessions |
| Logout with only `client_id` | Keycloak shows its confirmation page |
| Cookie sizes | `shop_at` 907 B, `shop_rt` 1,087 B, `shop_id` 1,597 B |
| Admin console | Refused through nginx; on `127.0.0.1:8081` only |

Confirmed in phase 2: `shop_at` reaches pages and APIs after `/auth/callback` and `/auth/refresh`.

## Phases

Each phase is re-planned before it starts. The next starts only when asked.

1. **Done:** Keycloak, its database, the importer, realm basics and every repository's `keycloak/` file, in the
   compose overlay, with `docker/sso-check.sh`.
2. **Done:** the Shell unit, end to end (re-planned 2026-11-11, because `/api/shell/me` needs the backend checks):
   the login lifecycle and the three cookies, the FES gate, `server-common`'s auth with the Keycloak adapter, the
   backend checks and `/me`, and the Shell client (user, logout, refresh wrapper, tabs, New Relic). Criteria 2, 3,
   6, 7, 8, 15, 16 and 17 pass for the Shell (`shop-shell`'s `sso.spec.ts`); MVP-1's suite passes with login off.
3. **Done:** Catalog and Cart: `server-common`'s auth (identical copies, the Shell's included), the FES gate on
   `entry.js` and the API with the `aud` shortcut, backend checks and `/me`, the clients' refresh wrapper, the
   Shell reading a failed MFE load's status, the standalone bypass. Criteria 2, 3, 4, 6, 7, 8, 12 and 15 pass for
   Catalog and Cart (`sso.spec.ts`, `standalone.spec.ts`); criterion 5 for every backend (`sso-check.sh`, check
   15); MVP-1's suite passes with login off.
4. **Done:** backend-to-backend access with API roles (Cart reads products from Catalog) and the operator token
   on the router; private backends again with the overlay on (one Keycloak network per unit). Criteria 10 and 11
   pass (`sso-check.sh` checks 16–18, `sso.spec.ts`); MVP-1's suite passes with login off.
5. **Done:** the IdP down (9), the adapter contract and its lint rule (13), New Relic (14), and criterion 7 across
   tabs. MVP-2 is complete. The tag `mvp-2` follows the user's own end-to-end manual verification.

## Results

MVP-2 is complete (2026-11-12). Every criterion, how it is verified, and its result on Docker Desktop:

| # | Verified by | Result |
|---|---|---|
| 1 | MVP-1's e2e suite with login off: locally, and on Docker without the overlay | 92 passed locally; 65 passed, 5 skipped on Docker |
| 2, 3, 6, 7, 8 | `shop-shell`'s `sso.spec.ts` (Shell, Catalog, Cart; 7 also in two tabs) | Pass |
| 4, 15 | `sso.spec.ts`: `bob` and the "no access" message, axe in both themes | Pass |
| 5 | `sso-check.sh` check 15 (each backend called directly) and check 11 (expired token) | Pass |
| 9 | `idp-down.spec.ts` (stops and starts Keycloak) | Pass |
| 10 | `sso-check.sh` check 17; `sso.spec.ts` (both users' carts from Catalog) | Pass |
| 11 | `sso-check.sh` check 18 | Pass |
| 12 | `standalone.spec.ts` (bypass, refuses production and Cloud Run) | Pass |
| 13 | Contract (`server-common/src/auth/idp/contract.ts`): unit tests against a fake IdP and an unreachable Keycloak; `sso-check.sh` check 19 against Keycloak. ESLint refuses Keycloak's endpoints and claim names outside `idp/keycloak.ts` in every repository | Pass |
| 14 | New Relic, queried through NerdGraph, confirmed in the UI | `enduser.id` on transactions of every FES and backend and on browser page views, interactions and AJAX calls; every value a `sub` (UUID); none with `@` |
| 16, 17 | `sso.spec.ts`, several tabs | Pass |

Also checked by `sso-check.sh`: Keycloak's settings re-applied without loss (8), the admin console only on
`127.0.0.1` (14), and network isolation (16).

## Appendix: IdP settings per repository

Each repository owns its settings, in `keycloak-config-cli` YAML (Keycloak's own realm format). Secrets are
never committed: `$(env:…)` is filled at import from gitignored `.env` files.

| File | Holds |
|---|---|
| [`shop-mfe-platform/keycloak/realm.yaml`](https://github.com/arun9483/shop-mfe-platform/blob/main/keycloak/realm.yaml) | Realm, lifetimes, test users `alice` and `bob`, `shop-ops` |
| [`shop-shell/keycloak/shop-shell.yaml`](https://github.com/arun9483/shop-shell/blob/main/keycloak/shop-shell.yaml) | `shell-fes` with its `sub` and audience mappers, `shell-backend`, `/shell/user` |
| [`shop-catalog/keycloak/shop-catalog.yaml`](https://github.com/arun9483/shop-catalog/blob/main/keycloak/shop-catalog.yaml), [`grants.yaml`](https://github.com/arun9483/shop-catalog/blob/main/keycloak/grants.yaml) | `catalog-backend`, Catalog's roles and groups; Cart's backend in `/catalog/api-read-products` |
| [`shop-cart/keycloak/shop-cart.yaml`](https://github.com/arun9483/shop-cart/blob/main/keycloak/shop-cart.yaml) | `cart-backend`, `/cart/user` |
| [`shop-router/keycloak/shop-router.yaml`](https://github.com/arun9483/shop-router/blob/main/keycloak/shop-router.yaml), [`grants.yaml`](https://github.com/arun9483/shop-router/blob/main/keycloak/grants.yaml) | `shop-router`, the `ops` role and group; `shop-ops` in `/shop-router/ops` |
