# 04 – Core packages

## Why core packages

The frontend must be as dumb as possible. Business decisions and everything OSP-specific live in the backend. What remains in the frontend is small but has to exist exactly once: which calls belong to "register a company", in which order, how the data is mapped, how progress is reported.

That part lives in **framework-free TypeScript packages**, one per backend module. Because they do not depend on React, the UI on top stays replaceable:

| Package | Backend module | Contains |
|---------|----------------|----------|
| `@portal/core-http` | – | HTTP client factory, error handling, CSRF and 401 hooks (shared by all modules) |
| `@portal/core-kernel` | – | Value objects and helpers used by several modules (`PermissionKey`, `Bpn`, `ErrorMessage`) |
| `@portal/identity-core` | identity / BFF | Session, login, logout |
| `@portal/registration-core` | registration | Register a company, application status |
| `@portal/marketplace-core` | marketplace | Apps, subscriptions |
| `@portal/<module>-core` | one per backend module | … |

Each core package:

- has **no dependency on React, Angular, Vue, Redux or any UI library**;
- generates its HTTP types from the module's `openapi.yaml`;
- exposes **use cases** as its public API: every operation a UI may trigger, reads included;
- contains **no business decisions**: what is allowed comes from the API (e.g. a `can_subscribe` flag); local checks are UX pre-checks that the API enforces again;
- is a **workspace package**: `apps/portal` and `apps/registration` consume it through the Nx workspace. Nothing is published to a package registry.

Micro-frontends are out of scope for the hackathon.

## What is generated and what is written by hand

| Part | Generated from `openapi.yaml`? | Why |
|------|:------------------------------:|-----|
| Request/response types, paths, parameters | **yes** (`openapi-typescript`) | They are exactly what the spec describes |
| Typed HTTP calls | **yes**, via `openapi-fetch` using the generated `paths` | No handwritten URLs or casts |
| Mappers (`snake_case` DTO ↔ `camelCase` domain) | no | Naming and shape of the domain is our decision |
| Use cases (order of calls, progress) | no | The spec knows endpoints, not workflows |
| UX pre-checks, value objects | no | Immediate feedback for the user; the API stays the authority |

The generated file is never edited. A generator run only ever replaces `src/generated/`.

## Package layout

```text
packages/registration-core/
  src/
    generated/
      registration-api.d.ts            # openapi-typescript output, committed
    data/
      registration-module.data.ts      # public module shape
    domain/
      data/                            # inputs, results (several interfaces per file allowed)
      domains/                         # business rules
      repositories/                    # ports
      testing/                         # in-memory port implementations
    application/
      useCases/                        # one class per operation
    infrastructure/
      constants/
      data/                            # DTO aliases, client type
      mappers/
      repositories/                    # port adapters using the generated client
    mocks/                             # MSW handlers (snake_case responses)
    index.ts                           # createRegistrationModule + public exports
    testing.ts                         # entry point @portal/registration-core/testing
    mocks.ts                           # entry point @portal/registration-core/mocks
  package.json                         # name: @portal/registration-core
  project.json                         # generate-api, build, test, lint
```

Inside a core package the same onion rule applies as everywhere: `application → domain`, `infrastructure → domain`, nothing imports `infrastructure` except `index.ts`. ESLint enforces it per folder (see [02](./02-workspace-and-tooling.md)).

---

## `@portal/core-http`

Shared by all core packages, so CSRF handling, 401 handling and error conversion exist exactly once.

```ts
// packages/core-http/src/data/core-module-config.data.ts
export interface CoreModuleConfig {
  baseUrl: string;
  getCsrfToken?: () => string | null;
  onUnauthorized?: () => void;
}

export interface CoreResponse<T> {
  data?: T;
  error?: unknown;
  response: Response;
}

export interface CoreHttpErrorData {
  status: number | null;
  message: string;
}
```

```ts
// packages/core-http/src/constants/http.constant.ts
export const HTTP = {
  CSRF_HEADER: 'X-CSRF-Token',
  UNAUTHORIZED: 401,
  NOT_FOUND: 404,
} as const;
```

```ts
// packages/core-http/src/core-http.exception.ts
export class CoreHttpException extends Error {
  constructor(
    public readonly status: number,
    message: string,
  ) {
    super(message);
    this.name = 'CoreHttpException';
  }
}
```

```ts
// packages/core-http/src/create-core-client.ts
import createClient, { type Client } from 'openapi-fetch';
import { HTTP } from './constants/http.constant';
import type { CoreModuleConfig } from './data/core-module-config.data';

export function createCoreClient<Paths extends object>(config: CoreModuleConfig): Client<Paths> {
  const client = createClient<Paths>({ baseUrl: config.baseUrl, credentials: 'include' });
  client.use({
    onRequest({ request }) {
      const csrf = config.getCsrfToken?.() ?? null;
      if (csrf !== null) {
        request.headers.set(HTTP.CSRF_HEADER, csrf);
      }
      return request;
    },
    onResponse({ response }) {
      if (response.status === HTTP.UNAUTHORIZED) {
        config.onUnauthorized?.();
      }
      return response;
    },
  });
  return client;
}
```

```ts
// packages/core-http/src/ensure-data.ts
import { CoreHttpException } from './core-http.exception';
import type { CoreResponse } from './data/core-module-config.data';

export async function ensureData<T>(call: Promise<CoreResponse<T>>): Promise<T> {
  const { data, error, response } = await call;
  if (error !== undefined || !response.ok) {
    throw new CoreHttpException(response.status, `${response.status} ${response.url}`);
  }
  return data as T;
}
```

```ts
// packages/core-http/src/ensure-ok.ts
import type { CoreResponse } from './data/core-module-config.data';
import { ensureData } from './ensure-data';

export async function ensureOk(call: Promise<CoreResponse<unknown>>): Promise<void> {
  await ensureData(call);
}
```

```ts
// packages/core-http/src/to-form-data.ts
export function toFormData(body: Record<string, Blob | string>): FormData {
  const formData = new FormData();
  for (const [key, value] of Object.entries(body)) {
    formData.append(key, value);
  }
  return formData;
}
```

```ts
// packages/core-http/src/csrf-token.store.ts
export class CsrfTokenStore {
  private static token: string | null = null;

  public static set(token: string | null): void {
    CsrfTokenStore.token = token;
  }

  public static get(): string | null {
    return CsrfTokenStore.token;
  }
}
```

The CSRF token lives in memory only, never in a store or `localStorage`. Any UI can use `CsrfTokenStore`, or pass its own `getCsrfToken`.

```ts
// packages/core-http/src/core-http-error.factory.ts
import { ErrorMessage } from '@portal/core-kernel';
import { CoreHttpException } from './core-http.exception';
import type { CoreHttpErrorData } from './data/core-module-config.data';

export class CoreHttpErrorFactory {
  public static from(error: unknown): CoreHttpErrorData {
    if (error instanceof CoreHttpException) {
      return { status: error.status, message: error.message };
    }
    return { status: null, message: ErrorMessage.of(error) };
  }
}
```

`CoreHttpErrorData` is a plain object on purpose: UI state libraries (Redux, NgRx, Pinia) can store it without serialisation problems.

## `@portal/core-kernel`

```ts
// packages/core-kernel/src/error-message.ts
export class ErrorMessage {
  public static of(error: unknown): string {
    if (error instanceof Error) {
      return error.message;
    }
    return String(error);
  }
}
```

```ts
// packages/core-kernel/src/valueObjects/permission-key.value-object.ts
export type PermissionKey = string & { readonly __brand: 'PermissionKey' };
```

```ts
// packages/core-kernel/src/factories/permission-key.factory.ts
import type { PermissionKey } from '../valueObjects/permission-key.value-object';

export class PermissionKeyFactory {
  public static create(value: string): PermissionKey {
    if (!/^[a-z]+(\.[a-z]+)+$/.test(value)) {
      throw new Error(`Invalid permission key: ${value}`);
    }
    return value as PermissionKey;
  }
}
```

`Bpn`, `Email` and paging types follow the same pattern (type in `valueObjects/` or `data/`, creation in `factories/`).

---

## Example: `@portal/registration-core` – "Register a company"

> **Superseded for OSP 2.0.** This example shows the mechanics of a core package (ports, adapters, mappers, result unions, progress). In OSP 2.0 the **Process Orchestration** runs the steps of the registration in the backend, so the frontend no longer chains the calls itself. The current registration building block is in [14 – Example: company registration in OSP 2.0](./14-example-registration.md).

In the legacy `portal-frontend-registration` this is a wizard spread over Redux slices. As a core use case it becomes one call with progress feedback. The flow uses these endpoints (paths from the legacy app, fields in `snake_case`; the real names come from the module's `openapi.yaml`):

1. `GET companyRoleAgreementData` – which agreements are mandatory for the chosen roles
2. `POST application/{id}/companyDetailsWithAddress`
3. `POST application/{id}/companyRoleAgreementConsents`
4. `POST application/{id}/documentType/{type}/documents`
5. `POST application/{id}/inviteNewUser` – once per user
6. `POST application/{id}/submitRegistration`

### Domain

```ts
// src/domain/data/register-company.data.ts
export interface CompanyAddressData {
  street: string;
  houseNumber: string;
  zipCode: string;
  city: string;
  countryCode: string;
}

export interface CompanyIdentifierData {
  type: string;
  value: string;
}

export interface InvitedUserData {
  email: string;
  firstName: string;
  lastName: string;
}

export interface RegisterCompanyData {
  applicationId: string;
  name: string;
  address: CompanyAddressData;
  identifiers: ReadonlyArray<CompanyIdentifierData>;
  companyRoles: ReadonlyArray<string>;
  consentedAgreementIds: ReadonlyArray<string>;
  commercialRegisterExtract: Blob;
  invitedUsers: ReadonlyArray<InvitedUserData>;
}
```

```ts
// src/domain/data/register-company-result.data.ts
export type RegistrationStep = 'company-details' | 'company-roles' | 'documents' | 'invite-users' | 'submit';

export interface RegisterCompanySubmitted {
  status: 'submitted';
}

export interface RegisterCompanyInvalid {
  status: 'invalid';
  errors: ReadonlyArray<string>;
}

export interface RegisterCompanyFailed {
  status: 'failed';
  step: RegistrationStep;
  message: string;
}

export type RegisterCompanyResult = RegisterCompanySubmitted | RegisterCompanyInvalid | RegisterCompanyFailed;

export interface RegisterCompanyOptions {
  onStep?: (step: RegistrationStep) => void;
}
```

```ts
// src/domain/repositories/registration.repository.ts
import type { InvitedUserData, RegisterCompanyData } from '../data/register-company.data';

export interface RegistrationRepository {
  getMandatoryAgreementIds(companyRoles: ReadonlyArray<string>): Promise<ReadonlyArray<string>>;
  saveCompanyDetails(input: RegisterCompanyData): Promise<void>;
  consentToRoles(applicationId: string, companyRoles: ReadonlyArray<string>, agreementIds: ReadonlyArray<string>): Promise<void>;
  uploadCommercialRegisterExtract(applicationId: string, file: Blob): Promise<void>;
  inviteUsers(applicationId: string, users: ReadonlyArray<InvitedUserData>): Promise<void>;
  submit(applicationId: string): Promise<void>;
}
```

```ts
// src/domain/domains/registration-validation.domain.ts
import type { RegisterCompanyData } from '../data/register-company.data';

export class RegistrationValidationDomain {
  public static validate(input: RegisterCompanyData, mandatoryAgreementIds: ReadonlyArray<string>): ReadonlyArray<string> {
    const errors: Array<string> = [];
    if (input.name.trim().length === 0) {
      errors.push('registration.company.name.required');
    }
    if (input.identifiers.length === 0) {
      errors.push('registration.company.identifier.required');
    }
    if (input.companyRoles.length === 0) {
      errors.push('registration.company.roles.required');
    }
    const missingAgreements = mandatoryAgreementIds.filter((id) => !input.consentedAgreementIds.includes(id));
    if (missingAgreements.length > 0) {
      errors.push('registration.company.agreements.missing');
    }
    return errors;
  }
}
```

This is a **UX pre-check**: the user sees missing data before six calls start. The registration API validates the same again and stays the authority. Validation errors are **i18n keys**, not texts. Every UI translates them itself.

### Application

```ts
// src/application/useCases/register-company.use-case.ts
import { ErrorMessage } from '@portal/core-kernel';
import type { RegisterCompanyData } from '../../domain/data/register-company.data';
import type {
  RegisterCompanyOptions,
  RegisterCompanyResult,
  RegistrationStep,
} from '../../domain/data/register-company-result.data';
import { RegistrationValidationDomain } from '../../domain/domains/registration-validation.domain';
import type { RegistrationRepository } from '../../domain/repositories/registration.repository';

export class RegisterCompanyUseCase {
  constructor(private readonly registration: RegistrationRepository) {}

  public async execute(input: RegisterCompanyData, options: RegisterCompanyOptions = {}): Promise<RegisterCompanyResult> {
    const mandatoryAgreementIds = await this.registration.getMandatoryAgreementIds(input.companyRoles);
    const errors = RegistrationValidationDomain.validate(input, mandatoryAgreementIds);
    if (errors.length > 0) {
      return { status: 'invalid', errors };
    }

    const steps: ReadonlyArray<[RegistrationStep, () => Promise<void>]> = [
      ['company-details', () => this.registration.saveCompanyDetails(input)],
      ['company-roles', () => this.registration.consentToRoles(input.applicationId, input.companyRoles, input.consentedAgreementIds)],
      ['documents', () => this.registration.uploadCommercialRegisterExtract(input.applicationId, input.commercialRegisterExtract)],
      ['invite-users', () => this.registration.inviteUsers(input.applicationId, input.invitedUsers)],
      ['submit', () => this.registration.submit(input.applicationId)],
    ];

    for (const [step, run] of steps) {
      options.onStep?.(step);
      try {
        await run();
      } catch (error) {
        return { status: 'failed', step, message: ErrorMessage.of(error) };
      }
    }
    return { status: 'submitted' };
  }
}
```

Progress is reported through a callback, not a store: the consumer decides where state lives.

### Infrastructure

```ts
// src/infrastructure/data/registration-infrastructure.data.ts
import type { Client } from 'openapi-fetch';
import type { components, paths } from '../../generated/registration-api';

export type RegistrationClient = Client<paths>;
export type CompanyDetailsRequestDto = components['schemas']['CompanyDetailsRequest'];
export type InviteUserRequestDto = components['schemas']['InviteUserRequest'];
```

```ts
// src/infrastructure/constants/document-type.constant.ts
export const COMMERCIAL_REGISTER_EXTRACT = 'COMMERCIAL_REGISTER_EXTRACT';
```

```ts
// src/infrastructure/mappers/company-details.mapper.ts
import type { RegisterCompanyData } from '../../domain/data/register-company.data';
import type { CompanyDetailsRequestDto } from '../data/registration-infrastructure.data';

export class CompanyDetailsMapper {
  public static toRequest(input: RegisterCompanyData): CompanyDetailsRequestDto {
    return {
      name: input.name,
      street_name: input.address.street,
      street_number: input.address.houseNumber,
      zip_code: input.address.zipCode,
      city: input.address.city,
      country_alpha2_code: input.address.countryCode,
      unique_ids: input.identifiers.map((identifier) => ({ type: identifier.type, value: identifier.value })),
    };
  }
}
```

```ts
// src/infrastructure/mappers/invite-user.mapper.ts
import type { InvitedUserData } from '../../domain/data/register-company.data';
import type { InviteUserRequestDto } from '../data/registration-infrastructure.data';

export class InviteUserMapper {
  public static toRequest(user: InvitedUserData): InviteUserRequestDto {
    return {
      email: user.email,
      first_name: user.firstName,
      last_name: user.lastName,
    };
  }
}
```

```ts
// src/infrastructure/repositories/registration-http.repository.ts
import { ensureData, ensureOk, toFormData } from '@portal/core-http';
import type { InvitedUserData, RegisterCompanyData } from '../../domain/data/register-company.data';
import type { RegistrationRepository } from '../../domain/repositories/registration.repository';
import { COMMERCIAL_REGISTER_EXTRACT } from '../constants/document-type.constant';
import type { RegistrationClient } from '../data/registration-infrastructure.data';
import { CompanyDetailsMapper } from '../mappers/company-details.mapper';
import { InviteUserMapper } from '../mappers/invite-user.mapper';

export class RegistrationHttpRepository implements RegistrationRepository {
  constructor(private readonly client: RegistrationClient) {}

  public async getMandatoryAgreementIds(companyRoles: ReadonlyArray<string>): Promise<ReadonlyArray<string>> {
    const data = await ensureData(this.client.GET('/api/registration/companyRoleAgreementData'));
    const roleAgreementIds = data.company_roles
      .filter((role) => companyRoles.includes(role.company_role))
      .flatMap((role) => role.agreement_ids);
    return data.agreements
      .filter((agreement) => agreement.mandatory && roleAgreementIds.includes(agreement.agreement_id))
      .map((agreement) => agreement.agreement_id);
  }

  public saveCompanyDetails(input: RegisterCompanyData): Promise<void> {
    return ensureOk(
      this.client.POST('/api/registration/application/{application_id}/companyDetailsWithAddress', {
        params: { path: { application_id: input.applicationId } },
        body: CompanyDetailsMapper.toRequest(input),
      }),
    );
  }

  public consentToRoles(
    applicationId: string,
    companyRoles: ReadonlyArray<string>,
    agreementIds: ReadonlyArray<string>,
  ): Promise<void> {
    return ensureOk(
      this.client.POST('/api/registration/application/{application_id}/companyRoleAgreementConsents', {
        params: { path: { application_id: applicationId } },
        body: {
          company_roles: [...companyRoles],
          agreements: agreementIds.map((agreementId) => ({ agreement_id: agreementId, consent_status: 'ACTIVE' })),
        },
      }),
    );
  }

  public uploadCommercialRegisterExtract(applicationId: string, file: Blob): Promise<void> {
    return ensureOk(
      this.client.POST('/api/registration/application/{application_id}/documentType/{document_type}/documents', {
        params: { path: { application_id: applicationId, document_type: COMMERCIAL_REGISTER_EXTRACT } },
        body: { document: file },
        bodySerializer: toFormData,
      }),
    );
  }

  public async inviteUsers(applicationId: string, users: ReadonlyArray<InvitedUserData>): Promise<void> {
    for (const user of users) {
      await ensureOk(
        this.client.POST('/api/registration/application/{application_id}/inviteNewUser', {
          params: { path: { application_id: applicationId } },
          body: InviteUserMapper.toRequest(user),
        }),
      );
    }
  }

  public submit(applicationId: string): Promise<void> {
    return ensureOk(
      this.client.POST('/api/registration/application/{application_id}/submitRegistration', {
        params: { path: { application_id: applicationId } },
      }),
    );
  }
}
```

### Public API

```ts
// src/data/registration-module.data.ts
import type { RegisterCompanyUseCase } from '../application/useCases/register-company.use-case';

export interface RegistrationModule {
  registerCompany: RegisterCompanyUseCase;
}
```

```ts
// src/index.ts
import { createCoreClient, type CoreModuleConfig } from '@portal/core-http';
import { RegisterCompanyUseCase } from './application/useCases/register-company.use-case';
import type { RegistrationModule } from './data/registration-module.data';
import type { paths } from './generated/registration-api';
import { RegistrationHttpRepository } from './infrastructure/repositories/registration-http.repository';

export function createRegistrationModule(config: CoreModuleConfig): RegistrationModule {
  const repository = new RegistrationHttpRepository(createCoreClient<paths>(config));
  return {
    registerCompany: new RegisterCompanyUseCase(repository),
  };
}

export type { RegistrationModule } from './data/registration-module.data';
export type {
  CompanyAddressData,
  CompanyIdentifierData,
  InvitedUserData,
  RegisterCompanyData,
} from './domain/data/register-company.data';
export type {
  RegisterCompanyOptions,
  RegisterCompanyResult,
  RegistrationStep,
} from './domain/data/register-company-result.data';
export type { RegistrationRepository } from './domain/repositories/registration.repository';
export { RegisterCompanyUseCase } from './application/useCases/register-company.use-case';
```

`RegistrationRepository` and `RegisterCompanyUseCase` are exported too, so tests and other UIs can plug in their own adapter (e.g. the in-memory one).

---

## Using a core package

### React (our apps, with Redux Toolkit)

See [06 – Data flow](./06-data-flow-redux.md): the container creates the module once, RTK Query endpoints and thunks call its use cases.

### Any other UI

The UI on top is replaceable. These two snippets show that the same package works unchanged under Angular or Vue; we do not build those UIs.

#### Angular

```ts
// registration.service.ts (an Angular UI)
import { Injectable, signal } from '@angular/core';
import {
  createRegistrationModule,
  type RegisterCompanyData,
  type RegisterCompanyResult,
  type RegistrationStep,
} from '@portal/registration-core';
import { environment } from '../environments/environment';

@Injectable({ providedIn: 'root' })
export class RegistrationService {
  private readonly registration = createRegistrationModule({ baseUrl: environment.apiBaseUrl });

  public readonly currentStep = signal<RegistrationStep | null>(null);

  public register(input: RegisterCompanyData): Promise<RegisterCompanyResult> {
    return this.registration.registerCompany.execute(input, {
      onStep: (step) => {
        this.currentStep.set(step);
      },
    });
  }
}
```

#### Vue

```ts
// registration.module.ts (a Vue UI)
import { createRegistrationModule } from '@portal/registration-core';

export const registration = createRegistrationModule({ baseUrl: import.meta.env.VITE_API_BASE_URL });
```

```ts
// use-register-company.ts (a Vue UI)
import { ref } from 'vue';
import type { RegisterCompanyData, RegisterCompanyResult, RegistrationStep } from '@portal/registration-core';
import { registration } from './registration.module';

export function useRegisterCompany() {
  const currentStep = ref<RegistrationStep | null>(null);

  function register(input: RegisterCompanyData): Promise<RegisterCompanyResult> {
    return registration.registerCompany.execute(input, {
      onStep: (step) => {
        currentStep.value = step;
      },
    });
  }

  return { currentStep, register };
}
```

---

## Getting the specs, generating, building

Each core package has these Nx targets in `project.json`:

```json
{
  "name": "registration-core",
  "projectType": "library",
  "tags": ["type:core", "scope:registration"],
  "targets": {
    "generate-api": {
      "command": "openapi-typescript ../../specs/registration/openapi.yaml -o src/generated/registration-api.d.ts",
      "options": { "cwd": "packages/registration-core" }
    },
    "check-api": {
      "command": "nx run registration-core:generate-api && git diff --exit-code -- packages/registration-core/src/generated"
    },
    "build": {
      "executor": "@nx/js:tsc",
      "outputs": ["{options.outputPath}"],
      "options": {
        "outputPath": "dist/packages/registration-core",
        "main": "packages/registration-core/src/index.ts",
        "tsConfig": "packages/registration-core/tsconfig.lib.json"
      }
    }
  }
}
```

- **Specs** live in `specs/<module>/openapi.yaml`. That makes builds reproducible and shows API changes in the PR diff.
- **CI** runs `check-api` for every core package: a spec change without regenerated types fails the build.
- **Build output**: ES modules plus `.d.ts`. The apps bundle the packages with Vite; nothing is published.

### Getting a spec into the workspace

The backend does not exist yet. Every backend module will provide an `openapi.yaml`; until then we write the specs for the pilot modules ourselves, together with the API team, and use them with MSW.

**By hand** – copy the file and regenerate:

```bash
cp <path-to-backend>/openapi.yaml specs/registration/openapi.yaml
npx nx run registration-core:generate-api
```

**By GitHub Action** – once the backend repositories exist, this workflow pulls a spec from a backend release and opens a PR. Start it by hand (`workflow_dispatch`) or let the backend pipeline trigger it after a release (`repository_dispatch`).

```yaml
# .github/workflows/sync-openapi.yml
name: Sync OpenAPI spec

on:
  workflow_dispatch:
    inputs:
      module:
        description: Module name, e.g. registration
        required: true
      source_repo:
        description: Backend repository (owner/name)
        required: true
      ref:
        description: Tag or branch of the backend release
        required: true
      spec_path:
        description: Path of openapi.yaml in the backend repository
        required: true
        default: openapi.yaml
  repository_dispatch:
    types: [openapi-released]

permissions:
  contents: write
  pull-requests: write

jobs:
  sync:
    runs-on: ubuntu-latest
    env:
      MODULE: ${{ inputs.module || github.event.client_payload.module }}
      SOURCE_REPO: ${{ inputs.source_repo || github.event.client_payload.source_repo }}
      REF: ${{ inputs.ref || github.event.client_payload.ref }}
      SPEC_PATH: ${{ inputs.spec_path || github.event.client_payload.spec_path }}
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm

      - run: npm ci

      - name: Download spec
        env:
          GH_TOKEN: ${{ secrets.BACKEND_REPO_TOKEN }}
        run: |
          mkdir -p "specs/${MODULE}"
          gh api "repos/${SOURCE_REPO}/contents/${SPEC_PATH}?ref=${REF}" \
            -H "Accept: application/vnd.github.raw" > "specs/${MODULE}/openapi.yaml"

      - name: Regenerate types
        run: npx nx run "${MODULE}-core:generate-api"

      - uses: peter-evans/create-pull-request@v7
        with:
          branch: chore/openapi-${{ env.MODULE }}-${{ env.REF }}
          title: "chore(${{ env.MODULE }}): sync openapi.yaml ${{ env.REF }}"
          commit-message: "chore(${{ env.MODULE }}): sync openapi.yaml ${{ env.REF }}"
          body: |
            Spec synced from `${{ env.SOURCE_REPO }}@${{ env.REF }}`.
            CI typecheck shows which mappers and adapters must follow the change.
```

`BACKEND_REPO_TOKEN` is a token with read access to the backend repositories. The backend pipeline triggers the sync with `POST /repos/<owner>/portal-nx/dispatches` and `{"event_type":"openapi-released","client_payload":{"module":"registration","source_repo":"<owner>/registration-service","ref":"v1.2.0","spec_path":"openapi.yaml"}}`.

## Rules for core packages

| Rule | Detail |
|------|--------|
| No business decisions | What is allowed or valid comes from the API. Local checks are UX pre-checks that the API enforces again |
| No framework, no state library | No React, Angular, Vue, Redux, RxJS. Only `openapi-fetch`, `zod` and our own `@portal/core-*` |
| Every UI operation is a use case | Reads too (`ListActiveAppsUseCase`). UIs never call the client directly |
| Plain data out | Anemics, result unions, `CoreHttpErrorData`. No class instances that a store cannot serialise |
| Errors as data where the UI must react | Result unions (`invalid`, `failed`) for expected outcomes; exceptions only for unexpected failures |
| Texts are keys | Validation and error messages are i18n keys |
| No browser globals except in infrastructure | `window`, `document`, `localStorage` only in adapters (e.g. the login redirect) |
