# BMAD `/epic-cycle` Command — Installation Kits

> [!NOTE]
> **Tested with [Claude Code](https://claude.com/claude-code) only.** These kits are written as prompts for an agentic coding assistant to execute, and the installed command relies on Claude Code specifics (slash commands, the `Skill`/`Agent` tools, subagent model parameters, permission modes). Other coding agents are untested.

> [!IMPORTANT]
> **Requires BMAD Method v6.11.0 or later.** These kits drive the v6.11 Phase-4 chain (`bmad-sprint-planning → bmad-build-auto → bmad-code-review`). BMAD [v6.11.0](https://github.com/bmad-code-org/BMAD-METHOD/releases) deprecated `bmad-create-story` / `bmad-dev-story` and retired the `bmad-sprint-status` data modes the earlier kits depended on. For a v6.10 project use the last pre-6.11 kits (commit `c2b7cb0` in this repo's history — base kit `2026-07-19.1`, parallel kit `2026-07-11.3`). The refactor rationale is in [docs/bmad-6.11-refactor-proposal.md](docs/bmad-6.11-refactor-proposal.md).

Three self-contained installation kits that add an autonomous, multi-agent epic development pipeline — the `/epic-cycle` slash command — to any [BMAD Method](https://github.com/bmad-code-org/BMAD-METHOD) v6.11+ project running under [Claude Code](https://claude.com/claude-code).

These are not shell scripts. Each document is a **prompt-as-installer**: you point a Claude Code session at it, and the session reads the steps, performs the file operations (with detection, backups, and grep-guarded idempotent patches), and verifies the result. All three kits are safe to re-run.

## What each kit does

### 1. `epic-cycle-workflow-creation.md` — base kit (required)

Installs the `/epic-cycle` slash command and its supporting BMAD skill customizations:

- **`.claude/commands/epic-cycle.md`** — the command itself. Given an epic range (e.g. `/epic-cycle 1-3`), a lead session drives the full BMAD story pipeline per epic: headless sprint planning (readiness gate) → `bmad-build-auto` **plan** spawn (Opus, `Halt after planning.`) → lead validates the spec → `bmad-build-auto` **implement** spawn (Sonnet; implements, self-reviews, commits) → QA test generation → adversarial `bmad-code-review` (Opus) → bounded rework loop (max 3 iterations) → per-story smoke → commit/push, plus per-epic branch management, retrospective, and merge gates. Every stage is spawned with an explicit stage→model map; the lead owns `sprint-status.yaml` writes through BMAD's deterministic `sprint_plan.py`.
- **`_bmad/custom/skill-rules.md`** — a cross-cutting rules registry (integration ACs, real-runtime test evidence, ADR enforcement, unattended-menu protocol, finding-disposition bar, clean-tree-before-dispatch, the ledger drain, and more) loaded by every BMAD skill via `persistent_facts`.
- **`_bmad/scripts/ledger.sh`** — the deferred-work ledger tool (bash + awk). Every finding the pipeline defers is filed with an owner, closed at the owning story's `ledger_adjudicated` gate, swept at the epic's burn-down gate (which can charter a Story N.9 burn-down), and re-decided — never relabelled — at the next epic's Story X.0. The lead reads the ledger only through `load`/`slice`/`show` and writes only through `new`/`append`, so the file stays small, union-merge-safe, and machine-countable (a field report of a 1,588-entry, never-shrinking ledger motivated this).
- **`_bmad/custom/bmad-*.toml`** — customization overrides wiring the rules registry into `bmad-build-auto`, `bmad-build`, `bmad-qa-generate-e2e-tests`, and `bmad-code-review` (and disabling `bmad-build`'s editor launch).

### 2. `skill-optimization-prompt.md` — model-tier pass (optional)

A cost/quality optimization pass over the installed BMAD skills. It writes a `model:` pin into each skill's `SKILL.md` frontmatter following an **expensive planner/reviewer + efficient implementer** strategy:

- **Opus** — architecture, spec/PRD authoring, story planning (`bmad-build-auto`'s plan tier), `bmad-review`, and every adversarial review gate.
- **Sonnet** — implementation (`bmad-build`), test writing, research, documentation, sprint planning, personas (the high-frequency workhorses).
- **Haiku** — `bmad-help` only on a vanilla install.

It also documents the per-project model-policy file (`_bmad/custom/model-overrides.yaml`) with telemetry-gated escalation for uncommon stacks (e.g. InterSystems ObjectScript) and an opt-in review-lane de-escalation experiment. Declining this pass is a valid configuration — `/epic-cycle`'s built-in stage→model map still governs the spawned pipeline stages either way.

### 3. `parallel-epic-cycle-workflow-creation.md` — parallel kit (optional)

Upgrades an installed `/epic-cycle` with **parallel epic execution**:

- **Orchestrator/runner modes** — the orchestrator analyzes epic dependencies (user-approved DAG), then dispatches background runner agents, each owning one epic end-to-end.
- **Submodule-aware git worktrees** — provisioning/teardown scripts (`_bmad/scripts/new-epic-worktree.sh` / `remove-epic-worktree.sh`, cross-platform bash) that mount modified submodules as gitlink-path worktrees with unpushed-work guards.
- **Runtime-lock protocol** — claim-file locks serializing stages that touch a shared live runtime (smoke, QA, ADR verifications).
- **Serialized, user-gated merge queue** — one epic at a time into the feature branch, submodules-first, with gitlink re-recording and conflict policy.
- **Containment (Rule 18)** — a returned agent is finished (`SendMessage` to any returned agent is forbidden at every depth; `bmad-build-auto`'s own "re-engage the subagent" text is pre-answered as "cannot"); every agent returns exactly once; the base kit writes the git/CI/`SendMessage` prohibitions into `bmad-build-auto`'s handoff and review-layer prompts via `_bmad/custom/*.toml`, so depth-3 agents carry them by construction; an orphan-writer protocol for when it happens anyway. The model-tier checkpoint now evaluates on every project (the policy file is needed only to apply a change).
- **Liveness doctrine** — every stage spawn is preceded by a `stage_spawned` write-ahead entry; liveness is a property of sessions, not files (a fresh sequential lead owns no live stages; an orchestrator asks the runner, never diagnoses a worktree from its artifacts); `SendMessage` is orchestrator→runner only, never to a stage agent; every `🚫` git/CI prohibition propagates into internal subagents alongside the working directory. (From a 2026-08-28 field incident: a depth-3 subagent pushed and deployed, and a live stage was re-dispatched onto.)

It patches the base kit's command at exact anchors (six grep-guarded edits), appends Rules 10–12 to the rules registry, and adds `_bmad/custom/parallel.yaml` for per-project tuning.

## Installation

### Prerequisites

- [Claude Code](https://claude.com/claude-code)
- [`uv`](https://docs.astral.sh/uv/) on PATH (Python 3.11+ is provisioned by uv) — BMAD v6.11's rendered skills (`bmad-build`, `bmad-build-auto`) halt without it
- A BMAD Method **v6.11.0+** project: `npx bmad-method install` (modules `core,bmm`, IDE `claude-code`), with `_bmad-output/implementation-artifacts` tracked in git (the pipeline's durable state lives there)
- git ≥ 2.34 (worktrees, `:(exclude)` pathspecs)
- For the parallel kit: `.claude/skills/`, `_bmad/scripts/`, `_bmad/config.toml`, and `_bmad/custom/` committed to git (epic worktrees only contain tracked files)

### Install order

Order matters — the model pass pins `model: opus` onto the command the base kit writes, and the parallel kit patches anchors the base kit provides:

1. `npx bmad-method install` (if BMAD isn't installed yet)
2. `epic-cycle-workflow-creation.md` — base kit
3. `skill-optimization-prompt.md` — optional model-tier pass
4. `parallel-epic-cycle-workflow-creation.md` — optional parallel kit

You don't have to chain these by hand: the parallel kit detects a missing base kit (and un-applied model pass) and offers to install them first.

### Running a kit

Open a Claude Code session **in the target project's root** and ask it to execute the kit, e.g.:

```
Read and execute C:\git\bmad-epic-cycle-command\epic-cycle-workflow-creation.md
```

The session will detect prior state, back up anything it overwrites (`<file>.bak-<UTC>`), apply the steps, run the validation greps, and record the installed version in `_bmad/custom/kit-versions.yaml`. Commit the resulting files when it finishes.

**Restart your session afterward.** A session that has already loaded `/epic-cycle` or the BMAD skills keeps the old text in context; the installed changes only take effect in a fresh session.

### Upgrading / re-running

- All kits are idempotent (detection steps and grep-guards skip work already done).
- Re-running the **base kit** overwrites `.claude/commands/epic-cycle.md`, which removes the model pin and the parallel kit's patches — re-run steps 3 and 4 afterward. It rewrites `_bmad/custom/skill-rules.md` too, carrying over the parallel kit's Rules 10–12 and any project-specific rules it finds (check the report it prints).
- The parallel kit's Step 1a compares `Kit-Version` headers against `_bmad/custom/kit-versions.yaml` and offers stale companions for re-run in canonical order.
- **Upgrading mid-epic:** a sequential run can be upgraded at a story boundary (after a `committed`, before the next story; clean tree, no agent in flight) — resume handles the new gates, and the ledger migration becomes part of the upgrade commit. An orchestrator (parallel) run cannot: runners work in epic-branch worktrees that don't contain the upgrade commit. Finish and merge the in-flight epics first. Both kits print the details at upgrade time.

## Usage

After installation, in a fresh Claude Code session in the project:

```
/epic-cycle 2      # run epic 2 (classic sequential flow)
/epic-cycle 2-4    # run epics 2 through 4; with the parallel kit + parallel.yaml,
                   # this triggers dependency analysis and concurrent runner dispatch
```

Unattended runs need an unattended permission mode on the **lead session** (subagents inherit it; the `Agent` tool's `mode` parameter is ignored on current Claude Code builds): start it with `claude --permission-mode bypassPermissions` (or `--dangerously-skip-permissions`), and only in a trusted or isolated environment. `_bmad-output/implementation-artifacts` must be tracked in git — it holds the pipeline's durable state (cycle logs, `sprint-status.yaml`, specs, the deferred-work ledger).

### Relationship to `bmad-loop`

BMAD's optional [`bmad-loop`](https://github.com/bmad-code-org/bmad-loop) module is a Python orchestrator that also drives `bmad-build-auto`, dispatching from a `bmad-spec` folder (`SPEC.md` + `stories.yaml`). `/epic-cycle` dispatches from `epics.md` + `sprint-status.yaml` and adds the branching doctrine, ADR gates, lead-side smoke, the Opus code-review gate, and per-epic worktrees. Both can be installed, but never run both over the same stories — pick one per epic.

## License

[MIT](LICENSE)
