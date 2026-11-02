# OpenCode Agent Orchestrator

A lightweight OpenCode configuration that implements a practical orchestrator pattern for AI-assisted software development.

The goal is to get the benefits of a multi-agent workflow - planning, execution, review, and security checks - without turning the setup into a complex agent framework or adding excessive token overhead.

## Why this exists

Single-agent workflows are often enough for small tasks, but they tend to become less reliable as work gets larger or more ambiguous. Common issues include:

- Planning and implementation getting mixed together
- Repeated context consuming unnecessary tokens
- Reviews being skipped or performed inconsistently
- Security, documentation, and testing concerns being handled too late
- Large tasks producing broad, hard-to-review diffs
- Inefficient token usage when the same model is applied to tasks of varying complexity

This repository provides a small orchestrator-based pattern that keeps the workflow structured while remaining easy to inspect, modify, and run.

## What this provides

- **A very simple orchestrator pattern** - A central agent coordinates planning, approval, implementation, and review phases that you can build upon and customize as needed.
- **Scoped subagents** - Specialized agents handle planning, read-only exploration, execution, testing, documentation, security, and code review
- **Token-conscious delegation** - Subagents receive focused tasks instead of the full problem context whenever possible
- **Approval-gated planning** - Non-trivial work is planned before implementation starts
- **Minimal framework overhead** - The system is implemented as OpenCode configuration, agent prompts, and skills
- **Reusable development skills** - Shared skill definitions for delegation, task management, Python quality, and skill authoring

## Design goals

This project is intentionally small. It is not trying to be a general-purpose agent platform.

It aims to:

- Keep orchestration understandable
- Reduce unnecessary context passed between agents
- Preserve code quality through explicit planning and review steps
- Make agent responsibilities clear
- Avoid excessive hidden behavior that obscures what is happening
- Provide a useful starting point for customizing OpenCode workflows
- Sync the improvements with my own personal workflow.

## Structure

```
├── 📁 agents
│   ├── 📝 api-docs-researcher.md
│   ├── 📝 code-executor.md
│   ├── 📝 code-explorer.md
│   ├── 📝 code-reviewer.md
│   ├── 📝 docs-reviewer.md
│   ├── 📝 host-security-investigator.md
│   ├── 📝 orchestrator.md
│   ├── 📝 plan-runner.md
│   ├── 📝 security-reviewer.md
│   ├── 📝 spec-critic.md
│   └── 📝 test-verifier.md
├── 📁 profiles
│   ├── 📁 balanced
│   │   └── 📁 glm-qwen
│   │       └── 📄 opencode.jsonc
│   └── 📁 budget
│       ├── 📁 glm-deepseek
│       │   └── 📄 opencode.jsonc
│       └── 📁 qwen-mimo
│           └── 📄 opencode.jsonc
├── 📁 skills
│   ├── 📁 agent-delegation
│   │   └── 📝 SKILL.md
│   ├── 📁 pythonic-quality
│   │   └── 📝 SKILL.md
│   ├── 📁 skill-creator
│   │   ├── 📁 references
│   │   │   ├── 📝 output-patterns.md
│   │   │   └── 📝 workflows.md
│   │   ├── 📁 scripts
│   │   │   ├── 🐍 init_skill.py
│   │   │   ├── 🐍 package_skill.py
│   │   │   └── 🐍 quick_validate.py
│   │   ├── 📄 LICENSE.txt
│   │   └── 📝 SKILL.md
│   └── 📁 task-management
│       ├── 📁 scripts
│       │   ├── 📄 migrate-schema.ts
│       │   └── 📄 task-cli.ts
│       ├── 📝 SKILL.md
│       └── 📄 router.sh
├── ⚙️ .gitignore
├── 📝 AGENTS.md
├── 📄 LICENSE
├── 📝 README.md
├── ⚙️ cli.json
├── 📄 opencode.jsonc
├── ⚙️ package-lock.json
├── ⚙️ package.json
└── ⚙️ tsconfig.json
```

## How the workflow works

The default entry point is the `orchestrator` agent.

For non-trivial tasks, the workflow typically follows this pattern:

1. **Orchestrator receives the user request**
2. **Planning is delegated** to a planning subagent
3. **A concrete plan is written** under `.opencode/plans/`
4. **The user approves or rejects the plan**
5. **Implementation is delegated** in scoped slices
6. **Review agents validate the result** for correctness, tests, documentation impact, and security concerns

This keeps the main agent focused on coordination instead of forcing one large prompt/session to handle every phase of the task.

Plan approval is not automated; the orchestrator continues Phase B itself.

## Agents

### Primary agents (Strong models) - GLM-5.3 Flash `max` (default config)

- **orchestrator** - Coordinates multi-phase work through plan files, approval gates, and implementation slices
- **build** - OpenCode Default Agent (Best for simple tasks)
- **plan** - OpenCode Default Agent  (Best for simple tasks)

### Subagents (Cheap models) - DeepSeek V4.1 Flash `max` (default config)

- **plan-runner** - Drafts implementation plans under `.opencode/plans/`
- **code-explorer** - Read-only exploration: maps architecture and locates symbols; the orchestrator's repo-discovery path.
- **code-executor** - Implements scoped coding tasks with minimal diffs
- **test-verifier** - Validates changes through tests, linting, and type checking
- **code-reviewer** - Reviews diffs for correctness, maintainability, and implementation risk
- **docs-reviewer** - Checks whether documentation needs to be updated
- **security-reviewer** - Identifies security risks in application code
- **spec-critic** - Challenges plans before coding starts
- **api-docs-researcher** - Researches external API documentation
- **host-security-investigator** - Assesses hosting and infrastructure security concerns

## When to use this

This configuration is useful when you want:

- More structure than a single-agent coding workflow
- Planning before implementation
- Smaller, easier-to-review diffs
- Separate review passes for tests, docs, code quality, and security
- A multi-agent setup that is still simple enough to understand and modify

It may be unnecessary for very small edits where a direct `build` agent is faster and cheaper.

## Playing with profiles

Slowly adding new opencode.jsonc files to `profiles/*` that can be used to quick explore different models
and combinations of agents. Simply use the default `opencode.jsonc` if you don't want to test anything.

Profiles are self-contained (they carry permissions, todo flag, and title/summary/compaction pins) and also MERGE over the global config via `OPENCODE_CONFIG`: per-key last-wins, agents maps merge, permission rules append after root rules — validate with `opencode --standalone`, since a plain `opencode debug agents` reads the running background service's root config and is a false pass.

Keep model refs on gateway-catalog variants only (`#max`, `#xhigh`; `mimo-v2.5` has no variants and stays bare) — v2.2.3 rejects unknown variant IDs at session-create; do not reintroduce custom or temperature variants.

You can select a profile by using OpenCode's `OPENCODE_CONFIG` environment variable, that makes it super easy:

Currently available test profiles:

| Profile | Configuration | Models |
|---------|---------------|--------|
| Default             | `opencode.jsonc`                          | GLM-5.3 Flash `max` / DeepSeek V4.1 Flash `max`
| Budget / GLM + DeepSeek | `profiles/budget/glm-deepseek/opencode.jsonc` | GLM-5.3 Flash `max` / DeepSeek V4.1 Flash `max`
| Balanced / GLM + Qwen | `profiles/balanced/glm-qwen/opencode.jsonc` | GLM-5.3 Flash `max` / Qwen3.8 Flash `xhigh`
| Budget / Qwen + MiMo | `profiles/budget/qwen-mimo/opencode.jsonc` | Qwen3.8 Flash `xhigh` / MiMo V2.5 (bare, no variants)

Replace `<profile>` below with the desired profile path, for example `balanced/glm-qwen`.

### Bash / Zsh

Current session:

```sh
export OPENCODE_CONFIG="$HOME/.config/opencode/profiles/<profile>/opencode.jsonc"
```

For persistent use, add the same `export` command to `~/.bashrc` or `~/.zshrc`.

### PowerShell

Current session:

```powershell
$env:OPENCODE_CONFIG = "$HOME\.config\opencode\profiles\<profile>\opencode.jsonc"
```

Persistent:

```powershell
[Environment]::SetEnvironmentVariable(
    "OPENCODE_CONFIG",
    "$HOME\.config\opencode\profiles\<profile>\opencode.jsonc",
    "User"
)
```

> These examples assume the repository is installed at `~/.config/opencode`. Adjust the base path if you installed it elsewhere.

### Fish

Current session:

```fish
set -x OPENCODE_CONFIG "$HOME/.config/opencode/profiles/<profile>/opencode.jsonc"
```

Persistent:

```fish
set -Ux OPENCODE_CONFIG "$HOME/.config/opencode/profiles/<profile>/opencode.jsonc"
```

## Skills

- **agent-delegation** - Decision table for routing work to the appropriate subagent
- **task-management** - CLI for tracking feature subtasks with dependencies
- **pythonic-quality** - Pythonic idioms, SOLID design, and Liskov-safe patterns
- **skill-creator** - Guide for creating effective skills

## Installation

1. Install [OpenCode](https://opencode.ai)

2. Clone this repository to your OpenCode config directory:

   ```bash
   git clone <repo-url> ~/.config/opencode
   ```

3. Install dependencies:

   ```bash
   cd ~/.config/opencode
   npm install
   ```

### Development notes

- `npm install` installs devDeps by default; under `NODE_ENV=production` use `npm install --include=dev`.
- Typecheck with `npx tsc --noEmit` (`tsconfig.json` now includes only `skills/**/*.ts`; plain `npx tsc` emits JS).

## Usage

Start OpenCode and describe your task.

The `orchestrator` agent is configured as the default entry point. For simple tasks, the system can proceed directly. For larger or riskier tasks, it routes work through planning, approval, implementation, and review phases.

The intended flow is:

```text
user request
   ↓
orchestrator
   ↓
plan-runner / plan
   ↓
user approval
   ↓
code-explorer (when needed)
   ↓
code-executor / build
   ↓
review subagents
```

## Configuration

Key settings in `opencode.jsonc`:

- **`default_agent: "orchestrator"`** - Sets the orchestrator as the default entry point
- **`permissions`** - Minimal workspace baseline (ask on `shell`, `edit`, and `external_directory`); each agent declares its own ordered tool policy in `agents/<id>.md` frontmatter
- **`experimental.todo: true`** - V2 defaults todowrite off; orchestrator/plan-runner rely on it.
- **external_directory ask rule** - V2 defaults to allow; restores the V1 ask-on-outside-project behavior. The global baseline asks only for shell, edit, and external_directory; read/glob/grep/webfetch/websearch follow OpenCode defaults, custom agents define their own ordered rules, and builtin shipped policies (explore's read-only denies, plan's edit scope, summary/compaction/title model pins) are re-pinned in `agents.*` to win over the baseline.
- **`agents.*.model`** - Configures model selection for primary agents and selected subagents; a ref may carry a `#variant` suffix (e.g. `opencode-go/glm-5.3-flash#max`). v2.2.3 rejects variant IDs not present in the model catalog at session-create, so only catalog variants are used. Per-agent `temperature` was deliberately dropped in the same fallback: with reasoning on, the deepseek/qwen families ignore it anyway, and no V1 behavior was lost at the gateway (V1 sent the same catalog efforts).
- **`agents.title.model`** - Sets the small model used by the built-in `title` agent (session titles); replaces the old `small_model` key. `agents.summary.model` and `agents.compaction.model` pin the maintenance agents the same way.

CLI-only settings (theme, keybinds, diffs, session view, animations, and `attention` alerts) live in the separate `cli.json` and are not read from `opencode.jsonc`.


## Changelog

### [1.1.6] - 2026-11-02

- **Added**
  - Built-in policy re-pins in `agents.*`: `explore` keeps its hard `edit` deny and `shell` allow; `plan` gets plan-dir `edit` allows (`.opencode/plans/**`, `.opencode/plan/**`); `summary` and `compaction` pinned to cheap models (bare `deepseek-v4.1-flash`), per-profile variants of the same pins; denies/asks restored even with the global ask baseline (configured agent rules are the last layer in V2).
  - `experimental.todo: true` (root and all profiles): V2 defaults todowrite off; orchestrator/plan-runner rely on it.
  - Profile self-containment: permissions array, `todo` flag, and `title`/`summary`/`compaction` pins live in each profile (catalog-only refs: `glm-5.3-flash#max`, `qwen3.8-flash#xhigh`, bare `mimo-v2.5`). Validation note in profiles section (`opencode --standalone` merge check).
  - `.gitignore` shields for local-only skills re-added (`/skills/context7/`, `/skills/gitnexus*`, `/skills/agent-memory`, `agent-memory.skill`).
- **Changed**
  - Native V2 config shapes in `opencode.jsonc` and profiles: `agents` map with joined `model#variant` refs (no separate `variant` key; no `providers` block); top-level ordered `permissions` array (shell/edit/external_directory ask) replacing the `permission` map; `autoupdate: true` → `update: "auto"`; `small_model` → `agents.title.model`.
  - Agent frontmatter (`agents/*.md` ×11): `permission` maps → ordered `permissions` arrays; action renames `bash`→`shell`, `task`→`subagent`, `write`→`edit`; dropped inert `doom_loop`/`lsp`/`list` entries and per-agent `external_directory` (global rule covers it); `todowrite` rules retained.
  - Deny-first `subagent "*"` allowlist hardening added to orchestrator/plan-runner/code-explorer; code-executor and host-security-investigator keep their pre-existing deny-first from V1.
  - Variant/temperature policy: (i) v2.2.3 rejects variant IDs not present in the model catalog at session-create, so only catalog variants are used; (ii) per-agent `temperature` was deliberately dropped in the same fallback — with reasoning on, deepseek/qwen families ignore it anyway, and no V1 behavior was lost at the gateway since V1 sent the same catalog efforts.
  - Plan approval is orchestration-driven: orchestrator-routed approvals continue Phase B in the orchestrator session; plan-routed approvals end there and you switch to `build` (or the orchestrator) to implement. No summarize/prompt handoff automation.
  - README current-state corrections: structure tree, profile table (incl. `budget/qwen-mimo`), model headings, permissions wording, `--standalone` profile-validation caveat, plugin claims removed.
- **Removed**
  - `plan-post-approval` plugin and `plugin-src/`; the `plugin` config entry; the `@opencode-ai/plugin` dependency (with package-lock regeneration; project version `1.1.5`→`1.1.6`); the plugin tsconfig include; auto-generated `.opencode/package.json` and lockfile.
  - Legacy client files: `tui.json` (also untracked the phantom skip-worktree-tracked empty blob) and the empty `command/` directory; the `tui.json` and `/command` ignore entries pruned.

### [1.1.5] - 2026-09-28

- **Added**
  - `cli.json` (OpenCode v2 CLI settings file): `vesper` theme, keybinds (`<leader>p` command palette, `enter` submit, `shift+enter` newline), word-wrapped diffs, session sidebar/scrollbar/thinking preferences, and animations.
  - Native `attention.notifications` alert in `cli.json`, replacing the notifier plugin with OpenCode's built-in system notifications.
- **Removed**
  - `@mohak34/opencode-notifier` plugin from the `plugin` array in `opencode.jsonc`. Notification duty now belongs to the native `attention` alerts in `cli.json` (OpenCode v2); the stale `opencode-notifier-state.json` entry was dropped from `.gitignore`.

### [1.1.4] - 2026-08-31

- **Added**
  - `profiles/budget/deepseek-vision/` profile (Deepseek V4 Pro primaries / Deepseek V4 Flash Vision Exp subagents).
  - `small_model` pin (`opencode-go/deepseek-v4-flash`) in `opencode.jsonc`.
  - `profiles/budget/qwen-mimo/` profile (Qwen3.8 Flash `xhigh` primaries / MiMo V2.5 subagents, no variants).
  - `profiles/budget/glm-deepseek/` profile (GLM 5.3 Flash `max` primaries / DeepSeek V4 Flash `max` subagents).
- **Changed**
  - Global provider `reasoningEffort` block replaced by per-agent `variant: "max"` entries in `opencode.jsonc`.
  - `@opencode-ai/plugin` aligned to 1.14.20; `tsconfig.json` now type-checks `plugin-src/`.
- **Fixed**
  - `profiles/budget/glm-qwen/`: `"variant:"` key typo and stale model comment.
  - Profiles no longer redeclare the `plugin` array — plugins are inherited from the global config, fixing the relative `plugin-src/` path from profile directories.

### [1.1.3] - 2026-06-21

- **Fixed**
  - Moved `plan_post_approval_handoff_agent` out of `agent.orchestrator` (where opencode routed it into the provider request `options`) and into the `plan-post-approval` plugin tuple options. Restores compatibility with strict providers (GLM, Vertex AI, Fireworks) that rejected the extra field with "Extra inputs are not permitted". Same class of fix as the `textVerbosity` removal in [1.1.1].
- **Changed**
  - Relocated `plugins/plan-post-approval.ts` to `plugin-src/plan-post-approval.ts` so it is loaded only via the explicit `plugin` tuple (with options) instead of auto-discovery, avoiding double-load.

### [1.1.2] - 2026-06-21

- **Changed**
  - Updated md files and `opencode.jsonc` so model and temperature are configurable inside the jsonc file.

### [1.1.1] — 2026-06-20

- **Fixed**
  - Removed textVerbosity parameter from `opencode.jsonc` so GLM and other models work properly

### [1.1.0] — 2026-05-12

- **Changed**
  - Orchestrator agent now don't have write permissions by default so we can force subagent usage.
  - Updated agent configurations (orchestrator and all subagents)
  - Updated `opencode.jsonc` with refined model routing, reasoning effort, and permission settings
  - Updated README with structure, workflow, and configuration documentation
- **Fixed**
  - Corrected model names in agent configurations
  - Added DeepSeek max reasoning effort support in `opencode.jsonc`
- **Removed**
  - Build and Plan customization. The config now uses the opencode default Build and Plan agents. Add `agents/build.md` and `agents/plan.md` if you want to override.

### [1.0.0] — 2026-05-02

Initial release: multi-agent orchestrator pattern with planning, code execution, and review subagents.

