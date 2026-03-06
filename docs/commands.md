# Commands Reference

Complete reference for all 5 Milo slash commands. Each entry covers: what it does, when to run it, prerequisites, what Claude asks you, what gets created, the human gate, and what comes next.

Run commands in order. Never skip a phase.

---

## `/setup-monorepo`

**Phase:** 0 — One-time project initialisation
**Run:** Once, ever, per project
**Who runs it:** Engineering or Product Lead

### Purpose

Bootstrap the mono-repo from scratch. Initialise git, create the standard directory structure, install BMAD, make the first commit, and optionally push to GitHub.

### Prerequisites

None. This is the very first command. Because `CLAUDE.md` doesn't exist yet, you can't run this as a slash command on your first session. Instead, say:

```
Read milo/workflows/setup-monorepo.md and run it.
```

After this runs and creates `CLAUDE.md`, all future sessions use slash commands normally.

### What Claude Asks You

One set of questions, all at once:

1. **Project name** — becomes the repo name and root README title
2. **Project description** — one sentence about what the product does
3. **BMAD** — already present in `bmad/`, at a local path, or clone from GitHub URL?
4. **GitHub** — create a new repo, push to an existing remote, or skip for now?

### What Gets Created

- `modules/` — empty directory for future module code
- `.gitignore` — standard ignore patterns
- `README.md` — project root readme with structure overview
- `CLAUDE.md` — the entry point that registers all Milo slash commands with Claude Code
- `bmad/` — BMAD installed here (cloned or copied based on your answer)
- Initial git commit

If you requested GitHub:
- GitHub repository created via `gh` CLI
- Initial commit pushed to `origin/main`

### Human Gate

Claude presents a final summary showing all created items, git status, and GitHub URL (if applicable). Asks for confirmation before completing.

### What Comes Next

Run `/define-standards`.

---

## `/define-standards`

**Phase:** 1 — One-time project setup
**Run:** Once, ever, per project (after `/setup-monorepo`)
**Who runs it:** Engineering or Product Lead

### Purpose

Establish the organisation-wide standards that every module in this project will follow. Creates the `global-picture/` infrastructure. This is the governance foundation — once approved, these standards apply to all modules without exception.

### Prerequisites

- `/setup-monorepo` has been run
- `CLAUDE.md` exists at the project root
- No `global-picture/` folder exists yet (or you've confirmed you want to overwrite)

### What Claude Asks You

Four grouped sets of questions. Claude presents each group and waits for your answers before moving to the next. Do not expect one-at-a-time questions.

**Group A — Language & Runtime:**
Primary language, runtime (if JS/TS), package manager, TypeScript strictness, ESLint config

**Group B — Backend OSS Packages:**
HTTP server framework, HTTP client, database ORM/client, validation library, logging library, testing framework, date/time library, auth/JWT library

**Group C — Frontend OSS Packages** (skip if no UI modules in the project):
UI framework, meta-framework, component library, styling approach, state management

**Group D — Coding & Architecture:**
Indentation, semicolons, module internal layer structure, error handling convention, testing approach, API versioning strategy

**Tip:** Prepare your answers to Group B especially. These are the most consequential choices — they apply to every module built after this.

### What Gets Created

**Directory structure:**
```
global-picture/
├── standards/
│   ├── coding-standards.md       ← formatting, naming, error handling
│   ├── oss-packages.md           ← approved packages with versions and usage notes
│   └── architectural-patterns.md ← module structure and deployment abstraction rules
├── api-docs/
│   └── README.md
├── features/
│   ├── registry.md
│   └── README.md
└── oss-registry/
    └── registry.json             ← all approved packages, modules_using: []
```

### Human Gate

Claude presents a summary of all key decisions made and asks:

> "Review the standards documents. Once approved, these govern all module development in this project. Do you approve, or are there corrections needed?"

Review the generated documents carefully. This is the last point where changes are free — amendments after this point require a formal standards update.

### What Comes Next

Run `/solution-architecture` with your first feature's product brief.

---

## `/solution-architecture`

**Phase:** 2 — Per feature
**Run:** Once per feature, before any module development for that feature
**Who runs it:** Engineering Lead or Senior Developer

### Purpose

Take a product brief (written in plain English) and apply Domain-Driven Design to decompose the feature into a set of modules with clear boundaries, defined interactions, and a dependency graph. The output tells you exactly what to build and in what order.

### Prerequisites

- `/define-standards` has been run and approved
- `global-picture/standards/` exists with all three standards documents
- You have a product brief for the feature (can be a document, a description, a set of requirements — whatever level of detail you have)

### What You Provide

Paste or describe your product brief when running the command. It can be:
- A detailed product requirements document
- A one-page feature description
- A set of user stories
- A rough feature outline

Milo will ask clarifying questions if anything is ambiguous. The more context you provide upfront, the fewer questions.

### What Claude Does

1. Reads your standards to understand the technical constraints
2. Scans `global-picture/api-docs/` for existing modules that might be reused
3. Applies event storming to your brief — lists all domain events
4. Groups events by domain, identifies commands, proposes module boundaries
5. Classifies each module (Standard API, Web UI, Mobile App, Component Library)
6. Identifies external services (Stripe, SendGrid, etc.) — these are not modules
7. Maps interactions: who calls who, what data flows
8. Builds the dependency graph with tiers
9. Notes any existing modules that need extension (and what's needed)
10. Generates the Solution Architecture document

Claude may ask clarifying questions during this process — particularly around boundary ambiguities or whether something should be one module or two.

### What Gets Created

```
global-picture/features/{feature-name}/
├── solution-architecture.md      ← the authoritative architecture document
```

The solution architecture document contains:
- Feature overview and scope
- Module inventory (new, existing-reuse, existing-needs-extension, external)
- Module interactions and data flow
- Module requirements (one section per module)
- Dependency graph with tier structure
- Open questions (if any remain)
- Architecture decisions and their rationale

### Human Gate

Claude presents the architecture summary and asks:

> "Review the solution architecture. Does the module decomposition make sense? Are the boundaries correct? Is anything missing from scope? Approve or request revisions."

This is the most important gate in Milo. Wrong boundaries here propagate through every module. Take the time to review:
- Is each module's bounded context clearly described?
- Does the "what this module does NOT do" section accurately exclude adjacent concerns?
- Does the dependency graph make sense? (Lower-level services at the top, UI at the bottom)
- Are any modules suspiciously large (might need splitting) or suspiciously small (might need merging)?

### What Comes Next

Run `/initiate-module` to start building the first module.

---

## `/initiate-module`

**Phase:** 4 — Per module, repeats until all modules complete
**Run:** Before each module, in dependency order
**Who runs it:** Developer (or Engineering Lead)

### Purpose

Select the next module to develop and generate its complete Product Brief. The Product Brief is the handoff document to BMAD — it must be so complete that BMAD can build the module with zero ambiguity.

### Prerequisites

- `/solution-architecture` has been run and approved for the relevant feature
- At least one module has status `not-started` with all its dependencies `complete`
- The `global-picture/api-docs/` entry exists for every dependency of the chosen module (this is enforced automatically — if a dependency isn't done, you can't initiate the dependent module)

### What Claude Does

1. Reads the Solution Architecture and extracts all module statuses
2. Determines which modules are unblocked (status: `not-started`, all dependencies: `complete`)
3. If one option: selects it automatically
4. If multiple options: presents the list and asks you to choose (multiple can run in parallel on separate branches — initiate each separately)
5. Reads the complete API contracts from `global-picture/api-docs/` for every dependency
6. Reads `global-picture/standards/oss-packages.md` for the approved package list
7. Generates the Product Brief with all 6 mandatory sections

### The 6 Sections of a Product Brief

**Section 1: Module Identity**
- Module name, type, feature
- Bounded context (one sentence in domain language)
- Explicit statement of what this module does NOT do

**Section 2: Functional Requirements**
- Every function the module must export, with exact signatures:
  - Function name
  - Parameter names and types
  - Return type (`Promise<T>`)
  - Errors thrown and when

**Section 3: Lower-Level Module Dependencies**
- Every internal module this module will call
- The exact function signatures available (copied from `global-picture/api-docs/`)
- Which functions will be used and for what purpose

**Section 4: OSS Packages**
- Every package this module will use, from the approved standards
- Import patterns
- Flag for any non-standard package needed (requires approval)

**Section 5: Deployment Abstraction Requirements**
- For Standard API modules: specifies all three layers (index.ts, server.ts, proxy.ts) with code skeletons
- BMAD must generate all three — not optional
- For UI/component modules: explicitly states no abstraction required

**Section 6: Success Criteria**
- Concrete, checkable conditions for "done"
- Not vague ("works correctly") but specific ("all exported functions have unit tests", "POST /api/auth/authenticateUser returns 401 on wrong credentials")

### What Gets Created

```
global-picture/features/{feature-name}/product-brief-{module-name}.md
```

The Solution Architecture document is also updated: the module's status changes from `not-started` to `in-development`.

### Human Gate

Claude summarises the Product Brief (function count, dependencies, packages, abstraction requirement) and asks:

> "Review the Product Brief for {module-name}. This is the document BMAD will use to build the module. Confirm requirements are complete, dependency references are correct, OSS packages are right, API contracts are accurate. Approve or request revisions?"

After approval, Claude gives you explicit instructions for the BMAD handoff (see [BMAD Coordination](bmad-coordination.md)).

### What Comes Next

Hand the Product Brief to BMAD. When BMAD finishes the module, run `/module-summary`.

---

## `/module-summary`

**Phase:** 5 close — Per module, after BMAD finishes
**Run:** After each module is built by BMAD
**Who runs it:** Developer (or Engineering Lead)

### Purpose

Validate a completed module against its Product Brief. Check that function signatures match, deployment abstraction is correctly implemented, OSS packages are compliant, and tests exist. On approval, officially mark the module complete, update all tracking files, and unblock dependent modules.

### Prerequisites

- BMAD has finished implementing the module
- The module's code exists in `modules/{module-name}/`
- The module's Product Brief exists in `global-picture/features/{feature-name}/`

### What Claude Does

**Scans the module code:**
- `index.ts` — what functions are exported? Do signatures match the Product Brief exactly?
- `server.ts` — does a REST server exist? One POST endpoint per exported function? Correct URL pattern?
- `proxy.ts` — does it check `DEPLOYMENT_MODE`? Does monolith path call functions directly? Does distributed path call the REST endpoint?
- Test files — do tests exist? What coverage is provided?
- `package.json` — what runtime dependencies are declared?

**Validates against the Product Brief:**

| Check | What's Verified |
|-------|----------------|
| API Contract | Every function in the brief exists in code with exact signature match |
| Scope | No functions exist in code that aren't in the brief (scope creep) |
| Dependency Usage | Dependencies imported via proxy (not direct index import) |
| Function Calls | Calls only functions listed in the brief's dependency section |
| OSS Compliance | Every package in `package.json` is either in the standards or has an approved deviation |
| Deployment Abstraction | All three layers present and correctly implemented |
| Tests | Unit tests exist for exported functions |

### Blocking vs Non-Blocking Issues

**Blocking issues** — module cannot be marked complete until resolved:
- Missing or incorrect function signatures
- Missing `server.ts` or `proxy.ts` (for Standard API modules)
- Non-standard OSS packages used without approval
- Dependencies imported directly (bypassing proxy)

**Non-blocking issues** — noted in the summary but don't block completion:
- Minor deviations from the brief with reasonable justification
- Test coverage below target (noted but doesn't block)

If there are blocking issues: Claude states them clearly and says "Return to BMAD to fix these, then re-run `/module-summary`."

### What Gets Created

```
global-picture/features/{feature-name}/module-summary-{module-name}.md
```

### Human Gate

Claude presents:
- Validation results for each check (pass / N issues)
- Any blocking issues (must be resolved before approval)
- Any deviations from the Product Brief
- Path to the summary document

If no blocking issues, asks:

> "Review the module summary. Does the module meet all Product Brief requirements? Are standards followed? Is the API contract correct? Approve to mark the module complete and unblock dependent modules."

### What Happens On Approval

Claude automatically updates all tracking files:

1. **Solution Architecture** — module status → `complete`
2. **`global-picture/api-docs/{module-name}.md`** — created (or updated) with the authoritative API contract for this module. Future modules that depend on this one read from here.
3. **`global-picture/oss-registry/registry.json`** — adds this module to `modules_using` for every package it uses
4. **`global-picture/features/registry.md`** — updates feature status to `complete` if all modules are done
5. **Module summary document** — status updated from `PENDING APPROVAL` to `APPROVED — {date}`

Claude then reports which modules are now unblocked as a result of this completion.

### What Comes Next

If modules remain: run `/initiate-module` for the next one.

If the feature is complete: all modules are done, the feature branch is ready for integration testing and merge to main.
