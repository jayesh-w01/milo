# Workflow: setup-monorepo

**Trigger:** `/setup-monorepo`
**Phase:** 0 — One-time project initialisation (run before everything else)
**Run by:** Human (Product/Engineering Lead)
**Prerequisites:** None. This is the very first command run in a new project.

---

## Purpose

Bootstrap the greenfield mono-repo from scratch. Initialise git, create the standard directory structure, install BMAD alongside Milo, make the first commit, and optionally push to GitHub. After this workflow completes, the project is ready for `/define-standards`.

---

## Step 1 — Assess Current State

Check the current working directory:

```
- Is this already a git repository? (check for .git/)
- Is milo/ present? (Milo must be here — it's how this workflow is running)
- Is bmad/ present?
- Are there any existing files/folders besides milo/?
```

Report findings clearly:

```
Current state:
  Git repository:  yes / no
  milo/ present:   yes / no
  bmad/ present:   yes / no
  Other contents:  {list or "none"}
```

If `milo/` is NOT present, something is wrong — Milo cannot run without its own files. Say:
> "Cannot find `milo/` in the current directory. Make sure you are running Claude Code from the project root where `milo/` is checked in. Current directory: {cwd}"
Stop.

---

## Step 2 — Gather Project Information

Ask the human (all at once):

> "Let's set up your mono-repo. A few quick questions:
>
> 1. **Project name** — what is this project called? (used for the repo name, directory naming, and README)
> 2. **Project description** — one sentence describing what this product does
> 3. **BMAD** — is BMAD already present in a `bmad/` folder, or does it need to be added?
>    - Already present
>    - I have a local copy at: {path}
>    - I need to clone it from GitHub (provide the BMAD repo URL)
> 4. **GitHub** — should we create a GitHub repository and push the first commit?
>    - Yes — create a new GitHub repo now (requires `gh` CLI authenticated)
>    - Yes — I already have a remote, just push
>    - No — skip GitHub for now"

Wait for answers before proceeding.

---

## Step 3 — Initialise Git Repository

If no `.git/` exists:

Run: `git init`

Confirm: "Initialised git repository."

If `.git/` already exists:

Confirm: "Git repository already initialised. Continuing."

---

## Step 4 — Create Mono-Repo Directory Structure

Create the following directories (skip any that already exist):

```
modules/          ← individual modules will be built here by BMAD
```

Create `.gitignore` at the project root:

```gitignore
# Dependencies
node_modules/
.pnp
.pnp.js

# Build outputs
dist/
build/
.next/
.nuxt/
out/

# Environment files
.env
.env.local
.env.*.local
!.env.example

# Logs
*.log
npm-debug.log*
yarn-debug.log*
pnpm-debug.log*

# OS files
.DS_Store
Thumbs.db

# IDE
.idea/
.vscode/
*.swp
*.swo

# Test coverage
coverage/
.nyc_output/

# Temporary files
*.tmp
*.temp
```

Create a minimal `README.md` at the project root:

```markdown
# {project-name}

{project-description}

## Structure

| Folder | Purpose |
|--------|---------|
| `milo/` | Milo — enterprise layer (standards, architecture, module orchestration) |
| `bmad/` | BMAD — module implementation workflows |
| `global-picture/` | Cross-module documentation: standards, API contracts, feature registry |
| `modules/` | Individual business modules |

## Getting Started

This project uses Milo + BMAD for structured enterprise development.

1. Open Claude Code in this directory
2. Run `/define-standards` to establish org-wide standards (if not already done)
3. Run `/solution-architecture` with your feature brief to begin feature development

## Workflows

| Command | When to Use |
|---------|------------|
| `/define-standards` | Once — sets org standards and creates `global-picture/` |
| `/solution-architecture` | Once per feature — DDD decomposition and module planning |
| `/initiate-module` | Before each module — generates Product Brief for BMAD |
| `/module-summary` | After each module — validates completion and unblocks next |
```

---

## Step 5 — Install BMAD

Handle the BMAD installation based on the human's answer in Step 2:

**If already present:**
Verify `bmad/` exists and contains expected BMAD files (look for a CLAUDE.md or similar entry point inside it).
Confirm: "BMAD found at `bmad/`. No action needed."

**If human provided a local path:**
Run: `cp -r {provided-path} ./bmad`
Confirm: "BMAD copied to `bmad/`."

**If cloning from GitHub:**
Run: `git clone {bmad-repo-url} bmad`
Confirm: "BMAD cloned to `bmad/`."

After installation, verify `bmad/` is present. If BMAD installation fails for any reason:
> "BMAD installation failed: {error}. You can add BMAD manually by placing it in a `bmad/` folder at the project root, then continue with the next steps."
Continue with the rest of the workflow — BMAD's absence does not block the mono-repo setup.

---

## Step 6 — Create Initial Commit

Stage and commit everything created so far:

```bash
git add milo/ bmad/ modules/ .gitignore README.md
git commit -m "chore: initial mono-repo setup with Milo and BMAD"
```

Confirm: "Initial commit created."

Show the commit summary:
```
Initial commit includes:
  milo/        — Milo enterprise layer
  bmad/        — BMAD implementation workflows (if installed)
  modules/     — empty, ready for module development
  .gitignore   — standard ignores
  README.md    — project overview
```

---

## Step 7 — GitHub Setup (if requested)

**If "create new GitHub repo":**

Check that `gh` CLI is available: `gh auth status`

If not authenticated:
> "`gh` CLI is not authenticated. Run `gh auth login` in your terminal, then re-run `/setup-monorepo` from Step 7."
Stop GitHub steps but confirm the rest of the setup is complete.

If authenticated, ask:
> "Create the GitHub repo as:
> - Public or private?
> - Organisation account or personal?"

Run:
```bash
gh repo create {project-name} --{public|private} --source=. --remote=origin --push
```

Confirm: "GitHub repository created and initial commit pushed."
Print the repository URL.

**If "already have a remote, just push":**

Ask: "What is the remote URL?"

Run:
```bash
git remote add origin {url}
git push -u origin main
```

Confirm: "Pushed to remote."

**If "skip GitHub":**

Note: "GitHub setup skipped. When ready to push, run: `git remote add origin {url} && git push -u origin main`"

---

## Step 8 — [HUMAN GATE] Confirm Setup Complete

Present the final summary:

```
## Mono-Repo Setup Complete

Project: {project-name}

Directory structure:
  milo/        ✓
  bmad/        ✓ / ✗ (not installed — add manually before module development)
  modules/     ✓ (empty — ready for development)
  .gitignore   ✓
  README.md    ✓

Git:           ✓ Initialised
First commit:  ✓ Created
GitHub:        ✓ Pushed to {url} / Skipped

Next step: Run /define-standards to establish organisation-wide standards.
```

Ask: **"Does this look correct? Any issues before we proceed to `/define-standards`?"**

Wait for confirmation.

---

## Step 9 — On Confirmation

> "Mono-repo is ready. Run `/define-standards` to establish your organisation's coding standards, OSS package choices, and create the `global-picture/` infrastructure."
