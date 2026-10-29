# 01 – Principles

Every design decision in this frontend follows the guardrails below. If a change conflicts with one of them, raise it before you build it.

## 1. The frontend is dumb

- The frontend holds **no tokens**, knows **no Keycloak realm, client ID or IdP URL**, and resolves **no roles**.
- Login, logout, token refresh and the role → permission mapping happen in the API (BFF pattern, see [05](./05-auth-and-session.md)).
- The frontend receives a list of **permission keys** from `GET /me` and only uses them to **hide** UI. The API **enforces** them.
- Branding (logo, feature flags) comes from the API (`GET /config`), not from build-time constants.
- Business decisions and everything OSP-specific live in the backend. If the UI needs to know whether an action is allowed, the API returns a flag (e.g. `can_subscribe`); the frontend does not compute it.

## 2. What is left in the frontend is framework-free

- The little logic the frontend still has – which calls belong to an action, in which order, how data is mapped, how progress is reported – lives in **core packages** (`packages/<module>-core`), one per backend module, without any UI or state framework ([04](./04-core-packages.md)).
- Their HTTP types are **generated** from the module's `openapi.yaml`; use cases and mappers are written by hand.
- Every operation a UI may trigger is a **use case**, reads included. A UI never talks to the API directly.
- Core packages are **workspace packages**: our apps consume them through Nx. Nothing is published to a package registry.
- Because the core packages are framework-free, the UI on top stays replaceable. Micro-frontends are a topic for after the hackathon.

## 3. DDD + onion, frontend only

- Each business area is a **module** (bounded context): one core package plus, in our portal, a `state` and a `feature` lib.
- Dependencies point **inward only**: `app → feature → state → core (application → domain)`. Inside a core package `infrastructure` implements the ports defined in `domain`; only the package's `index.ts` wires them.
- Entities are **anemic**: plain immutable types. What logic there is lives in **classic use cases** (`application`); `domain/domains` only holds UX pre-checks that the API enforces again.
- Domain folders are organised **by type**, not by entity: `anemics/`, `data/`, `domains/`, `events/`, `factories/`, `repositories/`, `valueObjects/`.

## 4. Redux Toolkit is the only state mechanism in our portal

- No React context for business state, no other state libraries.
- RTK Query is the cache: its endpoints use `queryFn` to call core use cases. There is no `fetchBaseQuery` and no handwritten URL in the React code.
- RTK slices hold **client state only** (filters, wizard progress, dialogs). Server data is never copied into slices.
- Core packages never depend on Redux, so the UI on top stays replaceable.

## 5. Everything that can be a package or lib is one

- `apps/*` contain only the composition root: `main.tsx`, router, store setup, use case container, providers.
- Framework-free code shared by several modules goes to `packages/core-http` or `packages/core-kernel`; React-only shared code goes to `libs/shared/*`. If you write something two modules could use, propose where it should live in the PR.

## 6. Rules are enforced by tooling, not by review

- Package, lib and module boundaries: `@nx/enforce-module-boundaries` with tags.
- Layers inside a core package: `import/no-restricted-paths`.
- Forbidden imports (any framework in `packages/`, MUI outside `ui-mui`, `openapi-fetch` outside core infrastructure): ESLint.
- Code style: ESLint + Prettier, run in lint-staged and CI.

## 7. Code rules

| Rule | Detail |
|------|--------|
| One class / one type per file | Exception: `data/` folders may hold several related interfaces in one file. A value object's type lives in `valueObjects/`, its creation/validation class in `factories/` |
| If statements always with braces | Body on its own line, never single-line ifs (`curly: ['error', 'all']`) |
| `.tsx` only where React lives | `libs/*/feature`, `libs/shared/ui-*`, app. Never in `packages/` |
| Barrel files | Each package and lib exposes its public API via `src/index.ts`. Never deep-import into another package or lib |
| Naming | Files `kebab-case` with a type suffix: `app.anemic.ts`, `bpn.value-object.ts`, `bpn.factory.ts`, `list-active-apps.use-case.ts`, `app-http.repository.ts`, `app.mapper.ts` |
| API JSON | The API uses `snake_case` fields. Generated DTO types mirror that; mappers in the core package convert to `camelCase` anemics |
| Generated code | `src/generated/` is never edited by hand |
| Runtime validation | `zod` (never class-validator) for `config.json` and, where needed, API responses |
| Legacy repos | Read-only reference. Nothing gets deleted there |

## 8. What we deliberately do not do

- No CQRS, no event sourcing in the frontend.
- No micro-frontends during the hackathon.
- No WebSockets. If live updates are needed, the API should offer SSE.
- No business logic in components and none in the core packages. A component that contains an `if` on business data is a smell: ask the API team for a flag; only a UX pre-check that the API enforces again may live in a core package's `domain/domains`.
