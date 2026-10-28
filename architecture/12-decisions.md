# 12 – Decisions

Short decision log. Add new entries at the top; never rewrite old ones, supersede them.

| # | Date | Decision | Why | Alternatives considered |
|---|------|----------|-----|-------------------------|
| 12 | 2026-10 | ESLint 9.39, not 10 | `eslint-plugin-react` and `eslint-plugin-jsx-a11y` do not support ESLint 10 yet | ESLint 10 without React plugins |
| 11 | 2026-10 | Runtime config via `/config.json` from a ConfigMap | One image for all environments | `import.meta.env` (one image per env), `window.__ENV__` |
| 10 | 2026-10 | RTK Query with `fetchBaseQuery` in `infrastructure`; `state` re-exports hooks | Keeps HTTP at the edge while keeping RTK Query's cache; one `createApi` for the app | `queryFn` calling use cases for every endpoint |
| 9 | 2026-10 | One `feature` lib per context | Clear ownership; lazy loading per page still possible | One lib per page |
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
- Spec URL for `generate-api-types` per environment.
- Whether `apps/registration` shares the domain/host with `apps/portal`.
