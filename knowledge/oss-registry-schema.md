# OSS Registry Schema
## Reference for Milo's define-standards and module-summary workflows

---

## Purpose

The OSS registry tracks every open-source package used across the project — which packages are standard, which modules use them, and their versions. It is the authoritative source for OSS consistency enforcement.

File location: `global-picture/oss-registry/registry.json`

---

## Schema

```json
{
  "version": "1.0",
  "project": "{project-name}",
  "last_updated": "2024-01-15T10:30:00Z",
  "packages": {
    "{package-name}": {
      "version": "{semver-range}",
      "category": "{category}",
      "standard": true,
      "description": "{one line — what this package does}",
      "modules_using": ["{module-name}", "{module-name}"],
      "added_in_workflow": "define-standards | module-summary",
      "notes": "{optional — any usage conventions}"
    }
  },
  "deviations": [
    {
      "module": "{module-name}",
      "package": "{package-name}",
      "version": "{version}",
      "reason": "{justification}",
      "approved_by": "{human-name}",
      "approved_date": "{ISO date}"
    }
  ]
}
```

---

## Field Definitions

### Package Object

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `version` | string | Yes | Semver range. Use `^` for minor-compatible ranges (e.g. `"^4.18.0"`). Use exact version only if required for stability. |
| `category` | string | Yes | See categories below. |
| `standard` | boolean | Yes | `true` = this is the approved standard package for this category. `false` = non-standard, permitted only where noted in `deviations`. |
| `description` | string | Yes | One sentence: what this package does. |
| `modules_using` | string[] | Yes | Module names that declare this package as a dependency. Updated by `/module-summary`. Initially `[]`. |
| `added_in_workflow` | string | Yes | Which workflow added this entry: `"define-standards"` or `"module-summary"`. |
| `notes` | string | No | Any project-specific usage convention for this package. |

### Deviations Array

Records any non-standard packages approved for use. Every non-standard package used in any module must have a deviations entry — no silent exceptions.

---

## Package Categories

Use these exact category strings in the `category` field:

| Category | Description | Examples |
|----------|-------------|---------|
| `http-server` | HTTP server / web framework | express, fastify, hono, koa |
| `http-client` | HTTP client for outbound requests | axios, node-fetch, got, ky |
| `orm` | Database ORM or query builder | prisma, typeorm, drizzle, knex |
| `db-client` | Raw database driver/client | pg, mysql2, mongodb, redis |
| `validation` | Input validation and schema parsing | zod, joi, yup, class-validator |
| `logging` | Structured logging | pino, winston, bunyan |
| `testing` | Test runner and assertion library | vitest, jest, mocha |
| `testing-http` | HTTP endpoint testing | supertest, @vitest/supertest |
| `auth` | Authentication and JWT handling | jsonwebtoken, jose, passport |
| `date-time` | Date and time manipulation | date-fns, dayjs, luxon |
| `env-config` | Environment variable loading | dotenv, @dotenvx/dotenvx, zod-env |
| `ui-framework` | Frontend UI framework | react, vue, svelte |
| `ui-meta-framework` | Frontend meta-framework | next, nuxt, sveltekit |
| `ui-components` | UI component library | radix-ui, shadcn/ui, mui |
| `ui-styling` | CSS / styling solution | tailwindcss, css-modules, styled-components |
| `state-management` | Frontend state management | zustand, jotai, redux-toolkit |
| `data-fetching` | Frontend data fetching / caching | tanstack-query, swr |
| `utility` | General utility functions | lodash, ramda |
| `crypto` | Cryptography and hashing | bcryptjs, argon2, crypto-js |
| `email` | Email sending | nodemailer, @sendgrid/mail, resend |
| `queue` | Job queues and background processing | bullmq, agenda |
| `sdk` | Third-party service SDK | stripe, @aws-sdk, firebase-admin |
| `other` | Packages that don't fit above | — |

---

## Example: Fully Populated Registry

```json
{
  "version": "1.0",
  "project": "acme-platform",
  "last_updated": "2024-01-15T14:22:00Z",
  "packages": {
    "fastify": {
      "version": "^4.26.0",
      "category": "http-server",
      "standard": true,
      "description": "Fast and low overhead web framework for Node.js",
      "modules_using": ["auth", "orders", "catalog"],
      "added_in_workflow": "define-standards",
      "notes": "Use the shared Fastify instance from lib/server.ts — never create new instances per module"
    },
    "zod": {
      "version": "^3.22.4",
      "category": "validation",
      "standard": true,
      "description": "TypeScript-first schema validation with static type inference",
      "modules_using": ["auth", "orders"],
      "added_in_workflow": "define-standards",
      "notes": "Define all input schemas in a schemas.ts file within each module"
    },
    "pino": {
      "version": "^8.19.0",
      "category": "logging",
      "standard": true,
      "description": "Very low overhead Node.js logger",
      "modules_using": ["auth"],
      "added_in_workflow": "define-standards",
      "notes": "Import the shared logger instance from lib/logger.ts"
    },
    "stripe": {
      "version": "^14.21.0",
      "category": "sdk",
      "standard": false,
      "description": "Stripe payment processing SDK",
      "modules_using": ["payments"],
      "added_in_workflow": "module-summary",
      "notes": "Only the payments module may use this SDK"
    }
  },
  "deviations": [
    {
      "module": "payments",
      "package": "stripe",
      "version": "^14.21.0",
      "reason": "Stripe SDK is required for payment processing and has no standard equivalent",
      "approved_by": "engineering-lead",
      "approved_date": "2024-01-15"
    }
  ]
}
```

---

## Workflow Operations

### define-standards writes:
- Initial registry with all standard packages
- `modules_using: []` for all (no modules exist yet)
- `standard: true` for all entries

### initiate-module reads:
- Full registry to determine which packages the module should use
- Passes the relevant packages to the Product Brief (Section 4)

### module-summary reads and writes:
- Reads: validates the module's actual `package.json` against the registry
- Writes: adds the module name to `modules_using` for each package it uses
- Writes: adds new non-standard package entries if approved
- Writes: adds deviation entries if applicable

---

## Enforcement Rules

1. **Every package used in a module must appear in the registry.** If it's not there, it hasn't been approved.

2. **Standard packages must be used.** If `zod` is the standard validation library, modules must use `zod` — not `joi`, not `yup`.

3. **Non-standard packages require a deviation entry.** If a module genuinely needs a package not in the standards (e.g. a third-party SDK), it must be flagged in the Product Brief, approved by a human, and recorded in `deviations[]`.

4. **Version ranges should be consistent.** If the standard says `"^4.18.0"` for a package, modules must not pin to `"4.18.3"` or upgrade to `"^5.0.0"` unilaterally. Version changes are a standards amendment.
