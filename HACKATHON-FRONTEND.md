# Hackathon: what frontend people build

Read this first if you are the frontend person in a team. It is meant to work without anyone explaining it to you.

## The setting in one paragraph

At the hackathon there is **one CSP and two OSPs**. Every OSP runs the portal modules itself and has **its own portal UI** (OSP A issues credentials itself, OSP B uses the CSP's issuer). The EBW-Connector (C5-M1), which checks the EUDI Wallet and the Business Wallet, is provided. Business decisions and everything OSP-specific live in the **backend**. The frontend is as dumb as possible: it shows what the API returns and sends what the user enters.

## What you build

The **portal UI of your team's OSP** for the modules your team works on. For every backend module that means two parts:

1. **A core package** – plain TypeScript, no framework. It wraps the module's API: generated types from the module's `openapi.yaml`, a small mapper from `snake_case` to `camelCase`, and one **use case** per thing a user can do ("start registration", "save company data", "submit"). No business rules: if the UI needs to know whether something is allowed, the API returns a flag.
2. **The UI on top** – pages that call the use cases and render the result. Our reference is React + Redux Toolkit; because the core packages are framework-free, another framework works on the same packages.

The first flow to build is **company registration**: connect the wallets through the EBW-Connector, fill in the company data (prefilled and partly verified from the Business Wallet), submit, wait for the review. The backend's Process Orchestration decides which step comes next; the UI only renders the current step. The complete sketch is in [architecture/14-example-registration.md](./architecture/14-example-registration.md).

## Where the code lives

In the **hackathon repository** (the organisers share the link). Inside it:

| What | Where |
|------|-------|
| API specs of the backend modules | `specs/<module>/openapi.yaml` |
| Core package per module | `packages/<module>-core/` (generated types in `src/generated/`, never edit them) |
| Shared HTTP setup (cookies, CSRF, 401) | `packages/core-http/` |
| UI state per module | `libs/<module>/state/` |
| Pages per module | `libs/<module>/feature/` |
| The apps that start everything | `apps/portal/`, `apps/registration/` |

If the repository does not use Nx, keep the same folders as npm workspaces. The layering matters, not the tool.

## Your first hour

1. Read [architecture/README.md](./architecture/README.md) (10 minutes) and skim [03 – Layers & libs](./architecture/03-layers-and-libs.md), especially the table "Where does my code go?".
2. Find out from your backend colleagues which modules your team builds and where their `openapi.yaml` is. Put it into `specs/<module>/`.
3. Generate the types for your module (`openapi-typescript specs/<module>/openapi.yaml -o packages/<module>-core/src/generated/<module>-api.d.ts`).
4. Write the first use case and its adapter, test it against an MSW mock, then build the page.
5. Agree early with the backend which flags and step keys the API returns. That is where most surprises come from.

## Rules that save you time

- **No business decisions in the UI.** No "is this allowed?", no "which step is next?". Ask the API team for a flag or a step key.
- **The UI never calls the API directly.** Always through a use case of the core package.
- **No tokens, no Keycloak in the browser.** Sign-in and session are handled by the backend; the browser only has a session cookie.
- **Texts are i18n keys.** Error and validation messages from the API are keys; the UI translates them.
- **Live updates via SSE**, never WebSockets and never polling.
- **Code style:** one class or type per file (`data/` folders may group interfaces), `if` always with braces.

## When you are stuck

- "Where does this go?" → [03 – Layers & libs](./architecture/03-layers-and-libs.md#where-does-my-code-go)
- "How does a core package look?" → [04 – Core packages](./architecture/04-core-packages.md)
- "How does sign-in work?" → [05 – Auth & session](./architecture/05-auth-and-session.md)
- "How does a whole flow look?" → [14 – Example: company registration](./architecture/14-example-registration.md)
- "Why was it decided like this?" → [13 – Decisions](./architecture/13-decisions.md)
