# Frequently Asked Questions

---

## General

### Can I use Milo on an existing codebase?

No. Milo is designed exclusively for greenfield projects — new products built from scratch in a mono-repo.

The reason: Milo's governance model assumes it defines all boundaries and standards from the beginning. Applying it to an existing codebase would require reverse-engineering domain boundaries that may already be violated, retroactively enforcing standards that weren't in place, and mapping an existing module graph that may have circular dependencies. That's not a workflow Milo supports.

If you're starting a new product and your existing codebase has patterns you want to reuse, consider extracting them as standalone packages that Milo-governed modules can depend on as external services.

---

### Does Milo write code?

No. Milo generates planning and governance documents. The code is written by BMAD. Milo tells BMAD exactly what to write (via the Product Brief), then validates that BMAD wrote it correctly (via `/module-summary`).

The one exception: Milo includes code skeletons in the Product Brief for the deployment abstraction layers — but these are templates for BMAD to fill in, not complete implementations.

---

### What language and framework does Milo support?

Milo is language-agnostic for the planning layer. The `/define-standards` workflow asks what language and packages your project uses — TypeScript, Python, Go, Java, or other — and all subsequent documents are tailored to your choices.

The code examples in `milo/knowledge/deployment-abstraction.md` are in TypeScript because it was the primary reference language during Phase I development. If your project uses a different language, Claude adapts the patterns — the three-layer principle (function exports, REST server, smart call wrapper) applies regardless of language.

---

### Is there a Phase 2?

Yes. Phase 2 introduces platform-level deployment tooling — specifically Google Service Weaver on GCP — which can manage distributed routing at the infrastructure level rather than the application level. The function export and REST server layers from Phase 1 remain unchanged in Phase 2.

Phase 2 is out of scope for this repository. Do not introduce any platform-specific deployment code in Phase 1 modules. The `DEPLOYMENT_MODE` environment variable handles routing for now; Phase 2 tooling will replace or augment the smart call wrapper when it arrives.

---

## Standards and Packages

### What if I want to change a standard mid-project?

Standards can be changed, but it's a formal process — not a one-person decision.

To amend a standard:
1. Open `global-picture/standards/` and update the relevant document
2. Update `global-picture/oss-registry/registry.json` if the change involves a package
3. Identify every module already using the old standard (check `modules_using` in the registry)
4. Update those modules to comply with the new standard
5. Re-run `/module-summary` for any affected modules
6. Add an entry to the Amendment Log at the bottom of the standards document: date, what changed, approved by

For OSS package changes specifically: changing the standard validation library mid-project means every module that used the old library needs to be updated. This is significant. The amendment process exists to force that conversation before the change is made.

---

### What if BMAD uses a package not in the standards?

`/module-summary` will flag it as a blocking issue. The module cannot be marked complete until the issue is resolved.

Resolution options:
1. **Replace the package** — BMAD updates the module to use the standard package instead
2. **Approve a deviation** — if the non-standard package is genuinely necessary (e.g. a third-party SDK with no standard equivalent), you approve it explicitly, and Milo records it in `global-picture/oss-registry/registry.json` under `deviations[]`

The right path depends on whether the non-standard package was an oversight (option 1) or a genuine requirement (option 2). Non-standard packages used silently — without flagging in the Product Brief and without deviation approval — are always treated as blocking issues.

---

### What if a module needs a package that isn't in the standards at all?

This should be caught during `/initiate-module`, not during `/module-summary`. When generating the Product Brief, if a module needs a package outside the standards (e.g. the Stripe SDK for a payments module), Milo flags it as an open question in the brief and asks for your approval before development starts.

If it somehow gets to `/module-summary` without being flagged: it's a blocking issue. You have the same two options as above.

---

## Module Development

### Can two developers work on different modules in parallel?

Yes — as long as those modules are in the same dependency tier (i.e. they don't depend on each other).

The process:
1. Run `/initiate-module` and select module A → approve the Product Brief
2. Run `/initiate-module` again and select module B → approve the Product Brief
3. Developer 1 builds module A in BMAD on branch `feature/module-a`
4. Developer 2 builds module B in BMAD on branch `feature/module-b`
5. When each module is done, run `/module-summary` to validate and complete it
6. Once both are marked complete, modules in the next tier become unblocked

The dependency graph from `/solution-architecture` tells you which modules can safely be parallelised.

---

### What happens if a module needs to call another that isn't done yet?

It can't be initiated. The dependency graph enforces this.

When you run `/initiate-module`, Milo checks the dependency graph and only considers modules where all dependencies are `complete`. If module B depends on module A, and module A is `in-development` or `not-started`, module B will not appear as an option.

This is intentional. The Product Brief copies exact function signatures from `global-picture/api-docs/` for every dependency. If a dependency isn't complete, its api-docs entry doesn't exist — the Product Brief can't be fully specified, so BMAD can't build correctly.

The fix: finish module A, run `/module-summary` to mark it complete, then `/initiate-module` will offer module B.

---

### What if an existing module needs new functions for a new feature?

This is handled in `/solution-architecture`. When scanning existing modules, Milo classifies them as "existing — needs extension" if new capabilities are required. The new functions become part of the current feature's scope.

The process:
1. The extended module is treated as a dependency that must be updated first
2. `/initiate-module` initiates the extended module first — its Product Brief includes both the existing functions (for context) and the new functions to add
3. BMAD adds the new functions to the existing module
4. `/module-summary` validates the additions and updates `global-picture/api-docs/` with the new function signatures
5. Dependent modules are then unblocked and can be initiated

The key rule: don't ask BMAD to add functions beyond what the Product Brief specifies. If the scope grows, regenerate the Product Brief.

---

### My module failed /module-summary validation. What now?

Read the blocking issues list carefully. Each issue is specific — a missing file, a wrong function signature, an incorrect import path.

Return to the BMAD session with the exact list of issues. For each one:
- Reference the specific Product Brief section it relates to
- Ask BMAD to fix only that issue — not to refactor the broader module

When BMAD makes the fixes, run `/module-summary` again. It validates the full module each time, so all previously-passing checks run again too.

If you disagree with a blocking issue (you think Milo is wrong), read the relevant standard or Product Brief section. If there's a genuine mistake in the brief, update the brief and re-run. Don't ask BMAD to deviate from the brief — the brief is the contract.

---

## global-picture/

### Do I ever edit global-picture/ manually?

Almost never. The entire point of `global-picture/` being owned by Milo workflows is consistency — if you edit one file manually, related files may become inconsistent (e.g. a module marked complete in `solution-architecture.md` but not reflected in `api-docs/` or `registry.json`).

The legitimate exceptions:
- Fixing a typo in a standards document
- Correcting a wrong function signature that was approved before development started
- Removing a feature folder for a feature that was cancelled

Even in these cases: after a manual edit, understand which other files it affects and whether they need to be updated too.

---

### What happens to global-picture/ if a feature is cancelled?

Nothing happens automatically. The feature folder and its documents remain. If you want to clean up:
- Remove `global-picture/features/{feature-name}/`
- Remove the feature row from `global-picture/features/registry.md`
- If any modules were marked `in-development`, update the solution architecture to reflect the cancellation

If modules from the cancelled feature were already marked `complete` and their api-docs published, those api-docs remain valid — other features can still depend on those completed modules.

---

## Claude Code

### What if Claude Code forgets the context mid-session?

Claude Code automatically compresses prior messages when approaching context limits. This is normal. The important thing is that `CLAUDE.md` is re-read at the start of every session — so the slash commands and Milo's operating rules are always restored.

If Claude seems confused mid-workflow, the safest recovery is to restart the session. The workflow files and `global-picture/` documents on disk are the ground truth — Claude can always re-read them.

---

### Can I run multiple Milo workflows in one session?

Yes. A typical session might run `/initiate-module` to generate a Product Brief, hand off to BMAD, and when BMAD finishes, run `/module-summary` in the same Claude Code session. There's no restriction on running multiple commands in sequence.

What you should not do is run two workflows simultaneously — finish one workflow completely (including its human gate) before starting the next.

---

### What if Claude proposes something in a workflow that I disagree with?

Every Milo workflow has human gates. At each gate, Claude asks for explicit approval. If you disagree with what was produced, say so and describe what needs to change — Claude will revise and re-present.

For `/solution-architecture` in particular, this back-and-forth is expected. Module boundaries are judgment calls, and Claude's first decomposition may not align with your domain knowledge. Push back, explain the business context, and Claude will revise the architecture.

The rule: never approve something you're not confident in just to move forward. Mistakes at the architecture level multiply — every module built on a wrong boundary inherits the problem.
