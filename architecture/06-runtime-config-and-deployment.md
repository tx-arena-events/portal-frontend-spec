# 06 – Runtime config & deployment

## Goal: build once, run everywhere

The frontend is built **once** into one Docker image. The same image runs in dev, int and prod. Everything that differs per environment comes from `/config.json` at runtime.

`import.meta.env` alone would bake values into the bundle at build time and force one image per environment. `window.__ENV__` globals are not used either.

## What is in `config.json`

Only what the frontend needs **before** it can talk to the API. Everything else (logo, feature flags, languages) comes from `GET /config` on the API.

```json
{
  "api_base_url": "https://api.portal.example.org",
  "environment": "int"
}
```

## `shared/infrastructure-config`

```text
libs/shared/infrastructure-config/src/
  data/env.data.ts            # EnvConfig interface
  schemas/env.schema.ts       # zod schema for config.json
  env.store.ts                # holds the loaded values
  load-env.ts                 # loads and validates once
  env.ts                      # ENV accessor used by other libs
  index.ts
```

```ts
// schemas/env.schema.ts
import { z } from 'zod';

export const envSchema = z.object({
  api_base_url: z.url(),
  environment: z.enum(['local', 'dev', 'int', 'prod']),
});
```

```ts
// load-env.ts
import { envSchema } from './schemas/env.schema';
import { EnvStore } from './env.store';

export async function loadEnv(): Promise<void> {
  if (import.meta.env.DEV) {
    EnvStore.set({ apiBaseUrl: import.meta.env.VITE_API_BASE_URL, environment: 'local' });
    return;
  }
  const response = await fetch('/config.json', { cache: 'no-store' });
  if (!response.ok) {
    throw new Error(`config.json not available: ${response.status}`);
  }
  const parsed = envSchema.parse(await response.json());
  EnvStore.set({ apiBaseUrl: parsed.api_base_url, environment: parsed.environment });
}
```

```ts
// env.ts
import { EnvStore } from './env.store';

export const ENV = {
  get apiBaseUrl(): string {
    return EnvStore.get().apiBaseUrl;
  },
  get environment(): string {
    return EnvStore.get().environment;
  },
};
```

`EnvStore.get()` throws if `loadEnv()` has not run. That makes a wrong bootstrap order fail loudly.

## Bootstrap order in `main.tsx`

`loadEnv()` must finish **before** any module that reads `ENV` is evaluated. Import the rest dynamically:

```tsx
// apps/portal/src/main.tsx
import { loadEnv } from '@portal/shared-infrastructure-config';

async function bootstrap(): Promise<void> {
  await loadEnv();
  const { renderApp } = await import('./render-app');
  renderApp(document.getElementById('root'));
}

bootstrap().catch((error: unknown) => {
  document.body.textContent = 'The portal could not start. Please try again later.';
  console.error(error);
});
```

`render-app.tsx` creates the store, the router and calls `createRoot(...).render(...)`.

## Local development

- `apps/portal/.env.local` (git-ignored): `VITE_API_BASE_URL=http://localhost:5173/mock-api` or the dev API.
- With MSW enabled (`VITE_USE_MOCKS=true`), `render-app` starts the MSW worker before rendering. Handlers live in each context's `infrastructure/src/mocks/` and are collected by the app.

## Docker image

```dockerfile
# apps/portal/Dockerfile
FROM node:22-alpine AS build
WORKDIR /workspace
COPY package.json package-lock.json ./
RUN npm ci
COPY . .
RUN npx nx build portal --configuration=production

FROM nginx:1.27-alpine
COPY apps/portal/nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /workspace/dist/apps/portal /usr/share/nginx/html
# config.json is mounted at runtime, never baked in
```

```nginx
# apps/portal/nginx.conf
server {
  listen 8080;
  root /usr/share/nginx/html;

  location = /config.json {
    add_header Cache-Control "no-store";
    try_files /config/config.json =404;
  }

  location /assets/ {
    add_header Cache-Control "public, max-age=31536000, immutable";
  }

  location / {
    add_header Cache-Control "no-cache";
    try_files $uri /index.html;
  }
}
```

Check the actual build output path (`dist/apps/portal` or `apps/portal/dist`) after generating the app and adjust the `COPY`.

## Kubernetes

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: portal-frontend-config
data:
  config.json: |
    { "api_base_url": "https://api.portal.example.org", "environment": "prod" }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: portal-frontend
spec:
  replicas: 2
  selector:
    matchLabels: { app: portal-frontend }
  template:
    metadata:
      labels: { app: portal-frontend }
    spec:
      containers:
        - name: portal
          image: <registry>/portal-frontend:<version>
          ports:
            - containerPort: 8080
          volumeMounts:
            - name: config
              mountPath: /usr/share/nginx/html/config
              readOnly: true
      volumes:
        - name: config
          configMap:
            name: portal-frontend-config
```

Operations requirements to hand over:

- Frontend and API share the same registrable domain (cookie `SameSite=Lax`, see [04](./04-auth-and-session.md)).
- The API allows the frontend origin in CORS with credentials.
- `config.json` per environment as a ConfigMap; no rebuild for config changes.
