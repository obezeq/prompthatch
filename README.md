# PromptHatch

> **Spec-driven AI development for Claude Code.** Hatch verified sessions from one prompt.

`Status: early development` · `Version: 0.0.2` · `License: MIT`

## Why this exists

Long autonomous AI coding sessions drift silently, introduce hidden bugs, and produce no audit trail. Most AI dev tools either give you a plan and trust the agent blindly, or let the agent run autonomously without verification. Both produce code that is hard to review and impossible to audit.

## What PromptHatch does

A meta-prompt that decomposes large AI development tasks into ordered, atomic sessions, each verified by an evidence-based auditor before the next one starts. Every prompt is a session, every session ends with proof, nothing advances without it.

## For whom

- Developers using Claude Code who want **structure over autonomy**
- Teams that need **audit trails** for AI-generated code (DORA, SOC 2, EU AI Act compliance)
- Solo developers tired of **half-finished AI sessions** drifting at hour 3

## Quickstart

```bash
# 1. Get the methodology
git clone https://github.com/obezeq/prompthatch.git temp && \
  cp -r temp/.scratch ./ && rm -rf temp

# 2. Detect installed skills (one-time per project)
#    Open fresh Claude Code, paste contents of .scratch/_detect-skills.md

# 3. Hatch your first initiative
#    Open another fresh Claude Code, paste contents of .scratch/_hatch.md
#    Add at the end:  slug: my-feature
```

Full walkthrough: [`.scratch/HOW-TO-USE.md`](.scratch/HOW-TO-USE.md)

## How it compares

|                                       | PromptHatch       | Spec Kit  | BMAD     | superpowers | Cline    |
|---------------------------------------|-------------------|-----------|----------|-------------|----------|
| Structured plan                       | ✓                 | ✓         | ✓        | partial     | partial  |
| Verify auditor (LLM-as-judge)         | ✓ + 6H re-exec    | ✗         | partial  | partial     | ✗        |
| Evidence schema (anti-hallucination)  | ✓ 7-field         | ✗         | ✗        | ✗           | ✗        |
| Multi-signal scope detection          | ✓ 5 signals       | ✗         | ✗        | ✗           | ✗        |
| Transactional issue creation          | ✓                 | ✗         | ✗        | ✗           | ✗        |
| Skill availability detection          | ✓                 | ✗         | ✗        | partial     | ✗        |
| FULL / LIGHT / MINIMAL modes          | ✓                 | ✗         | ✗        | ✗           | ✗        |
| GitHub + GitLab + no-platform support | ✓                 | GitHub    | partial  | partial     | partial  |
| Lightweight (no install, copy-paste)  | ✓ 4 files         | heavy     | heavy    | ✓           | install  |

## Three tracking modes

| Mode        | What it tracks                                              | Best for                              |
|-------------|-------------------------------------------------------------|---------------------------------------|
| **FULL**    | Parent issue + sub-issue per session + draft PR             | Teams sharing initiatives             |
| **LIGHT**   | Parent issue + draft PR with checklist                      | Solo developers + platform            |
| **MINIMAL** | Local branch only, no issues, no PR                         | Prototypes, no-platform projects      |

Mode auto-recommended based on project maturity. Details in [`HOW-TO-USE.md`](.scratch/HOW-TO-USE.md).

## What's inside

```
.scratch/
├── _detect-skills.md   ← run once per project (writes skills.json)
├── _hatch.md           ← run per initiative (creates plan + sessions)
├── HOW-TO-USE.md       ← workflow walkthrough
└── README.md           ← conventions
```

## Status

- ✓ Core hatch meta-prompt (mode-aware, multi-platform)
- ✓ Verify auditor with PHASE 6H re-execution (anti-hallucination)
- ✓ Skill detection probe + project maturity probe
- ✓ Auto-detection (pre-fills 6 of 10 questions)
- ✓ Multi-developer locking (POSIX `noclobber`)
- ☐ Stack-specific presets (Solidity, FastAPI, Spring Boot, Django, Go)
- ☐ CLI tool (`npx prompthatch init`)
- ☐ Documentation site

[Star ★](https://github.com/obezeq/prompthatch) to follow progress.

## License

MIT — see [LICENSE](LICENSE).

---

Built by [@obezeq](https://github.com/obezeq).
