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
| Router | Local reverse proxy (Express + `http-proxy-middleware`), the stand-in for a production CDN's path routing; not monitored. A dev-only tool in `shop-shell/router/`, not part of any unit and never deployed |
| Design system | Arun design system, `arun` brand. The Shell loads the brand's global CSS and the component CSS once; MFEs load no design-system CSS, so every unit uses the same `@arun-dev/ui` version |
| Unit styles | CSS Modules |
| Backend | Mocked with Express, one per unit: a stand-in for the separate backend service each environment has in production, not part of the unit. A dev-only `mock-backend/` in its unit's repository, never deployed |
| Monitoring | New Relic: Node agent in every FES and mock backend, browser agent (Pro+SPA, which records browser traces and single-spa route changes) in the Shell, distributed tracing on. Servers leave health, version and static file requests out of transactions and keep a whole trace when the caller kept it (`remote_parent_sampled: always_on`) |
| Client source maps | Every client builds hidden source maps (`build.sourcemap: 'hidden'`), uploads them to New Relic against each file's public URL on the router's origin, then deletes them from the build output |
| Server source maps | Every FES and mock backend runs its compiled output with `node --enable-source-maps`, so stacks sent to New Relic point at the TypeScript source |
| Unit tests | Vitest in every codebase, with React Testing Library for clients |
| End-to-end tests | Playwright, with axe for WCAG 2.2 AA, in `shop-shell/tests/e2e/` |

## Repositories

| Repository | Contents |
|---|---|
| [`shop-shell`](https://github.com/arun9483/shop-shell) | `client/`, `fes/`, `mock-backend/`, `server-common/`, `router/`, `packages/shell-stub/`, `tests/e2e/` |
| [`shop-catalog`](https://github.com/arun9483/shop-catalog) | `client/`, `fes/`, `mock-backend/`, `server-common/` |
| [`shop-cart`](https://github.com/arun9483/shop-cart) | `client/`, `fes/`, `mock-backend/`, `server-common/` |

`mock-backend/`, `router/` and `tests/e2e/` are dev-only and never deployed. `server-common/` holds the
servers' trace context, problem+json, health and version code; each repository keeps its own copy.
`packages/shell-stub/` is the one published package, a dev tool (see Standalone mode). The end-to-end tests start every unit whose repository is checked out next to
`shop-shell` and built, so they run on demand; each unit's own build and unit tests stay independent.

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

- Every Express app ends with an error handler that responds `500` with problem+json (see API conventions); the agent records the error on that request's transaction.
- Errors that are handled without failing the request are reported with `newrelic.noticeError`.

**Linking.** Distributed tracing carries one trace from a unit's client through its own FES to its mock
backend, so an error in any codebase can be followed to the request that caused it.

## Standalone mode

Each MFE also runs on its own, without the Shell app, on the Shell stub
[`@arun-shop/shell-stub`](https://www.npmjs.com/package/@arun-shop/shell-stub). Running together in the
Shell is unchanged.

- The stub's page loads what the Shell loads for every MFE: the design-system CSS, the Inter font, the
  Shell's base styles (the same `global.css`) and single-spa. It mounts the MFE through the same
  lifecycles and `domElement` prop, so the MFE's code is the same in both modes.
- The stub's server proxies `/mfe/<name>` and `/api/<name>` to the MFE's FES, so the browser uses one
  origin, as with the router.
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

The MVP is deployed on free tiers. Staging comes first; production follows the same design later.

| Piece | Where |
|---|---|
| Each FES and each mock backend | Google Cloud Run, one service per unit and environment, from a container image built per commit |
| The one public origin | A Cloudflare Worker on a `workers.dev` subdomain, doing the router's path routing and caching hashed files at the edge |
| Images | Google Artifact Registry, keeping only the latest few per service |
| The New Relic license key | Google Secret Manager, read by each service at start |

| Environment | Deployed | From |
|---|---|---|
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
| `NEW_RELIC_LICENSE_KEY` | every FES and mock backend | per developer |
| `NEW_RELIC_APP_NAME` | every FES and mock backend | `shell-fes`, `catalog-fes`, `cart-fes`, `shell-backend`, `catalog-backend`, `cart-backend` |
| `NEW_RELIC_USER_API_KEY` | source map upload in every client's build | per developer |
| `NEW_RELIC_BROWSER_APP_ID` | source map upload in every MFE client's build: the Shell's browser app, which reports MFE errors | per developer |
| `VITE_NEW_RELIC_*` | Shell client | per developer (public keys only) |

## Routing

The router is the one public origin (`http://localhost:3000`), as a CDN would be in production. It routes by
path, so each client reaches its own FES directly:

- `/mfe/<name>`, `/api/<name>` and everything under them go to `MFE_<NAME>_URL`.
- Everything else, including `/api/shell/*`, goes to `SHELL_URL`.

The router forwards requests and headers unchanged, including the trace headers, and is not part of any
trace. Clients call their API with relative paths (`/api/<name>/…`), so no CORS is needed. The Shell's static
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

**Health and version.** Served on each server's own port, not through the router, and left out of New
Relic transactions (`rules.ignore`) so checks do not drown real traffic.

| Endpoint | Responds with |
|---|---|
| `GET /health/live` | `200` while the process runs |
| `GET /health/ready` | `200` when it can serve traffic: an FES checks its mock backend's `/health/ready`; `503` otherwise |
| `GET /version` | name, version, commit and build time |

Health responses use `application/health+json` with `status` `pass` or `fail`. Every client build also
writes `version.json` with the same fields: `/version.json` for the Shell, `/mfe/<name>/version.json` for an
MFE.

## Scope

| Unit | Routes | Demonstrates |
|---|---|---|
| Shell | `/`, `/help` | Frame, static MFE list; home and help as lazy features |
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

Verified by Playwright against `http://localhost:3000` with all units running; health and version endpoints are checked on each server's own port. Criteria 7 and 8 are the exception: they are checked by hand in New Relic, because its data arrives minutes after the errors and traces are made.

1. Each unit runs from its own repository. Pointing the router's `MFE_<NAME>_URL` at another build swaps that MFE without rebuilding the Shell.
2. On `/`, the Shell's home chunk loads, but no other Shell feature chunk and no MFE code; the help chunk loads on the first visit to `/help`.
3. On `/catalog`, Catalog's entry loads, then only its list chunk; the detail chunk loads on the first visit to `/catalog/:id`. Leaving and returning to Catalog loads nothing again.
4. No entry contains feature code.
5. A `render` test error shows its unit's fallback: in an MFE, the Shell keeps working; in the Shell, the frame keeps working.
6. The browser talks only to the router's origin and the New Relic browser agent's own endpoints, each client's API calls reach only its own FES, and no `.map` file is served.
7. Every test trigger's error, including every client `kind`, appears in New Relic under its codebase's app (client errors attributed to their unit by file URL), and its stack resolves to the original TypeScript file, line and function.
8. For each unit, `kind=backend` produces one trace ID shared by the unit's client (reported under the Shell's browser app), its FES and its mock backend, whose span carries the error: Shell client → `shell-fes` → `shell-backend`; Catalog client → `catalog-fes` → `catalog-backend`; Cart client → `cart-fes` → `cart-backend`. The router does not appear in the trace.
9. No axe violations at WCAG 2.2 AA on every route.
10. Every API response carries a `traceresponse` whose trace ID matches the request's `traceparent`, and every error response is `application/problem+json` with that `traceId`.
11. Every FES and mock backend answers `/health/live`, `/health/ready` and `/version`; an FES's readiness fails when its mock backend is down. Every client serves its `version.json`.
12. Every timeout in the Timeouts table fails as listed, for every unit, and the rest of the shop keeps working.
13. Every MFE also runs standalone on the Shell stub with only its own servers: it loads lazily, looks as it does in the Shell, keeps its in-page links working and shows its fallbacks. In development mode its source runs through its own Vite in the stub, with HMR connected on the stub's origin.
