# 01 – Principles

Every design decision in this frontend follows the guardrails below. If a change conflicts with one of them, raise it before you build it.

## 1. The frontend is dumb

- The frontend holds **no tokens**, knows **no Keycloak realm, client ID or IdP URL**, and resolves **no roles**.
- Login, logout, token refresh and the role → permission mapping happen in the API (BFF pattern, see [04](./04-auth-and-session.md)).
- The frontend receives a list of **permission keys** from `GET /me` and only uses them to **hide** UI. The API **enforces** them.
- Branding (logo, feature flags) comes from the API (`GET /config`), not from build-time constants.

## 2. DDD + onion, frontend only

- Each business area is a **bounded context** with its own libs.
- Dependencies point **inward only**: `app → feature → state → application → domain`.
- `infrastructure` implements the ports defined in `domain`. Only the app (wiring) and `state` (hook re-exports) import it.
- Entities are **anemic**: plain immutable types. Logic lives in **classic use cases** (`application`) and pure domain services (`domain/domains`).
- Domain libs are organised **by type**, not by entity: `anemics/`, `data/`, `domains/`, `events/`, `factories/`, `repositories/`, `valueObjects/`.

## 3. Redux Toolkit is the only state mechanism

- No React context for business state, no other state libraries.
- RTK Query is the HTTP client and cache. It lives in `infrastructure` with `fetchBaseQuery`.
- RTK slices hold **client state only** (session, filters, wizard steps, dialogs). Server data is never copied into slices.

## 4. Everything that can be a lib is a lib

- `apps/*` contain only the composition root: `main.tsx`, router, store setup, DI container, providers.
- Generic code goes to `libs/shared/*`. If you write something that two contexts could use, move it to a shared lib (and say so in the PR).

## 5. Rules are enforced by tooling, not by review

- Layer and context boundaries: `@nx/enforce-module-boundaries` with tags.
- Forbidden imports (MUI outside `ui-mui`, React in `domain`, `fetch` outside `infrastructure`): ESLint.
- Code style: ESLint + Prettier, run in lint-staged and CI.

## 6. Code rules

| Rule | Detail |
|------|--------|
| One class / one type per file | Exception: `data/` folders may hold several related interfaces in one file. A value object's type lives in `valueObjects/`, its creation/validation class in `factories/` |
| If statements always with braces | Body on its own line, never single-line ifs (`curly: ['error', 'all']`) |
| `.tsx` only where React lives | `feature`, `ui`, app. Never in `domain`, `application`, `infrastructure` (except tests) |
| Barrel files | Each lib exposes its public API via `src/index.ts`. Never deep-import into another lib |
| Naming | Files `kebab-case` with a type suffix: `app.anemic.ts`, `bpn.value-object.ts`, `bpn.factory.ts`, `list-active-apps.use-case.ts`, `app-rtk.repository.ts`, `app.mapper.ts` |
| API JSON | The API uses `snake_case` fields. DTO types mirror that; mappers convert to `camelCase` anemics |
| Runtime validation | `zod` (never class-validator) for `config.json` and, where needed, API responses |
| Legacy repos | Read-only reference. Nothing gets deleted there |

## 7. What we deliberately do not do

- No CQRS, no event sourcing in the frontend.
- No WebSockets. If live updates are needed, the API should offer SSE.
- No business logic in components. A component that contains an `if` on business data is a smell: move the decision to a use case, domain service or selector.
