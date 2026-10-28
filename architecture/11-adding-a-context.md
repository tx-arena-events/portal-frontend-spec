# 11 – Recipe: adding a bounded context

Follow these steps in order for every new context. The example is `service-marketplace`; replace names accordingly. Use the marketplace pilot as the living reference.

## 0. Decide the cut

- Which legacy folders does it replace? (`portal-frontend/src/features/<x>`, pages under `src/components/pages/<X>`)
- Which pages belong to it, and which permission keys guard them?
- Does it need anything from another context? If yes: is it a shared concept (→ `shared/domain-kernel`) or an event (→ listener)?

Write the answers into `libs/<context>/README.md` first.

## 1. Generate the libs

```bash
npx nx g @portal/tools:context service-marketplace
```

The generator creates:

| Lib | Import path | Tags |
|-----|-------------|------|
| `libs/service-marketplace/domain` | `@portal/service-marketplace-domain` | `type:domain`, `scope:service-marketplace` |
| `libs/service-marketplace/application` | `@portal/service-marketplace-application` | `type:application`, `scope:service-marketplace` |
| `libs/service-marketplace/infrastructure` | `@portal/service-marketplace-infrastructure` | `type:infrastructure`, `scope:service-marketplace` |
| `libs/service-marketplace/state` | `@portal/service-marketplace-state` | `type:state`, `scope:service-marketplace` |
| `libs/service-marketplace/feature` | `@portal/service-marketplace-feature` | `type:feature`, `scope:service-marketplace` |

plus the domain folders (`anemics/ data/ domains/ events/ factories/ repositories/ valueObjects/`), the infrastructure folders (`data/generated/ schemas/ mappers/ repositories/ mocks/`), barrel files, and a `generate-api-types` target. Add the new scope to the `depConstraints` in `eslint.config.mjs` (the generator prints the line).

Until the generator exists, run `@nx/js:library` / `@nx/react:library` five times with `--directory`, `--tags` and `--importPath`.

## 2. Domain

1. One anemic per entity in `anemics/` (`service.anemic.ts`).
2. Value objects for IDs and validated values in `valueObjects/`.
3. A factory per anemic in `factories/`.
4. Ports in `repositories/` – only the methods the use cases need.
5. Business rules as domain services in `domains/`.
6. Inputs, filters, paging shapes in `data/`.
7. Unit tests next to each file.

Checklist: no imports from React, Redux, MUI; no `fetch`; one type per file outside `data/`; every `if` with braces.

## 3. Application

1. One use case per operation that has logic (validation, several calls, rules).
2. Constructor injection of ports only.
3. Tests with an in-memory repository from `domain/src/testing`.

Skip use cases for plain reads; the feature uses the endpoint hook.

## 4. Infrastructure

1. Run `nx run service-marketplace-infrastructure:generate-api-types` (or write DTOs by hand in `data/*.data.ts`, `snake_case`).
2. Mapper per anemic in `mappers/` (DTO → factory → anemic).
3. zod schema in `schemas/` where runtime validation is needed.
4. Endpoints with `portalApi.injectEndpoints` in `<context>.api.ts`, with `providesTags` / `invalidatesTags`. Add new tag types to `portalApi`.
5. Port adapters in `repositories/` on top of the endpoints (`initiate(..., { subscribe: false })`).
6. MSW handlers in `mocks/`.

## 5. State

1. `data/<context>-use-cases.data.ts` with the `UseCaseRegistry` augmentation.
2. Thunks with `createAppAsyncThunk` that call `extra.useCases.<context>.*`.
3. Slices for client state only; register them in `apps/portal/src/store.ts`.
4. Selectors that combine cache and client state.
5. `hooks.ts` that re-exports the endpoint hooks.

## 6. Feature

1. Pages in `pages/`, containers in `containers/`.
2. `<context>.pages.ts` with the `PageDefinition`s and `<context>.components.ts` with the lazy components (one export per file).
3. Translations in `src/locale/{de,en}/`.
4. Only `@portal/shared-ui-*` for UI; no `@mui/*`.

## 7. Wire it in the app

1. `container.ts`: create adapters and use cases for the context.
2. `store.ts`: add the context's slices.
3. `page-registry.ts`: add `*_PAGES` and `*_COMPONENTS`.
4. Menu config: add entries.
5. Register listeners if the context reacts to other contexts' events.

## 8. Done when

- [ ] `nx affected -t lint typecheck test build e2e` is green
- [ ] Coverage ≥ 90 % for `domain` and `application`
- [ ] Every page has a permission (or is explicitly public for logged-in users)
- [ ] The matching legacy `createApi` slices are listed as replaced in the PR description
- [ ] No new code imports `@catena-x/portal-shared-components`
- [ ] The context README is up to date
