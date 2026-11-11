# ADR 0003: Session cookies

- **Status:** Proposed
- **Date:** 2026-11-11

## Context

The Shell FES receives three tokens at sign-in ([ADR 0002](0002-authentication-architecture.md)):

- the **lightweight access token**, which every FES needs on every request;
- the **refresh token**, which only the Shell FES needs, to refresh;
- the **ID token**, which only the Shell FES needs, to sign out with `id_token_hint`.

Requirements:

- No server holds a session, so the tokens travel in cookies.
- No token is readable by page code.
- Each token reaches only the servers that need it.
- Units do not share secrets or library versions with the Shell.
- Open-source, well-tested code does the protocol work.

## Options

**A. One cookie with `express-openid-connect`.** The library's encrypted session cookie holds all three
tokens. Every FES reads it with the same library and secret.

**B. Unencrypted cookie readable by page code.** So the Shell can read `sub` for monitoring.

**C. Two cookies with `openid-client`.** `shop_at` holds the access token; `shop_rt` holds the refresh token
and ID token together, encrypted, on `Path=/auth`.

**D. Three cookies with `openid-client` (chosen).** One cookie per token:

| Cookie | Holds | `Path` |
|---|---|---|
| `shop_at` | Lightweight access token, not encrypted | `/` |
| `shop_rt` | Refresh token, encrypted | `/auth` |
| `shop_id` | ID token, encrypted | `/auth/logout` |

All are `HttpOnly; Secure; SameSite=Lax`.

**E. As D, without keeping the ID token.** Sign-out sends `client_id` instead of `id_token_hint`.

## Comparison

| | A | B | C | D | E |
|---|---|---|---|---|---|
| Tokens readable by page code | No | Yes: an XSS steals the refresh token | No | No | No |
| Refresh token sent to | Every FES, every request | Every FES | Shell FES, `/auth/*` | Shell FES, `/auth/*` | Shell FES, `/auth/*` |
| ID token sent to | Every FES, every request | Every FES | Shell FES, `/auth/*` | Shell FES, `/auth/logout` only | Not kept |
| Secret shared by every FES | Yes, which also decrypts the refresh token | No | No | No | No |
| Library version shared by every unit | Yes, the cookie format | No | No | No | No |
| Size on every request | Three tokens, possibly split into chunks | Large | One token | One token | One token |
| Sign-out | `id_token_hint` | — | `id_token_hint` | `id_token_hint` | The IdP's confirmation page |
| Sign-in code written by us | None | — | Routes and cookies | Routes and cookies | Routes and cookies |

`express-openid-connect` is built on `openid-client`. The protocol and checks it relies on (discovery, PKCE,
`state` and `nonce`, the code exchange, ID token checks, refresh, the end-session URL) all come from
`openid-client`, which is OpenID-certified. What the higher library adds is cookie handling, the part D needs
to work differently.

## Decision

**Option D: three cookies with `openid-client`.**

- `shop_at` is set with `Path=/` explicitly; a cookie set by an `/auth/*` response otherwise defaults to
  `Path=/auth`.
- The three cookies are set together at sign-in and refresh, and cleared together at sign-out and on a failed
  refresh. Each is cleared with its own `Path`.
- `shop_rt` and `shop_id` are encrypted with `jose` using `AUTH_COOKIE_KEY`, held only by the Shell FES.
  `shop_at` is a signed token any FES checks, carrying no personal data beyond `sub`, so it is not encrypted.
- `sub` reaches monitoring through `/api/shell/me`, not through a readable cookie.
- Without `shop_id`, sign-out falls back to `client_id`, and the IdP asks the user to confirm.

## Consequences

- The Shell FES owns about 200 lines of sign-in code: the four `/auth/*` routes, a short-lived encrypted
  cookie per sign-in attempt (`shop_tx_<state>`) for `state`, `nonce`, the PKCE verifier and `returnTo`, and
  the three cookies. It needs focused tests: `returnTo` must be a path on the shop's origin; `Origin` is
  checked on `POST /auth/refresh` and `POST /auth/logout`; cookie attributes; parallel refreshes; sign-ins
  started in several tabs.
- Every tab shares the cookies. Tabs coordinate refresh with a Web Lock and sign-in and sign-out with a
  `BroadcastChannel` ([mvp-2-sso.md, Several tabs](../mvp-2-sso.md#several-tabs)).
- No FES needs the Shell's secret or its library version; each checks `shop_at` with `jose`.
- Units hold no session, so any pod serves any request.

## Revisit when

- A maintained library supports separate cookies per token: replace the Shell FES's own routes with it.
- Tokens grow past about 4 KB per cookie.
