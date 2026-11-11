# Vision — Shell App + Micro Frontends

A frontend platform where a **Shell App** and independently owned **micro frontends (MFEs)** are built,
deployed and evolved on their own, yet work together as one product. This document states **what** the
platform must do, independent of technology. Technology choices are in [mvp.md](mvp.md).

A **unit** is the Shell or one MFE: a client plus its own **Front End Server (FES)**.

## Principles

- **Independence:** every unit has its own repository, build, deploy and rollback. One version of each unit is live per environment; deploying is releasing, and rollback redeploys the previous immutable build.

## Requirements

### Units
- The FES serves its client's static files and proxies its API calls; the browser never calls backends directly. Cross-domain data is combined on the backend.
- An FES may use any server technology.
- All clients use one common brand from the design system.
- All units are served from **one public origin**; the identity provider's sign-in pages are the only other origin the browser visits. Only public configuration reaches the browser.

### Shell App
- **Lightweight:** it owns only the frame, top-level routing, MFE loading, the event service, the theme switcher (light, dark or system), the user's sign-in and sign-out, and its own features.
- It loads MFEs from a **static list** kept in the Shell. Adding or removing an MFE changes the list and redeploys the Shell; deploying a new version of an MFE does not.
- A failing MFE is isolated behind a fallback.

### Micro frontends
- Each MFE can run **standalone** against a Shell stub, with theme switching.
- MFEs do not import each other or leak styles.

### Communication
- MFEs communicate only through the Shell, using **events** (publish/subscribe).
- The Shell gives MFEs the event service plus one read-only value: the active theme (light or dark). Nothing else is shared.

### Users and access
- Users sign in **once** with a shared **identity provider**, which can be replaced without changing how units work.
- The Shell owns the sign-in lifecycle: signing in, keeping the session alive and signing out. MFEs receive nothing for it from the Shell.
- Every unit's FES and backend verify the user on every request and decide access themselves, refusing when they cannot verify. The browser only decides what to show.
- No unit holds session state, and no credential the browser holds is readable by page code.
- A backend calling another unit's backend proves its own identity; the provider decides who may call it and knows its consumers.
- Each unit owns its own settings in the identity provider.
- A standalone MFE can run without the identity provider, in development only.

### Quality
- **Lazy loading** of MFEs and of chunks inside every unit, with size budgets.
- **First load:** a defined LCP target without SSR.
- **Observability:** source maps go only to the monitoring tool; errors are traced from browser to FES to backend.
- **Accessibility and tests:** WCAG 2.2 AA and integration tests.
