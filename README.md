# arun-mfe-platform

Micro frontend platform and infrastructure analysis.

This repository contains **only documentation and figures**. Code lives in separate repositories.

- [Vision](docs/vision.md): requirements that are independent of technology, framework and library.
- [MVP](docs/mvp.md): the MVP (POC) and the technology decisions behind it. Frozen at tag `mvp-1`.
- [MVP-2: single sign-on](docs/mvp-2-sso.md): sign-in with an identity provider, built on MVP-1.

## Architecture decisions

| ADR | Decision | Status |
|---|---|---|
| [0001](docs/adr/0001-shop-router-placement.md) | Keep `shop-router` separate from `shop-shell` | Accepted |
| [0002](docs/adr/0002-authentication-architecture.md) | Authentication architecture | Proposed |
| [0003](docs/adr/0003-session-cookies.md) | Session cookies | Proposed |
| [0004](docs/adr/0004-idp-adapter.md) | IdP adapter | Proposed |
| [0005](docs/adr/0005-backend-to-backend-access.md) | Backend-to-backend access | Proposed |
