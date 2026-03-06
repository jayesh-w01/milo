# Workflow: solution-architecture

**Trigger:** `/solution-architecture`
**Phase:** 2 — Per feature
**Run by:** Human (Developer / Architect)
**Prerequisites:**
- `/define-standards` has been run — `global-picture/standards/` must exist
- A product brief or feature requirements document is available (human provides it)

---

## Purpose

Convert a feature's product brief into a comprehensive Solution Architecture using Domain-Driven Design. Identify module boundaries, classify module types, map inter-module dependencies, and produce the dependency graph that drives all subsequent module development.

This workflow runs **once per feature**, before any module development begins.

---

## Step 1 — Validate Prerequisites

Check that `global-picture/standards/` exists and contains `oss-packages.md` and `coding-standards.md`.

If missing:
> "The `global-picture/standards/` folder is missing or incomplete. Run `/define-standards` before running `/solution-architecture`."
Stop.

Check `global-picture/features/registry.md` — note which features already exist and their status.

Check `global-picture/api-docs/` — note which modules already exist and what APIs they expose. This is critical: existing modules may be reused in this feature's architecture.

---

## Step 2 — Gather the Product Brief

Ask the human:
> "Provide the product brief or feature requirements for this feature. This can be:
> - A written description of the feature and its goals
> - User stories
> - A business requirements document
> - Or describe the feature now in plain language
>
> Also: what is the name of this feature? (used for the branch name: `feature/{name}`)"

Wait for the human to provide the feature requirements. Do not proceed until you have enough to understand:
- What the feature does for the end user
- What business capability it represents
- What success looks like

If the brief is too vague to decompose into modules, ask targeted clarifying questions before proceeding. Surface the specific ambiguity — don't ask open-ended "tell me more" questions.

---

## Step 3 — Analyse and Apply Domain-Driven Design

Read `milo/knowledge/ddd-principles.md` before this step.

Analyse the product brief through a DDD lens:

**3a. Identify Business Capabilities**
List all the distinct business capabilities the feature requires. A business capability is a thing the system must be able to *do*, not a technical component.

Examples: "authenticate a user", "process a payment", "manage a product catalogue", "send notifications"

**3b. Group Into Bounded Contexts**
Group related capabilities into bounded contexts. A bounded context is a domain area with:
- Its own ubiquitous language (the words used inside it mean specific things)
- Clear ownership boundaries
- Minimal coupling to other contexts

Rules for bounded contexts:
- One context should not reach into another context's data directly
- If two capabilities share the same core entities and language, they belong in the same context
- If two capabilities have different meanings for the same word (e.g. "customer" means different things in billing vs. support), they are separate contexts

**3c. Map to Modules**
Each bounded context becomes a module. Name modules after their domain (not their technology):
- `auth` not `jwt-service`
- `orders` not `order-processor`
- `catalog` not `product-database`

**3d. Identify Special Modules**
Flag any modules that are:
- **Web UI** — frontend application (consumes APIs, exposes no API of its own)
- **Mobile App** — mobile application (consumes APIs, exposes no API of its own)
- **UI Component Library** — shared component library (exports components, no REST API)

All other modules are **Standard API Modules** — they expose function exports and REST endpoints.

**3e. Identify External Services**
List any third-party services or external APIs the feature needs (Stripe, SendGrid, Firebase Auth, etc.). These are not modules — they are external dependencies a module integrates with.

---

## Step 4 — Scan Existing Modules

Check `global-picture/api-docs/` for every existing module.

For each existing module, determine:
- Does this feature use capabilities already provided by this module?
- Does this feature require *new* capabilities from this module?
- Is there a clean API already available, or does the module need extension?

Classify each module as:
- **Existing — reuse as-is** (use the existing API contract without changes)
- **Existing — needs extension** (new functions needed — see below)
- **New** (must be built as part of this feature)

**Handling "Existing — needs extension" modules:**

If an existing module needs new functions, those new functions become part of this feature's scope. Treat the extension as an additional work item:
1. Note the module and the required new functions in the Solution Architecture document under **Module Requirements**.
2. When generating the dependency graph, treat the extended module as a dependency that must be updated before dependent modules begin.
3. During `/initiate-module`, the extended module will be initiated first — its Product Brief will include both existing functions (for context) and the new functions to be added by BMAD.
4. Run `/module-summary` for the extended module after BMAD adds the new functions, updating its `global-picture/api-docs/` entry before dependents are unblocked.

> **Note:** Do not attempt to define new function signatures for an existing module here. Note the *capability needed* and let `/initiate-module` produce the precise API contract.

---

## Step 5 — Define Module Interactions

Map how modules interact with each other to deliver the feature.

**Rules:**
- All interactions are written as **function calls** — always. No HTTP, no REST, no message queues at this level. Deployment routing is handled by the smart call wrapper.
- Identify which module calls which, and what data flows between them.
- Draw the interaction flow: User → Web UI → Module A calls Module B calls Module C

For each interaction between two modules, note:
- Caller module
- Called module
- Function name (approximate — will be finalised in initiate-module)
- Data passed in / returned

---

## Step 6 — Build the Dependency Graph

From the module interactions, derive the development dependency order.

A module is **ready to develop** when all modules it depends on are marked complete.

Rules:
- A module that calls no other internal module has no dependencies — it can be developed first
- A module that calls other modules can only be developed after those modules are complete
- Circular dependencies indicate a module boundary problem — surface this to the human and propose a resolution before proceeding

Represent the graph as:

```
Module: {name}
  Type: Standard API | Web UI | Mobile App | UI Component Library
  Depends on: [{module-a}, {module-b}] or "none"
  Depended on by: [{module-c}] or "none"
  Status: not-started
```

Identify the **first tier** — modules with no dependencies that can be developed immediately.

---

## Step 7 — Generate the Solution Architecture Document

Create the file: `global-picture/features/{feature-name}/solution-architecture.md`

Use the template at `milo/templates/solution-architecture.md`.

The document must contain:
1. **Feature overview** — what the feature does, business objective
2. **Module inventory** — all modules (existing and new), type, and status
3. **Interaction flow** — how modules communicate (written as function calls)
4. **Requirements per module** — what each module must do for this feature
5. **Dependency graph** — development order, first tier highlighted
6. **External services** — third-party integrations and which modules own them
7. **Open questions** — any ambiguities that need resolution before development starts

Also create: `global-picture/features/{feature-name}/feature.md`

```markdown
# Feature: {feature-name}

**Status:** in-development
**Branch:** feature/{feature-name}
**Created:** {date}
**Modules:** {comma-separated list}

## Description
{one paragraph summary of the feature}

## Success Criteria
{what done looks like}
```

Update `global-picture/features/registry.md` — add the feature with status `in-development`.

---

## Step 8 — [HUMAN GATE] Solution Architecture Review

Present the solution architecture to the human for review. Structure your presentation:

```
## Solution Architecture — {feature-name}

### Module Inventory ({count} modules)

**New modules to build:**
| Module | Type | Depends On |
|--------|------|-----------|
| {name} | Standard API | {deps} |
...

**Existing modules being used:**
| Module | Change Required |
|--------|----------------|
| {name} | Reuse as-is / Needs extension |
...

**External services:**
| Service | Owned by Module |
|---------|----------------|
...

### Development Order
Tier 1 (no dependencies — start immediately): {modules}
Tier 2 (depends on Tier 1): {modules}
Tier 3 (depends on Tier 2): {modules}
...

### Interaction Flow
[Describe the flow in plain language + function call pseudocode]

### Open Questions
[List any ambiguities, risks, or decisions needed]
```

Ask: **"Review this solution architecture. Validate that: module boundaries are appropriate (DDD), dependencies are correctly identified, the flow makes sense, all new modules are justified. Do you approve, or are there revisions needed?"**

Wait for explicit approval. If revisions are requested, update the solution architecture document and re-present at this gate.

---

## Step 9 — On Approval

Confirm the feature is registered: "Feature `{feature-name}` registered in `global-picture/features/registry.md` with status `in-development`."

Instruct the human: "Next step: run `/initiate-module` to begin module development. Milo will select the first unblocked module from the dependency graph."
