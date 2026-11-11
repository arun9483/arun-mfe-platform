# ADR 0005: Backend-to-backend access

- **Status:** Accepted
- **Date:** 2026-11-11

## Context

The vision combines data across domains on the backend, never in the browser. For example, Cart's backend
reads product names from Catalog's backend for the user viewing their cart. The call must tell Catalog:

- **which service is calling**, so Catalog decides who may use its API and always knows its consumers;
- **which user it is for**, by `sub`.

A backend never forwards the user's lightweight access token ([ADR 0002](0002-authentication-architecture.md)).

## Options

**A. Service access token and `sub` (chosen).** The caller sends its own service access token as the bearer
and the user's `sub` in the request.

**B. Token exchange** (RFC 8693). The caller exchanges the user's token at the IdP for one with `sub` = the
user, `azp` = the caller, `aud` = the provider.

**C. Service access token and the user's token** in a second header.

**D. Forward the user's token** as the bearer.

## Comparison

| | A | B | C | D |
|---|---|---|---|---|
| Who vouches for the user | The calling service | The IdP | The IdP's signature on the user's token | The IdP |
| A compromised caller can act as | Any user, within its granted API roles | Only users whose tokens it holds | Only users whose tokens it holds | Any user whose token it holds, at any backend |
| Provider knows its consumers | Yes: API roles granted in its own settings | Yes | Yes | No |
| Reuses the user's token | No | Yes, sent to the IdP | Yes | Yes |
| IdP calls | Client credentials, cached | An exchange per user token, cached | Client credentials | None |
| Standard | OAuth 2.0 client credentials | RFC 8693 | A custom header | OAuth 2.0 bearer |

## Decision

**Option A.**

- Each provider defines **API roles** on its backend's client, for example `catalog-api:read-products`, and
  grants them to consumers' service accounts in its own `keycloak/` file. Each file is the full list of that
  unit's consumers.
- The provider checks the caller's token with `jose`: its own name in `aud` and the API role. Anything else is
  refused.
- Per endpoint, the provider decides whether it trusts the caller for the `sub` it sends, or also checks the
  user's own roles in its unit through the IdP adapter. Its API documentation says which.

A client still calls only its own FES. Links between units stay plain URLs, and events between MFEs carry only
IDs.

## Amended after phase 4 (2026-11-12)

- Built as Cart's backend reading names and prices from Catalog's `GET /service/products`. The user is sent in
  `X-Shop-User-Sub` and recorded only; Catalog trusts the caller for `read-products`.
- A service route accepts only service access tokens (no `sid`), and user routes only user tokens (with a `sid`).
- A token without the provider in `aud` is refused with `401`: Keycloak puts the provider there only when the
  caller holds one of its roles. `403` is for a token that names the provider but lacks the route's role.
- On Docker the provider and its granted callers share one network, `catalog-api`. A network is two-way.

## Consequences

- A compromised backend can act for any user, but only through the API roles granted to it. Keep each API
  role narrow, and check the user's own roles on endpoints that change data.
- Consumers are visible in three places: the provider's settings file, the IdP's role grants, and New Relic's
  service map.
- Contract tests between consumer and provider (for example Pact) are a later option, to see what a breaking
  change would break.

## Revisit when

- A cross-domain call writes sensitive per-user data: token exchange (B) for that call, so the IdP vouches for
  the user.
