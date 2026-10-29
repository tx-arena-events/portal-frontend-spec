# 02 – Workspace & tooling

## Current state

The workspace `portal-nx` exists:

- Nx 23.3.0, `@nx/js`, `@nx/react`, `@nx/eslint` installed
- TypeScript `~6.0.3`, npm as package manager
- Root package name `@portal/source` → every generated project is imported as `@portal/<name>`
- npm workspaces: `apps/*`, `libs/*/*`
- An empty `packages/` folder (only `.gitkeep`) – this is where the core packages go

Not done yet: `packages/*` in the npm workspaces, the app, lint configuration with boundaries, CI, the module generator.

## Workspace layout

```text
portal-nx/
  apps/
    portal/                 # our React portal (composition root)
    portal-e2e/
    registration/           # separate React app for company registration
    registration-e2e/
  packages/                 # framework-free, consumed by the apps through the workspace (not published)
    core-http/              # @portal/core-http
    core-kernel/            # @portal/core-kernel
    identity-core/          # @portal/identity-core
    registration-core/      # @portal/registration-core
    marketplace-core/       # @portal/marketplace-core
    <module>-core/
  libs/                     # React/Redux, used by the apps
    shared/
      state-core/  infrastructure-config/  ui-mui/  ui-layout/  ui-feedback/  i18n/  util/
    <module>/
      state/
      feature/
  specs/                    # openapi.yaml per backend module, copied from the module's release
    identity/openapi.yaml
    registration/openapi.yaml
    marketplace/openapi.yaml
  tools/
    generators/module/      # @portal/tools:module
```

## Versions

| Package | Version | Why |
|---------|---------|-----|
| `nx`, `@nx/*` | 23.3.0 | Workspace, generators, module boundaries |
| `typescript` | 6.0.3, pinned exactly | `typescript-eslint` 8.72 requires `<6.1.0`; TS 7 breaks type-aware linting |
| `eslint` | 9.39 | Flat config. Move to 10 once `eslint-plugin-react` and `eslint-plugin-jsx-a11y` support it |
| `typescript-eslint` | 8.72 | `strictTypeChecked` + `stylisticTypeChecked` |
| `eslint-plugin-import` | 2.32 | `import/no-restricted-paths` for the layers inside core packages |
| `openapi-typescript` | 7.13 | Types from each module's `openapi.yaml` |
| `openapi-fetch` | 0.17 | Typed, framework-free HTTP client on top of the generated types (core packages only) |
| `zod` | 4.x | Runtime validation |
| `react`, `react-dom` | 19.x (≥ 19.2.7) | Required by React Router 8 (portal only) |
| `react-router` | 8.5 | Data router (`createBrowserRouter`), Node ≥ 22.22 (portal only) |
| `@reduxjs/toolkit`, `react-redux` | 2.13 / 9.x | State and RTK Query cache (portal only) |
| `@mui/material` | 9.x | Only inside `libs/shared/ui-mui` (portal only) |
| `msw` | 3.x | API mocks for tests and local development (the backend does not exist yet) |
| Vite 8, Vitest, Playwright | current | Build, unit tests, E2E |

Pin TypeScript without a range (`"typescript": "6.0.3"`) and add a Renovate rule that blocks major TypeScript updates until typescript-eslint supports them.

## Step by step: finish the workspace

1. **Add the core packages to the npm workspaces** in the root `package.json`:
   ```json
   "workspaces": ["apps/*", "libs/*/*", "packages/*"]
   ```
2. **Generate the app**
   ```bash
   npx nx g @nx/react:app apps/portal --bundler=vite --linter=eslint --unitTestRunner=vitest --e2eTestRunner=playwright --style=css --routing
   ```
   Then replace `react-router-dom` with `react-router@8.5` if the generator added the old package.
3. **Generate the shared core packages**
   ```bash
   npx nx g @nx/js:library packages/core-http --importPath=@portal/core-http --bundler=tsc --unitTestRunner=vitest --tags=type:core-shared,scope:shared
   npx nx g @nx/js:library packages/core-kernel --importPath=@portal/core-kernel --bundler=tsc --unitTestRunner=vitest --tags=type:core-shared,scope:shared
   ```
4. **Add dependencies**
   ```bash
   npm i openapi-fetch zod
   npm i @reduxjs/toolkit react-redux react-router@8.5 @mui/material @emotion/react @emotion/styled i18next react-i18next
   npm i -D typescript-eslint eslint-plugin-react-hooks eslint-plugin-import msw openapi-typescript
   ```
   In each core package's own `package.json`, list only `openapi-fetch`, `zod` and `@portal/core-*` as dependencies. Nothing from React or Redux.
5. **Configure ESLint** (flat config, below).
6. **Prove the rules**: in a throwaway branch import `react` in a core package and `openapi-fetch` in a `feature` lib. `nx lint` must fail for both.
7. **Write the module generator** `tools/generators/module` ([12](./12-adding-a-module.md)).
8. **CI**: `npx nx affected -t lint typecheck test build e2e check-api` on every PR.
9. **lint-staged + Prettier**: `nx affected -t lint test --files=<staged>`.

## Tags

Every project gets exactly one `type:*` and one `scope:*` tag.

| Tag | Used by |
|-----|---------|
| `type:app` | `apps/portal`, `apps/registration` |
| `type:feature` | `libs/<module>/feature` |
| `type:ui` | `libs/shared/ui-*`, `libs/shared/i18n` |
| `type:state` | `libs/<module>/state`, `libs/shared/state-core`, `libs/shared/infrastructure-config` |
| `type:util` | `libs/shared/util` |
| `type:core` | `packages/<module>-core` |
| `type:core-shared` | `packages/core-http`, `packages/core-kernel` |
| `scope:<module>` | everything of one module, e.g. `scope:marketplace` for `marketplace-core`, `marketplace-state`, `marketplace-feature` |
| `scope:shared` | `packages/core-*` without a module, everything under `libs/shared` |
| `scope:app` | the app |

## ESLint flat config (root)

```js
// eslint.config.mjs
import nx from '@nx/eslint-plugin';
import importPlugin from 'eslint-plugin-import';
import reactHooks from 'eslint-plugin-react-hooks';
import tseslint from 'typescript-eslint';

const MODULES = ['identity', 'registration', 'marketplace'];

const FRAMEWORK_IMPORTS = ['react', 'react-*', '@reduxjs/*', '@mui/*', '@angular/*', 'vue', 'rxjs'];

const CORE_LAYER_ZONES = [
  { target: './packages/*-core/src/domain', from: './packages/*-core/src/application' },
  { target: './packages/*-core/src/domain', from: './packages/*-core/src/infrastructure' },
  { target: './packages/*-core/src/application', from: './packages/*-core/src/infrastructure' },
  { target: './packages/*-core/src/infrastructure', from: './packages/*-core/src/application' },
];

export default [
  ...nx.configs['flat/base'],
  ...nx.configs['flat/typescript'],
  ...nx.configs['flat/react'],
  ...tseslint.configs.strictTypeChecked,
  ...tseslint.configs.stylisticTypeChecked,
  {
    files: ['**/*.ts', '**/*.tsx'],
    languageOptions: {
      parserOptions: { projectService: true, tsconfigRootDir: import.meta.dirname },
    },
    plugins: { 'react-hooks': reactHooks, import: importPlugin },
    rules: {
      curly: ['error', 'all'],
      'max-classes-per-file': ['error', 1],
      'react-hooks/rules-of-hooks': 'error',
      'react-hooks/exhaustive-deps': 'error',
      '@nx/enforce-module-boundaries': [
        'error',
        {
          enforceBuildableLibDependency: true,
          allow: [],
          depConstraints: [
            { sourceTag: 'type:app', onlyDependOnLibsWithTags: ['*'] },
            { sourceTag: 'type:feature', onlyDependOnLibsWithTags: ['type:feature', 'type:state', 'type:ui', 'type:util', 'type:core', 'type:core-shared'] },
            { sourceTag: 'type:ui', onlyDependOnLibsWithTags: ['type:ui', 'type:util', 'type:core-shared'] },
            { sourceTag: 'type:state', onlyDependOnLibsWithTags: ['type:state', 'type:util', 'type:core', 'type:core-shared'] },
            { sourceTag: 'type:util', onlyDependOnLibsWithTags: ['type:util', 'type:core-shared'] },
            { sourceTag: 'type:core', onlyDependOnLibsWithTags: ['type:core-shared'] },
            { sourceTag: 'type:core-shared', onlyDependOnLibsWithTags: ['type:core-shared'] },
            { sourceTag: 'scope:shared', onlyDependOnLibsWithTags: ['scope:shared'] },
            ...MODULES.map((module) => ({
              sourceTag: `scope:${module}`,
              onlyDependOnLibsWithTags: [`scope:${module}`, 'scope:shared'],
            })),
          ],
        },
      ],
    },
  },
  {
    // core packages: no framework, layers inside the package
    files: ['packages/**/*.ts'],
    rules: {
      'no-restricted-imports': ['error', { patterns: [{ group: FRAMEWORK_IMPORTS, message: 'Core packages must stay framework-free.' }] }],
      'import/no-restricted-paths': ['error', { zones: CORE_LAYER_ZONES }],
    },
  },
  {
    // core domain and application: no browser globals, no HTTP client
    files: ['packages/*-core/src/domain/**/*.ts', 'packages/*-core/src/application/**/*.ts', 'packages/core-kernel/**/*.ts'],
    rules: {
      'no-restricted-imports': ['error', { patterns: [
        { group: FRAMEWORK_IMPORTS, message: 'Core packages must stay framework-free.' },
        { group: ['openapi-fetch', '@portal/core-http'], message: 'Only infrastructure talks HTTP.' },
      ] }],
      'no-restricted-globals': ['error', 'fetch', 'window', 'document', 'localStorage'],
    },
  },
  {
    // React side: never the HTTP client, MUI only inside the wrapper lib
    files: ['libs/**/*.ts', 'libs/**/*.tsx', 'apps/**/*.ts', 'apps/**/*.tsx'],
    ignores: ['libs/shared/ui-mui/**'],
    rules: {
      'no-restricted-imports': ['error', { patterns: [
        { group: ['@mui/*'], message: 'Import MUI wrappers from @portal/shared-ui-mui.' },
        { group: ['openapi-fetch'], message: 'Call core use cases instead.' },
      ] }],
    },
  },
];
```

Add every new module to `MODULES`; the generator prints a reminder.

`feature` may import core packages, but by convention only **types and value object factories** (e.g. `AppIdFactory`), never the `create<Module>Module` factories. Only the apps create modules.

## Nx targets

| Target | Where | What |
|--------|-------|------|
| `lint`, `typecheck`, `test` | every project | inferred by the Nx plugins |
| `generate-api` | every `<module>-core` | `openapi-typescript ../../specs/<module>/openapi.yaml -o src/generated/<module>-api.d.ts` |
| `check-api` | every `<module>-core` | regenerate and fail on a diff (spec changed, types not regenerated) |
| `build` | packages, apps | `@nx/js:tsc` for packages, Vite for the apps |
| `e2e` | `apps/portal-e2e`, `apps/registration-e2e` | Playwright against `nx serve` with MSW |
