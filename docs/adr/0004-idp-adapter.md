# ADR 0004: IdP adapter

- **Status:** Accepted
- **Date:** 2026-11-11

## Context

Most of MVP-2's authentication is standard and works with any compliant identity provider (IdP): sign-in
with authorization code and PKCE, refresh, sign-out (RP-Initiated Logout), discovery, `/userinfo`, client
credentials, and checking JWTs against the IdP's public keys.

Four things are specific to Keycloak:

- **The user by `sub`:** Keycloak's admin REST API.
- **The user's roles by `sub`:** the admin API's role mappings.
- **Roles in `/userinfo`:** Keycloak's claim layout, `resource_access.<client>.roles`.
- **Telling an access token from an ID token:** Keycloak's `typ` claim (`Bearer`, `ID`, `Refresh`). RFC 9068's
  header `typ: at+jwt` is not Keycloak's default.

The platform is meant to stay vendor-agnostic, with vendor specifics only in configuration and deploy steps,
as MVP-1 does for the cloud.

## Options

**A. Call Keycloak directly** wherever needed.

**B. One adapter per IdP (chosen).** A small interface in `server-common/`, with a Keycloak implementation.

**C. SCIM 2.0** (RFC 7643, 7644), the standard user API. Keycloak has no built-in SCIM, only community
extensions, and SCIM does not cover token types or role claims.

## Decision

**Option B.**

```ts
export interface IdpAdapter {
  getUser(sub: string): Promise<IdpUser | undefined>;               // backend
  getUnitRoles(sub: string, unit: string): Promise<string[]>;       // backend
  rolesFromClaims(claims: Record<string, unknown>, unit: string): string[];                       // FES
  isAccessToken(header: Record<string, unknown>, claims: Record<string, unknown>): boolean;       // FES, backend
}
```

- `IDP_ADAPTER` chooses the adapter; MVP-2 has one, `keycloak`.
- Nothing outside the adapter calls an IdP-specific endpoint or reads an IdP-specific claim. Everything else
  uses `OIDC_ISSUER_URL`, OIDC Discovery and standard claims.
- The adapter caches per `sub` and fails closed: an IdP error is `503`, never "allow".
- Every adapter passes one set of contract tests; the Keycloak adapter also runs against a real Keycloak.

## Consequences

- Moving to another IdP means a new adapter and that IdP's settings files, with no change to the units'
  flows, cookies or checks.
- The IdP settings themselves (`keycloak/<repository>.yaml`) stay specific to Keycloak, as cloud details are
  specific to their cloud.
- Another IdP must allow a refresh token to be used more than once (or rotate it with a reuse window),
  support RP-initiated logout, and issue a small access token with `sub` and an `aud` per backend.

## Revisit when

- A second IdP is needed: write its adapter against the contract tests.
- Keycloak gains built-in SCIM: `getUser` may use it.
