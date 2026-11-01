# MVP (POC)

The MVP implements a subset of the [vision](vision.md) with the leanest possible units, first in **local
development and end-to-end tests**. It never deviates from the vision; the vision's other requirements are
built on top of it incrementally. The MVP focuses on two things:

- **Lazy loading:** the Shell's features, each MFE's lean entry and the features inside each MFE load only when needed.
- **Error debugging:** errors from every unit's client and FES, and from the backends they call, reach New Relic, and source maps resolve them to original source.

## Stack

| Area | Choice |
|---|---|
| Language | TypeScript everywhere |
| Runtime | Node 24.22.1 or newer; older Node 24 releases abort when the New Relic agent is on |
| Client | React, built with Vite |
| Composition | single-spa + `single-spa-react` (its `errorBoundary` is the failing-MFE fallback; the Shell wraps its own features in a React error boundary) |
| FES | Express |
| Router | `shop-router`, a reverse proxy (Express + `http-proxy-middleware`) that routes by path behind nginx or any CDN. A component in its own repository, not part of any unit, so a Shell deploy never touches routing |
| Edge | nginx locally on Docker, for HTTPS, compression and caching by each FES's `Cache-Control`; any CDN can replace it |
| Design system | Arun design system, `arun` brand. The Shell loads the brand's global CSS and the component CSS once; MFEs load no design-system CSS, so every unit uses the same `@arun-dev/ui` version |
| Unit styles | CSS Modules |
| Backend | Mocked with Express, one per unit: a stand-in for the separate backend service each environment has in production, not part of the unit. A `mock-backend/` in its unit's repository |
| Monitoring | New Relic: Node agent in the router, every FES and every mock backend, browser agent (Pro+SPA, which records browser traces and single-spa route changes) in the Shell, distributed tracing on. Servers leave health, version and static file requests out of transactions (the router traces only `/api/*`) and keep a whole trace when the caller kept it (`remote_parent_sampled: always_on`) |
| Client source maps | Every client builds hidden source maps (`build.sourcemap: 'hidden'`), uploads them to New Relic against each file's public URL on `PUBLIC_ORIGIN`, then deletes them from the build output |
| Server source maps | The router, every FES and every mock backend run its compiled output with `node --enable-source-maps`, so stacks sent to New Relic point at the TypeScript source |
| Unit tests | Vitest in every codebase, with React Testing Library for clients |
| End-to-end tests | Playwright, with axe for WCAG 2.2 AA, in `shop-shell/tests/e2e/` |

## Repositories

| Repository | Contents |
|---|---|
| [`shop-router`](https://github.com/arun9483/shop-router) | The router, with its health and version endpoints |
| [`shop-mfe-platform`](https://github.com/arun9483/shop-mfe-platform) | `docker/`: the Compose stack and nginx config that run the whole shop on one machine. No application code |
| [`shop-shell`](https://github.com/arun9483/shop-shell) | `client/`, `fes/`, `mock-backend/`, `server-common/`, `edge/`, `packages/shell-stub/`, `tests/e2e/` |
| [`shop-catalog`](https://github.com/arun9483/shop-catalog) | `client/`, `fes/`, `mock-backend/`, `server-common/` |
| [`shop-cart`](https://github.com/arun9483/shop-cart) | `client/`, `fes/`, `mock-backend/`, `server-common/` |

`tests/e2e/` is never deployed. `edge/` is the Cloudflare Worker that staging uses (see Deployment). `server-common/` holds the
servers' trace context, problem+json, health and version code; each repository keeps its own copy.
`packages/shell-stub/` is the one published package, a dev tool (see Standalone mode). The end-to-end tests need `shop-router` checked out and built next to `shop-shell`, or at
`SHOP_ROUTER_DIR`, and start every unit whose repository is checked out there and built, so they run on demand; each unit's own build and unit tests stay independent.

## Lazy loading

Code loads in three levels, each only when it is needed:

1. **Shell features.** The Shell's entry holds only the frame, top-level routing and the static MFE list. Each Shell feature is a `React.lazy` chunk.
2. **MFE entry.** single-spa loads `/mfe/<name>/entry.js` on the MFE's first route match. `entry.js` is the only unhashed script: a one-line re-export of the hashed entry (`export * from './entry-<hash>.js'`). The hashed entry is lean: it holds only the single-spa lifecycles (`bootstrap`, `mount`, `unmount`) and the MFE's route table, with no feature code.
3. **MFE features.** Once the entry is mounted, each feature of the MFE is a `React.lazy` chunk, loaded on its first use.

React and React DOM are a separate vendor chunk in each unit, cached by the browser. Sharing one copy across
units is not part of the MVP.

**Caching.** The Shell's `index.html` (also served for deep links), `entry.js` and `version.json` are served
with `Cache-Control: no-cache`, so a deploy is live on the next page load. Every hashed file is served with `Cache-Control: public, max-age=31536000, immutable`.
Because every file holding code has a unique URL, each keeps its own source map in New Relic across deploys
and rollbacks.

## Error logging

Errors are traced in every unit's client and FES, and in the mock backend each FES calls.

**Clients.** The Shell's browser agent is the only one on the page, so every client error is reported under
the Shell's browser app.

- Uncaught errors and rejections are captured by the agent automatically.
- Errors caught by an MFE's `errorBoundary` or the Shell's error boundary are reported with the standard `reportError(error)`, which raises a normal `error` event for the agent to capture. No client code calls the Shell's `newrelic` object, so MFEs receive nothing from the Shell beyond what the vision allows.
- An error's unit is derived from the file URL in its stack: `/mfe/<name>/…` is that MFE, anything else is the Shell. No unit attribute is set in code, so uncaught errors are attributed as reliably as caught ones.

**FES and mock backends.** Each one is its own New Relic app, named `<unit>-fes` or `<unit>-backend`.
The router is the app `shop-router`, and its transactions are named per unit: `api/catalog`, `api/cart` and
`api/shell`. A unit's `500` is reported by that unit alone; the router reports its own `502` and `504`.

- Every Express app ends with an error handler that responds `500` with problem+json (see API conventions); the agent records the error on that request's transaction.
- Errors that are handled without failing the request are reported with `newrelic.noticeError`.

**Linking.** Distributed tracing carries one trace from a unit's client through the router and its own FES
to its mock backend, so an error in any codebase can be followed to the request that caused it.

## Standalone mode

Each MFE also runs on its own, without the Shell app, on the Shell stub
[`@arun-shop/shell-stub`](https://www.npmjs.com/package/@arun-shop/shell-stub). Running together in the
Shell is unchanged.

- The stub's page loads what the Shell loads for every MFE: the design-system CSS, the Inter font, the
  Shell's base styles (the same `global.css`) and single-spa. It mounts the MFE through the same
  lifecycles and `domElement` prop, so the MFE's code is the same in both modes.
- The stub's server proxies `/mfe/<name>` and `/api/<name>` to the MFE's FES, so the browser uses one
  origin, as with the router. It uses neither the router nor nginx.
- The New Relic browser agent runs only when the stub is given the Shell's browser keys.
- It is built from `shop-shell/packages/shell-stub`, with the same design-system versions as the Shell,
  and published from there. The `@arun-shop` npm scope holds the shop platform's packages; `@arun-mfe`
  is kept for packages shared by every MFE platform.
- Each MFE repository adds it as a dev dependency with two scripts. Nothing of it is deployed.

| Script | Command (Catalog) | Serves the MFE from |
|---|---|---|
| `standalone` | `shell-stub --name catalog --mfe-url http://localhost:3002 --port 3102` | its FES: the built files, as in the Shell |
| `standalone:dev` | the same with `--dev` | its own Vite, run inside the stub: its source, with HMR and React Fast Refresh |

**Development mode.** With `--dev`, the stub loads the MFE's own Vite (from its `node_modules`, with its
`vite.config.ts`) and runs it in the stub's process: one process, one port, HMR on the stub's origin. The
API still goes to the FES. The MFE needs no extra Vite configuration; `--dev-root` (default `client`) and
`--dev-entry` (default `src/entry.tsx`) cover other layouts. Browser debugging works on the original
TypeScript through Vite's source maps.

**Ports.** Each stub needs one port: Catalog `3102`, Cart `3103`. A port in use stops the stub with a
message saying what to do; `--port 0` picks any free port; `SHELL_STUB_PORT` and `SHELL_STUB_MFE_URL`
override a script's flags, for other ports on one machine without editing `package.json`.

**Releases.** The stub is released with changesets, as `arun-design-system` is: a changeset in
`shop-shell` makes the Release workflow (`release.yml`) open a version PR, and merging it publishes to
npm through trusted publishing, with provenance, a tag and a GitHub release. The stub holds no MFE list,
so adding, removing or changing an MFE needs no new release. A release is needed when the stub itself
changes, and whenever the Shell moves to another `@arun-dev/ui` or `@arun-dev/tokens` version: the stub
must load the same design-system CSS as the Shell, and every MFE then moves to that stub version.

## Theme

The Shell's header has a theme switcher: light, dark or system (the default). It sets the design
system's theme mode, `data-theme="light"` or `"dark"` on `<html>`, or none for system, so the brand's
`prefers-color-scheme` rules decide. MFEs need no code for it: their design-system components and
tokens follow the attribute. The Shell's base styles set `color-scheme` to match, so native parts
(scrollbars, pickers) follow too.

- The choice is kept in `localStorage` (`shop-theme`); system is the absence of a saved value.
- An inline script in `index.html` applies it before the first paint, so the page never flashes the
  other theme.
- The Shell stub renders the same switcher, so every MFE can be checked in each theme standalone.

Giving MFEs the active theme as a read-only value (see the vision) is not part of the MVP yet.

## Timeouts

Every slow part fails visibly instead of waiting forever. Each unit sets its own values in its `.env`;
client values are read at build time.

| Part | Default | Variable | When it is slower |
|---|---|---|---|
| Shell loading an MFE entry | 5 s | `VITE_MFE_LOAD_TIMEOUT_MS` (Shell) | The Shell's fallback for that MFE |
| An MFE's bootstrap or mount | 3 s | `VITE_MFE_MOUNT_TIMEOUT_MS` (Shell) | The Shell's fallback for that MFE |
| A lazy feature chunk | 5 s | `VITE_LAZY_CHUNK_TIMEOUT_MS` (every client) | That unit's fallback |
| A client API call | 4 s | `VITE_API_TIMEOUT_MS` (every client) | The page says it could not load its data |
| An FES calling its backend | 3 s | `BACKEND_TIMEOUT_MS` (every FES) | `504` problem+json with the trace ID; readiness fails |

A mock backend calls nothing, so it has no timeout of its own: it is the slow party an FES waits on.

## Deployment

The whole shop runs on one machine with Docker, from `shop-mfe-platform`:

```
Browser → nginx (HTTPS, cache) → shop-router → FES → mock backend
```

Each repository's `deploy/build.sh` builds its images on any machine with Docker. nginx names no domain,
and only nginx is published. Each backend has its own Docker network, so only its own FES can reach it. The
stack reports to the New Relic apps without the `staging` suffix, on hosts that start with `docker-`.

**Pods.** Every unit runs one pod by default; `ROUTER_PODS` and `<UNIT>_PODS` set more.

- **No session affinity.** No server holds session state, sets a cookie or needs a request to return to it.
  The theme is in the browser. Real backends must keep state in a shared store, never in pod memory.
- **Even traffic.** nginx looks the router up in DNS every 2 s and moves to the next pod within 1 s if one does
  not answer. The router opens a new connection per request, because a reused one stays on its first pod.
- **Graceful shutdown.** On `SIGTERM` or `SIGINT`, a server stops taking connections, finishes its requests in
  flight for up to 8 s, sends New Relic what it holds and exits. It gets 10 s before it is killed.
- **Liveness, not readiness.** Docker's health checks use `/health/live`, because `/health/ready` fails when a
  backend is down, and removing every FES pod would take its files and `problem+json` errors with it.
- **Deploys.** `docker/deploy.sh <service>` replaces all of a service's pods at once, so a new `entry.js` never
  asks an old pod for a chunk it lacks. The unit is unavailable for a few seconds. A rollout with no gap needs
  hashed files in storage shared by every pod.
- **nginx** stays one container locally; production uses a managed CDN or load balancer. Staging keeps
  `--max-instances 1`, so none of this applies there yet.

The MVP is also deployed on free tiers. Staging comes first; production follows the same design later.
Staging still uses the Cloudflare Worker: moving it to `shop-router` is separate work, and until then a routing
change goes in both.

| Piece | Where |
|---|---|
| Each FES and each mock backend | Google Cloud Run, one service per unit and environment, from a container image built per commit |
| The one public origin | A Cloudflare Worker on a `workers.dev` subdomain, doing the router's path routing. `workers.dev` has no edge cache |
| Images | Google Artifact Registry, keeping only the latest few per service |
| The New Relic license key | Google Secret Manager, read by each service at start |

| Environment | Deployed | From |
|---|---|---|
| Local (Docker) | By hand, with `docker/build.sh` and `docker compose up` | Each repository's checkout, built for `PUBLIC_ORIGIN` |
| Staging (pre-prod) | On every push or merge to `main` after lint, typecheck and tests pass, or by hand from any branch | The chosen ref, built for staging |
| Production (later) | Only by hand, from `main` or a release tag | The chosen ref, built for production; never promoted from staging |

- **Independent units.** Each repository deploys its own unit: its FES, its mock backend and its client.
  `shop-shell` also deploys the Worker, whose targets are settings, so swapping an MFE is a settings
  change, as with the router.
- **Private backends.** A mock backend accepts only its own FES, which calls it with a Google identity
  token; the browser can never reach it.
- **One build per environment.** A client is built for its environment (its public origin and New Relic
  keys), so its source maps match the deployed URLs.
- **Keyless deploys.** GitHub Actions reaches Google Cloud through workload identity federation, with no
  stored key. The Cloudflare deploy uses a token limited to Workers scripts.
- **Costs.** At most one instance per service and none when idle; a budget alert and a spend cap guard
  the bill.
- **No infrastructure details in the repositories.** Project, account and service identifiers and all
  keys live in GitHub environment variables and secrets (`staging`, later `production`), not in code or
  docs.

## Configuration

Every unit is wired only through environment variables, loaded from `.env` with `node --env-file`. Only
`VITE_`-prefixed variables reach the browser.

| Variable | Used by | Local value |
|---|---|---|
| `PORT` | router, every FES and mock backend | router `3000`; FES: Shell `3001`, Catalog `3002`, Cart `3003`; mock backends: Shell `4001`, Catalog `4002`, Cart `4003` |
| `BACKEND_URL` | every FES | Shell `http://localhost:4001`, Catalog `http://localhost:4002`, Cart `http://localhost:4003` |
| `SHELL_URL` | router | `http://localhost:3001` |
| `MFE_CATALOG_URL`, `MFE_CART_URL` | router | `http://localhost:3002`, `http://localhost:3003` |
| `HEALTH_ENDPOINTS_ENABLED` | router | on unless `false` |
| `ROUTER_PODS`, `SHELL_PODS`, `CATALOG_PODS`, `CART_PODS` | `docker/.env`, for the Docker stack | `1` |
| `PUBLIC_ORIGIN` | source map upload in every client's build; `docker/.env` for the Docker stack | `http://localhost:3000`; on Docker `https://shop.test` |
| `NEW_RELIC_LICENSE_KEY` | router, every FES and mock backend | per developer |
| `NEW_RELIC_APP_NAME` | router, every FES and mock backend | `shop-router`, `shell-fes`, `catalog-fes`, `cart-fes`, `shell-backend`, `catalog-backend`, `cart-backend` |
| `NEW_RELIC_USER_API_KEY` | source map upload in every client's build | per developer |
| `NEW_RELIC_BROWSER_APP_ID` | source map upload in every MFE client's build: the Shell's browser app, which reports MFE errors | per developer |
| `VITE_NEW_RELIC_*` | Shell client | per developer (public keys only) |

## Ports

Local development only; on Docker only nginx is published, on `443`, and staging and production have one public
origin. The last digit names the unit:
`1` Shell, `2` Catalog, `3` Cart.

| Port | Server | Purpose |
|---|---|---|
| `3000` | Router | The one public origin: `/mfe/<name>` and `/api/<name>` to that MFE's FES, everything else to the Shell's FES |
| `3001` | Shell FES | The Shell client (`index.html`, deep links, `version.json`) and `/api/shell/*` |
| `3002` | Catalog FES | `/mfe/catalog/*` and `/api/catalog/*` |
| `3003` | Cart FES | `/mfe/cart/*` and `/api/cart/*` |
| `4001` | Shell mock backend | Called only by the Shell's FES |
| `4002` | Catalog mock backend | Called only by Catalog's FES |
| `4003` | Cart mock backend | Called only by Cart's FES |
| `3102` | Catalog's Shell stub | Catalog standalone, proxying to `3002`, or its own Vite with `--dev` |
| `3103` | Cart's Shell stub | Cart standalone, proxying to `3003`, or its own Vite with `--dev` |

A request in the Shell goes browser → `3000` → `300x` → `400x`; standalone, browser → `310x` → `300x` →
`400x`. A server's own health and version are reached on its port; `3000` answers for the router and, through
`/health/units` and `/version/units`, for every FES (see API conventions).

## Routing

The router gives the shop one public origin, `http://localhost:3000` locally and behind nginx on Docker. It routes by
path, so each client reaches its own FES directly:

- `/mfe/<name>`, `/api/<name>` and everything under them go to `MFE_<NAME>_URL`.
- Its own `/health/*` and `/version*` stay with the router and never reach a unit.
- Everything else, including `/api/shell/*`, goes to `SHELL_URL`.

The router forwards requests and headers, and adds its own span to the trace through the New Relic agent,
which `NEW_RELIC_ENABLED=false` turns off. Clients call their API with relative paths (`/api/<name>/…`), so no CORS is needed. The Shell's static
MFE list holds only name, route prefix and entry path `/mfe/<name>/entry.js`. Pointing `MFE_<NAME>_URL` at
another build swaps that MFE without touching the Shell.

## API conventions

These apply to every API route: `/api/*` on an FES and every route of a mock backend. `/health/*` and `/version`
follow only the health and version rules below.

**Trace context** ([W3C Trace Context](https://www.w3.org/TR/trace-context/)). The trace ID travels inside
`traceparent` (`00-<trace-id>-<parent-id>-<flags>`); there is no separate trace ID request header.

- Every API request carries `traceparent` and `tracestate`. The browser agent adds them to client calls and the Node agent to FES-to-backend calls; no unit code sets them.
- Every API response carries `traceresponse` (W3C Trace Context Level 2) with the same trace ID, set by a middleware from `newrelic.getTraceMetadata()`.
- Every error response is `application/problem+json` ([RFC 9457](https://www.rfc-editor.org/rfc/rfc9457)) with a `traceId` member, so a failed call can be found in New Relic from the response alone.

**Health and version.** Every server, the router included, answers on its own port with one contract, and
checks are left out of New Relic transactions (`rules.ignore`) so they do not drown real traffic.

| Endpoint | Responds with |
|---|---|
| `GET /health/live` | `200` while the process runs |
| `GET /health/ready` | `200` when it can serve traffic: an FES checks its mock backend's `/health/ready`; `503` otherwise |
| `GET /version` | name, version, commit and build time |

Health responses follow the IETF health check draft
([`draft-inadarei-api-health-check-06`](https://datatracker.ietf.org/doc/html/draft-inadarei-api-health-check-06)):
`application/health+json` with `status`, `version`, `releaseId` and `serviceId`. There is no equivalent
standard for `/version`, which uses SemVer and an ISO 8601 build time. Both are sent with `no-store`.

The router's `/health/live` and `/health/ready` never depend on a unit, so one failing MFE cannot take it out of
service. It also answers for every unit. These are public and on by default, so they carry only names, statuses and
versions; `HEALTH_ENDPOINTS_ENABLED=false` makes them `404`. The Worker on staging does not serve them.

| Endpoint | Responds with |
|---|---|
| `GET /health/units` | `pass`, `warn` when an MFE is down, or `503` with `fail` when the Shell is down, and a `<name>:ready` entry in `checks` per unit |
| `GET /health/units/<name>` | That FES's own `/health/ready` answer, or `503` with `fail` |
| `GET /version/units` | The router's version and every FES's |
| `GET /version/units/<name>` | That FES's `/version` answer, or `503` with `fail` |

Each FES gets 1 s to answer, and answers are reused for 3 s, so public callers cannot multiply calls to it. A
backend that is down shows in its FES's readiness. Every client build also
writes `version.json` with the `/version` fields: `/version.json` for the Shell, `/mfe/<name>/version.json` for an
MFE.

## Scope

| Unit | Routes | Demonstrates |
|---|---|---|
| Shell | `/`, `/help` | Frame, static MFE list, theme switcher; home and help as lazy features |
| Catalog | `/catalog`, `/catalog/:id` | Lean entry; product list and product detail as lazy features |
| Cart | `/cart` | Lean entry; cart as a lazy feature that loads from its backend |

Every codebase has a test trigger, used to check error logging and source maps:

- **Client:** a test route in each unit, built as a lazy feature so the error comes from a lazy chunk: `/test-error` (Shell), `/catalog/test-error`, `/cart/test-error`. `?kind=` picks the error:
  - `render`: the component throws while rendering; the error boundary catches it and reports it with `reportError`.
  - `event`: a button's click handler throws; the agent captures the uncaught error.
  - `promise`: a click starts a promise that rejects without a handler; the agent captures the unhandled rejection.
  - `fes`, `backend`: the page calls the unit's FES or mock backend trigger below, so the server error starts from a real browser request.
- **FES:** `GET /api/<name>/test-error` throws in that unit's FES, where `<name>` is `shell`, `catalog` or `cart`.
- **Mock backend:** `GET /api/<name>/test-error?in=backend` is forwarded by the FES to a backend route that throws.

## Success criteria

Verified by Playwright against `http://localhost:3000` with all units running, and through a running stack with `E2E_BASE_URL`; each server's own health and version are checked on its own port. Specs that need a server's own port or start their own servers run only locally. Criteria 7 and 8 are the exception: they are checked by hand in New Relic, because its data arrives minutes after the errors and traces are made.

1. Each unit runs from its own repository. Pointing the router's `MFE_<NAME>_URL` at another build swaps that MFE without rebuilding the Shell.
2. On `/`, the Shell's home chunk loads, but no other Shell feature chunk and no MFE code; the help chunk loads on the first visit to `/help`.
3. On `/catalog`, Catalog's entry loads, then only its list chunk; the detail chunk loads on the first visit to `/catalog/:id`. Leaving and returning to Catalog loads nothing again.
4. No entry contains feature code.
5. A `render` test error shows its unit's fallback: in an MFE, the Shell keeps working; in the Shell, the frame keeps working.
6. The browser talks only to the public origin and the New Relic browser agent's own endpoints, each client's API calls reach only its own FES, and no `.map` file is served.
7. Every test trigger's error, including every client `kind`, appears in New Relic under its codebase's app (client errors attributed to their unit by file URL), and its stack resolves to the original TypeScript file, line and function.
8. For each unit, `kind=backend` produces one trace ID shared by the unit's client (reported under the Shell's browser app), the router, its FES and its mock backend, whose span carries the error: Shell client → `shop-router` → `shell-fes` → `shell-backend`; Catalog client → `shop-router` → `catalog-fes` → `catalog-backend`; Cart client → `shop-router` → `cart-fes` → `cart-backend`. `shop-router` is not marked as an error by a unit's `500`.
9. No axe violations at WCAG 2.2 AA on every route, in the light and the dark theme.
10. Every API response carries a `traceresponse` whose trace ID matches the request's `traceparent`, and every error response is `application/problem+json` with that `traceId`.
11. The router, every FES and every mock backend answer `/health/live`, `/health/ready` and `/version` with the same contract; an FES's readiness fails when its mock backend is down, and the router's own health does not. Through the public origin, `/health/units` and `/version/units` answer for every unit, and `/health/units/<name>` and `/version/units/<name>` for one. Every client serves its `version.json`.
12. Every timeout in the Timeouts table fails as listed, for every unit, and the rest of the shop keeps working.
13. Every MFE also runs standalone on the Shell stub with only its own servers: it loads lazily, looks as it does in the Shell, switches theme as in the Shell, keeps its in-page links working and shows its fallbacks. In development mode its source runs through its own Vite in the stub, with HMR connected on the stub's origin.
14. The theme switcher picks light, dark or system, in the Shell and on the Shell stub; the choice survives a reload with no flash of the other theme, and system follows the operating system.
