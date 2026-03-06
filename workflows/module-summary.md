# Workflow: module-summary

**Trigger:** `/module-summary`
**Phase:** 5 close — Per module
**Run by:** Human (Developer)
**Prerequisites:**
- BMAD has finished implementing the module
- The module's Product Brief exists in `global-picture/features/{feature-name}/`
- The module's code exists in the repository

---

## Purpose

Validate a completed module against its Product Brief, verify standards compliance and deployment abstraction correctness, generate a summary document, present it for human approval, and — on approval — formally mark the module complete, update all tracking files, and unblock dependent modules.

---

## Step 1 — Identify the Module

Ask the human:
> "Which module is complete and ready for summary? Provide the module name and feature name."

Locate:
- Product Brief: `global-picture/features/{feature-name}/product-brief-{module-name}.md`
- Module code: `modules/{module-name}/` (all modules live here — established by `/setup-monorepo`)
- Standards: `global-picture/standards/oss-packages.md`
- OSS Registry: `global-picture/oss-registry/registry.json`

Verify that `modules/{module-name}/` directory exists. If it does not:
> "Cannot find `modules/{module-name}/`. The module directory does not exist — BMAD may not have created it yet, or it was created at a different path. Confirm the module is fully implemented before running `/module-summary`."
Stop.

If the Product Brief doesn't exist:
> "No Product Brief found for `{module-name}`. A module must be initiated via `/initiate-module` before it can be summarised. Was this module built without going through Milo's initiate-module workflow?"

Surface this as a gap — do not proceed without a Product Brief to validate against.

---

## Step 2 — Scan the Module Code

Read the module's code directory. Specifically look for:

**2a. Entry point / index file** (`index.ts` or equivalent)
- What functions are exported?
- Do the exported function signatures match the Product Brief exactly?
  - Function name: exact match?
  - Parameter names and types: exact match?
  - Return type: exact match?

**2b. Server file** (`server.ts` or equivalent) — Standard API modules only
- Does a REST server file exist?
- Does it expose every exported function as a POST endpoint?
- Are the endpoint paths in the format `/api/{module-name}/{functionName}`?
- Does each handler call the corresponding function directly (no duplicate logic)?

**2c. Proxy / Smart Call Wrapper** (`proxy.ts` or equivalent) — Standard API modules only
- Does a proxy file exist?
- Does it check `DEPLOYMENT_MODE`?
- In monolith mode: does it call functions directly?
- In distributed mode: does it call the REST endpoint?
- Is the proxy what other modules import (not the index directly)?

**2d. Test files**
- Do tests exist for exported functions?
- What coverage is provided?

**2e. Package dependencies**
- Scan `package.json` (or equivalent) in the module directory
- List every runtime dependency declared

---

## Step 3 — Validate Against Product Brief

For each item in the Product Brief, check the corresponding code. Record findings.

### 3a. API Contract Validation

Compare each function in the Product Brief's Section 2 against the actual exports:

| Function | In Brief | In Code | Signature Match | REST Endpoint Exists |
|----------|----------|---------|-----------------|----------------------|
| {functionName} | ✓ | ✓/✗ | ✓/✗ | ✓/✗ |

Flag any:
- Functions in the brief that are missing from the code
- Functions in the code that are not in the brief (scope creep)
- Signature mismatches (different parameter types or return types)
- Missing REST endpoints

### 3b. Dependency Usage Validation

For each lower-level module listed in the Product Brief's Section 3:
- Is the module actually imported in the code?
- Is it imported via the proxy (correct) or directly from the module index (incorrect)?
- Are all usages calling functions that exist in `global-picture/api-docs/{dependency-name}.md`?

### 3c. OSS Package Validation

Compare the module's declared dependencies against `global-picture/standards/oss-packages.md`:

| Package Used | In Standards | Category | Version Match |
|-------------|-------------|----------|---------------|
| {package} | ✓/✗ | {category} | ✓/✗ |

Flag any:
- Non-standard packages (not in `oss-packages.md`)
- Version mismatches from the standard
- Missing standard packages that should have been used

### 3d. Deployment Abstraction Validation — Standard API Modules Only

Confirm all three layers are present:

```
Function Exports:   ✓/✗  — {notes}
REST Server:        ✓/✗  — {notes}
Smart Call Wrapper: ✓/✗  — {notes}
DEPLOYMENT_MODE check: ✓/✗ — {notes}
```

If any layer is missing or incorrect, this is a **blocking issue**. The module cannot be marked complete until deployment abstraction is correctly implemented.

---

## Step 4 — Build the Summary Document

Create: `global-picture/features/{feature-name}/module-summary-{module-name}.md`

Use `milo/templates/module-summary.md` as the structure.

The summary document contains:

**Header**
```
Module: {module-name}
Type: Standard API Module | Web UI | Mobile App | UI Component Library
Feature: {feature-name}
Summary date: {date}
Status: PENDING APPROVAL
```

**What Was Built**
One paragraph describing the module's bounded context and what was implemented. Use domain language, not technical language.

**API Exports** (Standard API modules)
Complete list of exported functions with final signatures (as implemented, not as planned):
```typescript
export async function {functionName}({params}): Promise<{ReturnType}>
```

Generated REST endpoints:
```
POST /api/{module-name}/{functionName}
Request: { {param}: {type} }
Response: { {field}: {type} }
```

**Dependencies Used**
Which lower-level modules were called and which functions were used.

**OSS Packages**
Final list of packages used, confirmed against standards.

**Deployment Abstraction**
Confirmation that all three layers (function exports, REST server, smart call wrapper) are implemented.

**Test Coverage**
Summary of tests written: what is covered, what is not.

**Deviations from Product Brief**
Any differences between what was specified in the Product Brief and what was actually built, with justification for each deviation.

**Validation Results**
Results from Step 3 — all checks, their outcomes, any issues found and how they were resolved.

---

## Step 5 — [HUMAN GATE] Module Summary Review

Present the summary to the human:

```
## Module Summary — {module-name}

### Validation Results
API Contract:             {pass / {N} issues}
Dependency Usage:         {pass / {N} issues}
OSS Compliance:           {pass / {N} issues}
Deployment Abstraction:   {pass / not required / {N} issues}

### Blocking Issues (if any)
[List any issues that must be resolved before approval]

### Deviations
[List any deviations from the Product Brief]

### Summary Document
global-picture/features/{feature-name}/module-summary-{module-name}.md
```

If there are **blocking issues**: state them clearly. The module cannot be marked complete until they are resolved. Ask: "These issues must be resolved before this module can be marked complete. Return to BMAD to fix them, then re-run `/module-summary`."

If there are **no blocking issues**: ask: **"Review the module summary. Does the module meet all Product Brief requirements? Are standards followed? Is the API contract correct? Approve to mark the module complete and unblock dependent modules."**

Wait for explicit approval.

---

## Step 6 — On Approval: Finalise the Module

### 6a. Update Module Status

In the Solution Architecture document (`global-picture/features/{feature-name}/solution-architecture.md`), update this module's status to `complete`.

### 6b. Update API Documentation

Create or update: `global-picture/api-docs/{module-name}.md`

This file is the authoritative API contract used by future modules that depend on this one. It must contain:

```markdown
# Module: {module-name}

**Type:** Standard API Module
**Status:** Complete
**Feature:** {feature-name}
**Last updated:** {date}

## Bounded Context

{One paragraph — what this module owns and what it does NOT own}

## Exported Functions

### {functionName}

**Signature:**
```typescript
export async function {functionName}({params}): Promise<{ReturnType}>
```

**Purpose:** {one sentence}
**Parameters:**
- `{param}`: `{type}` — {description}
**Returns:** `{ReturnType}` — {description}
**Throws:** `{ErrorType}` — {when}

**REST Endpoint:**
```
POST /api/{module-name}/{functionName}
Body: { {param}: {type} }
Response: { {field}: {type} }
```

**Import (via proxy):**
```typescript
import { {functionName} } from '@modules/{module-name}';
```
```

Repeat for every exported function.

### 6c. Update OSS Registry

In `global-picture/oss-registry/registry.json`, add this module's name to the `modules_using` array for every OSS package it uses.

### 6d. Update Features Registry

In `global-picture/features/registry.md`, update the feature row if all modules for this feature are now `complete` — change feature status to `complete`.

### 6e. Update Summary Document Status

In `global-picture/features/{feature-name}/module-summary-{module-name}.md`, change `Status: PENDING APPROVAL` to `Status: APPROVED — {date}`.

---

## Step 7 — Confirm and Report Unblocked Modules

After all updates:

```
## Module Complete — {module-name}

Updated files:
- global-picture/features/{feature-name}/solution-architecture.md  (module → complete)
- global-picture/api-docs/{module-name}.md  (created/updated)
- global-picture/oss-registry/registry.json  (packages updated)
- global-picture/features/{feature-name}/module-summary-{module-name}.md  (approved)

Newly unblocked modules (now ready to develop):
{list of modules whose dependencies are now all complete, or "none"}

Feature status: {in-development / complete — all modules done}
```

If the feature is now complete:
> "All modules for feature `{feature-name}` are complete. The feature branch is ready for integration testing and merge to main."

If modules remain:
> "Run `/initiate-module` to continue with the next module."
