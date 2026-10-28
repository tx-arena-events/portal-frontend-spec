# 09 – Testing

## Strategy per layer

| Layer | What | How | Target |
|-------|------|-----|--------|
| `domain` | Value objects, factories, domain services | Vitest, plain unit tests | ≥ 90 % coverage |
| `application` | Use cases | Vitest with **fake repositories** (in-memory classes implementing the port) | ≥ 90 % coverage |
| `infrastructure` | Mappers, zod schemas, adapters | Mappers/schemas as unit tests; endpoints and adapters against **MSW** | All mappers covered |
| `state` | Slices, selectors, thunks | Reducers and selectors as pure tests; thunks with a fake `extra.useCases` | Critical paths |
| `feature` | Pages and containers | Testing Library + MSW, real store via `createPortalStore` | Main flows |
| `ui` | Components | Testing Library, optional Storybook | Interaction-heavy components |
| app | Journeys | Playwright against `nx serve` with MSW mocks | Login redirect, marketplace, one admin flow |

## Fakes, not mocks

Implement ports with small in-memory classes in a `testing/` folder of the domain lib (exported via a separate entry point `@portal/marketplace-domain/testing`):

```ts
// libs/marketplace/domain/src/testing/in-memory-app.repository.ts
import type { AppAnemic } from '../anemics/app.anemic';
import type { AppRepository } from '../repositories/app.repository';
import type { AppId } from '../valueObjects/app-id.value-object';

export class InMemoryAppRepository implements AppRepository {
  public subscribed: Array<AppId> = [];

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
// libs/marketplace/application/src/useCases/subscribe-to-app.use-case.spec.ts
import { describe, expect, it } from 'vitest';
import { AppFactory, AppIdFactory } from '@portal/marketplace-domain';
import { InMemoryAppRepository } from '@portal/marketplace-domain/testing';
import { SubscribeToAppUseCase } from './subscribe-to-app.use-case';

describe('SubscribeToAppUseCase', () => {
  it('subscribes an app that is not subscribed yet', async () => {
    const app = AppFactory.create({ id: 'a1', name: 'App', provider: 'P', useCases: ['traceability'] });
    const repo = new InMemoryAppRepository([app]);

    await new SubscribeToAppUseCase(repo).execute(AppIdFactory.create('a1'));

    expect(repo.subscribed).toEqual(['a1']);
  });
});
```

## MSW

- Handlers live next to the endpoints: `libs/<context>/infrastructure/src/mocks/*.handlers.ts`, responses in `snake_case` like the real API.
- The same handlers serve three purposes: unit/integration tests, local development without the API, and Playwright runs in CI.
- Include a mock BFF for `/me`, `/config`, `/auth/login`, `/auth/logout` in `identity/infrastructure`. That unblocks development while the API is being rewritten.

## Contract safety

- DTO types are generated from the OpenAPI spec (`generate-api-types`). A changed spec breaks `typecheck` where mappers no longer fit. That is intended.
- For endpoints with zod schemas, add one test per schema that parses a recorded real response.

## CI gates

`npx nx affected -t lint typecheck test build e2e` must be green. Coverage thresholds are configured per lib in `vite.config.ts` for `domain` and `application`.
