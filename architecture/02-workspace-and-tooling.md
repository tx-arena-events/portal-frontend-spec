# 02 – Workspace & tooling

## Current state

The workspace `portal-nx` exists:

- Nx 23.3.0, `@nx/js`, `@nx/react`, `@nx/eslint` installed
- TypeScript `~6.0.3`, npm as package manager
- Root package name `@portal/source` → every generated lib is imported as `@portal/<name>`
- npm workspaces: `apps/*`, `libs/*/*`

Not done yet: the app, lint configuration with boundaries, CI, the context generator.

## Versions

| Package | Version | Why |
|---------|---------|-----|
| `nx`, `@nx/*` | 23.3.0 | Workspace, generators, module boundaries |
| `typescript` | 6.0.3, pinned exactly | `typescript-eslint` 8.72 requires `<6.1.0`; TS 7 breaks type-aware linting |
| `eslint` | 9.39 | Flat config. Move to 10 once `eslint-plugin-react` and `eslint-plugin-jsx-a11y` support it |
| `typescript-eslint` | 8.72 | `strictTypeChecked` + `stylisticTypeChecked` |
| `react`, `react-dom` | 19.x (≥ 19.2.7) | Required by React Router 8 |
| `react-router` | 8.5 | Data router (`createBrowserRouter`), Node ≥ 22.22 |
| `@reduxjs/toolkit`, `react-redux` | 2.13 / 9.x | State and RTK Query |
| `@mui/material` | 9.x | Only imported inside `libs/shared/ui-mui` |
| `zod` | 4.x | Runtime validation |
| `openapi-typescript` | 7.13 | DTO types from the API's OpenAPI spec |
| `msw` | 3.x | API mocks for development and tests |
| Vite 8, Vitest, Playwright | current | Build, unit tests, E2E |

Pin TypeScript without a range (`"typescript": "6.0.3"`) and add a Renovate rule that blocks major TypeScript updates until typescript-eslint supports them.

## Step by step: finish the workspace

1. **Generate the app**
   ```bash
   npx nx g @nx/react:app apps/portal --bundler=vite --linter=eslint --unitTestRunner=vitest --e2eTestRunner=playwright --style=css --routing
   ```
   Then replace `react-router-dom` with `react-router@8.5` if the generator added the old package.
2. **Remove the empty `packages/` folder** (only after the team agrees; it contains just `.gitkeep`).
3. **Add dependencies**
   ```bash
   npm i @reduxjs/toolkit react-redux react-router@8.5 @mui/material @emotion/react @emotion/styled zod i18next react-i18next
   npm i -D typescript-eslint eslint-plugin-react-hooks msw openapi-typescript
   ```
4. **Configure ESLint** (flat config, see below), including module boundaries.
5. **Add a broken import on purpose** (e.g. `domain` importing `react`) and check that `nx lint` fails. Remove it afterwards.
6. **Write the context generator** `tools/generators/context` (see [11](./11-adding-a-context.md)).
7. **CI**: `npx nx affected -t lint typecheck test build e2e` on every PR.
8. **lint-staged + Prettier**: `nx affected -t lint test --files=<staged>`.

## Tags

Every project gets exactly one `type:*` tag and one `scope:*` tag in its `project.json` (or `package.json` → `nx.tags`).

| Tag | Used by |
|-----|---------|
| `type:app` | `apps/portal`, `apps/registration` |
| `type:feature` | `libs/<context>/feature` |
| `type:ui` | `libs/shared/ui-*`, `libs/shared/i18n` |
| `type:state` | `libs/<context>/state`, `libs/shared/state-core` |
| `type:application` | `libs/<context>/application` |
| `type:infrastructure` | `libs/<context>/infrastructure`, `libs/shared/infrastructure-*` |
| `type:domain` | `libs/<context>/domain`, `libs/shared/domain-kernel` |
| `type:util` | `libs/shared/util` |
| `scope:<context>` | all libs of one context, e.g. `scope:marketplace` |
| `scope:shared` | everything under `libs/shared` |
| `scope:app` | the apps |

## ESLint flat config (root)

```js
// eslint.config.mjs
import nx from '@nx/eslint-plugin';
import tseslint from 'typescript-eslint';
import reactHooks from 'eslint-plugin-react-hooks';

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
    plugins: { 'react-hooks': reactHooks },
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
            { sourceTag: 'type:feature', onlyDependOnLibsWithTags: ['type:feature', 'type:state', 'type:ui', 'type:domain', 'type:util'] },
            { sourceTag: 'type:ui', onlyDependOnLibsWithTags: ['type:ui', 'type:util'] },
            { sourceTag: 'type:state', onlyDependOnLibsWithTags: ['type:state', 'type:application', 'type:domain', 'type:infrastructure', 'type:util'] },
            { sourceTag: 'type:application', onlyDependOnLibsWithTags: ['type:application', 'type:domain', 'type:util'] },
            { sourceTag: 'type:infrastructure', onlyDependOnLibsWithTags: ['type:infrastructure', 'type:domain', 'type:util'] },
            { sourceTag: 'type:domain', onlyDependOnLibsWithTags: ['type:domain', 'type:util'] },
            { sourceTag: 'type:util', onlyDependOnLibsWithTags: ['type:util'] },
            { sourceTag: 'scope:shared', onlyDependOnLibsWithTags: ['scope:shared'] },
            { sourceTag: 'scope:marketplace', onlyDependOnLibsWithTags: ['scope:marketplace', 'scope:shared'] },
            // one entry per context
          ],
        },
      ],
    },
  },
  {
    // MUI only inside the wrapper lib
    files: ['**/*.ts', '**/*.tsx'],
    ignores: ['libs/shared/ui-mui/**'],
    rules: {
      'no-restricted-imports': ['error', { patterns: [{ group: ['@mui/*'], message: 'Import MUI wrappers from @portal/shared-ui-mui.' }] }],
    },
  },
  {
    // domain and application are framework-free
    files: ['libs/*/domain/**/*.ts', 'libs/*/application/**/*.ts', 'libs/shared/domain-kernel/**/*.ts'],
    rules: {
      'no-restricted-imports': ['error', { patterns: [
        { group: ['react', 'react-*', '@reduxjs/*', '@mui/*'], message: 'domain/application must stay framework-free.' },
      ] }],
      'no-restricted-globals': ['error', 'fetch', 'window', 'document', 'localStorage'],
    },
  },
  {
    // fetch only in infrastructure
    files: ['libs/*/state/**', 'libs/*/feature/**', 'libs/shared/ui-*/**'],
    rules: { 'no-restricted-globals': ['error', 'fetch'] },
  },
];
```

The `state → infrastructure` edge is allowed by tag. Keep it narrow by convention: `state` only re-exports hooks and endpoint objects; it never imports adapters or mappers. Reviewers check that.

## Nx targets per lib

| Target | Where | What |
|--------|-------|------|
| `lint`, `typecheck`, `test` | every lib | inferred by the Nx plugins |
| `generate-api-types` | every `infrastructure` lib | `openapi-typescript <spec-url> -o src/data/generated/api.d.ts` |
| `build` | apps | Vite production build |
| `e2e` | `apps/portal-e2e` | Playwright against `nx serve` with MSW |
