# 08 – Routing & permissions

## Idea

The legacy `portal-frontend` describes every page, action and overlay as data in `types/Config.tsx` (`ALL_PAGES`, `ALL_ACTIONS`, `ALL_OVERLAYS`). Each entry has an `allowTo` function that checks Keycloak client roles through `services/AccessService.tsx`. The new portal keeps the idea of pages as data, but access is based on **permission keys from `/me`** instead of Keycloak roles.

- Every page is described once as data: name, path, optional param, required permission.
- Each module's `feature` lib exports its pages lazily.
- The app turns the page list into React Router 8 routes and wraps each in `RequireAccess`.
- Menus are built from the same page list, filtered by the same permissions.

## Page definition

```ts
// libs/shared/util/src/data/page.data.ts
import type { PermissionKey } from '@portal/core-kernel';

export interface PageDefinition {
  name: string;                 // unique, also the i18n key
  path?: string;                // defaults to name
  param?: string;               // optional ':param?'
  permission?: PermissionKey;   // undefined = any logged-in user
  props?: Record<string, unknown>;
}

export interface MenuItemDefinition {
  page: string;                 // references PageDefinition.name
  children?: ReadonlyArray<MenuItemDefinition>;
}
```

Each module exports its definitions and its components, one export per file:

```ts
// libs/marketplace/feature/src/marketplace.pages.ts
import { PermissionKeyFactory } from '@portal/core-kernel';
import type { PageDefinition } from '@portal/shared-util';

export const MARKETPLACE_PAGES: ReadonlyArray<PageDefinition> = [
  { name: 'marketplace', permission: PermissionKeyFactory.create('apps.view') },
  { name: 'appdetail', param: 'appId', permission: PermissionKeyFactory.create('apps.view') },
];
```

```ts
// libs/marketplace/feature/src/marketplace.components.ts
import { lazy } from 'react';

export const MARKETPLACE_COMPONENTS = {
  marketplace: lazy(() => import('./pages/marketplace.page')),
  appdetail: lazy(() => import('./pages/app-detail.page')),
};
```

## Building routes in the app

```tsx
// apps/portal/src/router.tsx
import { createBrowserRouter } from 'react-router';
import { Layout } from '@portal/shared-ui-layout';
import { RouteError } from '@portal/shared-ui-feedback';
import { SessionGate } from '@portal/identity-feature';
import { pageRegistry, toRoutes } from './page-registry';

export const router = createBrowserRouter([
  {
    path: '/',
    element: (
      <SessionGate>
        <Layout />
      </SessionGate>
    ),
    errorElement: <RouteError />,
    children: [
      {
        errorElement: <RouteError />,
        children: [...toRoutes(pageRegistry), { path: '*', element: <RouteError notFound /> }],
      },
    ],
  },
]);
```

`page-registry.ts` merges all modules' `*_PAGES` and `*_COMPONENTS`. `toRoutes` creates one route per page:

```tsx
// apps/portal/src/to-routes.tsx
import { Suspense } from 'react';
import type { RouteObject } from 'react-router';
import { RequireAccess } from '@portal/identity-feature';
import { PageLoader } from '@portal/shared-ui-feedback';
import type { RegisteredPage } from './data/registered-page.data';

export function toRoutes(pages: ReadonlyArray<RegisteredPage>): Array<RouteObject> {
  return pages.map(({ definition, Component }) => {
    const path = definition.path ?? definition.name;
    return {
      path: definition.param !== undefined ? `${path}/:${definition.param}?` : path,
      element: (
        <RequireAccess permission={definition.permission}>
          <Suspense fallback={<PageLoader />}>
            <Component {...definition.props} />
          </Suspense>
        </RequireAccess>
      ),
    };
  });
}
```

## `RequireAccess`

```ts
// libs/identity/feature/src/require-access.props.ts
import type { PropsWithChildren } from 'react';
import type { PermissionKey } from '@portal/core-kernel';

export interface RequireAccessProps extends PropsWithChildren {
  permission?: PermissionKey;
}
```

```tsx
// libs/identity/feature/src/require-access.tsx
import { useHasPermission } from '@portal/identity-state';
import { AccessDenied } from '@portal/shared-ui-feedback';
import type { RequireAccessProps } from './require-access.props';

export function RequireAccess({ permission, children }: RequireAccessProps) {
  const allowed = useHasPermission(permission);
  if (!allowed) {
    return <AccessDenied />;
  }
  return children;
}
```

`useHasPermission` is a hook in `libs/identity/state` on top of `HasPermissionDomain` from `@portal/identity-core` ([05](./05-auth-and-session.md)). Hiding is UX only: the API returns 403 for anything the user may not do.

## Menus

- `MAIN_MENU`, `USER_MENU`, `FOOTER_MENU` are `MenuItemDefinition` trees in the app config.
- A hook `useVisibleMenu(menu, pages)` in `libs/identity/state` removes items whose page requires a permission the user lacks (`HasPermissionDomain.check`), and drops empty parents.
- `shared/ui-layout` renders the result. It receives plain data; it does not know permissions.

## Migrating from the legacy role checks

| Legacy (`portal-frontend`, `types/Config.tsx`) | New |
|-----------------------------------------------|-----|
| `allowTo: () => userHasPortalRole(ROLES.APPSTORE_VIEW)` | `permission: PermissionKeyFactory.create('apps.view')` |
| `allowTo: () => userHasPortalRole(ROLES.APPSTORE_VIEW_SERVICES)` | `services.view` |
| `allowTo: () => userHasSemanticHubRole(ROLES.SEMANTICHUB_VIEW)` | `semantichub.view` |
| `allowTo: () => userHasRegistrationRole(ROLES.VIEW_REGISTRATION)` | `registration.view` |
| `companyHasRole(...)` (company roles) | a permission key the API derives from the company role |

Build the full mapping table in phase 0 from `ALL_PAGES`, `ALL_ACTIONS` and `ALL_OVERLAYS` in `types/Config.tsx`, the `ROLES` enum in `types/Constants.ts` and the `companyHasRole` checks of `portal-frontend`, together with the API team. Actions and overlays that are guarded today get a permission key too; their checks move into the feature that renders them (`useHasPermission`).
