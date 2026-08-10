# BMAD `/epic-cycle` Command — Installation Kits

> [!NOTE]
> **Tested with [Claude Code](https://claude.com/claude-code) only.** These kits are written as prompts for an agentic coding assistant to execute, and the installed command relies on Claude Code specifics (slash commands, the `Skill`/`Agent` tools, subagent model parameters, permission modes). Other coding agents are untested.

Three self-contained installation kits that add an autonomous, multi-agent epic development pipeline — the `/epic-cycle` slash command — to any [BMAD Method](https://github.com/bmad-code-org/BMAD-METHOD) v6 project running under [Claude Code](https://claude.com/claude-code).

These are not shell scripts. Each document is a **prompt-as-installer**: you point a Claude Code session at it, and the session reads the steps, performs the file operations (with detection, backups, and grep-guarded idempotent patches), and verifies the result. All three kits are safe to re-run.

## What each kit does

### 1. `epic-cycle-workflow-creation.md` — base kit (required)

Installs the `/epic-cycle` slash command and its supporting BMAD skill customizations:

- **`.claude/commands/epic-cycle.md`** — the command itself. Given an epic range (e.g. `/epic-cycle 1-3`), a lead session drives the full BMAD story pipeline per epic: sprint planning → story creation (lead-side, Opus-tier) → dev → QA test generation → adversarial code review → bounded rework loop (max 3 iterations) → per-story smoke → commit/push, plus per-epic branch management, retrospective, and merge gates. Dev/QA/review stages are spawned as subagents with an explicit stage→model map (Sonnet implements, Opus reviews).
- **`_bmad/custom/skill-rules.md`** — a cross-cutting rules registry (integration ACs, real-runtime test evidence, ADR enforcement, unattended-menu protocol, finding-disposition bar, and more) loaded by every BMAD skill via `persistent_facts`.
- **`_bmad/custom/bmad-*.toml`** — customization files wiring the rules registry into the story-creation, dev, QA, and code-review skills.

### 2. `skill-optimization-prompt.md` — model-tier pass (optional)

A cost/quality optimization pass over the installed BMAD skills. It writes a `model:` pin into each skill's `SKILL.md` frontmatter following an **expensive planner/reviewer + efficient implementer** strategy:

- **Opus** — architecture, spec/PRD authoring, story creation, and every adversarial review gate.
- **Sonnet** — implementation, test writing, research, documentation, personas (the high-frequency workhorses).
- **Haiku** — mechanical status/routing/indexing skills only.

It also documents the per-project model-policy file (`_bmad/custom/model-overrides.yaml`) with telemetry-gated escalation for uncommon stacks (e.g. InterSystems ObjectScript) and an opt-in review-lane de-escalation experiment. Declining this pass is a valid configuration — `/epic-cycle`'s built-in stage→model map still governs the spawned pipeline stages either way.

### 3. `parallel-epic-cycle-workflow-creation.md` — parallel kit (optional)

Upgrades an installed `/epic-cycle` with **parallel epic execution**:

- **Orchestrator/runner modes** — the orchestrator analyzes epic dependencies (user-approved DAG), then dispatches background runner agents, each owning one epic end-to-end.
- **Submodule-aware git worktrees** — provisioning/teardown scripts (`_bmad/scripts/new-epic-worktree.sh` / `remove-epic-worktree.sh`, cross-platform bash) that mount modified submodules as gitlink-path worktrees with unpushed-work guards.
- **Runtime-lock protocol** — claim-file locks serializing stages that touch a shared live runtime (smoke, QA, ADR verifications).
- **Serialized, user-gated merge queue** — one epic at a time into the feature branch, submodules-first, with gitlink re-recording and conflict policy.

It patches the base kit's command at exact anchors (six grep-guarded edits), appends Rules 10–12 to the rules registry, and adds `_bmad/custom/parallel.yaml` for per-project tuning.

## Installation

### Prerequisites

- [Claude Code](https://claude.com/claude-code)
- A BMAD Method v6 project: `npx bmad-method install` (modules `core,bmm`, IDE `claude-code`)

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
- Re-running the **base kit** overwrites `.claude/commands/epic-cycle.md`, which removes the model pin and the parallel kit's patches — re-run steps 3 and 4 afterward.
- The parallel kit's Step 1a compares `Kit-Version` headers against `_bmad/custom/kit-versions.yaml` and offers stale companions for re-run in canonical order.

## Usage

After installation, in a fresh Claude Code session in the project:

```
/epic-cycle 2      # run epic 2 (classic sequential flow)
/epic-cycle 2-4    # run epics 2 through 4; with the parallel kit + parallel.yaml,
                   # this triggers dependency analysis and concurrent runner dispatch
```

Unattended runs use `bypassPermissions` on spawned agents — run them only in a trusted or isolated environment.

## License

[MIT](LICENSE)
