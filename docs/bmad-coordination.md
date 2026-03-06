# Milo and BMAD — How They Work Together

Milo and BMAD are two separate tools that work in sequence on every module. Understanding their distinct roles — and exactly how you hand off between them — is essential for using Milo correctly.

---

## What Each Tool Does

### BMAD

BMAD (Business Minded Agile Development) is an agentic development framework that builds software modules. It takes a specification, breaks it into architecture decisions and stories, and implements the code. BMAD is excellent at building — it follows a structured process and produces working code.

What BMAD does not do is make enterprise-level decisions: it doesn't decide which modules to build, how to divide domain responsibilities, which OSS packages are approved across the project, or how modules should communicate in different deployment environments. That's where Milo comes in.

### Milo

Milo makes the decisions BMAD never asks about:
- **What** to build (module decomposition via DDD)
- **How** to build it (standards, patterns, deployment abstraction)
- **In what order** to build it (dependency graph)
- **Whether it was built correctly** (validation against the Product Brief)

Milo does not write application code. It writes planning and governance documents, and validates outputs.

---

## The Separation of Responsibilities

| Concern | Milo | BMAD |
|---------|------|------|
| Module decomposition | ✓ | — |
| Defining function signatures | ✓ | — |
| Coding standards and OSS choices | ✓ | — |
| Deployment abstraction pattern | ✓ specifies | ✓ implements |
| Writing business logic | — | ✓ |
| Writing tests | — | ✓ |
| Validating the output | ✓ | — |
| Updating project registries | ✓ | — |

---

## The Handoff Document: The Product Brief

The entire coordination between Milo and BMAD happens through one document: the **Product Brief**.

Milo generates the Product Brief during `/initiate-module`. It contains everything BMAD needs to build a module — and nothing is left vague:

- The module's exact bounded context (what it owns and what it does NOT own)
- Every function it must export, with exact signatures (parameter names, types, return types, errors)
- The exact function signatures available from every dependency (copied from `global-picture/api-docs/`)
- Which OSS packages to use (from the approved standards)
- Code skeletons for all three deployment abstraction layers
- Concrete success criteria (checkable conditions, not vague goals)

BMAD reads this document and builds exactly what it specifies. The Product Brief is the contract. BMAD does not improvise, extend scope, or make architectural decisions — it implements the specification.

---

## The Full Coordination Flow

```
[ Milo Session ]
      ↓
/solution-architecture → produces module list + dependency graph
      ↓
/initiate-module → selects next module → generates Product Brief
      ↓
Human reviews and approves Product Brief
      ↓
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
HANDOFF: Human opens BMAD session
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
      ↓
[ BMAD Session ]
      ↓
BMAD reads Product Brief from global-picture/features/{feature}/product-brief-{module}.md
      ↓
BMAD runs its architecture → stories → code workflow
      ↓
BMAD builds the module in modules/{module-name}/
      ↓
Module is complete
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
HANDOFF: Human returns to Milo session
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
      ↓
[ Milo Session ]
      ↓
/module-summary → validates the module against the Product Brief
      ↓
Human reviews validation results
      ↓
If blocking issues → back to BMAD to fix
If all pass → human approves
      ↓
Milo marks module complete, updates registries, unblocks dependents
      ↓
/initiate-module for next module
```

---

## Step-by-Step: The BMAD Handoff

### Step 1: Milo gives you explicit instructions after Product Brief approval

When you approve the Product Brief in `/initiate-module`, Milo tells you exactly what to do next:

> "Product Brief approved. Hand this to BMAD:
> 1. Open a new Claude Code session (or continue in BMAD's agent context)
> 2. Reference: `global-picture/features/{feature-name}/product-brief-{module-name}.md`
> 3. Run BMAD's `create-architecture` workflow using this Product Brief as input
> 4. BMAD will then proceed: Architecture → Stories → Code
>
> When BMAD finishes the module, return here and run `/module-summary`."

### Step 2: Open a BMAD session

Start a new Claude Code session, or open BMAD in whichever way your setup supports. The key is that this session runs in BMAD's context — with BMAD's `CLAUDE.md` active, not Milo's.

### Step 3: Reference the Product Brief

Tell BMAD where the specification is:

```
Build the module specified in global-picture/features/auth-feature/product-brief-auth.md
```

BMAD will read the Product Brief and proceed with its own workflow.

### Step 4: Let BMAD work

BMAD runs through its process. It may ask you questions — about implementation choices, specific behaviours, edge cases. Answer them. The key constraint: **BMAD must not deviate from the Product Brief's API contract**. If BMAD proposes a different function name or parameter type, push back and reference the brief.

### Step 5: Return to Milo when BMAD is done

When BMAD signals the module is complete, switch back to a Milo session (Claude Code with Milo's `CLAUDE.md` active) and run:

```
/module-summary
```

---

## The Validation Loop

Validation is where Milo enforces the contract. If BMAD's output doesn't match the Product Brief, Milo will block completion.

**Common blocking issues and what causes them:**

| Issue | Likely Cause | Fix |
|-------|-------------|-----|
| Function signature mismatch | BMAD renamed a parameter or changed a return type | Update the code to match the Product Brief exactly |
| Missing server.ts | BMAD skipped the REST server layer | BMAD must add it — refer back to Section 5 of the Product Brief |
| Missing proxy.ts | BMAD skipped the smart call wrapper | BMAD must add it — refer to `milo/knowledge/deployment-abstraction.md` for the pattern |
| Non-standard package used | BMAD chose a package outside the approved standards | Replace with the standard package, or flag for deviation approval and re-run |
| Direct import (not via proxy) | BMAD imported a dependency module directly from its index | Change to `@modules/{module-name}` which resolves to the proxy |

**How to send BMAD back:**

Return to the BMAD session and share the specific blocking issues Milo reported. Reference the Product Brief sections they correspond to. BMAD should fix only the flagged issues — not refactor the entire module.

When BMAD fixes the issues, run `/module-summary` again. It validates the full module each time.

---

## Running Modules in Parallel

If your dependency graph has multiple modules in the same tier with no dependencies on each other, they can be developed in parallel on separate branches.

**How to do it:**

1. In the Milo session, run `/initiate-module` and select module A
2. Approve the Product Brief for module A
3. Run `/initiate-module` again and select module B
4. Approve the Product Brief for module B
5. Open two BMAD sessions (or hand to two developers), one per module
6. Both modules are built in parallel
7. When each finishes, run `/module-summary` to validate and mark it complete

The important thing: run `/initiate-module` separately for each module. Don't try to batch them in one run.

---

## Two Claude Code Sessions or One?

In practice, Milo and BMAD both run in Claude Code. You have two options:

**Option A: Same session, different contexts**
Switch between Milo and BMAD by opening the relevant `CLAUDE.md`. Some teams do this in one terminal with the same working directory — just be clear about which mode you're in.

**Option B: Two terminals**
Keep a dedicated terminal for Milo and a dedicated terminal for BMAD. Cleaner context separation, harder to accidentally mix tools.

Either works. What matters is that when you're running a Milo command, Claude Code has Milo's `CLAUDE.md` active — and when you're running BMAD, BMAD's context is active.

---

## What BMAD Gets vs What Milo Validates

This table summarises what flows in each direction through the Product Brief:

**Milo gives BMAD:**
- Module identity and bounded context
- Exact function signatures (names, parameters, return types, errors)
- Exact dependency API contracts (what to call and what it returns)
- Approved OSS packages (what to use and how to import)
- Deployment abstraction code skeletons (what to implement)
- Success criteria (what done looks like)

**Milo validates from BMAD's output:**
- Are all specified functions implemented with correct signatures?
- Are there any extra functions not in the brief (scope creep)?
- Is every dependency imported via the proxy?
- Are all three abstraction layers implemented?
- Are OSS packages compliant with the standards?
- Do tests exist?
