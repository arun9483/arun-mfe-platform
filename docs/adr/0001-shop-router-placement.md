# ADR 0001: Keep shop-router separate from shop-shell

- **Status:** Accepted
- **Date:** 2026-11-03

## Context

Every request goes Browser → CDN or nginx → `shop-router` → FES → backend. The router routes by path:
`/mfe/<name>` and `/api/<name>` go to that MFE's FES, and everything else goes to the Shell FES. Neither the
router nor any FES holds session state, so both can run as many pods as needed.

The question is whether the Shell FES should take over routing on its own port, removing the
`shop-router` service to save infra cost.

The comparison assumes the planned direction:

- **Registry:** one per environment, holding name, `routePrefix` and upstream URL for each MFE. Each MFE's
  deploy writes its own entry. The router and Shell backend read it at start and poll it, keeping a
  last-known-good copy.
- **Entitlement:** a shared system, checked in three places:
  - **Browser:** frontend code calls an entitlement endpoint on the FES, which proxies to the entitlement
    system. This decides what the UI shows.
  - **FES:** checks before serving an MFE's `entry.js` and API calls.
  - **Backend:** checks every data call.

  The FES and backend checks are what protect data, and both refuse access if the check fails. The CDN
  does no check.

## Options

**A. Separate (current).** `shop-router` is its own service and repository.

**B. Merged.** The Shell FES proxies `/mfe/<name>` and `/api/<name>`, and serves its own files and
`/api/shell` in the same process.

## Comparison

### Performance

| | A: separate | B: merged |
|---|---|---|
| Hops for Shell requests (`index.html`, MFE list, `/api/shell`) | CDN → router → Shell FES | CDN → Shell FES |
| Hops for MFE requests | CDN → router → MFE FES | Same |
| Requests that skip a hop | None | Shell requests only: the page, the MFE list and Shell API calls. MFE files and API calls, most of the traffic, keep the same hops |
| Cold starts on staging (scale to zero) | Router and Shell FES on the first page | One fewer on the first page; MFE entries unchanged |

For example, a user opens `/catalog`, makes 5 Catalog API calls, goes to `/cart`, makes 2 Cart API calls
and 1 Shell API call. These 12 requests reach the server:

| Request | Count | Skips a hop in B? |
|---|---|---|
| `index.html` | 1 | Yes |
| `/api/shell/mfes` (the MFE list) | 1 | Yes |
| `/api/shell/*` | 1 | Yes |
| `/mfe/catalog/entry.js`, `/mfe/cart/entry.js` | 2 | No |
| `/api/catalog/*` | 5 | No |
| `/api/cart/*` | 2 | No |

Hashed assets are cached at the edge in both options, so neither the router nor a FES sees them after the
first request per release. The extra hop in A has not been measured.

### Security

| | A: separate | B: merged |
|---|---|---|
| Public-facing process can reach a backend | No | Yes: it reaches the Shell backend and holds its identity token |
| Entitlement checks | Browser (through the FES entitlement endpoint), FES and backend | Same |
| Registry exposure | Read only server-side, from a private store | Same |
| `/health/units`, `/version/units` | Internal-only or operator token | Same |

### Independent release and deployment

| | A: separate | B: merged |
|---|---|---|
| A Shell release redeploys the routing process | Never | Every time. The routing table still comes from the registry, but the proxy code, its dependencies (Express, `http-proxy-middleware`, Node) and the process running it ship in the Shell image |
| Deploy gap with all pods replaced at once | Shell paths fail; MFE paths keep working | Every path fails for a few seconds |
| With blue-green deploys | No gap | No gap, if the image also holds the previous release's hashed assets |
| A Shell bug, leak or blocked event loop at runtime | Affects Shell paths | Affects routing for every unit |
| Adding or swapping an MFE (with the registry) | A registry entry; no other deploy | Same |
| Owner of routing | Platform | Shell team |

When the Shell is down, new page loads fail in both options, since `index.html` comes from it. The
difference is users who already have the app open: in A their MFE keeps loading chunks and calling its
API; in B those requests fail too.

### Infra cost

| | A: separate | B: merged |
|---|---|---|
| Staging | Router scales to zero: about no cost | About the same |
| Production | Router baseline pods plus Shell pods | Router baseline pods saved, but Shell pods must carry all uncached traffic |
| Scaling | Router and Shell scale separately | One setting; Compose also ties `shell-backend` replicas to `SHELL_PODS`, which must be split |
| Pipelines, repositories, monitoring apps | One more of each | One fewer of each |

Total traffic is the same in both options. B saves only the router's fixed baseline, not a share of the
load.

### Work needed to merge

- **Readiness:** split `/health/ready`. The router's check must never depend on a unit, while the Shell's
  fails when its backend is down.
- **New Relic errors:** the router ignores `404` and `500`, so a unit's failure isn't counted twice, but the
  Shell must report its own.
- **New Relic transactions:** combine the rules for which requests become transactions.
- **Scaling:** separate the Shell backend's replica count from the Shell FES's.
- **Deploy wiring:** point nginx, the Worker and the Shell deploy at the merged service.
- **Tests:** rework the swap and e2e setup.
- **MVP criteria:** update criteria 8 (trace) and 11 (health and version).

## Decision

**Keep `shop-router` and `shop-shell` separate (option A).**

- **Separation of concerns:** the Shell owns the frame and its features, and the router owns routing.
  Merging redeploys the routing process with every Shell release, and lets a Shell fault break routing
  for every unit.
- **Small saving:** about nothing on staging, and only the router's baseline pods in production.
- **Small gain:** merging removes a hop on Shell requests only.
- **No consistency gain:** with a platform-owned registry, the router reads the registry directly.

## Router hop latency

Option A adds one hop to every MFE request. It has not been measured. In order:

1. **Fix the connection cost.** The router uses `agent: false`, so it opens a new connection per request.
   That is cheap over plain HTTP on Docker, but on staging every FES is `https://*.run.app`, so each API
   call likely pays a TLS handshake. Make it configurable per environment: keep-alive on Cloud Run, whose
   front end balances each request across instances, and `agent: false` on Docker, where reused
   connections pin traffic to one pod.
2. **Measure** `/api/<name>` through the router and directly to its FES, before and after the fix, with
   New Relic spans or `curl -w '%{time_total}'`. Keep the router and every FES in one region and network.
3. **Only if the hop still matters,** route by path at the edge (the Cloudflare Worker reading the
   registry, a CDN or a load balancer), so `/mfe/<name>/*` and `/api/<name>/*` go straight to the FES.
   This keeps one origin, but the edge config must be generated from the registry, and the operator
   health endpoints and per-unit New Relic names need a new home.

**Rejected:** the browser calling each FES on its own subdomain. CORS preflights, an extra connection
per origin, cookies across sites, a public entry point per FES and breaking MVP criterion 6 (one origin)
cost more than the hop they save.

## Consequences

- One more service, pipeline and monitoring app to run.
- Routing stays platform-owned, and MFE changes go through the registry, not through a Shell deploy.
- Blue-green deploys, with the previous release's hashed assets kept, remain worth adding to both the
  router and the Shell.

## Revisit when

- Production cost data shows the router's baseline pods are significant, **and**
- Blue-green deploys with the previous release's assets are in place, **and**
- The shared fate described above is accepted.

If merged, the work listed under "Work needed to merge" is part of the change.
