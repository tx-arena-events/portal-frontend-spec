# 09 – UI & shared libs

## `shared/ui-mui` – the only place for MUI

- Every MUI component the app uses is wrapped here (button, dialog, text field, data grid, …), with project defaults and theme augments.
- Application code imports `@portal/shared-ui-mui`, never `@mui/*` (enforced by ESLint).
- Build it fresh on MUI 9: theme, theme augments, hooks, wrappers. The legacy `portal-frontend` imports MUI 5 directly in about 160 component files and `@catena-x/portal-shared-components` in about 290; those usages tell you which wrappers are needed first.
- Only wrap what is used. Do not rebuild components that do not exist yet.

Structure per component (one type per file):

```text
button/
  button.tsx
  button.props.ts
  button.defaults.ts
  button.type.ts
  index.ts
```

## Replacing `portal-shared-components`

- The legacy `portal-frontend` builds most of its UI from `@catena-x/portal-shared-components` (MUI 7, jQuery, react-slick). The new portal does not depend on it.
- When a migrated page needs one of its components, build the equivalent in `shared/ui-mui` and use that. Do not import the legacy package in new code.
- Phase 7 removes the package once nothing references it. **Never** add jQuery.

## `shared/ui-layout`

- Header, mobile header, footer, main navigation, user info, breadcrumb, layout shell with `<Outlet />`. Reference: `components/shared/frame` in `portal-frontend` (`Header`, `Footer`, `MobileMenu`, `Logo`, `UserInfo`, `PageBreadcrumb`).
- Receives everything as props: menu items (already filtered), user display name, logo URL, language list.
- The logo URL comes from `GET /config` via the config state, never a hard-coded asset. This covers the customer requirement that the logo is replaceable.

## `shared/i18n`

- Initialise i18next inside a provider, not at module load. The legacy `services/I18nService.ts` in `portal-frontend` initialises globally; do not copy that.
- The legacy namespaces in `src/assets/locales/{de,en}` (`main`, `footer`, `apps`, `idp`, `notification`, `servicerelease`, `registration`) are split across the modules that own them.
- Resources per module: `libs/<module>/feature/src/locale/{de,en}/*.json`, merged by the app.
- Validation and error **keys** from the core packages (e.g. `registration.company.name.required`) are translated here as well; core packages never contain texts.
- Page names in `PageDefinition.name` double as translation keys for titles and menu entries.

## `shared/ui-feedback`

- `ErrorBoundary` (wraps the app; legacy: `components/pages/ErrorBoundary`), `RouteError` (router `errorElement`, also 404), `AccessDenied`, `PageLoader`, empty and error states for lists.
- `CoreHttpErrorData` from `@portal/core-http` (the error RTK Query stores) is mapped to a user message here (`QueryError`).

## Shared value objects live in `@portal/core-kernel`

Value objects used by several modules are framework-free, so they belong to the core packages, not to `libs/shared` ([04](./04-core-packages.md)):

| Type | Purpose |
|------|---------|
| `Bpn`, `Email` | Branded value objects; the type lives in `valueObjects/`, validation and creation in `factories/` (`BpnFactory.create(...)`) |
| `PermissionKey` | Branded string for permission checks, created with `PermissionKeyFactory.create('apps.view')` |
| `Page<T>`, `PageRequest` | Paging (in `data/`) |

## `shared/util`

Pure helpers for our React portal without React (date formatting, string helpers) and the routing data types `PageDefinition` and `MenuItemDefinition` (in `data/`). If it needs React, it is not `util`. If an Angular OSP would need it too, it belongs in `@portal/core-kernel`.

## When to create a new shared lib

Ask in the PR before you do. Typical candidates: a second module needs the same UI pattern (→ `libs/shared/ui-*`), the same value object (→ `@portal/core-kernel`) or the same HTTP behaviour (→ `@portal/core-http`). Propose the name and its tags (`type:*`, `scope:shared`).
