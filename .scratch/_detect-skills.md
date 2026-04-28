# Skill Detection Prompt — Run BEFORE the hatch

This is the prerequisite step for PromptHatch. Paste the block below into a fresh Claude Code conversation from your project root.

The prompt will scan:
- **User-level skills** in `~/.claude/skills/`, `~/.claude/agents/`, `~/.claude/plugins/`
- **Project-level skills** in `<repo>/.claude/skills/`, `<repo>/.claude/agents/`
- **MCP servers** in `~/.claude.json` (top-level + per-project)
- **Enabled plugins** in `~/.claude/settings.json`

It writes the inventory to `.scratch/skills.json`. The hatch reads that file in Step 0 and adapts session prompts accordingly (using inline fallbacks when a recommended skill is missing).

Run this **once per project**. Re-run after installing new skills so `skills.json` stays accurate.

---

```
=== META-PROMPT: DETECT CLAUDE CODE SKILLS ===

DETECT_SKILLS_VERSION: 0.0.2
SCHEMA_VERSION: 1.0

=== MISSION DECLARATION (read first, internalize, never relax) ===

This probe must produce a FLAWLESS inventory. No skills missed. No false positives. No fabricated entries. No assumptions where a filesystem read can give you certainty. Every entry in skills.json is backed by RE-EXECUTABLE EVIDENCE (file path + content hash or frontmatter quote), never by narrative confidence or training-data recall.

Quality target: production-grade. The hatch reads your output to make ABORT/CONTINUE decisions — a wrong skills.json silently breaks every subsequent initiative. If you cannot read a directory, say so explicitly. If a skill has malformed frontmatter, record it as broken; never invent a name. Cross-platform shells (Linux / macOS / WSL) MUST all work — if a command works only in bash, find the POSIX equivalent.

Internet research is permitted but not required for THIS probe (it operates against the local filesystem). However: when emitting install hints for missing REQUIRED skills, the hints MUST point to canonical 2026 sources verified to exist (the four-option block below has all been verified — do not improvise alternatives without WebSearch confirmation).

You are running the skill detection probe for PromptHatch. Your single job: enumerate every Claude Code skill, agent, plugin, and MCP server available in this environment, classify them as REQUIRED / RECOMMENDED / OPTIONAL / UNKNOWN, and write the result to `.scratch/skills.json`.

Do NOT write any other files. Do NOT modify the hatch. Do NOT proceed to scaffolding. Your output is a single JSON file plus a concise chat summary.

=== STEP 1 — VERIFY ENVIRONMENT ===

Run these checks. Abort with a clear error if any fails:

1. `git rev-parse --git-dir` succeeds (must be inside a git repo).
2. `mkdir -p .scratch` succeeds (need write access).
3. `command -v jq` succeeds OR fall back to degraded mode (note in output).
4. `command -v find` succeeds (POSIX requirement).

=== STEP 2 — PROBE USER-LEVEL SKILLS ===

Scan these paths (skip silently if missing):

```bash
USER_SKILLS_DIR="$HOME/.claude/skills"
USER_AGENTS_DIR="$HOME/.claude/agents"
USER_PLUGINS_DIR="$HOME/.claude/plugins"
USER_PLUGINS_REGISTRY="$HOME/.claude/plugins/installed_plugins.json"
USER_SETTINGS="$HOME/.claude/settings.json"
USER_CLAUDE_JSON="$HOME/.claude.json"
```

For each `SKILL.md` found under `$USER_SKILLS_DIR`:
- Resolve symlinks: `readlink -f <path>`
- Extract YAML frontmatter (between first two `---` delimiters):
  - `name` (required, must match parent folder name)
  - `description` (required, ≤1024 chars)
  - `allowed-tools` (optional)
- If `name` is missing OR doesn't match folder name: record under `broken_skills` with reason.

For each `*.md` under `$USER_AGENTS_DIR`:
- Extract frontmatter (`description`, `model`, `tools`).
- Use file basename (sans `.md`) as agent name.

For plugins:
- Read `$USER_SETTINGS` and extract `enabledPlugins` array — this is the source of truth.
- For each enabled plugin, locate its files under `$USER_PLUGINS_DIR/cache/<marketplace>/<plugin>/<version>/`.
- Inside each plugin directory, scan `skills/`, `commands/`, `agents/`, `hooks/` and add their entries (mark `source: "plugin"` and record the plugin name).

For MCP servers:
- Parse `$USER_CLAUDE_JSON` (top-level `mcpServers` keys only — do NOT read values; they may contain secrets).
- Also extract per-project keys: `projects[<this-repo-path>].mcpServers`.

=== STEP 3 — PROBE PROJECT-LEVEL SKILLS ===

Scan these paths inside the current repo (skip silently if missing):

```bash
PROJECT_SKILLS_DIR=".claude/skills"
PROJECT_AGENTS_DIR=".claude/agents"
PROJECT_SETTINGS=".claude/settings.json"
PROJECT_SETTINGS_LOCAL=".claude/settings.local.json"
```

Apply the same parsing logic as Step 2 to project-level paths.

Project-level skills override user-level skills with the same name (precedence: project > user > plugin > built-in).

=== STEP 4 — CLASSIFY ===

Use this classification rubric. The hatch depends on REQUIRED skills being present; missing them aborts Step 0.

REQUIRED (the methodology spine — abort hatch if missing):
- verification-before-completion
- systematic-debugging
- test-driven-development
- writing-plans
- executing-plans

RECOMMENDED — Frontend domain (warn if missing; hatch injects WebSearch+context7 fallback):
- gsap-core
- gsap-react
- frontend-design
- frontend-ux
- vercel-react-best-practices
- web-design-guidelines

RECOMMENDED — Backend domain:
- nodejs-best-practices
- claude-api

RECOMMENDED — Smart contracts domain:
- foundry
- solidity-security
- solidity-auditor
- evm-testing
- openzeppelin
- arbitrum
- arbitrum-dapp-skill

RECOMMENDED — Web3 domain:
- viem
- wagmi
- eth-concepts
- eip-reference
- account-abstraction

RECOMMENDED — Security domain:
- vulnhunter
- code-recon
- semgrep-solidity

OPTIONAL (bonus utility, not required):
- gsap-scrolltrigger
- gsap-timeline
- gsap-plugins
- gsap-utils
- gsap-performance
- finishing-a-development-branch
- requesting-code-review
- receiving-code-review
- subagent-driven-development
- dispatching-parallel-agents
- using-superpowers
- frontend-design (if not in frontend rec)
- foundry (if not in contracts rec)

UNKNOWN — any skill found that doesn't match these lists. Record as-is; hatch may use it if INDEX.md references it.

=== STEP 5 — WRITE skills.json ===

Write the result to `.scratch/skills.json` with this exact schema:

```json
{
  "schema_version": "1.0",
  "detected_at": "<ISO 8601 timestamp>",
  "host": "<output of: hostname>",
  "os": "<output of: uname -s>",
  "user_skills": [
    {
      "name": "<skill name from frontmatter>",
      "path": "<absolute path to SKILL.md>",
      "source": "user",
      "description": "<from frontmatter, truncated to 200 chars>",
      "allowed_tools": ["<list>"]
    }
  ],
  "project_skills": [
    {
      "name": "...",
      "path": "<absolute path>",
      "source": "project",
      "description": "...",
      "allowed_tools": ["..."]
    }
  ],
  "user_agents": [
    {
      "name": "<basename without .md>",
      "path": "<absolute path>",
      "model": "<from frontmatter or null>",
      "description": "<from frontmatter>"
    }
  ],
  "project_agents": [
    {"name": "...", "path": "...", "model": "...", "description": "..."}
  ],
  "enabled_plugins": [
    {
      "name": "<plugin name>",
      "marketplace": "<marketplace name>",
      "version": "<version>",
      "skills_provided": ["<skill names>"],
      "agents_provided": ["<agent names>"],
      "commands_provided": ["<command names>"]
    }
  ],
  "mcp_servers": {
    "user_level": ["<server names only, no values>"],
    "project_level": ["<server names only>"]
  },
  "all_skill_names": ["<deduplicated union of all skills, after precedence>"],
  "classification": {
    "required_present": ["<names>"],
    "required_missing": ["<names>"],
    "recommended_present_by_domain": {
      "frontend": ["<names>"],
      "backend": ["<names>"],
      "contracts": ["<names>"],
      "web3": ["<names>"],
      "security": ["<names>"]
    },
    "recommended_missing_by_domain": {
      "frontend": ["<names>"],
      "backend": ["<names>"],
      "contracts": ["<names>"],
      "web3": ["<names>"],
      "security": ["<names>"]
    },
    "optional_present": ["<names>"],
    "unknown_present": ["<names>"]
  },
  "broken_skills": [
    {
      "path": "<absolute path>",
      "reason": "<frontmatter mismatch, missing name, etc>"
    }
  ],
  "fallback_strategy": {
    "for_missing_recommended": "WebSearch + context7 with domain-specific queries cited in PR body",
    "for_missing_required": "ABORT hatch — install required skills first"
  }
}
```

ATOMIC WRITE PROTOCOL: write to `.scratch/skills.json.tmp`, validate JSON, then `mv -f .scratch/skills.json.tmp .scratch/skills.json`.

=== STEP 6 — REPORT TO USER ===

Emit a single concise chat summary (≤25 lines) following this template:

```
━━━ Skill detection complete ━━━

Host:           <hostname> (<os>)
Detected at:    <timestamp>

Inventory:
  User skills:        N
  Project skills:     N (<repo-name>)
  User agents:        N
  Project agents:     N
  Enabled plugins:    N
  MCP servers:        N user + N project

Classification:
  ✓ Required present:    N/5  [list missing if any]
  ✓ Recommended present: N total across domains
    • Frontend:     N present, M missing
    • Backend:      N present, M missing
    • Contracts:    N present, M missing
    • Web3:         N present, M missing
    • Security:     N present, M missing
  ◯ Optional present:    N
  ? Unknown skills:      N

[ONLY if required_missing is non-empty, emit:]
✗ MISSING REQUIRED SKILLS — hatch will abort:
  • <skill-name>: <how to install>

[ONLY if recommended_missing has items, emit:]
▲ Missing recommended skills (hatch will use WebSearch+context7 fallback):
  • <skill-name> (<domain>): consider installing for <reason>

Output written to: .scratch/skills.json (<size> bytes)

Next step:
  Open a fresh Claude Code conversation and paste .scratch/_hatch.md
  to start a new initiative. The hatch will read .scratch/skills.json
  and adapt session prompts to your environment.
```

=== INSTALLATION HINTS (for missing required skills) ===

The 5 required skills (`verification-before-completion`, `systematic-debugging`, `test-driven-development`, `writing-plans`, `executing-plans`) all live in the `obra/superpowers` skill pack. Print FOUR install options ranked by user context, so the user is unblocked regardless of their environment.

When ANY required skill is missing, emit this block verbatim (substitute `<missing>` with the actual missing names):

```
✗ Missing required skills: <missing>

These come from the obra/superpowers skill pack. Pick whichever option fits your environment:

OPTION 1 (recommended — interactive Claude Code session):
  /plugin marketplace add obra/superpowers-marketplace
  /plugin install superpowers@superpowers-marketplace

OPTION 2 (CI / scripts / headless):
  claude plugin marketplace add obra/superpowers-marketplace
  claude plugin install superpowers@superpowers-marketplace

OPTION 3 (project-aware multi-agent installer — skills.sh by Vercel Labs):
  npx skills add https://github.com/obra/superpowers
  # or to discover stack-relevant skills first:
  npx skills find <your-stack-keyword>

OPTION 4 (manual git clone fallback):
  git clone https://github.com/obra/superpowers ~/.claude/skills/obra-superpowers
  # then add the skills you need to ~/.claude/settings.json `enabledPlugins`

After installing, re-run .scratch/_detect-skills.md to refresh skills.json.
Then re-run .scratch/_hatch.md to hatch your initiative.
```

For RECOMMENDED-but-missing skills (warn, do not abort), append a second block recommending project-aware discovery:

```
▲ Missing recommended skills for your stack: <missing>

To discover and install stack-relevant skills:
  npx skills find <stack-keyword>     # e.g. "fastapi", "spring-boot", "solidity"
  npx skills add <picked-skill>

Or install all from a known marketplace:
  claude plugin marketplace add anthropics/claude-plugins-official
  claude plugin install <skill-name>@claude-plugins-official

The hatch will inject inline `WebSearch + context7` fallbacks for any
skill marked missing in skills.json — so the methodology still works,
just slower and less domain-specific.
```

Verifiable sources for these install paths:
- skills.sh — https://skills.sh/ (Vercel Labs, npx-based, multi-agent)
- claude-plugins-official — https://github.com/anthropics/claude-plugins-official (Anthropic, curated)
- obra/superpowers-marketplace — https://github.com/obra/superpowers-marketplace (accepted into official marketplace 2026-01)
- claude plugin docs — https://code.claude.com/docs/en/plugin-marketplaces

=== CRITICAL RULES ===

- Read paths only. Never read VALUES from `~/.claude.json` (only KEYS). MCP server tokens / API keys may be present.
- Do NOT echo skill descriptions verbatim if they contain secrets (very rare but possible in custom skills).
- Do NOT modify any file outside `.scratch/skills.json` and `.scratch/skills.json.tmp`.
- Do NOT proceed past Step 6 — report and stop.
- If `.scratch/skills.json` already exists, OVERWRITE atomically (this prompt is re-runnable by design).
- If you encounter a skill with malformed frontmatter, record it in `broken_skills`, do not abort.
- Use POSIX-compliant shell only (`sh`, not `bash`/`zsh`-specific). Test commands work with `sh -c '...'`.

=== END META-PROMPT ===
```

---

## Why this is a separate prompt

Skill detection is **stateless** and **idempotent** — it depends only on the filesystem state of `~/.claude/` and the project's `.claude/`, not on any initiative's slug or branch. Running it once per project (or after installing new skills) gives every subsequent hatch a current view of available capabilities without re-probing.

The hatch reads `.scratch/skills.json` in Step 0 and uses it to:
- Abort early if REQUIRED skills are missing (no point starting Step 2 if `verification-before-completion` is unavailable)
- Inject inline `WebSearch + context7` fallbacks in session prompts where a recommended skill is missing
- Tag the generated `INDEX.md` with `(installed)` or `(missing — fallback active)` per skill
- Keep `00-preamble.md` honest about what's actually available

Re-run this prompt anytime you install or remove a skill. The hatch is forward-compatible with any `skills.json` matching `schema_version: 1.0`.
