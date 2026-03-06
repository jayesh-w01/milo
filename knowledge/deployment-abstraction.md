# Deployment Abstraction
## Reference for Milo's initiate-module workflow and module-summary validation

---

## The Core Principle

Developers write plain function calls. Always. No HTTP calls, no REST calls, no fetch calls between modules in business code.

```typescript
// Developer writes this — always
import { authenticateUser } from '@modules/auth';
const result = await authenticateUser(userId, password);

// What actually runs depends on DEPLOYMENT_MODE:
// DEPLOYMENT_MODE=monolith    → direct function call, same process
// DEPLOYMENT_MODE=distributed → REST call to auth service endpoint
```

This separation means:
- Business logic contains zero deployment concerns
- The same codebase runs as a monolith in dev/staging and as distributed services in production
- Moving from monolith to distributed requires no code changes — only config

---

## The Three-Layer Pattern

Every Standard API module must generate three things. This is non-negotiable.

### Layer 1: Function Exports (`index.ts`)

The actual business logic. Standard TypeScript exports. This is the primary interface.

```typescript
// modules/auth/index.ts

import { db } from '@lib/database';
import { hashPassword } from './internal/crypto';
import type { AuthResult } from './types';

export async function authenticateUser(
  username: string,
  password: string
): Promise<AuthResult> {
  const user = await db.users.findOne({ username });
  if (!user || !await hashPassword.verify(password, user.passwordHash)) {
    throw new AuthenticationError('Invalid credentials');
  }
  const token = generateToken(user.id);
  return { token, expiresIn: 3600, userId: user.id };
}

export async function validateToken(
  token: string
): Promise<{ userId: string; valid: boolean }> {
  // implementation
}
```

Key rules:
- No deployment-specific code in this file
- No HTTP calls in this file
- All functions are `async` (consistent with distributed mode expectations)
- All functions are properly typed — input types and return types explicit

---

### Layer 2: REST API Server (`server.ts`)

Exposes every exported function as an HTTP endpoint. Handlers are thin wrappers — they extract parameters, call the function, and return the result. No business logic here.

```typescript
// modules/auth/server.ts

import { app } from '@lib/http-server'; // uses the standard HTTP framework from oss-packages
import { authenticateUser, validateToken } from './index';
import type { Request, Response } from '{http-framework}';

app.post('/api/auth/authenticateUser', async (req: Request, res: Response) => {
  try {
    const { username, password } = req.body;
    const result = await authenticateUser(username, password);
    res.json(result);
  } catch (error) {
    if (error instanceof AuthenticationError) {
      res.status(401).json({ error: error.message });
    } else {
      res.status(500).json({ error: 'Internal server error' });
    }
  }
});

app.post('/api/auth/validateToken', async (req: Request, res: Response) => {
  try {
    const { token } = req.body;
    const result = await validateToken(token);
    res.json(result);
  } catch (error) {
    res.status(500).json({ error: 'Internal server error' });
  }
});
```

Endpoint URL pattern: `POST /api/{module-name}/{functionName}`

Key rules:
- One endpoint per exported function
- Request body maps to function parameters
- Response body is the function return value
- HTTP status codes must be meaningful (400 for bad input, 401 for auth, 404 for not found, 500 for unexpected)
- Handlers call the function once — no logic duplication

---

### Layer 3: Smart Call Wrapper (`proxy.ts`)

The routing layer. Any module that imports from `@modules/auth` imports from the proxy, not from `index.ts` directly. The proxy checks `DEPLOYMENT_MODE` and routes accordingly.

```typescript
// modules/auth/proxy.ts

import * as authModule from './index';

const DEPLOYMENT_MODE = process.env.DEPLOYMENT_MODE ?? 'monolith';
const AUTH_SERVICE_URL = process.env.AUTH_SERVICE_URL ?? 'http://localhost:3001';

async function callRemote(functionName: string, args: Record<string, unknown>) {
  const response = await fetch(`${AUTH_SERVICE_URL}/api/auth/${functionName}`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(args),
  });
  if (!response.ok) {
    const error = await response.json();
    throw new Error(error.message ?? 'Remote call failed');
  }
  return response.json();
}

export const authenticateUser: typeof authModule.authenticateUser =
  DEPLOYMENT_MODE === 'distributed'
    ? (username, password) => callRemote('authenticateUser', { username, password })
    : authModule.authenticateUser;

export const validateToken: typeof authModule.validateToken =
  DEPLOYMENT_MODE === 'distributed'
    ? (token) => callRemote('validateToken', { token })
    : authModule.validateToken;
```

Alternative proxy pattern using JavaScript Proxy object (for modules with many functions):

```typescript
// modules/auth/proxy.ts

import * as authModule from './index';

const DEPLOYMENT_MODE = process.env.DEPLOYMENT_MODE ?? 'monolith';
const SERVICE_URL = process.env.AUTH_SERVICE_URL ?? 'http://localhost:3001';

function createRestProxy(moduleName: string, serviceUrl: string) {
  return new Proxy({} as typeof authModule, {
    get(_target, functionName: string) {
      return async (...args: unknown[]) => {
        const response = await fetch(`${serviceUrl}/api/${moduleName}/${functionName}`, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(Object.fromEntries(
            Object.keys(authModule[functionName as keyof typeof authModule]
              .toString()
              .match(/\(([^)]*)\)/)?.[1]
              ?.split(',') ?? []).map((p, i) => [p.trim(), args[i]])
          )),
        });
        return response.json();
      };
    },
  });
}

// Export whichever implementation is active
export const { authenticateUser, validateToken } =
  DEPLOYMENT_MODE === 'distributed'
    ? createRestProxy('auth', SERVICE_URL)
    : authModule;
```

> **Note on the Proxy pattern:** The explicit export pattern (first example) is preferred for type safety. The dynamic Proxy pattern works but loses TypeScript type checking. Use whichever is appropriate for the project's language and standards.

Key rules:
- The proxy must export the same function signatures as `index.ts`
- Types must match exactly (use `typeof authModule.functionName` to enforce this)
- Monolith mode: call the function directly — no HTTP overhead
- Distributed mode: call the REST endpoint via the standard HTTP client from org standards
- Service URLs come from environment variables — never hardcoded
- The proxy is the ONLY thing other modules import from this module

---

## How Other Modules Import

```typescript
// modules/orders/index.ts — calling the auth module

// CORRECT — import from proxy
import { authenticateUser } from '@modules/auth';

// WRONG — import from index directly (bypasses routing)
import { authenticateUser } from '@modules/auth/index';
```

The `@modules/auth` path alias should resolve to `modules/auth/proxy.ts`.

Configure this in `tsconfig.json` (TypeScript) or equivalent:
```json
{
  "compilerOptions": {
    "paths": {
      "@modules/*": ["modules/*/proxy.ts"]
    }
  }
}
```

---

## DEPLOYMENT_MODE Environment Variable

| Value | Behaviour |
|-------|----------|
| `monolith` | All modules linked together, all calls are direct function calls, single process |
| `distributed` | Each module deployed as a separate service, all calls are REST API calls |

Default: `monolith` (safe default — works without any infrastructure setup)

---

## Special Module Types

### Web UI / Mobile App
- Consumers only. They call API modules using function call syntax (via proxies).
- They do NOT generate `server.ts` or `proxy.ts`.
- In distributed mode, their proxy calls go to remote services.
- In monolith mode, the same code runs with direct function calls.

### UI Component Library
- Component exports only.
- No `server.ts` — components are not exposed via REST.
- No `proxy.ts` — components are always imported directly (never over network).
- Import pattern: `import { Button, Modal } from '@components/ui'`

---

## Validation Checklist (used by /module-summary)

For every Standard API module, confirm:

- [ ] `index.ts` exports all functions from the Product Brief
- [ ] All exported functions are `async` and typed
- [ ] `server.ts` exists and has one POST endpoint per exported function
- [ ] Endpoint paths follow `/api/{module-name}/{functionName}` pattern
- [ ] `server.ts` handlers call functions directly — no logic duplication
- [ ] `proxy.ts` exists and exports the same functions as `index.ts`
- [ ] `proxy.ts` checks `DEPLOYMENT_MODE`
- [ ] Monolith path calls `index.ts` directly
- [ ] Distributed path makes a fetch/HTTP call to the REST endpoint
- [ ] Service URL is from environment variable, not hardcoded
- [ ] Path alias `@modules/{module-name}` resolves to `proxy.ts`
- [ ] All other modules import this module via `@modules/{module-name}` (the proxy)

---

## Phase 2 Note

Phase 1 generates language-level deployment abstraction (TypeScript function calls vs REST HTTP calls).

Phase 2 will introduce platform-level deployment tooling (e.g. Google Service Weaver on GCP) that can manage the distributed routing at the infrastructure level. When Phase 2 is implemented, the smart call wrapper pattern may be replaced or augmented by the platform's native service-to-service communication. The function export and REST server layers will remain unchanged.

Do not introduce any platform-specific deployment code in Phase 1 modules.
