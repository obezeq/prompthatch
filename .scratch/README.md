# .scratch/

Methodology folder for multi-session AI development with Claude Code.

Copy this folder into the root of your project as `.scratch/`. The hatch meta-prompt inside (`_hatch.md`) generates per-initiative subfolders that decompose large tasks into ordered, atomic, verified sessions.

**This folder is the methodology, not output.** When you hatch a new initiative, the generated files (plan.md, prompts/, verify reports) appear in subfolders alongside this README.

## Rules

- **Gitignore decision is yours** — solo developers usually gitignore `.scratch/` (treat it as a personal scratchpad). Teams sharing initiatives across developers may commit it. Either works.
- **Not source of truth for production** — if something matters long-term, it belongs in `docs/`, `CLAUDE.md`, or the code itself. `.scratch/` is for the planning + execution audit trail of in-flight work.
- **No secrets** — never store API keys, passwords, or credentials here, even if gitignored.

## Structure

Each initiative gets its own directory with a consistent internal layout:

```
.scratch/
  _hatch.md                     <- meta-prompt to start a new initiative
  HOW-TO-USE.md                     <- workflow walkthrough
  README.md                         <- this file
  <initiative-slug>/                <- one directory per initiative (created by hatch)
    plan.md                         <- research, architecture, master plan
    00-preamble.md                  <- shared invariants for every session
    INDEX.md                        <- RAG-style navigation map
    issues.json                     <- transactional state (issue numbers, branch, PR)
    prompts/                        <- session prompts to execute the plan
      01-<step-name>.md             <- session 1
      02-<step-name>.md             <- session 2
      ...
      verify.md                     <- post-session auditor
      visual-audit-prompt.md        <- (optional, only for visual/frontend work)
    verification-report-<date>.md   <- audit reports (one per verify run)
```

## Conventions

- **Slug** = short kebab-case describing the initiative (e.g. `auth-system`, `payments-flow`, `notifications-module`)
- **plan.md** = always the master plan, one per initiative
- **prompts/** = numbered `NN-<name>.md` files, executed sequentially in fresh Claude Code sessions
- **00-preamble.md** = global invariants shared across all prompts in that initiative
- **verify.md** = generic auditor with PHASE 0 multi-signal scope detection — runs after each session

## How to start

When you hatch a new initiative, it appears here as a `<slug>/` subfolder. Read `HOW-TO-USE.md` for the full workflow walkthrough.
