# Workflow: initiate-module

**Trigger:** `/initiate-module`
**Phase:** 4 — Per module (repeats until all modules complete)
**Run by:** Human (Developer)
**Prerequisites:**
- `/solution-architecture` has been run and approved for this feature
- Solution Architecture document exists in `global-picture/features/{feature-name}/`
- At least one module in the dependency graph has status `not-started` with all its dependencies `complete`

---

## Purpose

Select the next module to develop based on the dependency graph and generate its complete Product Brief. The Product Brief is the handoff document to BMAD — it must be so complete and explicit that BMAD can build the module with zero ambiguity about boundaries, dependencies, standards, or API contracts.

---

## Step 1 — Validate Prerequisites

Check that a Solution Architecture document exists. If multiple features are `in-development`, ask the human which feature they are working on.

Read the complete Solution Architecture document for the selected feature.

Read `global-picture/standards/oss-packages.md` — you will need this for the Product Brief.

Read `global-picture/api-docs/` — collect the full API contracts of every module that has status `complete`. These are the function signatures available to the module being initiated.

---

## Step 2 — Analyse the Dependency Graph

From the Solution Architecture document, extract the current state of all modules:

For each module, determine:
- Its status: `not-started`, `in-development`, or `complete`
- Its dependencies (other modules it calls)
- Whether all its dependencies are `complete`

A module is **unblocked** if:
- Its status is `not-started`
- All modules it depends on are `complete`

List all unblocked modules.

If there are **no unblocked modules**:
> "No modules are currently unblocked. All `not-started` modules have at least one dependency that is not yet `complete`. Check the status of in-development modules and run `/module-summary` to mark completed modules before initiating the next one."
Stop.

---

## Step 3 — Module Selection

If **one module is unblocked**: select it automatically. State: "Next module to develop: `{module-name}`. It has no pending dependencies."

If **multiple modules are unblocked**: present options to the human:

> "Multiple modules are ready to develop in parallel:
>
> | Module | Type | Dependencies |
> |--------|------|-------------|
> | {name} | {type} | {deps — all complete} |
> | {name} | {type} | {deps — all complete} |
>
> Which module do you want to initiate? (You can develop multiple in parallel on separate branches — initiate each one separately.)"

Wait for selection.

---

## Step 4 — Generate the Product Brief

This is the most critical output Milo produces. Use `milo/templates/product-brief.md` as the structure.

The Product Brief must contain all six of the following sections — if any is missing or vague, the brief is incomplete.

---

### Section 1: Module Identity

```
Module name: {name}
Type: Standard API Module | Web UI | Mobile App | UI Component Library
Feature: {feature-name}
Bounded context: {one sentence describing the domain this module owns}
What this module does NOT do: {explicit exclusions — this prevents scope creep}
```

The bounded context description should use the ubiquitous language of the domain — not technical terms. Example: "The Auth module is responsible for verifying user identity and issuing session tokens. It does NOT manage user profiles, roles, or permissions — those belong to the User module."

---

### Section 2: Functional Requirements

List every function this module must expose as part of its API contract. For each function:

```
Function: {functionName}
Purpose: {one sentence}
Input: {parameter name}: {type} — {description}
        {parameter name}: {type} — {description}
Returns: Promise<{ReturnType}> — {description of what's returned}
Errors: {ErrorType} — {when this error is thrown}
```

These function signatures ARE the API contract. They drive both the TypeScript exports and the generated REST endpoints. Be precise about types.

Also list any **internal functions** the module needs — functions that are not exported but are required for the implementation. These are implementation details, not contract items.

If the module type is **Web UI or Mobile App**: skip this section (consumers have no API contract). List instead the pages/screens required and the API modules they consume.

If the module type is **UI Component Library**: list the components to export with their prop interfaces.

---

### Section 3: Lower-Level Module Dependencies

List every internal module this module will call, with the exact function signatures available.

For each dependency, copy the function signatures directly from `global-picture/api-docs/{module-name}.md`:

```
## Dependency: {module-name}

Import path: @modules/{module-name}

Available functions:
- {functionSignature}: {description}
- {functionSignature}: {description}

Functions used by this module:
- {functionName}({params}): used for {purpose}
```

If a dependency module is not yet complete (no api-docs entry), this module cannot be initiated. Surface this gap explicitly.

---

### Section 4: OSS Packages

List every OSS package this module will use, sourced from `global-picture/standards/oss-packages.md`.

```
| Package | Version | Category | Import |
|---------|---------|----------|--------|
| {name} | {version} | {category} | import ... from '{package}' |
```

State explicitly: "No packages outside this list may be added without a standards amendment."

If the module requires a package not in the standards (e.g. a highly specific third-party SDK), flag it:
> "Non-standard package required: `{package}`. Reason: {justification}. This must be approved before development starts."

Then add it as an open question in the brief for human decision.

---

### Section 5: Deployment Abstraction Requirements

Read `milo/knowledge/deployment-abstraction.md` before writing this section.

For **Standard API Modules**, this section is mandatory and non-negotiable. BMAD must generate:

**5a. Function Exports**
The module's primary interface. Every function listed in Section 2 must be exported from `{module-name}/index.ts` (or equivalent entry point for the project's language).

**5b. REST API Server**
A server file (`{module-name}/server.ts` or equivalent) that exposes every exported function as a REST endpoint.

Endpoint pattern:
```
POST /api/{module-name}/{functionName}
Request body: { ...functionParams }
Response: { ...returnValue }
```

The REST handler must call the function directly — no duplicate logic.

**5c. Smart Call Wrapper**
A proxy/wrapper (`{module-name}/proxy.ts` or equivalent) that intercepts calls to this module's functions and routes them based on `DEPLOYMENT_MODE`:
- `DEPLOYMENT_MODE=monolith` → direct function call
- `DEPLOYMENT_MODE=distributed` → REST call to the module's endpoint

Any module that imports from `@modules/{this-module-name}` imports from the proxy, not the index directly.

Include code skeleton examples for all three (adapted to the project's language and framework from `milo/knowledge/deployment-abstraction.md`).

For **Web UI / Mobile App**: state "No deployment abstraction required — consumer only."
For **UI Component Library**: state "No REST API generated — component exports only."

---

### Section 6: Success Criteria

List the conditions that define "done" for this module. Be concrete — not "works correctly" but checkable outcomes:

```
1. All functions in Section 2 are implemented and exported
2. Every exported function has a corresponding REST endpoint in server.ts
3. The smart call wrapper correctly routes based on DEPLOYMENT_MODE
4. All OSS packages used match the standards in Section 4
5. Unit tests exist for all exported functions
6. API documentation updated in global-picture/api-docs/{module-name}.md
7. [Feature-specific criteria]
```

---

## Step 5 — Write the Product Brief File

Create the file: `global-picture/features/{feature-name}/product-brief-{module-name}.md`

Populate it using the six sections above. Do not use vague placeholders — every field must have real content.

---

## Step 6 — Mark Module as In-Development

Update the Solution Architecture document — change the module's status from `not-started` to `in-development`.

---

## Step 7 — [HUMAN GATE] Product Brief Review

Present a summary:

```
## Product Brief — {module-name}

Type: {type}
Feature: {feature-name}

Exported functions: {count}
Dependencies on other modules: {list}
OSS packages: {count}
Deployment abstraction: {required / not required}

File: global-picture/features/{feature-name}/product-brief-{module-name}.md
```

Ask: **"Review the Product Brief for `{module-name}`. This is the document BMAD will use to build the module. Confirm: requirements are complete, lower-level module references are correct, OSS packages are right, API contracts are accurate. Approve or request revisions?"**

Wait for approval.

---

## Step 8 — On Approval

Instruct the human:

> "Product Brief approved. Hand this to BMAD:
>
> 1. Open a new Claude Code session in this same project directory (or continue if BMAD is already active)
> 2. If BMAD is not yet installed, run: `npx bmad-method install --modules bmm --tools claude-code`
> 3. In the Claude Code session, say: `"I want to create architecture for a new module. The Product Brief is at global-picture/features/{feature-name}/product-brief-{module-name}.md — read it first, then let's create the architecture."`
> 4. BMAD will run its `create-architecture` workflow. From there it proceeds: Architecture → Epics & Stories → Code
> 5. BMAD outputs go to `_bmad-output/` — check there for generated documents
>
> When BMAD finishes the module and the code is in `modules/{module-name}/`, return here and run `/module-summary`."
