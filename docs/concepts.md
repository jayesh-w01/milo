# Core Concepts

Understanding these concepts will make every Milo workflow make sense. You don't need to be an expert in any of these — Milo guides you through applying them — but knowing *why* they exist will help you make better decisions at each human gate.

---

## global-picture/ — The Living Reference

`global-picture/` is a folder that Milo creates at the root of your project. It is the single source of truth for the entire enterprise state of your project. Every Milo workflow reads from it. Every Milo workflow writes to it. BMAD reads it through the Product Brief.

```
global-picture/
├── standards/
│   ├── coding-standards.md       ← formatting, naming, error handling conventions
│   ├── oss-packages.md           ← the approved OSS packages for every category
│   └── architectural-patterns.md ← module structure, deployment abstraction rules
├── api-docs/
│   └── {module-name}.md          ← function signatures for every completed module
├── features/
│   ├── registry.md               ← all features and their status
│   └── {feature-name}/
│       ├── solution-architecture.md
│       ├── product-brief-{module-name}.md
│       └── module-summary-{module-name}.md
└── oss-registry/
    └── registry.json             ← which OSS packages are used by which modules
```

**The golden rule: never edit global-picture/ manually.**

Every file in it is owned by a specific Milo workflow. Manual edits create inconsistencies between files (e.g. a module marked complete in one file but not updated in the registry). If something is wrong, run the appropriate workflow to fix it — don't edit the files directly.

### What each subfolder does

**`standards/`** — Created by `/define-standards`. Contains the decisions made once that govern all modules forever. When BMAD builds a module, it must follow these. When `/module-summary` validates a module, it checks against these.

**`api-docs/`** — Created and updated by `/module-summary` when a module is approved. Each file is the authoritative API contract for a completed module. When `/initiate-module` generates a Product Brief, it copies the exact function signatures from here so BMAD knows precisely what's available to call.

**`features/`** — Contains everything related to a feature: the solution architecture, the product briefs for each module, and the module summaries. The `registry.md` tracks status across all features.

**`oss-registry/registry.json`** — Tracks every OSS package used across the project. Initialized by `/define-standards`, updated by `/module-summary` after each module completes. Enables enforcement: if a module uses a package not in the registry, it's a blocking issue.

---

## Domain-Driven Design (DDD)

Milo uses DDD to decompose a feature into modules. The goal of DDD is to ensure module boundaries map to real business domains — not to technical concerns, not to database tables, not to team org charts.

**Why this matters:** Modules built with correct domain boundaries can evolve independently. You can replace the implementation of one module without touching any other module. You can deploy them separately or together. The code reads like the business.

### The process Milo follows (during /solution-architecture)

**Step 1: Event Storming — what can happen?**

List everything that can happen in the feature as past-tense events:
- `UserRegistered`
- `EmailVerified`
- `OrderPlaced`
- `PaymentProcessed`
- `InvoiceGenerated`
- `NotificationSent`

**Step 2: Group by domain — what goes together?**

Events that share the same entities and language belong in the same group:
- Identity: `UserRegistered`, `EmailVerified`
- Commerce: `OrderPlaced`
- Finance: `PaymentProcessed`, `InvoiceGenerated`
- Comms: `NotificationSent`

**Step 3: Identify commands — what triggers each event?**

- `RegisterUser` → `UserRegistered`
- `PlaceOrder` → `OrderPlaced`
- `ProcessPayment` → `PaymentProcessed`

**Step 4: Name the modules — each group becomes a bounded context**

- `auth` — owns identity verification and session management
- `orders` — owns order creation and lifecycle
- `payments` — owns payment processing and invoicing
- `notifications` — owns all outbound communication

**Step 5: Draw explicit boundaries — what does each module own and NOT own?**

This is the most important step. For every proposed module, state clearly:
- What it is responsible for
- What it explicitly does NOT do (prevents scope creep)
- What data or capabilities it needs from other modules

### Bounded Context

A bounded context is the primary unit of modularity in Milo. It is a boundary within which the language, concepts, and rules are consistent. Everything inside the `orders` module is about orders. It doesn't know about how payments work internally. It just calls the `payments` module and trusts it.

**The test for a well-drawn boundary:** Can you describe what this module does in one sentence using business language — not technical language — and does everything inside it relate to that sentence?

- Good: "The `auth` module verifies user identity and issues session tokens."
- Bad: "The `auth` module handles the database calls for user login."

### Common Mistakes to Avoid

| Mistake | Why It's Wrong | Fix |
|---------|---------------|-----|
| Boundary around a technical layer ("the database module") | Business logic gets split, not encapsulated | Boundary around a business capability |
| One god module that does everything | Cannot evolve independently, impossible to test in isolation | Split by domain events |
| Separate module for every database table | This is just an ORM, not DDD | Group by business behaviour |
| Two modules writing to the same database table | Hidden coupling — changes in one break the other | One module owns its data exclusively |
| Module A calls Module B which calls Module A | Circular dependency — locks deployment order | Find the missing third module that owns the shared concern |

---

## Deployment Abstraction

This is Milo's most important technical contribution. It solves a real problem: how do you write code that works as a monolith in development but can be deployed as separate microservices in production — **without changing the code**?

The answer is the three-layer pattern. Every Standard API module must implement all three layers.

### The Core Idea

Developers always write plain function calls. They never write HTTP calls between modules. The routing — whether that call goes directly to the function or over the network to a service — is handled automatically by a wrapper that checks an environment variable.

```typescript
// This is all a developer ever writes:
import { authenticateUser } from '@modules/auth';
const result = await authenticateUser(username, password);

// In development (DEPLOYMENT_MODE=monolith):
// → calls the function directly in the same process

// In production (DEPLOYMENT_MODE=distributed):
// → makes an HTTP POST to http://auth-service/api/auth/authenticateUser
```

The developer's code doesn't change. The `DEPLOYMENT_MODE` environment variable changes.

### Layer 1: Function Exports (`index.ts`)

The actual business logic. This is what BMAD writes. Standard TypeScript exports with explicit types.

```typescript
// modules/auth/index.ts
export async function authenticateUser(
  username: string,
  password: string
): Promise<AuthResult> {
  // business logic here
}
```

Rules:
- No HTTP calls in this file
- No deployment-specific code
- All functions are `async` (consistent with distributed mode expectations)
- All parameters and return types are explicit

### Layer 2: REST Server (`server.ts`)

Exposes every exported function as an HTTP endpoint. Handlers are thin — they extract parameters, call the function, return the result. No logic duplication.

```typescript
// modules/auth/server.ts
app.post('/api/auth/authenticateUser', async (req, res) => {
  const { username, password } = req.body;
  const result = await authenticateUser(username, password);
  res.json(result);
});
```

Endpoint pattern: `POST /api/{module-name}/{functionName}`

One endpoint per exported function. Nothing more.

### Layer 3: Smart Call Wrapper (`proxy.ts`)

The routing layer. Any module that imports from `@modules/auth` imports this proxy — not the index directly. The proxy checks `DEPLOYMENT_MODE` and routes accordingly.

```typescript
// modules/auth/proxy.ts
import * as authModule from './index';

const DEPLOYMENT_MODE = process.env.DEPLOYMENT_MODE ?? 'monolith';
const AUTH_SERVICE_URL = process.env.AUTH_SERVICE_URL ?? 'http://localhost:3001';

export const authenticateUser: typeof authModule.authenticateUser =
  DEPLOYMENT_MODE === 'distributed'
    ? (username, password) =>
        fetch(`${AUTH_SERVICE_URL}/api/auth/authenticateUser`, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ username, password }),
        }).then(r => r.json())
    : authModule.authenticateUser;
```

The `@modules/auth` path alias resolves to `proxy.ts`, configured in `tsconfig.json`:

```json
{
  "compilerOptions": {
    "paths": {
      "@modules/*": ["modules/*/proxy.ts"]
    }
  }
}
```

### Why All Three Are Mandatory

If you skip `server.ts`: you can never deploy this module as a separate service — there's no HTTP interface.

If you skip `proxy.ts`: other modules import directly from `index.ts`, which bypasses the routing — you're permanently tied to monolith mode.

If you skip the path alias: even if `proxy.ts` exists, modules won't import it — they'll go directly to `index.ts`.

Milo's `/module-summary` will block completion if any of the three layers are missing or incorrectly implemented.

---

## The OSS Registry

The OSS registry (`global-picture/oss-registry/registry.json`) tracks every open-source package used across the project. It serves two purposes:

1. **Consistency enforcement** — if `zod` is the standard validation library, every module must use `zod`. Not `joi`. Not `yup`. The registry makes this visible and enforceable.
2. **Impact tracking** — when you want to upgrade a package version, the registry tells you every module that uses it, so you can assess the impact.

### How it works

- `/define-standards` creates the initial registry with all approved packages and `modules_using: []`
- `/initiate-module` reads the registry to populate the Product Brief's OSS package section
- `/module-summary` updates `modules_using` after a module is complete, and flags any non-standard packages as blocking issues

### Non-standard packages

Sometimes a module genuinely needs a package outside the standards (e.g. the Stripe SDK for a payments module). The process:

1. Flag it in the Product Brief during `/initiate-module`
2. Human approves it explicitly
3. `/module-summary` records it in the `deviations[]` array in the registry

Non-standard packages used silently (not flagged, not approved) are a blocking issue in `/module-summary`.

---

## Module Types

Not every module is the same. Milo recognises four types:

### Standard API Module

The most common type. Has real domain logic, exposes function exports and REST endpoints, gets full deployment abstraction.

Examples: `auth`, `orders`, `catalog`, `payments`, `notifications`

Required files: `index.ts`, `server.ts`, `proxy.ts`

### Web UI

A frontend application. It consumes Standard API modules via function calls (routed through proxies). It does not expose functions to other modules. It does not get `server.ts` or `proxy.ts`.

Example: `web-app`, `admin-dashboard`

### Mobile App

Same rules as Web UI — consumer only, no API contract, no deployment abstraction files.

### UI Component Library

A shared library of reusable UI components. Exports components and their prop interfaces. No REST API — components are always imported directly, never called over the network. No `proxy.ts`.

Example: `design-system`, `ui-components`

---

## The Dependency Graph

The dependency graph is produced by `/solution-architecture`. It defines which modules must be built before which, and organises them into tiers.

```
Tier 1 (no dependencies — build first):
  auth, catalog

Tier 2 (depends on Tier 1):
  orders (depends on: auth, catalog)
  user-profile (depends on: auth)

Tier 3 (depends on Tier 2):
  checkout (depends on: orders, payments)
  payments (depends on: auth)

Tier 4 (depends on all):
  web-app (depends on: auth, orders, catalog, checkout, user-profile)
```

**Why this order is enforced:**

When `/initiate-module` generates a Product Brief, it copies exact function signatures from `global-picture/api-docs/` for every dependency. If a dependency module isn't complete yet, its api-docs entry doesn't exist — meaning the Product Brief can't be generated. The dependency graph isn't just planning — it's a hard gate on what can be built next.

**Parallel development:**

Multiple modules in the same tier have no dependencies on each other. Teams can build them in parallel on separate branches. Run `/initiate-module` separately for each module you want to work on in parallel.
