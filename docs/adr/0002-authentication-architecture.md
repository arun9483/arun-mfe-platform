# ADR 0002: Authentication architecture

- **Status:** Proposed
- **Date:** 2026-11-11

## Context

MVP-1 has no users. MVP-2 adds single sign-on with Keycloak as the identity provider (IdP), on top of
MVP-1 and off by default ([mvp-2-sso.md](../mvp-2-sso.md)). The design must:

- keep every unit independent, with no server holding a session, so any pod serves any request;
- prove the user's identity on every hop: browser → FES → backend;
- let each unit decide its own access, failing closed;
- keep tokens away from page code, and give MFEs nothing from the Shell beyond what the vision allows;
- use open-source npm packages for authentication, token management and sign-out;
- let a backend learn who the user is, by `sub`, using **its own** service access token rather than reusing
  the user's token.

## Options

**A. Tokens in the browser.** A public client with `keycloak-js`: the browser holds the access and refresh
tokens and sends a bearer token on each call.

**B. Shell-owned sign-in, full access token.** The Shell FES signs users in; the access token carries the
profile and roles, so units authorize from the token alone.

**C. Shell-owned sign-in, lightweight access token, lookup by `sub` (chosen).** The Shell FES signs users in.
The access token carries only `sub`, `sid`, `iss`, `exp`, `iat`, `jti`, `azp` and `aud`. Each FES checks it
locally and gates on `/userinfo`; each backend checks it locally, takes `sub`, and looks up the user and its
roles at the IdP with its own service access token.

**D. As C, with introspection in each backend.** The backend sends the user's token to the IdP's
introspection endpoint and gets `active`, profile and roles in one call.

**E. The ID token as the bearer.** Units accept the Shell's ID token, which carries the profile.

**F. Each FES signs its own token.** The FES checks the ID token and signs a short-lived JWT for its backend
with its own key.

**G. Token exchange per FES.** Each FES exchanges the user's token at the IdP for one aimed only at its
backend.

## Comparison

| | A | B | C | D | E | F | G |
|---|---|---|---|---|---|---|---|
| Tokens readable by page code | Yes | No | No | No | No | No | No |
| MFEs need a token from the Shell | Yes, against the vision | No | No | No | No | No | No |
| IdP calls per request | None | None | FES and backend, cached | Backend, cached | None | None | One per FES, cached |
| Sign-out reaches units | At token expiry | At token expiry | FES within 30 s, backend within 5 min | Backend within its cache | At ID token expiry, no `/userinfo` | At ID token expiry | At token expiry |
| Backend reuses the user's token | — | — | No: looks up by `sub` with its own token | Yes, sent to the IdP | — | — | Yes, to the IdP |
| Audience per backend | Yes | Yes | Yes | Yes | No: `aud` is the sign-in client | Yes | Yes |
| Personal data in a cookie | — | Profile and roles | `sub` only | `sub` only | Profile | Profile | `sub` only |
| New signing keys or clients | No | No | No | No | No | A key per FES | A client per FES |
| Standard use of tokens | Yes | Yes | Yes | Yes | No: APIs take access tokens | Yes | Yes |

## Decision

**Option C.**

- **Sign-in.** Only the Shell FES signs users in, with the client `shell-fes`; FESes have no client of their
  own. Tokens live in `HttpOnly` cookies ([ADR 0003](0003-session-cookies.md)).
- **FES.** Checks the lightweight access token locally with `jose` (signature, `iss`, `exp`, `azp`, token
  type), gates on `/userinfo` cached per `sid`, and forwards the token to its backend.
- **Backend.** Checks the token locally (signature, `iss`, `exp`, its own name in `aud`, token type), takes
  `sub`, and gets the user and this unit's roles through the IdP adapter ([ADR 0004](0004-idp-adapter.md))
  with its own service access token. It never sends the user's token on.
- **Token type.** Every FES and backend accepts only access tokens as a bearer, so an ID token can never
  call an API.
- **Every route protected** when `AUTH_ENABLED=true`; hashed files stay public.
- **`/api/<unit>/me`** in every unit, with OIDC field names and only that unit's roles.
- **Each repository owns its IdP settings**, applied by `keycloak-config-cli`.
- **Standalone MFEs** use a development-only bypass instead of the IdP, refused in production.
- **Operators** read `/health/units*` and `/version/units*` with a service access token holding `ops`.

C keeps tokens out of page code, keeps the token on the wire small and free of personal data, and lets every
backend learn the user from the IdP with its own identity.

## Consequences

- Every FES and backend depends on the IdP for each uncached check. Checks fail closed, so an IdP outage
  stops the shop. Caches (about 30 s in a FES, 30–60 s in a backend) limit the load.
- After sign-out, a backend still accepts the user's token until it expires, at most 5 min.
- Every backend's service account can read all users (`view-users`).
- The three cookies are an exception to MVP-1's "no server sets a cookie" rule.
- MVP-1's criterion 6 gains the IdP's origin, during sign-in and sign-out only.

## Revisit when

- IdP lookups prove too slow or too costly: move roles into the token (B) for the units that need it.
- A backend must see sign-out at once: introspection (D) for that backend.
- `view-users` is too broad: Keycloak's fine-grained admin permissions, per backend.
- Entitlement arrives: it adds a check at each layer, on the same tokens.
