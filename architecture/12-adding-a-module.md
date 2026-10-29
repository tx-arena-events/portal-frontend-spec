# 12 – Recipe: adding a module

Follow these steps in order for every new backend module. The example is `service-marketplace`; replace the names. Use `marketplace` (read-heavy, `apps/portal`) and `registration` (workflow, `apps/registration`) as living references.

## 0. Decide the cut

- Which backend module and which `openapi.yaml` does it belong to?
- Which legacy folders does it replace? (`portal-frontend/src/features/<x>`, pages under `src/components/pages/<X>`)
- Which operations will a UI trigger? Each becomes a use case.
- Which decisions does the UI need (allowed, visible, editable)? Each must come from the API as a flag – agree them with the API team now.
- Which app shows it: `apps/portal` or `apps/registration`?
- Which pages belong to it, and which permission keys guard them?
- Does it need a concept from another module? → `@portal/core-kernel`. Does it react to another module? → listener in our state lib, never a core-to-core import.

Write the answers into `packages/service-marketplace-core/README.md` first.

## 1. Add the spec

Put the module's `openapi.yaml` into `specs/service-marketplace/openapi.yaml`: by hand, or with the sync workflow once the backend repository exists ([04](./04-core-packages.md#getting-a-spec-into-the-workspace)). As long as the backend does not exist, write the spec together with the API team.

## 2. Generate the projects

```bash
npx nx g @portal/tools:module service-marketplace
```

The generator creates:

| Project | Path | Import | Tags |
|---------|------|--------|------|
| core package | `packages/service-marketplace-core` | `@portal/service-marketplace-core` | `type:core`, `scope:service-marketplace` |
| state lib | `libs/service-marketplace/state` | `@portal/service-marketplace-state` | `type:state`, `scope:service-marketplace` |
| feature lib | `libs/service-marketplace/feature` | `@portal/service-marketplace-feature` | `type:feature`, `scope:service-marketplace` |

For the core package it also creates the folders (`generated/`, `data/`, `domain/{anemics,data,domains,events,factories,repositories,testing,valueObjects}`, `application/useCases/`, `infrastructure/{constants,data,mappers,repositories}`, `mocks/`), the entry points `index.ts`, `testing.ts`, `mocks.ts`, and the targets `generate-api`, `check-api`, `build`. Finally it prints the line to add the module to `MODULES` in `eslint.config.mjs`.

Until the generator exists, use `@nx/js:library` (core, `--bundler=tsc`) and `@nx/react:library` (state, feature) with `--importPath` and `--tags`.

## 3. Generate the API types

```bash
npx nx run service-marketplace-core:generate-api
```

Commit `src/generated/service-marketplace-api.d.ts`. Never edit it.

## 4. Core: domain

1. One anemic per business object in `domain/anemics/`.
2. Value object types in `domain/valueObjects/`, their factories in `domain/factories/`.
3. Factory input interfaces in `domain/data/`, a factory per anemic in `domain/factories/`.
4. Ports in `domain/repositories/` – only the methods the use cases need.
5. UX pre-checks (only if the user needs feedback before the calls start) in `domain/domains/`. The API enforces the same rules again; no other business rules in the frontend.
6. Inputs, results, filters in `domain/data/`; validation messages as i18n keys.
7. In-memory port implementations in `domain/testing/`.
8. Unit tests next to each file.

Checklist: no imports from any framework; no `fetch`, `window`, `openapi-fetch` or `@portal/core-http`; one type per file outside `data/`; every `if` with braces.

## 5. Core: application

1. One use case per operation a UI may trigger, **reads included**.
2. Constructor injection of ports only.
3. Keep them thin: call the ports, return plain data. Expected outcomes as result unions; progress via an options callback when a workflow has several steps.
4. Tests with the in-memory repositories.

## 6. Core: infrastructure

1. Client and DTO aliases from the generated types in `infrastructure/data/`.
2. A mapper per anemic in `infrastructure/mappers/` (`snake_case` DTO → factory → anemic, and back for requests).
3. Port adapters in `infrastructure/repositories/` with `createCoreClient`, `ensureData`, `ensureOk`.
4. zod schemas where runtime validation is needed.
5. MSW handlers in `mocks/`.
6. Adapter tests against the MSW handlers.

## 7. Core: public API

1. `data/<module>-module.data.ts` with the module interface listing all use cases.
2. `index.ts` with `create<Module>Module(config: CoreModuleConfig)` and the public type exports (anemics, inputs, results, ports, domain services, value object factories).

## 8. Our portal: state

1. `data/<module>-use-cases.data.ts` with the `UseCaseRegistry` augmentation.
2. `<module>.api.ts`: endpoints with `queryFn` → `toQueryResult(() => getUseCases(api).<module>.<useCase>.execute(...))`, with `providesTags` / `invalidatesTags`. Add new tag types to `portalApi`.
3. `hooks.ts` re-exporting the endpoint hooks.
4. Thunks only for workflows with progress; actions they dispatch in `actions/`.
5. Slices for UI state only, initial state in its own file; register them in `apps/portal/src/store.ts`.
6. Selectors that combine cache and UI state.

## 9. Our portal: feature

1. Pages in `pages/`, containers in `containers/`.
2. `<module>.pages.ts` with the `PageDefinition`s and `<module>.components.ts` with the lazy components.
3. Translations in `src/locale/{de,en}/`, including the core package's validation keys.
4. Only `@portal/shared-ui-*` for UI; no `@mui/*`, no `openapi-fetch`, no `create<Module>Module`.

## 10. Wire it in the app that shows it

1. `container.ts`: `<module>: create<Module>Module(moduleConfig)`.
2. `store.ts`: add the module's slices.
3. `page-registry.ts`: add `*_PAGES` and `*_COMPONENTS`.
4. Menu config: add entries.
5. Register listeners if the module reacts to other modules.

## 11. Done when

- [ ] `nx affected -t lint typecheck test build check-api e2e` is green
- [ ] Coverage ≥ 90 % for the core package's `domain` and `application`
- [ ] Every UI operation is a use case; no component or state lib talks HTTP
- [ ] Every page has a permission (or is explicitly open to every logged-in user)
- [ ] The matching legacy `createApi` slices are listed as replaced in the PR description
- [ ] No new code imports `@catena-x/portal-shared-components`
- [ ] The core package README explains its use cases and how to create the module
