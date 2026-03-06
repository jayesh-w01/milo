# Workflow: define-standards

**Trigger:** `/define-standards`
**Phase:** 1 — One-time project setup
**Run by:** Human (Product/Engineering Lead)
**Prerequisites:** Fresh mono-repo with Milo and BMAD checked in. No `global-picture/` folder should exist yet.

---

## Purpose

Establish the organisation-wide standards that every module in this project will follow. Create the `global-picture/` infrastructure that all subsequent Milo and BMAD workflows depend on. This workflow runs **once per project** and produces the enterprise foundation.

---

## Step 1 — Check Prerequisites

Scan the project root for:
- An existing `global-picture/` folder

If `global-picture/` already exists:
> "A `global-picture/` folder already exists. Running `/define-standards` again will overwrite the established standards. This affects all existing modules and features. Confirm you want to proceed, or use this workflow to review existing standards only."

Wait for explicit confirmation before continuing.

If no `global-picture/` exists, proceed to Step 2.

---

## Step 2 — Create the global-picture/ Directory Structure

Create the following folder and file structure:

```
global-picture/
├── standards/
│   ├── coding-standards.md       ← filled in Step 4
│   ├── oss-packages.md           ← filled in Step 4
│   └── architectural-patterns.md ← filled in Step 4
├── api-docs/
│   └── README.md                 ← see template below
├── features/
│   ├── registry.md               ← see template below
│   └── README.md
└── oss-registry/
    └── registry.json             ← initialized in Step 5
```

**`global-picture/api-docs/README.md`** content:
```markdown
# API Documentation

This folder contains function signature documentation for every completed module.
One file per module: `{module-name}.md`

Files are created and updated by the `/module-summary` workflow when a module is marked complete.
Do not edit manually.
```

**`global-picture/features/README.md`** content:
```markdown
# Features Registry

`registry.md` tracks all features across the project.
Each feature also has its own subfolder: `{feature-name}/`
containing the feature details and solution architecture document.

Status values: planned | in-development | complete
```

**`global-picture/features/registry.md`** content:
```markdown
# Features Registry

| Feature | Status | Modules Involved | Branch | Last Updated |
|---------|--------|-----------------|--------|--------------|
```

Confirm to the human: "Created `global-picture/` directory structure. Proceeding to gather organisational standards."

---

## Step 3 — Gather Organisational Standards (Interactive)

Ask the human the following questions. Group them into logical sets — do not fire one question at a time. Present each group, collect answers, then move to the next group.

---

### Group A — Language & Runtime

Ask all at once:

> "Let's establish your technology foundation. Please answer these:
>
> 1. **Primary programming language** — what language will all backend modules use? (TypeScript / Python / Go / Java / other)
> 2. **Runtime** (if TypeScript/JS) — Node.js, Deno, or Bun?
> 3. **Package manager** — npm, yarn, pnpm, bun?
> 4. **TypeScript strictness** (if applicable) — strict mode on or off? eslint config (standard / airbnb / custom)?"

---

### Group B — Backend OSS Packages

Ask all at once:

> "Now the OSS packages your backend modules will standardise on. Pick one for each — these will be enforced across all modules:
>
> 1. **HTTP server framework** — Express, Fastify, Hono, Koa, or other?
> 2. **HTTP client** — axios, node-fetch, got, ky, or native fetch?
> 3. **Database ORM/client** — Prisma, TypeORM, Drizzle, Knex, or other? (or N/A if no shared DB)
> 4. **Validation library** — Zod, Joi, Yup, class-validator, or other?
> 5. **Logging library** — Pino, Winston, Bunyan, or other?
> 6. **Testing framework** — Vitest, Jest, or other?
> 7. **Date/time library** — date-fns, dayjs, Luxon, or native Date?
> 8. **Auth/JWT library** — jsonwebtoken, jose, or other? (or N/A)"

---

### Group C — Frontend OSS Packages (skip if no UI modules)

> "If this project includes Web UI or Mobile App modules:
>
> 1. **UI framework** — React, Vue, Svelte, or other?
> 2. **Meta-framework** — Next.js, Nuxt, SvelteKit, Vite (SPA), or other?
> 3. **UI component library** — Radix UI, shadcn/ui, MUI, Ant Design, or custom?
> 4. **Styling** — Tailwind CSS, CSS Modules, styled-components, or other?
> 5. **State management** — Zustand, Jotai, Redux Toolkit, TanStack Query, or other?"

---

### Group D — Coding & Architecture Standards

> "Finally, a few architecture and style choices:
>
> 1. **Code style** — tabs or spaces? Spaces: 2 or 4? Semicolons: yes or no?
> 2. **Module internal structure** — do you want a standard internal layer structure? (e.g. controller / service / repository, or domain / application / infrastructure)
> 3. **Error handling convention** — throw exceptions, or return Result/Either types?
> 4. **Testing approach** — unit tests only, unit + integration, or full pyramid (unit + integration + e2e)?
> 5. **API versioning** — will REST endpoints be versioned? (e.g. `/api/v1/...`)"

---

## Step 4 — Generate Standards Documents

Using the human's answers, create the three standards documents. Use the template at `milo/templates/standards-doc.md` as the base.

### `global-picture/standards/oss-packages.md`

Document every OSS package choice with:
- Package name and version (use the latest stable version at time of writing if human didn't specify)
- Category (http-server, http-client, orm, validation, logging, testing, etc.)
- Why it was chosen (one line)
- Usage note (one line — how to import/use consistently)

Format:
```markdown
# OSS Package Standards

Last updated: {date}
Established by: define-standards workflow

These packages are mandatory across all modules. Using a different package for any of these
categories requires an explicit standards amendment — raise it before coding, not after.

## Backend Packages

| Category | Package | Version | Notes |
|----------|---------|---------|-------|
| HTTP Server | {package} | {version} | {note} |
| HTTP Client | {package} | {version} | {note} |
...

## Frontend Packages (if applicable)

| Category | Package | Version | Notes |
|----------|---------|---------|-------|
...
```

### `global-picture/standards/coding-standards.md`

Document all coding style and convention decisions. Include:
- Language version and compiler/transpiler settings
- Formatting rules (indent, semicolons, quotes, line length)
- Naming conventions (files, functions, types, constants)
- Module internal structure (layer pattern)
- Error handling convention
- Import ordering convention
- Testing standards (coverage expectations, test file naming)

### `global-picture/standards/architectural-patterns.md`

Document architecture-level decisions:
- Module structure (how a standard API module is organised internally)
- The deployment abstraction pattern — function exports + REST wrapper + smart call wrapper (reference `milo/knowledge/deployment-abstraction.md` for the pattern detail)
- API versioning approach
- Inter-module communication rule (always function calls in code — never direct HTTP)
- Event-driven patterns (if applicable)

---

## Step 5 — Initialize OSS Package Registry

Create `global-picture/oss-registry/registry.json` with every package from Step 4 pre-populated.

Use the schema defined in `milo/knowledge/oss-registry-schema.md`.

Initial state — all packages listed with `modules_using: []` since no modules exist yet.

Example structure:
```json
{
  "version": "1.0",
  "last_updated": "{ISO date}",
  "packages": {
    "fastify": {
      "version": "^4.0.0",
      "category": "http-server",
      "standard": true,
      "modules_using": []
    },
    "zod": {
      "version": "^3.22.0",
      "category": "validation",
      "standard": true,
      "modules_using": []
    }
  }
}
```

---

## Step 6 — [HUMAN GATE] Review and Approve

Present a clean summary to the human:

```
## define-standards — Output Summary

### Created Structure
- global-picture/standards/coding-standards.md
- global-picture/standards/oss-packages.md
- global-picture/standards/architectural-patterns.md
- global-picture/api-docs/README.md
- global-picture/features/registry.md
- global-picture/oss-registry/registry.json

### Standards Established
[List the key decisions made: language, runtime, key packages, style choices]

### OSS Packages Registered: {count}

### Next Step
Run /solution-architecture with your first feature's product brief.
```

Ask: **"Review the standards documents. Once approved, these govern all module development in this project. Do you approve, or are there corrections needed?"**

Wait for explicit approval.

---

## Step 7 — On Approval

Confirm: "Standards established. `global-picture/` is ready. The project's enterprise foundation is set."

Remind the human: "Next step: run `/solution-architecture` with your feature's product brief."
