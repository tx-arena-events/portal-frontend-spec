# 08 – UI & shared libs

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
- The legacy namespaces in `src/assets/locales/{de,en}` (`main`, `footer`, `apps`, `idp`, `notification`, `servicerelease`, `registration`) are split across the contexts that own them.
- Resources per context: `libs/<context>/feature/src/locale/{de,en}/*.json`, merged by the app.
- Page names in `PageDefinition.name` double as translation keys for titles and menu entries.

## `shared/ui-feedback`

- `ErrorBoundary` (wraps the app; legacy: `components/pages/ErrorBoundary`), `RouteError` (router `errorElement`, also 404), `AccessDenied`, `PageLoader`, empty and error states for lists.
- `ApiError` from `shared/infrastructure-http` is mapped to a user message here.

## `shared/domain-kernel`

Value objects and data types used by several contexts:

| Type | Purpose |
|------|---------|
| `Id`, `Bpn`, `Email` | Branded value objects; the type lives in `valueObjects/`, validation and creation in `factories/` (`BpnFactory.create(...)`) |
| `PermissionKey` | Branded string for permission checks, created with `PermissionKeyFactory.create('apps.view')` |
| `Page<T>`, `PageRequest` | Paging (in `data/`) |
| `PageDefinition`, `MenuItemDefinition` | Routing and menus (in `data/`) |

## `shared/util`

Pure helpers without dependencies (date formatting, string helpers). If it needs React, it is not `util`.

## When to create a new shared lib

Ask in the PR before you do. Typical candidates: a second context needs the same UI pattern, the same value object, or the same HTTP behaviour. Propose the lib name and its tags (`type:*`, `scope:shared`).
