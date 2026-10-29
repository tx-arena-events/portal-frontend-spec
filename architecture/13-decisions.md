# 13 – Decisions

Short decision log. Add new entries at the top; never rewrite old ones, supersede them.

| # | Date | Decision | Why | Alternatives considered |
|---|------|----------|-----|-------------------------|
| 21 | 2026-10 | Business decisions come from the API as flags (e.g. `can_subscribe`); core packages contain no business rules, only UX pre-checks the API enforces again | The frontend stays as dumb as possible; OSP-specific behaviour lives in the backend | Computing permissions/eligibility in domain services |
| 20 | 2026-10 | Registration is a separate Nx app `apps/registration` on the same libs and core packages | Matches the legacy split; scope of registration is still TBD and can evolve independently | Registration pages inside `apps/portal` |
| 19 | 2026-10 | Specs are pulled into `specs/<module>/` by hand or with the `sync-openapi` GitHub Action | The backend does not exist yet; later the backend pipeline can trigger the sync | Generating from a live server URL |
| 18 | 2026-10 | No package registry: core packages are workspace packages | OSP-specific behaviour lives in the backend; nothing outside this workspace consumes the packages | Publishing to an npm registry with `nx release` |
| 17 | 2026-10 | Our React portal reads and writes through RTK Query endpoints with `queryFn` that call core use cases; no `fetchBaseQuery` (supersedes #10) | Transport and mapping exist once, in the core package; RTK Query still provides cache, loading state and tag invalidation | `fetchBaseQuery` in the React code next to the core packages (duplicates URLs and mappers) |
| 16 | 2026-10 | Every operation a UI may trigger is a use case, reads included | No UI ever needs to know the HTTP client; one public API per module | Reads directly via the generated client |
| 15 | 2026-10 | `openapi-fetch` as HTTP client in the core packages | Typed directly from the generated `paths`, framework-free, small | Handwritten fetch wrappers; generated SDKs with their own runtime |
| 14 | 2026-10 | Business behaviour in framework-free core packages, one per backend module, types generated from the module's `openapi.yaml` | The remaining frontend logic exists once and the UI on top stays replaceable | React-only libs; generating complete use cases from the spec (the spec knows endpoints, not workflows); micro-frontends (after the hackathon) |
| 13 | 2026-10 | Specs vendored in `specs/<module>/openapi.yaml`, `check-api` in CI | Reproducible builds; API changes show up in the PR diff | Generating from a live server URL |
| 12 | 2026-10 | ESLint 9.39, not 10 | `eslint-plugin-react` and `eslint-plugin-jsx-a11y` do not support ESLint 10 yet | ESLint 10 without React plugins |
| 11 | 2026-10 | Runtime config via `/config.json` from a ConfigMap | One image for all environments | `import.meta.env` (one image per env), `window.__ENV__` |
| 10 | 2026-10 | ~~RTK Query with `fetchBaseQuery` in `infrastructure`; `state` re-exports hooks~~ (superseded by #17) | Keeps HTTP at the edge while keeping RTK Query's cache; one `createApi` for the app | `queryFn` calling use cases for every endpoint |
| 9 | 2026-10 | One `feature` lib per module | Clear ownership; lazy loading per page still possible | One lib per page |
| 8 | 2026-10 | React Router 8.5 | Current version; requires React ≥ 19.2.7, which the workspace has | React Router 7.x |
| 7 | 2026-10 | DTOs generated from OpenAPI (`openapi-typescript`), handwritten only as fallback | Contract changes break the build instead of production | Handwritten DTOs only |
| 6 | 2026-10 | Separate API domain with CORS and credentials, same registrable domain | Matches the planned deployment; keeps `SameSite=Lax` | Same-origin reverse proxy |
| 5 | 2026-10 | New workspace `portal-nx`, scope `@portal` | Clean start next to the legacy repos; `portal-frontend` stays the functional reference | Refactoring `portal-frontend` in place |
| 4 | 2026-10 | npm as package manager | The project will be continued by others after the hackathon; npm is universally known | Bun, pnpm |
| 3 | 2026-10 | TypeScript 6.0.3 instead of 7, ESLint instead of oxlint | Type-aware linting needs `typescript <6.1`; module boundaries need ESLint | TS 7 + oxlint |
| 2 | 2026-10 | No Keycloak in the frontend (BFF) | Tokens never reach the browser; security logic in one place | `keycloak-js` as before |
| 1 | 2026-10 | DDD + onion in the frontend, anemic entities, classic use cases, no CQRS | Strict, testable layering; matches the team's existing guides | Feature folders without layers |

## Open points

- Exact permission key list and role mapping (phase 0).
- The decision flags each module's API returns (e.g. `can_subscribe`), agreed per module with the API team.
- Backend repository and path of each module's `openapi.yaml` once the backend exists (input for the sync workflow).
- Scope of the registration app.
