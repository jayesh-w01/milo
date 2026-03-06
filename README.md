# Milo — Enterprise Layer for Greenfield Software Development

Milo is a structured enterprise planning and governance layer that works alongside BMAD to build greenfield software products. It answers three questions that BMAD never asks:

1. **What modules should we build, and where do the boundaries go?** — using Domain-Driven Design
2. **What standards must every module follow?** — OSS packages, coding conventions, architectural patterns
3. **How do modules talk to each other without being locked into a deployment model?** — deployment abstraction

Milo runs *before* BMAD on every module. It sets the foundation. BMAD builds within it.

---

## The Mental Model

```
Your product brief
      ↓
  [ Milo ]  ←── sets boundaries, standards, and contracts
      ↓
  Product Brief (per module)
      ↓
  [ BMAD ]  ←── builds the module within those constraints
      ↓
  [ Milo ]  ←── validates the output, marks complete, unblocks next
      ↓
  Next module
```

Milo is not a plugin or extension to BMAD. It is a sibling tool that runs in the same Claude Code environment. The handoff between them is a document — the Product Brief — and you (the human) are the bridge.

---

## What Milo Is (and Is Not)

**Milo is:** A set of markdown workflow files that Claude Code reads and executes as an intelligent enterprise architect agent. There is no binary to install, no server to run, no build step. The markdown files *are* Milo.

**Milo is not:** A code generator, a framework, or a library. It produces planning and governance documents, validates outputs, and directs BMAD to build code within the constraints it sets.

**Milo requires:** Greenfield projects only. It is designed for new products built from scratch in a mono-repo. Do not attempt to apply Milo to an existing codebase.

---

## The 5 Commands

All interaction with Milo happens through slash commands in Claude Code. Run them in order — never skip a phase.

| Command | Phase | Run | Purpose |
|---------|-------|-----|---------|
| `/setup-monorepo` | 0 | Once per project | Bootstrap the mono-repo: git init, directory structure, BMAD installation, first commit |
| `/define-standards` | 1 | Once per project | Establish org-wide standards: language, OSS packages, coding conventions, architecture |
| `/solution-architecture` | 2 | Once per feature | Apply DDD to a product brief, decompose into modules, produce dependency graph |
| `/initiate-module` | 4 | Before each module | Select next unblocked module, generate complete Product Brief for BMAD |
| `/module-summary` | 5 | After each module | Validate completed module, mark complete, update registries, unblock dependents |

---

## Prerequisites

- [Claude Code](https://docs.anthropic.com/claude-code) installed and authenticated
- [BMAD](https://github.com/bmad-dev/bmad-method) — the agentic development tool Milo coordinates with
- A fresh, empty directory for your new project (greenfield only)
- `gh` CLI installed and authenticated (only needed if you want GitHub repo creation automated)

---

## Quick Start

```bash
# 1. Create your project directory
mkdir my-new-project
cd my-new-project

# 2. Copy Milo into it
cp -r /path/to/milo ./milo

# 3. Open Claude Code
claude

# 4. Tell Claude to run the setup workflow (first time only — CLAUDE.md doesn't exist yet)
# In the Claude Code session, say:
"Read milo/workflows/setup-monorepo.md and run it."

# 5. From here, all subsequent sessions use slash commands:
/define-standards
/solution-architecture
/initiate-module
/module-summary
```

See [Getting Started](docs/getting-started.md) for the full step-by-step walkthrough.

---

## How a Feature Gets Built

```
/solution-architecture  ← paste in your product brief, get a module breakdown

    ↓ (repeat for each module in dependency order)

/initiate-module        ← Milo selects next module, generates Product Brief
    ↓
[switch to BMAD]        ← BMAD reads Product Brief, builds the module
    ↓
/module-summary         ← Milo validates, marks complete, unblocks next module

    ↓ (when all modules complete)

Feature branch ready for integration testing and merge
```

---

## Repository Structure

```
milo/
├── CLAUDE.md                    ← Entry point: read automatically by Claude Code
├── README.md                    ← This file
├── docs/                        ← Full documentation
│   ├── getting-started.md
│   ├── commands.md
│   ├── concepts.md
│   ├── bmad-coordination.md
│   └── faq.md
├── agents/
│   └── milo-architect.md        ← Claude's persona when running Milo
├── workflows/
│   ├── setup-monorepo.md        ← Phase 0
│   ├── define-standards.md      ← Phase 1
│   ├── solution-architecture.md ← Phase 2
│   ├── initiate-module.md       ← Phase 4
│   └── module-summary.md        ← Phase 5 close
├── templates/
│   ├── product-brief.md
│   ├── solution-architecture.md
│   ├── module-summary.md
│   └── standards-doc.md
└── knowledge/
    ├── ddd-principles.md
    ├── deployment-abstraction.md
    └── oss-registry-schema.md
```

When Milo runs in your project, it also creates:

```
your-project/
├── milo/                        ← Milo (this repo, copied in)
├── bmad/                        ← BMAD
├── modules/                     ← your code lives here (built by BMAD)
├── global-picture/              ← Milo's living reference (created by /define-standards)
│   ├── standards/
│   ├── api-docs/
│   ├── features/
│   └── oss-registry/
└── CLAUDE.md                    ← project root entry point (created by /setup-monorepo)
```

---

## Documentation

| Doc | What It Covers |
|-----|----------------|
| [Getting Started](docs/getting-started.md) | Full setup walkthrough, first session, common errors |
| [Commands Reference](docs/commands.md) | Every command in detail: inputs, outputs, human gates |
| [Concepts](docs/concepts.md) | DDD, deployment abstraction, global-picture, module types |
| [BMAD Coordination](docs/bmad-coordination.md) | How Milo and BMAD work together, handoff process |
| [FAQ](docs/faq.md) | Common questions and edge cases |

---

## Key Principles

- **Human gates are real.** Milo stops and waits for explicit approval at critical points. Every gate is a decision checkpoint — do not skip them.
- **global-picture/ is sacred.** Never edit it manually. All updates flow through Milo workflows.
- **Standards are enforced.** /module-summary blocks completion if a module violates OSS standards or is missing deployment abstraction layers.
- **Dependency order matters.** Build lower-level modules first. The dependency graph from /solution-architecture is the authority.
- **The Product Brief is the contract.** BMAD builds exactly what it says. If requirements change, regenerate the brief — don't ask BMAD to improvise.
