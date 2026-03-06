# Getting Started

This guide walks you through setting up Milo in a new project from scratch, running your first session, and understanding what everything does.

---

## Prerequisites

Before you start, make sure you have:

**1. Claude Code**
Install via npm:
```bash
npm install -g @anthropic-ai/claude-code
```
Then authenticate:
```bash
claude login
```

**2. BMAD**
BMAD is the agentic development tool that Milo coordinates with. Get it from the [BMAD repository](https://github.com/bmad-dev/bmad-method). You'll need it during `/setup-monorepo` — have it ready as a local folder or a GitHub URL.

**3. gh CLI** (optional, but recommended)
If you want Milo to create your GitHub repository automatically:
```bash
brew install gh   # macOS
gh auth login
```

**4. A fresh, empty directory**
Milo is for greenfield projects only. Do not use it on an existing codebase. Start with an empty folder.

---

## The Bootstrapping Problem (Read This First)

Milo works by putting its workflow files inside your project as a `milo/` subfolder, then placing a `CLAUDE.md` at the project root. When you open Claude Code, it reads `CLAUDE.md` automatically — that's how it knows about Milo and the slash commands.

**The catch:** `CLAUDE.md` doesn't exist yet when you first start. It gets created by `/setup-monorepo`. So you can't type `/setup-monorepo` on your first session — Claude doesn't know about it yet.

**The solution:** On your very first session, you tell Claude to read and run the workflow file directly. This is a one-time step. Every session after that uses slash commands normally.

---

## Step 1 — Create Your Project Directory

```bash
mkdir my-new-project
cd my-new-project
```

This directory will become the root of your mono-repo. Everything — Milo, BMAD, your code modules, and the global-picture folder — lives inside it.

---

## Step 2 — Copy Milo Into the Project

You need the Milo files inside your project as a `milo/` subfolder before opening Claude Code. There are two ways to do this:

**Option A: Clone from GitHub**
```bash
git clone https://github.com/jayesh-w01/milo milo
```

**Option B: Copy from a local version**
```bash
cp -r /path/to/milo ./milo
```

After this step, your directory should look like:
```
my-new-project/
└── milo/
    ├── CLAUDE.md
    ├── agents/
    ├── workflows/
    ├── templates/
    └── knowledge/
```

---

## Step 3 — Open Claude Code

From inside your project directory:
```bash
claude
```

Claude Code will open. It will look for a `CLAUDE.md` at the project root — which doesn't exist yet (it lives inside `milo/`, not the root). That's fine. You're about to create it.

---

## Step 4 — Run Setup (First Time Only)

In the Claude Code session, type:

```
Read milo/workflows/setup-monorepo.md and run it.
```

Claude will read the workflow and begin executing it. Here's what happens:

**Claude checks the current state:**
It looks for `.git/`, `milo/`, `bmad/`, and any other existing files. It reports what it finds.

**Claude asks you a set of questions (all at once):**
- Project name
- Project description (one sentence)
- Whether BMAD is already present, or needs to be installed
- Whether you want a GitHub repository created

**Answer them all in one reply.** For example:
```
1. acme-platform
2. B2B SaaS platform for invoice management
3. I need to clone it from GitHub: https://github.com/bmad-dev/bmad-method
4. Yes, create a new public GitHub repo under my personal account
```

**Claude then does the work:**
- Runs `git init`
- Creates the `modules/` directory
- Creates `.gitignore` with sensible defaults
- Installs BMAD (clones it to `bmad/`)
- Creates the project root `README.md`
- Creates `CLAUDE.md` at the project root (this is the key file that registers all slash commands)
- Makes the initial commit
- If you said yes to GitHub: creates the repo and pushes

**Human gate:** Claude presents a summary and asks for confirmation before finalising.

After you approve, your project looks like:
```
my-new-project/
├── CLAUDE.md              ← registers Milo's slash commands
├── README.md              ← project readme
├── .gitignore
├── milo/                  ← Milo enterprise layer
├── bmad/                  ← BMAD implementation workflows
└── modules/               ← empty, ready for development
```

---

## Step 5 — Every Session After This

From this point on, every time you open Claude Code in your project directory, it reads `CLAUDE.md` automatically and knows about all Milo commands. You just type slash commands:

```
/define-standards
```

No more "read this file and run it" — that was only the very first time.

---

## Your First /define-standards Session

This is the next command to run. It establishes your organisation's standards for the entire project. You only run it once.

**What Claude will ask you (in 4 grouped sets):**

**Group A — Language & Runtime**
- Primary programming language (TypeScript, Python, Go, etc.)
- Runtime (Node.js, Deno, Bun — if TypeScript)
- Package manager (npm, yarn, pnpm, bun)
- TypeScript strictness and ESLint config (if applicable)

**Group B — Backend OSS Packages**
- HTTP server framework (Express, Fastify, Hono, etc.)
- HTTP client (axios, node-fetch, native fetch, etc.)
- Database ORM/client (Prisma, Drizzle, Knex, etc.)
- Validation library (Zod, Joi, Yup, etc.)
- Logging library (Pino, Winston, etc.)
- Testing framework (Vitest, Jest, etc.)
- Date/time library (date-fns, dayjs, etc.)
- Auth/JWT library (jsonwebtoken, jose, etc.)

**Group C — Frontend OSS Packages** (skip if no UI modules)
- UI framework (React, Vue, Svelte)
- Meta-framework (Next.js, Nuxt, SvelteKit)
- Component library (shadcn/ui, Radix, MUI, etc.)
- Styling (Tailwind, CSS Modules, etc.)
- State management (Zustand, Jotai, Redux Toolkit, etc.)

**Group D — Coding & Architecture**
- Indentation and semicolons
- Module internal layer structure
- Error handling convention (exceptions vs Result types)
- Testing approach (unit only, unit + integration, full pyramid)
- API versioning strategy

**Tip:** Think about these before the session. Group B especially — these choices are locked in for the entire project. Changing them later requires a formal standards amendment.

After you answer all groups, Claude generates three documents:
- `global-picture/standards/coding-standards.md`
- `global-picture/standards/oss-packages.md`
- `global-picture/standards/architectural-patterns.md`

And initialises:
- `global-picture/oss-registry/registry.json`

**Human gate:** Claude presents all the outputs for your review and asks for explicit approval before finalising.

---

## What the Final Project Structure Looks Like

After `/define-standards` runs, your project is ready for feature development:

```
my-new-project/
├── CLAUDE.md
├── README.md
├── .gitignore
├── milo/
├── bmad/
├── modules/                    ← still empty
└── global-picture/
    ├── standards/
    │   ├── coding-standards.md
    │   ├── oss-packages.md
    │   └── architectural-patterns.md
    ├── api-docs/
    │   └── README.md           ← populated as modules complete
    ├── features/
    │   ├── registry.md
    │   └── README.md
    └── oss-registry/
        └── registry.json
```

From here, run `/solution-architecture` with your first feature's product brief.

---

## Common Setup Errors

**"Cannot find milo/ in the current directory"**
You ran Claude Code from the wrong directory, or forgot to copy Milo in. Check that `milo/` exists in your current working directory.

**"gh CLI is not authenticated"**
Run `gh auth login` in a separate terminal, complete the authentication, then come back to Claude Code.

**BMAD clone fails**
Check the URL is correct and you have internet access. If it fails, skip the GitHub option and copy BMAD manually later: `cp -r /path/to/bmad ./bmad`. BMAD's absence does not block the rest of setup.

**Slash commands not recognised after setup**
Make sure `CLAUDE.md` was created at the project root (not inside `milo/`). If it's missing, something went wrong in step 4. Check by running `ls CLAUDE.md` in your project root.

**Standards questions feel overwhelming**
Answer what you know. For anything you're unsure about, Milo will suggest sensible defaults based on your language and runtime choices. You can always see the generated documents and request corrections before approving.
