# MVP-3: Shared packages and isolated tenants (plan)

MVP-3 moves the logic every unit copies by hand into npm packages, and removes the configuration hard-coded in
source, so each tenant can run as its own deployment with its own IdP and domains. It builds on
[MVP-1](mvp.md) and [MVP-2](mvp-2-sso.md), both tagged (`mvp-1`, `mvp-2`).

- **Additive and off by default.** With default settings every unit behaves as in MVP-2, and MVP-1's criteria
  1–14 and MVP-2's criteria 1–17 still pass.
- **Docker Desktop only.** Staging and production come later.
- **Phase 1 done** (2026-11-15, [below](#phase-1-result)); phases 2 and 3 planned. Each phase is re-planned
  before it starts, and the next starts only when asked.

## Goals

1. **Don't repeat yourself:** logic we wrote once lives in packages, not in folders copied into every repository.
2. **Isolated tenants:** each tenant has its own stack, its own public domain and its own IdP. The same images
   serve every tenant; only configuration differs.

## Part 1: Shared packages

### Principle

Package **only code we wrote ourselves**. No wrapper around a third-party library: units keep calling
single-spa, React, New Relic and Vite directly.

### Duplication today

| Code | Lines | Copies |
|---|---|---|
| `server-common/src/auth/**`: token checks, OIDC metadata, caches, FES gate, backend checks, service tokens, cookies, dev bypass, IdP adapter, Keycloak adapter, contract | ~1,050 | 4: Shell, Catalog, Cart and the router's `src/auth`, byte-identical |
| `trace.ts`, `problem.ts`, `shutdown.ts` | ~95 | 4, identical |
| `health.ts`, `forward.ts`, `bin/write-version.mjs` | ~110 | 3–4 |
| Tests of the above | ~530 | 3–4 |
| Client `auth.ts` (`apiFetch`, refresh lock, login URL), `timeouts.ts` | ~105 | 3: identical in Catalog and Cart, a superset in the Shell |
| Theme key and no-flash script | small | Shell and shell-stub |

The router carries cookie and FES gate code it never uses, and "keep the auth copy in step" pull requests show
the cost of keeping the copies equal.

### Packages

| Package | Contents | Third-party |
|---|---|---|
| `@arun-dev/server`, with subpath exports `/http`, `/auth`, `/auth/keycloak` | `/http`: trace response, problem+json, health and version, `write-version`, graceful shutdown, `forward`, `requiredEnv`, backend auth headers. `/auth`: token checks, OIDC metadata, caches, FES gate, backend checks, service tokens, cookies, dev bypass, operator token, the login lifecycle, the IdP adapter contract and its exported contract tests. `/auth/keycloak`: the Keycloak adapter | Peer dependencies `express`, `jose`, `openid-client`; `newrelic` optional. All called directly |
| `@arun-dev/client` | `apiFetch` with the refresh lock, `startLogout`, `fetchWithTimeout` and `withTimeout`, login and logout URLs, theme storage key and no-flash script; the runtime config reader comes in phase 2, with the injection that uses it | None: plain TypeScript and browser APIs, so any framework can use it |

### Generic, not tied to MFEs

Neither package knows about MFEs: no single-spa, no `/mfe/*` paths, no `entry.js`. They implement a
backend-for-frontend login with HttpOnly cookies, token checks, problem+json, tracing and health, so any
Express + JavaScript application can use them, for example a single React SPA served by Express. MVP-3 removes
the MFE leftovers:

| Today | Change |
|---|---|
| Audience derived as `` `${unit}-backend` `` in `auth/backend.ts` and the Keycloak adapter | explicit `audience` and `rolesClient` options |
| "unit" in option names and messages ("No access to this unit") | neutral wording; messages are options |
| A user without a role always gets `403` | `requireRole` option, on by default |
| `newrelic` a required peer dependency | optional: its five calls run only when it is installed |
| Cookie names `shop_at`, `shop_rt`, `shop_id`, `shop_tx_*`; lock `shop-auth-refresh`; key `shop-theme` | one `cookiePrefix` option, default `shop` |
| "the shop" in messages | `appName` option, default "the shop" |

**Auth paths.** `/auth/*` is written out on 31 code lines today, 30 of them in code that moves into the
packages. One option, `authPaths = { base: '/auth', login, callback, refresh, resume, logout, loggedOut }`,
defaults to today's values. Only the base and the route names are configurable, not unrelated paths: cookie
scoping depends on one base (refresh token `Path=<base>`, ID token `Path=<base>/<logout>`, login cookie
`Path=<base>/<callback>`), so the cookie paths are derived from it and keep that guarantee automatically. The
client takes the same base (`createAuth({ basePath })`).

Still assumed: Node 24, Express 5, an OIDC identity provider, sessions in cookies set by the server
([ADR 0003](adr/0003-session-cookies.md)).

### Stays in each repository

ErrorBoundary, single-spa `entry.tsx`, routing hook, `vite.config.ts`, `entry.mjs`, `sourcemaps.mjs`, each
FES's `app.ts` (built from `@arun-dev/server` pieces), business routes, mock backends, Keycloak settings files
and the Shell's MFE list. They are small and mostly glue around a library; copying them is cheaper than an
abstraction.

**Lint and TypeScript settings stay as they are.** Each repository keeps its own `eslint.config.js` and
`tsconfig.base.json`; the MFE platform does not use `@arun-dev/eslint-config` or `@arun-dev/ts-config`.

### Clients share code at build time

MVP-2's client `auth.ts` says clients "share only these URLs and names, never code". `@arun-dev/client` is a
build-time library, like `@arun-dev/ui`, not something the Shell hands to MFEs at runtime, so the vision's
"the Shell gives MFEs the event service and the theme, nothing else" still holds. That comment changes with
MVP-3.

### Repository, scope and versions

| | Decision |
|---|---|
| Repository | New: `arun-web-foundation`, a pnpm workspace with `packages/server` and `packages/client`, its own lint and TypeScript settings taken from the shop's |
| npm scope | `@arun-dev`, beside `@arun-dev/ui`, `tokens` and `headless`. No `mfe` in the names: the packages are generic |
| Releases | changesets and `release.yml` with OIDC trusted publishing, as in `arun-design-system` |
| Versions | `0.x` while MVP-3 is in progress, `1.0.0` when it is done |
| This repository | Stays documentation only and links to the packages |

Units use published versions. Before a release, `pnpm link` or an override to the sibling checkout; Docker
builds always use published versions (the registry can lag up to about 8 minutes).

## Part 2: Isolated tenants

### What a tenant is

One tenant is one complete, independent deployment: its own edge, router, Shell, MFEs, backends, IdP and
monitoring. Nothing is shared at runtime, and every tenant runs the same images.

The **default tenant** (no tenant settings) is today's `shop.test` with `idp.test`, so MVP-1 and MVP-2 are
unchanged.

### Hard-coded values and their fix

| Where | Today | Fix |
|---|---|---|
| Shell, Catalog and Cart FES `index.ts` | `backendAudience: 'shell-backend'`, `'catalog-backend'`, `'cart-backend'` | `BACKEND_AUDIENCE`, default today's value |
| Mock backends | their own audience, e.g. `'catalog-backend'` | `OIDC_AUDIENCE`, same default |
| `shop-router/src/index.ts` | audience `'shop-router'`, role `'ops'` | `HEALTH_UNITS_AUDIENCE`, `HEALTH_UNITS_ROLE` |
| Auth code | `IDP_ADAPTER` accepts only `keycloak` | the adapter is chosen through `@arun-dev/server/auth` |
| `keycloak/*.yaml` in five repositories | `realm: shop` | `realm: $(env:OIDC_REALM)` (variable substitution is already on) |
| `shop-shell/keycloak/shop-shell.yaml` | redirect, web origin and post-logout URIs for `shop.test` and `localhost:3000` | from `$(env:PUBLIC_ORIGIN)`; `localhost:3000` only for the default tenant |
| `compose.sso.yaml` | `…/realms/shop` (4×), alias `shop.test` | `${OIDC_REALM:-shop}`, `${PUBLIC_HOST:-shop.test}` |
| `compose.yaml` | fixed container and network names | none; the Compose project name per tenant |
| `docker/certs/` | one committed certificate for `shop.test` and `idp.test` | a script generating one certificate per tenant from its `PUBLIC_HOST` and `IDP_HOST` |
| `realm.yaml` | test users `alice`, `bob` | per tenant, from its settings |
| e2e tests, `sso-check` | `shop.test`, `idp.test`, `alice`, `bob` | from env (`E2E_BASE_URL` exists; add the IdP host and users) |

Unchanged on purpose: `/api/<unit>` and `/mfe/<unit>` (the platform's contract), the `localhost:300x`
development defaults and the router's fixed `shell` unit.

### Client configuration: one build for every tenant

| Built into clients today | Clients | Tenant-specific? |
|---|---|---|
| `VITE_API_TIMEOUT_MS`, `VITE_LAZY_CHUNK_TIMEOUT_MS`, `VITE_MFE_*_TIMEOUT_MS` | all | No: platform defaults |
| `VITE_NEW_RELIC_*` (browser agent values) | Shell | Yes |
| `PUBLIC_ORIGIN`, `NEW_RELIC_USER_API_KEY`, `NEW_RELIC_BROWSER_APP_ID` (source-map upload) | all | Yes, for the upload only |

1. **The Shell reads its config at runtime.** The Shell FES already rewrites `index.html`; it also injects
   `<script type="application/json" id="app-config">` with the tenant's public values. No extra request and no
   effect on first paint. `@arun-dev/client` reads it.
2. **MFE clients have nothing tenant-specific.** Timeouts stay build-time defaults, and MFEs receive nothing new
   from the Shell.
3. **Source maps are uploaded at deploy, per tenant.** The build keeps hidden source maps outside the image;
   each tenant's deploy uploads them against that tenant's `PUBLIC_ORIGIN` with its keys.

Each unit's image is built once and runs for every tenant; an MFE release is one build and one deploy per tenant.

**Safeguards.** The New Relic browser values (account ID, trust key, agent ID, application ID, browser license
key) are public by design and are in today's Shell bundle, so moving them into `index.html` exposes nothing new.
Secrets never reach the browser: `NEW_RELIC_LICENSE_KEY` (servers), `NEW_RELIC_USER_API_KEY` (deploy only),
`OIDC_CLIENT_SECRET` and the backends' client secrets, `AUTH_COOKIE_KEY`.

1. **Allow-list.** The Shell FES injects a fixed list of named public fields and never reads `process.env` as a
   whole; nothing outside the list can leak.
2. **Naming.** Public browser values have their own prefix (e.g. `PUBLIC_NEW_RELIC_BROWSER_KEY`), which no secret
   uses. `VITE_NEW_RELIC_LICENSE_KEY` is renamed, since it reads like the server license key.
3. **Tests.** A unit test checks the injected JSON holds exactly the allowed fields; an e2e check confirms the page
   source contains none of the secrets above.
4. **Escaping.** Values are written with `<` escaped, so no value can close the script tag.

### Two tenants on Docker Desktop

| | Tenant A | Tenant B |
|---|---|---|
| Shop | `https://orga.shop.test` | `https://orgb.shop.test` |
| IdP | `https://orga-idp.test`, its own Keycloak | `https://orgb-idp.test`, its own Keycloak |
| Realm | `orga` | `orgb` |
| Compose project | `tenant-orga` | `tenant-orgb` |
| Certificate | `orga.shop.test`, `orga-idp.test` | `orgb.shop.test`, `orgb-idp.test` |
| New Relic | its own apps and keys | its own apps and keys |

- **Its own Keycloak per tenant**, not a realm on a shared one: the tenant controls its IdP, and nothing can be
  shared by accident.
- **Realm names other than `shop`** catch anything that still assumes it.
- **The IdP on its own domain.** A shop and its IdP need not share a domain, as with today's `shop.test` and
  `idp.test`. The shop's cookies never reach the IdP; login and logout are full-page redirects, so the IdP's own
  cookies work on its domain. The units reach the IdP only through `OIDC_ISSUER_URL`.
- **Different hosts, never ports.** Cookies are per host, not per port (MVP-2 finding F4), so tenants on one host
  with two ports would share cookies.
- **One certificate per tenant**, covering its two hosts. Each tenant's nginx holds only its own.
- **A local front door.** Docker Desktop can publish port 443 once, so one nginx routes by host name (TLS
  passthrough) to each tenant's own nginx. It holds no certificates and no logic, and stands in for each tenant's
  own DNS and load balancer. It is used only for several tenants; the default stack still publishes its nginx
  directly.

```
docker/
  compose.yaml, compose.sso.yaml        values from the tenant's settings
  compose.front-door.yaml               new, several tenants only
  tenants/
    orga.env.example, orgb.env.example  committed; orga.env and orgb.env gitignored (secrets)
  tenant.sh up|down|deploy <tenant>     docker compose -p tenant-<id> --env-file tenants/<id>.env
  certs/                                generated per tenant; keys not committed
```

A tenant's settings file holds `TENANT_ID`, `PUBLIC_HOST`, `IDP_HOST`, `OIDC_REALM`, client IDs and secrets,
audiences, `AUTH_COOKIE_KEY`, New Relic keys and app names, and test users. `/etc/hosts` gets one line per host.

### Not in MVP-3

| Item | Why |
|---|---|
| A different MFE set per tenant | The vision keeps a static MFE list in the Shell; per-tenant sets belong to entitlement and the platform registry. Every tenant gets the same MFEs; each tenant's router already points at its own MFE builds |
| A second IdP vendor | Two Keycloaks with different realms, hosts and client IDs prove nothing is hard-coded. Another vendor needs its own adapter and settings, a phase of its own. The contract tests stay exported for it |
| Staging and production | Docker Desktop only |

## Phases

1. **Packages.** Create `arun-web-foundation`; extract `@arun-dev/server` and `@arun-dev/client` with the generic
   options above; publish `0.x`; switch the Shell, Catalog, Cart and the router to them and delete their copies.
   No behaviour change: MVP-1's and MVP-2's suites, `sso-check.sh` and every unit test pass. Criteria 1–3.
2. **Configuration.** Replace the hard-coded values with settings whose defaults are today's values; template the
   Keycloak settings files; drop fixed Compose names; generate certificates per tenant; inject the Shell's runtime
   config with the four safeguards; upload source maps at deploy; read hosts and users in tests from env.
   Criteria 1, 7 and 10.
3. **Two tenants.** Tenant settings files, a Compose project per tenant, the front door and the certificates;
   run `orga` and `orgb` side by side. Criteria 4–9. Then `1.0.0` and the `mvp-3` tag on every repository.

## Phase 1 result

Done on 2026-11-15. `@arun-dev/server` and `@arun-dev/client` are published from
[arun-web-foundation](https://github.com/arun9483/arun-web-foundation) by its release workflow, with provenance;
every unit uses them and holds no copy (criteria 2 and 3).

| Repository | Pull request |
|---|---|
| arun-web-foundation | #1 workspace, #2 `@arun-dev/server`, #3 `@arun-dev/client`, #5 `startLogout`; releases 0.1.0 (server, client), 0.2.0 (client) |
| shop-router | #12 |
| shop-catalog | #17 |
| shop-cart | #17 |
| shop-shell | #22; `@arun-shop/shell-stub` 0.3.1 |
| shop-mfe-platform | #11: `sso-check` runs the adapter contract from `@arun-dev/server` |

**Checked** on Docker Desktop after each unit switched, with the images from `deploy/build.sh`: MVP-1's e2e suite
with login off (65 passed, 5 that need a server's own port skipped), MVP-2's `sso.spec.ts`, the IdP-down test and
`sso-check.sh` (all 20 checks, the ID-token expiry one included), so criterion 1 holds.

**Changed from the plan**
- The IdP adapter's `getUnitRoles(sub, unit)` is `getClientRoles(sub, client)`, and `requireUser` takes
  `audience`, `rolesClient` and `requireRole` in place of `unit`.
- `authNames({ cookiePrefix, authPaths })` gives every cookie name and login path; `appName` and the messages
  are options. Defaults are today's values.
- The first release of each package was a hand-published `0.0.0` placeholder, so the trusted publisher could be
  set up; `0.1.0` onwards come from the release workflow.
- Our own packages are excluded from pnpm's one-day `minimumReleaseAge` in every repository that uses them.

**Found and fixed:** the flaky "logging out in one tab sends the other to login" test was a real bug. A call
answered just after the logout response got `401`, and the client went to login, which cancelled the logout's
redirects, so `/auth/logged-out` never told the other tabs. `startLogout()`, called when the logout form is sent,
stops that; a new e2e test forces the race.

**Left as is:** the `X-Shop-User-Sub` header name, and `connectIdp` choosing only Keycloak (phase 2).

## Success criteria

1. With default settings, MVP-1's criteria 1–14 and MVP-2's criteria 1–17 pass unchanged.
2. No unit repository holds a copy of the shared server or client code; all use `@arun-dev/server` and
   `@arun-dev/client`, published from `arun-web-foundation`.
3. `@arun-dev/client` has no runtime dependencies, and `@arun-dev/server` wraps no third-party library.
4. Two tenants, `orga` and `orgb`, run side by side on Docker Desktop from the same images, each with its own
   Keycloak, realm, domains and New Relic apps.
5. A user logged in on one tenant is not logged in on the other, and neither tenant's cookies reach the other's
   hosts.
6. Stopping one tenant's IdP or stack does not affect the other.
7. No tenant value is in source: outside defaults and documentation, a search for `shop.test`, `idp.test` and
   `realms/shop` finds nothing.
8. MVP-2's `sso.spec.ts` and `sso-check.sh` pass against each tenant.
9. Client errors from each tenant resolve to source in that tenant's New Relic app (checked by hand).
10. The Shell's injected config holds exactly the allowed public fields, and no page contains a secret.
