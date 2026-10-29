# 10 – Testing

## Strategy

| Part | What | How | Target |
|------|------|-----|--------|
| core `domain` | Value object factories, anemic factories, UX pre-checks | Vitest, plain unit tests | ≥ 90 % coverage |
| core `application` | Use cases | Vitest with **fake repositories** (in-memory classes implementing the port) | ≥ 90 % coverage |
| core `infrastructure` | Mappers, HTTP adapters | Mappers as unit tests; adapters against **MSW** in Node with the module's handlers | All mappers and adapters covered |
| `@portal/core-http` | Client middleware, `ensureData`, error factory | Vitest + MSW | 100 % (small, used everywhere) |
| `libs/<module>/state` | Endpoints, slices, selectors, thunks | Real store from `createPortalStore` with a **fake module object** in `extra.useCases` | Critical paths |
| `libs/<module>/feature` | Pages, containers | Testing Library + MSW, real core modules, real store | Main flows |
| `libs/shared/ui-*` | Components | Testing Library, optional Storybook | Interaction-heavy components |
| app | Journeys | Playwright against `nx serve` with MSW | Login redirect, marketplace, register a company |

Core packages are tested **without any UI**. If a flow works in the core tests, it works in every app on top.

## Fakes, not mocks

Each core package ships its in-memory port implementations through a separate entry point `@portal/<module>-core/testing`, so both apps can use them in their tests.

```ts
// packages/marketplace-core/src/domain/testing/in-memory-app.repository.ts
import type { AppAnemic } from '../anemics/app.anemic';
import type { AppRepository } from '../repositories/app.repository';
import type { AppId } from '../valueObjects/app-id.value-object';

export class InMemoryAppRepository implements AppRepository {
  public readonly subscribed: Array<AppId> = [];

  constructor(private readonly apps: ReadonlyArray<AppAnemic>) {}

  public findActive(): Promise<ReadonlyArray<AppAnemic>> {
    return Promise.resolve(this.apps);
  }

  public findById(id: AppId): Promise<AppAnemic | null> {
    return Promise.resolve(this.apps.find((app) => app.id === id) ?? null);
  }

  public subscribe(id: AppId): Promise<void> {
    this.subscribed.push(id);
    return Promise.resolve();
  }
}
```

```ts
// packages/marketplace-core/src/application/useCases/subscribe-to-app.use-case.spec.ts
import { describe, expect, it } from 'vitest';
import { AppFactory } from '../../domain/factories/app.factory';
import { AppIdFactory } from '../../domain/factories/app-id.factory';
import { InMemoryAppRepository } from '../../domain/testing/in-memory-app.repository';
import { SubscribeToAppUseCase } from './subscribe-to-app.use-case';

describe('SubscribeToAppUseCase', () => {
  it('passes the subscription to the repository', async () => {
    const app = AppFactory.create({ id: 'a1', name: 'App', provider: 'P', canSubscribe: true });
    const repository = new InMemoryAppRepository([app]);

    await new SubscribeToAppUseCase(repository).execute(AppIdFactory.create('a1'));

    expect(repository.subscribed).toEqual(['a1']);
  });
});
```

## MSW

- Handlers live in each core package under `src/mocks/` and are exported as `@portal/<module>-core/mocks`. Responses are `snake_case` like the real API.
- The same handlers serve: core adapter tests (MSW in Node), feature tests, local development of both apps without the API, and Playwright in CI.
- `@portal/identity-core/mocks` contains a mock BFF for `/me`, `/config`, `/auth/login`, `/auth/logout`. That unblocks development while the API is being rewritten.

## Contract safety

- Types are generated from `specs/<module>/openapi.yaml`. A spec change breaks `typecheck` wherever a mapper or adapter no longer fits. That is intended.
- `check-api` fails if the spec changed but the generated types were not regenerated.
- For endpoints with a zod schema, add one test per schema that parses a recorded real response.

## CI gates

`npx nx affected -t lint typecheck test build check-api e2e` must be green. Coverage thresholds are configured per project in `vite.config.ts`: 90 % for core `domain` and `application`.
