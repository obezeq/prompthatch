# Hatch Prompt — New `.scratch/` Initiative (v0.0.2)

**Prerequisite:** Before running this hatch for the first time in a project, paste `.scratch/_detect-skills.md` into a fresh Claude Code conversation. That prompt scans your installed skills (user-level + project-level) and writes `.scratch/skills.json`. The hatch reads that file in Step 0 and adapts to your environment. Re-run `_detect-skills.md` whenever you install or remove a skill.

After `skills.json` exists, paste the block below into a fresh Claude Code conversation from your project root. The agent will verify state, study the methodology, ask 10 clarifying questions (most pre-filled by auto-detection), enter Plan Mode with parallel research, generate a full `.scratch/<slug>/` scaffolding, and (depending on the mode you choose) wire issues + branch + PR transactionally.

Use this when starting a new multi-session initiative (any feature, module, or refactor that benefits from structured decomposition).

---

```
=== META-PROMPT: HATCH NEW SCRATCH INITIATIVE ===

HATCH_VERSION: 0.0.2
SCHEMA_VERSION: 1.0

=== MISSION DECLARATION (read first, internalize, never relax) ===

This work must operate FLAWLESSLY under real-world conditions. No half-finished implementations. No untested edge cases. No "probably works" shortcuts. No hidden bugs. Nothing that breaks existing functionality. Every claim of completion is backed by RE-EXECUTABLE EVIDENCE (exit_code + stdout from concrete commands), never by narrative confidence or training-data recall.

Quality target: production-grade. Ship only work whose correctness you can DEMONSTRATE with re-runnable evidence — not work you believe is correct. If you cannot prove it works, it does not work.

Internet research is MANDATORY, not optional. When uncertain about any library API, framework pattern, language version behavior, security recommendation, deprecation, or 2026-canonical practice: use WebSearch + context7 BEFORE adopting the pattern. Do not guess. Do not rely on training data alone. Verify against current authoritative sources (official docs, RFCs, vendor changelogs, CVE databases). The MIT-cutoff of training data is irrelevant to 2026 reality — the web is.

Parallel research is PREFERRED over sequential reasoning. When multiple independent unknowns exist, fan out work via parallel subagents (the maturity-adapted parallel-research step does this automatically; if you encounter further independent unknowns mid-execution, spawn additional agents without asking).

Skills are MANDATORY tools, not optional flavor. /skill verification-before-completion is required before closing any session. /skill systematic-debugging is required before proposing any fix. All REQUIRED skills (per skills.json) must be available — if any is missing, hatch ABORTS in Step 0.

You are helping me hatch a new initiative in .scratch/ using the methodology described in this folder. Execute Steps 0 → 6 in order. Do NOT skip ahead. Do NOT start executing sessions — the hatch ends when scaffolding is created and (if applicable) issues + branch + PR are wired.

This hatch supports three tracking modes (chosen in Q4d):
- FULL — parent issue + sub-issue per session + draft PR + transactional issues.json (best for teams; requires GitHub or GitLab)
- LIGHT — parent issue + draft PR with session checklist + lightweight issues.json (best for solo + platform)
- MINIMAL — local feature branch + state.json only (best for prototypes or no-platform projects)

And three hosting platforms (chosen in Q4c):
- github — uses `gh` CLI
- gitlab — uses `glab` CLI
- none — no platform CLI, local git only (forces MINIMAL mode)

GitLab caveat (read once, remember): GitLab does NOT have native sub-issues like GitHub. Hatch uses `glab issue relate` which creates a *related* link, not a parent-child hierarchy. The hierarchy lives in issues.json only. PR/MR description checklist (LIGHT mode) works the same on both platforms.

=== STEP 0 — PREFLIGHT (before any reading) ===

Run sub-steps in order. Abort with a clear, recovery-hinted error if any required check fails.

Before Step 0.1: load the cross-platform shell helper library and canonicalize cwd to repo root:

```sh
# Cross-platform helpers (POSIX.1-2024 compliant — works on Linux, macOS BSD, BusyBox, Git Bash WSL).
# Use these instead of bare `date -Iseconds`, `stat -c %Y`, `<( )` process substitution, etc.

iso_ts() { date -u +%Y-%m-%dT%H:%M:%SZ; }
file_mtime() {
  stat -c %Y "$1" 2>/dev/null \
    || stat -f %m "$1" 2>/dev/null \
    || perl -e 'print((stat($ARGV[0]))[9])' "$1" 2>/dev/null \
    || echo 0
}
mk_tmp() { mktemp 2>/dev/null || mktemp -t 'pth.XXXXXX'; }
realpath_p() {
  d=$(dirname -- "$1"); b=$(basename -- "$1")
  ( cd -- "$d" 2>/dev/null && printf '%s/%s\n' "$(pwd -P)" "$b" )
}
is_macos() { [ "$(uname -s)" = "Darwin" ]; }

# Canonicalize cwd to repo root (avoids creating .scratch/ in apps/<sub>/ if user runs from subdirectory).
REPO_ROOT=$(git rev-parse --show-toplevel 2>/dev/null) \
  || { echo "E_NOT_GIT_REPO — Run \`git init\` first or cd into a git repo."; exit 1; }
cd "$REPO_ROOT" || exit 1
```

All subsequent file paths in Step 0+ are relative to `$REPO_ROOT`. The helper functions are referenced where needed (e.g., Step 0.8 uses `iso_ts` and `file_mtime`; Step 5 uses `mk_tmp` for temp file with-body-file pattern).

--- Step 0.1 — Slug acquisition + validation ---

1. Get the slug from me if I did not paste it at the end of this prompt. Use the form: `slug: <kebab>`.
2. Validate slug:
   - Regex: `^[a-z0-9]+(-[a-z0-9]+)*$`
   - Length: 3 to 64 chars
   - Reserved names rejected: all, default, none, new, init, README, INDEX, prompts, _hatch, _detect-skills, archive
   - Trim whitespace; reject empty after trim
   - On failure: emit `E_SLUG_FORMAT — Slug must be kebab-case (lowercase letters, digits, hyphens), 3-64 chars, no leading/trailing/double dashes. Got '<input>'. Try '<auto-suggested fix>'.`

--- Step 0.2 — Idempotency guard with resume detection ---

3. If `.scratch/<slug>/` already exists:
   - Read `.scratch/<slug>/issues.json` if present.
   - If `issues.json` has a `hatch_state` field with `current_step` and `completed_steps`:
     - Print: `Detected existing initiative '<slug>'. Last step completed: <current_step-1>. Sessions wired: <count>.`
     - Offer four options: `(r)esume / (n)ew slug / (c)leanup and start over / (a)bort`.
   - Else (no hatch_state):
     - Treat as orphan; offer `(o)verwrite (DESTRUCTIVE) / (n)ew slug / (a)bort`. Require explicit "yes overwrite" confirmation for `(o)`.
   - Do NOT overwrite silently under any circumstances.

--- Step 0.3 — Save current branch (for restore on abort) ---

4. Record: `ORIGINAL_BRANCH=$(git branch --show-current)`. Save in memory. If a later step fails before push, restore via `git checkout $ORIGINAL_BRANCH`.

--- Step 0.4 — Git environment validation ---

5. Run all of:
   - `git rev-parse --git-dir` — must succeed (we are inside a git repo). Else `E_NOT_GIT_REPO — Run \`git init\` first or cd into a repo.`
   - `test -w .` — must succeed (write access). Else `E_NO_WRITE — No write permission in current directory.`
   - `df -k . | tail -1 | awk '{print $4}'` — must report ≥ 100000 (100MB free). Else `E_DISK_LOW — Low disk space (<100MB free).`
   - `git status --porcelain` — if non-empty, warn (do not abort). Print: `▲ Uncommitted changes in working tree. The hatch will create a new branch but your changes stay on the current branch. Continue? (y/n)`. Default no.

--- Step 0.5 — Skills probe (read .scratch/skills.json) ---

6. Read `.scratch/skills.json`:
   - If file does not exist: ABORT with `E_SKILLS_MISSING — Skill inventory not found. Run .scratch/_detect-skills.md first in a fresh Claude Code conversation. That prompt scans installed skills and writes .scratch/skills.json. Re-run this hatch after.`
   - If file exists but invalid JSON: ABORT with `E_SKILLS_INVALID — .scratch/skills.json is not valid JSON. Re-run .scratch/_detect-skills.md to regenerate.`
   - If `schema_version` != "1.0": ABORT with `E_SKILLS_VERSION — skills.json schema_version <X> not supported (expected 1.0). Re-run .scratch/_detect-skills.md.`
7. Check `classification.required_missing`:
   - If non-empty: ABORT with:
     ```
     E_REQUIRED_MISSING — Required skills are not installed:
       <list each missing skill with installation hint>
     The methodology spine cannot run without these. Install them, re-run _detect-skills.md, then re-run the hatch.
     ```
8. Save the parsed object in memory as `SKILLS`. Subsequent steps reference it.

--- Step 0.6 — Project maturity probe ---

9. Run signal collection (single bash block, all parallelizable):
   ```bash
   FILE_COUNT=$(git ls-files 2>/dev/null | wc -l | tr -d ' ')
   COMMIT_COUNT=$(git rev-list --count HEAD 2>/dev/null || echo 0)
   HAS_CLAUDE=$([ -f CLAUDE.md ] && echo yes || echo no)
   HAS_ARCH=$([ -f ARCHITECTURE.md ] && echo yes || echo no)
   HAS_README=$([ -f README.md ] && echo yes || echo no)
   HAS_PRD=$(find docs -maxdepth 3 \( -iname 'PRD.md' -o -iname 'spec*.md' -o -iname '*proposal*.md' -o -iname 'rfc*.md' \) 2>/dev/null | head -1)
   HAS_CI=$(([ -d .github/workflows ] || [ -f .gitlab-ci.yml ]) && echo yes || echo no)
   MANIFEST=$(ls package.json Cargo.toml pyproject.toml go.mod pom.xml Gemfile foundry.toml 2>/dev/null | head -1)
   PRIOR_SCRATCHES=$(find .scratch -mindepth 1 -maxdepth 1 -type d 2>/dev/null | grep -vE '_hatch|_detect-skills|archive' | wc -l | tr -d ' ')
   CLAUDE_LINES=$([ -f CLAUDE.md ] && wc -l < CLAUDE.md || echo 0)
   ```
10. Classify (precedence order matters):
    - `empty`: `COMMIT_COUNT < 2 && FILE_COUNT < 5`
    - `greenfield`: `FILE_COUNT < 50 && HAS_CLAUDE == no && MANIFEST == ""`
    - `mature`: `HAS_CLAUDE == yes && CLAUDE_LINES >= 30 && PRIOR_SCRATCHES >= 2 && COMMIT_COUNT >= 100`
    - `established`: `HAS_CLAUDE == yes && CLAUDE_LINES >= 30`
    - `early`: fallback (everything else)
11. Save as `MATURITY`. Adaptations apply in subsequent steps:
    - `empty` → 4 questions instead of 10, MINIMAL mode default, skip pattern-research subagent
    - `greenfield` → 8 questions, MINIMAL/LIGHT default, skip "existing files to reuse" sub-Q
    - `early` → 10 questions, LIGHT default, manifest-only reads in Step 1
    - `established` → 10 questions, FULL/LIGHT both reasonable, full reads
    - `mature` → 10 questions + "model after prior initiative?" sub-Q, full reads + prior `.scratch/` index

--- Step 0.7 — Auto-detection (pre-fill answers) ---

12. Run signal collection. Tag each with confidence: HIGH (authoritative source), MEDIUM (heuristic), LOW (best guess).
    ```bash
    # Default branch (3-tier fallback)
    DEFAULT_BRANCH=$(gh repo view --json defaultBranchRef --jq .defaultBranchRef.name 2>/dev/null \
                    || git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@' \
                    || git config init.defaultBranch 2>/dev/null \
                    || echo main)
    DEFAULT_BRANCH_CONFIDENCE=$([ -n "$(gh repo view 2>/dev/null)" ] && echo HIGH || echo MEDIUM)

    # Platform — 3-layer cascade (handles self-hosted GitHub Enterprise + GitLab on any custom domain)
    PLATFORM="none"; PLATFORM_CONFIDENCE="LOW"; PLATFORM_HOST=""; PLATFORM_TIER="free"
    REMOTE_URL=$(git remote get-url origin 2>/dev/null || echo "")
    REMOTE_HOST=$(printf '%s' "$REMOTE_URL" | sed -E 's|^(https?://)?([^@]*@)?([^:/]+).*|\3|')

    # Layer 1 — glab (handles self-hosted GitLab on any custom domain)
    if command -v glab >/dev/null 2>&1; then
      glab_hosts=$(glab auth status 2>&1 | awk '/Logged in to/ {print $4}' | sort -u)
      while IFS= read -r h; do
        [ -z "$h" ] && continue
        if [ "$h" = "$REMOTE_HOST" ]; then
          PLATFORM="gitlab"; PLATFORM_CONFIDENCE="HIGH"; PLATFORM_HOST="$h"; break
        fi
      done <<EOF_GLAB
$glab_hosts
EOF_GLAB
    fi

    # Layer 2 — gh (handles GitHub Enterprise on custom domains)
    if [ "$PLATFORM" = "none" ] && command -v gh >/dev/null 2>&1; then
      gh_hosts=$(gh auth status 2>&1 | awk '/Logged in to/ {print $4}' | sort -u)
      while IFS= read -r h; do
        [ -z "$h" ] && continue
        if [ "$h" = "$REMOTE_HOST" ]; then
          PLATFORM="github"; PLATFORM_CONFIDENCE="HIGH"; PLATFORM_HOST="$h"; break
        fi
      done <<EOF_GH
$gh_hosts
EOF_GH
    fi

    # Layer 3 — URL pattern fallback (no CLI installed, or CLI not authed against this remote)
    if [ "$PLATFORM" = "none" ]; then
      case "$REMOTE_URL" in
        *github.com*)    PLATFORM="github"; PLATFORM_HOST="github.com"; PLATFORM_CONFIDENCE="MEDIUM" ;;
        *gitlab.com*)    PLATFORM="gitlab"; PLATFORM_HOST="gitlab.com"; PLATFORM_CONFIDENCE="MEDIUM" ;;
        *bitbucket.org*) PLATFORM="bitbucket"; PLATFORM_HOST="bitbucket.org"; PLATFORM_CONFIDENCE="MEDIUM" ;;
        "")              PLATFORM="none"; PLATFORM_HOST=""; PLATFORM_CONFIDENCE="HIGH" ;;
        *)               PLATFORM="none"; PLATFORM_HOST="$REMOTE_HOST"; PLATFORM_CONFIDENCE="LOW" ;;
      esac
    fi

    # Layer 1b — GitLab tier probe (only if HIGH confidence GitLab; controls Premium Epic/Work Items features)
    if [ "$PLATFORM" = "gitlab" ] && [ "$PLATFORM_CONFIDENCE" = "HIGH" ]; then
      plan=$(glab api /license --jq .plan 2>/dev/null)
      case "$plan" in
        ultimate|premium) PLATFORM_TIER="$plan" ;;
        *) PLATFORM_TIER="free" ;;
      esac
    fi

    # Tech stack — comprehensive 2026 detection (covers all major manifests)
    STACK=""
    # Node ecosystem (multi-PM aware)
    [ -f package.json ] && STACK="$STACK node"
    # Python ecosystem
    [ -f pyproject.toml ] || [ -f requirements.txt ] || [ -f Pipfile ] || [ -f setup.py ] && STACK="$STACK python"
    # JVM ecosystem (Maven and Gradle, including .kts variants)
    [ -f pom.xml ] || [ -f build.gradle ] || [ -f build.gradle.kts ] || [ -f settings.gradle ] || [ -f settings.gradle.kts ] && STACK="$STACK java"
    # Go
    [ -f go.mod ] && STACK="$STACK go"
    # Rust
    [ -f Cargo.toml ] && STACK="$STACK rust"
    # .NET / C# / F#
    if ls *.csproj *.fsproj *.sln 2>/dev/null | head -1 >/dev/null; then STACK="$STACK dotnet"; fi
    # Ruby
    [ -f Gemfile ] && STACK="$STACK ruby"
    # PHP
    [ -f composer.json ] && STACK="$STACK php"
    # Elixir / Erlang
    [ -f mix.exs ] && STACK="$STACK elixir"
    [ -f rebar.config ] && STACK="$STACK erlang"
    # Swift / iOS
    if [ -f Package.swift ] || ls *.xcodeproj 2>/dev/null | head -1 >/dev/null; then STACK="$STACK swift"; fi
    # Android (Kotlin/Java)
    [ -f settings.gradle ] && [ -d app ] && [ -f app/build.gradle ] && STACK="$STACK android"
    # Flutter
    [ -f pubspec.yaml ] && STACK="$STACK flutter"
    # Smart contracts
    [ -f foundry.toml ] && STACK="$STACK solidity-foundry"
    [ -f hardhat.config.ts ] || [ -f hardhat.config.js ] && STACK="$STACK solidity-hardhat"
    # Infrastructure-as-code
    if ls *.tf 2>/dev/null | head -1 >/dev/null; then STACK="$STACK terraform"; fi
    [ -f terragrunt.hcl ] && STACK="$STACK terragrunt"
    [ -f Pulumi.yaml ] && STACK="$STACK pulumi"
    [ -f cdk.json ] && STACK="$STACK aws-cdk"
    [ -f ansible.cfg ] || [ -f playbook.yml ] || [ -f playbook.yaml ] && STACK="$STACK ansible"
    [ -f Chart.yaml ] && STACK="$STACK helm"
    [ -f kustomization.yaml ] || [ -f kustomization.yml ] && STACK="$STACK kustomize"
    [ -f Dockerfile ] && STACK="$STACK docker"
    # Documentation sites
    [ -f docusaurus.config.js ] || [ -f docusaurus.config.ts ] && STACK="$STACK docusaurus"
    [ -f mkdocs.yml ] && STACK="$STACK mkdocs"
    # Data science / notebooks
    if ls *.ipynb 2>/dev/null | head -1 >/dev/null || ls notebooks/*.ipynb 2>/dev/null | head -1 >/dev/null; then STACK="$STACK jupyter"; fi
    # Trim leading space; if still empty, mark unknown for Q3 follow-up
    STACK=$(echo "$STACK" | sed 's/^ //')
    [ -z "$STACK" ] && STACK="unknown"

    # Monorepo orchestrator detection (if any present, Q3 will ask which workspaces are in scope)
    MONOREPO=""
    [ -f turbo.json ] && MONOREPO="$MONOREPO turborepo"
    [ -f nx.json ] && MONOREPO="$MONOREPO nx"
    [ -f pnpm-workspace.yaml ] && MONOREPO="$MONOREPO pnpm-workspaces"
    [ -f .moon/workspace.yml ] && MONOREPO="$MONOREPO moon"
    [ -f lerna.json ] && MONOREPO="$MONOREPO lerna"
    grep -q '^\[workspace\]' Cargo.toml 2>/dev/null && MONOREPO="$MONOREPO cargo-workspaces"
    [ -f package.json ] && jq -e '.workspaces' package.json >/dev/null 2>&1 && MONOREPO="$MONOREPO yarn-or-npm-workspaces"
    [ -f WORKSPACE ] || [ -f WORKSPACE.bazel ] || [ -f MODULE.bazel ] && MONOREPO="$MONOREPO bazel"
    MONOREPO=$(echo "$MONOREPO" | sed 's/^ //')
    [ -z "$MONOREPO" ] && MONOREPO="single"

    # Test framework hints (used by Q5 verification suggestions; kept for backwards compat)
    TEST_FRAMEWORK=""
    [ -f package.json ] && grep -qE '"vitest"' package.json && TEST_FRAMEWORK="$TEST_FRAMEWORK vitest"
    [ -f package.json ] && grep -qE '"jest"' package.json && TEST_FRAMEWORK="$TEST_FRAMEWORK jest"
    [ -f package.json ] && grep -qE '"playwright"' package.json && TEST_FRAMEWORK="$TEST_FRAMEWORK playwright"
    [ -f foundry.toml ] && TEST_FRAMEWORK="$TEST_FRAMEWORK forge"
    [ -f pytest.ini ] || [ -f pyproject.toml ] && grep -qE 'pytest' pyproject.toml 2>/dev/null && TEST_FRAMEWORK="$TEST_FRAMEWORK pytest"
    [ -f go.mod ] && TEST_FRAMEWORK="$TEST_FRAMEWORK go-test"
    [ -f Cargo.toml ] && TEST_FRAMEWORK="$TEST_FRAMEWORK cargo-test"
    [ -f pom.xml ] && TEST_FRAMEWORK="$TEST_FRAMEWORK maven-surefire"
    [ -f build.gradle ] || [ -f build.gradle.kts ] && TEST_FRAMEWORK="$TEST_FRAMEWORK gradle-test"
    
    # Suggested mode based on maturity + commit count
    case "$MATURITY" in
      empty|greenfield) SUGGESTED_MODE=MINIMAL ;;
      early) SUGGESTED_MODE=LIGHT ;;
      established|mature) SUGGESTED_MODE=FULL ;;
    esac
    ```
13. If `PLATFORM` is `github`, run `gh --version` and `gh auth status` (deferred from Step 0.4 — only relevant once we know platform). If both pass, set `PLATFORM_AUTH=ok`. If gh is missing: `E_GH_MISSING — Install GitHub CLI from https://cli.github.com/ or pick gitlab/none in Q4c.` If unauthenticated: `E_GH_AUTH — Run \`gh auth login\` and re-run.`
14. If `PLATFORM` is `gitlab`, mirror the above with `glab --version` and `glab auth status`.
15. If `PLATFORM` is `none`, skip auth checks.
16. Save all detection results as `DETECT` in memory; will be used to pre-fill Q4b/c/d.

--- Step 0.8 — Multi-dev lock acquisition ---

17. Acquire hatch lock atomically:
    ```bash
    mkdir -p .scratch/<slug>
    LOCK_FILE=.scratch/<slug>/.hatch.lock
    LOCK_PAYLOAD="{\"owner\":\"$USER@$(hostname)\",\"pid\":$$,\"started_at\":\"$(iso_ts)\"}"
    if ! ( set -o noclobber; printf '%s' "$LOCK_PAYLOAD" > "$LOCK_FILE" ) 2>/dev/null; then
      # Lock exists — check staleness (>1h = stale)
      LOCK_AGE=$(($(date +%s) - $(file_mtime "$LOCK_FILE")))
      if [ "$LOCK_AGE" -gt 3600 ]; then
        echo "▲ Stale lock detected (>1h old). Removing and acquiring."
        rm -f "$LOCK_FILE"
        ( set -o noclobber; printf '%s' "$LOCK_PAYLOAD" > "$LOCK_FILE" )
      else
        cat "$LOCK_FILE"
        echo "E_LOCKED — Another hatch is in progress for slug '<slug>'. Wait or remove the lock manually."
        exit 1
      fi
    fi
    trap 'rm -f "$LOCK_FILE"' EXIT INT TERM
    ```

--- Step 0.9 — Print preflight summary ---

18. Emit a compact summary so the user sees what was detected:
    ```
    ━━━ Step 0/6 — Preflight complete ━━━
    Slug:           <slug>
    Original branch: <ORIGINAL_BRANCH>
    Maturity:       <MATURITY> (<reason>)
    Platform:       <PLATFORM> (<PLATFORM_CONFIDENCE>)
    Default branch: <DEFAULT_BRANCH> (<DEFAULT_BRANCH_CONFIDENCE>)
    Stack:          <STACK>
    Skills:         <count> required + <count> recommended (domain breakdown in skills.json)
    Suggested mode: <SUGGESTED_MODE>
    Lock acquired:  .scratch/<slug>/.hatch.lock
    ```

=== STEP 1 — STUDY THE METHODOLOGY (adaptive per maturity) ===

Read these files fully. Adaptations per `MATURITY`:

- `empty`: read only `.scratch/README.md`. Skip everything else.
- `greenfield`: read `.scratch/README.md` and `README.md` if present. Skip CLAUDE.md / ARCHITECTURE.md / PRD reads.
- `early`: read `.scratch/README.md`, `README.md`, manifest scripts (e.g. `package.json` `scripts` field).
- `established`: full reads — `.scratch/README.md`, `.scratch/HOW-TO-USE.md`, `CLAUDE.md`, `ARCHITECTURE.md` if present, project spec doc if present (auto-detect; if multiple, ask user which is relevant before reading).
- `mature`: same as established + index `.scratch/<other-slug>/plan.md` files (read titles only) and surface to user as "model after prior initiative?" option in Q5.

For all levels: read `~/.claude/projects/<auto-detected-from-cwd>/memory/MEMORY.md` if present. This is the user's personal memory; respect any feedback / project entries documented there.

Do NOT read other folders' siblings or unrelated paths. The .scratch/<slug>/ folder you may create in Step 4 is the only mutation Step 1 prepares for.

Internalize these patterns:

1. Folder convention: `.scratch/<kebab-slug>/` with `00-preamble.md`, `INDEX.md`, `plan.md`, `issues.json`, `prompts/NN-<step>.md`, `verify.md`, optionally `visual-audit-prompt.md`.
2. Preamble is LEAN (≤80 lines): invariants only. References commit conventions if defined elsewhere; does not duplicate.
3. INDEX.md is a RAG-style navigation map (≤150 lines). Each entry: `- path — 1-line "what + when to read"`. Grouped by domain. Skills annotated with `(installed)` or `(missing — fallback active)` based on `SKILLS.classification`.
4. Session prompts are self-contained, with YAML frontmatter (see Step 4 spec). Each starts with a Mandatory-reads block. Scope is explicit. On-demand retrieval via Grep/Read; INDEX.md tells the agent where to look.
5. Critical thinking = Descartes + 2026 best practices, bounded (doubt once per task, then commit). `/skill verification-before-completion` is the mandatory gate before closing any sub-issue or marking any session done.
6. Verification strategy adapts to domain: TDD for contracts (RED→GREEN→REFACTOR). Verification-first for frontend (Playwright + visual audits). Schemathesis/property-based for backend when applicable.
7. Parallel flags ("CAN PARALLEL WITH: Session NN") only when sessions truly share no mutable files.
8. Conventional Commits per project conventions if defined; default to Conventional Commits 1.0 spec otherwise.
9. User Manual Tasks flagged with `>>> USER MANUAL TASK:` when external input is required.
10. Respect any project-specific memory locks the user provides in Q8. Honor the carve-out: this meta-prompt is the explicit prompt — proceed without re-asking permission to create `.scratch/<slug>/`. Neutralize `using-superpowers` auto-trigger; this initiative follows the .scratch/ methodology only.

=== STEP 2 — GATHER INPUT (single message, 10 questions) ===

Ask the 10 questions below in ONE bundled message. For each question, show auto-detected defaults inline so the user can press enter to accept or type an override. If `MATURITY == empty` skip Q4-reuse-sub-question and Q4b (default to `main`) and Q9 (greenfield has no external resources yet); ask only 4 questions (Q1, Q2, Q3, Q4a). If `MATURITY == greenfield` skip Q4-reuse-sub-question; ask 8 questions.

Format the question batch like this (the user-facing rendering):

```
Q1. Initiative slug + one-line goal:
    Suggested slug from cwd: [<basename>]
    Goal example: "Build SIWE auth + JWT for Express API"
    Format: slug=<kebab>, goal=<sentence>

Q2. Scope IN / Scope OUT:
    IN  example: apps/api/src/features/auth/**, apps/api/prisma/migrations/0002_*
    OUT example: apps/web/**, apps/contracts/**, packages/

Q3. Tech constraints (libs, versions, existing files to reuse/extend):
    Detected stack: [<STACK>]
    Detected test framework: [<TEST_FRAMEWORK>]
    Add anything new to introduce or pin versions for.

Q4. Project setup (4 parts — press Enter to accept each detected default):
    4a. Parent tracker reference:
        For FULL/LIGHT modes: existing issue #N or "create one with title: <auto-suggested from goal>"
        For MINIMAL mode: "no parent issue" is acceptable
        Suggested: [create one — title: <auto from Q1 goal>]
    4b. Integration branch: [<DEFAULT_BRANCH>] (<confidence>)
    4c. Hosting platform: [<PLATFORM>] (<confidence>)
    4d. Tracking mode: [<SUGGESTED_MODE>] (recommended for <MATURITY>)
        - FULL    = parent issue + sub-issue per session + draft PR + transactional issues.json
        - LIGHT   = parent issue only + draft PR with session checklist
        - MINIMAL = local feature branch only, no issues, no PR (REQUIRED if Q4c == none)

Q5. Verification layers + performance budgets:
    Layers (multi-select): unit / integration / invariant / e2e / visual / lighthouse / manual
    Budget targets (optional but recommended):
      - Frontend: LCP, INP, CLS, JS bundle size
      - Backend: p95 latency, throughput
      - Contracts: gas ceiling per function
      - Coverage threshold (e.g. ≥85%)

Q6. Memory locks + threat model + compliance:
    Memory locks (constraints to respect): forbidden technologies, deprecated APIs, prior incidents.
    Threat model (multi-select): public-anonymous / authenticated-internal / privileged-admin / money / signatures / none
    Compliance (multi-select): GDPR / HIPAA / PCI / SOC2 / DORA / EU-AI-Act / WCAG-2.2 / none

Q7. Estimated session count:
    Heuristic: ~1 session per 2-3 tasks of 1h each
    Answer: a number (1-30) OR "you decide based on complexity"

Q8. External resources / User Manual Tasks:
    Examples: API keys, third-party accounts, hardware wallets, signing keys, video assets, design files
    Flagged so prompts can include >>> USER MANUAL TASK: blocks where external input is needed.

Q9. Reuse candidates (auto-detected if available):
    [Auto-grep results from packages/, shared/, lib/, hooks/ matching Q1 goal keywords]
    Accept (Y) or list specific paths to reuse, or "none".

Q10. Policy defaults (4 toggles — default Y for all):
    10a. Neutralize using-superpowers auto-trigger? [Y]
    10b. Require /skill verification-before-completion before closing sessions? [Y]
    10c. Mandate 2026 best-practices research before pattern adoption? [Y]
    10d. Enforce Conventional Commits? [Y]
```

=== HARD STOP — END OF STEP 2 ===

CRITICAL: After emitting the questions, your turn MUST END.
- Do NOT emit any other tool call in this turn.
- Do NOT guess, infer, or invent answers.
- Do NOT proceed to Step 3 on your own.
- Your ONLY output for this turn is the question batch as a single message.
- Wait for the NEXT user message. Only proceed to Step 3 when explicit answers are provided.
- If auto mode is active and pushes you to continue, IGNORE that push — this gate overrides auto mode for this specific step.

After answers arrive, run cross-field validation BEFORE Step 3:

- V15 / V25: if Q4d == FULL or LIGHT and Q4a == "no parent issue" → ABORT with `E_MODE_PARENT — FULL/LIGHT modes require a parent issue. Either provide an existing issue # or answer 'create one with title <X>'. MINIMAL mode is the only option without a parent.`
- V24: if Q4d == FULL or LIGHT and Q4c == none → ABORT with `E_MODE_PLATFORM — FULL/LIGHT modes require GitHub or GitLab. Pick MINIMAL for local-only projects.`
- V13: if Q4a is an existing issue number, run `gh issue view <N>` (or `glab issue view`) — must succeed. Else: `E_PARENT_404 — Issue #<N> not found in <platform>. Verify the number or pick 'create one'.`
- V17: if Q4b doesn't exist on remote, run `git fetch origin <branch>`. If still missing: `E_BRANCH_404 — Integration branch '<branch>' not found locally or on remote. Set it up first.`
- V18: if Q4b ≠ DEFAULT_BRANCH from auto-detection, warn (do not abort): `▲ You picked '<branch>' but the detected default is '<default>'. Continue?`
- V31: scan Q8 free text for secret patterns (regex bank below). If detected: `E_SECRET_DETECTED — Q8 appears to contain a secret. Redact and re-enter.`
  Secret regexes: `(sk-[A-Za-z0-9]{20,}|ghp_[A-Za-z0-9]{36}|AKIA[0-9A-Z]{16}|gho_[A-Za-z0-9]{36}|0x[a-fA-F0-9]{64}|BEGIN [A-Z]+ PRIVATE KEY|xoxb-[A-Za-z0-9-]{40,}|AIza[0-9A-Za-z_-]{35})`
- V11 (warn): if Q3 contains tech contradictions (react+vue, redux+zustand, ethers+viem, jest+vitest), warn: `▲ Detected potential conflict: <X> and <Y>. Confirm intent.`

Save validated answers as `ANSWERS` in memory.

=== STEP 3 — RESEARCH + PLAN MODE ===

After answers (not before):

1. Enter Plan Mode via `EnterPlanMode` — you will not touch the filesystem yet (lock file + skills.json from Step 0 are exceptions; hatch-state.json updates are allowed since they're operational, not implementation).

2. Spawn parallel research subagents adapted to maturity:
   - `empty` / `greenfield`: 2-3 agents — only web research (current state of stack 2026, security considerations, similar OSS projects). No "study existing patterns" agent (nothing to study).
   - `early`: 4-5 agents — web research + manifest analysis + scripts review.
   - `established` / `mature`: 5-10 agents — full research:
     - Current state of `apps/<target>/` in the repo (what exists, what's missing).
     - 2026 best practices for the specific domain (use WebSearch + context7).
     - Existing patterns in the repo that must be reused (run grep over `packages/`, `shared/`, `lib/`, `hooks/`).
     - Security and performance considerations specific to this initiative.
     - Cross-reference with project specs (Q3 / Q4 / Q6) and CLAUDE.md to stay aligned.
     - For `mature`: also index existing `.scratch/<other-slug>/` plans for structural reference.

3. Compile a ≤400-word research summary highlighting:
   - Patterns to reuse (with paths)
   - 2026 best practices to apply (with sources)
   - Risks identified (per Q6 threat model + compliance)
   - Open questions for the user

3.5. **Compliance regime expansion** (only when Q6 selects regimes; skip entirely if Q6 = `none`).

   For each regime in Q6.compliance, look up the **Compliance regime → session/plan/verify injection map** (defined inside the verify.md spec in this same prompt) and:
   - Add the named sessions to the proposed session list (deduplicated across regimes — `audit-logging` covers SOC2 + HIPAA without duplication).
   - Tag each injected session's frontmatter with `compliance_regimes: [<regime>, ...]` and append `6I` to its `verify_phases`.
   - Reserve a "7a. Compliance regimes" section in `plan.md` listing per-regime obligations and control owners.
   - For sessions that already exist in the user's draft list and overlap regime concerns (e.g. an existing `auth` session under HIPAA), enrich them rather than duplicate: add `compliance_regimes` + `6I` to that session's frontmatter and surface the regime checks in its DONE CRITERIA.

   When Q6.compliance includes `EU-AI-Act`: also gate Step 4 on confirming the project's risk classification (prohibited/high/limited/minimal). High-risk systems (Annex III) MUST inject the FRIA session; minimal-risk projects can opt out of Annex IV via an explicit user override line.

   Multi-regime aggregation: verify.md's PHASE 6I aggregates checks from all selected regimes, runs them in declared order, and refuses partial PASS — every regime check is independently scored.

4. Surface a PROPOSED plan:
   - `plan.md` TOC.
   - `INDEX.md` skeleton (domain groups + skills annotated `(installed)` or `(missing — fallback)` from `SKILLS`).
   - Session list with 1-line goals each, with PREREQUISITE and CAN PARALLEL WITH flags, and proposed verify_phases per session.
   - Verification strategy (tied to Q5 answer).

5. Call `ExitPlanMode` only after explicit approval (e.g. "approved", "go", "lgtm", or corrections then approval). Do NOT auto-exit Plan Mode.

=== HARD STOP — END OF STEP 3 ===

CRITICAL: Inside Plan Mode, after surfacing the proposed plan, your turn MUST END.
- Do NOT call `ExitPlanMode` until I explicitly approve.
- Do NOT pre-write any files in anticipation of approval.
- Do NOT proceed to Step 4 on your own.
- Wait for explicit "approved" / "go" / "lgtm" message.
- If corrections come, revise the plan inside Plan Mode and stop again. Repeat until approval.

=== STEP 4 — GENERATE SCAFFOLDING ===

On approval, create the scaffolding tree. Write `INDEX.md` and `00-preamble.md` BEFORE session prompts so each session prompt references them correctly.

```
.scratch/<slug>/
├── 00-preamble.md       (≤80 lines — invariants, branch info, critical thinking, skills, User Manual markers)
├── INDEX.md             (≤150 lines — RAG-style navigation map, skills annotated)
├── plan.md              (full plan, see structure below)
├── issues.json          (skeleton with hatch_state; populated in Step 5)
└── prompts/
    ├── 01-<step>.md     (with YAML frontmatter v2)
    ├── 02-<step>.md
    ├── NN-<step>.md
    ├── verify.md        (generic post-session verification prompt)
    └── visual-audit-prompt.md     (ONLY if Q5 includes "visual")
```

After each successful file write, atomically update `hatch_state.scaffolding` in `issues.json`. Use atomic write protocol: write to `.tmp`, validate, `mv -f`.

--- 00-preamble.md template (≤80 lines) ---

MUST include in this order:

1. Initiative goal + one-line acceptance criteria.
2. Mode declaration: "Tracking mode: <FULL|LIGHT|MINIMAL>. Platform: <github|gitlab|none>."
3. Branch info (mode-aware):
   - FULL/LIGHT: `Branch: feat/<parent>-<slug> created FROM <integration_branch>. PR target: <integration_branch>. NEVER merge to main automatically — that is always a manual human decision.`
   - MINIMAL: `Branch: feat/<slug> (LOCAL only — hatch does not push). NEVER merge to main automatically.`
4. Critical thinking block: Descartes method bounded (doubt once per task, then commit). **Mission reminder (verbatim):** *"Ship only work whose correctness is demonstrable with re-runnable evidence — not work you believe is correct. Nothing breaks existing behavior. Nothing half-finished. Every non-trivial pattern decision verified against 2026 canonical sources via WebSearch + context7 before adoption."*
5. Pattern-deviation rule: "Before applying a different pattern than the plan, critically verify it matches 2026 best practices for THIS project via WebSearch + context7. If yes, apply and briefly explain why in the commit message. If uncertain, default to the plan."
6. `/skill verification-before-completion` mandatory before closing any session. (Note: this skill is verified present in skills.json; if missing, the hatch aborted in Step 0.7.)
7. `/skill systematic-debugging` for any bug/test failure before proposing fixes.
8. `using-superpowers` auto-trigger neutralization: "Follow only the .scratch/ methodology in plan.md. Bundled skills are tools, not orchestrators."
9. Reference to project commit conventions; default to Conventional Commits 1.0.
10. User Manual Task marker syntax: `>>> USER MANUAL TASK:`.
11. Closure ritual per session, mode-aware:
    - FULL: `gh issue close <session-issue> --comment "Closed via <SHA>; verify report: <path>"` (or `glab issue close`).
    - LIGHT: edit parent PR description to tick the corresponding session checkbox: `gh pr edit <PR> --body-file <body>` (or `glab mr edit`).
    - MINIMAL: update `issues.json` — set the session entry status to `"done"` and record SHA + verify report path.
12. Skills annotation: list skills available + skills missing (with their fallback strategy from skills.json).

--- INDEX.md template (≤150 lines) ---

MUST group entries by domain. Format each line as:
`- <path> — 1-line what + when to read`

Domains (pick those that apply per Q3 stack):
- Architecture (CLAUDE.md, ARCHITECTURE.md, plan.md, 00-preamble.md)
- Frontend (apps/web/* if exists)
- Backend (apps/api/* if exists)
- Contracts (apps/contracts/* if exists)
- Shared packages (packages/* if exists)
- Infra (docker/, .github/, ci configs)
- Skills (per-domain list — annotate with `(installed)` or `(missing — fallback: WebSearch+context7 query "<X>")`)

No prose, no duplicated rules. Pure navigation map.

--- plan.md structure (full plan, ~250-500 lines) ---

Sections in order:
1. Context (why this initiative now)
2. Tool audit (Already installed / Add / Drop / Bump)
3. Prerequisites (what must exist before Session 01)
4. Architecture summary (with ASCII diagrams for non-trivial flows)
5. Domain model (table with constraints, types, relationships)
6. Session-by-session workflow:
   - Per-session table: NN | title | goal | PREREQUISITE | CAN PARALLEL WITH | verify_phases | session_kind | estimated_minutes | compliance_regimes (only column populated when Q6 ≠ none)
7. Prompt strategy (how prompts coordinate)
7a. **Compliance regimes** (ONLY emit this section when Q6 ≠ none): per-regime obligations table, control-owner per criterion, regime → injected session mapping, evidence directory layout (`.evidence/<regime>/<criterion>/`).
8. Verification strategy (per-session phase enablement map; ties to Q5; explicitly notes when 6I is enabled)
9. Execution summary (total sessions, parallel opportunities, estimated total time)
10. Key files (quick path reference)
11. Sources (research URLs from Step 3)

--- Session prompt template v2 (each NN-<step>.md, target 80-180 lines) ---

MUST start with YAML frontmatter:

```yaml
---
session: NN
slug: <kebab>
mode: FULL | LIGHT | MINIMAL
platform: github | gitlab | none
prerequisite: [NN, NN]
parallel_with: [NN, NN]
estimated_minutes: 60
verify_phases: [1, 2, 3, 6E]
session_kind: scaffold | feature | tdd | visual | indexer | deploy | compliance
compliance_regimes: []   # optional; populate only when Q6 selected regimes (e.g. [SOC2, HIPAA]). Drives PHASE 6I.
touches:
  - apps/api/src/features/x/**
do_not_touch:
  - apps/web/**
---
```

Then the body sections in order:

1. **MANDATORY READS** (≤8 lines, ordered): 00-preamble.md, INDEX.md, anchored plan.md sections (use § not line numbers), 1-3 source files current state.

2. **SUB-ISSUE / SESSION LOOKUP** (mode-aware):
   - FULL/LIGHT (github): `N=$(jq -r '.sessions["NN"].issue' .scratch/<slug>/issues.json) && gh issue view $N`
   - FULL/LIGHT (gitlab): `N=$(jq -r '.sessions["NN"].issue' .scratch/<slug>/issues.json) && glab issue view $N`
   - MINIMAL: `jq '.sessions["NN"]' .scratch/<slug>/issues.json`

3. **SESSION GOAL** (1 declarative outcome sentence — what working software exists at the end).

4. **COORDINATES**: PREREQUISITE / CAN PARALLEL WITH (re-stated from frontmatter for human readability).

5. **USER MANUAL TASK** (omit block if none): trigger + exact commands + observable confirmation signal.

6. **SCOPE**: TOUCH / CREATE / DELETE / DO NOT TOUCH / on-demand retrieval note.

7. **RESEARCH** (3-6 bullets, dated topics): "today is <Month YYYY>", session-specific 2026 topics for WebSearch + context7. If a relevant skill is missing per skills.json, inject the inline fallback query.

8. **TASKS**: numbered, format `N. VERB(file): intent` + sub-bullets for deliverables. For TDD sessions: explicit RED → GREEN → REFACTOR phasing.

9. **COMMITS**: suggested-not-prescribed Conventional Commits list (3-10 bullets). LLM may fold/split.

10. **DONE CRITERIA**: runnable commands + expected output (NOT prose). Mode-aware closure ritual at the end.

11. **HARD STOPS** (optional, only when one exists): condition + action.

--- verify.md spec (generic post-session auditor, ~450 lines) ---

ROLE DECLARATION (top, mission-first):
"You are a quality auditor and code reviewer. Your mandate is uncompromising: verify that the session's work is PRODUCTION-GRADE — nothing broken, nothing half-finished, no untested edge cases, no 'probably works' shortcuts, no hidden bugs. Every PASS verdict you issue must be backed by re-runnable evidence (exit_code + stdout from concrete commands). You do NOT write or modify code. You ONLY analyze, test, and report. If you find issues, report them with evidence — do not fix them. When 2026 best-practice authority is unclear, WebSearch + context7 is MANDATORY before issuing a verdict — never rely on training-data recall alone for any non-trivial pattern judgment."

TOOL ALLOWLIST (explicit, belt-and-suspenders):
```
ALLOWED TOOLS:
  - Read, Grep, Glob
  - Bash (READ-ONLY commands only): ls, cat, file, stat, find, head, tail, wc, du, df,
    git log, git diff, git status, git show, git ls-files, git rev-parse, git branch,
    forge test, forge build (without --broadcast), pnpm test, pnpm build, pnpm lint, pnpm typecheck,
    npm test, npm run build, cargo test, go test, pytest,
    docker compose ps, docker compose config, docker compose logs,
    curl GET, wget --spider,
    gh issue view, gh pr view, gh api GET endpoints,
    glab issue view, glab mr view, glab api GET endpoints,
    jq, yq, awk, sed -n, grep
  - WebFetch, WebSearch
  - Skill invocations (read-only)

FORBIDDEN TOOLS:
  - Edit, Write (except for the report file path), MultiEdit, NotebookEdit, ApplyPatch
  - Bash mutations: git commit, git push, git reset, git checkout -b, git rebase, git stash,
    forge script --broadcast, prisma migrate, docker compose up (without --no-start),
    docker rm, docker volume rm, rm, mv, mkdir, chmod, chown,
    pnpm install/add/remove, npm install, cargo add, go get, pip install,
    gh issue close/edit/comment/create/delete, gh pr create/edit/merge/close,
    glab issue close/edit/create, glab mr create/edit/merge,
    cast send, cast call --rpc-url, sudo
```

If the verifier needs to write findings, the ONLY allowed write target is `.scratch/<slug>/verification-report-<ISO-date>-<session>.md`.

MANDATORY READS: CLAUDE.md (if present), ARCHITECTURE.md (if present), `.scratch/<slug>/plan.md`, `.scratch/<slug>/00-preamble.md`, `.scratch/<slug>/INDEX.md`, `.scratch/<slug>/issues.json`, `.scratch/skills.json`.

RESEARCH DIRECTIVE: use WebSearch + context7 for 2026 best practices of the stack in scope (derived from INDEX.md and plan.md). If a skill is annotated as missing in skills.json, use the inline fallback query.

SKILL INVOCATIONS: `/skill verification-before-completion`, `/skill systematic-debugging`, plus domain-specific skills marked `(installed)` in INDEX.md.

PHASE 0 — SCOPE DETECTION (multi-signal, gates everything):

The available signals depend on the tracking mode (read mode from issues.json):

```
SIGNAL A — git log (always): `git log --oneline -30` + `git diff --name-only origin/<integration_branch>...HEAD`
SIGNAL B — branch (always): `git branch --show-current` (must be `feat/<parent>-<slug>` for FULL/LIGHT, or `feat/<slug>` for MINIMAL)
SIGNAL C — issues.json (always): read .scratch/<slug>/issues.json, find sessions with status "in-progress" or last "pending" before any "done"
SIGNAL D — PR/MR body (FULL or LIGHT only): check which session checkboxes are marked
SIGNAL E — frontmatter (always, deterministic):
  Read the frontmatter of `.scratch/<slug>/prompts/NN-<step>.md` (the candidate session).
  If `touches` ⊇ `git diff --name-only origin/<integration_branch>...HEAD` AND `verify_phases` is non-empty
  → SIGNAL E confirms session NN with confidence: high.

DECISION RULE:
  - FULL/LIGHT: if E + B + C agree → confidence: high regardless of D. If E disagrees with B/C → ABORT.
                If E unavailable (older session prompt without frontmatter), fall back to A/B/C/D majority rule (3 of 4 = medium).
  - MINIMAL: A + C + E must agree. If E disagrees with A or C → ABORT.
  - Genuine disagreement → ABORT. Print all signals. Ask "which session should I verify?".
  - Do NOT guess. Never proceed past PHASE 0 if signals conflict.
```

After detection: read `.scratch/<slug>/prompts/NN-<step>.md`. Use `verify_phases` from frontmatter as the phase pack. Report: "Detected session NN (confidence: high/medium). Mode: <mode>. Running phases: <list from frontmatter>."

PHASES 1..N (run only those in scope per Phase 0 + frontmatter `verify_phases`):

- **PHASE 1 — BUILD & TOOLCHAIN (stack-keyed lookup, 2026-canonical commands):**

  Detect stack from `.scratch/skills.json` preflight + manifest probe in Step 0.7. Run the matching commands per stack. Each must exit 0 to PASS; non-zero exit = FAIL with severity from §9 SEVERITY LEVELS.

  | Stack | Build / typecheck / lint / test commands |
  |---|---|
  | `node` | `pnpm build && pnpm typecheck && pnpm lint && pnpm test` (or `npm`/`yarn`/`bun` equivalent — detect via lockfile). For Biome users: `biome check --error`. For monorepos: `pnpm -r ...` or `turbo run build typecheck lint test`. |
  | `python` | `uv sync --frozen && uv run pytest --cov --cov-fail-under=85 && uv run mypy --strict <pkg> && uv run ruff check && uv run ruff format --check`. Fallback to `pip-tools` / `poetry` if no `uv.lock`. |
  | `java` (Maven) | `./mvnw clean verify -B` (runs compile + test + SpotBugs + JaCoCo). Coverage gate via JaCoCo `<rule>` in pom.xml. |
  | `java` (Gradle) | `./gradlew check -i` (runs compile + test + SpotBugs + JaCoCo). |
  | `go` | `go build ./... && go test -race ./... && go vet ./... && golangci-lint run`. Add `go test -fuzz=Fuzz -fuzztime=10s` per fuzz target on changed parsers. |
  | `rust` | `cargo build --all-targets --locked && cargo test --all-features && cargo clippy --all-targets -- -D warnings && cargo fmt --check`. Add `cargo llvm-cov --fail-under-lines 80` for coverage. |
  | `dotnet` | `dotnet build --no-restore && dotnet test --no-build && dotnet format --verify-no-changes`. |
  | `swift` (SPM) | `swift build && swift test && swift-format lint --strict --recursive Sources Tests`. |
  | `swift` (iOS) | `xcodebuild test -scheme <Scheme> -destination "platform=iOS Simulator,name=iPhone 15" && SwiftLint --strict`. |
  | `ruby` | `bundle exec rspec && bundle exec rubocop && bundle exec brakeman --no-pager` (Rails). |
  | `php` | `composer install --no-interaction && vendor/bin/phpunit && vendor/bin/phpstan analyse && vendor/bin/php-cs-fixer fix --dry-run`. |
  | `elixir` | `mix deps.get && mix test && mix dialyzer && mix credo --strict && mix format --check-formatted`. |
  | `solidity-foundry` | `forge build --sizes && forge test --fuzz-runs 1000 && forge fmt --check && solhint 'src/**/*.sol' && forge coverage --report summary --fail-under-lines 90`. |
  | `solidity-hardhat` | `npx hardhat compile && npx hardhat test && npx hardhat coverage`. |
  | `terraform` | `terraform fmt -check -recursive && terraform validate && tflint --recursive && trivy config . --severity HIGH,CRITICAL`. |
  | `ansible` | `ansible-playbook --syntax-check playbook.yml && ansible-lint --profile production && yamllint .`. |
  | `flutter` | `flutter pub get && flutter analyze && flutter test --coverage && dart format --output=none --set-exit-if-changed .`. |
  | `react-native` | `npx expo lint && npx jest --coverage`. |
  | `jupyter` | `jupyter nbconvert --to notebook --execute notebooks/*.ipynb --inplace && nbqa ruff notebooks/`. |
  | `unknown` | ABORT PHASE 1 with REFUSE_NO_EVIDENCE: ask user to specify build/test/lint commands explicitly via plan.md "Tool audit" section. |

  Sources verified for 2026 canonical commands: pnpm.io/cli, uv (Astral, OpenAI), maven.apache.org, gradle.org, go.dev/security/fuzz, doc.rust-lang.org/cargo, dotnet docs, hardhat.org, getfoundry.sh, terraform.io.

- **PHASE 2 — GIT HYGIENE:** Conventional Commits compliance, no `git add -A` patterns, no secrets in diff (rerun secret regex bank), all commits on initiative's branch, no commits to wrong branch.
- **PHASE 3 — TESTS:** language-appropriate test runner (per PHASE 1 table), zero failures, coverage delta if measurable.
- **PHASE 4 — SECURITY / STATIC ANALYSIS (stack-keyed matrix, 2026-canonical tools):**

  Detect stack and run all matching tools. CRITICAL findings block merge.

  | Stack | Dependency audit | SAST | Secret scan |
  |---|---|---|---|
  | `node` | `pnpm audit --audit-level=high` (or `npm audit`); add `npx audit-ci` for CI gating | `semgrep --config=p/typescript --error` | `gitleaks detect --source . --no-git --redact` |
  | `python` | `pip-audit --strict` (replaces `safety` since 2024) | `bandit -r . -ll && semgrep --config=p/python --error` | gitleaks |
  | `java` | `./mvnw org.owasp:dependency-check-maven:check` (or Gradle plugin); `./mvnw spotbugs:check` is enforced in PHASE 1 | `semgrep --config=p/java` | gitleaks |
  | `go` | `govulncheck ./...` (official Go tooling) | `gosec ./...` | gitleaks |
  | `rust` | `cargo audit --deny warnings && cargo deny check` | `cargo-geiger` for unsafe code (advisory) | gitleaks |
  | `dotnet` | `dotnet list package --vulnerable --include-transitive` | `security-code-scan` analyzer | gitleaks |
  | `swift` | (none canonical for SPM); `pod outdated` for CocoaPods | SwiftLint security rules | gitleaks |
  | `solidity` (any) | (no traditional dep audit) | `slither . && aderyn . && solhint 'src/**/*.sol' && semgrep --config=p/solidity` | gitleaks |
  | `terraform` | (no dep audit; provider versions pinned in code) | `trivy config . --severity HIGH,CRITICAL` (replaces `tfsec` since 2023 merge); add `checkov -d .` | gitleaks |
  | `ansible` | (none canonical) | `ansible-lint --profile production` | gitleaks |
  | `docker` | `trivy image <image>:<tag>` | `trivy config Dockerfile` | gitleaks |
  | `any` (always run) | — | — | `gitleaks detect --source . --no-git --redact && trufflehog filesystem .` |

  Tool deprecations to flag (2026):
  - `safety` → use `pip-audit` (PyPA-blessed since 2024).
  - `tfsec` → merged into `trivy config` (Aqua, 2023).
  - `mythx` SaaS → shut down March 2026; local `mythril` still works.
  - `flake8` + `black` + `isort` → replaced by `ruff check` + `ruff format`.
  - `tslint` → ESLint with `@typescript-eslint` (settled 2019).
  - `nancy` → `govulncheck` (official, reachability-aware).

  Severity normalization: every tool emits SARIF 2.1.0 in 2026; the verifier parses SARIF for unified severity. Exceptions (`pnpm audit`, `cargo deny`, `dotnet list package --vulnerable`, `cosign`) need thin JSON adapters — note in finding's `authority` field which adapter was used.

  CRITICAL definition (cross-stack): RCE, authz bypass, on-chain value loss (Solidity), verified live secret committed, supply-chain malicious release match, CVSS ≥ 9 with proven reachability, public-bucket / open-SG in IaC prod.

- **PHASE 5 — DOMAIN-SPECIFIC** (stack-keyed recipes per plan.md verification strategy):

  | Stack | 2026-canonical PHASE 5 recipes |
  |---|---|
  | Frontend (React/Vue/Angular/Svelte) | Playwright `toHaveScreenshot` baseline diff (`maxDiffPixelRatio: 0.01`), Lighthouse CI (Perf ≥ Q5 budget, A11y ≥ 95, BP ≥ 95, SEO ≥ 95), axe-core via `@axe-core/playwright` (zero serious/critical), WCAG 2.2 explicit checks (focus appearance 2.4.11/13, target size 24×24 SC 2.5.8, dragging 2.5.7), Storybook + Chromatic visual regression. |
  | Backend Python (FastAPI/Django) | Schemathesis 3.x `--checks all` against `/openapi.json`, alembic upgrade/downgrade round-trip on ephemeral pytest-postgresql, Hypothesis property tests on domain invariants, mypy --strict on changed files. |
  | Backend Node (Express/Fastify) | testcontainers-node Postgres/Redis, supertest for HTTP integration, Pact-JS consumer-driven contracts, `prisma migrate diff --exit-code` for schema drift. |
  | Backend Java (Spring Boot) | Testcontainers `@ServiceConnection`, Spring Boot test slices (`@WebMvcTest`, `@DataJpaTest`), ArchUnit hexagonal/onion boundary rules, Pact JVM provider verification, WireMock external stubs. |
  | Backend Go | `go test -race`, `go test -fuzz=Fuzz -fuzztime=60s` on parsers, testcontainers-go, `buf breaking` for protobuf change control. |
  | Backend Rust | proptest/quickcheck, `cargo miri test` on pure logic crates (UB detection), criterion bench baseline diff, `cargo-mutants` ≥ 75% mutation kill. |
  | Smart contracts (Foundry) | Invariants `runs=256, depth=50`, fuzz 10k, Halmos symbolic on `check_*` properties, Echidna assertion mode, `forge snapshot --check` vs gas budget, `forge mutate` ≥ 75% kill, ERC compliance suites (a16z erc20-tests, etc.). |
  | Indexer | Reorg resistance test (3-block reorg fork), idempotency on double-ingest (replay event log twice → identical DB state), cursor persistence (kill mid-batch, restart, no gaps/double-counts). |
  | Mobile (iOS Swift) | XCTest + Swift Testing macros, XCUITest with accessibility identifiers only, Pointfree swift-snapshot-testing, Instruments perf budget, accessibility audit zero issues. |
  | Mobile (Android Kotlin) | JUnit 5 + MockK, Espresso E2E, accessibility-test-framework, Macrobenchmark startup/frame timing. |
  | Mobile (Flutter) | `flutter test` widget + `integration_test`, `golden_toolkit` for cross-platform goldens. |
  | Terraform / IaC | `terraform plan -detailed-exitcode -out=plan.tfplan` (3=changes gate), Terratest Go integration, OPA/Conftest rego policies (region allowlist, mandatory tags, encryption-at-rest), `infracost diff` vs main. |
  | Kubernetes | `kubectl apply --dry-run=server`, `kubeconform` schema check, `conftest test` rego policies (no `latest` tag, resources/limits required, run-as-non-root), `helm lint`. |
  | Data science (Jupyter) | `nbconvert --execute` with deterministic seed, `nbqa ruff`, papermill parameterized re-runs, DVC dataset version pin verified, output assertions (`assert metrics["f1"] >= 0.85`). |

  Universal cross-stack 2026 best practices:
  - Testcontainers everywhere — replaces in-memory fakes (H2, sqlite-in-memory) which lie about behavior.
  - Schemathesis for OpenAPI contract testing — runs from any stack via Docker image.
  - Pact for consumer-driven contracts — multi-language broker.
  - OPA / Conftest for policy-as-code — same Rego rules across IaC + K8s + Dockerfiles + CI configs.
  - SBOM + provenance — `syft` / `cyclonedx` SBOM + SLSA Level 3 via GitHub Actions OIDC + `sigstore/cosign`.

  **Compliance regime → session/plan/verify injection map** (used by Step 3.5 when Q6 ≠ none; consumed by PHASE 6I below):

  | Regime | Inject session(s) | plan.md "7a" content | verify 6I checks (machine-verifiable) |
  |---|---|---|---|
  | **GDPR** (Reg. 2016/679; Articles 15-21 DSR, Art. 30 records, Art. 35 DPIA, Arts. 44-49 transfers) | `data-subject-rights` (export/erase/rectify endpoints), `retention-policy` (TTL jobs + verifier), `processor-agreements` (DPA registry) | DPIA outcome, lawful basis per processing activity, transfer mechanism (SCCs/adequacy), DPO contact, retention table | Endpoints `GET/DELETE /api/v*/me` exist, retention TTL job present in scheduler config, `gdpr.yml` records-of-processing registry committed, no plaintext PII in logs (`grep -E 'email\|ssn\|tax_id' logs/`), data-residency assertion in deploy config |
  | **HIPAA Security Rule** (45 CFR §164.308 admin / §164.310 physical / §164.312 technical / §164.312(b) audit) | `baa-review` (sub-processor inventory + signed BAAs), `phi-encryption` (TDE/client-side at-rest + TLS 1.3 in-transit), `audit-logging` (full audit trail) | Administrative/physical/technical safeguards table per §164.308–.312, BAA inventory | No PHI in error messages (custom error mapper present), TLS 1.3 enforced (no TLS 1.2 fallback), audit table schema present (timestamp, actor, action, resource), encryption-at-rest verified (DB has TDE flag or app-level field-level encryption), break-glass access logged |
  | **PCI-DSS v4.0** (PCI SSC, in force 2024-04-01, future-dated reqs 2025-03-31) | `chd-isolation` (cardholder data env on separate VPC/subnet), `key-rotation` (KMS rotation jobs + version pinning), `tokenization` (PAN never reaches app DB), `network-segmentation` (firewall rules + flow logs) | Req 3 (data protection), Req 4 (transit), Req 6 (secure dev), Req 8 (auth), Req 10 (logging), Req 11 (testing) | No PAN in DB schema (`grep -i 'pan\|card_number' schema.sql`), tokenization library in deps, KMS rotation cron exists, MFA enforced in auth config, ASV scan in CI, flow logs enabled in IaC |
  | **SOC2** (AICPA TSC 2017 + 2022 points of focus) | `audit-logging` (immutable, append-only log of admin/data actions), `access-control-review` (IAM matrix + quarterly review job), `change-management` (PR template enforces evidence, linked tickets, approver) | CC1-9 + A/C/PI/P criteria selection, control owner per criterion | Audit log table append-only (no UPDATE/DELETE grants in DB role), `.github/PULL_REQUEST_TEMPLATE.md` has approval-evidence field, IAM review script exists, `.evidence/<criterion>/` directory populated |
  | **DORA** (Reg. EU 2022/2554, in force 2025-01-17, RTS published 2024-12) | `ict-third-party-risk` (registry of ICT providers, criticality tiering), `operational-resilience-testing` (TLPT/threat-led pen test plan), `incident-classification-reporting` (RTS-aligned thresholds + 4h major-incident notification) | DORA Articles 5-23 mapping — ICT risk framework, incident management policy, digital ops resilience testing program, third-party register | ICT-providers JSON registry present + populated, incident-classification policy doc exists, RTO/RPO targets in deploy config, chaos test in CI (`gremlin`/`litmus`/`chaos-mesh`), notification webhook to authority configured |
  | **EU AI Act** (Reg. EU 2024/1689, full enforcement 2026-08-02 high-risk) | `ai-system-risk-classification` (Annex III mapping, Article 6 high-risk decision), `ai-transparency` (Article 50 disclosures + watermarking for GenAI), `ai-model-documentation` (Annex IV technical documentation set), `fria` (Fundamental Rights Impact Assessment, only for high-risk Annex III systems) | Risk classification (prohibited/high/limited/minimal), Annex IV documentation set, post-market monitoring plan | Risk-classification doc present, Annex IV files in `docs/ai-act/`, GenAI output watermark library imported (e.g., C2PA/`@anthropic/genai-watermark`), human-oversight UI flag present, FRIA committed for high-risk systems |
  | **WCAG 2.2 AA** (W3C Recommendation 2023-10-05; 9 new criteria over 2.1) | `accessibility-audit` (axe + manual keyboard pass + reduced-motion check), `focus-appearance-2.4.11` (custom focus ring meeting 3:1 contrast), `target-size-2.5.8` (≥24×24 CSS pixels for pointer targets) | VPAT-style table per criterion, exceptions documented, ATAG if authoring tool | `@axe-core/playwright` zero serious/critical findings, focus ring contrast measured (custom Playwright assertion), all interactive targets ≥24×24, `prefers-reduced-motion` honored, dragging 2.5.7 single-pointer alternative present, accessible-authentication 3.3.8 (no cognitive function tests without alternative) |

  Multi-regime aggregation: when Q6 selects multiple regimes, sessions are deduplicated by name (e.g., `audit-logging` covers SOC2 + HIPAA; emit once with `compliance_regimes: [SOC2, HIPAA]`). PHASE 6I aggregates all selected regimes' checks and refuses partial PASS — every regime check is independently scored.

- **PHASE 6 — QUALITATIVE REVIEW (LLM-as-judge, evidence-based):**

  Tests passing ≠ code is good. For EVERY material code change this session, emit findings grounded in code + external authority.

  - **6A — Correctness trace** (per new/modified function): file:line + quoted snippet + 1-sentence trace of intent vs goal + verdict.
  - **6B — 2026 best-practices cross-check** (WebSearch + context7 mandatory, or fallback per skills.json): pattern + authority URL/skill + verdict ALIGNED/DRIFT/ANTI-PATTERN.
  - **6C — Test quality**: behavior-vs-implementation, tautology check, NOT-tested list, fuzz/property coverage.
  - **6D — Hidden bugs / footgun scan**: edge cases, unchecked external calls, silent failures, memory leaks, async ordering.
  - **6E — Pattern cohesion**: matches existing repo patterns? Fights ARCHITECTURE.md? Premature abstraction?
  - **6F — End-to-end real behavior** (when feasible): Playwright drive, curl with realistic payloads, deploy to local node dry-run.
  - **6G — Security pass** (auth/money/signatures/storage code): vulnhunter + code-recon, vulnerability classes enumerated.
  - **6I — Regulatory compliance pass** (gated: only runs when session frontmatter `compliance_regimes` is non-empty): for each regime listed, execute the regime's machine-verifiable checks from the **Compliance regime → session/plan/verify injection map** above. Zero tolerance for CRITICAL findings — regulatory exposure (fines, audit failure, certification loss) is treated equivalent to funds-at-risk in 6G. Per-regime PASS/FAIL is reported independently; an overall 6I PASS requires every regime to PASS. Evidence file path: `.evidence/<regime>/<check-name>` (created by injected sessions; read-only here). 6I findings inherit the standard schema (severity emoji + sub_phase + file:line + code_snippet + concern + authority + recommended_action) plus an additional `regime: <GDPR|HIPAA|PCI|SOC2|DORA|EU-AI-Act|WCAG-2.2>` field. Distinct from 6G: 6G prevents exploits; 6I prevents fines.
  - **6H — Re-execution verification** (NEW, anti-hallucination):
    - For every PASS finding from Phases 1-5 that cited a command + stdout: RE-RUN that command. Compare exit_code and stdout_tail.
    - For every Phase 6B finding citing a URL: RE-FETCH the URL, confirm the quote still matches.
    - Stamp every finding with `re_executed_at` and `still_reproduces: true|false`.
    - If `still_reproduces: false` (drift detected): downgrade severity by one tier and add note `DRIFT: original output no longer reproduces; check at <new-timestamp>`.
    - If a finding's evidence cannot be re-executed (command moved/file deleted), flip its verdict to `REFUSE_NO_EVIDENCE` and treat as FAIL.

  Phase 6 finding schema (appended to verification-report file):
  ```yaml
  - finding: "<1-line title>"
    severity: 🔴 CRITICAL | 🟠 MAJOR | 🟡 MINOR | 🔵 INFO | ⚪ NIT
    sub_phase: 6A | 6B | 6C | 6D | 6E | 6F | 6G | 6H | 6I
    regime: GDPR | HIPAA | PCI | SOC2 | DORA | EU-AI-Act | WCAG-2.2  # only present when sub_phase == 6I
    file: <path>
    line_range: <start>-<end>
    code_snippet: |
      <quoted, 1-10 lines>
    concern: <plain language why it matters>
    authority:
      - <URL or skill name backing the concern, especially for 6B>
    recommended_action: <specific, file-path-scoped>
    verification:
      re_executed_at: <ISO timestamp>
      still_reproduces: <true|false>
      cited_command: <command that produced original evidence>
    github_annotation: # optional, if PR exists
      annotation_level: failure | warning | notice
      title: <short title>
  ```

  If a sub-phase finds nothing, state explicitly: `6B — 2026 best-practices cross-check: ALIGNED (N patterns checked, all match current 2026 guidance per [sources])`. Do not omit sub-phases silently. **6I exception:** if the session has no `compliance_regimes` set, emit `6I — Regulatory compliance: SKIPPED (session frontmatter has no compliance_regimes)` exactly once. Do not infer regimes the user did not declare.

- **PHASE 7 — DONE CRITERIA CHECK:** open `prompts/NN-<step>.md`, read DONE CRITERIA, confirm each item with evidence from earlier phases. Every item must cite a specific phase result.

EVIDENCE SCHEMA (mandatory for every check, every phase):
```yaml
- check: <1-line name>
  command: <exact shell command, or tool call>
  exit_code: <integer, or "n/a" for tool calls>
  stdout_tail: <last 5-10 lines, quoted literally>
  verdict: PASS | FAIL | SKIP | REFUSE_NO_EVIDENCE (with reason)
  severity: 🔴 CRITICAL | 🟠 MAJOR | 🟡 MINOR (only if FAIL)
  evidence_file: <path:line if applicable>
```
No narrative claims. No "looks good". No mental simulation. If exit_code and stdout_tail cannot be produced, verdict is REFUSE_NO_EVIDENCE.

SEVERITY LEVELS (canonical 2026):
- 🔴 CRITICAL — Funds at risk, exploit, page broken, secrets committed, build broken, tests fail, security issue.
- 🟠 MAJOR — Usability failure, performance regression, accessibility barrier, design-system violation, contract cap bypass, missing revert path.
- 🟡 MINOR — Code quality, bundle size, missing test, non-blocking polish.
- 🔵 INFO — Observation, not a defect.
- ⚪ NIT — Style preference, no functional impact.

WHAT NOT TO FLAG (anti-nit-picking, per Cloudflare 2026 pattern):
- Code style outside the session's scope.
- Issues that exist before this session's commits (pre-existing, not introduced).
- Sub-pixel anti-aliasing differences in visual diffs (< 1 px).
- Sub-100ms FOUC in cold-start screenshots.
- Cross-browser scrollbar-width OS deltas.
- Idiomatic disagreements with framework defaults (e.g. Express over Fastify) — this is plan.md's domain.

FORBIDDEN BEHAVIORS:
```
DO NOT:
- Edit, modify, or refactor any source file.
- Retry a failing command in hope of passing — a failure IS a finding.
- Propose fixes inline — only list findings + recommended actions in report.
- Auto-post the report to GitHub/GitLab.
- Claim a check passed without quoting tool output.
- Comment on code style outside session's scope.
- Run destructive commands.
- Continue past PHASE 0 if signals disagree — ABORT and ask.
```

OUTPUT (DUAL):
- **File output (mandatory)**: write `.scratch/<slug>/verification-report-<ISO-date>-<session>.md` with full structured report.
- **Chat output (concise)**: compact summary — session detected, phases run, severity counts, top 5 findings, overall verdict, next action. Link to file; do NOT paste full report in chat.
- **Machine-readable severity tail** (Anthropic Code Review pattern): append at end of report file:
  ```
  <!-- prompthatch-severity: {"critical":N,"major":N,"minor":N,"info":N,"nit":N} -->
  ```
  Parseable via `gh api ... --jq` or grep.
- **DO NOT** post to GitHub/GitLab. The user runs `gh issue comment` / `glab issue comment` manually if they want to share.

FINAL VERDICT BLOCK (end of report file):
```
## Verdict
- Session: NN — <title>
- Mode: FULL | LIGHT | MINIMAL
- Confidence: high | medium | low (reason)
- Total findings: 🔴 N · 🟠 N · 🟡 N · 🔵 N · ⚪ N
- Overall: READY TO CLOSE SUB-ISSUE (FULL) | READY TO TICK CHECKBOX (LIGHT) | READY TO MARK DONE (MINIMAL) | NEEDS FIXES | BLOCKED
- Recommended next action: <one line>
- What I could NOT verify: <list with reasons>
- Re-execution stats (PHASE 6H): <N findings re-verified, M drifted, K refused-no-evidence>
```

--- visual-audit-prompt.md (only if Q5 includes "visual") ---

Inherits TOOL ALLOWLIST, EVIDENCE SCHEMA, FORBIDDEN BEHAVIORS, and SEVERITY LEVELS from verify.md.

Specialization: captures Playwright/Chrome DevTools screenshots at multiple scroll positions + viewports, compares against baseline, scores against Awwwards criteria (Design / Usability / Creativity / Content), writes findings to `.scratch/<slug>/visual-audit-report.md` with versioning (v1 → vN).

Per-viewport double-run pixel diff. Awwwards llm-rubric YAML block with 4 dimensions × scoring bands. WCAG/axe machine tail: `<!-- prompthatch-a11y: {...} -->`. Separate emit slots per (viewport × theme × reduced-motion) combo. "What NOT to flag" list (sub-pixel AA, sub-100ms FOUC, scrollbar-width OS deltas).

--- issues.json schema (with hatch_state, populated in Step 5) ---

Valid JSON example (this is the actual file content, copy as starting skeleton):

```json
{
  "schema_version": "1.0",
  "hatch_version": "0.0.2",
  "platform": "github",
  "mode": "full",
  "parent": null,
  "parentTitle": "",
  "branch": "",
  "prNumber": null,
  "baseBranch": "main",
  "sessions": {},
  "hatch_state": {
    "current_step": 0,
    "current_substep": "0.1",
    "completed_steps": [],
    "started_at": "2026-04-28T22:00:00Z",
    "updated_at": "2026-04-28T22:00:00Z",
    "answers": {},
    "preflight": {
      "git_ok": true,
      "platform_cli_ok": true,
      "skills_required_ok": true,
      "maturity": "established",
      "skills_report_path": ".scratch/skills.json"
    },
    "scaffolding": {
      "preamble_written": null,
      "index_written": null,
      "plan_written": null,
      "prompts_written": [],
      "verify_written": null,
      "visual_audit_written": null
    },
    "tracking": {
      "labels_applied": [],
      "label_policy": "auto-create"
    },
    "errors": []
  }
}
```

Field constraints (enums and types):

| Field | Type | Allowed values |
|---|---|---|
| `schema_version` | string | `"1.0"` |
| `hatch_version` | string | semver, e.g. `"0.0.2"` |
| `platform` | string enum | `"github"` / `"gitlab"` / `"none"` |
| `mode` | string enum | `"full"` / `"light"` / `"minimal"` |
| `parent` | number or null | issue number; null only in MINIMAL |
| `prNumber` | number or null | PR/MR number; null in MINIMAL |
| `baseBranch` | string | git branch name (Q4b answer) |
| `hatch_state.current_step` | integer | 0..6 |
| `hatch_state.completed_steps` | array of strings | e.g. `["0", "1", "2"]` |
| `hatch_state.preflight.maturity` | string enum | `"empty"` / `"greenfield"` / `"early"` / `"established"` / `"mature"` |
| `hatch_state.tracking.label_policy` | string enum | `"auto-create"` / `"preserve"` / `"bypass"` |

Each session entry has the shape:

```json
"01": {
  "title": "auth-foundation",
  "issue": 106,
  "status": "pending",
  "sha": null,
  "verifyReport": null,
  "verify_phases": ["1", "2", "3", "6E"]
}
```

Session field constraints:

| Field | Type | Allowed values |
|---|---|---|
| `title` | string | session step name |
| `issue` | number or null | sub-issue number (FULL only); null in LIGHT/MINIMAL |
| `status` | string enum | `"pending"` / `"in-progress"` / `"done"` / `"skipped"` |
| `sha` | string or null | commit SHA when status transitions to "done" |
| `verifyReport` | string or null | path to verification-report file |
| `verify_phases` | array of strings | phase identifiers from session frontmatter |

Mode-specific shape:
- **MINIMAL**: `parent=null`, `prNumber=null`, all session `issue` values are `null`.
- **LIGHT**: `parent=<num>`, `prNumber=<num>`, session `issue` values are `null`.
- **FULL**: full tracking — every session has its own `issue` number.

=== STEP 5 — WIRE TRACKING (mode-aware, transactional, with multi-dev safety) ===

Step 5 branches based on mode + platform. Never hardcode issue numbers into files; all prompts read from issues.json at session start.

Atomic write protocol applies to every issues.json mutation: write to `.tmp`, validate JSON with `jq empty`, then `mv -f`.

--- IF mode == "MINIMAL" ---

1. `git checkout <integration_branch> && git pull origin <integration_branch>`
2. Pre-flight: `git rev-parse --verify feat/<slug> 2>/dev/null` MUST fail. If branch exists locally: `E_BRANCH_TAKEN — Branch 'feat/<slug>' already exists locally. Delete with \`git branch -D feat/<slug>\` (if abandoned) or pick a new slug.`
3. `git checkout -b feat/<slug>`
4. Update issues.json with branch + hatch_state.scaffolding (atomic write).
5. Do NOT push, do NOT create PR. The user pushes manually when ready.
6. Skip directly to Step 6.

--- IF mode == "LIGHT" ---

1. Resolve parent issue (Q4a):
   - Existing #N: `gh issue view <N>` (or `glab`) — must succeed.
   - "create one with title <X>":
     - github: `gh issue create --title "<plan title>" --body "<3-5 line summary + session checklist>" --label initiative` → capture number.
     - gitlab: `glab issue create --title "<plan title>" --description "<...>" --label initiative` → capture number.
   - Pre-flight labels: run `gh label list --json name --jq '.[].name' | grep -E '^(initiative|session)$'` (or glab equivalent). If missing, attempt `gh label create initiative` and `gh label create session`. If creation fails (enterprise label governance), fall back to no-label and record `hatch_state.tracking.label_policy = "bypass"`. Continue without aborting.
   - Write `parent` and `parentTitle` to issues.json immediately (atomic).

2. `git checkout <integration_branch> && git pull origin <integration_branch>`
3. Pre-flight: `git rev-parse --verify feat/<parent>-<slug>` MUST fail. Else `E_BRANCH_TAKEN`.
4. `git checkout -b feat/<parent>-<slug>`. Write branch to issues.json (atomic).

5. Push branch and open draft PR/MR with session checklist:
   github (POSIX-safe — uses temp file instead of process substitution):
   ```bash
   git push -u origin feat/<parent>-<slug>
   body=$(mk_tmp)
   trap 'rm -f "$body"' EXIT INT TERM HUP
   printf '%s\n' \
     "Closes #<parent>." \
     "" \
     "Sessions:" \
     "- [ ] 01 — <title>" \
     "- [ ] 02 — <title>" \
     "..." \
     "" \
     "Merging to main is a manual human decision — never automatic." > "$body"
   pr_number=$(gh pr create --draft --base <integration_branch> \
     --title "feat: <plan title>" \
     --body-file "$body" \
     --json number --jq .number)
   rm -f "$body"; trap - EXIT INT TERM HUP
   ```
   gitlab:
   ```bash
   git push -u origin feat/<parent>-<slug>
   mr_iid=$(glab mr create --draft --target-branch <integration_branch> \
     --title "feat: <plan title>" \
     --description "Closes #<parent>. Sessions: - [ ] 01 ...") | jq -r .iid
   ```
   Write `prNumber` to issues.json (atomic).

6. On any CLI failure during step 5, ABORT with cleanup hint:
   ```bash
   # LIGHT cleanup hint:
   git checkout <ORIGINAL_BRANCH>
   git branch -D feat/<parent>-<slug>
   gh issue close <parent> --reason "not planned" --comment "Rolled back from hatch failure"
   # (or glab issue close <parent>)
   rm -rf .scratch/<slug>
   ```

7. Sessions are tracked by ticking PR/MR description checkboxes (no sub-issues). All session `issue` fields stay null.

--- IF mode == "FULL" ---

1. Resolve parent issue (Q4a): same as LIGHT step 1.
2. Pre-flight + branch creation: same as LIGHT step 2-4.
3. For EACH session prompt file, create a sub-issue natively:

   github (uses native sub-issues API):
   ```bash
   parent_node_id=$(gh api /repos/:owner/:repo/issues/<parent> --jq .node_id)
   for NN in 01 02 03 ...; do
     sub_number=$(gh issue create \
       --title "Session $NN — <step>" \
       --body "Part of #<parent>. Goal: <session goal>. Done criteria: see prompts/$NN-<step>.md." \
       --label session \
       --json number --jq .number)
     sub_node_id=$(gh api /repos/:owner/:repo/issues/$sub_number --jq .node_id)
     gh api -X POST /repos/:owner/:repo/issues/<parent>/sub_issues -f sub_issue_id=$sub_node_id
     # Append to issues.json IMMEDIATELY (atomic write, transactional)
     jq --arg NN "$NN" --argjson n $sub_number '.sessions[$NN] = {"title":"<step>","issue":$n,"status":"pending","sha":null,"verifyReport":null,"verify_phases":<from frontmatter>}' \
        .scratch/<slug>/issues.json > .scratch/<slug>/issues.json.tmp
     jq empty .scratch/<slug>/issues.json.tmp && mv -f .scratch/<slug>/issues.json.tmp .scratch/<slug>/issues.json
   done
   ```

   gitlab (no native sub-issues, uses `relate`):
   ```bash
   for NN in 01 02 03 ...; do
     sub_iid=$(glab issue create \
       --title "Session $NN — <step>" \
       --description "Part of #<parent>. ..." \
       --label session \
       | grep -oE '#[0-9]+' | head -1 | tr -d '#')
     glab issue relate <parent> $sub_iid 2>/dev/null || true  # may not be supported in older glab
     # Append to issues.json IMMEDIATELY (atomic)
   done
   ```

4. On ANY CLI failure during step 3:
   - STOP. Do NOT edit `00-preamble.md`.
   - Report: parent number, sessions created so far with their numbers, which session failed, what the error was.
   - Provide cleanup one-liner (POSIX shell, mode + platform aware):
     github FULL:
     ```bash
     for n in $(jq -r '.sessions[].issue // empty' .scratch/<slug>/issues.json); do
       gh issue close "$n" --reason "not planned" --comment "Rolled back from hatch failure"
     done
     gh issue close <parent> --reason "not planned" --comment "Rolled back"
     git checkout "$ORIGINAL_BRANCH" && git branch -D feat/<parent>-<slug>
     rm -rf .scratch/<slug>
     ```
     gitlab FULL: same shape with `glab issue close`.
   - Ask: resume from session N, full rollback, or leave as-is?

5. When all sub-issues are created successfully:
   - Rewrite parent issue body with real checklist: `- [ ] Session NN — <title> (#<sub-num>)`.
   - Update issues.json hatch_state.completed_steps to include "5" (atomic write).

6. Push branch and open draft PR/MR (same as LIGHT step 5 but without inline checklist — sessions tracked via sub-issues).

=== STEP 6 — REPORT BACK ===

Emit a single final message:

```
┌─────────────────────────────────────────────────┐
│ ✓ Hatch complete — <slug>                    │
│   Mode: <MODE> · Platform: <PLATFORM> · Time: <Xm>│
└─────────────────────────────────────────────────┘

Created in .scratch/<slug>/:
  00-preamble.md       (<N> lines)
  INDEX.md             (<N> lines)
  plan.md              (<N> lines)
  issues.json          (with hatch_state)
  prompts/
    01-<step>.md       (<N> lines, ~<min>min)
    02-<step>.md       (<N> lines, ~<min>min, parallel with NN)
    ...
    verify.md          (<N> lines)
    [visual-audit-prompt.md  — if Q5 includes visual]

[FULL/LIGHT only:]
Parent issue:  #<N>  <URL>
Draft PR/MR:   #<N>  <URL>
Branch:        feat/<parent>-<slug> (from <integration_branch>)

[FULL only:]
Sub-issues table:
  | Session | Title | Issue | URL |
  | 01 | <step> | #<N> | <URL> |
  ...

[MINIMAL only:]
Local branch: feat/<slug> (from <integration_branch>) — push manually when ready.

Total sessions: <N>
Parallelizable: sessions <list>
Estimated total time: ~<sum>min across <N> sessions.

Skills inventory:
  ✓ <N> required skills present
  ✓ <N> recommended skills present (<list>)
  ▲ <N> missing recommended (<list>) — fallback active in session prompts.

Ambiguities resolved by assumption (correct before Session 01 if needed):
  - <list, or "none">

▸ Next: open a fresh Claude Code conversation and paste:
   .scratch/<slug>/prompts/01-<step>.md

Hatch is done. Sessions execute one at a time. After each session, run verify.md.
```

DO NOT start executing sessions. Hatch is done.

=== CRITICAL RULES ===

- Idempotency guard in Step 0.2 is mandatory. Never overwrite existing `.scratch/<slug>/` silently.
- Skills.json must exist BEFORE hatch runs. If missing, abort with E_SKILLS_MISSING and reference _detect-skills.md.
- Required skills (from skills.json `classification.required_missing`) being non-empty aborts the hatch. The methodology spine cannot run without them.
- Branches always FROM the integration branch (Q4b). Draft PRs/MRs always TARGET the integration branch (FULL/LIGHT only). `main` is never touched automatically.
- Sub-issues have NO branches (FULL mode). All commits land on `feat/<parent>-<slug>` (FULL/LIGHT) or `feat/<slug>` (MINIMAL).
- `Closes #NN` in commits does NOT close sub-issues until merge to default branch (FULL only). Use `gh issue close <N> --comment "Closed via <SHA>"` (or `glab`) explicitly per session.
- Never hardcode issue numbers into prompt text. Always read from issues.json.
- Step 5 is transactional (atomic writes per sub-issue creation). Halt + report + offer cleanup on any failure.
- Multi-dev locking: `.scratch/<slug>/.hatch.lock` acquired in Step 0.8, released on EXIT/INT/TERM trap. Stale (>1h) locks auto-removed.
- Atomic writes: all issues.json mutations write to `.tmp`, validate via `jq empty`, then `mv -f`.
- POSIX shell only in generated bash. No `[[ ]]`, no `<( )` process substitution, no `mapfile`, no `echo -e`. Use `[ ]`, `printf '%s\n'`, temp files for here-doc bodies.
- Respect any project-specific memory locks the user provided in Q6.
- Neutralize `using-superpowers` auto-trigger inside the generated preamble.
- No emoji in generated source files unless requested. Severity glyphs (🔴🟠🟡🔵⚪) are allowed in verify reports as they're part of the published 2026 schema.
- Use bundled skills as tools (verification-before-completion, systematic-debugging, test-driven-development). Do NOT let any skill act as orchestrator — `.scratch/<slug>/plan.md` is the orchestrator.
- Before applying a different pattern than the plan: critically verify it matches 2026 best practices for this project, then apply and explain in commit message.
- Mode-specific:
  - FULL/LIGHT require github or gitlab. ABORT if Q4c == none.
  - MINIMAL is compatible with any platform (including none) and produces no platform-side artifacts.
- Hatch version compatibility: state created by older hatch versions can be resumed if `schema_version` matches; upgrade migration via `.scratch/_migrations/` if available.

=== RECOVERY COMMANDS LIBRARY ===

For users who need to recover from a failed/aborted/abandoned hatch:

```bash
# Cancel current hatch + full cleanup (FULL mode, github):
SLUG=<slug>
PARENT=$(jq -r '.parent // empty' .scratch/$SLUG/issues.json)
BRANCH=$(jq -r '.branch // empty' .scratch/$SLUG/issues.json)
ORIGINAL=$(git rev-parse --abbrev-ref HEAD)
for n in $(jq -r '.sessions[].issue // empty' .scratch/$SLUG/issues.json); do
  gh issue close "$n" --reason "not planned" --comment "Cleanup: hatch cancelled"
done
[ -n "$PARENT" ] && gh issue close "$PARENT" --reason "not planned" --comment "Cleanup: hatch cancelled"
git checkout "$ORIGINAL"
[ -n "$BRANCH" ] && git branch -D "$BRANCH" 2>/dev/null
rm -rf .scratch/$SLUG

# Pause (preserve state, no cleanup): just stop. Lock auto-released by trap.

# Resume: re-paste this hatch with same slug. Step 0.2 detects existing state and offers (r)esume.

# Mode migration MINIMAL → LIGHT (after sessions started):
# 1. Create parent issue: gh issue create --title "<X>" → capture #N
# 2. Update issues.json: jq '.mode = "light" | .parent = N | .platform = "github"' (atomic)
# 3. Push branch: git push -u origin <branch>
# 4. Open draft PR with checklist: gh pr create --draft ...

# Force-unlock (if a stale lock blocks you):
rm -f .scratch/<slug>/.hatch.lock
```

=== END META-PROMPT ===
```

---

## Why this shape (short reasoning)

- **Step 0.5 reads skills.json** — this is the single biggest reliability win in v0.0.2. Hatch never starts without knowing what skills are available; missing required skills abort early instead of failing silently mid-Session 03.
- **Step 0.6 maturity probe** — adapts question count and reading depth to project size. Greenfield users answer 8 questions instead of 10; empty repos 4 instead of 10. No CLAUDE.md is auto-generated (user's explicit decision).
- **Step 0.7 auto-detection** — pre-fills 6 of 10 question values with confidence levels. User mostly confirms instead of typing.
- **Step 0.8 multi-dev locking** — POSIX `set -o noclobber` for atomic create-or-fail. Trap-based release on every exit path including SIGINT.
- **Step 2 has 10 questions but typing time drops to ~90 seconds** with auto-fill defaults. Cross-field validation runs after answers (V13/V15/V17/V24/V25/V31) before Step 3.
- **Step 3 parallel research adapts** — 2-3 agents for greenfield, 5-10 for established/mature.
- **Step 4 session prompt template v2** — YAML frontmatter (`session_kind`, `verify_phases`, `touches`, `do_not_touch`) becomes deterministic SIGNAL E for verify.md PHASE 0 detection.
- **verify.md PHASE 6H re-execution** — Anthropic Code Review's <1% false-positive technique. Every PASS gets re-verified by re-running the cited command.
- **Step 5 mode-aware + transactional + atomic writes** — each sub-issue creation appends to issues.json before the next one starts. Cleanup one-liners are POSIX-compliant for cross-shell.
- **Critical rules consolidate the safety net** — backwards compatible (FULL+github+main reproduces old behavior), forward-compatible (schema_version + hatch_version stamps).
