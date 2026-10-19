# Epic Cycle Workflow — Installation Kit

**Kit-Version:** 2026-10-19.1

**Requires BMAD Method 6.11.0–6.12.x, not v7** (`_bmad/_config/manifest.yaml` → `installation.version`; 6.12.2 recommended — its helper scripts write UTF-8 on Windows). BMAD v7, which the 6.12.2 installer advertises as a preview, replaces what this kit drives; Step 1 and the command's pre-flight refuse it. This kit drives the v6.11 Phase-4 chain — `bmad-sprint-planning → bmad-build-auto → bmad-code-review` — and its rendered skills need `uv` (Python 3.11+ provisioned by uv) on PATH. For BMAD ≤ 6.10 projects use the last pre-6.11 kit release; do not install this kit there.

A self-contained kit for installing the `/epic-cycle` slash command and its BMAD skill customizations into any BMAD project. Run this as a Claude Code session — the session reads each step, performs the indicated file operations, and verifies the result.

## What this installs

| File | Purpose |
| --- | --- |
| `_bmad/custom/skill-rules.md` | Cross-cutting rules registry, loaded by every BMAD skill via `persistent_facts` |
| `_bmad/custom/bmad-build-auto.toml` | Loads `skill-rules.md` for the unattended build skill (the plan + implement stages) |
| `_bmad/custom/bmad-build.toml` | Loads `skill-rules.md` for the interactive build skill and disables its `open_spec` editor launch (manual runs in a pipeline project) |
| `_bmad/custom/bmad-qa-generate-e2e-tests.toml` | Loads `skill-rules.md` for the QA skill |
| `_bmad/custom/bmad-code-review.toml` | Loads `skill-rules.md` for the code-review skill |
| `_bmad/scripts/ledger.sh` | Deferred-work ledger tool (bash + awk): the only reader/writer of `deferred-work.md` the pipeline uses — `load` / `slice` / `show` / `new` / `append` |
| `_bmad/custom/epic-cycle-unattended.md` | Doctrine for `/epic-cycle --unattended` and `--review`, read only under those flags (kit-owned; overwritten on re-install) |
| `_bmad/custom/unattended.yaml` | The unattended policy: what the loop may decide, what it parks for the owner, what stops it — project-owned, written from the template only when absent, never overwritten |
| `_bmad/scripts/unattended-inbox.sh` | The owner's inbox for unattended runs (bash + awk): an append-only journal, the rendered `inbox.md`, the run lock, hold requests |
| `.gitignore` | Gains `_bmad-output/unattended/` — the run state directory (inbox, run lock, resume prompts) stays out of git |
| `.gitattributes` | Gains `_bmad/scripts/*.sh text eol=lf` — the kit's bash scripts stay LF in every checkout and worktree |
| `.claude/commands/epic-cycle.md` | The slash command body (self-contained at runtime) |

Removed by this kit version (they targeted the deprecated `bmad-create-story` / `bmad-dev-story` shims, which the pipeline no longer invokes): `_bmad/custom/bmad-create-story.toml`, `_bmad/custom/bmad-dev-story.toml` — backed up, then deleted in Step 1.

Optional, project-defined later:

| File | Purpose |
| --- | --- |
| `_bmad/custom/branch-naming.yaml` | Per-project tracker / branch-naming convention overrides |

---

## Step 1: Detect prior state

Before writing any files, inspect what's already present and decide whether to back up.

Run these checks (the Claude session executes each; report findings inline):

```bash
# 0. BMAD version and runtime prerequisites (HARD gates)
grep -A1 "^installation:" _bmad/_config/manifest.yaml | grep version          # must be >= 6.11.0 and 6.12.x at most
# BMAD v7 gate — v7 (previewed with `npx skills add bmad-code-org/BMAD-METHOD` + `bmad setup`) replaces the review layers, sprint planning and story files this kit drives
grep -c '^\[\[workflow.review_layers\]\]' .claude/skills/bmad-build-auto/customize.toml   # >= 1 (6.11–6.12.x declare 4); 0 = BMAD-V7
ls .claude/skills/bmod-method/bmod.toml ~/.claude/skills/bmod-method/bmod.toml 2>/dev/null   # no output; a project hit = BMAD-V7, a user-directory hit only = V7-GLOBAL
uv --version                                                                  # must succeed
uv run --no-cache _bmad/scripts/resolve_config.py --project-root . --key core.project_name   # exit 0 = uv can run the BMAD scripts (Python >= 3.11); exit 3 = no Python 3.11+
test -f .claude/skills/bmad-build-auto/SKILL.md && echo "BUILD-AUTO-PRESENT" || echo "BUILD-AUTO-ABSENT"
test -f .claude/skills/bmad-sprint-planning/scripts/sprint_plan.py && echo "SPRINT-PLAN-PRESENT" || echo "SPRINT-PLAN-ABSENT"
test -f _bmad/scripts/render_skill.py && echo "RENDER-PRESENT" || echo "RENDER-ABSENT"
git check-ignore -q _bmad-output/implementation-artifacts && echo "OUTPUT-IGNORED" || echo "OUTPUT-TRACKED"   # must be TRACKED

# 1. Does the slash command already exist?
test -f .claude/commands/epic-cycle.md && echo "PRESENT" || echo "ABSENT"

# 2. Does it reference deprecated multi-agent patterns (excluding this kit's own Anti-Patterns bullet that forbids them), or is it the pre-6.11 command?
grep -E "TeamCreate|TeamDelete|team_name|shutdown_request|shutdown_response|STATUS: completed|STATUS: clarification_needed" .claude/commands/epic-cycle.md 2>/dev/null | grep -v "^- \*\*TeamCreate" | wc -l   # SendMessage is not a marker: the current kits name it in prohibitions and the parallel resume
grep -c "Halt after planning" .claude/commands/epic-cycle.md 2>/dev/null   # 0 with the file present = pre-6.11 command (the v6.11 command contains it many times)

# 3. Are the BMAD .toml customizations present? Which generation?
ls _bmad/custom/bmad-*.toml 2>/dev/null
ls _bmad/custom/bmad-create-story.toml _bmad/custom/bmad-dev-story.toml 2>/dev/null   # legacy (pre-6.11) files

# 4. Do any .toml files carry an `on_complete` hook?
grep -l "on_complete" _bmad/custom/bmad-*.toml 2>/dev/null

# 5. Does a prior rules registry exist?
ls _bmad/custom/*-skill-rules.md _bmad/custom/skill-rules.md 2>/dev/null

# 6. Does a ledger exist, and is it in the DW-entry grammar (Kit-Version >= 2026-08-27.1)?
test -f _bmad-output/implementation-artifacts/deferred-work.md && echo "LEDGER-PRESENT" || echo "LEDGER-ABSENT"
grep -c "^### DW-[0-9]*: " _bmad-output/implementation-artifacts/deferred-work.md 2>/dev/null   # 0 with the file present = legacy grammar
grep -cE "^- \*\*\[|^## Deferred from:|^- source_spec:" _bmad-output/implementation-artifacts/deferred-work.md 2>/dev/null   # legacy entry shapes

# 7. Unattended-mode files (Kit-Version >= 2026-10-19.1)?
test -f _bmad/custom/unattended.yaml && echo "POLICY-PRESENT" || echo "POLICY-ABSENT"
WB="$(sed -n 's/^worktree_base: *//p' _bmad/custom/parallel.yaml 2>/dev/null | sed 's/ *#.*//' | tr -d "\"'\r" | sed 's/[[:space:]]*$//')"; WB="${WB:-.worktrees}"
ls _bmad-output/unattended/run.lock "$WB/.coordination/unattended/run.lock" 2>/dev/null   # any present = an unattended run (sequential, or parallel) may be live
```

**Decision table:**

| Finding | Action |
| --- | --- |
| BMAD-V7 — `bmad-build-auto`'s installed `customize.toml` declares no `[[workflow.review_layers]]`, or `.claude/skills/bmod-method/bmod.toml` exists (a mixed install — a 6.12 manifest with v7 skills on top — counts). Check this row first: on v7 the rows below would wrongly advise an upgrade | HALT with: "This kit supports BMAD Method 6.11–6.12.x. This project runs BMAD v7 (<the marker found>), which replaces the review layers, sprint planning and story files the kit drives — its overrides would stop v7's renderer. To keep using these kits, keep the project on 6.12.x (`npx bmad-method@6.12 install --action update --modules bmm --tools claude-code --yes` in the project directory); try the v7 preview in a separate clone." Install nothing. |
| V7-GLOBAL — `bmod-method/bmod.toml` only under `~/.claude/skills` (a v7 preview installed for the user, not for this project) | HALT and ask: user-level skills with the same names can load alongside, or instead of, the project's 6.12 skills. Continue only once the user has removed the global preview, or confirms that the project's own skills take precedence in their setup — then record `v7_global_accepted: <UTC date>` in `_bmad/custom/kit-versions.yaml` (Step 5), which the command's pre-flight reads instead of halting on every run. |
| BMAD version < 6.11.0, or `BUILD-AUTO-ABSENT` / `SPRINT-PLAN-ABSENT` / `RENDER-ABSENT` | HALT. Tell the user to upgrade BMAD (`npx bmad-method install` in the project, choose the update path) or to use the pre-6.11 kit release. Nothing in this kit works against the old pipeline. |
| `uv` missing | HALT with the install pointer (https://docs.astral.sh/uv/). `bmad-build-auto` renders via `uv run` and halts on activation without it; every other BMAD skill resolves its customization through `uv run` too. |
| `OUTPUT-IGNORED` | HALT: `_bmad-output/implementation-artifacts` is gitignored. The pipeline's durable state (cycle logs, `sprint-status.yaml`, specs, `deferred-work.md`) must be committed on the epic branch — Rule 16's bookkeeping commits, resume, and the parallel merge queue all depend on it. Have the user drop `_bmad-output/` (or at least `implementation-artifacts/`) from `.gitignore` and commit the directory; resume after. |
| Slash command absent | Proceed to Step 2 — clean install. |
| Slash command present and the `Halt after planning` count (check 2, second command) is 0 | This is an upgrade from the shim-era pipeline. Back up, report, continue — the overwrite IS the upgrade. If a cycle log for an in-flight epic exists (`_bmad-output/implementation-artifacts/cycle-log-epic-*.md` with entries but no `epic_merged_to_feature`/`epic_merge_skipped`), WARN: that epic was logged by the old pipeline; the new command resumes it story-by-story (stories at `committed` are safe; a story mid-`dev_complete` under the old dev-story skill must be finished or reverted by hand first). Read the user the "Mid-epic upgrade" note in Step 5 before continuing, and HALT if a parallel run (`cycle-log-parallel.md` with open epics, or live `.worktrees/epic-*`) is in flight. |
| Any in-flight epic (same evidence) on a v6.11 command | Same "Mid-epic upgrade" note (Step 5): confirm with the user that no story stage is mid-spawn and the tree is clean, that this is not a live orchestrator run, and that the ledger migration (if offered) will be part of this upgrade commit. |
| Legacy `bmad-create-story.toml` / `bmad-dev-story.toml` present | Back up each to `<file>.bak-<UTC>`, then DELETE both. The skills they customize are deprecated shims the pipeline never invokes; leaving the files is harmless but misleading. If the user still runs those shims by name and wants the rules loaded there, they can restore the backups. |
| Slash command present, `Halt after planning` count > 0, zero deprecated-pattern matches (after the exclusion) | A v6.11 command written by this kit — a plain re-run/upgrade. Back up to `.claude/commands/epic-cycle.md.bak-<UTC>` and continue. |
| Slash command present with deprecated-pattern matches | Back up to `.claude/commands/epic-cycle.md.bak-<UTC>`, report the matching lines to the user, continue. |
| Slash command present containing `## Mode Resolution` (the parallel kit is installed) | Back up as above, then WARN the user before overwriting: this install removes the parallel kit's patches and any `model:` frontmatter pin. After Step 5 completes, re-run `skill-optimization-prompt.md` (if the project uses the model-tier pass) and `parallel-epic-cycle-workflow-creation.md` — both are idempotent. |
| `.toml` files absent | Proceed to Step 2. |
| `.toml` files present without `on_complete` | Back up each to `<file>.bak-<UTC>`, overwrite in Step 2. |
| `.toml` files present with `on_complete` | Back up each, overwrite in Step 2 (the new versions omit `on_complete`). |
| `_bmad/custom/skill-rules.md` present | Back up to `<file>.bak-<UTC>`. Before overwriting in Step 2, extract and PRESERVE: (a) the parallel kit's `## Rule 10` – `## Rule 12` blocks, re-inserted immediately before `## Rule 13` in the new file (so a parallel install is not silently un-installed); (b) everything under `## Project-specific rules` (rules numbered 21+ and any prose), re-appended verbatim. Report what was carried over. |
| `POLICY-PRESENT` (check 7) | Keep `_bmad/custom/unattended.yaml` — it is project-owned. Report its `version:` and any top-level key the File 8 template has that it lacks, so the user can add it. |
| A `run.lock` present (check 7) | Ask the user whether an unattended run is live (`bash _bmad/scripts/unattended-inbox.sh <its directory> run-status` shows its holder and heartbeat). If it is, stop: upgrade at a hold or after the run ends, never under a live run. |
| `_bmad/custom/epic-cycle-overlay.md` present | Keep it — no kit writes it. Step 3c re-applies its entries after Steps 2–3b (command entries after the parallel kit's patches when that kit is installed). A project upgrading from a kit version without the overlay moves its hand additions to the command and the overrides into it first — once. |
| `LEDGER-PRESENT` in legacy grammar (check 6: no `### DW-` headings but legacy shapes > 0) | Offer a one-time **ledger migration** (do it in a bookkeeping commit of its own, never mid-story): back up the file; for each legacy entry create a DW entry via `bash _bmad/scripts/ledger.sh <ledger> new …` carrying its summary, source, severity (default `med`), fix-risk (default `med`), footprint (default `out-of-footprint`), a one-line evidence digest, its LAST recorded status mapped onto the Rule 15 vocabulary (`deferred`/`open` → `open`; `wontfix` → `wontfix-accepted` with `reopen_if=unknown`; `resolved` → `resolved-by:<story or unknown>`), and an owner by class — **triage at migration, not later**: an entry only a human can close (a playtest, a measurement on hardware the pipeline lacks, an authored asset, a ratification) → `status=decision-pending owner=burndown note=human=<what>`; a product/preference call → `status=decision-pending owner=burndown`; story-closable work → `routed owner=<live story key, else burndown>`, except a LOW that is not a ≤15-line two-way door → `wontfix-accepted reopen_if=unknown` (Rule 15: `burndown` never owns a LOW). Report the class counts; the `decision-pending` entries reach the user at the first epic-close decision sheet (SC-4) and the next Story X.0, not at a burn-down story that cannot close them. Rename the backup to `deferred-work.legacy.md` (kept for forensics, never read by the pipeline). Report `total`/`open` before and after. A ledger with hundreds of entries is migrated by a spawned Sonnet subagent working from `grep -n` slices, never by loading the file into the lead. Declining the migration means the next epic's `ledger_load` gate will halt on the legacy file. |
| Prior rules registry under a different name (e.g., `<project>-skill-rules.md`) | Rename to `skill-rules.md` after backing up, OR back up + delete (the new install creates `skill-rules.md` from scratch — manual merge if the prior registry had project-specific rules). |

**Backup convention:** `<original-path>.bak-<UTC-timestamp>` (e.g., `.claude/commands/epic-cycle.md.bak-2026-05-22T14-30-00Z`). Backups stay in place so the developer can diff post-install.

Surface the detection report to the user before writing anything. If anything looks unusual (e.g., a prior registry with custom project-specific rules), pause and ask whether to preserve those rules in the new `skill-rules.md`.

---

## Step 2: Write BMAD skill customizations

### File 1: `_bmad/custom/skill-rules.md`

Write the following content verbatim (the block is delimited by FOUR-backtick fences because it contains three-backtick fences of its own — everything between the four-backtick lines is the file):

````markdown
# BMAD Skill Rules

Loaded as `persistent_facts` by every BMAD skill on activation (via `_bmad/custom/<skill>.toml`). Project-specific rules are appended under "Project-specific rules" at the end, numbered after Rule 20.

## Rule 1 — Integration ACs (`bmad-build-auto` / `bmad-build` planning step)

Every story spec that introduces a service, module, or shared component MUST include at least one Integration AC — as a row in the spec's I/O & Edge-Case Matrix or an item under Tasks & Acceptance — of the form:

> *Consumer `X` reads from this service/module and produces observable effect `Y`.*

The integration AC must be testable by the consumer's automation tier (unit, integration, E2E, browser-MCP, API smoke), not by inspecting the introducing module's internal state.

A spec that introduces a service without naming any consumers must explicitly say so in an "Integration ACs" note under `## Design Notes` ("No consumers in this story; the first consumer will be Story X.Y."). Silence is not acceptable.

## Rule 2 — Consumed-by linkage and reuse (`bmad-build-auto` / `bmad-build` planning step; reuse also binds implement and `bmad-code-review`)

Every service-introducing spec carries a `Consumed-by:` list under `## Design Notes` naming downstream consumer stories by ID and purpose.

Every consumer spec carries a `Consumes:` list under `## Design Notes`, and its Integration ACs exercise the consumer against a real instance — not a mock. These live in the non-frozen Design Notes section so later amendments never fight the spec's frozen intent block.

**Extend, don't copy.** When a story adds a class, method, function, or helper that resembles an existing sibling, the spec names the sibling it extends under `## Design Notes`, or records why a copy is the better choice. The implementation prefers extending; a copy carries its stated reason in the spec or the commit message — never in a narrating code comment. Code review files an unjustified copy as a MED naming both locations, and a copy with a weak reason as a LOW; a copy the spec forces is `routed` to the named story that will unify it, never closed `by-design` (a terminal entry is never scheduled). A preference, not a ban: some copies are right. (Field report 2026-10-10: a blind quality scan across three model tiers found the same weakness under all of them — new code written as a near-copy of a sibling, one method repeating about 60 of its sibling's 70 lines — and review rarely flagged it, because each copy works.)

## Rule 3 — Real-runtime test evidence (`bmad-code-review`)

A code review MUST NOT approve a story whose code touches a user-facing surface unless the QA-generated test suite includes at least one test that exercises the deliverable against its real target runtime:

- UI / browser-deployed — browser-MCP or Playwright test asserting on observable DOM / render state.
- CLI / library — actual invocation with stdout / stderr / exit-code / produced-file assertions.
- Service / API — a real HTTP request with status code + response body + side-effect assertions.

This is distinct from the lead's manual per-story smoke, which runs *after* code review as a separate workflow gate. Rule 3 governs the *test artifacts* code review can inspect; the manual smoke is a later, independent check.

Pure non-user-facing stories (build pipeline, internal tooling, refactor) are exempt; note the exemption in the review. Missing real-runtime test evidence on a user-facing story is a HIGH finding.

## Rule 4 — Closing summary in the final message (`bmad-qa-generate-e2e-tests`, `bmad-code-review`, under `/epic-cycle`)

`bmad-build-auto` is exempt: its contract is the spec file itself — frontmatter `status` (`done` | `blocked` | `ready-for-dev`) plus the `## Auto Run Result` section (`Status:` / `Blocking condition:`). The lead reads the spec, never the agent's prose. Every other skill, when invoked under `/epic-cycle`, MUST end its final assistant message with these sections in order:

```markdown
## Files Modified
- <full path from repo root>
(or "(none)")

## Tests Added
- <full path from repo root>
(or "(none)")

## Decisions
- <one-line summary>
(or "(none)")

## Issues Encountered
- <one-line summary>
(or "(none)")
```

The closing summary is part of the agent's normal output. If the agent forgets the sections, the lead reconstructs the file list from `git status --short` — normal extraction.

If the skill cannot make confident progress for ANY reason — ambiguous ACs, missing prerequisite, a user-preference choice, an environment failure, or anything risking a stated constraint — halt BEFORE the closing summary and end with a `## Clarification Needed` section instead. State the question, what was tried, and what's blocking, in one paragraph. **Unattended runs** (`/epic-cycle --unattended`): your spawn prompt's `## Unattended run` block says which small preference choices you decide yourself and how to record them; everything else still ends here.

Outside `/epic-cycle`, this rule does not apply — emit a normal completion summary.

## Rule 5 — NFR tripwire response (`bmad-build-auto`, `bmad-code-review`)

If an NFR is found to be unmeasurable, mathematically impossible, internally contradictory, or otherwise un-implementable as worded:

- **`bmad-build-auto`** cannot amend planning artifacts mid-run. HALT with status `blocked` and blocking condition `intent gap`, naming the NFR, why it is un-implementable, and the amendment you recommend. The lead amends the planning artifact (`prd.md`, `architecture.md`, or `epics.md`) in place, records original-vs-amended wording with rationale in the spec's `## Spec Change Log` and an inline `[AMENDED <date> — see the story change log]` marker at the amended text, re-runs the epic-context pre-warm (the amendment makes the planning artifacts newer than `epic-<N>-context.md`), resets the spec's `status` (`draft` to re-plan, `in-progress` to re-implement), commits, and re-dispatches on the spec path.
- **`bmad-code-review`** files an NFR worked around with code comments + `deferred-work.md` instead of a planning-artifact amendment as a HIGH finding.

**Who decides the amendment.** The lead — in a parallel run, the epic-runner IS the lead for its epic, and its own epic's story blocks in `epics.md` plus everything under `implementation-artifacts` are always within its authority (Rule 11 governs concurrency between epics, not this). Two tiers:

- **Apply and report** (no halt): an amendment that preserves the product promise — restating an observable or measurement so it is reachable/measurable while the intent is unchanged, correcting a wrong file/API/name the AC cites, and root-file bookkeeping another rule already mandates (an attribution row before a port lands, a shipped notices file kept consistent with `ATTRIBUTIONS.md`). Apply it under the discipline above and list it under `amendments:` in the story's `committed` entry (sequential) or the completion contract (runner) for post-hoc review.
- **Ask first** (`## Clarification Needed` / intent-gap halt to the user): anything that changes what the product does — narrowing, dropping, or deferring an AC or scope to another story/epic; touching another epic's story blocks, epic boundaries, or dependency statements; a new third-party dependency or licence; a numeric target with product meaning. These are the decisions a human owns; "I verified it is unreachable" is the evidence for the question, not a licence to answer it. Under `--unattended` a stage still escalates them; the lead classifies them against the project's policy (`_bmad/custom/unattended.yaml`; doctrine U5).

Either tier: if the amendment changes what an AD's Rule, a convention, or a Stack row says — or settles something the spine lists under `## Deferred` — the spine is updated in the same commit (Rule 20); a planning artifact that contradicts the spine is a conflict, not an amendment. If the amendment retitles a story, its sprint key changes: re-run `SPRINT_PLAN generate` with `--set <new key>=<the old key's status>` (generate drops the old key and would create the new one at `backlog`), then `LEDGER slice <old key>` → `append … owner=<new key> by=<gate> note=retitled` for every entry, in the same commit (Rule 17 (1)).

Do NOT work around with code comments + `deferred-work.md`.

## Rule 6 — AD violations are HIGH severity (`bmad-build-auto`, `bmad-code-review`)

The project's architecture decisions are the `### AD-n — {decision}` blocks (**Binds / Prevents / Rule**, optionally tagged `[ADOPTED]`) of its **architecture spine** — `ARCHITECTURE-SPINE.md`, written by `bmad-architecture` under `{planning_artifacts}/architecture/<run-folder>/` — together with its Consistency Conventions and Stack tables. The spine is BMAD's decision registry; this method has no separate ADR folder, and `/epic-cycle` keeps none. The lead resolves *which file* is the spine once per epic — the newest `ARCHITECTURE-SPINE.md` with frontmatter `status: final`, else the newest spine, else (pre-spine projects) any `*architecture*.md` under the planning artifacts, else none — logs `spine_resolved`, and passes its path into every spawn prompt. That is a read-side lookup only; writes to the spine happen at the moment of each decision (Rule 20), never batched to epic end.

An AC implementation that violates an AD — wrong tool stack, wrong architectural pattern, a dependency in a forbidden direction, a contradicted Rule — is a HIGH-severity finding, not a LOW deferrable.

`bmad-build-auto` must consult the spine for any architectural or methodology decision referenced in the story's ACs / spec Design Notes — at planning time (record the governing `AD-n` ids under `## Design Notes`) and at implementation time (match the implementation to each AD's Rule).

`bmad-code-review` must:

1. Cross-check each AC against the spine's ADs — the ids recorded under `## Design Notes` first, then any AD whose `Binds:` covers the story's area.
2. Verify that AD-constrained implementations match the AD's Rule.
3. File mismatches as HIGH. Auto-resolve inline where reasonable; otherwise pause for the lead.

(The identifiers `adr_verifications_complete`, `adr_constrained_acs`, and the parallel kit's `adr_verifications` lock stage keep their spelling — read them as "architecture-decision verifications" — so existing logs and `parallel.yaml` files stay valid.)

## Rule 7 — Sub-agent tool inventory is harness-inherited (all skills)

Sub-agents spawned by `/epic-cycle` inherit whatever MCP namespaces and tools are mounted on the harness running the lead. There is no project-local mechanism to add a tool just for sub-agents.

**Implication:** AD-tooled AC verifications (browser-MCP smokes, performance traces, audits) are placed on the **lead**, not on sub-agents. Sub-agent MCP propagation is best-effort defense-in-depth, not the primary gate. Note that `bmad-build-auto` itself spawns an implementation subagent and parallel review subagents (one nesting level below the stage agent); they inherit the same harness inventory.

## Rule 8 — Test discoverability (`bmad-qa-generate-e2e-tests`)

Generated tests MUST be discoverable by the project's default test suite — (a) correct naming convention, (b) not excluded by ignore files, (c) not tagged in a way that opts them out of the default run.

A test that exists but does not run in the default suite is invisible to CI and to the next story's regression check. Undiscoverable tests are a HIGH finding on subsequent code review.

## Rule 9 — Unattended menu protocol (`bmad-code-review`, `bmad-qa-generate-e2e-tests`, under `/epic-cycle`)

Interactive BMAD skills contain checkpoints — numbered menus, "halt and wait for confirmation" steps (`bmad-code-review` has four on its happy path plus a large-diff chunking offer; `bmad-qa-generate-e2e-tests` opens by asking what to test). `bmad-build-auto` has none (it is the unattended variant by design — its only exits are the HALT statuses); `bmad-build` is never a pipeline stage. When an interactive skill is invoked under `/epic-cycle`:

- If the spawn prompt **pre-answers** a checkpoint, take the pre-answered option and continue without waiting for a human.
- If a checkpoint is **not** pre-answered, it is a genuine decision point — stop and emit `## Clarification Needed` (Rule 4). Never guess, and never sit waiting for input that cannot arrive.
- **Exception — `/bmad-retrospective`:** deliberately human-in-the-middle even under `/epic-cycle`. Its checkpoints, party-mode dialogue, and WAIT points are NEVER pre-answered, and it is never invoked with `-H` / `--headless`. It runs lead-side, so every elicitation reaches the user directly. The one exception to the exception: an `--unattended` run whose owner chose a headless retrospective in the project's policy (`gates.retrospective: headless`) runs it lead-side as `-H <epic>` and reads its verdict (doctrine U9).

Outside `/epic-cycle`, checkpoints elicit the user normally.

## Rule 13 — Working-directory, target, and read discipline (all skills and their internal subagents, under `/epic-cycle`)

Every spawned agent operates from the absolute working directory stated in its spawn prompt — in a parallel run that is the epic worktree root, never the main checkout. Verify `git rev-parse --show-toplevel` equals the stated path BEFORE reading state, editing files, or auditing. Skills that spawn internal subagents (`bmad-code-review`'s review layers `blind-hunter` / `edge-case-hunter` / `verification-gap` / `acceptance-auditor`, `bmad-build-auto`'s implementation-handoff subagent and its review layers, party-mode personas) must pass the path and the verification requirement — **together with every `🚫` prohibition in their own spawn prompt** (no `git commit` / `git push` / `git reset` / `git rebase`, nothing that mutates a remote or triggers CI or a deploy) — into every internal prompt, in the same breath. Only the layer that spawned a subagent commits; a subagent that reports having committed or pushed is a protocol violation — verify that `git rev-parse HEAD` still equals the sha recorded before the spawn, and check `gh run list`, instead of trusting the self-report — never `git log --branches --not --remotes`, which lists every branch's unpushed commits (the story's own, and in worktrees other epics') — then halt to the layer above. (Incident 2026-08-28: a depth-3 handoff subagent received the working directory because Rule 13 mandated it and never received the git prohibition because nothing did; it committed, pushed, and triggered a live deploy.) Findings produced from the wrong checkout are invalid — discard and re-run; never report them. (Pilot 2026-07-10: an Acceptance Auditor defaulted to the main checkout, saw a clean tree, and reported a completed story as unimplemented — 5 false CRITICAL findings.)

**Runtime targets get the same discipline.** Every tool call and script that touches a runtime — a database, an app server, an emulator, a test instance — names its target explicitly; a tool that would silently fall back to a default target (a default connection profile, `localhost`, the first matching container) is configured to fail instead, because a misrouted call to a healthy, unrelated instance returns plausible results and no error. Verify the target's identity (name, port, version), not just its health, before acting. An agent stops, recreates, or tears down only runtime resources its own session created and can name from its own transcript; everything else is reported and left running — a banner or a generated file that looks like yours is not evidence. (Field report 2026-10-10: a call that named no target answered from an unrelated instance with the same namespace and credentials; a lead tore down the owner's second dev instance because a generated file's banner looked like its own.)

**Read once, then work from what you hold.** Read each document once and work from the copy in your context. Re-read only a file that may have changed since you read it — your own edit, a subagent's, a merge, or after your context was compacted — and then only the changed region, by line range, when you know it. Write a generated diff to one file and consult it by section rather than regenerating it. Large shared documents are read in the parts you need: the plan stage maps the story's `AD-n` blocks from the architecture spine; later stages read the ADs the spec's `## Design Notes` cites (and any whose `Binds:` covers the story's area), not the whole spine. Pass this rule into every subagent prompt with the working directory. (Field report 2026-10-10: a 115 KB spine was paged through by 59 agents in 36 hours, and stage contexts grew from about 57K tokens at spawn to 350K–450K. The saving is fewer agents reading each document, in the parts they need — few agents re-read anything.)

## Rule 14 — Source-text discipline: ASCII escapes, contract-only comments (`bmad-build-auto`, `bmad-qa-generate-e2e-tests`, `bmad-code-review`)

Author non-ASCII characters in source code via escape sequences (`'\u2026'`, `'\uFEFF'`), never as literal bytes — literal invisible or exotic characters are unreviewable in diffs and get silently normalized by editing tools. A literal non-ASCII byte in code is a review finding; the fix is the escape form. Prose files (markdown docs, comments where project convention allows) are exempt. (Pilot: a literal U+FEFF inside a BOM-strip regex and a raw U+2026 both required review patches; the reviewer's own Edit tool normalized the literal BOM away mid-patch.)

**Comments state the contract, never the history.** A doc comment says what the code promises and what it needs; review rounds, findings, and rework narration belong in the spec, never in code. Correct a wrong claim by replacing it, not by appending a correction. A test's header says what it pins and what it needs. A rework pass's `## Auto Run Result` covers only that pass, and once the spec carries the template's `oversized` warning, a re-open appends only the open items. (Field report 2026-10-10: one early story's spec reached 664 KB against a template that aims for 900–1,600 tokens; doc comments made up about a third of its classes' lines, about a hundred of them narrating review rounds — and reviewers then filed findings against the narration.)

## Rule 15 — Finding disposition bar (`bmad-code-review` + lead/runner, under `/epic-cycle`)

Every review finding is dispositioned AT REVIEW TIME — fix, or close with a named reason. "Deferred" is an explicit exception, never a default state. Each finding carries four fields: severity (high/med/low), **fix-risk** (low/med/high + one-line justification), footprint (in-story / in-epic / out-of-footprint), spec-status (clear / spec-bound). Disposition is a lookup, not a debate:

| Finding | Disposition |
| --- | --- |
| HIGH | Fix now (rework loop, at any review while iterations remain). Sole exception: explicit user waiver via `## Clarification Needed`, logged `high_waived` — under `--unattended` a waiver is owner-only: the story parks. |
| MED, fix-risk ≤ med, **in-story** | Fix now — by the reviewer, in the review pass, whenever it can (it may edit files; the unattended menu applies every patch). What it cannot patch re-opens the story for the ONE bounded MED iteration (shared with the Fix Pack) if this is the story's first review; on a re-review it is `routed owner=<a later story key in this epic that touches the area, else burndown>` with its `DW-n:` bullet — never a second re-open (exit condition below). "The story that will touch it later" is still not a disposition: routing under this row is a named owner with a planned bullet (Rule 17 (1b)). |
| MED, fix-risk ≤ med, **in-epic** (a file this story did not touch but a story of this epic owns) | `routed owner=<the story key in this epic that owns the file and has not yet reached cr_complete, else burndown>` with its `DW-n:` bullet now (Rule 17 (1b)) — that story's plan spawn receives it as inbox and its gate adjudicates it. Does not block story `done`. (Field report 2026-09-11: in-epic MEDs treated as fix-now were one of six rules that re-opened one story eight times.) |
| MED, fix-risk high OR out-of-footprint | `escalated` — ledgered (`owner=burndown` is the grammar's parking slot, not a promise of work) and decided by the user at this epic's **decision sheet** (SC-4). Does not block story `done`. |
| LOW two-way door (≈≤15 changed lines incl. its test, **in-footprint — in-story or in-epic**, spec-clear, suite green after) | Fix in-story: mechanical → reviewer patches; behavioral-but-trivial → `## Fix Pack`, ONE bounded dev iteration. Leftovers take the rows below. |
| LOW that is not a two-way door (bigger than a fix-pack item, or out-of-footprint) | Close `wontfix-accepted` at emission with `reopen_if=<observable probe>`. A LOW not worth 15 lines now is not worth a story later. **`burndown` never owns a LOW** (field report 2026-08-29: 17 of 30 burn-down entries were LOWs routed instead of closed). |
| Fix would alter AC'd/specified behavior (spec-bound) | Close `by-design` at emission; rationale recorded once. Reopens only via spec amendment (Rule 5 path or an X.0 product decision). |
| No realistic user-reachable failure (theoretical hardening) | Close `wontfix-theoretical` at emission + one line naming what would make it real. |
| Real MED/HIGH issue, out-of-footprint | `routed` — ONE canonical ledger entry per root cause; later sightings append an `occurrence`, never a new entry. Occurrence count is a priority signal. |
| Genuine product/preference call | `decision-pending` — batched to the user at this epic's **decision sheet** (SC-4), or earlier — as an `intent gap` at spec validation — the moment a later story's spec depends on it; decided once; terminal either way. |
| Closable only by a human (a playtest, a measurement on hardware the pipeline lacks, an authored asset, a ratification) | `decision-pending` with `note=human=<what>`. No story can close it, so it never takes `routed owner=burndown`; at the decision sheet it moves to `sprint-status.yaml` `action_items` (owner: the user) and the entry closes `dropped note=action_item`. |

**Exit condition (the loop must be able to end).** A story is re-opened by a HIGH, by a first review's unpatched in-story MED (one bounded iteration), by a Fix Pack (the same iteration), or by a failed smoke / red CI — nothing else. **A re-review that finds no HIGH closes the story:** the reviewer patches what it can, files every remaining MED with an owner under the rows above (`routed` / `escalated`), and sets `done`; MEDs alone never re-open a reworked story. A decision taken on a story that has already reviewed with no HIGH (a check-in answer, a decision-sheet call on one of its `escalated` / `decision-pending` entries) does not re-open it either: it lands as a `DW-n:` acceptance bullet on a named later story or the burn-down story. (Field report 2026-09-11: under "fix every in-footprint MED now" plus a whole-story re-review after every rework, "complete" was not a reachable state — 9 implement passes and 3 reviews on one story with zero HIGH since the third review.)

**Ledger grammar** (`_bmad-output/implementation-artifacts/deferred-work.md`). One entry per root cause:

```
### DW-<n>: <one-line summary>
- source: <spec file or stage> | severity: <high|med|low> | fix-risk: <low|med|high> | footprint: <in-story|in-epic|out-of-footprint>
- evidence: <why this is real — written once, at most 3 lines>
- <UTC> status=<status> owner=<story-key|burndown> by=<stage> note=<at most 120 chars>
- <UTC> occurrence=<story-key>
```

The body is written once. Everything after it is an append-only **trailer** of single-line, fixed-grammar annotations, and the **effective status and owner are whatever the latest trailer line set** — latest by its UTC timestamp (equal timestamps: the later line), so the order a union merge leaves the lines in never decides it — an instrument reads the trailer, never the prose, so an entry grows by one short line per transition and never by accretion. Statuses — non-terminal: `open | routed | escalated | decision-pending`; terminal: `by-design | wontfix-theoretical | wontfix-accepted | dropped | resolved-by:<story-key>`. `wontfix-accepted` = real and reachable but decided not worth pursuing; its `note=` MUST carry `reopen_if=<observable probe>`. `dropped` = invalid, duplicate, or not pipeline work (moved to `action_items` — the note says so). Every non-terminal entry has exactly one effective owner: a story key (the story that will close it) or `burndown` (this epic's burn-down gate, Rule 17 — for `routed` entries that is story-closable MED/HIGH work; for `escalated` / `decision-pending` entries it means "for the user at the decision sheet"); `owner=none` is illegal after the epic's `ledger_load` gate. All reads and writes go through `bash _bmad/scripts/ledger.sh <ledger> load | slice | show | new | append` — the lead and every agent use `slice`/`show`, never the whole file (a mature ledger is megabytes), and write only with `new` (canonical entry, next id) or `append` (one trailer line). Entries and trailer lines are only ever added, so the file stays union-merge safe. **Owners are validated on write:** `ledger.sh` refuses an `owner=` that is neither `burndown` nor a `development_status:` key in the sibling `sprint-status.yaml` (a retitled story changes its key; an entry on the old key is invisible to every gate — field report 2026-08-30: two entries orphaned with no error and no count anomaly). `LEDGER load` reports `owner_unknown=N` and marks such owners `UNKNOWN`; `LEDGER slice unknown` lists them; `LEDGER_OWNER_CHECK=off` exists for migration only. Consult `slice all` (grep the summaries) before filing: an adjudicated root cause gets an `occurrence=<story-key>` line, never a new entry. Terminal-entry notes are calibration context for future reviews. Anything the size of real work is a story, not a fix-pack item.

**Every entry is born with an owner.** (a) `bmad-build-auto` never writes the ledger — it records deferred findings in the spec frontmatter `deferred:` list. The **lead harvests** each new item after `dev_complete` via `ledger.sh new` with `by=harvest`: in-story footprint → `status=open owner=<this story key>` (adjudicated at this story's gate); in-epic → `status=routed owner=<the story key that will touch it>`; out-of-footprint → `status=routed owner=burndown`. (b) `bmad-code-review` files its `defer` findings the same way (`by=cr`) instead of the skill's default `## Deferred from:` bullets — the spawn prompt says so. (c) The lead/runner writes every disposition and occurrence. An item that never reaches the ledger is invisible to the drain — the harvest is mandatory, not best-effort.

## Rule 16 — Clean tree before every `bmad-build-auto` dispatch (lead/runner, under `/epic-cycle`)

`bmad-build-auto` requires a clean working tree at its step-01 version-control sanity check (HALT on a dirty tree / `version-control metadata not writable`) — that check runs on story-key and intent dispatches (the plan stage); a spec-path dispatch early-exits past it — and it commits only the reviewed-diff files at finalize, then verifies the whole tree is clean (HALT `finalization left repository dirty`). Either way, anything uncommitted when a dispatch starts ends the run. The lead's write-ahead bookkeeping (cycle log, `sprint-status.yaml`, `deferred-work.md`, a freshly planned spec, the epic-context cache, QA's test summary, saved patches) therefore MUST be committed before every dispatch: **write-ahead → bookkeeping commit → spawn.** Bookkeeping commits stage only `_bmad-output/implementation-artifacts/**` (cycle logs, `sprint-status.yaml`, `deferred-work.md`, `spec-*.md`, `epic-*-context.md`, `tests/test-summary.md`, `*.patch`), plus the planning decisions the same commit records — the planning folder's `epics.md` (story insertions, Rule 5 amendments) and the architecture spine's folder (`ARCHITECTURE-SPINE.md`, `.memlog.md`; Rule 20) — and `_bmad/custom/model-overrides.yaml` for a model-tier change — never `.vscode/settings.json`, never source files — land on the epic branch under the SC-3 assertion, and use a `chore(epic-N): bookkeeping <story-id> <stage>` message — committed with an explicit pathspec (`git add -- <paths> && git commit -m … -- <paths>`; the add because a pathspec commit refuses a path git does not know yet, such as a new spec or patch), never a bare `git commit`, so nothing else that happens to be staged rides along. This requires `_bmad-output/implementation-artifacts` to be **tracked** (not gitignored) in the project — the pre-flight checks it. A `blocked` result naming either condition above is a missed bookkeeping commit until proven otherwise — fix the tree, do not re-spawn blindly.

**The one sanctioned source-file bookkeeping commit — the rework commit.** Before a rework re-dispatch (Rework Loop, Fix Pack, or a failed smoke that re-enters implement), QA's test files and the reviewer's applied patches are still uncommitted source changes (those spawns are forbidden to commit). The lead commits them together with the re-opened spec and the bookkeeping files as `chore(epic-N): rework <story-id> iteration <k>` on the epic branch (SC-3 assertion) so the tree is clean for the dispatch. That commit is part of the story's history, not a violation of this rule. Under `--unattended`, the doctrine's U4 adds two: a set-aside — one forward commit that restores a stopped story's source paths to its `story_base`, its changes saved as a patch — and a pick-up, the forward commit that applies that patch again once the owner has answered.

## Rule 17 — The drain is a stage, not a sentence (lead/runner, under `/epic-cycle`)

Every gate that files into the ledger is matched by a **logged stage that closes entries**; a ledger with writers and no drain is monotonic by construction (field report 2026-08-27: 0 → 1,588 entries in 114 days, never once shrinking, under a version of this workflow that filed at thirteen gates and read at one). The drain has four parts, each enforced by the cycle log:

1. **Born owned, and the owner exists** (Rule 15): a harvested or reviewer-filed entry carries `owner=<story-key>` or `owner=burndown` from its first trailer line, and the key is validated against the tracker at write time. The epic-start `ledger_load` gate re-owns any stray `owner=none` and repairs any `UNKNOWN` owner (`LEDGER slice unknown`): a retitle keeps the `<epic>-<story>-` prefix, so when exactly one tracker key shares the phantom's prefix the gate re-owns to it (`by=load note=retitle_repair`, logged `ledger_owner_repaired`); otherwise it HALTs — an entry no gate can reach is worse than a halt. A Rule 5 amendment that retitles a story re-owns that key's entries in the same commit.
1b. **Routed means planned.** Whenever an entry becomes `routed owner=<story key>` — at harvest, at `cr_complete`, at adjudication re-own, at the burn-down overflow, at the decision sheet — the party with authority over that story's block in `epics.md` (the lead/runner for its own epic; the sequential lead or the orchestrator, post-merge, for another epic) inserts `- DW-n: <summary> (ledger; routed by <gate> <date>)` under the receiving story's acceptance criteria, at most `routed_story_max` per story (default 6; `routed_max:` in `_bmad/custom/ledger.yaml`), the excess staying ledger-only and counted. The plan spawn for that story receives its full `LEDGER slice <key>` as an inbox and must address or decline each entry; spec validation checks that it did (`owned_ledger=`). Adjudication then checks a promise the story was shaped around instead of discovering a surprise it was never designed to fix (field report 2026-08-30: fifteen entries on one unplanned story; `ledger_adjudicated` can audit but not steer).
2. **Per-story adjudication** — `ledger_adjudicated`, mandatory between `cr_complete` and `smoke_complete`: every entry whose effective owner is this story's key is dispositioned against the story's DELIVERED scope (the diff since `story_base` — the baseline its first implement pass recorded, which no rework overwrites — and its tests), never its title — `resolved-by:<key>` with evidence, re-owned to a named story key with the residual restated in the note, or terminal. `ledger.sh slice <key>` reads empty before smoke.
3. **Burn-down gate** — `ledger_burndown_complete|skipped`, mandatory after the epic's last `committed` and before `epic_status_done`. Its input is **story-closable work only**: the `routed`/`open` entries owned by `burndown` or by any of this epic's story keys. `escalated` and `decision-pending` entries are not work — they are counted (`to_decision_sheet=N`) and handed to the decision sheet (5), never re-owned or "decided" here. What remains is resolved or made terminal now, chartered into ONE burn-down story (**`N.<M+1>`, M = the highest existing story number under Epic N across BOTH `epics.md` and `sprint-status.yaml`** — the tracker can hold hotfix stories the epics file does not; gaps are irrelevant and `N.0` counts, so the number always sorts last and never collides. There is no conventional number: a fixed ninth-story number collided on every epic with nine or more stories, and "the next unused number" filled a gap below an existing story. An "Epic N burn-down" story that already exists — a resumed gate — is reused, never duplicated) of at most `burndown_story_max` entries (default 12) chosen by priority — severity, `occurrence=` count, low fix-risk first — that runs the full per-story pipeline before the epic closes, and the explicit **overflow** is re-owned to a SPECIFIC next-epic story key with the reason in the note and counted (`overflow=N`). "The next X.0" is not an owner; `burndown` ownership does not survive the gate. The gate reports the epic's drain ratio (`filed_this_epic` / `closed_this_epic`, from `LEDGER load` at `ledger_load` and now) so an under-sized drain is a number, not an inference.
4. **Story X.0 is a CLOSE, not a relabel:** its outcomes are `resolved-by`, own (X.0 itself or a named story key in this epic), or terminal. "Defer with rationale" does not exist.
5. **The epic-close decision sheet — `decision_sheet_recorded`, at the SC-4 merge gate.** Every `escalated` and `decision-pending` entry (any owner) is presented to the user in one sitting, each with a recommended disposition, and decided once: a product call becomes terminal or owned by a named story; a human-owned item (`note=human=`) moves to `sprint-status.yaml` `action_items` and closes `dropped note=action_item`. The human's decisions are written as trailer lines `by=merge_gate` — in the epic-branch checkout before the merge (sequential lead) or in the main checkout on the feature branch immediately after it (orchestrator; the ledger is union-merge, so ordering is irrelevant). A runner never decides these: it lists them under `decisions_for_user:` in its completion contract. This is the one place a human sees the ledger, and it is a gate the human already attends. Under `--unattended` the lead classifies each entry against the project's policy instead (doctrine U9): owner-only entries wait in the owner's inbox with the trailer `note=inbox=<id>`, and the decision sheet, the burn-down gate and X.0 skip them until the owner answers.

Resume refuses `smoke_complete` without (2) for that story and refuses `epic_status_done` without (3) for that epic; SC-4 does not ask the merge question until (5) has been presented. Planning sees the load: `ledger_load` runs before sprint planning and its counts go into the sprint-planning argument and the retrospective's context. A rule that says "periodic burn-down" with no stage that executes it never runs.

## Rule 18 — Containment: returned agents are finished, prohibitions reach every depth (all agents at all depths, under `/epic-cycle`)

1. **A returned agent is finished.** `SendMessage` to any agent whose `Agent` call has already returned is forbidden at every depth and in every direction — supervisor→stage, stage→its own internal subagent, peer→peer (the four named cases elsewhere are examples, not the list). To give a returned subagent more work, spawn a fresh one with the work written into the durable artifact. The single sanctioned `SendMessage` in the whole system is orchestrator→runner (parallel kit). On this harness `SendMessage` is not synchronous: a "re-engage the subagent with a synchronous message" instruction (e.g. `bmad-build-auto` step-04's patch handling) MUST be treated as "it cannot be re-engaged" — apply the patches yourself or spawn a fresh subagent.
2. **Return exactly once.** Every agent returns when its work is complete and never before; an interim or progress return while its own subagents are still working strands them as orphan writers whose supervisor has gone. If you cannot finish, HALT properly (blocked / `## Clarification Needed`) — that is a completion, not an interim.
3. **Every prohibition propagates to every depth that can violate it**, in the same breath as the working directory (Rule 13): the git/CI/deploy prohibitions, this rule, and any `🚫` in the spawn prompt go into every internal subagent prompt. The base kit additionally writes them into `bmad-build-auto`'s `implementation_handoff` and every review layer's `instruction` via `_bmad/custom/*.toml`, so depth-3 prompts carry them by construction. **Stateful test execution against a shared runtime is part of the propagated list:** one test class or suite per tool call (a suite that would not finish inside one call's cap, item 5, is split into classes or batches), wait for it to complete, never several test-runner calls in one message (they run concurrently and race on shared fixtures), never a re-submit on a client-side timeout. (Field report 2026-09-11: a depth-2 handoff subagent issued 18 test-runner calls in one message; an uninstall raced an install and left a database mounted with no file behind it, blocking seven test classes until a human restart. The project's own ledger had recorded the timeout re-submit race a day earlier — in a file no prompt reads.)
4. **Orphan protocol** — on evidence of a live writer whose supervisor has returned (two-point mtime sampling of the tree ~45 s apart; a stop/agent listing that shows a returned agent still running): do NOT re-spawn (that adds a writer); attempt `TaskStop` by the orphan's agent id or name; if it is refused, ask the top-level session — the sequential lead, or the orchestrator in a parallel run — to `TaskStop` it, because the main session can stop background agents that another agent spawned (Claude Code ≥ 2.1.203); only if that fails too, wait for quiescence (three consecutive unchanged samples). Then reconcile the tree into a single commit, log `protocol_violation` with what was observed, and escalate to the layer above. A completion notice for the orphan, wherever it surfaces, is a hint that it has stopped; the samples still decide.
5. **Synchronous is stated, never assumed.** The harness may run an `Agent` call in the background unless the call passes `run_in_background: false` — current Claude Code builds do, and the word "synchronously" in a prompt does not set the flag — so every `Agent` call at every depth passes `run_in_background: false` (several such calls in ONE message still run in parallel and resolve together). The one exception is the parallel kit's epic-runner spawn, whose orchestrator is built to wait on notifications. Every command, test run, and build runs in the foreground of its own tool call — never `run_in_background: true` on a shell call. A foreground call is capped by the harness's Bash timeout — 2 minutes by default and 10 minutes at most (`timeout: 600000`) unless the launch environment raises `BASH_MAX_TIMEOUT_MS` — and a call that reaches its timeout is not stopped but moved to the background, where it dies when the subagent that started it returns. So every test or build call passes `timeout: 600000` (or the raised maximum), and a run that would take longer is split into foreground calls that each finish inside the cap, never detached: one test class per call is always safe; a batch of classes is sized from the project's recorded per-class timings, if it keeps any (name the file under Project-specific rules), to finish in about 8 minutes. Nothing waits on `Monitor` or on a notification. The harness itself steers long waits to `Monitor`, and for a subagent that is the trap: a subagent that arms `Monitor` and stops has returned, whatever it meant — ending the turn IS returning, and that is item 2's interim return. No agent ends its turn while anything it started is still running. This item propagates to every depth exactly like item 3. (Field logs from one project, 2026-09-10 → 2026-10-08: 33 `protocol_violation` entries of one shape — a stage, handoff, review layer, or test sweep still running in the background after its owner returned — several of them "despite" a prompt that said synchronously; three full test sweeps died mid-run and two handoffs kept writing with no supervisor. The kit had never named the flag. Later the same day, with the flag stated, a handoff whose sweep could not fit in one 10-minute call ran it in the background, armed two `Monitor`s, and returned; the sweep died at 10 of 88 classes — foreground-only without a batching rule is an instruction that cannot be obeyed.)
6. `ListAgents` reports process liveness, not protocol state: a `SendMessage`-resumed agent shows `running` after it has returned. It is sufficient to tell whether a runner is alive or dead; it is NOT sufficient to establish that stage discipline holds — for that, ask the owning runner or sample mtimes. Where `ListAgents` is not available, use what the harness reports for the runner's task (its completion notification, a task listing); if nothing establishes liveness, the liveness HALT applies.

(Incidents 2026-08-28, twice: a prohibition stated only at supervisor level was broken one level below it — first the git prohibition by a handoff subagent, then the `SendMessage` prohibition by a stage agent following the skill's own re-engage text. Each closed list was one direction short.)

## Rule 19 — Falsifiability and hermeticity: a test whose red has not been observed, or whose green depends on another test, is not evidence (`bmad-qa-generate-e2e-tests`, `bmad-build-auto` Matrix Test Audit, `bmad-code-review` `verification-gap`, the lead's ADR gate and pre-push check, under `/epic-cycle`)

For every acceptance criterion, its **pinning test** — the one assertion that stands for "this AC holds" — must be demonstrated to fail against a named **mutation**: state the smallest change to the code under test that would violate the AC, apply it, observe red, revert, confirm the working tree is byte-identical to before (`git status --short` and `git diff --stat` unchanged), and record it next to the test in the spec's `## Verification` as `mutation: <what was changed> → <which test went red>`. One demonstrated mutation per AC, not per assertion — the discipline scopes to the pinning test, not the suite.

Structurally unfalsifiable assertions are defects, not evidence: an expected value the type system makes the only possible value (`toEqual([])` against a `readonly never[]`), a subject that was never constructed (asserting a position with no ball spawned), two runs compared that both produce nothing, a test whose green depends on the defect it claims to guard against, a module with no executed test host. `verification-gap` flags every one of these and every AC whose pinning test has no recorded mutation. The lead's ADR gate may not record `result=pass` for an ADR-tooled AC without naming the mutation it applied — a pass with no mutation is `result=unverified` and is not a pass.

**Severity, and who writes the line (Rule 15 lookup).** An AC with **no pinning test**, or a pinning assertion that is **structurally unfalsifiable**, is a MED in-story finding with low fix-risk — the reviewer patches it in-pass or it takes the story's one MED iteration. A pinning test that **exists** but whose `mutation:` line is missing or stale is a **LOW the reviewer closes inside the review pass**: name the mutation, apply it, observe red, revert, and write or correct the line in the spec's `## Verification` (the one sanctioned spec edit in a review — it is a tracking section, not the intent) — it never re-opens a story. Whoever adds or materially changes a pinning test — QA, the implement stage, a rework pass — writes or updates its `mutation:` line in the same pass, so the next review finds nothing stale. (Field report 2026-09-11: 11 of 62 self-review mediums across nine implement passes were "no mutation line" on tests the previous rework had just changed — the loop manufacturing its own next round.)

**Hermeticity: a test passes for its own reasons.** Every test a story adds or changes creates and removes the state it asserts on, depends on no other test's leftovers and no run order, and waits for the system under test to signal that it is ready before acting. The plan names the shared surfaces the story changes — a `Shared surfaces:` line under `## Verification` listing any list, menu, roster, schema, or count that other tests assert on — and carries the standing criterion "every test added or changed passes on a fresh environment, in either order". The implement stage updates every existing test that asserts the old shape of a shared surface (search the test trees for it); QA writes self-contained tests; `verification-gap` and code review file a test that is not self-contained as a MED in-story finding. Before the story is pushed, the lead runs the story's added and changed tests on a fresh environment — rebuilt by the project's own recipe if it has one (record the recipe under Project-specific rules), else a clean checkout — once forward and once in reverse order; a red takes the failed-smoke path and is logged `fresh_check`. (Field report 2026-10-10: for a stretch, 6 of 7 stories' first CI runs went red on their own tests — stale shared-surface assertions, state the test never created, actions before the screen had loaded — each costing a rework cycle and a CI run of about 50 minutes; after this check, about 1 in 8.)

(Field report 2026-08-29: five classes of vacuous test across one epic — every one found by adversarial review or by a later story, none by a gate designed to catch it; the lead's own AD gate recorded `pass` for an assertion typed `readonly never[]`. Test count grew 518 → 659; discriminating power did not track it.)

## Rule 20 — Decisions with architectural weight go into the spine (lead/runner, under `/epic-cycle`)

The spine is the source of truth for architecture decisions, and the pipeline makes decisions: a clarification answer, a Rule 5 amendment, a decision-sheet call. A decision has **architectural weight** when it constrains more than the current story — a dependency direction or seam contract (a new or amended AD), a convention independent builders would drift on (a Consistency Conventions row), a pinned language / framework / dependency / licence (a Stack row), or a numeric invariant later stories assume (an AD's Rule). Such a decision is written into the spine **at the moment it is made** — in the same bookkeeping commit as the clarification answer, amendment, or decision-sheet line — by the lead, in the spine's own grammar. It is never held for the end of the epic, a completion contract, or a later gate: a decision that exists only in an agent's context is lost to a crash, a context compaction, or a mid-epic upgrade, and the write-ahead doctrine applies to it exactly as to a cycle-log entry. (Parallel runs: each runner writes into its own worktree's spine the same way; only the next `AD-n` comes from the orchestrator's counter — parallel kit.) Everything else stays where it lands today (spec change log, ledger trailer).

**Light path (default).** Add the next `AD-n` — ids ascend and are never reused or renumbered — with Binds / Prevents / Rule and `[ADOPTED]`; or amend an existing AD's Rule keeping its id; or add/edit a Conventions or Stack row; or move an item out of `## Deferred`. Then: `uv run --no-cache _bmad/scripts/memlog.py append --workspace <spine-dir> --type decision --text "<AD-n: what, and what it prevents>"` (the memlog is what `bmad-architecture` treats as the authority on what was decided); `uv run --no-cache .claude/skills/bmad-architecture/scripts/lint_spine.py --workspace <spine-dir>` and require `"ok": true` — it exits 0 even when it found no spine to lint (`"ok": false` with no findings) — HALTing on `"ok": false` or on any `ad_id` / `ad_fields` / `placeholder` finding; refresh the spine's `updated:`; cite the AD id in the ledger trailer line, `## Spec Change Log`, or clarification record; bookkeeping-commit. The spine is a planning artifact, so the epic-context pre-warm re-runs before the next plan spawn and the decision reaches every later story. Log `spine_updated ad=<AD-n|conv|stack|deferred> reason=<clarification|rule5|decision_sheet> by=<lead|runner>`. **No spine file** (`spine_resolved` found only a fallback `*architecture*.md`, or none): there is no memlog and nothing to lint — `memlog.py append` raises without an existing `.memlog.md`. Write the decision into the fallback document when there is one (else into the spec's `## Design Notes` and the ledger trailer), log `spine_updated … path=<file|none> lint=n/a`, and recommend `bmad-architecture` to the user at the epic's close.

**Update intent (conflicts).** A decision that contradicts an existing AD is a re-architecture, not an amendment — never edit around it. Run `bmad-architecture`'s **Update** intent on the spine interactively (a human-in-the-middle run, like the retrospective) and let it resolve `conflicts_with_prior_decisions`; resume the pipeline only when the spine is `status: final` again and the pre-warm has re-run. Under `--unattended`, the doctrine's U7 makes "contradicts" testable and chooses between comply, amend (a veto item) and stop.

`decision-pending` entries with architectural weight are mirrored into the spine's `## Deferred` section with their DW id, so the spine also says what it has not decided yet. A spine written once and never updated is the failure mode BMAD's own docs warn about; a decision that lives only in a trailer line or a chat answer never reaches the next epic's plan stage (field case 2026-08-29: a licence decision and a tick-rate ratification recorded nowhere the plan stage reads).

## Project-specific rules (add below as retros surface them)

> Add additional rules here as retrospectives identify durable patterns. Number sequentially after Rule 20. Each rule should state what it applies to, the obligation, and (briefly) why.
````

The customization mechanism (BMAD ≥ 6.11): each skill ships `.claude/skills/<skill>/customize.toml` (installer-owned, overwritten on update); project overrides live in `_bmad/custom/<skill-dir-name>.toml` (team, committed) and `<skill-dir-name>.user.toml` (personal, gitignored). Scalars replace, lists append, arrays-of-tables merge by `id`/`code`. The key is the **skill directory name**. An override that fails to parse makes the *rendered* skills (`bmad-build`, `bmad-build-auto`) HALT; the non-rendered skills (`bmad-code-review`, `bmad-qa-generate-e2e-tests`, sprint planning, retrospective) instead fall back to a manual merge and continue silently — so Step 4's parse check is the only guard for those files, and it is mandatory.

**Derive the layer texts from the installed skill — never paste them blind.** Upstream renames placeholders and rewrites layer prompts between releases (6.12.0 replaced the inline `{diff_output}` with a diff file path, `{diff_file}`, and added `{claims_file}`; BMAD's unreleased next major drops `review_layers` for review lenses and halts the render on any override key the skill does not declare), and an id-keyed override replaces the *whole* layer, so a pasted older text would silently reference a placeholder that no longer exists. Procedure for File 2 and File 5: read the installed `.claude/skills/<skill>/customize.toml`; copy `implementation_handoff` (File 2) and every `[[workflow.review_layers]]` entry — `id`, `name`, `when` if present, `instruction` — **verbatim from the installed file**; then apply exactly these transformations: (a) append the containment paragraph/sentence shown below to the end of the handoff and of every layer instruction; (b) in `blind-hunter`, replace the line that sets a findings quota (6.12: "Compute your finding floor N … then find at least N issues to fix or improve."; 6.11: "Find at least ten issues to fix or improve.") and the line "If you have zero findings, re-check and keep thinking; do not stop with an empty list." with the softened lines shown below (if upstream rewords the quota again, soften whichever line carries it); (c) in `verification-gap`, append the Rule 19 falsifiability-and-hermeticity brief shown below after the layer's "Read … and follow it" line (or at the end if there is none); (d) on the launch line that opens the handoff and every layer instruction (the `Launch a … with this prompt:` line, which tells the composing agent to spawn), insert the clause "synchronously — pass `run_in_background: false` on the Agent call and wait for its report (several such calls in one message still run in parallel) —" immediately before `with this prompt:`, exactly as the listings below show (Rule 18 (5): the harness may background a launch that does not name the flag, and the word "synchronous" in the skill's own step files does not set it; a launch line that already names `run_in_background: false` is left as it is). The listings below are the result of that procedure on v6.12.0 / v6.12.1 (their `customize.toml` files are byte-identical) — use them to check your output, and paste them only when the installed default is byte-identical to the 6.12 text they were derived from. (The appended containment is a paragraph of its own in each layer, after the installed "Do not invoke any skill …" line.) Step 4 verifies that every `{placeholder}` in an override also appears in the installed `customize.toml`.

### File 2: `_bmad/custom/bmad-build-auto.toml`

Besides loading the rules registry, this override rewrites the prompts `bmad-build-auto` composes for its **depth-3 subagents** (the implementation handoff and the four review layers) so that Rule 18's containment reaches them by construction — the stage agent's diligence is no longer the only thing between a handoff subagent and `git push`. Scalars replace the skill defaults; the `[[workflow.review_layers]]` entries replace by `id`; the texts below are the v6.12 rendering of the derive procedure above (installed defaults plus the containment paragraph, with every launch line naming `run_in_background: false` per transformation (d)), with two deliberate deviations: the `blind-hunter` quota (6.12's finding floor "find at least N … do not stop with an empty list") is softened to an expectation, not a quota — a quota manufactures findings that Opus triage then pays to dismiss (field report 2026-08-29: 6–28 dismissed per review pass) — and the `verification-gap` layer carries Rule 19's falsifiability-and-hermeticity brief. A project may switch a layer off with `instruction = ""` in its override (deleting the entry does nothing — the merge brings the default back); switching off `edge-case-hunter` also drops the spec-claims check, which the skill's review step sends to that layer alone. Re-check them against `.claude/skills/bmad-build-auto/customize.toml` after a BMAD upgrade.

```toml
# /epic-cycle: rules registry + Rule 18 containment for the unattended build skill's internal subagents.

[workflow]
persistent_facts = [
  "file:{project-root}/_bmad/custom/skill-rules.md",
]

implementation_handoff = """
Launch a subagent with no prior conversation context, synchronously — pass `run_in_background: false` on the Agent call and wait for its report (several such calls in one message still run in parallel) — with this prompt:

> Read {spec_file} fully and implement it — the spec is the sole source of truth. Load every file listed in its frontmatter `context:` before you start.
>
> When done, report what you changed, how you verified it, and anything left incomplete or risky.
>
> Containment (/epic-cycle Rule 18): work only in the working directory you were given — verify `git rev-parse --show-toplevel` matches it before acting. You may edit files. You may NOT run `git commit`, `git push`, `git reset`, `git rebase`, or anything that mutates a remote or triggers CI or a deploy — the agent that spawned you commits. You may NOT `SendMessage` any agent, and you will not be messaged after you return: return exactly once, when your work is complete, with the full report above; a partial or interim return strands your work. Run stateful tests against a shared runtime (a live database, an instance, a dev server) one test class or suite per tool call and wait for each to complete — never several test-runner calls in one message, never a re-submit on a client-side timeout. Every call that touches a runtime names its target explicitly — never a tool's default target — and you tear down only what you started. Run every command and test in the foreground of its own tool call with `timeout: 600000` and wait for it — the default timeout is 2 minutes, and a call that reaches its timeout is moved to the background, where it dies when you return. Never `run_in_background: true` on a shell call, never a `Monitor` wait, and any `Agent` call you make passes `run_in_background: false`. Work that would outlast one call's maximum timeout is split into foreground calls that each finish inside it — one test class per call is always safe; batch classes only as far as the project's recorded timings allow — never detached. Arming `Monitor` and stopping is returning, whatever you meant: never end your turn while anything you started is still running — that is an interim return. Read each file once and work from what you hold; re-read only what may have changed since (your own edit, a merge), by line range. If you cannot finish, say exactly what is missing in the report and stop.
"""

[[workflow.review_layers]]
id = "blind-hunter"
name = "Blind Hunter"
instruction = """
Launch a context-free subagent synchronously — pass `run_in_background: false` on the Agent call and wait for its report (several such calls in one message still run in parallel) — with this prompt:

Conduct a review of CONTENT.
Look for what's missing, not only what's wrong.
Find every real issue to fix or improve. As a guide to how hard to look — not a quota — expect about N = min(floor(sqrt(kB) + 1), 10) of them, where kB is the diff file's size in kilobytes; if you find fewer, say so rather than pad the list.
Output a Markdown list of findings only — no severity, priority, or ranking.
If the content is empty, stop and say so.
If you have zero findings, re-check once; an empty list after a second pass is a valid result — report it as such, do not manufacture findings.

CONTENT: the unified diff at `{diff_file}`. Read that file — it is the content under review.

Do not invoke any skill, and do not spawn subagents of your own — you are the reviewer. Return your findings as text in your final message; do not route them through any findings-reporting tool the host may offer.

Containment (/epic-cycle Rule 18): you read and report only — do not edit files, do not run any git command that writes, do not SendMessage any agent, run at most one test class or suite at a time, in the foreground (never several test-runner calls in one message, never in the background), and return exactly once.
"""

[[workflow.review_layers]]
id = "edge-case-hunter"
name = "Edge Case Hunter"
instruction = """
Launch a context-free subagent synchronously — pass `run_in_background: false` on the Agent call and wait for its report (several such calls in one message still run in parallel) — with this prompt:

Read `{skill-root}/review-prompts/edge-case-hunter.md` completely and follow it as your review instructions.

claims_file (leave unread until your instructions call for it): {claims_file}

Review content: the unified diff at `{diff_file}`. Read that file — it is the content under review.

Do not invoke any skill, and do not spawn subagents of your own — you are the reviewer. If the instruction file is unreadable, report that exact failure and stop. Return your findings as text in your final message; do not route them through any findings-reporting tool the host may offer.

Containment (/epic-cycle Rule 18): you read and report only — do not edit files, do not run any git command that writes, do not SendMessage any agent, run at most one test class or suite at a time, in the foreground (never several test-runner calls in one message, never in the background), and return exactly once.
"""

[[workflow.review_layers]]
id = "verification-gap"
name = "Verification Gap Reviewer"
instruction = """
Launch a context-free subagent synchronously — pass `run_in_background: false` on the Agent call and wait for its report (several such calls in one message still run in parallel) — with this prompt:

Read `{skill-root}/review-prompts/verification-gap.md` completely and follow it as your review instructions. Additionally (/epic-cycle Rule 19, falsifiability and hermeticity): flag every assertion that cannot fail — an expected value the type system makes the only possible value (e.g. `toEqual([])` against a `never[]`), a subject that was never constructed, two runs compared that both produce nothing, a test whose green depends on the defect it guards against, a module with no executed test host — every acceptance criterion whose pinning test has no `mutation:` line in the spec's `## Verification`, and every added or changed test that is not self-contained: it relies on state it did not create, on another test's leftovers or on run order, or it acts before the system under test signals that it is ready.

Review content: the unified diff at `{diff_file}`. Read that file — it is the content under review.

Do not invoke any skill, and do not spawn subagents of your own — you are the reviewer. If the instruction file is unreadable, report that exact failure and stop. Return your findings as text in your final message; do not route them through any findings-reporting tool the host may offer.

Containment (/epic-cycle Rule 18): you read and report only — do not edit files, do not run any git command that writes, do not SendMessage any agent, run at most one test class or suite at a time, in the foreground (never several test-runner calls in one message, never in the background), and return exactly once.
"""

[[workflow.review_layers]]
id = "intent-alignment"
name = "Intent Alignment Auditor"
instruction = """
Launch a subagent synchronously — pass `run_in_background: false` on the Agent call and wait for its report (several such calls in one message still run in parallel) — with this prompt:

You are an intent-alignment auditor. You have no other context about how this change was produced. Here is the verbatim intent this work started from:

{verbatim_intent}

The diff is the unified diff at `{diff_file}`. Read that file — it is the change under review.

Your task is strictly descriptive — do not prescribe additional work. Report: (1) the defensible readings of the intent, enumerated; (2) which reading this diff implements; (3) where the readings and the diff diverge — specifically, which surface the intent's expectations live at versus which surface the diff's changes and its tests exercise.

Do not invoke any skill, and do not spawn subagents of your own — you are the reviewer. Return your findings as text in your final message; do not route them through any findings-reporting tool the host may offer.

Containment (/epic-cycle Rule 18): you read and report only — do not edit files, do not run any git command that writes, do not SendMessage any agent, run at most one test class or suite at a time, in the foreground (never several test-runner calls in one message, never in the background), and return exactly once.
"""
```

### File 3: `_bmad/custom/bmad-build.toml`

```toml
# /epic-cycle: the interactive build skill is NOT a pipeline stage, but developers run it
# by hand in pipeline projects. Load the rules registry, and disable open_spec (its default
# launches `code -r` — an editor window — which must never fire from an unattended session).
[workflow]
persistent_facts = [
  "file:{project-root}/_bmad/custom/skill-rules.md",
]
open_spec = ""
```

### File 4: `_bmad/custom/bmad-qa-generate-e2e-tests.toml`

```toml
[workflow]
persistent_facts = [
  "file:{project-root}/_bmad/custom/skill-rules.md",
]
```

### File 5: `_bmad/custom/bmad-code-review.toml`

Same pattern as File 2 — derive from the installed `.claude/skills/bmad-code-review/customize.toml`: the four review layers are the installed defaults plus Rule 18 containment (these layers only read, so the containment is short), the softened `blind-hunter` quota, and Rule 19's falsifiability-and-hermeticity brief on `verification-gap`. The listing below is the v6.12 rendering (the layers read the diff from `{diff_file}`; `acceptance-auditor` keeps its `when` gate on `{review_mode}`); the transformations never touch a placeholder.

```toml
# /epic-cycle: rules registry + Rule 18 containment for the code-review skill's review layers.

[workflow]
persistent_facts = [
  "file:{project-root}/_bmad/custom/skill-rules.md",
]

[[workflow.review_layers]]
id = "blind-hunter"
name = "Blind Hunter"
instruction = """
Launch a context-free subagent synchronously — pass `run_in_background: false` on the Agent call and wait for its report (several such calls in one message still run in parallel) — with this prompt:

Conduct a review of CONTENT.
Look for what's missing, not only what's wrong.
Find every real issue to fix or improve. As a guide to how hard to look — not a quota — expect about N = min(floor(sqrt(kB) + 1), 10) of them, where kB is the diff file's size in kilobytes; if you find fewer, say so rather than pad the list.
Output a Markdown list of findings only — no severity, priority, or ranking.
If the content is empty, stop and say so.
If you have zero findings, re-check once; an empty list after a second pass is a valid result — report it as such, do not manufacture findings.

CONTENT: the unified diff at `{diff_file}`. Read that file — it is the content under review.

Do not invoke any skill, and do not spawn subagents of your own — you are the reviewer. Return your findings as text in your final message; do not route them through any findings-reporting tool the host may offer.

Containment (/epic-cycle Rule 18): you read and report only — do not edit files, do not run any git command that writes, do not SendMessage any agent, run at most one test class or suite at a time, in the foreground (never several test-runner calls in one message, never in the background), and return exactly once.
"""

[[workflow.review_layers]]
id = "edge-case-hunter"
name = "Edge Case Hunter"
instruction = """
Launch a context-free subagent synchronously — pass `run_in_background: false` on the Agent call and wait for its report (several such calls in one message still run in parallel) — with this prompt:

Read `{skill-root}/review-prompts/edge-case-hunter.md` completely and follow it as your review instructions.

claims_file (leave unread until your instructions call for it): {claims_file}

Review content: the unified diff at `{diff_file}`. Read that file — it is the content under review.

Do not invoke any skill, and do not spawn subagents of your own — you are the reviewer. If the instruction file is unreadable, report that exact failure and stop. Return your findings as text in your final message; do not route them through any findings-reporting tool the host may offer.

Containment (/epic-cycle Rule 18): you read and report only — do not edit files, do not run any git command that writes, do not SendMessage any agent, run at most one test class or suite at a time, in the foreground (never several test-runner calls in one message, never in the background), and return exactly once.
"""

[[workflow.review_layers]]
id = "verification-gap"
name = "Verification Gap Reviewer"
instruction = """
Launch a context-free subagent synchronously — pass `run_in_background: false` on the Agent call and wait for its report (several such calls in one message still run in parallel) — with this prompt:

Read `{skill-root}/review-prompts/verification-gap.md` completely and follow it as your review instructions. Additionally (/epic-cycle Rule 19, falsifiability and hermeticity): flag every assertion that cannot fail — an expected value the type system makes the only possible value (e.g. `toEqual([])` against a `never[]`), a subject that was never constructed, two runs compared that both produce nothing, a test whose green depends on the defect it guards against, a module with no executed test host — every acceptance criterion whose pinning test has no `mutation:` line in the spec's `## Verification`, and every added or changed test that is not self-contained: it relies on state it did not create, on another test's leftovers or on run order, or it acts before the system under test signals that it is ready.

Review content: the unified diff at `{diff_file}`. Read that file — it is the content under review.

Do not invoke any skill, and do not spawn subagents of your own — you are the reviewer. If the instruction file is unreadable, report that exact failure and stop. Return your findings as text in your final message; do not route them through any findings-reporting tool the host may offer.

Containment (/epic-cycle Rule 18): you read and report only — do not edit files, do not run any git command that writes, do not SendMessage any agent, run at most one test class or suite at a time, in the foreground (never several test-runner calls in one message, never in the background), and return exactly once.
"""

[[workflow.review_layers]]
id = "acceptance-auditor"
name = "Acceptance Auditor"
when = 'Only when {review_mode} = "full".'
instruction = """
Launch a subagent synchronously — pass `run_in_background: false` on the Agent call and wait for its report (several such calls in one message still run in parallel) — with this prompt:

You are an Acceptance Auditor. Review the provided diff against `{spec_file}` and any loaded context docs. Check for: violations of acceptance criteria, deviations from spec intent, missing implementation of specified behavior, contradictions between spec constraints and actual code. Output findings as a Markdown list. Each finding: one-line title, which AC/constraint it violates, and evidence from the diff.

Diff: the unified diff at `{diff_file}`. Read that file — it is the content under review.

Do not invoke any skill, and do not spawn subagents of your own — you are the reviewer. Return your findings as text in your final message; do not route them through any findings-reporting tool the host may offer.

Containment (/epic-cycle Rule 18): you read and report only — do not edit files, do not run any git command that writes, do not SendMessage any agent, run at most one test class or suite at a time, in the foreground (never several test-runner calls in one message, never in the background), and return exactly once.
"""
```

### File 6: `_bmad/scripts/ledger.sh` (verbatim)

Cross-platform bash (Git Bash on Windows; bash 3.2+ on macOS/Linux; any awk). Invoked as `bash _bmad/scripts/ledger.sh …`, so no executable bit is required. Create `_bmad/scripts/` if absent (BMAD ≥ 6.11 already ships Python helpers there; this file sits beside them and is never touched by the BMAD installer, which only writes files listed in its own manifest).

```bash
#!/usr/bin/env bash
# Deferred-work ledger tool for /epic-cycle (skill-rules Rule 15 + Rule 17).
#
# Cross-platform: Git Bash on Windows, bash 3.2+ on macOS/Linux; needs awk (gawk, mawk, or
# BSD awk) and date. Output is pure ASCII key=value / TAB-separated text for the lead to parse.
# The lead NEVER reads the whole ledger: it uses `load`, `slice`, and `show`, and writes only
# through `new` and `append` (append-only trailer lines; union-merge safe).
#
# Entry grammar (one entry = one heading, a fixed body written once, then trailer lines):
#   ### DW-41: <one-line summary>
#   - source: <spec or stage> | severity: <high|med|low> | fix-risk: <low|med|high> | footprint: <in-story|in-epic|out-of-footprint>
#   - evidence: <why this is real, <= 3 lines>
#   - <UTC> status=<status> owner=<story-key|burndown|none> by=<stage> note=<short>
#   - <UTC> occurrence=<story-key>
# Effective status/owner = the trailer line with the LATEST UTC timestamp that sets each (equal timestamps: the later
# line), so the order a union merge leaves the lines in never decides it. Statuses:
#   non-terminal: open | routed | escalated | decision-pending
#   terminal:     by-design | wontfix-theoretical | wontfix-accepted | dropped | resolved-by:<story-key>
set -euo pipefail

FILE="${1:-}"; CMD="${2:-}"
usage() {
  cat >&2 <<'EOF'
usage: bash ledger.sh <deferred-work.md> <command> [args]
  load                        counts: total open routed escalated decision_pending terminal status_unknown owner_unknown, then owner:<key>=<n> for non-terminal
                              (an owner that is not `burndown` and not a key in the sibling sprint-status.yaml is suffixed " UNKNOWN")
  slice <owner>|all|unknown   non-terminal entries: DW-n TAB status TAB owner TAB summary (`unknown` = owners the tracker does not know)
  show DW-<n>                 print one entry verbatim
  next-id                     next unused DW number
  new "<summary>" "<source>" "<severity>" "<fix-risk>" "<footprint>" "<evidence>" "<status>" "<owner>" "<by>" "<note>"
                              append a canonical entry with the next id; prints DW-<n>
                              (with LEDGER_ID_COUNTER=<file> set, the id is claimed from that shared counter
                              under <file>.lock instead -- parallel epics, so two worktrees never issue the same id)
  append DW-<n> "<trailer>"   add one trailer line to that entry (UTC prepended), e.g.
                              "status=resolved-by:3-4-retry-hardening by=adjudication note=commit 9f8e7d6"
Owner validation: `new` and any `append` carrying owner= refuse an owner that is neither `burndown` nor a
`development_status:` key in <ledger-dir>/sprint-status.yaml (skipped when that file is absent, or with
LEDGER_OWNER_CHECK=off). A retitled story changes its key; a stale key would orphan the entry from every gate.
Status validation: `new` and any `append` carrying status= refuse a status outside the Rule 15 grammar
(LEDGER_STATUS_CHECK=off to bypass, migration only). A trailer's key=value fields end at its first `note=`:
text inside a note is never read as a field.
EOF
  exit 1
}
[ -n "$FILE" ] && [ -n "$CMD" ] || usage
case "$CMD" in load|slice|show|next-id|new|append) ;; *) usage ;; esac
if [ ! -f "$FILE" ]; then
  case "$CMD" in
    load) echo "total=0 open=0 routed=0 escalated=0 decision_pending=0 terminal=0 status_unknown=0 owner_unknown=0"; exit 0 ;;
    slice) exit 0 ;;
    next-id) echo 1; exit 0 ;;
    new) printf '# Deferred Work Ledger\n\nSee _bmad/custom/skill-rules.md Rule 15 (entry grammar) and Rule 17 (the drain).\n' > "$FILE" ;;
    *) echo "ERROR: $FILE not found" >&2; exit 1 ;;
  esac
fi

# Shared scanner: computes effective status/owner per entry, then acts per mode.
scan() {
  awk -v mode="$1" -v arg="${2:-}" '
    function terminal(s) { return (s == "by-design" || s == "wontfix-theoretical" || s == "wontfix-accepted" || s == "dropped" || s ~ /^resolved-by:/) }
    function known(s) { return (terminal(s) || s == "open" || s == "routed" || s == "escalated" || s == "decision-pending") }
    function emit(   k) {
      if (id == "") return
      n++; ids[n] = id; st[n] = status; ow[n] = owner; sm[n] = summary; blk[n] = block
    }
    /^### DW-[0-9]+: / {
      emit(); id = $2; sub(/:$/, "", id); summary = $0; sub(/^### DW-[0-9]+: /, "", summary)
      status = "open"; owner = "none"; sts = ""; ots = ""; block = $0; next
    }
    id != "" {
      if ($0 ~ /^### /) { emit(); id = ""; next }
      block = block "\n" $0
      if ($0 ~ /^- [0-9][0-9][0-9][0-9]-[0-9][0-9]-[0-9][0-9]T[0-9:]+Z /) {
        ts = $2
        for (i = 3; i <= NF; i++) {
          if ($i ~ /^note=/) break
          if ($i ~ /^status=/) { if (ts >= sts) { status = substr($i, 8); sts = ts } }
          else if ($i ~ /^owner=/) { if (ts >= ots) { owner = substr($i, 7); ots = ts } }
        }
      }
    }
    END {
      emit()
      if (mode == "load") {
        for (i = 1; i <= n; i++) {
          total++
          if (terminal(st[i])) term++
          else { cnt[st[i]]++; own[ow[i]]++; if (!known(st[i])) sunk++ }
        }
        printf "total=%d open=%d routed=%d escalated=%d decision_pending=%d terminal=%d status_unknown=%d\n", total, cnt["open"]+0, cnt["routed"]+0, cnt["escalated"]+0, cnt["decision-pending"]+0, term+0, sunk+0
        for (k in own) printf "owner:%s=%d\n", k, own[k]
      } else if (mode == "slice") {
        for (i = 1; i <= n; i++) if (!terminal(st[i]) && (arg == "all" || ow[i] == arg)) printf "%s\t%s\t%s\t%s\n", ids[i], st[i], ow[i], sm[i]
      } else if (mode == "show") {
        for (i = 1; i <= n; i++) if (ids[i] == arg) { print blk[i]; found = 1 }
        if (!found) { print "ERROR: " arg " not found" > "/dev/stderr"; exit 1 }
      } else if (mode == "next-id") {
        for (i = 1; i <= n; i++) { v = ids[i]; sub(/^DW-/, "", v); if (v + 0 > max) max = v + 0 }
        print max + 1
      } else if (mode == "exists") {
        for (i = 1; i <= n; i++) if (ids[i] == arg) found = 1
        exit found ? 0 : 1
      }
    }
  ' "$FILE"
}

now() { date -u +%Y-%m-%dT%H:%M:%SZ; }

# Parallel runs: every epic worktree holds its own copy of the ledger, so the local `next-id` (max + 1) would hand
# the same id to two epics, and the union merge keeps both headings. With LEDGER_ID_COUNTER pointing at a shared
# counter file (the parallel kit's coordination dir), `new` claims max(counter, local next) under an exclusive
# lock file and writes that + 1 back. A missing or non-numeric counter is an error, never a silent fallback.
# A claim holds the lock for milliseconds, so a lock older than 10 minutes was left by a crashed claim: it is broken
# (with a warning naming its content) rather than blocking every later `new` in every worktree.
claim_counter_id() {
  counter="$1"; lock="$1.lock"; tries=0
  until ( set -o noclobber; printf 'pid=%s acquired_at=%s\n' "$$" "$(now)" > "$lock" ) 2>/dev/null; do
    if [ -n "$(find "$lock" -mmin +10 2>/dev/null)" ]; then
      echo "WARNING: breaking stale $lock ($(cat "$lock" 2>/dev/null)) -- a claim takes milliseconds" >&2
      rm -f "$lock"; continue
    fi
    tries=$((tries + 1))
    [ "$tries" -lt 120 ] || { echo "ERROR: $lock held for 120 s by a live claim; report it -- never delete another party's lock" >&2; return 1; }
    sleep 1
  done
  n="$(cat "$counter" 2>/dev/null || true)"
  case "$n" in ''|*[!0-9]*) rm -f "$lock"; echo "ERROR: $counter is missing or not a number (the orchestrator seeds it)" >&2; return 1 ;; esac
  local_next="$(scan next-id)"
  [ "$n" -ge "$local_next" ] || n="$local_next"
  printf '%s\n' "$((n + 1))" > "$counter"
  rm -f "$lock"
  printf '%s\n' "$n"
}

# Owner validation (Rule 15/17). The tracker lives next to the ledger; its `development_status:` keys are the
# only legal owners besides `burndown`. Field report 2026-08-30: two entries sat on a retitled story's old
# key and were invisible to every drain, with no error and no count anomaly.
TRACKER="$(dirname "$FILE")/sprint-status.yaml"
tracker_keys() {
  awk '/^development_status:/ { f = 1; next } f && /^[^ #]/ { f = 0 } f && /^  [^ #][^ ]*:([ \t]|$)/ { k = $1; sub(/:$/, "", k); print k }' "$TRACKER"
}
# Status validation (Rule 15 grammar). A status outside it is neither terminal nor one of the non-terminal states the
# gates select by, so the entry silently escapes them; refuse it on write the way an unknown owner is refused.
check_status() {
  [ "${LEDGER_STATUS_CHECK:-on}" != "off" ] || return 0
  case "$1" in
    open|routed|escalated|decision-pending|by-design|wontfix-theoretical|wontfix-accepted|dropped|resolved-by:?*) return 0 ;;
  esac
  echo "ERROR: status=$1 is not in the grammar (open|routed|escalated|decision-pending|by-design|wontfix-theoretical|wontfix-accepted|dropped|resolved-by:<story-key>); LEDGER_STATUS_CHECK=off to bypass (migration only)." >&2
  return 1
}
# The key=value fields of a trailer end at its first `note=` token; nothing after it is a field.
trailer_field() { printf '%s' "$2" | awk -v k="$1" '{ for (i = 1; i <= NF; i++) { if ($i ~ /^note=/) exit; if (index($i, k "=") == 1) { print substr($i, length(k) + 2); exit } } }'; }
owner_check_active() { [ "${LEDGER_OWNER_CHECK:-on}" != "off" ] && [ -f "$TRACKER" ]; }
check_owner() {
  owner_check_active || return 0
  case "$1" in burndown|"") return 0 ;; esac
  # Readers here consume the whole stream: under pipefail, an early exit (grep -q, head) SIGPIPEs the producer
  # once the tracker outgrows the pipe buffer, and a valid key then reads as unknown.
  if ! tracker_keys | grep -Fx -- "$1" >/dev/null; then
    pre="$(printf '%s' "$1" | sed -E 's/^([0-9]+-[0-9]+)-.*/\1-/')"
    cand="$(tracker_keys | awk -v p="$pre" 'index($0, p) == 1 && n < 3 { printf "%s ", $0; n++ }')"
    echo "ERROR: owner=$1 is not a story key in $TRACKER (same-number keys: ${cand:-none}). Use the exact key or burndown; LEDGER_OWNER_CHECK=off to bypass (migration only)." >&2
    return 1
  fi
}

case "$CMD" in
  load)
    if owner_check_active; then
      scan load | awk -v keys="$(tracker_keys | tr '\n' ' ')" '
        BEGIN { n = split(keys, a, " "); for (i = 1; i <= n; i++) known[a[i]] = 1 }
        /^total=/ { hdr = $0; next }
        /^owner:/ { k = $0; sub(/^owner:/, "", k); sub(/=.*/, "", k); if (k != "burndown" && !(k in known)) { unknown++; $0 = $0 " UNKNOWN" } }
        { lines[++m] = $0 }
        END { print hdr " owner_unknown=" unknown + 0; for (i = 1; i <= m; i++) print lines[i] }'
    else
      scan load | awk '/^total=/ { print $0 " owner_unknown=0"; next } { print }'
    fi ;;
  slice)
    [ -n "${3:-}" ] || usage
    if [ "$3" = "unknown" ]; then
      owner_check_active || exit 0
      scan slice all | awk -F'\t' -v keys="$(tracker_keys | tr '\n' ' ')" '
        BEGIN { n = split(keys, a, " "); for (i = 1; i <= n; i++) known[a[i]] = 1 }
        $3 != "burndown" && !($3 in known)'
    else scan slice "$3"; fi ;;
  show)    [ -n "${3:-}" ] || usage; scan show "$3" ;;
  next-id) scan next-id ;;
  new)
    [ $# -eq 12 ] || { echo "ERROR: new needs exactly 10 arguments (see usage)" >&2; usage; }
    SUMMARY="$3"; SOURCE="$4"; SEV="$5"; RISK="$6"; FOOT="$7"; EVID="$8"; STATUS="$9"; OWNER="${10}"; BY="${11}"; NOTE="${12}"
    # The trailer writes its own `note=`; a caller that passed one (Rule 15 writes `note=human=<what>`) would record `note=note=...`.
    NOTE="${NOTE#note=}"
    case "$SUMMARY$SOURCE$EVID$NOTE" in *$'\n'*) echo "ERROR: arguments must be single-line" >&2; exit 1 ;; esac
    check_owner "$OWNER" || exit 1
    check_status "$STATUS" || exit 1
    if [ -n "${LEDGER_ID_COUNTER:-}" ]; then NUM="$(claim_counter_id "$LEDGER_ID_COUNTER")" || exit 1
    else NUM="$(scan next-id)"; fi
    ID="DW-$NUM"
    {
      printf '\n### %s: %s\n' "$ID" "$SUMMARY"
      printf -- '- source: %s | severity: %s | fix-risk: %s | footprint: %s\n' "$SOURCE" "$SEV" "$RISK" "$FOOT"
      printf -- '- evidence: %s\n' "$EVID"
      printf -- '- %s status=%s owner=%s by=%s note=%s\n' "$(now)" "$STATUS" "$OWNER" "$BY" "$NOTE"
    } >> "$FILE"
    echo "$ID" ;;
  append)
    [ -n "${3:-}" ] && [ -n "${4:-}" ] || usage
    ID="$3"; LINE="$4"
    case "$LINE" in *$'\n'*) echo "ERROR: trailer must be a single line" >&2; exit 1 ;; esac
    OWNER_TOK="$(trailer_field owner "$LINE")"; [ -z "$OWNER_TOK" ] || check_owner "$OWNER_TOK" || exit 1
    STATUS_TOK="$(trailer_field status "$LINE")"; [ -z "$STATUS_TOK" ] || check_status "$STATUS_TOK" || exit 1
    scan exists "$ID" || { echo "ERROR: $ID not found" >&2; exit 1; }
    TS="$(now)"
    # Insert the trailer as the last line of the entry (before the next heading or EOF). Pure line insertion: union-merge safe.
    # The trailer reaches awk through the environment: `awk -v` would turn C:\new or \d+ in a note into escapes.
    LEDGER_LINE="- $TS $LINE" awk -v id="$ID" '
      function flushpending() { print line; if (blanks != "") { printf "%s", blanks; blanks = "" } }
      BEGIN { inside = 0; line = ENVIRON["LEDGER_LINE"] }
      /^### DW-[0-9]+: / {
        if (inside) { flushpending(); inside = 0 }
        h = $2; sub(/:$/, "", h); if (h == id) inside = 1
        print; next
      }
      /^### / { if (inside) { flushpending(); inside = 0 } print; next }
      {
        if (inside) {
          # buffer trailing blank lines so the trailer lands directly after the last content line
          if ($0 ~ /^[[:space:]]*$/) { blanks = blanks $0 "\n"; next }
          if (blanks != "") { printf "%s", blanks; blanks = "" }
          print; next
        }
        print
      }
      END { if (inside) flushpending() }
    ' "$FILE" > "$FILE.tmp" && mv "$FILE.tmp" "$FILE"
    echo "$ID $TS $LINE" ;;
esac
```

### File 7: `_bmad/custom/epic-cycle-unattended.md` (verbatim; kit-owned)

The doctrine for `/epic-cycle --unattended` and `--review`. The command reads it only under those flags, so attended runs carry none of it. A re-install overwrites it; project additions belong in the policy (File 8) or the overlay. Write the following content verbatim (FOUR-backtick fences: the file contains a three-backtick block):

````markdown
# `/epic-cycle --unattended` and `--review` — doctrine

Kit-owned (written by `epic-cycle-workflow-creation.md`; a re-install overwrites it — project additions belong in `_bmad/custom/unattended.yaml` or the overlay). **Read this file in full when `/epic-cycle` is invoked with `--unattended` or `--review`, and again after every compaction.** Under those flags it overrides every passage of the command it names. Where it is silent, the command applies — except that inside the development loop any stop the command would put to the user is classified per U3 instead: an unattended loop never asks. An attended run reads it only for the case in U10's last bullet.

Names used below:
- **the policy** — `_bmad/custom/unattended.yaml` (project-owned; its categories are data). **This doctrine** — `_bmad/custom/epic-cycle-unattended.md`. In orchestrator mode both are read from the MAIN checkout (absolute paths), never from a worktree's copy.
- **`<state>`** — the run state directory, outside git's view: `_bmad-output/unattended/` (sequential; gitignored by the kit) or `<worktree_base>/.coordination/unattended/` (orchestrator mode). Always an absolute path in commands and prompts.
- **`INBOX`** — `bash _bmad/scripts/unattended-inbox.sh <state>`.
- **`SRC`** — the pathspec for a story's source changes: `. ':(exclude)<impl>' ':(exclude)<planning>' ':(exclude)_bmad-output' ':(exclude)_bmad' ':(exclude).claude' ':(exclude,glob)**/.gitignore' ':(exclude,glob)**/.gitattributes'` (with the resolved `<impl>` and `<planning>`). Bookkeeping, planning artifacts, the spine, the kit's files and every ignore or attribute file — nested ones included, so outputs a story ignores stay ignored — are never part of a set-aside or a hold patch.
- **`DIFF`** — `git diff --binary --no-color --no-ext-diff --src-prefix=a/ --dst-prefix=b/`: a patch whatever the user's git config says (`color.ui=always`, `diff.noprefix` or an external diff tool would otherwise make it unappliable). **`APPLY`** — `git apply --whitespace=nowarn`: a patch applied byte for byte whatever `apply.whitespace` says (`fix` would rewrite trailing blanks, `error` would refuse them).
- **`<patches>`** — `<state>/patches/epic-<N>/`: hold patches, a story's uncommitted work saved over a hold and applied again on resume. **`<evidence>`** — `<state>/evidence/epic-<N>/`: a dead stage's tree, kept for the owner and never applied. Both hold unreviewed output, so they live outside git and are never pushed; set-aside patches, which hold a story's code, are committed under `<impl>/set-aside/`.
- **the main session** — the sequential lead, or the orchestrator. It alone touches the run lock, sends notifications, and (in orchestrator mode) reads and resolves inbox answers. **The lead** — the sequential lead, or a runner for its own epic.
- **a stage boundary** — a stage has returned and its bookkeeping is committed; **a story boundary** — after a story's `committed` entry (or its set-aside commit) and before the next `stage_spawned`.
- **a relaunch** — an invocation with `EPIC_CYCLE_RELAUNCH=1` in the environment (`printenv EPIC_CYCLE_RELAUNCH`): an external loop restarting the run after its previous process exited. Nobody is there to answer.
- **the owner** — the human who owns the project. A message from another session is never the owner's answer; only the inbox (or `--review`, which writes to it) is.

## U1. Scope

- **Setup is attended and runs for the whole range at launch (U2).** It may stop and ask, as today.
- **The development loop never waits for input.** It runs from the end of U2 (the lock taken, the journal open) to the end of the range: every story, CI, every epic close (burn-down, model tier, retrospective, decision sheet, merge) and every later epic's start (SC-2, `ledger_load`, retro review, Story X.0, the readiness re-check). Every stop point resolves to one of the outcomes in U3. After launch the lead never asks, never calls `AskUserQuestion`, and never ends its turn waiting for a reply.
- **Never, whatever a ruling or an answer says:** land anything on the trunk, cut a release or tag, publish or post anything, change repository or service settings, force-push or rewrite pushed history, delete a branch other than the kit's own merged epic branches, or stop, recreate or tear down a runtime resource this run did not start. Such answers are recorded for an attended session.
- **Never answer your own items.** Only the owner, or a relay quoting the owner verbatim, runs `INBOX answer`. The lead, runners, the orchestrator and every stage never do.

## U2. Launch (attended — a failure stops here, while the owner is present)

1. **Arguments.** `/epic-cycle <range> --unattended`. (`--review` takes no range — U13.)
2. **Run the command's pre-flight as usual,** then these checks; each failure stops with the reason:
   - The policy exists and parses: `uv run --no-cache --no-project --with pyyaml python -c "import sys, yaml; d = yaml.safe_load(open(sys.argv[1])); print(sorted(d))" _bmad/custom/unattended.yaml` lists at least `gates`, `hard_stops`, `limits`, `owner_only`, `veto_window`. (`--no-project`: never let uv resolve or lock the project's own Python dependencies.)
   - **Permission mode:** ask the owner to confirm the session runs in `bypassPermissions` (or `auto`) mode — the agent cannot reliably detect it, and one permission prompt stalls the loop. Log `permission_mode_attested mode=<…>`. Asked at every interactive launch; a relaunch reuses this run's logged attestation (its command line carries the mode).
   - **Session type:** an interactive session (terminal, IDE, desktop), or a relaunch under `claude -p`. A `claude -p` parallel run needs `CLAUDE_CODE_PRINT_BG_WAIT_CEILING_MS=0`.
   - **CI mode.** If the pre-flight found `ci=none` or `ci=unavailable` and the policy's `ci.without_ci` is empty, ask the owner: `local_green` (merge on the full suite plus the smoke) or `hold` (merges wait; a sequential run then ends at the first epic close). Log `ci_mode_chosen without_ci=<…>`.
   - **Repository visibility:** `gh repo view --json visibility -q .visibility` (or ask); log it. On a public repository, pushing the epic branch is routine; a secret in anything pushed is a security incident (U3).
   - **Push notifications:** check whether a push-notification tool is available to this session (Claude Code's `PushNotification`, loaded with `ToolSearch` if deferred). If not, log `notify_unavailable` — the inbox alone carries the items.
   - **The run lock:** `INBOX run-status`. `none` → continue. A lock (exit 3) names a run id, holder and heartbeat age:
     - this run's id and a relaunch → the previous process exited: take the lock over at step 4;
     - otherwise ask the owner whether that session is still running — yes → stop (never two sessions on one run); no → take it over at step 4 with reason `owner-confirmed: <what they said>`. A relaunch that finds another run's lock stops.
   - A `<state>/HOLD`: a fresh launch clears one left by an earlier run (`INBOX hold-clear "fresh launch"`). A resume follows U11 — an interactive re-invocation clears it; a relaunch clears only a usage hold and honours any other.
   - **Orchestrator mode only:** the dependency graph is approved and its hash current.
3. **Setup for the whole range.**
   - Sequential: run the first epic's start as the command orders it (Task Sequence, per-epic steps 1–6), with every attended stop. For each later epic in the range, run headless sprint planning once now and read its readiness `gate`: resolve a FAIL or `blocked` with the owner now. Later epics' X.0 and retro review run in the loop (U8).
   - Orchestrator mode: the orchestrator's own launch work (graph approval, spine and ledger counters) plus headless sprint planning for every epic in the range in the main checkout, resolving FAIL or `blocked` with the owner now. Every epic start a runner performs after dispatch is loop work (U8).
4. **Start the run:** `INBOX init <run-id>` (`<run-id>` = `<UTC as YYYYMMDDTHHMMZ>-<range>`, or the logged id when resuming); take the lock — `INBOX run-start <run-id> lead` (sequential) or `… orchestrator`, or `INBOX run-takeover <run-id> <holder> "<reason>"` per step 2. Both print `token=<t>`: log it (`lock_token=`, on `unattended_started` / `unattended_resumed`) and pass it to every later `run-boundary` and `run-end`. (`run-start` exits 3 when a lock exists; `run-takeover` exits 1 on another run's lock without the owner's confirmation.) A `run-boundary` or `run-end` refused with exit 4 means another session has taken this run over: stop at once, writing nothing more. Log `unattended_started run=<id> policy=<git hash-object of the policy> doctrine=<git hash-object of this doctrine> range=<range> ci=<mode> visibility=<…> notify=<push|inbox-only>`; write the first resume prompt (U11). From here on, the loop runs unattended.

## U3. Outcomes and classification

| Outcome | Meaning | Record |
|---|---|---|
| **decide** | Rule using the policy's `product_defaults`, the spine and the rules; continue | `ruling … category=decide` (log and run report only — never the inbox) |
| **veto** | Decide now; the owner may reverse before the next release | `ruling … category=veto reversal=<what undoing it takes>`; `INBOX add veto …` |
| **park** | Only on an explicit `owner_only` match: the unit waits, independent work continues (U10) | `story_parked` (or the ledger marker, U9); `INBOX add parked …`; push |
| **stop** | The unit cannot proceed but nothing owner-only is at stake: stop the smallest unit that contains it (story, epic, slot or run) | `unattended_stop unit=<…> reason=<…>`; `INBOX add stopped …` (options for `--review`); push |
| **hold** | Finish the current stage, record everything, end cleanly (U11) | `hold_ordered`, `hold_complete`; push |

**Classify** a question by comparing it against the policy's `hard_stops` first, then `owner_only`, then `veto_window`; anything else is **decide**.
- **Park only on an explicit `owner_only` match.** Ambiguity between decide and veto resolves to **veto**, never to park.
- A security incident is checked first: a secret in a commit, log, trace or any printed output, a push to any remote other than `origin`, or a visibility change → stop the affected unit, never rewrite history, notify at once.
- Every other blocker is a **stop** of the smallest unit that contains it.
- A `ruling` line names the question, the choice, the rejected options and why, the category, and `policy=<hash>`. Commit subjects name a ruling that changed files.
- `INBOX add` reuses an open `parked`, `stopped`, `held-merge` or `retro` item for the same unit (`U-n EXISTS`); `veto`, `hold` and `report` items are always added. So a resume never re-adds a veto: each `ruling … category=veto` line records its `inbox=U-n`, and a ruling already logged is not added again.

**`limits`** (from the policy): `clarification_rounds` per stage per story; `split_children` per original story; `fix_forward_per_cause` per first-red sha; `gap_fill_per_epic`; `veto_open_max` open veto items (at the limit, merges to the feature branch are held until `--review`); `unverified_stories` during a CI outage. Passing a limit is a stop of that unit.

**Integrity at every story boundary:** `git hash-object` the policy and this doctrine (the main checkout's copies). A hash that differs from the last `unattended_started` / `unattended_resumed` entry, with no resume in between, stops the run: an agent edited the rules mid-run. The owner changes the policy during a hold and commits it (sequential: on the branch the checkout is on; orchestrator mode: in the main checkout); the resume logs the new hash. Anything under `_bmad/` and `.claude/` is `owner_only` except the kit's own run writes (`_bmad/custom/epic-dependencies.yaml` hash re-records; `_bmad/custom/model-overrides.yaml` for the mandatory rollback).

## U4. Setting a story aside, and picking it up again

A story that parks or stops **after its implement stage has produced code** is set aside, so its code never sits under later work. **`story_base`** is the latest `story_base=` logged for the story: on its first implement `stage_spawned` (the command logs it there and on the first `dev_complete`), on its first implement spawn after a set-aside or a re-plan, and at every pick-up.

**Set aside only when it is safe; otherwise stop the epic instead** (with an inbox item, and nothing restored):
- another story's commit sits on top of the story's own — a story whose `committed` sha descends from `story_base` (for example a successor stacked during a CI outage);
- the story changed a submodule (`git diff --no-ext-diff --no-color --submodule=short --ignore-submodules=none <story_base> -- SRC` shows a `Subproject commit` change, or `git status --porcelain --ignore-submodules=none` shows a submodule dirty — `--ignore-submodules=none` because a `.gitmodules` `ignore =` setting would hide both): submodule stories are not set aside;
- `story_base` does not resolve (`git rev-parse --verify <story_base>^{commit}` fails);
- an untracked file outside `SRC`'s exclusions that no stage reported (step 2).

**Steps:**
1. `mkdir -p <impl>/set-aside`. The patch is `<impl>/set-aside/<story>-<n>.patch`, `<n>` one past the story's existing patches.
2. Intent-to-add only the untracked files the story's stages reported adding (QA's `## Tests Added`, the review's `## Files Modified`): `git add -N -- <path>`, one call per path that exists (a reported path that is gone is skipped). Then `git status --porcelain --ignore-submodules=none -- SRC` must show no `??` — any other untracked file (a stage's scratch output, a report, an env file) stops the epic and is listed in the inbox; it is never patched or committed.
3. `DIFF <story_base> -- SRC > <patch>` — it must exit 0 (a failing diff is not an empty patch: stop the epic). An empty patch means nothing to take off: skip to step 5 — and if `bmad-build-auto` had already reverted the code and saved its own patch (a review-step intent gap, named in the spec's `## Review Triage Log`), log that patch's path on `story_set_aside`; the pick-up applies it through the re-dispatch protocol's review-step patch path. Otherwise `APPLY -R --check <patch>` must exit 0 — if not, stop the epic and restore nothing.
4. `git restore --source=<story_base> --staged --worktree -- SRC`. Check that the index now equals `story_base` on `SRC` (`git diff --cached --quiet <story_base> -- SRC`), that the tree matches the index (`git diff --quiet -- SRC`), and that `git ls-files --others --exclude-standard -- SRC` prints nothing. If not, put the work back (`APPLY --index <patch>` — it applies exactly onto the restored state) and stop the epic. If the put-back fails too (a file the OS would not release), commit the patch alone — `git add -- <patch> && git commit -m "chore(epic-N): save <story> patch" -- <patch>`, a pathspec commit, so the half-restored tree stays out — stop the epic, and list the tree's state in the inbox item.
5. Set the spec `status: ready-for-dev` and `SPRINT_PLAN generate --set <key>=ready-for-dev` (with the rework re-open, the only sanctioned downgrades). Commit: `git add -u -- SRC`; the story's ignore and attribute changes, which stay in force — `git add -u -- <path>` for each file `git diff --no-color --name-only -- ':(glob)**/.gitignore' ':(glob)**/.gitattributes'` lists (a `git add` pathspec that matches nothing would fail); then `git add` the patch and the bookkeeping files by path — never an untracked source file — and `git commit -m "chore(epic-N): set aside <story> (<U-id or reason>)"`. A forward commit, never a reset, so it is safe whether or not the story was pushed; it is Rule 16's second sanctioned source-file commit. Log `story_set_aside story=<key> patch=<path> sha=<commit>`. The story's inbox item also lists its planning changes, which stay in force (`git diff --name-only <story_base> -- <planning>`).
6. If the story had been pushed, push the set-aside commit and resolve CI for it like any head: green continues; red stops the epic.

**Picking it up again** (when its answer is applied — U10), only at a story boundary with a clean source tree and nothing staged (`git status --porcelain --ignore-submodules=none -- SRC` prints nothing and `git diff --cached --quiet` exits 0; if not, stop the epic — never reset over work). An empty patch has nothing to apply: re-implement the story from its spec, or, if `story_set_aside` named build-auto's own saved patch, take the re-dispatch protocol's review-step patch path with that patch.
1. `APPLY --3way <the story's latest patch>` on the current head. Judge by the exit code only: git 2.34 prints `does not exist in index` for new files on a clean apply.
2. Exit 0 → commit what the apply staged, at once, with a bare commit: `git commit -m "chore(epic-N): pick up <story> (<U-id>)"` (nothing else is staged — the gate checked; a pathspec commit would re-read modes from the tree and drop executable bits where `core.fileMode=false`). It is Rule 16's third sanctioned source-file commit, a forward commit like the set-aside: from here on, a crash, a hold or a dead stage can no longer lose the picked-up code. Log `story_unparked story=<key> via=<inbox|review> story_base=<the commit before the pick-up commit> sha=<the pick-up commit>`; set the spec `status: in-review`, its `baseline_revision` (and `baseline_commit`) to that `story_base`, so review sees the whole story, and `review_loop_iteration: 0` (build-auto would otherwise halt at its first loopback on the count that set the story aside; the kit's own rework count still continues); `SPRINT_PLAN generate --set <key>=in-progress`. Then dispatch build-auto on the spec — a spec-path dispatch resumes at review, as the re-dispatch protocol's review-step path does, with nothing left to apply — or continue at the stage the answer calls for.
3. Exit ≠ 0 → `git restore --source=HEAD --staged --worktree -- SRC` (source paths only — the bookkeeping not yet committed, an answer just applied among it, stays), re-plan the story (`status: draft`, `review_loop_iteration: 0`), and keep the patch for reference.

A story parked **before** implement (at plan or spec validation) has no code: log `story_parked` and move on; its spec waits as it is. When it parked on an undecided ledger entry (a decision dependency), its inbox item's unit is that entry (`DW-<n>`, which also gets the `note=inbox=<U-id>` trailer), so one answer decides the entry — closed `by=owner` — and releases the story, which resumes at spec validation.

## U5. The loop's stop points

Each row overrides the command passage it names.

**Waiting** — for CI, a rerun, or a queued run — is a bounded poll in one foreground call that ends inside the 10-minute cap, repeated while the run is still going: `for i in 1 2 3 4 5 6 7 8; do s="$(gh run view <id> --json status -q .status)"; [ "$s" = completed ] && break; sleep 60; done` (`gh run watch` blocks until the run ends, so it can outlive the cap). Never `Monitor`: the lead's turn never ends with something in flight. (The orchestrator keeps waiting on runner notifications, as today.)

**Stage results**
- **`## Clarification Needed`** from a stage, or a build-auto clarification HALT (Clarification protocol, Handling Clarifications): classify (U3). Decide or veto → write the ruling into the spec (`## Design Notes`, committed — Rule 16) and re-spawn through the re-dispatch protocol; owner-only → park. At most `limits.clarification_rounds` rounds per stage per story; then stop the story. Pipeline or environment defects (dirty tree, no subagents, a failed pre-warm) are fixed, never "answered".
- **Build-auto non-convergence** (`review repair loop exceeded`, `implementation verification failed`, `patch verification failed`, `matrix test audit failed`, `handoff conflicts with spec`): as the rework cap.
- **Rework cap reached** (Rework Loop): stop the story and set it aside (U4). No cap override unattended: an owner's "override" answer is logged `rework_cap_override … authorized_by=owner via=<inbox|review> items=<the answer's finite list>`. CI and smoke reworks count toward the cap, and a picked-up story's count continues where it stopped.
- **A HIGH the review asks to waive** (Rule 15): owner-only — park, and set the story aside.
- **Rule 5 tier-2 amendment:** a split into siblings planned next in the same epic, within `limits.split_children`, is decide. Moving an AC to a not-yet-planned story inside the release a `milestones` entry defines is veto (never an AC a release gate, milestone or end-to-end acceptance test depends on). Dropping or deferring scope out of the release, moving an AC when no milestone defines the release, a product number, an epic boundary or a dependency statement → park. Adding an AC to another epic's not-yet-planned story is veto (orchestrator mode: the orchestrator rules it).
- **Decision dependency at spec validation** (the command's "an intent gap for the user NOW"): rule the undecided item if its category allows; otherwise park the story.
- **Missing Integration AC at spec validation:** option (a), re-plan, once. Still missing: option (b) only if a later story of this epic is named as its consumer; otherwise stop the story. Never option (c).
- **A destructive check with nowhere disposable to run it:** stop the story (a tooling need).
- **Epic-context compilation fails:** retry once; then stop the epic.
- **AD conflict:** U7.
- **Smoke, fresh-check or AD-tooled verification failure:** the rework path (option b).
- **Orphan writer or protocol violation** (Rule 18): the orphan protocol, `protocol_violation`; a live writer that cannot be stopped stops that epic (or slot).
- **Stage API error:** rate limit or overload → the existing backoff with the retry watchdog. Usage or spend limit → a usage hold (U11) if the lead can still act — the stage the limit cut off is dead: its tree is saved as evidence and logged `stage_dead` with the stage to re-run from (U11) — otherwise the session ends, and the resume prompt from the last story boundary carries the run.
- **Model-tier checkpoint:** evaluate and log; offers are recorded, never applied (`gates.model_tier: record_only`). The mandatory mixed-tier rollback still applies.

**CI** ("CI Is a Gate")
- **Red on the story just committed:** the rework path. Unfixable within the cap → stop the story and set it aside (U4).
- **Red matching a ledgered flake:** one rerun and an `occurrence=` bump; green continues, red takes the rework path.
- **Infrastructure reds:** re-run the workflow (`gh run rerun <id>`) and watch it — the run's own duration paces the retries. Three infrastructure reds in a row, or a run still queued past `ci.queue_minutes` (counted from the push's log entry), → the CI outage rule: resolution point 1 is relaxed for at most `limits.unverified_stories` story — its successor may be committed locally, unpushed. When CI returns, push the stacked heads one at a time, oldest first, resolving each before the next push, so a red is charged to the right story. Blocked past `ci.blocked_hours` → stop the epic. (A red on a story with a stacked successor stops the epic — U4 never sets it aside.)
- **Inherited red:** rerun once. Then one fix-forward story per root cause (keyed by the first red sha, `limits.fix_forward_per_cause`), only if the failing code lies in this epic's own changes (`git diff --name-only <feature>...HEAD`); it is a veto item. Otherwise, or after that, stop the epic.
- **CI unavailable at the merge gate** (SC-4's "ask the user to confirm CI by hand"): as chosen at launch — merge on local green, or hold the merge.

**Epic start and close** — U8 and U9, including X.0's "batched to the user in one sitting", the decision sheet, SC-4's merge question and "Resume across the end-of-epic gates" ("re-asking … is correct") — none of them asks.

**Workspace and resume**
- **SC-7 (unsure where to commit):** stop the run.
- **Integrity checks, divergence, ambiguous liveness** (Resume Semantics): stop that epic.
- **Resume:** U11's unattended resume semantics.
- **A permission prompt or `AskUserQuestion`:** must not occur. Anything that would ask is an outcome in U3.

## U6. The stage Unattended block

Render this block from the policy into every stage spawn prompt, after the Spawn Prompt Skeleton's items (a runner renders it into its own stages' prompts; the block's prohibition binds stages, not the runner). The policy's `stage_preamble`, when set, goes at the very top of the prompt instead.

```text
## Unattended run
Nobody will answer a question until this run ends. Product defaults (tie-breaks, most important first): <product_defaults>.
Owner-fixed values (use them as given, never re-derive them): <owner_fixed, or "none">.
Decide small ambiguities yourself. Small means: within the spec's stated intent, reversible inside this story, touching none of the categories below, and contradicting no AD or owner-fixed value. Record each choice — plan: under `## Design Notes`, "Decided in-stage"; implement: in the spec; QA and review: in `## Decisions`.
Escalate anything else with `## Clarification Needed`, and escalate freely: when in doubt, escalate. Never park anything and never ask the owner yourself.
Always escalate (owner-only): <owner_only>. Escalate (veto window): <veto_window>.
Never read or write <state>, and never run _bmad/scripts/unattended-inbox.sh. Propagate this prohibition, with the others, to every agent you spawn.
```

## U7. AD conflicts: comply first

1. **Comply.** Reshape the story's approach to meet the AD. If an AC cannot be met within it, move that AC to a not-yet-planned story inside the release a `milestones` entry defines (veto; never one a release gate, milestone or end-to-end acceptance test depends on). With no milestone defining the release, that move is owner-only: park.
2. **Amend, with a veto window,** if compliance is impossible and the amendment is not a re-architecture: Rule 20's light path, with an `[AMENDED <date>, <story>, unattended ruling]` marker in the Rule text; `lint_spine.py` must return `"ok": true`; log `spine_updated reason=unattended_ruling`; `INBOX add veto …`.
3. **Otherwise stop the story** — or park it if the amendment touches an `owner_only` category. `--review` records the owner's choice; `bmad-architecture`'s interactive Update is recommended after the run.

**A re-architecture** (Rule 20: "a decision that contradicts an existing AD") is any amendment that (a) changes the AD's Binds or Prevents; (b) removes or reverses something its Rule requires or forbids, rather than extending the Rule to a case it does not address or tightening it; or (c) touches an AD that merged code or another open epic relies on — its id appears in a `done` story's spec, or in another open epic's specs (`grep -l "AD-<n>" <impl>/spec-*.md`; orchestrator mode: every worktree's `<impl>` too).

**Orchestrator mode:** a runner never amends an AD on its own; the question goes to the orchestrator (U14).

## U8. A later epic's start (inside the loop)

Sequentially, the next epic starts only after the previous one merged (otherwise the run ends — U9, U10). In orchestrator mode, a runner's whole epic start is this section. It runs unattended:
- **SC-2** creates the epic branch deterministically, as today.
- **`ledger_load`:** an owner it cannot repair is re-owned `owner=burndown by=load note=owner_unresolved` — a decide ruling — never a halt.
- **Retro review** is lead-automated triage, as today; it reads the headless retrospective's document.
- **Story X.0, step 5:** entries whose latest note is `inbox=<id>` (owner-only questions waiting for the owner) are skipped — they keep their owner and status until answered. Other undecided `decision-pending` entries are classified (U3): decided, vetoed, or — owner-only — parked into the inbox with the trailer `note=inbox=<U-id>`. Open action items are included as today, the headless retrospective's among them; they are also listed in the inbox (U9).
- **The readiness re-check** (headless sprint planning): PASS continues; CONCERNS continues with a veto item; FAIL, or an integrity error, stops that epic.

## U9. Epic close

- **Burn-down gate:** unchanged; it needs no human.
- **Retrospective:** `/bmad-retrospective -H <epic>` via the `Skill` tool, lead-side (`gates.retrospective: headless`; `skip` → log `epic_retro_skipped reason=policy`) — sequential: where the command runs it; orchestrator mode: by the runner, after its burn-down gate and before its integrate-forward (U14). Headless, it writes the retrospective document (`headless: true` in its frontmatter, an Assumptions section, an acceptance verdict), marks the retro key `done` (meaning "the retrospective ran"), and appends its proposed action items as `open`; it does not change earlier action items' status. Commit its outputs (bookkeeping). Read `verdict` from the document's frontmatter: `rejected` holds the merge (`INBOX add held-merge …`); `accepted` and `accepted-with-open-items` continue. Add `INBOX add retro "Epic <N>" "<verdict; n findings; new action items>" …` so the owner can drop items through `--review`. Log `epic_retro_complete mode=headless verdict=<…>`.
- **Decision sheet** (Rule 17 (5)): classify each `escalated` / `decision-pending` entry (U3). Owner-only → `INBOX add parked "DW-<n>" …` and the trailer `status=decision-pending owner=burndown by=unattended note=inbox=<U-id>`; the decision sheet, the burn-down gate and X.0 skip entries whose latest note is `inbox=…`. Veto → decided with `by=unattended` plus a veto item. The rest → decided `by=unattended`. Human-only entries (`note=human=`) move to `action_items` as today. Log `decision_sheet_recorded … by=unattended`.
- **Merge to the feature branch** (`gates.merge_to_feature: when_green`; SC-4 asks nothing). All must hold: every story `done`, none parked or set aside (a dropped story — `story_dropped`, U10 — is closed); the burn-down gate logged; a verdict that is not `rejected` (or the retrospective skipped by policy); the decision sheet recorded; fewer than `limits.veto_open_max` open veto items. Then:
  1. **Integrate forward first** if `origin/<feature>` is not an ancestor of the epic head: merge it into the epic branch; resolve conflicts per `gates.merge_conflicts` (`resolve`: plain unions — both sides kept, no behaviour choice — are decided; behaviour choices are ruled as veto items; code an AD governs goes through U7; an `owner_only` match parks; an unclassifiable conflict holds the merge. `ask`: any conflict holds the merge). Run the full suite and the smoke on the merged tree, push, and require CI green on that exact head.
  2. **Merge** (SC-4, now conflict-free): the `epic_merge_started` marker, the merge, the push, the branch deletes, verification on origin, the record commit — which also carries a copy of `inbox.md` as `<impl>/unattended-inbox-snapshot.md`, so the owner's decisions live in git.
  3. **Resolve CI for the new feature head.** Flake or infrastructure → retried. A story red → no new epic starts: a sequential run ends with its report.

  Any condition failing holds the epic unmerged (`INBOX add held-merge …`). A sequential run then ends with its report (the next epic needs this one merged); in orchestrator mode, dependants wait and independent epics continue. A held merge is released by the owner's answer (through the inbox or `--review`): the run merges at its next story boundary, or first thing when it is next launched.
- **Final epic of the range:** the ledger sweep is classified like the decision sheet; the residue goes into the run report.

## U10. Park, stop and continue

- A parked or stopped story is logged and gets an inbox item; if it has code, it is set aside (U4). It gets no ledger entry: the log, the inbox and the tracker carry it.
- **Take the next independent story** in epic order. A story is *not* independent if `epics.md` dependency statements, its `Consumes:` list, or its acceptance criteria name a parked, stopped or waiting story; if it is a split sibling of one (split children depend on parked siblings by default); or if its own plan stage, told which stories are parked, escalates because it needs one.
- **If no story in the epic is independent:** with `gates.gap_fill: burndown`, take a story named for this epic in `gates.gap_fill_stories`, or charter a gap-fill cleanup story from closable ledger entries owned by this epic (Rule 17's chartering rules; titled "Epic N gap-fill", numbered like any inserted story, never reused as the burn-down story), at most `limits.gap_fill_per_epic`. Failing that: a sequential run ends as `stopped: remaining work blocked` with its report — it never moves to another epic while this one is blocked, and never sits waiting. A runner ends with `## Epic Runner Ended reason=parked` (U14).
- **Picking up answers** — the main session (orchestrator mode: U14), at every story boundary, and at every launch or resume before it computes the resume point — unless a story is mid-pipeline (an open `stage_spawned`; a story whose latest anchor is past `story_created` but is not `committed`, `story_set_aside`, `story_parked`, `story_dropped` or a stop of it; a hold patch under `<patches>`; or a dirty source tree): then that story is finished first and the answers are applied at the next story boundary. `INBOX answers`; for each (`U-n`, `answer_ref`, by, via, answer, quote): if `owner_decision id=<U-id> at=<answer_ref>` is already logged, the session that logged it may have died before applying it — if its effect is in place (a `story_unparked` — or its pick-up commit, which a crash may have left unlogged: log `story_unparked` then — the merge record, the ledger line), only resolve; if not, apply it now without logging it again; never apply twice. Otherwise log `owner_decision id=<U-id> at=<answer_ref> via=<inbox|review> by=<who>` with the answer (and the quote, for a relay) FIRST, then apply it — un-park through U4's pick-up, re-plan (spec `status: draft`, `review_loop_iteration: 0`), reverse a veto ruling (as a routed ledger entry or a fix story), merge a held epic (U9), close a ledger question with `by=owner`, approve a graph change, or change a retrospective's action items (`uv run --no-cache .claude/skills/bmad-retrospective/scripts/sprint_status.py update --file <impl>/sprint-status.yaml --epic <N> --set-action-status '[{"id":"<id>","status":"<open|in-progress|done>"}]'` for the transitions the owner confirmed; a dropped item is set `done`, the drop noted in the log — the script knows no other status) — then `INBOX resolve <U-id> lead <answer_ref> "<sha or note>"` (a refusal saying "already resolved" means `--review` resolved it first: fine). The script refuses a stale `answer_ref`, so an answer that arrived meanwhile is applied too, never dropped. An answer that would act outward (U1's never-list) is recorded, not executed. **Order:** answers that need no clean tree first (ledger questions, vetoes, action items, re-plans), then held merges, then at most one pick-up — the earliest story in epic order. A pick-up leaves its story mid-pipeline, so further un-park answers stay unlogged until the next story boundary. "Drop this story" (scope removal — owner-only) means: remove its block from `epics.md`, `SPRINT_PLAN generate`, record its delivered work and its set-aside patch in a ledger entry, log `story_dropped story=<key> id=<U-id>`, and resolve. `story_dropped` is a terminal anchor: the resume point, U9's merge condition and the attended check below count the story as closed.
- **An attended run after an unattended one:** if the log holds `story_set_aside` or `story_parked` entries without a later `story_unparked` or `story_dropped`, or `<patches>` holds patches, the attended lead reads U4, U10 and U11 before resuming. It takes the run lock for its duration (`INBOX run-start attended-<UTC> attended`, released with `run-end` at its end), so a relaunch loop cannot join; it applies hold patches at their story's resume point (U11) and picks parked stories up with the owner, never re-implementing a set-aside story over its patch; it records each owner's answer through the script (`--by owner --via attended`), logs `owner_decision`, and resolves the item. A dead run's lock is released by the owner: `INBOX run-takeover <its id> owner "owner-confirmed: <what they said>"`, then `INBOX run-end <its id> <the printed token> "released"`.

## U11. Hold and resume

**Triggers** (checked at every stage boundary, and at launch): `<state>/HOLD` exists (`INBOX hold "<reason>"`); the first usage warning the session can see; a usage- or spend-limit error the lead can still act on; the policy's `hold.at` (a clock time — never honoured when it is earlier than the run lock's `started`: the run's start or the owner's latest resume, which a relaunch keeps) or `hold.after_hours` (counted from the same `started`). Quota itself is mostly invisible to the agent, hence the owner-set times. A hold the run orders itself is written `INBOX hold "auto: usage <what it saw>"` or `INBOX hold "auto: time <hold.at|hold.after_hours>"`; one the owner orders has any other reason. The script never lets a weaker hold replace a stronger one (the owner's, then a time hold, then a usage hold).

**Steps** — a hold never interrupts a stage, however long it runs:
1. Log `hold_ordered reason=<…>`; start no new stage.
2. When the running stage returns, do its normal bookkeeping. A stage a usage or spend limit cut off did not finish: its tree is a dead stage's, saved as the resume rule below says (`stage_dead`), never as a hold patch. Otherwise the story's work that is still uncommitted by design (QA's tests, review patches — every stage's since the story's last commit) is saved outside git and taken out of the tree, and the save is logged `hold_saved story=<key> stage=<stage> patch=<path|none> tree=<clean|in_place|partial|submodule>` — the resume acts on it. If the work touches a submodule (`git status --porcelain --ignore-submodules=none` shows one dirty), leave it in place (`tree=submodule`) and record it in the hold report; the resume stops that epic. Otherwise: `mkdir -p <patches>`; `git add -N -- SRC` (every untracked file under `SRC` — this patch is never pushed, and the resume brings them back untracked); `DIFF HEAD -- SRC > <patches>/<story>-<stage>.patch` (it must exit 0; an empty patch is deleted — nothing to hold); `APPLY -R --check` it — if that fails, undo the intent-to-add (`git restore --staged -- SRC`) and leave the work in place (`tree=in_place`); `git restore --staged --worktree -- SRC`; then `git status --porcelain --ignore-submodules=none -- SRC` must print nothing (`tree=clean`) — if it prints anything, put the work back (`APPLY <patch>`: `tree=in_place`; if that fails too, `tree=partial`). Whatever fails, the patch is kept, and never deleted while it holds the only copy of the work. **No WIP commits.**
3. Log `hold_complete shas=<…>` (write-ahead, so the log never trips the next clean-tree gate), then commit and push bookkeeping under the normal push rules (a copy of `inbox.md` as `<impl>/unattended-inbox-snapshot.md` included — the patches stay outside git); leave in-flight CI running.
4. Refresh the resume prompt; record which throwaway runtimes are up and which run started them.
5. Verify every reported sha on origin; list local-only shas as such.
6. `INBOX add hold …`; push notification; `INBOX run-end <run-id> <token> hold`; end. (Orchestrator mode: after every runner has ended — U14.)

**The resume prompt is kept current.** At every story boundary the lead rewrites `<state>/resume/epic-<N>.md` from the log, so a session that dies without warning (a hard usage limit, a crash) leaves a current one. It contains: the role and invocation (`/epic-cycle <range> --unattended`, the run id, both hashes); the owner-fixed values; the git-verified state (branch, origin and local heads, each unpushed commit and what it holds, spec status, the ledger load, parked and set-aside stories); every runtime target in use (dev and throwaway), what is loaded there, and which run started it; the project's `stage_preamble`; "First, check the CI result for the last pushed `headSha`"; the next steps in order; and how to end. It never depends on chat history. The main session also records the run's heartbeat (see the resume semantics below).

**Unattended resume semantics** (override Resume Semantics under the flag):
- Re-invoking the same command resumes. On a **relaunch, nothing asks**: re-run the pre-flight and U2's automatic checks; take every owner answer this run already logged from the log (`permission_mode_attested`, `ci_mode_chosen`, a lead-model acceptance); the telemetry gate logs and continues; a check that would need a new answer stops the run with its report. An interactive re-invocation may ask U2's launch questions again.
- Take the run lock over per U2; log `unattended_resumed run=<id> policy=<hash> doctrine=<hash> lock_token=<t>` — this is how an owner's committed policy change is accepted. **A `<state>/HOLD`:** an interactive re-invocation clears it (`INBOX hold-clear "resumed by the owner"`). A relaunch clears only a usage hold (`INBOX hold-clear "relaunch" --usage-only`, which leaves any other in place); a time hold or the owner's it honours — it runs the hold steps for whatever the dead session left (a dead stage's tree saved as the resume rule below says), logs `hold_complete`, and ends.
- A hold patch (`<patches>/<story>-<stage>.patch`) belongs to the story that was mid-pipeline: that story is the resume point, and its latest `hold_saved` says where the work is. `tree=clean`: after the clean-tree gate (`git status --porcelain --ignore-submodules=none -- SRC` prints nothing — the story's ignore and attribute changes stay in the tree across a hold), `APPLY <patch>` — no `--index`: the tree returns exactly to its pre-hold state, new files untracked and nothing staged, so nothing a stage left lying around reaches the story's commit. Exit ≠ 0 → nothing was applied (a plain apply is atomic): keep the patch — it is the only copy — and stop that epic with an inbox item naming it. A tree already dirty where `APPLY -R --check <patch>` exits 0 had the patch applied before a crash: treat it as `tree=in_place`. `tree=in_place`: the dirty tree is the held work, not a dead stage's — confirm it with `APPLY -R --check <patch>` (if that fails, stop that epic). Then delete the patch, log `hold_restored story=<key>`, and continue that story at its next stage. `tree=partial` or `tree=submodule`: stop that epic. (Orchestrator mode: each runner applies its own worktree's hold patches — U14.)
- Answers per U10. With no story mid-pipeline, the resume point is the earliest story whose anchor is not `committed` or `story_dropped`, **skipping parked, stopped and set-aside stories that are still unanswered, and stories waiting on an epic that has not merged.**
- An open clarification is classified and ruled or parked, not surfaced.
- **A dead stage's tree** — a dirty source tree that no `hold_saved tree=in_place` accounts for (a fresh lead with no orchestrator above it, or a runner re-spawned after its predecessor died), or a stage a usage limit cut off: save it with step 2's tools as `<evidence>/<story>-dead-<stage>-<UTC>.patch` — evidence only, never applied — and log `stage_dead story=<key> stage=<stage> evidence=<path> rerun_from=<stage>` before the restore. `rerun_from` is the earliest stage whose output the save took out: QA's tests and review patches stay uncommitted until the story's commit, so a code review that died after QA re-runs QA first; a pick-up that died between its apply and its commit re-runs the pick-up. Until a stage completes after it, the latest `stage_dead` decides where the story resumes, whatever open `stage_spawned` follows it. Before re-running `bmad-build-auto`, set the spec `status: in-progress` if nothing since its `baseline_revision` is committed (`git diff --quiet <baseline_revision> HEAD -- SRC` exits 0): `in-review` would send it to review over code the save took out. (A picked-up story's code is in its pick-up commit, so its review re-runs as it was.) If the save fails (step 2's checks), keep the evidence and stop that epic.
- Divergence, branch drift, a diverged head, ambiguous liveness → stop that epic.
- Idempotent steps (sprint planning, `spine_resolved`) re-run; the attended setup gates this run already logged are skipped.
- The main session records its heartbeat (`INBOX run-boundary <run-id> <token> "<where>"`) at every stage boundary — the orchestrator at every notification and every dispatch — so the lock's age says whether the run is alive. A heartbeat refused with exit 4 means another session took the run over: stop at once.
- **The log beats the resume prompt** wherever they disagree.
- A resume is always a fresh session or a fresh spawn, above all after an account switch. Never message a runner whose call has returned, except through the parallel kit's sanctioned resume.

## U12. The inbox, notifications, and the run report

- **The inbox** is written only through `INBOX` (an append-only journal plus a rendered `inbox.md`; locked writes). Kinds: `parked`, `stopped`, `veto`, `held-merge`, `hold`, `retro`, `report`. An item's unit names what it blocks (`Story 4.2`, `Epic 4`, `DW-41`); its options and recommendation are what the owner answers. Routine `decide` rulings never go in.
- **Integrity:** the main session runs `INBOX verify` before every stage spawn and after every return (runners: before each of theirs). A mismatch means a hand edit (or a crash mid-write): `INBOX render` saves the edited file aside (`inbox.md.edited-<UTC>`, listed in the inbox) and rebuilds `inbox.md` from the journal. A hand edit is never applied; tell the owner once (an inbox `report` item and a push: "your edit was saved but not applied — answer with the script or `--review`"). Stages are forbidden to touch `<state>` (U6). `verify` covers `inbox.md` only: a determined agent with a shell could run the script itself — which is why outward actions never execute unattended (U1), and why `--review` shows every answer with who gave it, when, and through what.
- **Answers** come from the owner (`INBOX answer U-<n> "<answer>" --by owner`, or `--review`), or through a relay: a companion session, or anyone acting for the owner, runs `INBOX answer U-<n> "<answer>" --by <who> --quote "<the owner's words, verbatim>"`. A companion learns of new items by watching `inbox.md`; nothing messages it (Rule 18).
- **Notifications** are sent only by the main session, with `notify.push: true`: on park, stop, hold and run end — never for routine rulings. Each carries the item's one-line question and recommendation, not just "see inbox". In orchestrator mode, runners report parks and stops as they happen (U14), so the orchestrator can notify at once.
- **Run report** — `INBOX add report …` and a log entry `unattended_report` at completion, hold or stop: stories merged, with times; rulings, with shas; open veto items; parked, stopped and set-aside stories; held merges; CI health; pace (stories per day, wall time per story); the remaining queue; risk against `milestones`; anything the owner should know before the next release. Log `unattended_report` first, then commit the inbox snapshot (`<impl>/unattended-inbox-snapshot.md`) — in the final merge's record commit when the run ends with a merge (U9), otherwise in the epic branch's last bookkeeping commit — then `INBOX run-end <run-id> <token> <reason>`.

## U13. `/epic-cycle --review` (attended)

The owner runs it when checking in, from any session in the project. **It only records:** it reads, explains and recommends, and writes nothing but inbox answers through the script — no commit, no merge, no checkout, no cycle-log line — so it is safe beside a live run. The run applies every answer (U10).

1. **Find the run state:** `_bmad-output/unattended/` and, when `_bmad/custom/parallel.yaml` exists, `<worktree_base>/.coordination/unattended/` (its `worktree_base`). Review each that has a journal. `INBOX run-status` shows whether a run holds the lock, and its last heartbeat.
2. Read the inbox and the logs; re-verify the reported branch heads on origin with `git ls-remote` (no fetch: it would contend with a live run's pushes); other shas are verified by the run, which fetches.
3. Walk through, with a recommendation each, recording each answer with `INBOX answer U-<n> "<answer>" --by owner --via review`:
   1. **Parked questions.**
   2. **Stopped units:** what happened; options — re-plan, an override with a finite item list, drop the story (owner-only scope removal).
   3. **Veto-window rulings:** keep, or reverse.
   4. **Held merges:** merge now, or keep holding.
   5. **Headless retrospectives:** the verdict, the findings, the new action items (keep or drop), and the proposed status changes to earlier items (each confirmed or not).
   6. **Amended ADs:** keep; a reversal is a fix story; `bmad-architecture`'s interactive Update is recommended once no run is live.
   7. **Dependency-graph changes awaiting re-approval** (orchestrator mode).
   8. **Relayed answers:** confirm each (re-record it `--by owner`), or replace it.
   9. **Hand edits saved aside:** what they said, and the answer to record instead.
4. **A summary,** and how to start or resume the run so the answers are applied. Items that need no pipeline work (a veto kept, a report read) are resolved here: `INBOX resolve <U-id> review <answer_ref|-> "<note>"` — "already resolved" means the run got there first, which is fine. Everything else stays answered for the run.

## U14. Orchestrator mode (parallel kit)

**The orchestrator** is the main session (U2–U13 apply to it). It passes `--unattended`, the run id, both hashes, the absolute `<state>` path and `hold_at=<UTC|none>` in every runner spawn and resume prompt — `hold_at` is the earlier of the policy's `hold.at` (unless that is earlier than the run lock's `started`) and `started` plus `hold.after_hours`.

**A runner** is the lead for its epic (U3–U10 apply), with these exceptions:
- It skips U2 and U13 entirely and never touches the run lock: never `INBOX init`, `run-*`, `answers`, `resolve` or `answer`. It may `INBOX add` and `INBOX hold` (a time or usage trigger only), and runs `INBOX verify` around its stages.
- It reads the policy and this doctrine from the main checkout (absolute paths), checks their hashes against the ones in its prompt (a mismatch stops its epic), and logs `runner_unattended_start run=<id> policy=<hash> doctrine=<hash>`.
- Its epic start is loop work (U8).
- **It reports each park or stop as it happens:** after `INBOX add`, it ends its turn with `## Clarification Needed` beginning `PARKED U-<n>:` or `STOPPED U-<n>:` and a one-line summary. The orchestrator notifies the owner and resumes it with "continue with independent work".
- **Cross-epic matters** go to the orchestrator as `## Clarification Needed`: contended paths, other epics' stories, AD amendments. The orchestrator classifies and rules — or parks — and resumes the runner through the sanctioned orchestrator→runner message.
- **A story that must wait for another epic** (a contended path that epic owns): the runner sets it aside (U4) with reason `waiting_on epic <M>`, logs `story_waiting story=<key> on=<M>`, and continues with independent work. It is skipped until epic M has merged.
- **Hold:** at every stage boundary it checks `<state>/HOLD`, the clock against the `hold_at` in its prompt, and any usage warning or limit it has seen. A time or usage trigger it writes itself (`INBOX hold "auto: time hold_at"` / `"auto: usage <what it saw>"`), then it holds (U11) — so a time hold never waits for a notification, and the orchestrator stops dispatching into a spent quota.
- **It runs the headless retrospective** after its burn-down gate and before its integrate-forward (contract item 9), commits its outputs, and reports `retro: verdict=<…> doc=<path>` in the contract.
- **When nothing independent remains** (gap-fill included), or at a hold, it finishes its bookkeeping (a hold: U11 step 2 in its worktree), refreshes its epic's resume prompt (`<state>/resume/epic-<N>.md`), and ends with `## Epic Runner Ended` followed by `epic: {N}`, `reason: parked|stopped|waiting|hold`, `waiting_on: <M>` (for `waiting`), `head: <sha>`, `unpushed_work: none` (or the list), `inbox: <U-ids>`, `applied: <U-n at=<answer_ref>, …>|none`. An epic that finishes always ends with the Complete contract (carrying `retro:` and `applied:`), never `ready_for_merge: false`: a runner that cannot complete ends with `reason: stopped`.
- **On re-dispatch** it is given its epic's resume prompt and any answers to apply (`apply U-n at=<answer_ref>: <answer>`); it applies its own worktree's hold patches at its resume point (U11), applies the answers there (U10's steps, U4's pick-up), and reports each as `applied:`. Told "integrate forward, then pick up <story>", it does exactly that.

**The orchestrator:**
- **Answers:** only the orchestrator reads `INBOX answers` — on every notification, and before it ends. It routes each answer to the epic that owns the unit: if that epic's runner has ended, it re-dispatches the runner with the answer in its prompt (`apply U-n at=<answer_ref>: <answer>`); if the runner is live, it passes the answer in its next resume message. Never two runners on one epic. Answers that belong to no running or ended epic — a held merge, a graph re-approval, a merged epic's retrospective items, a decision-sheet question the orchestrator parked, a veto reversal for a merged epic — the orchestrator applies itself in the main checkout (the merge queue; a ledger trailer; `sprint_status.py`; a routed fix story), following U10's write-ahead. Every `applied: U-n at=<answer_ref>` line a runner returns — in a Complete contract, an Ended contract or a clarification — is resolved at once (`INBOX resolve U-n orchestrator <answer_ref> "<epic N>"`).
- **Notification handling:** setup stops before dispatch are surfaced to the owner, as today. Every runner-reported item after dispatch is loop work: classified (U3); `decisions_for_user:` items as at the decision sheet (U9); an unmeetable `TOOLING:` need stops that slot's story; ambiguous runner liveness stops that slot. Each notification also refreshes the heartbeat (`INBOX run-boundary`). `## Epic Runner Ended` is recorded in `dispatch.yaml` by its reason and **never re-spawned blindly**: `parked` / `stopped` → `waiting` until an answer for that epic arrives; `waiting` → `waiting` with `waiting_on: <M>`, re-dispatched ("integrate forward, then pick up <story>") once epic M has merged; `hold` → `held`, re-dispatched when the run resumes.
- **AD amendments:** allowed (U7 step 2) only when no other in-flight epic's specs cite the AD; the runner then writes it in its worktree with `orchestrator` in the marker. If another in-flight epic cites it, the story stops — a re-architecture across epics is the owner's. Two runners therefore never amend one AD.
- **Contended paths:** a story that must edit a path another in-flight epic owns waits, set aside; the orchestrator records it in `dispatch.yaml` (`waiting_on: epic <M>`) and re-dispatches the waiting epic after epic M merges. Two epics waiting on each other (a cycle in `waiting_on`) can never both proceed: stop both stories, with an inbox item.
- **Merge queue (SC-4-P):** `when_green` replaces `queue-ask`. A `retro: verdict=rejected` (or any failing U9 condition) holds the merge: the orchestrator owns it — `dispatch.yaml` state `held_merge`, an inbox `held-merge` item — and a "merge now" answer releases it (if the feature branch has moved, one more integrate-forward first: resuming the runner if it is live, otherwise re-dispatching it once with "integrate forward, then complete"). If the feature branch moved after the runner integrated forward, resume the runner for one more integrate-forward rather than resolving anything in the orchestrator. Dispatch continues along the approved graph; a split, gap-fill or fix-forward story inside one epic re-records the graph hash (`reason=split|gap_fill|fix_forward`); any other structural change pauses only the affected epics, with an inbox item, until the owner re-approves the graph. A red feature head → no new dispatch; running runners finish; inbox item. The merge record carries the inbox snapshot (U9).
- **Time and usage holds:** the orchestrator evaluates the policy's `hold.at` / `hold.after_hours` (against the run lock's `started`) and any usage warning at every notification and dispatch, and writes `INBOX hold "auto: time …"` / `"auto: usage …"` itself; runners check the same triggers at their stage boundaries (above). A runner's `reason: hold` with `<state>/HOLD` present starts the orchestrator's hold.
- **Idle slots:** dispatch the next eligible epic into a freed slot — never while `<state>/HOLD` exists. When no runner is live and nothing is dispatchable or answered, end the run with its report — never idle.
- **Hold:** write `<state>/HOLD` (`INBOX hold`); dispatch nothing more. Each runner ends at its next stage boundary with `## Epic Runner Ended reason: hold` (state `held`). Wait for every runner, however long its current stage takes — never stop a live runner, never write in its worktree. Then verify, log `hold_complete`, notify, `INBOX run-end`, and end. **On resume:** clear the hold as U11 says (an interactive re-invocation always; a relaunch only a usage hold — it honours the owner's and a time hold), then re-dispatch from `dispatch.yaml`, within `max_parallel_epics`: every `held` epic first, with its epic's resume prompt; a `waiting` epic only once its answer has arrived or its `waiting_on` epic has merged; a `held_merge` epic never — it is the merge queue's (a released one whose runner is gone is re-dispatched once, as above).
- **Stale runtime locks after a crash:** a resumed orchestrator in a fresh session sweeps every lock whose `acquired_at` predates its own start (those holders died with the old session) and logs `runtime_lock_swept reason=prior_session`. Within a session, the kit's rule stands: sweep only a lock whose owner is confirmed dead.
````

### File 8: `_bmad/custom/unattended.yaml` (template — write ONLY if the file is absent)

The project's unattended policy. **Never overwrite an existing file** — it is project-owned; Step 1 reported any keys it lacks. Write this template when absent:

```yaml
# /epic-cycle --unattended policy. Project-owned: the kit writes this template only when the file is absent and
# never overwrites it. The categories are data the lead classifies against (doctrine: epic-cycle-unattended.md U3):
# hard_stops first, then owner_only (park), then veto_window (decide and report); anything else is decided.
# Changing this file mid-run stops the run at the next story boundary: change it during a hold, commit it, then resume
# (the resume records the new version). Orchestrator mode reads it from the main checkout.
version: 2
product_defaults:      # tie-break principles for calls made alone, most important first (edit for this project)
  - The architecture spine is the contract; where it is silent, choose the simplest option consistent with it.
  - When a choice trades safety for convenience, keep the safe behaviour and make it convenient.
  - Measure, don't guess; anything not yet measured is labelled (inference).
  - Smaller is better; split rather than stretch a story.
  - Text claims only what the product does.
owner_fixed: []        # values the owner chose; quoted verbatim in every stage, runner and resume prompt; never re-derived
stage_preamble: ""     # a project line that must head every stage prompt (e.g. "one test-runner call per message")
milestones: []         # [{date: 2026-11-30, scope: [1, 2, 3, 4, 5, 6], name: "release 1"}] - defines "the release";
                       # the run report measures pace and risk against these. With none, every AC move is owner_only.
licences_allowed: [MIT, BSD-2-Clause, BSD-3-Clause, Apache-2.0, ISC]
hard_stops:            # checked FIRST. Stop the affected unit (story, epic, slot or run); record; continue elsewhere
  - a security incident (a secret in a commit, log, trace or ANY printed output; pushing to any remote other than origin; changing repository or package visibility) - never rewrite history
  - a red CI head that is not a known flake and cannot be fixed inside its story, nor fixed forward (limits.fix_forward_per_cause)
  - a conflict that cannot be classified
  - agent liveness that cannot be established; a live orphan writer that cannot be stopped
  - a tooling need that cannot be met
  - a workspace integrity error; a policy or doctrine file whose hash changed without a logged resume
owner_only:            # park: record in the inbox, continue with independent work. Park ONLY on an explicit match here.
  - releases, tags, release pull requests, anything landing on the trunk
  - anything outward-facing (publishing, posting, filing external reports, DNS or hosting, repository or service settings, making anything public)
  - dropping scope from a release, or deferring it out of the release; moving an AC when no milestone defines the release
  - changing a number stated as a product target in the PRD or epics (implementation details such as timeouts, batch sizes and retry counts are decide)
  - changing epic boundaries, story-to-epic assignment, or dependency statements
  - waiving a HIGH finding
  - money, accounts, keys, quota; a dependency under a licence not in licences_allowed
  - destroying or migrating data on a shared (non-throwaway) runtime
  - agent instruction files (CLAUDE.md, AGENTS.md, anything under .claude/) and anything under _bmad/ (the kit's own run-state writes excepted)
  - stopping, recreating or tearing down a runtime resource this run did not start (using one is fine)
  - rewriting pushed history (force-push); deleting branches or tags other than the kit's own merged epic branches
veto_window:           # decide now and report; the owner may reverse before the next release
  - security posture; how secrets and credentials are stored or handled
  - privileges, roles and access grants
  - a change to how stored data is kept (schema, format, retention)
  - weakening a guarantee stated in the PRD, the spine or a spec's intent
  - public-facing text NOT already specified in the UX or PRD; claims about what the product does; licence and attribution notices; project or owner identity
  - a new third-party dependency under a licence in licences_allowed, pinned to an exact version
  - moving an AC to a not-yet-planned story in the same release (never one a release gate, milestone or end-to-end acceptance test depends on)
  - adding an AC to another epic's not-yet-planned story (sequential; parallel runs route this to the orchestrator)
  - amending an architecture decision (AD), within the bounds of the doctrine's U7
gates:
  retrospective: headless      # headless | skip (skip for quota-tight runs)
  model_tier: record_only      # offers are recorded, never applied (tier changes are spend); the mandatory rollback still applies
  merge_to_feature: when_green # when_green | hold
  merge_conflicts: resolve     # resolve | ask. Overrides parallel.yaml integrate_conflicts for the run
  gap_fill: burndown           # burndown | none
  gap_fill_stories: []         # standing cleanup stories, per epic, that plan the gap-fill supply (e.g. ["4-9-epic-4-cleanup"])
limits:
  clarification_rounds: 3      # per stage per story; then the story stops
  split_children: 6            # stories one original story may become; past that the story stops
  fix_forward_per_cause: 1     # per first-red sha; then the epic stops
  gap_fill_per_epic: 2         # gap-fill stories one epic may charter while its other stories are blocked
  veto_open_max: 20            # open veto items; at the limit, merges to the feature branch are held until --review
  unverified_stories: 1        # stories that may be committed past an unresolved CI result during a CI outage
ci: {queue_minutes: 60, blocked_hours: 6, without_ci: ""}   # without_ci: local_green | hold - asked at launch when empty and CI is none/unavailable
hold: {at: "", after_hours: 0} # your time hold: a clock time (ISO 8601, UTC) or hours since the run started or you last resumed it; a relaunch honours it
notify: {push: true}
```

### File 9: `_bmad/scripts/unattended-inbox.sh` (verbatim)

Cross-platform bash, like `ledger.sh` (invoked as `bash _bmad/scripts/unattended-inbox.sh <state> …`; no executable bit needed). The inbox is the owner's channel in an unattended run: items are added by the lead, answered by the owner (or a relay quoting the owner) or through `--review`, and resolved exactly once — against the latest answer's timestamp, so an answer that arrives meanwhile is never dropped. Only this script writes the journal and the rendered `inbox.md`; a hand edit of `inbox.md` is saved aside and never applied. It also keeps the run lock (holder, host, heartbeat) and hold requests.

```bash
#!/usr/bin/env bash
# The owner's inbox for an unattended /epic-cycle run (doctrine: _bmad/custom/epic-cycle-unattended.md, U12-U13).
#
# Cross-platform: Git Bash on Windows, bash 3.2+ on macOS/Linux; needs awk, sed, date, find, cmp, uname.
# Everything lives in the run state directory <state>, outside git's view (sequential: _bmad-output/unattended/,
# gitignored; orchestrator mode: <worktree_base>/.coordination/unattended/):
#   inbox.journal  append-only event log, one TAB-separated line per event -- the source of truth
#   inbox.md       rendered from the journal on every write -- the file people read; never edited by hand
#   run.lock       present while a run is live: run id, holder, host, heartbeat
#   HOLD           present while a hold is requested
# Only this script writes the journal and inbox.md. A hand edit of inbox.md is never applied: the next write saves
# it as inbox.md.edited-<UTC> (listed in the inbox) and re-renders. `verify` covers inbox.md only -- a determined
# agent with a shell can still run this script, which is why the doctrine never executes outward actions unattended.
set -euo pipefail

usage() {
  cat >&2 <<'EOF'
usage: bash unattended-inbox.sh <state-dir> <command> [args]
  init <run-id>                  create the state directory, journal and inbox (idempotent; a new run id is recorded)
  add <kind> "<unit>" "<question>" "<options>" "<recommendation>" "<blocks>"
                                 add an item; prints U-<n>. kind: parked|stopped|veto|held-merge|hold|retro|report.
                                 A parked, stopped, held-merge or retro item is one per unit: an open one is reused
                                 ("U-<n> EXISTS"). veto, hold and report items are always added.
  answer U-<n> "<answer>" --by <who> [--quote "<the owner's words>"] [--via <channel>]
                                 record an answer; anyone but `owner` must pass --quote (a relay quotes the owner verbatim)
  answers                        answered, unresolved items: U-n TAB answer_ref TAB by TAB via TAB answer TAB quote
                                 (answer_ref = <UTC>#<journal line>: it names one answer exactly)
  resolve U-<n> <applied_by> <answer_ref|-> ["<note>"]
                                 mark an item resolved -- once. <answer_ref> must be the latest answer's (from `answers`),
                                 so an answer that arrived meanwhile is never silently dropped; `-` resolves an item nobody
                                 answered (a report, a hold)
  list [open|all]                U-n TAB kind TAB unit TAB state TAB question (default: open = not resolved)
  hold "<reason>"                request a hold (writes HOLD). Holds rank: the owner's (any reason not starting
                                 "auto:"), then "auto: time ...", then "auto: usage ..."; a weaker one never replaces a
                                 stronger one ("HOLD KEPT: <reason>")
  hold-clear "<reason>" [--usage-only]
                                 clear it (a resume does this after a hold); --usage-only (a relaunch) clears only an
                                 "auto: usage" hold and otherwise exits 5 with "HOLD KEPT: <reason>"
  run-status                     print the run lock with its heartbeat age; exit 3 if one exists, 0 ("none") if not
  run-start <run-id> <holder>    take the run lock and print `token=<t>`; exit 3 (printing it) if one exists
  run-takeover <run-id> <holder> "<reason>"
                                 replace a lock that names <run-id> (a resume) -- or, with reason "owner-confirmed ...",
                                 any lock; prints a new `token=<t>`, so the replaced session's next heartbeat fails.
                                 A takeover of the same run keeps its `started` (hold times count from it); an
                                 owner-confirmed one starts the clock again
  run-boundary <run-id> <token> "<where>"   heartbeat; refused (exit 4) unless the lock holds this run id AND token
  run-end <run-id> <token> "<reason>"       release the run lock; refused (exit 4) on any other run id or token
  verify                         exit 0 if inbox.md is exactly the journal's rendering, else 1
  render                         save any hand edit, then rebuild inbox.md from the journal
Values are single-line, contain no TAB, and the unit and question are non-empty; run ids and holders have no spaces.
EOF
  exit 1
}

STATE="${1:-}"; CMD="${2:-}"
[ -n "$STATE" ] && [ -n "$CMD" ] || usage
J="$STATE/inbox.journal"; MD="$STATE/inbox.md"; LOCK="$STATE/inbox.lock"; RL="$STATE/run.lock"
now() { date -u +%Y-%m-%dT%H:%M:%SZ; }
single() { case "$1" in *$'\n'*|*$'\t'*|*$'\r'*) echo "ERROR: values must be single-line with no TAB" >&2; exit 1 ;; esac; }
need() { [ -n "$2" ] || { echo "ERROR: $1 is required" >&2; exit 1; }; }
rank() { case "$1" in "auto: usage"*) echo 0 ;; "auto:"*) echo 1 ;; *) echo 2 ;; esac; }   # hold strength
need_journal() { [ -f "$J" ] || { echo "ERROR: $J not found; run init first" >&2; exit 1; }; }

# Writers take an exclusive lock (writes take milliseconds). A lock is stale when its content has not changed for a
# whole retry interval AND its writer is gone (same host, dead pid) -- or when it is older than 10 minutes. The content
# is re-read just before removal, so a waiter never removes a lock another waiter has just taken.
lock() {
  tries=0; prev=""
  until ( set -o noclobber; printf 'pid=%s host=%s at=%s\n' "$$" "$(uname -n)" "$(now)" > "$LOCK" ) 2>/dev/null; do
    held="$(cat "$LOCK" 2>/dev/null || true)"
    if [ "$tries" -gt 0 ] && [ "$held" = "$prev" ]; then
      hpid="$(printf '%s' "$held" | sed -n 's/.*pid=\([0-9]*\).*/\1/p')"; hhost="$(printf '%s' "$held" | sed -n 's/.*host=\([^ ]*\).*/\1/p')"
      if [ -n "$(find "$LOCK" -mmin +10 2>/dev/null)" ] || { [ -n "$hpid" ] && [ "$hhost" = "$(uname -n)" ] && pid_gone "$hpid"; }; then
        if [ "$(cat "$LOCK" 2>/dev/null || true)" = "$held" ]; then echo "WARNING: breaking stale $LOCK ($held)" >&2; rm -f "$LOCK"; prev=""; continue; fi
      fi
    fi
    prev="$held"
    tries=$((tries + 1)); [ "$tries" -lt 120 ] || { echo "ERROR: $LOCK held for 120 s by a live writer ($held)" >&2; exit 1; }
    sleep 1
  done
  trap 'rm -f "$LOCK"' EXIT
}

# A pid is gone only if `kill -0` says "No such process" (another user's live process answers "not permitted").
pid_gone() { err="$(LC_ALL=C kill -0 "$1" 2>&1)" && return 1; case "$err" in *"o such process"*) return 0 ;; esac; return 1; }

# The rendering: deterministic, from the journal alone (optionally its first N lines). The footer records how many
# journal lines it reflects, so a render that a crash left behind is never mistaken for a hand edit.
render_to() {
  awk -F'\t' -v lim="${2:-0}" '
    lim > 0 && NR > lim { exit }
    { cnt = NR }
    function sec(k) { return k == "parked" ? 1 : k == "stopped" ? 2 : k == "veto" ? 3 : k == "held-merge" ? 4 : k == "hold" ? 5 : k == "retro" ? 6 : 7 }
    $2 == "init"    { run = $4; next }
    $2 == "add"     { n++; id[n] = $3; at[$3] = $1; kind[$3] = $4; unit[$3] = $5; q[$3] = $6; opt[$3] = $7; rec[$3] = $8; blk[$3] = $9; next }
    $2 == "answer"  { ans[$3] = $5 " (by " $4 (($7 != "") ? " via " $7 : "") ", " $1 "#" NR (($6 != "") ? "; quoting the owner: \"" $6 "\"" : "") ")"; next }
    $2 == "resolve" { res[$3] = "resolved by " $4 ", " $1 (($6 != "") ? " -- " $6 : ""); next }
    $2 == "edit"    { ne++; ed[ne] = $4 " (" $1 ")"; next }
    $2 == "hold"    { hold = "requested " $1 ": " $4; next }
    $2 == "hold-clear" { hold = ""; next }
    END {
      print "# Unattended inbox" (run != "" ? " -- run " run : "")
      print ""
      print "Answer an item with `bash _bmad/scripts/unattended-inbox.sh <this directory> answer U-<n> \"<answer>\" --by owner`, or run `/epic-cycle --review`. This file is generated: a hand edit is saved aside and never applied."
      if (hold != "") { print ""; print "**Hold " hold "**" }
      split("Parked|Stopped|Veto window|Held merges|Holds|Retrospectives to review|Run reports", title, "|")
      for (s = 1; s <= 7; s++) {
        hdr = 0
        for (i = 1; i <= n; i++) {
          u = id[i]; if (sec(kind[u]) != s || (u in res)) continue
          if (!hdr) { print ""; print "## " title[s]; hdr = 1 }
          print ""; print "### " u " | " kind[u] " | " unit[u] " | " at[u]
          print "question: " q[u]; print "options: " opt[u]; print "recommendation: " rec[u]; print "blocks: " blk[u]
          print "answer: " ((u in ans) ? ans[u] : "(none yet -- answer with the script or --review)")
        }
      }
      if (ne) { print ""; print "## Hand edits (saved, not applied)"; for (i = 1; i <= ne; i++) print "- " ed[i] }
      hdr = 0
      for (i = 1; i <= n; i++) {
        u = id[i]; if (!(u in res)) continue
        if (!hdr) { print ""; print "## Resolved"; hdr = 1 }
        print "- " u " | " kind[u] " | " unit[u] ": " q[u] " -> " ((u in ans) ? ans[u] : "(no answer)") "; " res[u]
      }
      print ""; print "<!-- journal: " cnt " lines -->"
    }' "$J" > "$1.tmp.$$" && mv "$1.tmp.$$" "$1"
}

# Called under the lock before every write: a hand edit of inbox.md is saved and recorded, never applied.
save_edit() {
  [ -f "$MD" ] || return 0
  render_to "$STATE/.expected"
  if ! cmp -s "$STATE/.expected" "$MD"; then
    seen="$(sed -n 's/^<!-- journal: \([0-9]*\) lines -->$/\1/p' "$MD" | tail -1)"
    if [ -n "$seen" ]; then render_to "$STATE/.earlier" "$seen"; fi
    if [ -n "$seen" ] && cmp -s "$STATE/.earlier" "$MD"; then
      :   # an earlier rendering left by a crash between a journal write and its render -- not a hand edit
    else
      f="inbox.md.edited-$(date -u +%Y%m%dT%H%M%SZ)-$$"; cp "$MD" "$STATE/$f"
      printf '%s\tedit\t-\t%s\n' "$(now)" "$f" >> "$J"
      echo "NOTE: inbox.md had a hand edit; saved as $f (not applied)" >&2
    fi
  fi
  rm -f "$STATE/.expected" "$STATE/.earlier"
}

item_exists()   { awk -F'\t' -v id="$1" '$2 == "add" && $3 == id { f = 1 } END { exit f ? 0 : 1 }' "$J"; }
item_resolved() { awk -F'\t' -v id="$1" '$2 == "resolve" && $3 == id { f = 1 } END { exit f ? 0 : 1 }' "$J"; }
lock_field()    { [ -f "$RL" ] && awk -v k="$1" '{ for (i = 1; i <= NF; i++) if (index($i, k "=") == 1) { print substr($i, length(k) + 2); exit } }' "$RL"; }
write_lock() {  # run-id holder started token where
  printf 'run=%s holder=%s host=%s started=%s token=%s heartbeat=%s heartbeat_epoch=%s last_boundary=%s\n' \
    "$1" "$2" "$(uname -n)" "$3" "$4" "$(now)" "$(date -u +%s)" "$5" > "$RL.tmp.$$" && mv "$RL.tmp.$$" "$RL"
}
nospace() { case "$2" in *" "*) echo "ERROR: $1 must not contain spaces" >&2; exit 1 ;; esac; }
new_token() { printf '%s-%s' "$(date -u +%s)" "$$"; }
own_lock() {    # run-id token -- exit 4 unless this session holds the lock
  [ -f "$RL" ] || { echo "ERROR: no run.lock -- this session no longer holds the run" >&2; exit 4; }
  [ "$(lock_field run)" = "$1" ] && [ "$(lock_field token)" = "$2" ] || { echo "ERROR: run.lock is held by run $(lock_field run) (holder $(lock_field holder)), not this session -- stop" >&2; exit 4; }
}

case "$CMD" in
  init)
    need "run-id" "${3:-}"; single "$3"; mkdir -p "$STATE"; lock
    [ ! -f "$J" ] || save_edit
    if [ ! -f "$J" ] || [ "$(awk -F'\t' '$2 == "init" { r = $4 } END { print r }' "$J")" != "$3" ]; then
      printf '%s\tinit\t-\t%s\n' "$(now)" "$3" >> "$J"
    fi
    render_to "$MD"; echo "inbox=$MD" ;;
  add)
    [ $# -eq 8 ] || { echo "ERROR: add needs exactly 6 arguments (see usage)" >&2; usage; }
    case "$3" in parked|stopped|veto|held-merge|hold|retro|report) ;; *) echo "ERROR: kind=$3 is not one of parked|stopped|veto|held-merge|hold|retro|report" >&2; exit 1 ;; esac
    need "unit" "$4"; need "question" "$5"
    for v in "$4" "$5" "$6" "$7" "$8"; do single "$v"; done
    need_journal; lock; save_edit
    case "$3" in veto|hold|report) existing="" ;;
      *) existing="$(K="$3" U="$4" awk -F'\t' '$2 == "add" && $4 == ENVIRON["K"] && $5 == ENVIRON["U"] { last = $3 } $2 == "resolve" { r[$3] = 1 } END { if (last != "" && !(last in r)) print last }' "$J")" ;;
    esac
    if [ -n "$existing" ]; then render_to "$MD"; echo "$existing EXISTS"; exit 0; fi
    n="$(awk -F'\t' '$2 == "add" { c++ } END { print c + 1 }' "$J")"
    printf '%s\tadd\tU-%s\t%s\t%s\t%s\t%s\t%s\t%s\n' "$(now)" "$n" "$3" "$4" "$5" "$6" "$7" "$8" >> "$J"
    render_to "$MD"; echo "U-$n" ;;
  answer)
    ID="${3:-}"; ANS="${4:-}"; need "U-<n>" "$ID"; need "answer" "$ANS"
    BY=""; QUOTE=""; VIA=""; shift 4
    while [ $# -gt 0 ]; do
      case "$1" in
        --by|--quote|--via) [ $# -ge 2 ] || { echo "ERROR: $1 needs a value" >&2; exit 1; }
          case "$1" in --by) BY="$2" ;; --quote) QUOTE="$2" ;; --via) VIA="$2" ;; esac; shift 2 ;;
        *) usage ;;
      esac
    done
    need "--by" "$BY"
    [ "$BY" = "owner" ] || [ -n "$QUOTE" ] || { echo "ERROR: an answer relayed by '$BY' must quote the owner verbatim (--quote)" >&2; exit 1; }
    for v in "$ID" "$ANS" "$BY" "$QUOTE" "$VIA"; do single "$v"; done
    need_journal; lock; save_edit
    item_exists "$ID" || { echo "ERROR: $ID not found" >&2; exit 1; }
    ! item_resolved "$ID" || { echo "ERROR: $ID is already resolved" >&2; exit 1; }
    printf '%s\tanswer\t%s\t%s\t%s\t%s\t%s\n' "$(now)" "$ID" "$BY" "$ANS" "$QUOTE" "$VIA" >> "$J"
    render_to "$MD"; echo "$ID answered" ;;
  answers)
    need_journal
    awk -F'\t' '$2 == "answer" { a[$3] = $1 "#" NR "\t" $4 "\t" $7 "\t" $5 "\t" $6; if (!($3 in seen)) { seen[$3] = 1; ord[++n] = $3 } } $2 == "resolve" { r[$3] = 1 }
      END { for (i = 1; i <= n; i++) if (!(ord[i] in r)) print ord[i] "\t" a[ord[i]] }' "$J" ;;
  resolve)
    ID="${3:-}"; BY="${4:-}"; AT="${5:-}"; NOTE="${6:-}"
    need "U-<n>" "$ID"; need "applied_by" "$BY"; need "answer_ref (or -)" "$AT"
    for v in "$ID" "$BY" "$AT" "$NOTE"; do single "$v"; done
    need_journal; lock; save_edit
    item_exists "$ID" || { echo "ERROR: $ID not found" >&2; exit 1; }
    ! item_resolved "$ID" || { echo "ERROR: $ID is already resolved -- an item is resolved exactly once" >&2; exit 1; }
    latest="$(awk -F'\t' -v id="$ID" '$2 == "answer" && $3 == id { t = $1 "#" NR } END { print t }' "$J")"
    if [ "$AT" = "-" ]; then
      [ -z "$latest" ] || { echo "ERROR: $ID has an answer ($latest); resolve with that answer_ref after applying it" >&2; exit 1; }
    else
      [ "$AT" = "$latest" ] || { echo "ERROR: $ID: the latest answer is from ${latest:-nobody}, not $AT -- read \`answers\` again and apply that one" >&2; exit 1; }
    fi
    printf '%s\tresolve\t%s\t%s\t%s\t%s\n' "$(now)" "$ID" "$BY" "$AT" "$NOTE" >> "$J"
    render_to "$MD"; echo "$ID resolved" ;;
  list)
    need_journal
    awk -F'\t' -v which="${3:-open}" '$2 == "add" { n++; id[n] = $3; k[$3] = $4; u[$3] = $5; q[$3] = $6 } $2 == "answer" { a[$3] = 1 } $2 == "resolve" { r[$3] = 1 }
      END { for (i = 1; i <= n; i++) { x = id[i]; s = (x in r) ? "resolved" : (x in a) ? "answered" : "open"
            if (which == "all" || s != "resolved") print x "\t" k[x] "\t" u[x] "\t" s "\t" q[x] } }' "$J" ;;
  hold|hold-clear)
    need "reason" "${3:-}"; single "$3"
    case "${4:-}" in "") ;; --usage-only) [ "$CMD" = hold-clear ] || usage ;; *) usage ;; esac
    need_journal; lock; save_edit
    cur=""; [ ! -f "$STATE/HOLD" ] || cur="$(sed -n 's/^requested_at=[^ ]* reason=//p' "$STATE/HOLD")"
    if [ -f "$STATE/HOLD" ]; then
      if [ "$CMD" = hold ] && [ "$(rank "$3")" -lt "$(rank "$cur")" ]; then render_to "$MD"; echo "HOLD KEPT: $cur"; exit 0; fi
      if [ "${4:-}" = --usage-only ] && [ "$(rank "$cur")" -ne 0 ]; then render_to "$MD"; echo "HOLD KEPT: $cur"; exit 5; fi
    fi
    printf '%s\t%s\t-\t%s\n' "$(now)" "$CMD" "$3" >> "$J"
    if [ "$CMD" = hold ]; then printf 'requested_at=%s reason=%s\n' "$(now)" "$3" > "$STATE/HOLD"; else rm -f "$STATE/HOLD"; fi
    render_to "$MD"; echo "$CMD recorded" ;;
  run-status)
    if [ -f "$RL" ]; then
      e="$(lock_field heartbeat_epoch)"; age=$(( ( $(date -u +%s) - ${e:-0} ) / 60 ))
      echo "LIVE? $(cat "$RL") heartbeat_age_min=$age"; exit 3
    fi
    echo "none" ;;
  run-start)
    need "run-id" "${3:-}"; need "holder" "${4:-}"; single "$3"; single "$4"; nospace "run-id" "$3"; nospace "holder" "$4"; mkdir -p "$STATE"; lock
    if ! ( set -o noclobber; : > "$RL" ) 2>/dev/null; then
      e="$(lock_field heartbeat_epoch)"; echo "LIVE? $(cat "$RL") heartbeat_age_min=$(( ( $(date -u +%s) - ${e:-0} ) / 60 ))"; exit 3
    fi
    t="$(new_token)"; write_lock "$3" "$4" "$(now)" "$t" "start"
    [ ! -f "$J" ] || { save_edit; printf '%s\trun-start\t-\t%s %s\n' "$(now)" "$3" "$4" >> "$J"; render_to "$MD"; }
    echo "run.lock taken: run=$3 holder=$4 token=$t" ;;
  run-takeover)
    need "run-id" "${3:-}"; need "holder" "${4:-}"; need "reason" "${5:-}"; single "$3"; single "$4"; single "$5"; nospace "run-id" "$3"; nospace "holder" "$4"
    [ -f "$RL" ] || { echo "ERROR: no run.lock to take over; use run-start" >&2; exit 1; }
    lock; [ -f "$RL" ] || { echo "ERROR: no run.lock to take over; use run-start" >&2; exit 1; }
    held="$(lock_field run)"
    case "$5" in owner-confirmed*) ;; *) [ "$held" = "$3" ] || { echo "ERROR: run.lock names run $held, not $3; only the owner can release another run's lock (reason \"owner-confirmed ...\")" >&2; exit 1; } ;; esac
    st="$(now)"; case "$5" in owner-confirmed*) ;; *) st="$(lock_field started)"; [ -n "$st" ] || st="$(now)" ;; esac
    t="$(new_token)"; write_lock "$3" "$4" "$st" "$t" "takeover"
    [ ! -f "$J" ] || { save_edit; printf '%s\trun-takeover\t-\t%s %s from %s: %s\n' "$(now)" "$3" "$4" "$held" "$5" >> "$J"; render_to "$MD"; }
    echo "run.lock taken over: run=$3 holder=$4 (was $held) token=$t" ;;
  run-boundary)
    need "run-id" "${3:-}"; need "token" "${4:-}"; single "$3"; single "$4"; single "${5:-}"
    own_lock "$3" "$4"; lock; own_lock "$3" "$4"   # checked again under the writer lock: a takeover may have just landed
    write_lock "$3" "$(lock_field holder)" "$(lock_field started)" "$4" "${5:-}" ;;
  run-end)
    need "run-id" "${3:-}"; need "token" "${4:-}"; need "reason" "${5:-}"; single "$3"; single "$4"; single "$5"
    own_lock "$3" "$4"; lock; own_lock "$3" "$4"
    rm -f "$RL"
    [ ! -f "$J" ] || { save_edit; printf '%s\trun-end\t-\t%s: %s\n' "$(now)" "$3" "$5" >> "$J"; render_to "$MD"; }
    echo "run.lock released ($5)" ;;
  verify)
    need_journal; lock; T="$STATE/.verify"; render_to "$T"
    if cmp -s "$T" "$MD"; then rm -f "$T"; echo "ok"; else rm -f "$T"; echo "MISMATCH: inbox.md is not the journal's rendering (a hand edit, or a crash mid-write)"; exit 1; fi ;;
  render)
    need_journal; lock; save_edit; render_to "$MD"; echo "rendered" ;;
  *) usage ;;
esac
```

**`.gitignore`:** append `_bmad-output/unattended/` (create the file if absent; skip if the line is already there). The run state directory must stay out of git: an owner's answer written while a stage runs must not dirty the tree that `bmad-build-auto`'s finalize checks.

**`.gitattributes`:** append `_bmad/scripts/*.sh text eol=lf` (create the file if absent; skip if the line is already there). A CRLF checkout — Git for Windows' default — breaks `ledger.sh` and `unattended-inbox.sh` (`set: pipefail: invalid option name`), and every worktree is a fresh checkout.

---

## Step 3: Write `.claude/commands/epic-cycle.md`

Write everything between the BEGIN and END markers below to that file path, verbatim, including the frontmatter — from the `---` line after the BEGIN marker's closing fence to the line before the END marker's opening fence (the fence lines around each marker are not part of the file). Overwrite any existing file (you should have already backed it up in Step 1).

```text
=== BEGIN .claude/commands/epic-cycle.md ===
```

---
description: Run the BMAD Method epic development cycle for one or more epics
---

You are executing the BMAD Method development implementation cycle for one or more epics. Stories run strictly sequentially — one story, one stage at a time, in one checkout (see "Smart Parallelism" for why).

**After any context compaction, re-read this file (`.claude/commands/epic-cycle.md`) before your next action** — at least "Task Sequence", the section for the stage or gate you are in, and "Lead Context Management". Compaction keeps only about the first 5,000 tokens of this command, which end before the stage instructions; a lead working from the summary alone drops the stage blocks, the pre-answers and the containment rules from every later spawn prompt.

**Arguments:** $ARGUMENTS — an epic range (e.g., `1-3` for Epics 1 through 3, `2` for a single epic), optionally followed by `--unattended`; or `--review` alone. If there is no range and no `--review`, prompt the user for the range; a range with `--review` is an argument error.

**`--unattended` or `--review`: read `_bmad/custom/epic-cycle-unattended.md` (the doctrine) in full before anything else, and again after every compaction.** Under those flags it overrides every passage it names — the passages below carry an *Unattended:* pointer to its section (U1–U14) — and after the attended setup the development loop never waits for input: every stop point is decided, vetoed, parked, stopped or held. Without those flags, read it only for the case in its U10 (an attended run after an unattended one). **`--review` alone runs only the doctrine's U13** — the attended review of an unattended run's inbox — and none of the pre-flight gates, setup or Task Sequence below.

## Pre-flight Runtime Check

Uses the standard `Agent` tool to spawn pipeline-stage subagents and the `Skill` tool for lead-side skill invocations. No experimental flag required.

**Lead model gate (first action, before any other gate):** this command's `model: opus` frontmatter pin is officially documented but currently IGNORED at runtime (anthropics/claude-code #45191) — never assume it applied. Read your own model declaration from your system prompt ("You are powered by the model named …"). If the lead is an Opus-tier-or-higher model (Opus, or an above-Opus tier such as Fable/Mythos): proceed and log `lead_model_gate model=<detected> action=proceed`. Otherwise STOP and ask the user: switch the session (`/model opus`, or restart with `claude --model opus`) and re-invoke, or explicitly accept a lower-tier lead for this run — acceptance is logged `lead_model_gate model=<detected> action=user_accepted` so any later quality dip is attributable. Rationale: the lead's judgment artifacts (story specs, merge-conflict resolutions, AD verdicts, DAG analysis) are the pipeline's least-reviewed outputs; the stage→model map assumes they come from the highest tier in the run. Stage subagents are unaffected either way — they receive explicit `model` parameters on their Agent calls, which DO work. Above-Opus note: the gate accepts Fable/Mythos-tier leads, but Opus is the intended price/performance point for the lead (~2x the cost for marginal lead-side gain) — treat an above-Opus lead as a deliberate exception for an exceptionally hard epic, never a standing default.

If `Agent` is a deferred tool, load its schema via `ToolSearch` with `"select:Agent"` before first use. If `Agent` is unavailable, halt and surface to the user.

**BMAD + runtime gate (second action):** read `installation.version` from `_bmad/_config/manifest.yaml` and require ≥ 6.11.0 — this command drives `bmad-build-auto`, which does not exist earlier (6.10 projects: use the pre-6.11 kit). **BMAD v7 gate**, checked before the version comparison (a v7 project may have no manifest, or a stale 6.12 one under v7 skills): `.claude/skills/bmad-build-auto/customize.toml` (or its plugin-namespaced equivalent) must declare `[[workflow.review_layers]]`, and no `bmod-method/bmod.toml` may exist under the project's `.claude/skills/`. (One only under `~/.claude/skills/` is a user-level v7 preview: if `_bmad/custom/kit-versions.yaml` records `v7_global_accepted:`, log `v7_global accepted=<date>` and continue; otherwise HALT and tell the user to remove the preview or re-run the kit's install Step 1, which records the acceptance.) Otherwise HALT with: "This kit supports BMAD Method 6.11–6.12.x. This project runs BMAD v7 (<the marker found>), which replaces the review layers, sprint planning and story files the kit drives — its overrides would stop v7's renderer. To keep using these kits, keep the project on 6.12.x (`npx bmad-method@6.12 install --action update --modules bmm --tools claude-code --yes` in the project directory); try the v7 preview in a separate clone." Run `uv --version`; HALT with the install pointer (https://docs.astral.sh/uv/) if it fails — `bmad-build-auto` renders itself through `uv run _bmad/scripts/render_skill.py` and halts on activation without it. Then run `uv run --no-cache _bmad/scripts/resolve_config.py --project-root <abs-root> --key core.project_name`: exit 0 proves uv can run BMAD's scripts (Python ≥ 3.11 available or provisioned); exit 3 means no Python 3.11+ — HALT. Verify these exist: `.claude/skills/bmad-build-auto/SKILL.md`, `.claude/skills/bmad-code-review/SKILL.md`, `.claude/skills/bmad-qa-generate-e2e-tests/SKILL.md`, `.claude/skills/bmad-sprint-planning/scripts/sprint_plan.py` (on a plugin-namespaced install, the equivalent paths). Verify the durable state is tracked: `git check-ignore -q _bmad-output/implementation-artifacts` must FAIL (exit 1) — if the directory is gitignored, HALT: cycle logs, `sprint-status.yaml`, specs and the ledger must live on the epic branch (resume, bookkeeping commits, and the parallel merge queue depend on it); the user removes `_bmad-output/` from `.gitignore` (keeping `_bmad/render/`, `*.user.toml`, `.worktrees/` ignored). Then determine how CI can be read (see "CI Is a Gate"): `ci=gh` when `.github/workflows/` holds a `*.yml` or `*.yaml` workflow, `gh auth status` succeeds, and `git remote get-url origin` is a GitHub URL; `ci=none` only when the repo has no CI configuration at all — no GitHub workflow and none of `.gitlab-ci.yml`, `azure-pipelines.yml`, `.circleci/config.yml`, `Jenkinsfile`, `bitbucket-pipelines.yml`, `.buildkite/`, `.travis.yml`, `.drone.yml`; `ci=unavailable` otherwise (CI exists, but this session cannot read its results) — surface `unavailable` once ("CI results cannot be read from this session; the merge gate will ask you to confirm CI by hand") rather than proceeding as if the suite's local result were the shipped one. Log `runtime_gate bmad=<version> uv=<version> ci=<gh|none|unavailable>`. Resolve `<project_name>`, `<impl>` and `<planning>` with BMAD's config resolver (the shorthand paragraph below). Read the launch environment ("Permission Mode": `printenv CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH CLAUDE_CODE_RETRY_WATCHDOG BASH_MAX_TIMEOUT_MS`) and log `launch_env`; an explicit depth below what the run needs (2 sequential, 3 parallel) → HALT and say which setting to raise.

**Unattended run lock (attended runs — under `--unattended` the doctrine's U2 decides instead, and a runner skips this):** if `_bmad-output/unattended/run.lock` or `<worktree_base>/.coordination/unattended/run.lock` exists, an unattended run may hold this checkout — show `bash _bmad/scripts/unattended-inbox.sh <its directory> run-status` and stop; only the owner can confirm that run is dead and release its lock (doctrine U10, last bullet). Two sessions never work one checkout.

**Telemetry gate (third action):** scan every existing `cycle-log-epic-*.md` for `model_tier_checkpoint` entries whose `result` is `recommend_escalate` or `offer_mixed` with `applied=false`. If any exist, surface them once before starting ("Epic N's checkpoint recommended escalating `implement` to Opus (evidence …); no policy file exists / the user declined — apply now, or continue as is?") and log `telemetry_gate pending=<N>`. Accumulated telemetry that nobody acts on is the failure mode this gate exists for.

**`SPRINT_PLAN` shorthand.** Throughout this command, `SPRINT_PLAN` means `uv run --no-cache <path-to>/bmad-sprint-planning/scripts/sprint_plan.py`, where `<path-to>` is `.claude/skills` (or the plugin-namespaced skills directory). Three more placeholders recur below: `<impl>` and `<planning>` (also written `<implementation_artifacts>` / `<planning_artifacts>`) and `<project_name>`. Resolve all three once, at pre-flight, through BMAD's own resolver — `uv run --no-cache _bmad/scripts/resolve_config.py --project-root <abs-root> --key core.project_name --key modules.bmm.implementation_artifacts --key modules.bmm.planning_artifacts` (JSON out) — and substitute `{project-root}`; by default they are `_bmad-output/implementation-artifacts` and `_bmad-output/planning-artifacts`. Never read them from `_bmad/config.toml` alone: the resolver merges the project's config layers (`_bmad/config.toml`, `_bmad/custom/config.toml`, and the user files), as the rendered skills (`bmad-build-auto`) do. Four pipeline skills do not: `bmad-code-review`, `bmad-qa-generate-e2e-tests`, `bmad-sprint-planning` and `bmad-retrospective` read `_bmad/bmm/config.yaml`. So compare: the resolver's `implementation_artifacts` and `planning_artifacts` must equal those keys in `_bmad/bmm/config.yaml` (after substituting `{project-root}`); a mismatch → HALT — those four skills would write somewhere other than the lead and build-auto. Align the two (re-run the BMAD installer, or set the path in both files) and re-invoke. It is BMAD's deterministic tracker script (JSON in, JSON out; `uv run` may print an `Installed N package(s)` line to stderr — ignore it). Its three subcommands replace the retired `/bmad-sprint-status mode=data|validate`:

- `SPRINT_PLAN status --status-file <impl>/sprint-status.yaml` → per-status counts, `open_action_items`, `recommendation.story_key`, `all_done`.
- `SPRINT_PLAN validate --status-file <impl>/sprint-status.yaml` → `valid`, `problems`.
- `status` has no per-story list. **`TRACKER_READ`** — `awk '/^development_status:/ { f = 1; next } f && /^[^ #]/ { f = 0 } f && /^  [^ #]/ { print $1, $2 }' <impl>/sprint-status.yaml` — prints `key: status` per line; it is the one sanctioned per-story read (a read, never an edit).

**`LEDGER` shorthand.** `LEDGER` means `bash _bmad/scripts/ledger.sh <impl>/deferred-work.md` (installed by this kit). `LEDGER load` → counts + `owner:<key>=<n>` lines; `LEDGER slice <owner>|all` → `DW-n TAB status TAB owner TAB summary` for non-terminal entries; `LEDGER show DW-n` → one entry; `LEDGER new "<summary>" "<source>" <severity> <fix-risk> <footprint> "<evidence>" <status> <owner> <by> "<note>"` → creates an entry, prints its id; `LEDGER append DW-n "status=… owner=… by=… note=…"` → one trailer line. Nobody in the pipeline reads or edits `deferred-work.md` any other way.
- `SPRINT_PLAN generate --epic-file <planning>/epics.md --status-file <impl>/sprint-status.yaml --stories-dir <impl> --project <project_name> --date "<MM-DD-YYYY HH:MM>" --set <key>=<status> [--set …]` → the lead's ONLY sanctioned status writer. The merge itself never downgrades, but **`--set` is the script's repair path and is applied unconditionally — it WILL downgrade.** Before every `--set`, read the key's current value (`SPRINT_PLAN status` or the file) and refuse to write a lower rank (`backlog < ready-for-dev < in-progress < review < done`) — the exceptions are the Rework Loop's re-open of a `done` story (its step 1), logged `rework_opened`, and an unattended set-aside (doctrine U4). `--set` on a key that is not in the generated plan fails (`ok: false`, "not in the generated plan") and writes nothing — a story must exist in the epic file first. Preserves comments; reports `explicit_set`. Use `--dry-run` when unsure. (`<epic-file>` may repeat for multi-file epics; `<project_name>`, `<impl>` and `<planning>` are the resolver's values from the shorthand paragraph above.)

## Task Sequence

**Per Epic (setup, executed once per epic before any stories):**

1. **Lead** verifies a clean working tree across the parent repo and every submodule listed in `.gitmodules`; halts on dirty state.
2. **Lead** determines per-repo resume mode (see "Resume Semantics").
3. For repos in FRESH mode: **Lead** verifies (or creates with user authorization) the feature branch per Rule SC-1, then verifies (or creates) the epic branch `{TICKET}-epic{N}` per Rule SC-2.
4. **Lead** checks every affected repo out to `{TICKET}-epic{N}` and logs `epic_branch_checked_out` (on the run's first epic, after first writing the held pre-flight entries — see "Cycle Log Format", `runtime_gate`); then resolves the **architecture spine** (Rule 6: newest `ARCHITECTURE-SPINE.md` with `status: final` under `<planning>/architecture/*/`, else the newest spine, else any `*architecture*.md` under `<planning>`, else none), counts its `### AD-` blocks, and logs `spine_resolved path=<path|none> ads=<N> status=<final|draft|none>`. The path goes into every spawn prompt; Rule 20 writes go to this file.
5. **Lead** measures the ledger (`LEDGER load`; re-owns any `owner=none` entry) and logs `ledger_load`, then executes `/bmad-sprint-planning` directly, **headless**, with the load in its argument, and branches on its readiness `gate` (see "Sprint Planning Per Epic" and "Ledger Drain").
6. If a previous epic's retrospective, the ledger (`LEDGER slice all`), or open `action_items` in `sprint-status.yaml` have unresolved items, **Lead** reviews them, **closes** each (resolve / own / terminal — Rule 17), and creates Story X.0 via a `bmad-build-auto` plan spawn (see "Retrospective Review & Story X.0 Creation").

**Per Story (executed once per story in the epic):**

1. **Lead** asserts the branch, writes the `stage_spawned stage=plan …` write-ahead entry to the cycle log, and makes the bookkeeping commit (Rule 16 — the tree MUST be clean before every `bmad-build-auto` dispatch). Every later stage spawn (implement, qa, code-review) is preceded by its own `stage_spawned` entry the same way.
2. Agent — **PLAN**: `/bmad-build-auto` on the Opus tier, prompt = the explicit sprint-status story key plus `Halt after planning.`, plus the story's **ledger inbox** (`LEDGER slice <key>` pasted in — every entry addressed in Tasks & Acceptance or declined under Design Notes with a reason; Rule 17 (1b)). It writes `spec-{key}.md` and halts with status `ready-for-dev`. **Lead captures the spec path** and logs `story_created`. The plan does not wait on the previous story's CI run — a plan writes a spec, not code, so it cannot build on a red (see step 4).
3. **Lead** gates the spec: Integration-AC validation (Rules 1/2), AD mapping against the spine (Rule 6), sections the template may have deleted, decision dependency (a spec that builds on a `decision-pending` ledger entry is an intent gap for the user NOW, not at the merge gate — *Unattended:* ruled or parked, U5); logs `spec_validated`; sets the story `ready-for-dev` via `SPRINT_PLAN generate --set`; bookkeeping commit (everything under `implementation-artifacts`: spec, log, tracker).
4. **Lead** resolves the previous story's `ci=pending` (see "CI Is a Gate", resolution point 1) — a red reworks that story first while this story's spec waits at `ready-for-dev`. Then the lead sets the story `in-progress` (+ `epic-{N}` lift) via `SPRINT_PLAN`, bookkeeping commit; then Agent — **IMPLEMENT**: `/bmad-build-auto` on the Sonnet tier, prompt = the spec path. It implements (via its handoff subagent), self-reviews with its review layers, finalizes the spec `status: done`, and makes a local commit. **Lead** reads the spec frontmatter (`baseline_revision`; accept `baseline_commit` if a newer build-auto writes that key instead — the retrospective skill already calls `baseline_revision` deprecated), harvests `deferred:` into the ledger — each item born with an owner via `LEDGER new` (Rule 15) — sets the story `review` via `SPRINT_PLAN`, and logs `dev_complete` with `build_sha=`. The story's FIRST `dev_complete` also logs `story_base=<baseline_revision>` (the commit its first implement `stage_spawned` recorded; an unattended pick-up logs a new one — doctrine U4 — and the gates then use the latest): every rework overwrites `baseline_revision`, so the gates that judge the story's whole delivered scope — adjudication, smoke, the fresh-environment check — diff from `story_base`; only the scoped re-review uses the rework's own baseline.
5. **Lead** executes any AD-tooled AC verifications, each with a named, applied, and reverted mutation (Rule 19 — a pass without one is `unverified`; see "AD-Aware Execution").
6. Agent: `/bmad-qa-generate-e2e-tests` (scope pre-answered to this story's deliverable = `git diff --name-only <baseline_revision>..HEAD`; one demonstrated mutation per AC, Rule 19).
7. Agent: `/bmad-code-review` on the Opus tier, with the spec path as the explicit argument (`review_mode = full`). After it returns: `SPRINT_PLAN validate` (the skill's own sprint-status sync is pre-answered to the script, but verify) and `grep -c "^## Deferred from:" <impl>/deferred-work.md` — if that count grew, the reviewer ignored the ledger pre-answer: harvest each stray bullet with `LEDGER new … cr`, strip the stray block (the one sanctioned edit — those bullets are not DW entries), and log `ledger_stray_bullets harvested=N`. Copy the reviewer's `entries:` line into `cr_complete` (root-cause counts, `rows=`, `unresolved_high_med=`). If the review leaves the story `in-progress` (a HIGH, or — first review only — an unpatched in-story MED), run the Rework Loop (max 3 iterations; each re-review scoped to the rework; a re-review with no HIGH closes the story) before proceeding.
8. **Lead** adjudicates the ledger slice this story owns (Rule 17): every entry with effective `owner=<this story key>` is closed against DELIVERED scope — `resolved-by` with evidence, re-owned with the residual restated, or terminal — until `LEDGER slice <key>` reads empty. Logs `ledger_adjudicated`.
9. **Lead** performs per-story smoke and then the fresh-environment check — the story's added and changed tests on a fresh environment, forward and reversed (Rule 19; see "Per-Story Smoke"); logs `smoke_complete` and `fresh_check`.
10. **Lead** commits (QA tests, review patches, ledger, bookkeeping) and pushes — **only to the epic branch** in every affected repo, never to main/master/develop (Rule SC-3 + SC-6) — then records the CI run the push started on the `committed` entry (`ci=pending run=<id>`, or `ci=none` / `ci=unavailable` per the pre-flight; see "CI Is a Gate"), and logs `story_cycle` (see "Cycle Log Format"). The result is resolved before the next story's implement spawn, never assumed.

A story therefore produces several commits on the epic branch: bookkeeping commits, the plan commit (spec), `bmad-build-auto`'s finalize commit (`build_sha`), and the lead's post-smoke commit (`committed sha`). That is by design; the epic's `--no-ff` merge preserves the graph.

**End of Epic (executed once per epic after all stories):**

1. **Lead** runs the **burn-down gate** (Rule 17, see "Ledger Drain"): every story-closable (`routed`/`open`) ledger entry owned by `burndown` or by one of this epic's story keys is resolved / made terminal now, chartered into ONE burn-down story `N.<M+1>` (M = the highest existing story number under Epic N in `epics.md` or `sprint-status.yaml`; at most `burndown_story_max` entries, by priority) that runs the full per-story pipeline before the epic closes, or — the explicit overflow — re-owned to a specific next-epic story key. `escalated` / `decision-pending` entries are not work: they are counted and handed to the decision sheet (step 5). Logs `ledger_burndown_complete` (or `ledger_burndown_skipped reason=empty`).
2. **Lead** sets `epic-{N}: done` via `SPRINT_PLAN generate --set epic-{N}=done` — a transition no skill writes (see "Status Ownership") — and logs `epic_status_done`.
3. **Lead** runs the model-tier checkpoint on EVERY project (see "Model Strategy") and logs `model_tier_checkpoint` with its evaluation; a change to tiers is applied only when `_bmad/custom/model-overrides.yaml` exists and the user approves (then also `model_tier_changed`) — on a project without the policy file a met threshold is surfaced as a recommendation, never silently dropped.
4. **Lead** pauses: "Run a retrospective?" If yes, execute `/bmad-retrospective` fully interactive — the human-in-the-middle exception; see "Retrospective Per Epic".
5. **Lead** presents the **epic-close decision sheet** (Rule 17 (5): every `escalated` and `decision-pending` entry with a recommended disposition, decided in one sitting; human-owned items move to `action_items`), records the answers as `by=merge_gate` trailer lines, logs `decision_sheet_recorded`, and then pauses: "Merge `{TICKET}-epic{N}` into the feature branch and delete the epic branch (local + remote)?" — per Rule SC-4. If yes, execute the merge in each affected repo (submodules-first, parent last).

## Execution Guidelines

Each pipeline-stage task is delegated via the `Agent` tool. Lead-side skills (sprint planning, retrospective) are invoked directly via the `Skill` tool. Story planning is a **spawned** stage in v6.11 (`bmad-build-auto` never advances to another story, so the old race-ahead concern that kept story creation lead-side is gone) — what stays lead-side is the *validation* of the plan. For `bmad-build-auto` stages the completion contract is the spec file (frontmatter `status` + `## Auto Run Result`), not prose. For QA/CR stages, if a return is missing closing-summary sections, the lead extracts the file list from `git status --short` plus `git diff --name-only <baseline_revision>..HEAD` — this is normal extraction.

`/bmad-code-review` triages findings into **decision-needed / patch / defer / dismiss** buckets at **low / medium / high** severity. Under `/epic-cycle` the review agent applies every patch-bucket finding and resolves decision-needed items using best judgment plus the BMAD skill rules; only a genuine user-preference or spec-contradiction question comes back as `## Clarification Needed`. The BMAD skills must be invoked via `Skill`; don't skip steps. `/bmad-retrospective` is the one deliberate human-in-the-middle exception to the autonomous loop — it MUST run in interactive mode, and every one of its elicitation stops must reach the user (*Unattended:* headless, `-H <epic>`, doctrine U9).

## Permission Mode (Critical)

Subagents **inherit the lead session's permission mode** — on current Claude Code builds the `Agent` tool's `mode` parameter is accepted but ignored ("Deprecated; ignored. Subagents inherit the parent session's permission mode"). The effective control is therefore how the lead session was started: for an unattended run launch it with `claude --permission-mode bypassPermissions` (or `--dangerously-skip-permissions`), or `--permission-mode auto` where the harness offers it with a configured environment. Pass `mode: "bypassPermissions"` on every `Agent` call whose schema offers a `mode` parameter — harmless where ignored, honoured on older builds; where the schema has no `mode` (load it with `ToolSearch` if `Agent` is deferred), omit it, because an unknown parameter fails the call. Without an unattended permission mode the subagent prompts for every file edit and bash command and the pipeline stalls. No permission mode auto-answers `AskUserQuestion` — that tool always elicits the human. Therefore `/bmad-retrospective` is lead-only (spawned subagents cannot reliably surface their elicitation). Run unattended cycles only in a trusted/isolated environment.

Pipeline-stage subagents must be spawned **synchronously** — never backgrounded, and synchronous is a parameter, not an adjective: every `Agent` call passes `run_in_background: false` (Rule 18 (5)). On current Claude Code builds an `Agent` call that omits the flag runs in the background — it returns at once and its work continues unsupervised — so "spawn synchronously" in prose is not enough, at any depth. A backgrounded subagent never hands control back in an unattended run and the pipeline stalls (this is why `bmad-build-auto` mandates synchronous subagent calls). Parallel batches are still synchronous: N `Agent` calls in ONE message resolve together.

`bmad-build-auto` **requires** subagents of its own (it HALTs `blocked` / `no subagents` otherwise): the stage agent that runs it must be `subagent_type: "general-purpose"` (full tool set including `Agent`). Nested spawning to depth 3 (lead → build-auto stage agent → its handoff/review subagents) is verified on Claude Code (2026-08-26 spike).

**Launch environment (unattended runs).** Set these where the lead session is launched (the shell, or the user's `settings.json` `env` — a project's `.claude/settings.json` `env` applies only once the workspace is trusted). They are Claude Code settings as of 2.1.29x; check them against your version. The pre-flight reads them with `printenv` and logs `launch_env depth=<n|default> watchdog=<0|1> bash_max_ms=<n|default>`.

- `CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH` — nesting is limited to 3 layers below the main session by default. A sequential run needs 2 (stage → its handoff and review layers), a parallel run 3 (runner → stage → handoff and layers); `4` leaves headroom. At the limit an agent silently loses the `Agent` tool and does the work itself: `bmad-build-auto` then halts `no subagents`, and other skills run their review layers inline without saying so.
- `CLAUDE_CODE_RETRY_WATCHDOG=1` — retries rate-limit and overload errors (`429` / `529`) indefinitely with backoff instead of failing the stage. A `429` that reports a spend or usage limit still fails at once (see the API-error handling in "Agent Invocation Pattern").
- `BASH_MAX_TIMEOUT_MS` (with `BASH_DEFAULT_TIMEOUT_MS`) — raises the foreground cap of Rule 18 (5), default 10 minutes, for a project whose test batches need longer.
- `CLAUDE_CODE_MAX_CONCURRENT_SUBAGENTS` — 20 running subagents per session by default, nested ones included; over it an `Agent` call fails with `Concurrent subagent limit reached`. A parallel run's peak is about `max_parallel_epics` × (runner + stage + its review layers).
- Headless launches (`claude -p`) only: `CLAUDE_CODE_PRINT_BG_WAIT_CEILING_MS=0`, or the process stops waiting for background runners after 10 idle minutes.

**Supervised, not unattended.** "Unattended" describes the permission mode, not the loop. Measured on a greenfield epic (field report 2026-08-29: 8 stories, 9 clarifications, 7 user decisions, zero false halts) the pipeline needs roughly **one human round trip per story**, peaking where planning artifacts meet reality for the first time, plus one decision sheet at each epic's merge gate. Every one of those halts is load-bearing: a run launched by someone who expects to walk away will experience correct behaviour as a malfunction, and the tempting reactions — pre-authorizing everything, widening footprints blindly — are worse than the wait. The levers that reduce the rate without reducing the halts are Rule 5's apply-and-report tier (amendments that preserve intent are applied, then reviewed) and Rule 11's authority/concurrency split (parallel kit); the halts themselves are not tunable. **`--unattended` changes this for the development loop only:** setup still stops, while the owner is present; after it, the loop decides small questions, parks owner-only ones and keeps going, under the project's policy (`_bmad/custom/unattended.yaml`) and the doctrine. The halts become inbox items and veto-window rulings the owner reviews with `/epic-cycle --review`.

## Model Strategy (Critical for cost + quality)

The pipeline runs **efficient implementer + expensive reviewer**: a Sonnet-tier model writes code and tests; an Opus-tier model reviews them. This also satisfies BMAD's own guidance that code review should run in "fresh context, ideally different LLM" than the implementer.

Per-skill model pinning ("skill optimization" — writing `model:` into each SKILL.md's frontmatter) is an **optional, per-project** pass: upstream BMAD v6 ships no `model:` frontmatter, the field is not an officially documented SKILL.md field, and some projects deliberately skip the pass. `/epic-cycle` must behave identically either way, so the command carries the default stage→model map itself:

| Stage | Skill | Model | Why |
| --- | --- | --- | --- |
| Plan | `/bmad-build-auto` + `Halt after planning.` | `opus` | The story spec is the highest-leverage context-fusion step; its judgment lets everything downstream run cheaper. Its epic-context compile subagent inherits this tier. |
| Implement | `/bmad-build-auto` on the planned spec | `sonnet` | Near-Opus coding quality. Its implementation-handoff subagent and its self-review layers (`blind-hunter`, `edge-case-hunter`, `verification-gap`, `intent-alignment` by default) inherit this tier ("same model capability as the current session") — a cheap self-review, not the gate. A project may switch a self-review layer off with `instruction = ""` in its `bmad-build-auto` override (File 2's note: deleting the entry does nothing, and switching off `edge-case-hunter` drops the spec-claims check); the code-review stage is the one never to cut. |
| QA | `/bmad-qa-generate-e2e-tests` | `sonnet` | Bounded test implementation |
| Code review | `/bmad-code-review` | `opus` | The independent adversarial gate — BMAD's own template note: "Dev moves story to 'review', then runs code-review (fresh context, different LLM recommended)". Its review layers (`blind-hunter`, `edge-case-hunter`, `verification-gap`, `acceptance-auditor`) run at the model passed on this spawn because nested subagent spawns inherit the parent's model when no `model` parameter is given. Review MUST NOT run on a lighter model than the implementation. A project's `review_tier: mixed` (see the model-policy file below) re-tiers the three hunter layers only. |

On an **unpinned** project the map is the operative source. On a **pinned** project frontmatter resolves first and normally agrees with the map; if a pin disagrees, the pin wins — treat it as a deliberate project re-pin (never "correct" it), and the `model=` telemetry records what actually ran either way. The one skill that runs at two tiers is `bmad-build-auto`: a frontmatter pin on it documents the **plan** tier; the implement tier comes from the map or the policy file's `implement:` key, never from the frontmatter.

**Upstream levers (v6.11):** `bmad-build-auto`'s `implementation_handoff` and every skill's `review_layers[].instruction` are free-prose overrides in `_bmad/custom/<skill>.toml` — the sanctioned way to route one internal subagent to another model or an external tool. `/epic-cycle` does not need them (it passes `model` on the stage spawn), but a project that must run the implement stage inline (e.g. a runner without nested-spawn headroom) can put `model: sonnet` into the handoff prose there.

### Project model-policy file (`_bmad/custom/model-overrides.yaml`, optional — highest priority)

A project-local re-pin surface that survives BMAD upgrades (the installer never touches `_bmad/custom/`). When present, model resolution becomes: **overrides file → frontmatter → stage map → inherit**. Schema:

```yaml
# Project-local model policy. overrides win over SKILL.md frontmatter and the stage map.
stack_risk: uncommon            # uncommon | standard (default). uncommon arms the end-of-epic model-tier checkpoint.
review_tier: full-opus          # full-opus (default) | mixed — controls bmad-code-review's internal review layers
overrides:                      # optional: STAGE -> tier re-pins (plan | implement | qa | code-review)
  implement: opus               # bmad-build-auto implement stage (plan stays at its own key)
history:                        # append-only; every change records its evidence
  - date: 2026-07-19
    action: escalate-implement-to-opus
    evidence: "epic-2: high+med avg 2.5/story with implement=claude-sonnet-5; 2 rework loops (language semantics)"
    approved_by: user
```

Keys are **stages**, not skills, because `bmad-build-auto` serves two stages at two tiers. (Pre-6.11 policy files keyed `bmad-dev-story` / `bmad-quick-dev` / `bmad-qa-generate-e2e-tests`: read them as `implement` / `implement` / `qa` and offer to rewrite the file once.)

### Model-tier checkpoint (end of epic, lead-side)

**Evaluation is unconditional; application is gated.** The checkpoint evaluates on every project, immediately after `epic_status_done`, against this epic's cycle log, and always logs `model_tier_checkpoint` (a default project has no policy file, yet every `cr_complete` writes the telemetry the checkpoint reads — evaluating only on armed projects meant the numbers were written by everyone and read by no one; field report 2026-08-28: mean 4.75 high+med per story, 3 of 4 stories reworked, never evaluated). Applying a change requires `_bmad/custom/model-overrides.yaml` (create it on the user's approval if absent) plus the user's explicit approval; the mandatory mixed-tier rollback needs no approval but does need the file (it only exists if `mixed` was set). On a project without the file, a met threshold is surfaced with the evidence and the offer to create the file — `result=recommend_escalate armed=false`. (In Orchestrator Mode, the orchestrator runs it in the main checkout at the merge gate, after `epic_merged_to_feature`, against the epic's per-worktree cycle log.)

- **Escalation (uncommon stack):** if mean `cr_complete` **`unresolved_high_med`** ≥ 2 per story with the implement stage on a Sonnet-tier model — the high+med root-cause entries the reviewer could not resolve in its own pass, i.e. what escaped both the implementer and the reviewer's patches; NOT the gross `high+med` found, which a four-layer adversarial review scores at 7–10 on stories that need one pass (field report 2026-09-11: 8, 7 and 10 on three one-pass stories against a threshold of 2 — the metric did not discriminate) — OR ≥ 2 stories entered the rework loop for language-semantics defects, OR ≥ 2 stories' `dev_complete` show `review_loop_iteration ≥ 3` (build-auto's own bad-spec/patch loop thrashing) → recommend re-pinning `implement` / `qa` to `opus` for this project. Surface the evidence to the user; on approval, update `overrides:` + `history:`. Rationale: two rework iterations (extra Sonnet dev + extra Opus review) cost more than Opus-dev. Counts are root-cause entries after the reviewer's grouping (`rows=` is raw layer volume and never a threshold input); a `cr_complete` from before Kit-Version 2026-09-11.1 lacks `unresolved_high_med` — fall back to `high+med` for that story and log `metric=gross` on the checkpoint so the reading is known to be inflated.
- **De-escalation offer (review layers):** after ≥ 2 consecutive epics with zero high and ≤ 1 medium finding per epic from Opus-tier review, the lead MAY offer `review_tier: mixed` (`blind-hunter` / `edge-case-hunter` / `verification-gap` → sonnet; `acceptance-auditor` stays at the parent model; never below the implementation tier).
- **Rollback (mandatory, not user-gated):** if `review_tier: mixed` is active and any `smoke_complete` in this epic recorded `defects_caught>0`, revert to `full-opus` immediately and record it.

**Telemetry is mandatory:** every evaluation logs `model_tier_checkpoint`; every change to model-overrides.yaml logs `model_tier_changed` with `direction=`, `stages=`, `from=`, `to=`, `reason=`, and `evidence=` (see Workflow Telemetry). A tier change absent from the cycle log did not happen — never edit the policy file without the paired log entry and matching `history:` record.

Lead-run gate skills (sprint planning, retrospective) and lead-side gates (spec validation, AD verifications, smoke, merge-conflict decisions) execute inline on the lead's model — run the lead on an Opus-tier model so those judgment calls get maximum quality. Story planning is a spawned Opus stage, so it is tier-guaranteed independently of the lead.

## Skill Tool Invocation (Critical)

All BMAD skills must be invoked via the **`Skill` tool**, not interpreted inline. Agent spawn prompts must explicitly state: "use the `Skill` tool to invoke /bmad-build-auto" (or the relevant skill). Without this directive, agents may try to execute skill logic themselves. For `bmad-build-auto` this matters twice over: its SKILL.md is a bootstrap that runs `uv run render_skill.py` and then follows the rendered `workflow.md` — an agent that "reads the skill" instead of invoking it never renders and runs stale or unresolved workflow text.

## Agent Invocation Pattern (Required)

Each pipeline-stage subagent is a single `Agent` tool call. The `Agent` tool returns the agent's final assistant message as its result — that is the completion signal. No separate envelope, no team membership, no shutdown handshake.

For each pipeline stage, the lead:

1. **Resolves the stage's model.** Resolution order: (a) if `_bmad/custom/model-overrides.yaml` exists and its `overrides:` map names the **stage** (`plan` | `implement` | `qa` | `code-review`), that wins unconditionally (project model policy — see Model Strategy); (b) else if the stage skill's YAML frontmatter — `.claude/skills/<skill-name>/SKILL.md`, or the plugin-namespaced install path — declares a `model:` field (e.g. `opus` / `sonnet` / `haiku`), that wins for every stage except `implement` (a `bmad-build-auto` pin documents the plan tier only); (c) otherwise use the Model Strategy map above (plan → `opus`, implement → `sonnet`, qa → `sonnet`, code-review → `opus`); (d) for a stage not in the map, omit the parameter and let the sub-agent inherit the lead's model. Read the actual file and frontmatter — do not assume. On a vanilla BMAD install (c) is the operative source (upstream v6.11 still ships no `model:` frontmatter); projects that ran a model-pinning pass over their skills resolve via (b), and the pins normally agree with the map.
2. **Writes the `stage_spawned` entry** (`stage=<plan|implement|qa|code-review> spawn_at=<UTC> model=<resolved> agent_name=<name passed on the Agent call> cycle_iteration=N`; a story's first implement spawn — and, under `--unattended`, its first after a set-aside or a re-plan — also carries `story_base=<HEAD sha>`) to the cycle log IMMEDIATELY before the call — never after. Then, if the stage is `bmad-build-auto`, **ensures the tree is clean** (Rule 16): every pending bookkeeping write including this entry is committed first; `git status --short` must be empty in the stage's working directory (and in every mounted submodule) — the one exception is the review-step intent-gap patch re-dispatch (re-dispatch protocol, step 2), whose applied patch stays uncommitted by design. An open `stage_spawned` (no closing success entry, no `<stage>_clarification_requested`) is the only durable evidence that a stage may be in flight — resume reads it before any re-dispatch.
3. **Spawns** the subagent via `Agent` with `name: "<story-key>-<stage>-<iteration>"` (the `agent_name` recorded above — pass `name` only when the tool's schema offers it; the log entry identifies the stage either way) and:
   - `subagent_type: "general-purpose"` (required for `bmad-build-auto` stages — they spawn their own subagents)
   - `model: <the model resolved in step 1>` — pass this so the stage runs on the model the pipeline intends, NOT the lead's inherited model. Omit the parameter only when step 1 resolved nothing (stage not in the map and no frontmatter declaration).
   - `mode: "bypassPermissions"` — only when the `Agent` schema offers `mode` (see Permission Mode); omit it where the schema has none
   - `run_in_background: false` — always explicit, never omitted (Rule 18 (5)): the harness default may be background, and a backgrounded stage returns at once while its work runs on with no supervisor
   - `description: <3-5 word task description>`
   - `prompt: <full task — see Spawn Prompt Skeleton below>`
4. **Reads the result.** For `bmad-build-auto` stages: re-read the spec file from disk — frontmatter `status` (`ready-for-dev` after a plan spawn; `done` after an implement spawn; `blocked` otherwise) and the `## Auto Run Result` section (`Status:` / `Blocking condition:`); the agent's prose is advisory. For QA/CR stages: the returned message's closing-summary sections (`## Files Modified`, `## Tests Added`, `## Decisions`, `## Issues Encountered`).
   **A harness error is not a stage result.** If the `Agent` result is an error from the harness rather than the stage's report — `Agent terminated early due to an API error: …`, or a rate-limit, overload, or usage-limit message — the stage did not finish. Log `stage_api_error stage=<stage> error=<rate_limit|overload|usage_limit|other>`; wait (rate limit or overload: about 5, then 10, then 20 minutes; a spend or usage limit: STOP and surface it to the user — waiting will not clear it; *Unattended:* hold, U11); then re-spawn the stage per the **Re-dispatch protocol** ("Clarification protocol"): its open `stage_spawned` entry stands, a `bmad-build-auto` stage resumes from the spec's `status`, and the tree is checked first — a stage cut off mid-edit can leave changes behind, which Rule 18 (4)'s quiescence samples and the Rule 16 clean-tree gate catch. It is never a failed story and never a rework iteration. (Field report 2026-10-10: at least seven runner kills from rate limits or quota in one month-long run.)
5. **Falls back** to `git diff --name-only <baseline_revision>..HEAD` + `git status --short` if closing sections are missing.
6. **Records** the stage in the cycle log, setting `model=<the model the sub-agent actually ran on>` (the resolved stage model, or the inherited lead model when none was declared) per the telemetry spec.
7. **Proceeds** to the next stage. No shutdown step.

**Lead-run gate skills run on the lead's model.** `/bmad-sprint-planning` and `/bmad-retrospective` are invoked by the lead directly via the `Skill` tool (not spawned as sub-agents), so they execute in the lead's context and their frontmatter `model:` is NOT applied — there is no per-skill model switch for inline lead execution. This is by design (these gates must stay lead-side; see "Sprint Planning Per Epic" and "Retrospective Per Epic"). Honoring `model:` therefore applies to the spawned pipeline stages (plan / implement / qa / code-review) only.

### Spawn Prompt Skeleton

Every Agent spawn prompt must include, in this order:

1. The literal marker `**Epic Cycle Stage: <plan|implement|qa|code-review> for Story <id>**`.
2. The dispatch anchor: for **plan**, the sprint-status story key (e.g. `3-2-digest-delivery`) — never a spec path, never a spec folder; for **implement / qa / code-review**, the spec file path captured at `story_created`.
3. The list of files modified by upstream stages (for QA: `git diff --name-only <baseline_revision>..HEAD`; for code review: that list plus QA's `## Tests Added`).
4. The architecture spine path from `spine_resolved` (`<planning>/architecture/<run-folder>/ARCHITECTURE-SPINE.md`, or "none") as factual context — its `AD-n` blocks are the project's architecture decisions (Rule 6).
5. The directive: `Use the Skill tool to invoke /<bmad-skill-name>.` For `bmad-build-auto` the skill's own argument is the dispatch anchor from item 2 — for the plan stage append the exact sentence `Halt after planning.` to the argument.
6. The stage-specific rule block (see below) — for QA/CR including its **pre-answered checkpoints**: every interactive menu/halt the skill will hit, with the chosen option, so the agent never waits on input that cannot arrive (Rule 9). `bmad-build-auto` has no checkpoints; its block carries rules only.
7. The completion-contract directive — for QA/CR quote the closing-summary section names inline; for `bmad-build-auto` state that the spec file's frontmatter `status` + `## Auto Run Result` is the contract and that the agent must not push.
8. Skill-specific context.

**Unattended runs:** after these items, append the `## Unattended run` block the doctrine renders from the policy (U6); the policy's `stage_preamble`, when set, heads the prompt instead.

### Stage-specific rule blocks (copy into spawn prompts)

**Plan spawn (`bmad-build-auto`, Opus tier) — append:**

```text
Dispatch: invoke /bmad-build-auto via the Skill tool with the argument `<story-key> Halt after planning.` — the sprint-status story key (e.g. `3-2-digest-delivery`) followed by that exact sentence. This is the skill's epic-story path: it resolves the epic/story numbers from the key, reuses the committed `<impl>/epic-<N>-context.md` (or would compile it via its own subagent — the lead pre-warms it so that never happens), loads the previous `done` story of this epic for continuity, plans the spec, verifies it against its READY-FOR-DEVELOPMENT standard, sets frontmatter `status: ready-for-dev`, and HALTs. Plan ONLY this story. Do NOT supply a spec folder or stories.yaml (folder+id dispatch is for bmad-spec projects, not this pipeline). Do NOT implement anything.

Rules for this stage (from skill-rules.md):

- Rule 1/2 (Integration ACs, Consumed-by/Consumes): if this story introduces a service, module, or shared component, the spec MUST carry at least one Integration AC (an I/O & Edge-Case Matrix row or a Tasks & Acceptance item of the form "consumer X reads from this and produces observable effect Y") — or an explicit "No consumers in this story; the first consumer will be Story X.Y." note under ## Design Notes — plus `Consumed-by:` / `Consumes:` lists under ## Design Notes.
- Rule 5 (NFR tripwire): an NFR that is unmeasurable, contradictory, or impossible as worded is an `intent gap` HALT that names the NFR and your recommended amendment — never plan around it.
- Rule 6 (architecture decisions): consult the spine at the path above and record the `AD-n` ids that govern this story under ## Design Notes (create the section if the template would omit it), so the implement stage and the reviewer can check against each AD's Rule. If an AC contradicts an AD's Rule, that is an `intent gap` HALT naming the AD — never plan around it.
- Rule 13 (working directory): operate from the absolute working directory stated above; verify `git rev-parse --show-toplevel` matches before reading planning artifacts; pass the same path + verification into the epic-context compile subagent. Rule 13 also covers runtime targets (name every one explicitly — never a tool's default target — and tear down only what your session started) and reading (each document once; re-read only what may have changed, by line range) — pass both into every subagent prompt with the path.
- Rule 18 (5) and the verification arrangement (`## Verification`): mark every command `(loop)` — the story's own tests, run on every verification pass — or `(once)` — the full suite, run once before the implement stage finalizes. Follow the project's verification arrangement if its rules record one ("Per-Story Smoke"): a tier left to CI is omitted while CI is available; a lead-owned full run means no `(once)` lines at all. Every command must finish inside one foreground call (10 minutes with `timeout: 600000`, unless the launch environment raises the cap) — the implement stage may not background it — so write a long suite as several commands (one test class per command is always safe; batches sized from the project's recorded timings, if it keeps any, to about 8 minutes), never as one run-everything line.
- Rule 19 (hermeticity): add a `Shared surfaces:` line under `## Verification` naming every list, menu, roster, schema, or count this story changes that other tests assert on (or `none`), and the standing criterion "every test added or changed passes on a fresh environment, in either order".
- Rule 2 (extend, don't copy): if the story adds a class, method, function, or helper that resembles an existing sibling, name the sibling it extends under ## Design Notes, or record why a copy is the better choice.
- Rule 17 (ledger inbox): the entries this story already owns are listed below (`LEDGER slice <key>`; the ones routed early also appear as `DW-n:` bullets in the story's acceptance criteria). For EACH: either address it — a Tasks & Acceptance item or I/O matrix row that cites `DW-n` — or decline it under ## Design Notes as `Declined DW-n: <one-line reason>` (out of scope for this story's intent, superseded, belongs elsewhere). Never ignore one; an entry you neither address nor decline fails the lead's spec gate.

Ledger inbox for <story-key>:
<paste of `LEDGER slice <story-key>`, or "(none)">

Completion contract: the spec file (its path is your final report's first line, as `SPEC: <path>`) with frontmatter `status: ready-for-dev` and a `## Auto Run Result` section that contains the two lines `Status: <final status>` and `Blocking condition: <condition or none>` verbatim (the skill's HALT protocol spells them out only for the skeleton case — emit them for the existing-spec case too). If you HALT with `status: blocked`, report the blocking condition verbatim. A valid, committed `epic-<N>-context.md` already exists — reuse it; do not recompile. 🚫 Do NOT `git commit` or `git push` — the plan stage leaves the spec uncommitted for the lead's validation gate. **Propagate this prohibition verbatim into every subagent you spawn** (investigation subagents, the epic-context compile subagent if it ever runs), in the same breath as the working directory (Rule 13): they may read, and write only the files the skill names; they may NOT run `git commit`, `git push`, `git reset`, `git rebase`, or anything that mutates a remote or triggers CI; they may NOT `SendMessage` any agent, and you may NOT `SendMessage` them once they have returned (Rule 18 — spawn a fresh one instead). Rule 18 (5): every `Agent` call you make — investigation subagents, the compile subagent — passes `run_in_background: false` (several in ONE message still run in parallel); run every command in the foreground of its own tool call; never wait on `Monitor` or end your turn while anything you started is still running. **Return exactly once**, when the spec is written and the skill has HALTed — never an interim message.
```

**Implement spawn (`bmad-build-auto`, Sonnet tier) — append:**

```text
Dispatch: invoke /bmad-build-auto via the Skill tool with the argument = the spec file path above (frontmatter status `ready-for-dev`). The skill resumes at its implement step: it captures `baseline_revision`, runs its implementation-handoff subagent against the spec, runs the spec's `## Verification` commands and the Matrix Test Audit (acceptance criteria are judged at its review step), runs its review layers in parallel, triages (`patch` fixes inline; `defer` goes to the spec frontmatter `deferred:` list; `intent_gap` HALTs), then finalizes: `status: done`, followed by ONE local commit of the reviewed diff plus the spec. The tree is clean on entry (the lead verified it) and the skill verifies it is clean on exit.

Rules for this stage (from skill-rules.md):

- Rule 5 (NFR tripwire): an un-implementable NFR is an `intent gap` HALT with your recommended amendment; do NOT work around with code comments + deferred-work.md.
- Rule 6 (architecture decisions): match the implementation to the `AD-n` Rules recorded under the spec's ## Design Notes and any other AD whose `Binds:` covers this story's area (spine path above). An AD conflict you cannot resolve within the spec is an `intent gap`, not a judgment call.
- Rule 13 (working directory): operate from the absolute working directory stated above; verify `git rev-parse --show-toplevel` matches before editing anything, and pass the same path + verification requirement into the handoff subagent and every review-layer subagent. Rule 13 also covers runtime targets (name every one explicitly — never a tool's default target — and tear down only what your session started) and reading (each document once; re-read only what may have changed, by line range) — pass both into every subagent prompt with the path.
- Rule 14 (source text): author non-ASCII characters in source via escape sequences (`'…'`), never literal bytes; comments state the contract, never review history (that belongs in the spec).
- Rule 2 (extend, don't copy): prefer extending the sibling the spec names; a copy carries its reason in the spec or the commit message, never in a code comment.
- Rule 15 (ledger): record deferred findings ONLY in the spec's frontmatter `deferred:` list exactly as the skill specifies — the lead harvests them into deferred-work.md; do not write the ledger yourself.
- Rule 19 (falsifiability, at the source): when you or your handoff subagent add or materially change an acceptance criterion's pinning test, write or update its `mutation:` line in the spec's `## Verification` in the same pass — name the mutation, apply it, observe red, revert (tree byte-identical). A test that exists with a missing or stale line is a LOW your own `verification-gap` layer will flag: close it in-pass by demonstrating the mutation and writing the line — never `defer` it. An AC with no pinning test, or an assertion that cannot fail, is a MED to patch.
- Rule 16 (clean tree): your finalize commit lands on the current branch (the epic branch); commit exactly what the skill's finalize step specifies and nothing else. 🚫 NEVER `git push`. **Propagate this prohibition verbatim into the handoff subagent and every review-layer subagent you spawn, in the same breath as the working directory (Rule 13): they may edit files, and they may NOT run `git commit`, `git push`, `git reset`, `git rebase`, or any command that mutates a remote or triggers CI or a deploy. Only you — the layer that spawned them — commit, and only at finalize.** A subagent that reports having committed or pushed is a protocol violation: verify that `git rev-parse HEAD` still equals the sha you recorded before spawning it, and check `gh run list`, rather than trusting the self-report, do not force-push or roll back anything, and HALT with status `blocked` and blocking condition `subagent protocol violation: <what it did, shas>` so the lead decides.

- Rule 18 (containment) — pre-answered override of the skill's step-04 `patch` handling: the step-03 handoff subagent can NEVER be "re-engaged" (`SendMessage` is not synchronous on this harness; a message to a returned agent creates an orphan writer) — apply patch findings yourself, or spawn a FRESH subagent with the findings written into the spec's `## Tasks & Acceptance`. Never `SendMessage` any agent. While a subagent of yours is running you do not edit the tree (two writers in one worktree produce false protocol alarms); your own test authoring (Matrix Test Audit) happens after it returns. **Stateful tests against a shared runtime run one test class or suite per tool call** — wait for each to complete, never several test-runner calls in one message, never a re-submit on a client-side timeout; propagate this into the handoff subagent in the same breath as the git prohibitions (Rule 18 (3)). **Synchronous is a parameter (Rule 18 (5)):** the handoff subagent and every review layer are launched with `run_in_background: false` on the `Agent` call (the layers together in ONE message still run in parallel) — the skill's own "synchronously" does not set it, and an omitted flag backgrounds the call on current harnesses. Where the skill runs the spec's `## Verification` commands, run the `(loop)` lines on every verification pass and the `(once)` lines exactly once, after the last patch and before the finalize commit. Every command, `## Verification` run, and test sweep runs in the foreground of its own tool call with `timeout: 600000` (the 10-minute cap unless the launch environment raises it; the default is 2 minutes, and a call that reaches its timeout is moved to the background, where it dies when the agent that started it returns) — never `run_in_background: true` on a shell call; a sweep that would outlast the cap is split into foreground calls that each finish inside it (one class per call is always safe; batches sized from the project's recorded timings, if any, to about 8 minutes), never detached. Never wait on `Monitor` — a subagent that arms it and stops has returned; never end your turn while anything you started is still running. Propagate all of this into the handoff subagent too. Return exactly once.

Completion contract: the spec file's frontmatter `status` (`done` on success; `blocked` + blocking condition otherwise) and its `## Auto Run Result` section, which must contain the two lines `Status: <final status>` and `Blocking condition: <condition or none>` verbatim. **Return exactly once, when your work is complete** — an interim or progress return while any subagent of yours is still working strands it as an orphan writer with a dead supervisor. On `blocked`, HALT and report — do not improvise a fix by re-running steps outside the skill.
```

**Implement re-spawn on rework (Rework Loop step 2) — additionally append, after the block above:**

```text
Rework re-dispatch: this spec is `in-progress` because the lead re-opened it after code review (rework iteration <k> of 3; the loop's exit is a re-review that finds no HIGH). Pre-answered overrides: (a) step-01 routes `in-progress` → implement: work ONLY the unchecked `[Review]` / `[Smoke]` / `[CI]` items under ## Tasks & Acceptance and the tests they name — do not re-derive, widen, or re-verify the rest of the story; (b) step-04's follow-up review recommendation is evaluated as a FOLLOW-UP pass (`followup_pass` = `true`): `true` only if this pass patched a `high`, otherwise the work has converged and patch volume is never grounds; (c) Rule 19: every pinning test you touch gets its `mutation:` line written or updated in this pass; (d) Rule 14: this pass's `## Auto Run Result` covers only this pass, and if the spec carries the `oversized` warning, add nothing to it beyond what the open items require.
```

**QA spawn — append:**

```text
Pre-answered checkpoint (Rule 9): the skill's first step asks what to test — answer: the deliverable of Story <id>, i.e. the files listed above (the diff since `<baseline_revision>`), whose acceptance criteria are the spec's Tasks & Acceptance and I/O & Edge-Case Matrix at <spec path>. Do not ask; do not broaden scope to unrelated features. If the project has no test framework yet, choose the stack-appropriate default, record the choice in ## Decisions, and proceed.

Rules for this stage (from skill-rules.md):

- Rule 8 (test discoverability): generated tests MUST be discoverable by the project's default test suite — (a) correct naming convention, (b) not excluded by ignore files, (c) not tagged in a way that opts them out of the default run.
- Rule 19 (falsifiability): for each acceptance criterion, name the mutation that would make its pinning test fail, apply it, confirm red, revert, and confirm the tree is byte-identical to before the mutation (`git status --short` / `git diff --stat` unchanged — a leaked mutation is a defect you shipped). Record it next to the test in the spec's `## Verification` as `mutation: <what> → <test that went red>` (create that section if the planner omitted it — the spec template deletes it for trivial stories). One mutation per AC, not per assertion. A test whose red you have not observed is not evidence; an assertion that cannot fail (expected value forced by the type, subject never constructed, two empty runs compared) is a defect to fix, not a test to count. Report `mutations_demonstrated=<N>` in ## Decisions.
- File-list completeness: append every test file you create to the spec's `## Verification` section (marked `(QA)`), so the spec remains the complete record of the story's files — not just your closing summary.
- Rule 19 (hermeticity): every test you write creates and removes the state it asserts on, depends on no other test's leftovers or run order, and waits for the system under test to signal readiness before acting.
- Rule 13 (targets and reading): name every runtime target explicitly — never a tool's default target — and tear down only what your session started; read each document once and work from your copy.

🚫 Do NOT `git commit` or `git push`. Leave ALL changes uncommitted in the working tree — the lead commits after the per-story smoke gate. Propagate this prohibition verbatim into any subagent you spawn (`git commit` / `git push` / `git reset` / `git rebase` / anything that mutates a remote or triggers CI are all forbidden for them too), in the same breath as the working directory (Rule 13); verify a subagent's self-report by checking that `git rev-parse HEAD` still equals the sha you recorded before spawning it, before relying on it. Rule 18: never `SendMessage` any agent; never message a subagent after it has returned (spawn a fresh one); every `Agent` call you make passes `run_in_background: false`, and every test run is in the foreground of its own tool call with `timeout: 600000`, split into calls that each finish inside the 10-minute cap — never `Monitor`, never end your turn with anything you started still running (Rule 18 (5)); **return exactly once**, when your tests are written and run — never an interim message.

End your final message with these sections, in order:

## Files Modified
- <full path from repo root>
(or "(none)")

## Tests Added
- <full path from repo root>
(or "(none)")

## Decisions
- <one-line summary>
(or "(none)")

## Issues Encountered
- <one-line summary>
(or "(none)")

If you cannot make confident progress for ANY reason — ambiguous ACs, missing prerequisite, user-preference choice, environment failure, or anything risking a stated constraint — STOP before the closing summary and end with a "## Clarification Needed" section instead (Rule 4): the question, what was tried, what is blocking. Do not guess; do not soldier on.
```

**Code-review spawn — append:**

```text
Pre-answered checkpoints (Rule 9 — the skill halts at each of these; take the answer below and continue, do not wait for a human):

1. Step-01 target + context: the explicit argument is the spec file at <spec path> (Tier 1 — this sets `{review_mode}` = "full" and arms the acceptance-auditor layer; never declare "no spec"). Its frontmatter carries `baseline_commit: <sha>` (the lead mirrored build-auto's `baseline_revision` there) — use it as the diff baseline exactly as step-01 describes; do NOT fall through to Tier 3/4. Diff source, in the skill's own terms: commit range `<baseline_revision>..HEAD` plus uncommitted working-tree changes plus untracked files — write `{diff_file}` (step-01's uniquely-named temp file) as `git diff <baseline_revision>` and append, for each untracked file, `git diff --no-index /dev/null <path>`. The layers read that file; so do you — once, then by section (Rule 13). Set {story_key} = `<story-key>` explicitly (the skill only discovers it on its own sprint-status scan, which Tier 1 skips) so its step-04 sprint-status sync works. Step-01's summary confirmation ("HALT and wait for user confirmation to proceed") → confirmed.
1b. Large-diff offer (step-01: "if `wc -l {diff_file}` exceeds approximately 3000 lines, warn the user and offer to chunk the review") → decline; review in full.
1c. Spec layout: this is a `bmad-build-auto` spec. Its `## Tasks & Acceptance` section IS the "Tasks/Subtasks section" — append your `### Review Findings` subsection there. There is NO "story file Status section": the spec's frontmatter `status:` is build-auto's machine state and you must not change it — the skill's "update the story file Status" step is satisfied by the sprint-status sync in 1e and by your closing summary.
1d. SKILL.md's "if you need an explicit user instruction to run subagents, ask once now" → you have it: run every review layer as a subagent.
1e. Sprint-status sync (step-04 §6 "load the FULL file … save preserving comments") → do NOT rewrite the YAML by hand. Run `uv run --no-cache <path-to>/bmad-sprint-planning/scripts/sprint_plan.py generate --epic-file <planning>/epics.md --status-file <impl>/sprint-status.yaml --stories-dir <impl> --project <project_name> --date "<MM-DD-YYYY HH:MM>" --set <story-key>=<done|in-progress>` (the lead has filled every placeholder in this prompt) — it is the only writer that preserves the file's shape.
2. Decision-needed findings → resolve each with best judgment against skill-rules.md and the architecture spine's ADs. Only a genuine user-preference or spec-contradiction question becomes ## Clarification Needed. If you resolve one as `defer`, the skill's "quick one-line reason for deferring?" prompt is answered by the Rule 15 disposition and note you file (below) — do not wait.
3. Patch-handling menu → option 1 (apply every patch).
4. Final next-steps menu → option 3 (done). Do NOT start the next story.

Rules for this stage (from skill-rules.md):

- Rule 3 (real-runtime test evidence): user-facing surface approved without a real-runtime test in the QA suite = high finding. (Distinct from the lead's manual per-story smoke, which is a separate later gate.)
- Rule 5 (NFR tripwire): unmeasurable NFR worked around with code comments + deferred-work.md instead of planning-artifact amendment = high finding.
- Rule 6 (AD violations): for each AC constrained by an `AD-n` in the spec's ## Design Notes — and any AD in the spine whose `Binds:` covers this story — verify the implementation matches the AD's Rule. Mismatch = high (not a low deferrable).
- Rule 1 (Integration ACs): service-introducing story missing an Integration AC = high finding.
- Rule 2 (extend, don't copy): new code that is a near-copy of an existing sibling with no stated reason is a MED naming both locations (patch it by extending the sibling when fix-risk allows); a weak reason is a LOW; a copy the spec forces is `routed` to the story that will unify it, never `by-design`. Check this yourself, with the repository in view — the context-free layers cannot.
- Rule 19 (hermeticity): an added or changed test that is not self-contained — it relies on state it did not create, on another test's leftovers or run order, or acts before the system signals readiness — is a MED in-story finding; so is an entry on the spec's `Shared surfaces:` line whose old-shape assertions the implementation left unchanged.
- Rule 15 (finding disposition bar): whatever the skill's own triage buckets are called (decision-needed / patch / defer / dismiss today; a five-way verdict with `intent_gap` / `bad_spec` / `reject` routes on upstream `main`), Rule 15's table governs the disposition of every finding that survives triage. Disposition every finding AT REVIEW TIME per the Rule 15 lookup, emitting fix-risk (low/med/high + one-line justification) on each. "In-footprint" means the EPIC's footprint (in-story or in-epic) — a file this story did not touch but this epic owns is in-footprint — but fix-now applies only in-story; an in-epic MED is routed to the story that owns the file (below). Spec-bound → close `by-design` at emission; theoretical (no realistic user-reachable failure) → close `wontfix-theoretical` + one line naming what would make it real; MED with fix-risk ≤ med, in-story → patch it NOW when you can (you may edit files; the patch menu applies every patch); what you cannot patch leaves the story `in-progress` for ONE rework iteration on the story's FIRST review only — on a re-review (this prompt then carries a "Rework re-review" block) it is `routed owner=<a later story key in this epic that touches the area, else burndown>` and never blocks `done`; MED with fix-risk ≤ med, in-epic (a file a story of this epic owns, not this one) → `routed owner=<that story's key if it has not reached cr_complete, else burndown>`, non-blocking; LOW two-way door (≈≤15 changed lines incl. its test, in-footprint, spec-clear) → patch now if mechanical, else return it in a `## Fix Pack` section for one bounded dev iteration; LOW that is not a two-way door → `wontfix-accepted` with `reopen_if=<observable probe>` at emission — `burndown` never owns a LOW; MED with high fix-risk or out-of-footprint → `escalated` (decided by the user at the merge-gate decision sheet, non-blocking); real MED/HIGH out-of-footprint → `routed` canonical entry — CONSULT deferred-work.md FIRST and append an occurrence instead of re-filing an adjudicated item; genuine product call → `decision-pending`; closable only by a human → `decision-pending` with `note=human=<what>`. The skill's own `defer` bucket ("pre-existing issue not caused by the current change") is subordinate to this table: a pre-existing in-story MED is still patched now or takes the one MED iteration, a pre-existing in-epic MED is still routed to a named owner, and a pre-existing LOW is still fixed or closed with a probe — "pre-existing" is a fact about blame, not a disposition.
- Rule 19 (falsifiability): an acceptance criterion with NO pinning test, or a pinning assertion that cannot fail (see the `verification-gap` layer's brief), is a MED in-story finding with low fix-risk → patch it now, or it takes the story's one MED iteration. A pinning test that EXISTS but has no `mutation:` line in the spec's `## Verification`, or a stale one (a rework changed the test), is a LOW you close inside this pass: name the mutation, apply it, observe red, revert (tree byte-identical), and write or correct the line under `## Verification` — that section is a tracking doc you MAY edit; this is the one exemption from the skill's "reject any finding whose fix is to edit the spec under review" rule, and it never re-opens the story. Name the AC in either case.

Review-layer model policy: read `_bmad/custom/model-overrides.yaml` (relative to the working directory stated in this prompt). If it sets `review_tier: mixed`, launch the defect-discovery layers — `blind-hunter`, `edge-case-hunter`, `verification-gap` (by `id` in the skill's review_layers) — with an explicit `model: sonnet` parameter on their Agent calls, and launch `acceptance-auditor` with no model override so it inherits your model. If the file is absent or says `review_tier: full-opus`, launch every layer with no model override (they inherit your model). Never launch any review layer on a lighter tier than the model that implemented the story. State in ## Decisions which review tier was applied and which layers ran.

Every ledger write follows Rule 15's grammar and goes through the tool — pre-answered override of the skill's step-04 "append to deferred-work.md": do NOT write the skill's default `## Deferred from:` bullets. Instead, for each `defer` finding: first `bash _bmad/scripts/ledger.sh <ledger> slice all` and grep the summaries — an existing root cause gets `append DW-n "occurrence=<story-key>"`, never a new entry; otherwise `new "<summary>" "<spec-basename>" <severity> <fix-risk> <footprint> "<evidence, one line>" <status> <owner> cr "<note>"` where status/owner follow Rule 15's disposition table: `escalated owner=burndown` (MED, high fix-risk or out-of-footprint — "for the user at the decision sheet"), `routed owner=<story key in this epic that will touch it, else burndown>` (real MED/HIGH out-of-footprint; an in-epic MED, owner = the story that owns the file; an unpatched in-story MED on a re-review — never a LOW, and never an unpatched in-story MED on a first review, which re-opens the story instead), `decision-pending owner=burndown` (genuine product call, or `note=human=<what>` for work only a human can do), `wontfix-accepted` with `reopen_if=` (a LOW that is not a two-way door), `by-design` / `wontfix-theoretical` (terminal — owner is still required; use the story key). Entries born from this review carry `by=cr`. Never open or rewrite the ledger file directly; never write `owner=none`.

🚫 Do NOT `git commit` or `git push`. You MAY edit files to apply patches and update tracking docs (the spec's `### Review Findings`, deferred-work.md, sprint-status.yaml), but the lead commits everything (submodules-first) after the per-story smoke gate. **Propagate this prohibition verbatim into every review-layer subagent you launch (`blind-hunter`, `edge-case-hunter`, `verification-gap`, `acceptance-auditor`), in the same breath as the working directory (Rule 13): they read and report; they may NOT edit files, `git commit`, `git push`, `git reset`, `git rebase`, or run anything that mutates a remote or triggers CI.** A layer that reports having changed or committed anything is a protocol violation — verify with `git status --short` and by checking that `git rev-parse HEAD` still equals the sha you recorded before launching it, discard that layer's findings, and note it in `## Issues Encountered`. Rule 18: never `SendMessage` any agent; a returned layer is finished (re-launch, never resume); launch every review layer with `run_in_background: false` on its `Agent` call — all of them in ONE message, so they still run in parallel — because an omitted flag backgrounds the layer on current harnesses and the skill's "synchronously" does not set it (Rule 18 (5)); run any command in the foreground with `timeout: 600000`, split into calls that fit the 10-minute cap, never wait on `Monitor`; **return exactly once**, after the skill's final menu — never an interim message. Do NOT change the spec's frontmatter `status:` (it is bmad-build-auto's machine state; the lead re-opens the spec when rework is needed) — record your verdict in sprint-status.yaml and in your closing summary only. Do NOT write cycle-log entries — that is the lead's job.

The skill sets the story's final status itself: `done` when all decision-needed + patch findings are resolved and no high/medium remains unresolved (mediums dispositioned `escalated` or `routed` per Rule 15 are ledgered with an owner, not blocking); otherwise back to `in-progress`. Exit condition (Rule 15): on a re-review that finds no HIGH, every remaining MED is `routed` / `escalated` with an owner and the status is `done` — a reworked story is never left `in-progress` on MEDs alone. Do not override that decision.

End your final message with these sections, in order:

## Files Modified
- <full path from repo root>
(or "(none)")

## Tests Added
- <full path from repo root>
(or "(none)")

## Decisions
- final story status (`done` or `in-progress`), resolved/deferred/dismissed counts, any high-severity fix names
- `entries: high=N med=N low=N rows=N unresolved_high_med=N` — `high`/`med`/`low` count root-cause ENTRIES after step-03 grouping, `rows` the raw findings the layers reported before grouping, `unresolved_high_med` the high+med entries you left unresolved after your own patches (routed, escalated, or re-opening the story). The lead copies this line into `cr_complete`; a row count in the severity fields is a telemetry bug (field report 2026-09-11: duplicates across four layers roughly doubled the checkpoint's reading).
(or "(none)")

## Issues Encountered
- <one-line summary>
(or "(none)")

If you cannot make confident progress for ANY reason — a genuine user-preference or spec-contradiction question, a missing prerequisite, an environment failure — STOP before the closing summary and end with a "## Clarification Needed" section instead (Rule 4): the question, what was tried, what is blocking. Do not guess; do not soldier on.
```

**Code-review re-spawn on rework (Rework Loop step 4) — additionally append, after the block above:**

```text
Rework re-review (iteration <k> of 3): the spec's `baseline_commit` is the REWORK's baseline (`<rework baseline sha>`, the commit build-auto captured after the rework commit), so the diff you review is `<rework baseline>..HEAD` plus the working tree — the rework, not the whole story. The whole story was reviewed at round 1 against `<original baseline sha>`; read wider when a finding needs context, but file a finding outside the rework range only when it is HIGH. Checklist — the previous round's unresolved items, each to be confirmed fixed or reported still open: <list>. Exit condition (Rule 15): if this review finds no HIGH, the story closes — patch what you can, file every remaining MED with an owner (`routed` / `escalated`), set `done`; do not leave it `in-progress` on MEDs alone. A HIGH re-enters the loop (the lead enforces the cap).
```

### Clarification protocol

*Unattended:* the lead classifies and rules or parks instead of surfacing (doctrine U5); the re-spawn mechanics below are unchanged.

If an agent returns with a `## Clarification Needed` section instead of the closing summary, the lead:

1. Reads the question from the returned message.
2. Surfaces it to the user with Story ID + context.
3. Waits for the user's answer.
4. Re-spawns the same stage's agent with the clarification baked into the prompt or written to the spec's `## Design Notes` (committed first — Rule 16). If the answer has architectural weight (Rule 20 — a seam, a convention, a dependency or licence, a numeric invariant), it is written into the spine in that same commit and the AD id is cited where the answer is recorded.
5. Logs `<stage>_clarification_requested` before the re-spawn; the re-spawn's normal success entry (`story_created` / `dev_complete` / `qa_complete` / `cr_complete`) closes it.

**`bmad-build-auto` results map onto this protocol by blocking condition** (read from the spec's `## Auto Run Result`; the plan/implement stages emit no `## Clarification Needed` section):

*Unattended:* each "surface to the user" in this table is classified instead (doctrine U5): non-convergence is the rework cap, an intent gap is a clarification.

| Spec `status` / blocking condition | Lead action |
| --- | --- |
| `ready-for-dev` (plan spawn) | Success — log `story_created`, run the spec gates. |
| `done` (implement spawn) | Success — harvest `deferred:`, set `review`, log `dev_complete`. |
| `blocked`: `intent gap`, `unclear intent`, `missing previous-story continuity decision`, `matrix ambiguity`, `spec failed ready-for-development standard`, or an unresolved-questions list | Clarification path. Surface the condition + `## Auto Run Result` to the user with Story ID. On answer, follow the **re-dispatch protocol** below. Log `<stage>_clarification_requested` / `<stage>_complete`. |
| `blocked`: `version-control metadata not writable`, `finalization left repository dirty`, `no subagents`, `context compilation verification failed` | Pipeline/environment defect (a missed bookkeeping commit — Rule 16; a non-general-purpose spawn; a missing or unparseable `epics.md` / failed epic-context pre-warm). Halt loudly, fix the cause, re-dispatch per the protocol. Never "answer" these as clarifications. |
| `blocked`: **free-text** condition naming a dirty tree, untracked files, or a branch mismatch (step-01 has no canonical string for this — `version-control metadata not writable` fires only when `git add --refresh` itself fails) | The commonest Rule 16 miss: uncommitted cycle-log / tracker / spec writes, or a wrong checkout. Same handling as the row above — reconcile (`git status --short`; bookkeeping commit or `git checkout {TICKET}-epic{N}`), log `protocol_violation` only if an agent left the tree dirty, and re-dispatch per the protocol. |
| `blocked`: `review repair loop exceeded 5 iterations (non-convergence)`, `implementation verification failed`, `patch verification failed`, `matrix test audit failed`, `handoff conflicts with spec` | Treat as a failed rework cap: surface to the user with the spec's `## Review Triage Log`. The spec is wrong more often than the code. |
| `blocked`: `no epic spec found`, `no stories.yaml found`, `story id not found in stories.yaml`, `ambiguous story file match`, `unrecognized status in existing story file`, `missing spec_file before implementation`, `planned spec file disappeared before implementation` | A `/epic-cycle` dispatch bug (folder+id dispatch used in sprint mode; a spec path that does not exist or whose frontmatter was hand-mangled). Fix the anchor and re-dispatch per the protocol. |
| `blocked`: `blocked spec supplied`, `story already blocked` | The lead re-dispatched on a spec still at `status: blocked` — a kit bug, not a new clarification. Apply the re-dispatch protocol (reset the status first). |

**Re-dispatch protocol (any re-spawn after a `blocked` result).** build-auto routes strictly by the spec's frontmatter `status`: `draft` → plan, `ready-for-dev` / `in-progress` → implement, `in-review` → review, `done` → follow-up review pass, `blocked` → HALT `blocked spec supplied`. So a re-spawn on an unchanged `blocked` spec can never proceed, and a re-spawn with the **story key** while a spec already exists resumes it only if it is still `draft` — at any other status the skill creates `spec-<key>-2.md` and ignores the edited file. Therefore, before every re-dispatch:

1. Apply the answer to the spec. For a **plan-stage** halt the answer usually belongs inside the `<intent-contract>` block (build-auto preserves that block verbatim when it resumes a `draft`) and/or `## Design Notes`; for an **implement-stage** halt it belongs in the non-frozen sections (`## Design Notes`, `## Tasks & Acceptance`, `## Spec Change Log`). Note the change in `## Spec Change Log`. If the answer has architectural weight, write it into the spine first (Rule 20) and cite the AD id.
2. Reset the frontmatter `status`: `draft` to re-plan (the next run re-derives the spec around the preserved intent block and honours `Halt after planning.`); `in-progress` (or `ready-for-dev`) to re-implement. **Exception — an `intent gap` raised at build-auto's review step** (the spec's `## Review Triage Log` entry names a saved patch file under `<impl>`): the skill saved the attempted change as that patch and reverted the code, so do not re-implement — after applying the amendment, `git apply --whitespace=nowarn <patch>` (byte for byte, whatever `apply.whitespace` says), verify it applied cleanly, set `status: in-review`, and the re-spawn resumes at review with the work intact (the 2026-08-29 field report's "expensive mid-implement halt" is exactly this case). This one dispatch is exempt from the clean-tree gate: the applied patch is source code, which a bookkeeping commit never stages — commit only the spec and the log, then spawn. A spec-path dispatch skips build-auto's own tree check, its review diffs from the spec's baseline (which covers the applied patch), and its finalize commits the patch with the rest of the reviewed diff.
3. Bookkeeping commit (Rule 16).
4. Re-spawn the same stage **on the spec path** — never on the story key once a spec exists, and **never by `SendMessage` to the old stage agent**. Its `Agent` return value was its completion signal; an agent "resumed" after returning is alive with no durable record of it (no open `stage_spawned`), invisible to resume detection, and a de-facto background stage. The re-spawn's apparent cost is the price of durability: the answer lives in the spec, not in a chat message, so the story survives a crash between the answer and the re-spawn.

### Pipeline Flow

```
Lead resolves the architecture spine (newest final ARCHITECTURE-SPINE.md, else newest, else *architecture*.md, else none); logs spine_resolved — path included in every spawn prompt.

For each epic in range:
  Lead verifies clean working tree (parent + every submodule); halts on dirty state.

  # IDE file-sync toggle applicability — conditional, IRIS/ObjectScript projects only (see Source Control Branching)
  ideSync = (.vscode/settings.json exists AND objectscript.conn.active == true)
  If ideSync: record original value   # sync STAYS ON except inside the two branch-op windows; never stage settings.json
  If ideSync: set objectscript.conn.active = false   # >>> Window A start — wraps resume-checkout + SC-1/SC-2 + epic-branch checkout

  # Resume-mode detection (see Resume Semantics)
  For each affected repo, lead determines mode by cross-referencing:
    - cycle-log-epic-{N}.md existence + entries
    - sprint-status.yaml story states for this epic
      (SPRINT_PLAN status for the computed snapshot — per-status counts, recommendation.story_key;
       SPRINT_PLAN validate first if the YAML's integrity is in doubt; per-story states via the
       TRACKER_READ shorthand — status has no per-story list)
    - spec-<key>.md frontmatter status per story (draft | ready-for-dev | in-progress | in-review | done | blocked)
    - {TICKET}-epic{N} branch presence locally and on remote
  Modes per repo: FRESH | RESUME | REMOTE_ONLY | LOCAL_ONLY | AMBIGUOUS | INTEGRITY_ERROR
    INTEGRITY_ERROR or AMBIGUOUS → halt and surface to user
    LOCAL_ONLY                   → halt and ask user (remote deletion intentional?)
    REMOTE_ONLY                  → fetch + checkout {TICKET}-epic{N}; then RESUME
    RESUME                       → run Cross-stage integrity checks; on pass, set resume point
    FRESH                        → fall through to SC-1 / SC-2

  # SC-1 / SC-2 — only where FRESH
  For each repo in FRESH:
    Lead verifies the feature branch
      If missing → STOP and ask user (TICKET, Description, root); validate; create; push; log feature_branch_created
    Lead verifies {TICKET}-epic{N}
      If missing → branch off the feature branch (deterministic); push; log epic_branch_created
  Lead checks every affected repo out to {TICKET}-epic{N}; logs epic_branch_checked_out
  If ideSync: restore objectscript.conn.active to original   # <<< Window A end — MANDATORY (try/finally; restore even on halt/failure)

  Lead: LEDGER load → re-own every owner=none entry (append owner=<story key>|burndown) → logs ledger_load   # see Ledger Drain
    (a legacy-grammar ledger — no `### DW-` headings — HALTS here: run the Step-1 migration first)
  Lead executes /bmad-sprint-planning via Skill tool, HEADLESS, with "Ledger load: <the load line>" in its argument; parses the trailing JSON:
    gate=FAIL or status=blocked → STOP; surface findings (and the saved implementation-readiness.md if written)   # Unattended: at a later epic's start, FAIL stops that epic (U8)
    gate=CONCERNS → surface once; continue
    gate=PASS → continue
  Lead logs sprint_planning_complete gate=<PASS|CONCERNS>
  If Epic N-1 retrospective exists OR LEDGER slice all is non-empty OR sprint-status.yaml action_items has status=open entries,
     AND no prior retro_review_* entry for this epic:
    Lead reads all sources, CLOSES every ledger item (resolved-by | own | terminal — Rule 17), inserts Story X.0 into epics.md,
      plans it via the normal PLAN spawn, appends the triage table to the spec; logs retro_review_complete
  Else if no prior retro_review_* entry:
    Lead logs retro_review_skipped reason=no_retro_no_open_ledger_no_open_action_items
  Else:
    (resume case — gate already passed; skip)

  # Epic-context pre-warm (see "Epic Context Pre-warm") — after any epics.md edit, before the first plan spawn
  Lead renders bmad-build-auto, spawns an Opus subagent on the rendered compile-epic-context.md for epic N, verifies the header,
    bookkeeping commit; logs epic_context_compiled

  For each story, strictly sequentially, starting from the resume point:
    If any file in planning_artifacts is newer than epic-{N}-context.md → re-run the pre-warm + bookkeeping commit first
    Lead asserts current branch == {TICKET}-epic{N} in every affected repo; halts on mismatch
    Lead writes stage_spawned (stage= spawn_at= model= agent_name= cycle_iteration=) IMMEDIATELY BEFORE each Agent call —
      for build-auto stages inside the bookkeeping commit that precedes the spawn; closed by the stage's success entry or a clarification entry

    # --- PLAN (Opus) ---
    Lead logs stage_spawned stage=plan; bookkeeping commit (Rule 16: tree clean before every build-auto spawn)
    Lead invokes Agent for /bmad-build-auto with "<story-key> Halt after planning." (model=opus unless re-pinned)
      → re-reads spec-<key>.md: status ready-for-dev → captures spec path; logs story_created path=<spec> build_status=ready-for-dev
      → status blocked → Clarification protocol (build-auto table)
    Lead validates the spec: Integration ACs (Rules 1/2), AD mapping (Rule 6), deleted sections re-created, decision dependency   # see "Lead Validates the Story Spec"
    Lead logs spec_validated; SPRINT_PLAN generate --set <key>=ready-for-dev; bookkeeping commit (everything under implementation-artifacts)

    # --- IMPLEMENT (Sonnet) ---
    Lead resolves the PREVIOUS story's ci=pending (gh run view) → logs ci_resolved   # resolution point (1): the plan above did not wait
      failure → classify (logs ci_red_classified): infra → timed retry, never rework | flake → one rerun + flake: ledger entry | story → HIGH: re-open THAT story's spec with a [CI] task, one rework iteration, re-smoke, re-commit — this story's spec waits at ready-for-dev (inherited red → surface to the user)
    Lead: SPRINT_PLAN generate --set <key>=in-progress --set epic-{N}=in-progress; bookkeeping commit
    Lead invokes Agent for /bmad-build-auto with the spec path (model=sonnet unless re-pinned)
      → re-reads spec: status done → build_sha = HEAD (its finalize commit); asserts tree clean
        Lead harvests spec frontmatter deferred: → LEDGER new … <status> <owner> harvest (born owned, Rule 15); SPRINT_PLAN generate --set <key>=review
        Lead mirrors baseline_revision into the spec frontmatter as baseline_commit (bmad-code-review reads that key)
        logs dev_complete build_sha=<sha> baseline_revision=<sha> review_loop_iteration=<n> followup_review_recommended=<bool> deferred=<n>
      → status blocked → Clarification protocol (build-auto table)
    Lead executes AD-tooled AC verifications against the spine (lead-side, sequential per AC, one mutation each — Rule 19); logs adr_verifications_complete
    Lead invokes Agent for /bmad-qa-generate-e2e-tests (model=sonnet unless re-pinned) → reads ## Tests Added → logs qa_complete
    Lead invokes Agent for /bmad-code-review with the spec path (model=opus unless re-pinned) → reads return
      → verifies sprint-status.yaml moved to done|in-progress; if still review, SPRINT_PLAN generate --set <key>=<reviewer's final status>; SPRINT_PLAN validate
      → checks for stray "## Deferred from:" bullets in deferred-work.md (reviewer ignored the ledger pre-answer) → harvest via LEDGER new, strip, log ledger_stray_bullets
      → logs cr_complete
    If review returned a ## Fix Pack (trivial LOW, Rule 15): the reviewer has already applied mechanical patches; behavioral-but-trivial items go through ONE rework iteration below (cycle_iteration+1) — bounded: exactly one fix-pack iteration per story, shared with the first-review MED iteration; the smoke gate still follows
    While review left story in-progress (a HIGH; or, on the story's FIRST review only, an unpatched in-story MED with fix-risk ≤ med — `routed` / `escalated` mediums never block) AND rework iterations < 3:   # see Rework Loop
      Lead re-opens the spec (status in-progress; unresolved findings appended as unchecked Tasks & Acceptance items); logs rework_opened trigger=<high|med_first_review|fix_pack|smoke|ci> items=<csv>; REWORK commit (QA tests + reviewer patches + bookkeeping)
      Lead re-spawns /bmad-build-auto on the spec (model=sonnet; cycle_iteration+1; "Rework re-dispatch" block appended: work only the open items, followup_pass=true) → harvest deferred:, logs dev_complete
      Lead re-runs AD verifications if rework touched AD-constrained ACs; SPRINT_PLAN generate --set <key>=review
      Lead sets spec baseline_commit = this rework's baseline_revision (scoped re-review: <rework baseline>..HEAD, never the whole story again); re-spawns /bmad-code-review with the "Rework re-review" block → logs cr_complete
      A re-review that finds no HIGH closes the story (remaining MEDs routed/escalated with owners) — the loop exits here even if MEDs remain (Rule 15 exit condition)
    If still in-progress after 3 iterations (Unattended: no override — stop the story, set it aside; U4-U5): STOP; logs rework_cap_reached; surface outstanding findings to user — an override is accepted only with a finite item list and closes at the next review without HIGH (rework_cap_override items=… close=…; see Rework Loop)
    Lead adjudicates LEDGER slice <key> against delivered scope (resolved-by | re-own | terminal) until empty; logs ledger_adjudicated   # Rule 17
    Lead performs per-story smoke (lead-side); logs smoke_complete
    Lead runs the fresh-environment check (the story's added + changed tests on a fresh environment, forward then reversed — Rule 19); logs fresh_check; red → failed-smoke path
    Lead asserts current branch == {TICKET}-epic{N}; commits (QA tests, review patches, bookkeeping) + pushes ONLY to {TICKET}-epic{N} (submodules first if applicable)
    Lead verifies the push on origin (git fetch; merge-base --is-ancestor) and the run's headSha; logs committed ci=pending run=<id> (gh run list --branch --commit; ci=none|unavailable per pre-flight); logs story_cycle; next story
    # the pending run is resolved before the NEXT story's implement spawn — its plan proceeds meanwhile (see IMPLEMENT above)

  # Burn-down gate (Rule 17) — before the epic closes
  Lead: LEDGER slice burndown + slice <each story key of epic N>:
    empty → logs ledger_burndown_skipped reason=empty
    else  → set aside every escalated / decision-pending entry for the decision sheet (to_decision_sheet=N; never re-own or decide them here)
            resolve / terminal what can be closed now; if what remains exceeds ledger_cap (default 8) OR any remaining entry has 2+ occurrences:
              charter Story N.<M+1> "Epic N burn-down" (M = highest existing story number under Epic N across epics.md + sprint-status.yaml; reuse an existing burn-down story) — at most burndown_story_max (default 12) entries by priority (severity, occurrences, low fix-risk first)
              (insert into epics.md, SPRINT_PLAN generate, pre-warm, full per-story pipeline incl. its own ledger_adjudicated)
            every entry still non-terminal (the overflow) → append owner=<specific next-epic story key> with the reason (never "burndown", never "none")
            logs ledger_burndown_complete open_before= resolved= terminal= chartered= chartered_size= overflow= reowned= to_decision_sheet= open_after= cap= story_max= filed_this_epic= closed_this_epic= drain_ratio=
  Lead: SPRINT_PLAN generate --set epic-{N}=done (no skill writes this transition); bookkeeping commit; logs epic_status_done
  Lead evaluates this epic's cycle log against the escalation / de-escalation / rollback thresholds — EVERY project; logs model_tier_checkpoint armed=<bool> result=…
    result=recommend_escalate|offer_mixed → surface evidence to the user; on approval (creating model-overrides.yaml if absent): update overrides+history; logs model_tier_changed   # Unattended: recorded, never applied (U5)
    result=revert_required (mixed active, smoke defects) → mandatory: update the file; logs model_tier_changed
  Lead pauses: "Run a retrospective?"   # human-in-the-middle gate — never auto-answered, never times out into a default (Unattended: headless -H <epic>, verdict read; U9)
    If yes → /bmad-retrospective via Skill tool, fully interactive (all WAITs reach the user), with the burn-down counts in its argument; logs epic_retro_complete
    If no  → logs epic_retro_skipped reason=user_declined
  Lead presents the epic-close decision sheet (Rule 17 (5)): every escalated + decision-pending entry, one sitting, recommended disposition each   # Unattended: classified; owner-only → inbox (U9)
    → writes the answers as by=merge_gate trailer lines (human-owned → action_items + dropped note=action_item); bookkeeping commit; logs decision_sheet_recorded
  Lead pauses: "Merge {TICKET}-epic{N} → feature, delete {TICKET}-epic{N}?" → if yes:   # Unattended: when_green, integrate forward first (U9)
    Logs epic_merge_started head=<CI-verified sha> on the epic branch; bookkeeping commit (the merge carries it to origin)   # the write-ahead marker
    If ideSync: set objectscript.conn.active = false   # >>> Window B start — wraps the merge branch ops
    For each affected repo, submodules-first then parent:
      checkout feature; pull; merge --no-ff {TICKET}-epic{N}; push feature; delete {TICKET}-epic{N} local + remote
    If ideSync: restore objectscript.conn.active to original   # <<< Window B end — MANDATORY (try/finally; restore even on conflict/halt)
    Verifies on origin; logs epic_merged_to_feature on the feature branch; commits it (chore(epic-N): record merge); pushes
  Else:
    Logs epic_merge_skipped
  Logs epic complete; next epic
```

### Smart Parallelism (retired under v6.11 — stories run strictly sequentially in one checkout)

Story-level batching inside a single working tree is no longer possible:

- **Implement** spawns: each `bmad-build-auto` run makes its own finalize commit and then verifies the tree is clean — two concurrent runs in one checkout see each other's uncommitted work and HALT `finalization left repository dirty` (or race on the git index).
- **Plan** spawns: build-auto's step-01 version-control check ("require a clean working tree … HALT on a dirty tree") runs *after* it has loaded/compiled the epic context, so a sibling plan spawn that has already written its untracked `spec-<key>.md` makes the slower sibling halt; siblings also race on `epic-<N>-context.md`.

Concurrency therefore lives one level up — the parallel kit's per-epic worktrees (Orchestrator Mode). Within an epic: one story at a time, one stage at a time. Emitting multiple pipeline-stage `Agent` calls in one message is an anti-pattern under v6.11.

**Write-ahead rule (extended by Rule 16):** write the cycle log entry for a completed stage BEFORE the next dependent action, and COMMIT it before the next `bmad-build-auto` spawn. For `committed`: write immediately after `git push` returns success. If crashed between push success and log write, inspect `git log --oneline` on resume; if the matching commit exists, write the missing log entry — do NOT re-run the commit. The same applies to `dev_complete`: build-auto's finalize commit (`build_sha`) plus spec `status: done` without a `dev_complete` entry means the entry was lost, not the work — write it, harvest `deferred:`, set `review`, continue.

### Rework Loop (implement ↔ code review, bounded, with an exit)

`/bmad-code-review` closes the story itself: when every decision-needed + patch finding is resolved and no high/medium severity remains unresolved (`routed` / `escalated` mediums are ledgered with an owner and do not count), it sets the story (and sprint-status.yaml) to `done`; otherwise it sets `in-progress` and leaves unchecked review-finding items (action items) in the spec file.

**What re-opens a story (Rule 15):** a HIGH at any review; an unpatched in-story MED with fix-risk ≤ med at the story's FIRST review (one bounded iteration, shared with the Fix Pack); a failed smoke; a red CI run. Nothing else — in-epic MEDs are routed to the owning story, and **a re-review that finds no HIGH closes the story** with its remaining MEDs routed or escalated. (Field report 2026-09-11: without this exit, "fix every in-footprint MED" + a whole-story re-review after every rework + stale mutation lines from the rework's own test changes produced 9 implement passes, 3 reviews and 13 stage spawns on one story that had had zero HIGH since its third review; the cap was overridden with "continue until complete", and complete was not a reachable state.)

`bmad-dev-story`'s review-continuation mode has no v6.11 successor, so the lead re-opens the spec for `bmad-build-auto` instead. When the review returns `in-progress`:

1. **Lead re-opens the spec:** set frontmatter `status: in-progress`; append every unresolved finding as an unchecked item under `## Tasks & Acceptance` (`- [ ] [Review] <finding> — <file:line> — <what the fix must do>`); leave the frozen `<intent-contract>` block untouched; note the iteration in `## Spec Change Log` (Rule 14: once the spec carries the `oversized` warning, add nothing but the open items — no restated history); log `rework_opened cycle_iteration=N iteration=<k> trigger=<high|med_first_review|fix_pack|smoke|ci|cap_override> items=<csv>` — the item list is the iteration's whole scope. If the tracker reads `done` (a Fix Pack, a failed smoke, or a CI red after code review closed the story), `SPRINT_PLAN generate --set <key>=in-progress` — the one sanctioned downgrade, and only with `rework_opened` logged for that story; step 3 moves it back to `review`. **Rework commit:** commit QA's tests, the reviewer's applied patches, and the bookkeeping files together (`chore(epic-N): rework <story-id> iteration <k>`) — they are still uncommitted (the QA and CR spawns are forbidden to commit), and a spec-path dispatch skips build-auto's pre-flight tree check, so anything left uncommitted would be silently absorbed into build-auto's next diff and finalize commit.
2. Re-spawn `/bmad-build-auto` on the spec (Sonnet tier, `cycle_iteration` incremented) **with the "Rework re-dispatch" block appended** to the implement spawn prompt. Its resume table routes `in-progress` → implement — a first pass from the skill's point of view (its `followup_pass` is set only for a `done` spec, so without the block every rework ran as a first pass and its convergence clause never engaged): the block pre-answers step-04's follow-up recommendation as a follow-up pass and confines the work to the open items. It captures a new `baseline_revision` (after the rework commit), works the open tasks, self-reviews, finalizes `status: done`, commits. Harvest `deferred:`; log `dev_complete` (its `followup_review_recommended` is now meaningful: `true` only if the pass patched a HIGH).
3. Re-run AD-tooled verifications if the rework touched AD-constrained ACs. Then `SPRINT_PLAN generate --set <key>=review` (the previous code review moved the tracker to `in-progress`; the re-review expects `review`).
4. Re-spawn `/bmad-code-review` **scoped to the rework** — set the spec's `baseline_commit` to THIS rework's `baseline_revision` (the skill's step-01 uses that key as its diff baseline, so the reviewer diffs `<rework baseline>..HEAD` + working tree) and append the "Code-review re-spawn on rework" block: the previous round's unresolved items as its checklist, the original baseline for context, findings outside the rework range only when HIGH, and the exit condition. The whole story was reviewed once, at round 1; re-reading it after every rework re-samples the same finding distribution (and blows through the skill's own ~3,000-line chunking guard — field report 2026-09-11: round 3 re-read 7,247 lines for 0 HIGH). Fresh eyes on old code always find something tolerable.
5. **Cap: 3 rework iterations.** (*Unattended:* no override — the story stops and is set aside, doctrine U4–U5.) Non-convergence after 3 → STOP, log `rework_cap_reached outstanding=<csv>`, and surface the outstanding findings to the user (mirrors `bmad-build-auto`'s own 5-iteration review-repair cap). Endless implement↔review ping-pong is a signal the spec is wrong, not the code. **Cap override contract:** the lead accepts an override only as `rework_cap_override cap=3 used=3 authorized_by=user items=<finite csv of finding / DW ids> close=next_review_without_high` — an open-ended instruction ("continue until complete") is answered with the item list to confirm, never acted on; the extra iteration works exactly those items, and the story closes at its next review unless that review finds a HIGH.

**Decisions do not re-open a reviewed story.** A check-in answer or a decision-sheet call on one of this story's `escalated` / `decision-pending` entries, taken after a review that found no HIGH, becomes a `DW-n:` acceptance bullet on a named later story or the burn-down story (Rule 17 (1b)) — not a rework of this one. If the story is still inside its loop (a HIGH outstanding), the decision's items ride along in the current iteration's `items=` list.

The Fix Pack path (trivial LOW, Rule 15): mechanical items are patched by the reviewer inline; behavioral-but-trivial items take exactly ONE iteration of the loop above — the same single iteration a first review's unpatched in-story MEDs use.

Only a story at `done` proceeds to smoke + commit.

### Status Ownership (who writes sprint-status.yaml)

`bmad-build-auto` never touches `sprint-status.yaml` (only the interactive `bmad-build` syncs it, and that skill is not a pipeline stage), so under v6.11 the **lead** owns the early story transitions and writes them ONLY through `SPRINT_PLAN generate --set` (deterministic, preserves comments; `--set` is unconditional, so the lead checks the current value first and never writes a lower rank):

| Transition | Written by |
| --- | --- |
| file generation, `backlog` seeding, `epic-N` keys, `epic-N-retrospective: optional` | `/bmad-sprint-planning` (headless) via `sprint_plan.py generate` |
| story `backlog → ready-for-dev` (after the plan spawn is validated) | **the lead** — `SPRINT_PLAN generate --set <key>=ready-for-dev` |
| story `ready-for-dev → in-progress` + epic `backlog → in-progress` lift (immediately before the implement spawn) | **the lead** — `--set <key>=in-progress --set epic-{N}=in-progress` |
| story `in-progress → review` (after the implement spawn's spec reads `status: done`) | **the lead** — `--set <key>=review` |
| story `review → done` (or back to `in-progress`) | `/bmad-code-review` (its step-04 sync). The skill only sets `{story_key}` itself when it found the story via its sprint-status scan; on an explicit spec-path dispatch the spawn prompt must state the key, and the **lead verifies after `cr_complete`** that the file moved — if it still reads `review`, apply the reviewer's reported final status with `SPRINT_PLAN generate --set` (this is the one sanctioned lead write of `done`/`in-progress` for a story). |
| `epic-{N}-retrospective: optional → done`, `action_items` append | `/bmad-retrospective` via its `sprint_status.py update` |
| **epic `in-progress → done`** (after the last story reaches `done`) | **the lead** — `--set epic-{N}=done`; log `epic_status_done` |

Status enum (kebab-case, unchanged in BMAD v6.11): story `backlog → ready-for-dev → in-progress → review → done`; epic `backlog → in-progress → done`; retrospective `optional → done`; action items `open → in-progress → done`; legacy aliases `drafted` ≡ `ready-for-dev`, `contexted` ≡ `in-progress` (normalized by the script). Never downgrade a status, except the Rework Loop's re-open of a `done` story and an unattended set-aside — `--set` would happily do it (it is the script's repair path), so read before you write; never hand-edit the YAML.

**Do not confuse the two status vocabularies.** The spec file's frontmatter `status` is `bmad-build-auto`'s machine state: `draft | ready-for-dev | in-progress | in-review | done | blocked`. `in-review` (spec, mid-self-review) is NOT `review` (sprint-status, awaiting code review), and a spec at `done` means "build-auto finished", not "story done" — the story is `done` only when `bmad-code-review` says so. Never copy one vocabulary into the other.

### Per-Story Smoke (Critical Gate)

After code review (HIGH/MED resolved) and before commit, the lead performs a per-story smoke — a direct exercise of the story's deliverable in its target runtime. Mandatory; only the method varies.

**Method by deliverable runtime:**

- UI / browser-deployed: drive the dev server (or a deployed build) via browser-automation MCP. Navigate, exercise, assert on DOM / render / console.
- CLI / library: invoke the new command or library entrypoint against a real runtime. Assert on stdout / stderr / return code / produced files.
- Service / API: real HTTP request against the local server (or staging). Assert on status code + response body + side-effect surface.
- Other: whatever exercise mirrors production use. Minimum: the lead invoked the new code path against a real runtime and observed expected outcomes via an out-of-band channel.

The smoke is not a substitute for automated tiers; it's the final check that the wired-up system end-to-end produces the user-observable outcome the story promises. It is also platform-bound: a suite that passes on the lead's machine is evidence about that machine, not about the CI runner or the deployment target (field case: a test that regex-scraped a nested runner's summary passed on Windows and failed on Ubuntu, where the runner colourised its output — unreproducible locally by construction). CI is the only gate that runs on the platform the project ships from; see "CI Is a Gate".

**Mechanics:**

1. After `cr_complete` (HIGH/MED resolved), determine the smoke method from the story's diff (`git diff --name-only <story_base>..HEAD`) plus the spec's Tasks & Acceptance / I/O matrix.
2. Execute, capture evidence (screenshots, stdout, response body).
3. On failure: do NOT commit as-is (*Unattended:* always (b)). Either (a) surface to user, or (b) treat the failure as an unresolved HIGH finding and run one Rework Loop iteration (re-open the spec with the failure as a `[Smoke]` task, rework commit, implement re-spawn, code-review re-spawn) then re-smoke. Failed smoke is HIGH, never deferrable.
4. On success: log `<UTC> TAB Story <id> TAB smoke_complete TAB method=<browser|cli|api|other> result=pass iterations=<N> defects_caught=<N> evidence=<path-or-summary> model=<lead-model>`. `iterations` is 1 for first-run pass, bump on re-smoke. `defects_caught` = count of bugs the smoke caught that automated tiers passed.
5. Run the fresh-environment check below, then proceed to commit.

**Verification arrangement.** By default the plan marks each `## Verification` command `(loop)` or `(once)`: the implement stage runs the story's own tests (`(loop)`) on every verification pass and the full suite (`(once)`, in batches that fit the foreground cap) exactly once, after its last patch and before it finalizes; CI then runs everything on the platform the project ships from. Two project options, each recorded under Project-specific rules so the plan writes `## Verification` to match. **(a) CI-fresh tiers** — a test tier that CI runs on fresh instances, and that is resolved before the next implement spawn ("CI Is a Gate"), may be left to CI and dropped from `(once)`; the exemption lapses whenever CI is `none` or `unavailable`. **(b) Lead-owned full run** — a project whose implement stage runs on a tier lighter than Sonnet, or whose full suite takes several times the foreground cap, moves the `(once)` run up to the lead (the runner, in a parallel run): the lead runs it after code review and before this smoke, in capped foreground batches like any other agent — moving the run up does not lift the cap, and a runner that ends its turn has returned too (Rule 18 (5)) — and a red takes the failed-smoke path. The trade for (b): a regression outside the story's own tests surfaces one implement-and-review round later than it would in-stage; the gain: the long run is done by the longest-lived, strongest agent instead of the cheapest and shortest-lived one. (Field reports 2026-10-08 and 2026-10-10: inside one epic each story ran the full browser suite 4–9 times, about 12 minutes each; two Haiku handoffs backgrounded their sweeps despite explicit foreground instructions.)

**Fresh-environment check (Rule 19 — after the smoke, before the commit).** Run the story's added and changed tests (`git diff --name-only <story_base>..HEAD`) on a fresh environment — rebuilt by the project's recipe if its rules record one (a throwaway database or instance, a fresh container), else a clean checkout — once in their normal order and once reversed. A red is a HIGH that takes the failed-smoke path. Log `<UTC> TAB Story <id> TAB fresh_check TAB env=<recipe|clean-checkout> tests=<n> result=<pass|fail> order=forward,reverse`. In a parallel run the runner uses its own throwaway, never another runner's (Rule 13).

### CI Is a Gate, Not an Assumption

Every quality gate in this pipeline runs the suite locally; none of them is the platform the project ships from. The CI run a push starts is therefore a first-class result, recorded write-ahead and resolved before the next irreversible step — never awaited per story (wall-clock), never assumed green.

**Pre-flight (once per run, on `runtime_gate`):** `ci=gh` when `.github/workflows/` holds a `*.yml` or `*.yaml` workflow, `gh auth status` succeeds, and `origin` is a GitHub remote; `ci=none` only when the repo has no CI configuration at all (no GitHub workflow and none of `.gitlab-ci.yml`, `azure-pipelines.yml`, `.circleci/config.yml`, `Jenkinsfile`, `bitbucket-pipelines.yml`, `.buildkite/`, `.travis.yml`, `.drone.yml`); `ci=unavailable` when CI exists but its results cannot be read here — `gh` missing or unauthenticated, a non-GitHub remote, or another CI system — surface `unavailable` once ("CI results cannot be read from this session; the merge gate will ask you to confirm CI by hand") rather than silently skipping.

**At `committed`:** after the push, `gh run list --branch {TICKET}-epic{N} --commit <sha> --json databaseId,status,conclusion -L 1` (retry a few times over ~30 s — the run registers shortly after the push); record `ci=pending run=<id>` (or `ci=success` if it already finished; `ci=none` / `ci=unavailable` per pre-flight). No waiting. Push the code commit on its own and confirm its run registered for that exact sha (match `headSha`) before any further push to the branch — a workflow with `concurrency: cancel-in-progress` cancels the run being waited on when a later push lands. Never put a CI-skip token in a code commit's message: a quoted `[skip ci]` skips the run all the same.

**Resolution points — each logs `ci_resolved story=<id> run=<id> result=<success|failure|cancelled>`:** (1) before the next story's implement spawn (Task Sequence per-story step 4) — the next story's plan, spec validation and spec gate proceed meanwhile, and a red reworks the previous story first while the new spec waits at `ready-for-dev` (field report 2026-10-10: waiting at the plan spawn idled the lead about 30 minutes per story); (2) at the burn-down gate; (3) at SC-4 / SC-4-P as the merge precondition (`ci_green_verified` on the branch head, or `merge_gate_blocked reason=ci_red`); (4) at resume (cross-stage integrity check 6). `gh run view <id> --json status,conclusion`; still `in_progress` → wait here, with jitter, up to the project's usual run time.

**On `failure` — classify first, then act; log `ci_red_classified run=<id> class=<story|flake|infra> test=<name|none>`.** `infra`: the run failed before any test ran (runner lost, image-pull or registry limit, network, provisioning) — retry it on a timer (about 10, 20, then 40 minutes) and never count it as rework; three infra reds in a row → surface to the user (*Unattended:* re-run and poll, then doctrine U5's CI rows). `flake`: the failing test already has an open `flake:` ledger entry, or it fails and then passes on one unchanged rerun — rerun once, and file a `flake: <test>` ledger entry (or append an `occurrence=`) routed to the burn-down with priority; the same test red twice on the same head is `story`. `story`: anything else — the path that follows. (Field report 2026-10-10: five different flaky tests forced reruns in two days, and one merge needed five attempts — a flake, then three registry pull-limit failures in which no test ran.)

**On a `story` red:** if the failing job's first red commit is the story just committed → a HIGH finding against it, never deferrable, same path as a failed smoke: re-open the spec with `- [ ] [CI] <job>: <failure summary> — <link>` under Tasks & Acceptance, rework commit, implement re-spawn, code-review re-spawn, re-smoke, re-commit, new `ci=pending`. If the red predates the story (inherited) → surface to the user with the run link and the first red sha; do not start new work on a red branch (*Unattended:* rerun, then at most one fix-forward per root cause, else stop the epic — U5). `cancelled` → re-check the latest run for the branch head before deciding.

**Runner mode (parallel kit):** identical inside the worktree; the completion contract carries `ci: <success|none|unavailable> run=<id>` for `head:`, and the orchestrator verifies it independently before the merge gate.

**Verified on origin, never inferred from an exit status.** `committed`, `ci_green_verified` and `epic_merged_to_feature` are written only after `git fetch` and `git merge-base --is-ancestor <sha> origin/<branch>` succeed for the pushed or merged sha, and — for a green — `gh run view <id> --json headSha,conclusion` names that exact head. (Field report 2026-10-10: a merge was reported but never pushed; an `&&` guard skipped the push silently.)

### Ledger Drain (Critical Gates — Rule 17)

The deferred-work ledger has as many drains as it has inflows, and every drain is a logged stage. The lead never opens `deferred-work.md`; every read is `LEDGER load|slice|show` and every write is `LEDGER new|append` (see the shorthand under Pre-flight; grammar and vocabulary in Rule 15).

**Epic start — `ledger_load` (before sprint planning).** `LEDGER load`. If the file exists but `total=0` while `grep -c "^### DW-"` is 0 and the file is non-empty, it is a legacy-grammar ledger — HALT and run the kit's Step-1 migration. Re-own every `owner:none` entry (`LEDGER slice none`; `append DW-n "owner=<story key in this epic|burndown> by=load note=…"`). **Repair every `UNKNOWN` owner** (`load` marks them; `LEDGER slice unknown` lists them): take the owner's `<epic>-<story>-` prefix and look it up in `SPRINT_PLAN status` / the tracker — exactly one key with that prefix → `append DW-n "owner=<that key> by=load note=retitle_repair"`; zero or several → HALT and surface (`ledger_owner_repaired repaired=N halted=N`; *Unattended:* at a later epic's start, re-own `owner=burndown note=owner_unresolved` instead — U8). An entry on a key no gate will ever query is the silent failure this check exists for. Log `ledger_load total= open= routed= escalated= decision_pending= terminal= owner_unknown= burndown=<owner:burndown count>`; put the same line into the headless sprint-planning argument (`Ledger load: …`) so the readiness gate can weigh it, and size Story X.0 by it.

**Harvest — at `dev_complete`, born owned.** For each item in the spec's frontmatter `deferred:` list: `LEDGER slice all` + grep the summaries for the same root cause → `append DW-n "occurrence=<key>"`; otherwise `LEDGER new "<summary>" "<spec path>" <severity from the item, else med> <fix-risk assessed by the lead> <footprint> "<evidence, one line>" <status> <owner> harvest "<note>"` — in-story → `open owner=<this key>`; in-epic → `routed owner=<the story key that will touch it>`; out-of-footprint → `routed owner=burndown`. `dev_complete` records `deferred=<count harvested>`. **Every `routed owner=<story key>` written here, at `cr_complete`, or at adjudication also becomes a `- DW-n: <summary> (ledger; routed by <gate> <date>)` bullet under that story's acceptance criteria in `epics.md`** — the lead/runner owns its own epic's blocks (Rule 11), so this is immediate, in the same bookkeeping commit; a key in another epic is written by the sequential lead now or by the orchestrator post-merge. At most `routed_story_max` bullets per story (default 6; `routed_max:` in `_bmad/custom/ledger.yaml`) — the excess stays ledger-only and is counted. Log `ledger_routed_planned story=<key> entries=N excess=N by=<gate>`; the `epics.md` edit makes the planning artifacts newer than the epic context, so the pre-warm re-runs before the next plan spawn (Rule 17 (1b)).

**Per-story adjudication — `ledger_adjudicated` (after the last `cr_complete`, before smoke).** `LEDGER slice <this story key>`. For each entry, judge it against DELIVERED scope — `git diff --name-only <baseline_revision>..HEAD`, the tests, the spec's Verification — never against the story title:

| Finding | Trailer line |
| --- | --- |
| Covered by the delivered change | `status=resolved-by:<key> by=adjudication note=<commit/test evidence>` |
| Real, not covered, belongs to another story in this epic | `status=routed owner=<that key> by=adjudication note=<residual, restated>` |
| Real, not covered, belongs to no story yet | `status=routed owner=burndown by=adjudication note=<residual, restated>` |
| Real but decided not worth pursuing | `status=wontfix-accepted by=adjudication note=reopen_if=<observable probe>` |
| Spec-bound / no realistic failure / invalid or duplicate | `status=by-design` / `status=wontfix-theoretical` / `status=dropped` (+ note) |

`LEDGER slice <key>` must read empty before `smoke_complete`. Log `ledger_adjudicated owned=N resolved=N reowned=N terminal=N`. An entry the story was supposed to close but did not is a real defect: if it is LOW and a two-way door, fix it in the Fix Pack iteration; otherwise re-own it honestly — re-owning is not failure, silent carry-forward is.

**Burn-down gate — `ledger_burndown_complete|skipped` (after the last story's `committed`, before `epic_status_done`).** First resolve the last story's `ci=pending` (`ci_resolved resolved_at=burndown`; a red branch is handled per "CI Is a Gate" before the epic closes). Then enumerate `LEDGER slice burndown` plus `LEDGER slice <key>` for each of this epic's story keys (the latter should be empty after adjudication; anything there is a missed gate — adjudicate it now). If the union is empty → `ledger_burndown_skipped reason=empty`. Otherwise:

1. **Set aside what is not work.** Every `escalated` and `decision-pending` entry in the union goes to the decision sheet below (`to_decision_sheet=N`). The gate never re-owns, closes, or "decides" one of these — a burn-down story cannot ratify a tick rate or run a playtest, and a runner has no standing to make a product call.
2. Resolve or make terminal everything story-closable that can be closed on evidence now (same table as adjudication, `by=burndown`). A `burndown`-owned LOW is a Rule 15 miss: fix-pack-sized → into the chartered story; otherwise `wontfix-accepted reopen_if=`.
3. If the remainder exceeds `ledger_cap` (default 8; a project may set `cap:` in `_bmad/custom/ledger.yaml`) OR any remaining entry has two or more `occurrence=` lines, charter **Story `N.<M+1>` — "Epic N burn-down"** (M = the highest existing story number under Epic N, read from BOTH the `### Story N.M` headings in `epics.md` and the `N-M-*` keys in `sprint-status.yaml` — gaps are irrelevant and `N.0` counts, so the number always sorts last and never collides. There is no conventional number, and "the next unused number" is wrong because it can fill a gap below an existing story. If `epics.md` already holds an "Epic N burn-down" story — a resumed gate — reuse it and its key. Derive the sprint key from the number actually used): insert it under Epic N in `epics.md` with one acceptance bullet per entry (`DW-n: <summary>`), **at most `burndown_story_max` entries** (default 12; `story_max:` in `_bmad/custom/ledger.yaml`) chosen by priority — HIGH/MED severity, then `occurrence=` count, then lowest fix-risk — `SPRINT_PLAN generate`, re-run the pre-warm, bookkeeping commit, then run the full per-story pipeline on it (plan → validate → implement → AD gate → QA → CR → **its own `ledger_adjudicated`** → smoke → commit). Exactly one burn-down story per epic, of bounded size — the gate is bounded in both dimensions; a story that takes everything is either enormous or a relabelling in disguise.
4. Every entry still non-terminal after that — the **overflow**, counted `overflow=N` — gets `append DW-n "owner=<specific story key in the next epic> by=burndown note=<why it waits>"` **and a `DW-n:` acceptance bullet in that story's block** (Rule 17 (1b): the sequential lead writes it now; a runner leaves it to the orchestrator's post-merge step, which reads the ledger — no contract line needed). `burndown` owns no work after this gate and `owner=none` never is. If no next-epic story can own it, it is `wontfix-accepted` with a probe, or `decision-pending owner=burndown note=to_x0:<what to decide>` for the decision sheet — the one `burndown` owner allowed after this gate, because it is a question deferred to the next epic's X.0, not work. (The next X.0's key does not exist until that epic's step 6a inserts it, and `ledger.sh` refuses an owner the tracker does not know.) The next epic's X.0 is a mandatory fix-or-close pass over the overflow (see "Retrospective Review & Story X.0").
5. Log `ledger_burndown_complete open_before=N resolved=N terminal=N chartered=N chartered_size=N overflow=N reowned=N to_decision_sheet=N open_after=N cap=N story_max=N filed_this_epic=N closed_this_epic=N drain_ratio=X.XX`, where `filed_this_epic` = `total` now − `total` at this epic's `ledger_load`, `closed_this_epic` = `terminal` now − `terminal` then, and `drain_ratio` = closed / filed (1.00 = the epic closed as much as it filed). Resume refuses `epic_status_done` without a `ledger_burndown_*` entry for the epic.

**Epic-close decision sheet — `decision_sheet_recorded` (at SC-4, before the merge question; Rule 17 (5)).** (*Unattended:* classified; owner-only entries go to the inbox — doctrine U9.) The one place a human sees the ledger, at a gate the human already attends. Build it from `LEDGER slice all` filtered to `escalated` and `decision-pending` (any owner), plus — orchestrator mode — the runner's `decisions_for_user:` contract lines. Present every entry in one sitting with its evidence digest (`LEDGER show DW-n`, never the file) and a **recommended disposition**, then write the human's answer as exactly one trailer line `by=merge_gate`:

| The user says | Trailer line |
| --- | --- |
| Fix it — in a named story | `status=routed owner=<story key> by=merge_gate note=<decision>` + a `DW-n:` acceptance bullet in that story's `epics.md` block (Rule 17 (1b); bounded by `routed_story_max`) |
| Not worth it | `status=wontfix-accepted by=merge_gate note=reopen_if=<probe>` |
| It is the design | `status=by-design by=merge_gate note=<why>` |
| A human will do this (`note=human=`) | Append it to `sprint-status.yaml` `action_items` with BMAD's own writer — `uv run --no-cache .claude/skills/bmad-retrospective/scripts/sprint_status.py update --file <impl>/sprint-status.yaml --epic <N> --add-action '[{"action":"<text> (DW-n)","owner":"user"}]'` (it appends `status: open` and a stable `id`; never hand-edit the YAML) — then `status=dropped by=merge_gate note=action_item:<id>`. The item now surfaces at every retrospective and X.0 gate until the user closes it. |
| Decide later | Not an option at this sheet for a product call — it becomes `decision-pending owner=burndown note=to_x0:<what to decide>` (the next X.0's key does not exist yet, and `ledger.sh` refuses unknown owners) and is asked again at the next epic's X.0, decided once there. On the range's final epic there is no next X.0: decide now, or move it to `action_items` as a human-owned item. |

A decision with architectural weight (Rule 20) is also written into the spine in the same commit — the trailer line cites the AD id; a `to_x0` deferral with architectural weight is mirrored into the spine's `## Deferred`. Sequential lead: write the lines in the epic-branch checkout and bookkeeping-commit before asking the merge question. Orchestrator: present the sheet before the merge question, write the lines in the MAIN checkout on the feature branch immediately after the merge (post-merge bookkeeping commit; `deferred-work.md` is union-merge so order is irrelevant). Log `decision_sheet_recorded decided=N to_action_items=N to_x0=N` (or `decision_sheet_skipped reason=empty`). The retrospective that follows receives the same counts.

**Retrospective input.** Pass `Ledger: open_before=… open_after=… chartered=… overflow=… drain_ratio=… decisions=…` in the `/bmad-retrospective` argument so the human-in-the-middle sees the drain, not just the inflow.

**Final epic of the range.** (*Unattended:* classified like the decision sheet — U9.) After its burn-down gate, `LEDGER slice all` is the range's residue: every entry must be terminal or owned by a named story key outside the range — surface the list at the SC-4 merge gate as the ledger sweep (it is the decision sheet plus the overflow). A `note=to_x0` deferral cannot outlive the final epic: the decision sheet decides it or moves it to `action_items`.

### Epic Context Pre-warm (per epic, lead-driven)

`bmad-build-auto`'s epic-story path compiles `<implementation_artifacts>/epic-<N>-context.md` on first use — by spawning a subagent *before* its own clean-tree check, so the freshly written, untracked cache file would trip that check on the very first plan spawn of every epic (and again after any planning-artifact edit, which invalidates the cache). The lead therefore owns the compile:

1. Render the skill once: `uv run --no-cache _bmad/scripts/render_skill.py --project-root <abs-root> --skill <abs-root>/.claude/skills/bmad-build-auto`. It prints `read and follow <snapshot-dir>/workflow.md`; take `<snapshot-dir>`.
2. Spawn a synchronous Opus subagent (`general-purpose`, `model: opus`, `run_in_background: false` — Rule 18 (5)) whose prompt is: "Read fully and follow `<snapshot-dir>/compile-epic-context.md` for epic `<N>`. Working directory `<abs-root>` (Rule 13). Write only `<implementation_artifacts>/epic-<N>-context.md`."
3. Verify the file exists, is non-empty, and starts with `# Epic <N> Context:`; otherwise halt (`context compilation verification failed` — usually a missing or unparseable `epics.md`; *Unattended:* retry once, then stop the epic — U5).
4. Bookkeeping commit; log `epic_context_compiled`.

Run it after sprint planning and the Story X.0 gate (both may edit planning artifacts) and again before any plan spawn whenever a file under `planning_artifacts` is newer than the cache (Rule 5 amendments, X.0 insertion, user edits). With a valid, committed cache the plan spawn reuses it and never recompiles.

### Retrospective Review & Story X.0 Creation (Critical Gate)

After sprint planning, before building the story list, review the previous epic's retrospective and create a cleanup story. Mandatory — closes the feedback loop between retrospectives and sprint planning.

1. Calculate previous epic number (N-1 if processing N).
2. Search for `_bmad-output/implementation-artifacts/epic-{N-1}-retro-*.md`. If multiple, latest by mtime; tie-break by lexicographic filename. Log which file was selected.
3. If a retrospective exists, extract: action items (status: completed / in-progress / not addressed), deferred review findings, preparation tasks for current epic.
4. Also take the ledger's non-terminal slice (`LEDGER slice all`, plus `LEDGER show DW-n` for any entry whose summary is not enough — never the whole file), and the `action_items:` block of `sprint-status.yaml` (`SPRINT_PLAN status` → `open_action_items`; v6.11 retrospectives append their action items there with `status: open`).
5. **Close every ledger item — this is a CLOSE, not a relabelling (Rule 17).** The input is every non-terminal entry owned by `burndown` (including the decision sheet's `note=to_x0` deferrals), by `none`, or by a story key of an earlier epic; an entry already routed to a story of this or a later epic is planned work and stays where it is. Each such entry gets exactly one trailer line, `by=x0`: **resolve** (`status=resolved-by:<story> note=<evidence>` — it was fixed since it was filed); **own** (`owner=<N>-0-… ` for items Story X.0 will do, or `owner=<a specific story key of THIS epic whose gate genuinely fails while the item stands>`); or **terminal** (`by-design` / `wontfix-theoretical` / `wontfix-accepted note=reopen_if=…` / `dropped`). Undecided `decision-pending` items (normally only those the previous epic's decision sheet deferred here as `owner=burndown note=to_x0:…`) are batched to the user in one sitting and leave this step decided (*Unattended:* classified instead; owner-only entries wait with `note=inbox=` — U8); human-owned items live in `action_items`, not the ledger. There is no "defer with rationale" and nothing may stay `owner=burndown` or `owner=none`. Default-INCLUDE in X.0 every entry that is (a) trivial + low fix-risk by now, or (b) has two or more `occurrence=` lines — excluding one requires written justification in its note (the burden flips to exclusion). Open action items are included unless the user explicitly defers them. If the previous epic's burn-down gate left more than `ledger_cap` entries owned by this epic's keys, X.0 is a mandatory fix-or-close pass over all of them. The X.0 key enters the tracker only at step 6a, and `ledger.sh` refuses an owner the tracker does not know: write every other disposition here, and the `owner=<N>-0-…` trailers right after 6a's `SPRINT_PLAN generate`.
6. Create Story X.0 in four steps (the tracker script rejects `--set` for any key not derived from the epic files, so the story must exist in `epics.md` first):
   - **6a:** Insert `### Story {N}.0: Epic {N-1} Deferred Cleanup` as the first story under Epic {N} in `epics.md`, in the same heading/format as the sibling stories, with a one-paragraph description and the included items as acceptance bullets (one per triaged item, naming its source). Re-run `SPRINT_PLAN generate` (no `--set`) so the key `{N}-0-epic-{N-1}-deferred-cleanup` appears as `backlog`; confirm it in `new_entries`.
   - **6b:** Re-run the epic-context pre-warm (the `epics.md` edit made the planning artifacts newer than `epic-{N}-context.md`); bookkeeping commit (epics.md + tracker + context + log).
   - **6c:** Spawn the normal **plan** stage with the story key + `Halt after planning.` — Story X.0 is now an ordinary epic story. Validate the spec as usual; append the full triage table under its `## Design Notes` (rows of `Item (DW-n where applicable) | Source (retro | ledger | action_items) | Disposition (resolved-by | own:<key> | terminal:<status>)`; header notes which Epic N-1 the triage covers + the date). Story X.0 closes its own owned entries at its `ledger_adjudicated` gate like any other story.
   - **6d:** `SPRINT_PLAN generate --set {N}-0-epic-{N-1}-deferred-cleanup=ready-for-dev`; bookkeeping commit. Under the parallel kit, the orchestrator treats an `epics.md` change whose only diff is an appended Story X.0 as hash-exempt (re-record `epics_md_hash` without a re-approval).
7. Skip Story X.0 ONLY if all three sources are empty. If the retro is missing but the ledger or action items have entries, do NOT skip — execute steps 5-6 from those sources. If nothing is skipped, the pre-warm in 6b doubles as the epic's initial pre-warm; if X.0 is skipped, run the pre-warm now.
8. Log `retro_review_complete source_retro=<path-or-empty> resolved=N owned=N terminal=N dropped=N load_before=N load_after=N cap=N` or `retro_review_skipped`. Closing a previous epic's action item (`sprint_status.py update --set-action-status`) happens only with the user's confirmation, at the next retrospective — never silently here.

### Source Control Branching (Critical Gates)

These fire at the very start of each epic, before sprint planning or story work. Apply uniformly to the parent repo and every submodule in `.gitmodules`. For multi-repo projects (separate child repos rather than git submodules), the user enumerates affected repos up-front; the rules apply to each.

**Precondition — clean working tree.** Before any branching rule, the lead runs `git status --short` against the parent repo and every submodule and halts on dirty state. Non-negotiable. (If the IDE file-sync toggle below applies, run this clean-tree check FIRST; the toggle is then applied only transiently around each branch-changing operation and restored immediately after, so between operations `.vscode/settings.json` is back at its original value and the tree is clean. Never stage the transiently-toggled file into any commit.)

#### IDE file-sync toggle (conditional — IRIS/ObjectScript projects only)

Some IRIS/ObjectScript projects run an IDE extension that bidirectionally syncs the workspace with a live IRIS server (the VSCode-ObjectScript extension with `objectscript.conn.active: true` in `.vscode/settings.json`). When a git operation that **changes branches** rapidly rewrites the working tree, that sync fights the operation — it pushes the new tree to the server, the server recompiles and re-formats, then pushes the formatted files back to disk — producing phantom drift (e.g. spurious whitespace-only diffs), compile cascades, or partially-applied state that has to be reverted before the next commit.

This subsection is self-contained — it does not depend on any project's CLAUDE.md or rules files.

**Applicability — detect once, up front (after the initial clean-tree check):**

1. If `.vscode/settings.json` does NOT exist, or has no `objectscript.conn` block, or `objectscript.conn.active` is not `true` → the extension isn't actively syncing (or this isn't an ObjectScript/IRIS project). **SKIP this toggle entirely; it never applies.** Non-ObjectScript projects are never affected.
2. Otherwise the toggle is in effect for this run: record the original value (almost always `true`) so each window can restore it. The sync stays ENABLED except inside the branch-operation windows defined below.

**Scope — wrap ONLY branch-changing operations; do NOT disable the sync for the whole epic.** Disable the sync immediately before, and restore it immediately after, each git operation (or contiguous group of operations) that moves HEAD and rewrites the working tree: `checkout`, `checkout -b`, `merge`, `pull` (when it advances HEAD), `rebase`, `reset --hard`, `cherry-pick`, `stash apply`/`pop`, `revert`, `submodule update`. Within `/epic-cycle` there are two such windows (plus any resume-time checkout):

- **Window A — epic-start branching (SC-1 + SC-2 + checkout):** wrap the feature-branch checkout/creation, the epic-branch creation, and the `git checkout {TICKET}-epic{N}`. Restore the moment the epic branch is checked out.
- **Window B — SC-4 end-of-epic merge:** wrap the `checkout <feature>` / `pull` / `merge --no-ff` / branch deletes (per affected repo, submodules-first). Restore the moment the merge + deletes complete.

Read-only git ops (`status`/`diff`/`log`) and `add`/`commit`/`push`/`fetch` do NOT change branches and are NOT wrapped. The per-story pipeline stays on `{TICKET}-epic{N}` and only commits/pushes, so it runs with the sync **enabled**, as normal. Between windows the sync is on and `.vscode/settings.json` sits at its original value — clean — so the clean-tree check and the per-story commits never see a toggle-induced modification.

To wrap a window: set `objectscript.conn.active: false` → perform the branch operation(s) → set it back to the recorded original value.

**Restoration is MANDATORY and is the #1 failure mode of this rule.** Each window's restore must run — on success, on halt, on failure, on merge conflict, or on user cancellation. Treat each window like a `try/finally`: the restore belongs on the always-runs path, never only after the success path. The transiently-toggled `.vscode/settings.json` must never be `git add`-ed into any commit; if a window overlaps a commit (the SC-4 merge commit), verify settings.json is not staged. If you discover a window's restore was missed, restore immediately and tell the user the workspace was left disconnected from IRIS and for roughly how long.

**Crash recovery.** If a prior run crashed mid-window, `.vscode/settings.json` may be left at `active: false`. On a fresh run, if detection finds `active: false` where a live connection is expected, surface it to the user before proceeding — it may be an orphaned toggle from a crash, or an intentional manual disconnect; only the user knows.

This toggle is purely an IRIS/ObjectScript convenience; it has no effect on, and must not be applied to, non-ObjectScript projects.

#### Rule SC-1 — Feature branch verification (epic-start)

The lead checks for a feature branch in every affected repo. Branch name follows the project's configured pattern (see "Tracker format flexibility" below); default is `feature/{TICKET}_{Description}`.

- **If the feature branch exists** (locally or fetch-discoverable on remote): check it out, verify it's up to date with its remote, continue.
- **If the feature branch does NOT exist:** STOP and ask the user:
  1. Should the feature branch be created?
  2. The exact `{TICKET}` (validated against the project's `ticket_format` regex).
  3. The exact `{Description}` (validated against `description_format`).
  4. Which root does it branch from? Default precedence: `origin/develop` → `origin/main` → `origin/master`. The lead surfaces candidates the repo actually has and asks the user to confirm.

  **Root freshness check (mandatory, per affected repo, BEFORE creating anything):** `git fetch origin`, then compare the chosen remote root against its local counterpart: `git rev-list --left-right --count origin/{root}...{root}`.
  - **Local ahead (0 behind / N ahead):** STOP. The local branch carries N commit(s) origin does not have — basing on `origin/{root}` will REMOVE from the working tree every tracked file those commits added (this has deleted a project's entire `.claude/` + `_bmad/` tooling mid-session). Show the commits (`git log --oneline origin/{root}..{root}`) and ask the user: (a) push local `{root}` to origin first, then base on `origin/{root}` (recommended); (b) base on local `{root}` instead; (c) base on `origin/{root}` anyway, explicitly acknowledging the listed content disappears from the working tree. Log `root_freshness_resolved` with the choice.
  - **Diverged (both counts nonzero):** HALT and surface both commit lists — the user reconciles the trunk before any branching (same doctrine as a resume INTEGRITY_ERROR; never pick a side silently).
  - **Equal, or local behind:** proceed with `origin/{root}` (the normal case).

  On user authorization: validate the resulting branch name, then `git fetch origin && git checkout -b <validated-name> <confirmed-root> && git push -u origin <validated-name>` per affected repo. Log `feature_branch_created`.

Merging the feature branch into `develop` / `main` is OUT of scope for `/epic-cycle` — that's a PR-review / code-owner workflow. `/epic-cycle` creates and merges INTO the feature branch only.

##### Tracker format flexibility (per-project configuration)

The default assumes a JIRA-style tracker. Real projects use JIRA, Linear, GitHub Issues, Azure DevOps, or no tracker at all. The lead reads the project's branch-naming config from the first of these locations to exist:

1. `_bmad/custom/branch-naming.yaml` (preferred)
2. A `## Branch naming` section in CLAUDE.md
3. The defaults below

**Config schema:**

```yaml
# _bmad/custom/branch-naming.yaml
feature_pattern: "feature/{TICKET}_{Description}"  # template with {TICKET} and {Description}
epic_pattern: "{TICKET}-epic{N}"  # template for epic branch; {TICKET} reused from SC-1, {N} is epic number
ticket_format: "^([A-Z]+-\\d+|SPIKE|EXPLORE|REFACTOR)$"  # regex; tracker IDs OR named exceptions
ticket_required: true   # if false, {TICKET} may be empty
description_format: "^[a-z][a-z0-9-]{2,60}$"  # kebab-case, 3-60 chars, starts with letter
separator: "_"          # between TICKET and Description in the feature template
```

**Defaults if no config:**

| Field | Default |
| --- | --- |
| `feature_pattern` | `feature/{TICKET}_{Description}` |
| `epic_pattern` | `{TICKET}-epic{N}` |
| `ticket_format` | `^[A-Z]+-\d+$` (JIRA/Linear-style) |
| `ticket_required` | `true` |
| `description_format` | `^[a-z][a-z0-9-]{2,60}$` |
| `separator` | `_` |

**Validation at the SC-1 user prompt:**

1. Ask user for `{TICKET}` and `{Description}`, showing an example derived from the configured pattern.
2. Validate each against the configured regex.
3. If invalid: surface the mismatch with the offending regex shown. Offer: (a) re-enter, (b) override and use as-is (logs a warning), (c) update the project config.
4. If valid: render the branch name and confirm before creating.

**Ticketless work.** The default `ticket_format` allows `SPIKE`, `EXPLORE`, `REFACTOR` as named exceptions (e.g., `feature/SPIKE_audio-latency-probe`). Projects can broaden or narrow this list.

**Branch-name safety (non-negotiable).** Refuse any branch name with spaces, shell metacharacters (`*`, `?`, `[`, `]`, `;`, `&`, `|`, `<`, `>`, `$`, `` ` ``, newline), or git-reserved sequences (`..`, `@{`, leading `-`, trailing `.`). These guards are independent of the project config.

#### Rule SC-2 — Epic branch verification (epic-start)

After the feature branch is in place, the lead checks for an epic branch in every affected repo. The branch name follows the project's configured `epic_pattern` (see "Tracker format flexibility" below); default is `{TICKET}-epic{N}` (e.g., `PROJ-1234-epic1`). The ticket prefix is required so that multiple projects committing to the same repository under different tickets do not collide on a shared `epic1` / `epic2` namespace.

`{TICKET}` is the same value validated and recorded by SC-1 for the feature branch — the lead reuses it; do not re-prompt.

- **If the epic branch exists** (local or remote): check it out, continue.
- **If the epic branch does NOT exist:** create deterministically off the feature branch — no user prompt; the name is derived from the epic number and the SC-1 ticket. `git checkout <feature> && git pull && git checkout -b {TICKET}-epic{N} && git push -u origin {TICKET}-epic{N}`. Log `epic_branch_created`.

**Resume semantics:** mid-epic resume should find `{TICKET}-epic{N}` already in place. If the lead is resuming with prior `committed` entries in the cycle log but `{TICKET}-epic{N}` is missing locally AND on remote, that is a workspace-integrity error — halt and surface. Do NOT silently re-create.

**Ticketless projects.** If the project's `ticket_required` is `false` and no ticket was supplied at SC-1, the default `epic_pattern` resolves to a leading-hyphen name (`-epic1`), which is git-invalid. Projects in this mode MUST override `epic_pattern` (e.g., to `epic{N}` for single-project repos, or to `{DESCRIPTION}-epic{N}` to derive the prefix from the feature branch description). The lead validates the resolved epic branch name against the same branch-name safety rules as SC-1 and halts on failure.

#### Rule SC-3 — Commits go ONLY to the epic branch

Every commit during the epic cycle lands on `{TICKET}-epic{N}` in the affected repo. The lead asserts `git branch --show-current == "{TICKET}-epic{N}"` immediately before every `git commit` and halts on mismatch. The one exception is SC-4's merge record (`chore(epic-N): record merge`), committed on the feature branch after the merge, where the assertion names `<feature>` instead. Applies to submodules (each submodule's HEAD must be on its own `{TICKET}-epic{N}` before the parent's `git add <submodule-path>`).

Push frequency: per story, to the epic branch's remote.

#### Rule SC-4 — End-of-epic merge gate (user decision point)

After the retrospective gate (whether opted in or not), the lead pauses (*Unattended:* no question — `when_green`, doctrine U9):

> "Epic {N} is complete. Merge `{TICKET}-epic{N}` into the feature branch and delete the epic branch (local + remote) in every affected repo?"

**Precondition — the branch head's CI is green.** `gh run list --branch {TICKET}-epic{N} --commit $(git rev-parse HEAD) --json databaseId,status,conclusion -L 1` (or the equivalent for the project's CI): `success` (or `ci=none` for a project without CI) is required; `failure` → log `merge_gate_blocked reason=ci_red run=<id> job=<name>` and surface the failing job instead of the merge question — the branch is not mergeable while it is red, whatever the local suite says (field report 2026-08-30: a branch red for ~16 h passed two full story pipelines); `pending` → wait; `unavailable` → say so and ask the user to confirm CI by hand (*Unattended:* as chosen at launch — U2). Log `ci_green_verified run=<id> head=<sha>` on success.

Before asking, (1) surface the burn-down gate's result for this epic (`ledger_burndown_*` entry: what was resolved, made terminal, chartered into the burn-down story, the overflow re-owned to which next-epic keys, and the drain ratio), and (2) run the **epic-close decision sheet** (see "Ledger Drain"): every `escalated` and `decision-pending` entry, presented once with a recommended disposition, the user's answers written as `by=merge_gate` trailer lines and logged `decision_sheet_recorded` — the burn-down gate set these aside precisely because only the user can decide them. If this is the range's FINAL epic, follow the merge with the ledger sweep: `LEDGER slice all` — every remaining non-terminal entry must already be owned by a named story key outside the range; present the list with a recommended disposition for each — the end-of-project triage, sized for one sitting.

If **yes**, first write the write-ahead marker on the epic branch — log `epic_merge_started head=<the CI-verified head> feature=<feature>` and bookkeeping-commit it (no separate push: it would start another CI run for a log line, and the merge carries it to origin) — then execute the merge — submodules-first, then the parent (mirrors per-story Submodule Commit Order to avoid broken pointers on the feature branch's remote). This is "Window B" for the IDE file-sync toggle: if that toggle applies, set `objectscript.conn.active: false` before the first `checkout` below and restore it immediately after the deletes complete (mandatory, `try/finally`; restore even if the merge conflicts and halts):

For each affected repo, ordered submodules-first:

1. `git checkout <feature>`
2. `git pull origin <feature>` (in case it moved while the epic was in flight)
3. `git merge --no-ff {TICKET}-epic{N} -m "Merge {TICKET}-epic{N}: <one-line summary>"` (preserves the epic branch's commit graph)
4. `git push origin <feature>`
5. `git branch -d {TICKET}-epic{N}` (refuses if not fully merged — the safety we want)
6. `git push origin --delete {TICKET}-epic{N}`

Then record it: once the merge is verified on origin, append `epic_merged_to_feature` to this epic's cycle log — the merge brought the log onto the feature branch — and commit it there as `chore(epic-N): record merge`, push, and verify. This is the one sanctioned commit on the feature branch (SC-3); without it the next epic's clean-tree check halts on the uncommitted log line.

If submodules are involved, the parent's merge step (3) brings in submodule pointer updates that already exist on `{TICKET}-epic{N}` from per-story commits. Because the submodules' own feature branches were merged in the preceding pass, those pointers now resolve cleanly on the submodules' remotes. Verify with `git submodule status` before the final parent push.

If **no**, log `epic_merge_skipped reason=<short>` and leave `{TICKET}-epic{N}` intact.

#### Rule SC-5 — Epic re-open recreates the epic branch

If an epic is re-opened (e.g., the next epic's retrospective surfaces work that belongs on the prior epic, or the user explicitly reopens), the `{TICKET}-epic{N}` branch must be recreated.

- If the prior `{TICKET}-epic{N}` was merged and deleted: branch a new `{TICKET}-epic{N}` off the current feature-branch HEAD (picks up any interim feature-branch progress).
- If the prior `{TICKET}-epic{N}` was never merged and still exists: check it out as-is; do not branch a parallel `{TICKET}-epic{N}`.

Log `epic_branch_reopened reason=<short>`.

#### Rule SC-6 — NEVER commit directly to `main`, `master`, or `develop`

The lead refuses any commit (story, retrospective, hotfix, anything) when the current branch is `main`, `master`, or `develop` in any affected repo. Absolute defensive default. If the user explicitly directs a direct-to-trunk commit (emergency hotfix outside the epic cycle), that's OUT of scope — the user performs it manually.

Pairs naturally with remote branch protection on trunks.

#### Rule SC-7 — If unsure where to commit, STOP and ask

Branching state can drift across sessions. If the lead cannot confidently identify the right branch to commit to — multiple feature branches with similar names, missing epic branch mid-resume, ambiguous parent between `develop` and `main`, etc. — STOP and ask the user. Do not guess. (*Unattended:* stop the run — U5.)

#### Rule SC-8 — Parallel epics on the same feature branch

Multiple `/epic-cycle` runs may execute concurrently against the same feature branch — typically when different agents drive Epic A and Epic B simultaneously.

**Per-agent isolation: one working tree per agent.** Git only allows one HEAD per working directory. Concurrent agents on different epic branches require either:

- `git worktree add <path> {TICKET}-epic{N}` (recommended) — separate worktrees share the same `.git` object store. Cleanup: `git worktree remove <path>` after the epic merges.
- Separate full clones (simpler but heavier).

A single working directory running two agents on different epic branches is not supported — it requires constant branch-switching that corrupts file state.

**Branch creation under parallelism.** Each agent runs Rule SC-2 independently. `{TICKET}-epic{N}` is created off the feature branch's current HEAD at the moment that agent starts. If Epic A started earlier and a hotfix or sibling-epic merge landed on feature in between, Epic B's `{TICKET}-epic{N}` branches off a newer feature HEAD. That's fine — the `--no-ff` merge at SC-4 handles three-way reconciliation.

**Per-story commits.** Independent branches, independent remotes — no race. Each agent's pre-commit branch assertion is per-agent and per-worktree.

**Merge serialization.** The `git push` to feature is a single-writer point on the remote, so SC-4 must serialize across agents:

- **Coordinated:** each SC-4 sequence starts with `git pull origin <feature>`. If Epic A already merged, Epic B's pull picks that up before its own merge runs. Standard three-way merge from there.
- **Near-simultaneous:** the user picks the order. The second agent re-pulls feature after the first lands.

**Conflict handling at merge time.** (*Unattended:* integrate forward first and resolve per the policy's `gates.merge_conflicts` — doctrine U9.) If `git merge --no-ff {TICKET}-epic{N}` produces conflicts (against another epic's already-merged work, a hotfix on feature, or any other drift since branching), the lead STOPs and surfaces to the user. Auto-resolution is forbidden — git's conflict heuristics can silently drop intentional changes. The user resolves in the working tree, then signals the lead to continue.

Submodules are independent under this rule too — each submodule's `{TICKET}-epic{N}` merges into its own feature branch sequentially; submodule conflicts require user resolution before the parent's submodule pointer is bumped.

**Out-of-scope coordination:** if two parallel epics' planning artifacts overlap (both stories' ACs touch the same files), that's a sprint-planning conflict the user resolves before parallel execution. Rule of thumb: if two epics' Files-to-Modify tables overlap by more than ~20%, run sequentially.

#### Sub-repository vs submodule terminology

"Submodule" = git submodule (registered in `.gitmodules`; `git -C <path>` operates on it as a child repo). "Sub-repository" = a non-submodule child repo under one umbrella (separate clones the umbrella project orchestrates). Rules SC-1 through SC-8 apply to both. If ambiguous, that's a SC-7 STOP-and-ask trigger.

### Resume Semantics (Critical)

`/epic-cycle` is designed to be resumable across interrupts, context exhaustion, explicit pauses, and clarification gates spanning days. A later session must pick up exactly where the prior one left off — no re-doing work, no skipping work.

#### Resume-mode detection (epic-start)

The lead determines mode before running SC-1 / SC-2:

1. Read `_bmad-output/implementation-artifacts/cycle-log-epic-{N}.md` **from the epic branch**, where it lives until the merge — `git show {TICKET}-epic{N}:<path>`, else `git show origin/{TICKET}-epic{N}:<path>` after a fetch — and from the feature branch when neither branch exists (a merged epic's log is there). The working tree's copy is the log only when the epic branch is checked out.
2. Get this epic's story states from `sprint-status.yaml`: `SPRINT_PLAN status` (per-status counts, `recommendation.story_key`, `open_action_items` — it has no per-story list) and `TRACKER_READ` for each story's own status; run `SPRINT_PLAN validate` first if the file's integrity is in doubt. Also read each `spec-{N}-*.md` frontmatter `status` — it is `bmad-build-auto`'s own durable state and the tie-breaker when the log and the tracker disagree.
3. For each affected repo, check whether `{TICKET}-epic{N}` exists locally and on the remote.

| Cycle log | `{TICKET}-epic{N}` local | `{TICKET}-epic{N}` remote | Mode | Action |
| --- | --- | --- | --- | --- |
| Missing / empty | Missing | Missing | **FRESH** | Run SC-1, SC-2. Create branches. |
| Missing / empty | Exists | Exists | **AMBIGUOUS** | If the branch has no commits beyond the feature branch (`git rev-list --count <feature>..{TICKET}-epic{N}` is 0), adopt it as FRESH — a crash between SC-2 and the first entry. Otherwise halt; ask user whether to (a) adopt existing branch and start logging against it (existing commits accepted as-is), (b) start a new epic under different `N`, or (c) inspect manually. |
| Missing / empty | Exists | Missing | **UNPUSHED_NEW** | No commits beyond the feature branch → push it (`git push -u origin {TICKET}-epic{N}`), then FRESH. Otherwise AMBIGUOUS. |
| Missing / empty | Missing | Exists | **REMOTE_NEW** | No commits beyond the feature branch → `git fetch && git checkout {TICKET}-epic{N}`, then FRESH. Otherwise AMBIGUOUS. |
| Has entries | Exists | Exists | **RESUME** | Compute resume point from the log. |
| Has entries | Missing | Exists | **REMOTE_ONLY** | `git fetch && git checkout {TICKET}-epic{N}`; then RESUME. |
| Has entries | Exists | Missing | **LOCAL_ONLY** | Halt; ask user (remote deletion intentional?). |
| Has entries, the last gate entry `epic_merged_to_feature` | Missing | Missing | **COMPLETE** | The epic is merged and its branch deleted. Skip it. |
| Has entries, an `epic_merge_started` with no `epic_merged_to_feature` | any | any | **MERGING** | See "Resume across the end-of-epic gates". |
| Has entries | Missing | Missing | **INTEGRITY_ERROR** | Halt loudly. Log claims work that branches no longer carry. |

Detection runs per affected repo. Different repos may legitimately be in different modes (parent RESUME, submodule FRESH).

#### Resume-point computation (within-epic)

An attended run whose log holds `story_set_aside` or `story_parked` entries without a later `story_unparked` or `story_dropped`, or whose unattended run state holds patches (`<state>/patches/`), reads the doctrine's U4, U10 and U11 first — never re-implement a set-aside story over its patch, never skip a held story's saved work. *Unattended:* the doctrine's U11 resume semantics override this subsection and the integrity checks' "surface to user": parked and set-aside stories are skipped, an open clarification is ruled, a dead stage's tree is saved as evidence and the story re-run from its logged `rerun_from`, an integrity error stops the epic.

For a repo in RESUME mode:

1. Bucket cycle-log entries by story ID; highest-stage entry per story is its resume anchor.
2. Earliest story whose anchor is not `committed` = resume point. Work resumes at the next pipeline stage after the anchor.
3. If a story has a `<stage>_clarification_requested` without that stage's subsequent success entry (`story_created` / `dev_complete` / `qa_complete` / `cr_complete`), the resume point is "answer the clarification + re-spawn" (re-dispatch protocol for build-auto stages). The lead surfaces the question before spawning.
4. **Reconcile with the spec's own state** (`bmad-build-auto` writes it durably; the log can lag by one crash): spec `ready-for-dev` + no `story_created` → the plan finished, the log entry was lost — write it, continue at validation. Spec `done` + `build_sha` (the commit touching the spec) reachable on the epic branch + no `dev_complete` → write the entry, harvest `deferred:`, continue at AD verifications; never re-spawn the implement stage over finished work. Spec `in-progress`/`in-review` + dirty tree is **ambiguous on its face**: it is what a dead build-auto leaves behind AND what a healthy, still-running one looks like (`in-review` is its mid-self-review state; a working agent's tree is dirty because it is working). Liveness is a property of sessions, not of files — a stage agent lives exactly as long as the session that spawned it — so branch on ownership: (i) **this lead is a fresh session with no orchestrator above it** → no stage of this pipeline can be alive; it died mid-run: surface the uncommitted diff to the user (keep and re-spawn per the re-dispatch protocol — its resume table continues from `in-progress` — or discard); never auto-clean. (ii) **Orchestrator mode** → the worktree belongs to a runner; the orchestrator NEVER diagnoses a worktree's in-flight state from artifacts. Check the open `stage_spawned` entry, then the runner's liveness (`ListAgents` — which reports process liveness only: sufficient for runner alive/dead, NOT for whether a stage agent has returned, since a resumed agent shows `running` after returning; Rule 18); if the runner is alive, ASK it whether that stage's `Agent` call has returned — it is the only party that knows — and let it act; only a runner that is confirmed dead (whose stage agents are therefore dead too) makes the artifact reading valid, and then the re-spawned runner performs this step, not the orchestrator. If liveness cannot be established either way, HALT and surface — an ambiguous liveness state is a workspace-integrity error, not a judgment call. A re-spawn onto a live stage puts two agents in one worktree (incident 2026-08-28: ~30 minutes of concurrent patching). Spec `blocked` → the clarification is still open.
5. `epic_context_compiled` present but `epic-{N}-context.md` missing or older than a planning artifact → re-run the pre-warm before the next plan spawn.

#### Cross-stage integrity checks (before resuming any stage)

1. **`committed sha=X` and `build_sha=Y` reachable on `{TICKET}-epic{N}`.** For every `committed` / `dev_complete` entry, verify `git merge-base --is-ancestor <sha> {TICKET}-epic{N}`. Failure = branch drift; halt.
2. **Local and remote `{TICKET}-epic{N}` HEADs match.** `git rev-parse {TICKET}-epic{N}` vs `git rev-parse origin/{TICKET}-epic{N}` (post-fetch). Diverged non-fast-forward = halt. Strictly ahead = log `resume_local_ahead`, push. Strictly behind = fetch + fast-forward.
3. **`sprint-status.yaml` agrees with cycle log and specs.** Story marked `done` in YAML with no `committed` entry (or vice versa), or `review` in YAML with a spec not at `done` = divergence; surface to user. Exemption: a story whose highest anchor is `cr_complete` with a Fix Pack or rework in flight legitimately reads `done` (code review wrote it) or `in-progress` before its `committed` entry exists — resume at the rework/smoke step, not as divergence. `SPRINT_PLAN validate` must report `valid: true`.
4. **Submodule pointer consistency.** If `{TICKET}-epic{N}` on the parent recorded a submodule pointer at commit S, the submodule's `{TICKET}-epic{N}` must contain S. Mismatch = INTEGRITY_ERROR.
5. **Clean tree before the first build-auto spawn** (Rule 16) — a dirty tree at resume is evidence (check 4 in resume-point computation), never something to silently commit or discard.
6. **No `committed` entry is left at `ci=pending`.** Resolve each one (`gh run view <id> --json status,conclusion`) and log `ci_resolved` before resuming any stage; a `failure` is handled per "CI Is a Gate" before new work starts.

#### Resume interactions with parallel epics (SC-8)

If Epic A is being resumed and Epic B merged to feature in the interim, Epic A's `{TICKET}-epic{N}` is unaffected — reconciliation happens at Epic A's eventual SC-4 merge. If Epic A's own `{TICKET}-epic{N}` was force-pushed by another contributor during the pause, Check 1 fails and the lead halts.

#### Resume across the Story X.0 / retro-review gate

- `retro_review_complete` or `retro_review_skipped` entry → already done, skip.
- No such entry → run the gate.
- Entry present but Story X.0 has no `story_created` → treat Story X.0 as the first incomplete story and resume there.

#### Resume across the end-of-epic gates

- Last story `committed` but no `ledger_burndown_*` entry → resume at the burn-down gate (Rule 17); never write `epic_status_done` without it. A story with `cr_complete` but no `ledger_adjudicated` resumes at adjudication, never at smoke.
- Log shows `epic_status_done` but no `epic_retro_*` entry → run the model-tier checkpoint first if no `model_tier_checkpoint` entry exists for this epic (every project; the evaluation is idempotent — re-running it is harmless); then resume at the retrospective question.
- `epic_merge_started` but no `epic_merged_to_feature` → the merge was interrupted. If the logged `head` is an ancestor of `origin/<feature>`, the merge landed: finish only what is missing (the branch deletes, the record commit) and write the entry — never merge again. Otherwise, if the epic branch still exists, resume at the merge question; if neither, INTEGRITY_ERROR.
- `epic_retro_*` present but no `epic_merged_to_feature` / `epic_merge_skipped` → resume at SC-4: the decision sheet first if no `decision_sheet_*` entry exists for this epic (Rule 17 (5) — its trailer lines are idempotent to re-present; entries already `by=merge_gate` are shown as decided), then the merge question.
- Both gates are user decision points; re-asking after a long pause is correct behavior. (*Unattended:* neither asks — U9.) A run interrupted mid-retrospective resumes by re-asking the retrospective question (the skill's own output artifacts show how far it got); never auto-complete a half-finished retro.

#### Resume across Sprint Planning

Sprint planning is idempotent — `sprint-status.yaml` is regenerated each run. Always re-run on resume; the skill is a no-op when the YAML is current. The cycle log is append-only, so the new `sprint_planning_complete` entry coexists with the prior one (highest-timestamp wins).

#### Workspace-integrity errors are NOT auto-recoverable

INTEGRITY_ERROR or any cross-stage-check failure must halt. Auto-recovery paths ("the branches must have been pruned; recreate them") lose work. Only the user knows whether missing state was intentional (cleanup) or accidental (mistake).

When halting on an integrity error, surface:

1. What the log says happened (story IDs, stages, recorded shas).
2. What the workspace shows (branches present/missing, HEAD shas).
3. The specific check that failed.
4. Options for the user (re-create from log, abandon log, inspect manually).

Never guess.

#### Resume vs starting a new epic

If the cycle log for epic N is missing but cycle log for epic N-1 exists and shows N-1 completed, that's a normal FRESH start for epic N. SC-1 / SC-2 create the new branch. The retro-review gate triages N-1's deferred items into Story N.0.

If cycle logs for both epic N and epic N+1 are present with entries, two epics are in flight (parallel per SC-8). Resume each independently in its own worktree.

### Sprint Planning Per Epic (Critical Gate)

Before processing any stories for an epic:

1. Execute `/bmad-sprint-planning` directly via the `Skill` tool, **headless** — the skill defines no flag for this; pass the words `headless, intent sprint-planning` as its argument (its `## Headless Mode` section keys off being "invoked headless"). In v6.11 the skill opens with the implementation-readiness gate (it absorbed `bmad-check-implementation-readiness`), then hands the mechanical work to `sprint_plan.py generate`.
2. Parse the JSON block it ends with: `{status, intent, gate, status_file, findings, warnings}`.
   - `gate: FAIL` or `status: blocked` (duplicate epic versions, unreconciled orphans, an unconfirmed fix) → STOP; surface `findings` and the saved `implementation-readiness.md` if written. The user resolves interactively. (*Unattended:* at launch this is resolved with the owner; at a later epic's start, FAIL stops that epic — U8.)
   - `gate: CONCERNS` → surface the findings once, log them, continue.
   - `gate: PASS` → continue.
3. `sprint-status.yaml` is now current, all stories tracked, status mismatches caught; run `SPRINT_PLAN validate` as a belt-and-braces check.
4. Log `sprint_planning_complete gate=<PASS|CONCERNS>`. Bookkeeping commit if the tracker changed.

Re-runs are idempotent — the script's merge is preserve-never-downgrade.

### Retrospective Per Epic (User Decision Point — the human-in-the-middle exception)

*Unattended:* the retrospective runs headless (`/bmad-retrospective -H <epic>`) and its verdict is read from its document — doctrine U9. This section's "never pass `-H`" governs attended runs.

The retrospective is the ONE deliberate human-in-the-middle exception to the autonomous loop. Its value is the human's judgment about what just happened — automating it defeats its purpose. Concretely: never spawn it as a subagent, never pre-answer its party-mode elicitation or WAIT points (Rule 9's pre-answer protocol explicitly exempts it), and never pass its `-H` / `--headless` flag.

After all stories in an epic complete:

1. Announce: "Epic X is complete. Run a retrospective before moving to the next epic? (yes/no)"
2. **Wait for the user's response.** If the user is away, the run simply pauses at this gate — resume semantics pick it back up later; do not time out into a default answer.
3. **Yes:** execute `/bmad-retrospective` directly via the `Skill` tool, fully interactive, passing the explicit epic number (its `sprint_status.py detect-epic --epic {N}` gate then lists unfinished stories; if the range ended early, the machine verdict `rejected` is expected and the human overrides). Never pass `-H`. Log `epic_retro_complete verdict=<accepted|accepted-with-open-items|rejected>` when it finishes (the skill itself marks `epic-{N}-retrospective: done` and appends its action items to `sprint-status.yaml`). Bookkeeping commit.
4. **No:** log `epic_retro_skipped reason=user_declined`; continue.

`AskUserQuestion` always elicits the human regardless of `bypassPermissions`. The lead executes the skill — do NOT spawn an agent for it (spawned subagents cannot reliably surface elicitation).

### Lead Validates the Story Spec (Critical Gate)

Story creation is the Opus **plan** spawn of `bmad-build-auto` (`<story-key> Halt after planning.`). The gate is what happens between that spawn and the implement spawn: the lead reads the spec it produced (`spec-<key>.md` — capture the path from the agent's `SPEC:` line and confirm by globbing `spec-<epic>-<story>-*.md`) and validates it before any code exists. This replaces the pre-6.11 "lead creates the story file" gate; the race-ahead concern is gone (build-auto never advances to another story), the validation is not.

**Integration AC validation.** Read the spec's Tasks & Acceptance, I/O & Edge-Case Matrix, and Design Notes: does this story introduce a service, module, or component that later stories will consume? Indicators: a new file under `services/` or `lib/`; a new exported class/factory/module; a `Consumed-by:` list; an AC describing a public surface other stories will call.

If yes — the story is service-introducing — it MUST EITHER (a) contain at least one Integration AC of the form "consumer X reads from this service and produces observable effect Y," OR (b) carry an explicit note under `## Design Notes` stating "No consumers in this story; the first consumer will be Story X.Y." (the Rule 1 escape clause for services with no consumers yet in the epic). If neither is present, pause for the user (*Unattended:* option (a) once, then (b) only with a named consumer, else stop the story — U5; never (c)):

> "Story <id> introduces <service-name>. No Integration AC and no 'no consumers yet' declaration found. Choose: (a) re-plan — the lead adds the Integration-AC requirement inside the spec's `<intent-contract>` block, sets frontmatter `status: draft`, commits, and re-spawns the plan stage on the spec path with `Halt after planning.` (re-dispatch protocol — re-spawn on the spec path: a story-key re-spawn resumes a `draft` spec in place but creates `spec-<key>-2.md` at any other status); (b) name the future consumer story and add the 'No consumers in this story; the first consumer will be Story X.Y.' note to Design Notes (lead edit; commit); (c) proceed without (producer-consumer wire-up defects can ship green)."

**AD mapping.** Note which ACs are AD-constrained (Rule 6 — an `AD-n` in the spine binds this story's area) and which are AD-tooled (an AD commits to a tool stack for the verification — lead-executed later); confirm the spec's Design Notes name the governing `AD-n` ids — add them if the plan missed any (lead edit, non-frozen section). **Sections the template may have deleted:** the spec template tells the planner to remove `## Design Notes` and `## Verification` when trivial; if either is absent, the lead creates it now (empty heading + one line) so Rules 1/2/6/19 and QA's file-list rule have somewhere to write — log `sections_created=<csv|none>` on `spec_validated`.

**Spec sanity.** Frontmatter `status: ready-for-dev`; `## Auto Run Result` present; no `blocked`. A spec the READY-FOR-DEVELOPMENT standard rejected arrives as `blocked` / `spec failed ready-for-development standard` — clarification path, not a gate pass.

**Decision dependency.** `LEDGER slice all | grep decision-pending`: if the spec's Design Notes, Tasks, or I/O matrix cite one of those entries, or build on a value the ledger records as undecided (a rate, a threshold, a ratification), the story is about to be built on a question nobody has answered — an `intent gap` for the user NOW, not at the merge gate (field case 2026-08-29: a tick-rate ratification sat `decision-pending` for seven stories). Surface it with `LEDGER show DW-n`, record the answer as a trailer line (`by=spec_gate`) and in the spec's `## Spec Change Log`, and log `decision_dependency=<DW-n csv>` on `spec_validated`.

**Shared-runtime mutation.** Does any AC, I/O row, or `## Verification` command mutate system-level state on a runtime other stories or the lead also use — databases, namespaces, security objects, scheduled tasks, web applications, OS services, a shared dev server? If yes, the spec's `## Verification` must name a **throwaway environment** (a fresh container, a scratch namespace or database, a disposable instance) for its destructive checks, and every test-running party for this story (build-auto's handoff, QA, the review layers, the lead's smoke) runs stateful classes one at a time (Rule 18 (3)). A spec that names none: the lead adds the environment (lead edit, non-frozen section) or raises an `intent gap` if the project has nowhere disposable to run it (*Unattended:* stop the story — U5). Log `mutates_shared_runtime=true|false` on `spec_validated`. (Field report 2026-09-11: an installer story verified against the one long-lived instance; a rework's concurrent test runs left it with a database mounted over a deleted directory and the story blocked on a human restart.)

**Ledger inbox (Rule 17 (1b)).** `LEDGER slice <key>` again: every entry the story owns must be either cited in Tasks & Acceptance / the I/O matrix or listed as `Declined DW-n: <reason>` under Design Notes. An entry the plan ignored → the lead adds it as a Tasks & Acceptance item (lead edit, non-frozen section) or, if the plan's intent genuinely excludes it, re-owns or closes it now (`by=spec_gate` — the lead has the authority and the vocabulary here; adjudication after `done` does not). A declined entry is dispositioned the same way in the same commit. Log `owned_ledger=<DW-n csv|none> addressed=N declined=N` on `spec_validated`.

If NOT service-introducing (refactor, doc-only, internal cleanup, defect-fix), proceed. Then: log `spec_validated service_introducing=<bool> integration_ac=<present|declared-none|user-waived> adr_constrained_acs=<csv-or-empty> decision_dependency=<csv-or-none> mutates_shared_runtime=<bool>`; `SPRINT_PLAN generate --set <key>=ready-for-dev`; bookkeeping commit (everything under `implementation-artifacts`). This workflow gate is the binding enforcement.

### Context Handoff Between Stages (Critical)

The **spec file path** is the canonical context anchor, passed forward to every downstream agent. File lists flow from git (build-auto commits its work) and from the closing-summary sections of the QA/CR returns.

1. Plan → lead: the spec path (from the `SPEC:` line; verified by glob).
2. Lead → Implement: the spec path only (build-auto loads the spec's `context:` docs itself).
3. Implement → QA: lead reads `baseline_revision` from the spec frontmatter (or `baseline_commit` if that is the key a newer build-auto writes — accept either), computes `git diff --name-only <baseline_revision>..HEAD`, passes spec path + that list to QA.
4. QA → Code reviewer: lead reads `## Tests Added` from QA, passes spec path + `baseline_revision` + the diff list + QA's tests.
5. Code reviewer → Commit: lead stages the union of QA's tests, the reviewer's patches, and bookkeeping files (build-auto's own work is already committed).

If a QA/CR return is missing closing sections, fall back to `git status --short` plus the diff list above.

### Lead Context Management (long-run hygiene)

The lead's context window is the scarcest resource in a multi-epic run. Fresh context per stage is the point of the subagent design — both Anthropic's long-horizon-agent guidance and BMAD's own docs ("run each skill in a fresh context window") converge on it. Lead-side rules:

- The cycle log + `sprint-status.yaml` + spec files are the durable state — treat them as external memory (write-ahead, then act). Never rely on conversation memory for a resume or gating decision.
- Do NOT pull stage diffs, full test output, or whole spec files into the lead's context; the completion contracts exist precisely so stage detail stays in the stage. The lead reads a spec only at gates that require it (spec validation, AD mapping, rework re-open, smoke planning) and otherwise reads only its frontmatter and `## Auto Run Result`.
- After a context compaction/summarization event, re-anchor before the next action: re-read this command file (at least "Task Sequence" and the section for the current stage or gate — compaction keeps only its first ~5,000 tokens), then the cycle-log tail, the sprint-status snapshot (`SPRINT_PLAN status`), the current spec path and its frontmatter `status`. The write-ahead rule guarantees the state is recoverable from those files; the command text is what says what to do with it.
- Never load the rendered `bmad-build-auto` workflow into the lead's context — it runs inside the stage agent. The lead only ever reads spec frontmatter and the `## Auto Run Result` section.
- Never open `deferred-work.md`. A healthy ledger is small; an unhealthy one is megabytes, and either way `LEDGER load|slice|show` returns exactly the lines a gate needs.
- Budget: one story's pipeline should cost the lead only its gate-skill invocations, spawn prompts, returned closing summaries, and log writes.

### AD-Aware Execution (Required — the architecture spine)

The project's architecture decisions live in its spine — the `AD-n` blocks of `ARCHITECTURE-SPINE.md` (Rule 6), resolved once per epic and logged `spine_resolved`. ADs commit the project to specific tooling, methodology, and architectural patterns; an AC satisfied by the wrong tool stack is equivalent to a HIGH-severity defect. BMAD itself enforces none of this at build time (no Phase-4 skill reads the spine directly — it reaches `bmad-build-auto` only through the compiled epic context), so this gate and Rule 6 are the enforcement. The spine is also **written** by the pipeline: Rule 20 puts every decision with architectural weight into it at the moment it is made, so the next epic's plan stage sees it. A project with no spine (`spine_resolved path=none`) runs this section as `none_required` — and should get one from `bmad-architecture` before its next epic.

**Layer 1 — Lead-executed AD-tooling gate (between `dev_complete` and the QA spawn).**

After dev returns and before QA is spawned, the lead inspects the story's ACs for any that map to AD-committed agent-time tooling (visual verification, performance profiling, audits, etc.). For each matched AC, the lead drives the verification using its own tool inventory.

This gate exists because MCP tool inventories may not propagate reliably to spawned subagents. The lead always has the MCP servers at session level.

**Mechanics:**

1. Read the spec (path captured at `story_created`).
2. For each AC, consult the spine's ADs (Design Notes ids first, then any AD whose `Binds:` covers the area). If an AD's Rule commits to a specific tool stack for that AC, it's "AD-tooled."
3. Drive each AD-tooled AC verification using the relevant MCP / tool; record pass/fail + evidence paths. **Rule 19 applies to this gate most of all** — it is lead-executed and therefore trusted: for each AD-tooled AC name the mutation that would flip the result, apply it, observe the failure, revert, and confirm the tree is byte-identical (`git status --short` / `git diff --stat`). A `pass` with no named mutation is recorded `result=unverified`, not `pass` (field case: `pass_commands_empty` recorded against a value typed `readonly never[]`).
4. Append one cycle-log entry per story: `<UTC> TAB Story <id> TAB adr_verifications_complete TAB <metadata>` — including `mutations=<ac:what,…|none>`.
5. On failure, surface to user before spawning QA (*Unattended:* the rework path — U5).
6. Pass results to code reviewer's spawn-prompt context.
7. If no AD-tooled ACs, emit `adr_verifications_complete result=none_required` and proceed. (The stage name keeps its historical `adr_` spelling — see Rule 6.)

**Layer 2 — the spine path in every spawn prompt.**

The lead resolves the spine once per epic (`spine_resolved`) and includes its path in every agent spawn prompt as factual context. Agents must consult the ADs for architectural and methodology decisions referenced in their ACs — the plan stage records the governing ids under `## Design Notes`; the code reviewer verifies implementations match each AD's Rule — violations are HIGH (Rule 6).

**Layer 3 — decisions flow back (Rule 20).** Clarification answers, Rule 5 amendments, and decision-sheet calls with architectural weight are written into the spine by the lead (light path: new/amended AD, Conventions or Stack row, `memlog.py append`, `lint_spine.py`, commit; a conflict with an existing AD → `bmad-architecture` Update intent, interactive). The pre-warm then recompiles the epic context, so the decision reaches the next plan spawn. Logged `spine_updated`.

## When to Pause

*Unattended:* stages still halt with `## Clarification Needed` for anything outside the Unattended block's decide-small rule (U6); the lead classifies instead of pausing (U5).

Within each agent, halt and surface a clarification via `## Clarification Needed` if ANY hold:

- Ambiguous ACs or requirements.
- Missing prerequisite — story references data/code/context not present.
- Multiple reasonable design options where user preference matters.
- Environment or dependency failure blocking the work.
- Proceeding would risk breaking a stated constraint (security, compliance, performance, correctness, an AD's Rule).

Do not guess; do not soldier on. A short pause beats a wrong implementation unwound later.

## Handling Clarifications

*Unattended:* classify, then rule or park — doctrine U5.

When an agent returns with `## Clarification Needed`:

1. Read the question from the returned message.
2. Surface it to the user with Story ID + context.
3. Wait for the user's answer.
4. Re-spawn the same stage's agent with the clarification baked in (prompt, or the spec's `## Design Notes` — committed first, Rule 16; an answer with architectural weight also goes into the spine, Rule 20). Never `SendMessage` the old agent (see Anti-Patterns).
5. Log `<stage>_clarification_requested` before re-spawn; the stage's normal success entry (`story_created` / `dev_complete` / `qa_complete` / `cr_complete`) closes it.

**Key distinction:** clarification-needed is not a completion — closing sections won't be present. Detect via the `## Clarification Needed` heading (QA/CR stages) or the spec's `status: blocked` (build-auto stages — see the blocking-condition table and re-dispatch protocol under "Clarification protocol").

## Submodule / Sub-Repository Commit Order (Critical, if Applicable)

Applies to projects with git submodules OR sub-repositories. Skip if neither applies.

When stories modify files in child-repo directories:

1. **Commit and push inside each affected child first.** For git submodules, this produces an updated submodule pointer the parent will reference.
2. **Then commit and push in the parent.** For submodules, stage both parent files AND the updated pointer (`git add <submodule-path>`). For sub-repositories, the parent references children only at workflow level; children should still be pushed first.

If the parent is pushed with a submodule pointer that doesn't exist on the submodule's remote, other developers get checkout failures. Always submodules-first.

After each story, run `git -C <child-path> status --short` for every affected child to detect changes.

## Completion Logging

At each story completion, write a brief log entry: story ID/name, files touched, key design decisions, issues auto-resolved vs requiring user input.

### Cycle Log Format (enables resume)

Per-stage log entries, append-only. File: `_bmad-output/implementation-artifacts/cycle-log-epic-{N}.md`.

**Format (TAB-separated, exactly four fields):**

```
<UTC-timestamp> TAB <Story <id> | Epic <N>> TAB <stage> TAB <metadata>
```

- Fields separated by a literal TAB (`\t`), not spaces.
- The **metadata** field is whitespace-separated `key=value` pairs. Multi-values comma-separated; no spaces or tabs inside values (percent-encode if needed). Keys are lowercase snake_case.
- **Every sha is read, never recalled.** A `sha=` / `head=` / `build_sha=` / `baseline_revision=` value — in a log entry, a `## Clarification Needed`, or a completion contract — is produced by `git rev-parse --short <ref>` at the moment of writing, never from earlier context (field case 2026-08-29: a clarification quoted a HEAD that did not exist in the repository). The party reading a quoted sha verifies it resolves (`git cat-file -e <sha>`) before acting on it and logs a mismatch as `report_error fault=<lead|stage|runner> field=<name> reported=<sha> actual=<sha>`.
- **Durable facts are verified on origin.** `committed`, `ci_green_verified` and `epic_merged_to_feature` are written only after `git fetch` and `git merge-base --is-ancestor <sha> origin/<branch>` succeed, and a green only when `gh run view <id> --json headSha,conclusion` names that exact head ("CI Is a Gate").
- **A CI-fix commit is logged under the story it fixes** (`committed … fix_for=<story-key>`), never under the story in flight — otherwise one story's cycle time absorbs another's fix (field report 2026-10-10: one story read 16.4 hours when it took 5.6).
- Two entry kinds (distinguished by field 2):
  - **Story-level:** `Story <id>` (most entries).
  - **Epic-level:** `Epic <N>` (branch lifecycle + the optional epic summary).

**Valid story-level stages, in order:** `stage_spawned` (write-ahead marker IMMEDIATELY before every stage `Agent` call — see below), `story_created` (plan spawn returned `ready-for-dev`), `spec_validated` (lead gate passed; `ready-for-dev` written), `dev_complete` (implement spawn returned `done`; `review` written), `adr_verifications_complete` (mandatory, between dev and qa; emits `result=none_required` if no AD-tooled ACs), `qa_complete`, `cr_complete`, `ledger_adjudicated` (mandatory, after the last cr and before smoke — Rule 17), `smoke_complete` (mandatory, between adjudication and commit), `fresh_check` (mandatory, after smoke — Rule 19; metadata `env=<recipe|clean-checkout> tests=<n> result=<pass|fail> order=forward,reverse`), `committed`, `story_cycle` (mandatory, right after `committed` — one line the telemetry reads instead of re-deriving it: `plan_to_commit_min=<n> implement_passes=<n> reviews=<n> rework=<n> first_ci=<success|story|flake|infra|pending|none> models=plan:<m>,implement:<m>,qa:<m>,cr:<m>`; `first_ci` is back-filled by a later `ci_resolved story=<id>` entry, never by editing this line). Also story-level, whenever they occur: `stage_api_error` (a stage `Agent` call ended on a harness error — metadata `stage= error=<rate_limit|overload|usage_limit|other> wait_min=<n>`; closed by the re-spawn's normal entry) and `ci_red_classified` (metadata `run=<id> class=<story|flake|infra> test=<name|none>`). Clarification events use `<stage>_clarification_requested` (`<stage>` ∈ `plan | dev | qa | cr`), closed by that stage's normal success entry on the re-spawn — `story_created` for `plan`, `dev_complete`, `qa_complete`, `cr_complete` — never by a literal `<stage>_complete` (there is no `plan_complete`). Optional: `bookkeeping_committed sha=<short>` after each Rule 16 commit (aids resume forensics; not required). Rework-loop entries (story-level, between `cr_complete` and the next `stage_spawned`): `rework_opened` — the lead re-opened the spec; metadata `cycle_iteration=N iteration=<k> trigger=<high|med_first_review|fix_pack|smoke|ci|cap_override> items=<csv of finding / DW ids> scope_baseline=<sha>`. `rework_cap_reached` — the third iteration did not converge; metadata `cycle_iteration=N rework_iterations_used=3 cap=3 outstanding=<csv> result=stop_and_surface`. `rework_cap_override` — the user authorized one more iteration; metadata `cap=3 used=3 authorized_by=user items=<finite csv> close=next_review_without_high` — an override without `items=` is not writable: ask for the list first (Rework Loop step 5).

**`stage_spawned` — the in-flight marker.** Written IMMEDIATELY BEFORE the `Agent` call for any pipeline stage, never after. Metadata: `stage=<plan|implement|qa|code-review> spawn_at=<UTC> model=<id> agent_name=<name passed on the Agent call> cycle_iteration=N`, plus `story_base=<HEAD sha>` on a story's first implement spawn (and, unattended, its first after a set-aside or a re-plan). Closed by that stage's success entry (`story_created` / `dev_complete` / `qa_complete` / `cr_complete`) or by a `<stage>_clarification_requested`; the success entry's `spawn_at=` repeats the marker's value. A `stage_spawned` with no closing entry is the ONLY durable evidence that a stage may still be in flight — resume MUST check for it before any re-dispatch, and (orchestrator mode) it names the runner-owned agent to ask about. The durable record is otherwise empty for exactly the window in which liveness matters; this entry fills it.

**Valid epic-level stages:**

- `feature_branch_created` — SC-1 created the feature branch. Metadata: `repos=<paths>` `ticket=<id>` `description=<desc>` `root=<origin/branch>`.
- `epic_branch_created` — SC-2 created `{TICKET}-epic{N}`. Metadata: `repos=<paths>` `from=<feature-sha>`.
- `epic_branch_checked_out` — Lead checked out `{TICKET}-epic{N}` (resume or after creation). Metadata: `repos=<paths>` `head=<sha>`.
- `epic_branch_reopened` — SC-5 recreated after a prior merge. Metadata: `reason=<short>` `from=<feature-sha>`.
- `lead_model_gate` — Pre-flight lead-tier check. Metadata: `model=<detected>` `action=proceed|user_accepted`.
- `runtime_gate` — Pre-flight BMAD/uv/CI check. Metadata: `bmad=<version>` `uv=<version>` `ci=<gh|none|unavailable>`. (Pre-flight entries are held until Task Sequence step 4 has checked the FIRST epic of the range out to its epic branch, then written to that epic's cycle log, as `Epic <N>`, ahead of this run's other entries and committed with its first bookkeeping commit — never written before step 1's clean-tree check, which they would fail, and never on the feature branch or the trunk. A crash before then loses nothing: the pre-flight runs again on every invocation.)
- `launch_env` — Pre-flight read of the launch environment ("Permission Mode"). Metadata: `depth=<n|default>` `watchdog=<0|1>` `bash_max_ms=<n|default>`.
- `telemetry_gate` — Pre-flight scan for unapplied model-tier recommendations. Metadata: `pending=<N>` `action=applied|declined|none`.
- `protocol_violation` — Any layer observed a containment breach (Rule 18: subagent commit/push, orphan writer, interim return, `SendMessage` to a returned agent). Metadata: `stage=<name>` `depth=<N>` `agent=<name>` `violation=<short>` `consequence=<short>` `detected_by=<short>` `outcome=<short>`. Never omitted "because it was handled" — the record is the point.
- `root_freshness_resolved` — SC-1 root freshness check needed a user decision. Metadata: `repo=<path>` `root=<origin/branch>` `choice=push_local_first|base_on_local|base_on_origin_anyway`.
- `epic_context_compiled` — Lead-driven epic-context pre-warm wrote `epic-{N}-context.md`. Metadata: `sha=<bookkeeping-commit>` `reason=<initial|planning_artifact_newer|x0_inserted|nfr_amendment>` `model=<id>`.
- `spine_resolved` — The architecture spine the epic will read and write (Rule 6/20). Metadata: `path=<path|none> ads=<N> status=<final|draft|none>`.
- `spine_updated` — A decision with architectural weight was written into the spine (Rule 20). Metadata: `ad=<AD-n|conv|stack|deferred> reason=<clarification|rule5|decision_sheet> by=<lead|runner> story=<key|none> lint=ok`.
- `ledger_stray_bullets` — A reviewer wrote `## Deferred from:` bullets instead of ledger entries; the lead harvested them. Metadata: `harvested=N story=<key>`.
- `ledger_owner_repaired` — `ledger_load` found `UNKNOWN` owners and re-owned them by unique `<epic>-<story>-` prefix (Rule 17 (1)). Metadata: `repaired=N halted=N` (`halted` > 0 means an ambiguous key stopped the run).
- `ledger_routed_planned` — `DW-n:` acceptance bullets written into receiving stories' `epics.md` blocks (Rule 17 (1b)). Metadata: `story=<key> entries=N excess=N by=<harvest|cr|adjudication|burndown|merge_gate>`.
- `merge_gate_blocked` — SC-4 precondition failed. Metadata: `reason=<ci_red|…> run=<id> job=<name> head=<sha>`.
- `ci_green_verified` — The branch head's CI run is `success`, read independently by the lead/orchestrator. Metadata: `run=<id> head=<sha>`.
- `ledger_load` — Epic-start ledger measurement, before sprint planning (Rule 17). Metadata: `total=N open=N routed=N escalated=N decision_pending=N terminal=N burndown=N reowned_none=N`.
- `sprint_planning_complete` — Sprint planning done. Metadata: `gate=<PASS|CONCERNS>`.
- `ledger_burndown_complete` / `ledger_burndown_skipped` — End-of-epic burn-down gate (Rule 17), after the last `committed` and before `epic_status_done`. Metadata: `open_before=N resolved=N terminal=N chartered=N chartered_size=N overflow=N reowned=N to_decision_sheet=N open_after=N cap=N story_max=N filed_this_epic=N closed_this_epic=N drain_ratio=X.XX` for complete; `reason=empty` for skipped.
- `decision_sheet_recorded` / `decision_sheet_skipped` — Epic-close decision sheet at SC-4 (Rule 17 (5)): every `escalated` / `decision-pending` entry decided by the user in one sitting. Metadata: `decided=N to_action_items=N to_x0=N`; `reason=empty` for skipped.
- `report_error` — A quoted fact (sha, count, path) in a contract, clarification, or log entry did not match the repository. Metadata: `fault=<lead|stage|runner> field=<name> reported=<value> actual=<value> impact=<short>`.
- `retro_review_complete` / `retro_review_skipped` — Retro + Story X.0 gate done (start-of-epic CLOSE of the previous epic's retro artifacts, the ledger slice, and open action items). Metadata: `source_retro=<path-or-empty>` `resolved=<N>` `owned=<N>` `terminal=<N>` `dropped=<N>` `load_before=<N>` `load_after=<N>` `cap=<N>` for complete; `reason=<short>` for skipped.
- `epic_retro_complete` / `epic_retro_skipped` — End-of-epic retrospective gate (the fully interactive, human-in-the-middle run of `/bmad-retrospective` — distinct from `retro_review_*` above). Metadata: `reason=<short>` for skipped.
- `epic_status_done` — Lead set `epic-{N}: done` in sprint-status.yaml after the last story reached `done`. Metadata: `stories=<N>`.
- `model_tier_checkpoint` — End-of-epic tier evaluation, EVERY epic on every project. Metadata: `armed=true|false` (policy file present) `stack_risk=<uncommon|standard|none>` `review_tier=<full-opus|mixed>` `implement_model=<id>` `high_med_avg=<per-story>` `unresolved_high_med_avg=<per-story>` `metric=<unresolved|gross>` (`gross` only when some `cr_complete` in the epic predates 2026-09-11.1 and lacks the field) `rework_stories=<N>` `review_loop_thrash=<N>` `rework_lang_defects=<N>` `review_high=<N>` `review_med=<N>` `result=hold|recommend_escalate|offer_mixed|revert_required` `applied=true|false|user_declined`.
- `model_tier_changed` — model-overrides.yaml was updated (paired with a `history:` entry in the file). Metadata: `direction=escalate|de_escalate|revert` `scope=implement_qa|review` `stages=<csv>` `from=<tier>` `to=<tier>` `reason=<short>` `evidence=<percent-encoded summary of the triggering telemetry>` `approved_by=user|mandatory_rollback`. The *why* is non-optional: an entry without `reason=` and `evidence=` is a telemetry bug.
- `resume_local_ahead` — Resume check 2 found local ahead of remote; lead pushed. Metadata: `repo=<path>` `pushed_shas=<N>`.
- `epic_merge_skipped` — User declined SC-4 merge. Metadata: `reason=<short>`.
- `epic_merge_started` — SC-4 write-ahead marker, on the epic branch before the first checkout. Metadata: `head=<epic head sha>` `feature=<branch>`.
- `epic_merged_to_feature` — SC-4 merge completed (written on the feature branch and committed as `chore(epic-N): record merge`). Metadata: `repos=<paths>` `feature_sha=<sha>` `merge_sha=<sha>` `submodules=<paths-or-empty>`.
- Unattended runs (doctrine U2–U12): `unattended_started` / `unattended_resumed` (`run= policy= doctrine= range= lock_token=`), `permission_mode_attested`, `ci_mode_chosen`, `notify_unavailable`, `ruling` (`question= choice= rejected= why= category=<decide|veto|park|stop> policy=`, plus `reversal=` for a veto), `story_parked`, `story_set_aside` (`story= patch= sha=`), `story_unparked` (`via= story_base=`), `story_dropped` (`story= id=`), `story_waiting` (`story= on=`), `runner_unattended_start` (`run= policy= doctrine=`), `inbox_hand_edit` (`file=`), `rework_cap_override … authorized_by=owner via=`, `owner_decision` (`id= at=<answer_ref> via= by=`), `unattended_stop` (`unit= reason=`), `hold_ordered`, `hold_saved` (`story= stage= patch= tree=`), `hold_restored` (`story=`), `stage_dead` (`story= stage= evidence= rerun_from=`), `hold_complete` (`shas=`), `unattended_report`.
- `epic_summary` (optional, once per epic after the last `committed`) — see Workflow Telemetry.

**Standardized telemetry (on every `*_complete` entry):**

- `spawn_at=<UTC>` — when the lead invoked `Agent` (omit on lead-driven stages).
- `model=<id>` — which model the agent ran (e.g., `claude-opus-4-8`, `claude-sonnet-5`, `claude-haiku-4-5`). For spawned pipeline stages this is the model resolved per Agent Invocation Pattern step 1 (model-overrides.yaml, else frontmatter override, else the Model Strategy map) and passed on the `Agent` call. For lead-run gate stages, the lead's model. A run where every stage shows the lead's model is a signal that the `model` parameter was not passed on the spawns — `story_created` and `cr_complete` should show the Opus-tier id, `dev_complete` and `qa_complete` the Sonnet-tier id, unless a project re-pinned them.
- `cycle_iteration=N` — defaults to 1; increment on re-spawn after downstream rejection or clarification.

**Stage-specific telemetry (when available):**

- `story_created`: `path=<spec> build_status=ready-for-dev spec_tokens=N epic_context=<compiled|reused>`.
- `spec_validated`: `service_introducing=true|false integration_ac=<present|declared-none|user-waived> adr_constrained_acs=<AD-n csv-or-empty> decision_dependency=<DW-n csv|none> sections_created=<csv|none> owned_ledger=<DW-n csv|none> addressed=N declined=N mutates_shared_runtime=true|false`.
- `dev_complete`: `build_sha=<short> baseline_revision=<short> [story_base=<short>] review_loop_iteration=N followup_review_recommended=true|false deferred=N loc_added=N loc_removed=N files=N nfr_tripwires=N adr_violations_surfaced=N` (`story_base` appears on the story's first `dev_complete` only and is never rewritten; `review_loop_iteration` and `followup_review_recommended` are read from the spec frontmatter; `deferred` is the count harvested into the ledger).
- `adr_verifications_complete`: `tool=<id> acs=<comma-separated-list> result=pass|fail|unverified|none_required evidence=<path-or-empty> mutations=<ac:what,…|none>` (`pass` requires a mutation per AC — Rule 19).
- `qa_complete`: `tests_added=N mutations_demonstrated=N first_run_failures=N clarifications=N closing_sections_present=true|false` (a persistent gap between `tests_added` and `mutations_demonstrated` is the visible signal of vacuous coverage).
- `committed`: also `amendments=<file:line csv|none>` — every Rule 5 apply-and-report amendment this story made to a planning artifact or root file, for post-hoc review.
- `cr_complete`: `resolved=N fixed_at_source=N by_design=N wontfix_theoretical=N routed=N escalated=N decision_pending=N deferred=N dismissed=N high=N med=N low=N rows=N unresolved_high_med=N clarifications=N closing_sections_present=true|false` — `high`/`med`/`low` are root-cause ENTRIES after the reviewer's step-03 grouping, copied from its `entries:` closing line; `rows` is the raw finding count the layers reported before grouping (volume, never a threshold input); `unresolved_high_med` is the high+med entries the reviewer left unresolved after its own patches — the model-tier checkpoint's escalation input (field report 2026-09-11: four layers filing one root cause as four rows roughly doubled the gross count the checkpoint read).
- `ledger_adjudicated`: `owned=N resolved=N reowned=N terminal=N` (`owned` = slice size before; `resolved+reowned+terminal` must equal it).
- `smoke_complete`: `method=<browser|cli|api|other> result=pass|fail iterations=N defects_caught=N evidence=<path>`.
- `committed`: `sha=<short-hash> submodules=<paths-or-empty> ci=<pending|success|failure|none|unavailable> run=<id|none>` (`pending` is resolved by a later `ci_resolved`; a `committed` without `ci=` is a pre-2026-08-30.2 entry).
- `ci_resolved`: `story=<id> run=<id> result=<success|failure|cancelled> resolved_at=<next_plan|burndown|merge_gate|resume>`.

**Cost telemetry (when available):** `input_tokens=N output_tokens=N cost_usd=X.XX` on any `*_complete` entry. Omit if not extractable.

**Example** (TABs shown as `→`; actual file uses literal tabs):

```
2026-05-18T13:55:02Z→Epic 1→feature_branch_created→repos=. ticket=PROJ-1234 description=initial-foundation root=origin/main
2026-05-18T13:55:10Z→Epic 1→epic_branch_created→repos=. from=a1b2c3d
2026-05-18T13:55:11Z→Epic 1→epic_branch_checked_out→repos=. head=a1b2c3d
2026-05-18T13:55:40Z→Epic 1→ledger_load→total=0 open=0 routed=0 escalated=0 decision_pending=0 terminal=0 burndown=0 reowned_none=0
2026-05-18T13:56:00Z→Epic 1→sprint_planning_complete→gate=PASS model=claude-opus-4-8
2026-05-18T13:56:30Z→Epic 1→retro_review_skipped→reason=no_retro_no_open_ledger_no_open_action_items
2026-05-18T13:57:10Z→Epic 1→epic_context_compiled→sha=b2c3d4e reason=initial model=claude-opus-4-8
2026-05-18T14:20:00Z→Story 1.5→stage_spawned→stage=plan spawn_at=2026-05-18T14:20:05Z model=claude-opus-4-8 agent_name=1-5-render-engine-plan-1 cycle_iteration=1
2026-05-18T14:23:11Z→Story 1.5→story_created→spawn_at=2026-05-18T14:20:05Z model=claude-opus-4-8 path=_bmad-output/implementation-artifacts/spec-1-5-render-engine.md build_status=ready-for-dev spec_tokens=1420 epic_context=compiled
2026-05-18T14:23:40Z→Story 1.5→spec_validated→service_introducing=true integration_ac=present adr_constrained_acs=ac5 model=claude-opus-4-8
2026-05-18T14:23:48Z→Story 1.5→stage_spawned→stage=implement spawn_at=2026-05-18T14:23:50Z model=claude-sonnet-5 agent_name=1-5-render-engine-implement-1 cycle_iteration=1
2026-05-18T14:24:02Z→Story 1.5→dev_complete→spawn_at=2026-05-18T14:23:50Z model=claude-sonnet-5 build_sha=9f8e7d6 baseline_revision=a1b2c3d review_loop_iteration=1 followup_review_recommended=false deferred=1 files=3 loc_added=412 cycle_iteration=1
2026-05-18T14:25:00Z→Story 1.5→adr_verifications_complete→tool=chrome_devtools_mcp acs=ac5 result=pass evidence=path/to/evidence/ model=claude-opus-4-8
2026-05-18T14:29:47Z→Story 1.5→qa_complete→spawn_at=2026-05-18T14:25:30Z model=claude-sonnet-5 tests=tests/render-engine.test.ts tests_added=14 closing_sections_present=true
2026-05-18T14:33:18Z→Story 1.5→cr_complete→spawn_at=2026-05-18T14:30:15Z model=claude-opus-4-8 resolved=2 deferred=0 high=1 med=1 low=0 closing_sections_present=true
2026-05-18T14:33:50Z→Story 1.5→ledger_adjudicated→owned=1 resolved=1 reowned=0 terminal=0 model=claude-opus-4-8
2026-05-18T14:34:00Z→Story 1.5→smoke_complete→method=browser result=pass iterations=1 defects_caught=0 evidence=path/to/screens/ model=claude-opus-4-8
2026-05-18T14:34:30Z→Story 1.5→committed→sha=abc1234 submodules=
2026-05-18T17:58:10Z→Epic 1→ledger_burndown_complete→open_before=3 resolved=1 terminal=1 chartered=0 reowned=1 open_after=1 cap=8
2026-05-18T17:58:40Z→Epic 1→epic_status_done→stories=6
2026-05-18T18:02:11Z→Epic 1→epic_merged_to_feature→repos=. feature_sha=def5678 merge_sha=fed8765 submodules=
```

**Parsing rule:** split each line on TAB into exactly 4 fields; split metadata on whitespace into `key=value` tokens; split each value on `,` for lists.

On restart, scan the cycle log for the highest-stage entry per story / epic to compute the resume point.

## Workflow Telemetry

The cycle log is the primary telemetry surface. Standardized metadata + stage-specific keys make per-stage cost, quality, and model attribution computable without extra instrumentation.

**What the metadata enables:**

- Per-stage duration by model — `entry.timestamp − entry.spawn_at` grouped by `entry.model`.
- Bug rate by upstream model — `cr_complete.high + cr_complete.med` (root-cause entries) per story, grouped by `dev_complete.model`; `cr_complete.unresolved_high_med` for what escaped both the implementer and the reviewer's own patch pass (the escalation signal); `cr_complete.rows` for raw review volume.
- Rework rate by model — count of `cycle_iteration > 1` entries grouped by model.
- Rework convergence — `rework_opened` per story (its `trigger=`), `rework_cap_reached` / `rework_cap_override` events, and `followup_review_recommended=true` on a rework's `dev_complete` (post-2026-09-11.1: a HIGH was patched in that pass).
- Test-pyramid leak rate — `smoke_complete.defects_caught > 0` events.
- NFR-tripwire surfacing rate — `dev_complete.nfr_tripwires` grouped by model.
- Closing-section reliability — `closing_sections_present=false` rate grouped by model.

**`epic_summary` entry (optional, once per epic after the last `committed`):**

```
<UTC> TAB Epic <N> TAB epic_summary TAB stories=N wall_clock_hours=X.X total_high=N total_med=N total_low=N total_smoke_defects=N rework_events=N opus_stage_count=N sonnet_stage_count=N haiku_stage_count=N input_tokens_total=N output_tokens_total=N cost_usd=X.XX
```

Derivable from per-stage entries; a convenience, not a source of truth.

## Anti-Patterns (Do NOT Use)

- **TeamCreate / SendMessage / TeamDelete / team_name / shutdown handshakes** — Inter-agent messaging is unreliable. Use plain `Agent` tool calls; the return value IS the completion signal.
- **`SendMessage`-resuming a pipeline stage agent** — a stage agent's `Agent` return value IS its completion signal. Resuming one afterwards leaves an agent that has "returned" yet is still running: unrepresentable in the cycle log (its `stage_spawned` was already closed), invisible to resume detection, and a de-facto background stage in violation of the synchronous-stage rule — the exact setup for two agents editing one worktree. When a stage HALTs for clarification, the ONLY sanctioned continuation is a fresh re-spawn per the re-dispatch protocol, with the answer written into the durable artifact (the spec), which is also what makes it resumable after a crash. The rule is universal (Rule 18): `SendMessage` to ANY agent whose `Agent` call has returned is forbidden at every depth and in every direction — lead→stage, runner→stage, orchestrator→stage, runner→runner, and a stage agent→its own internal subagent are examples, not the list. A returned agent is finished; spawn a fresh one. Orchestrator→runner is the sole sanctioned use (parallel kit). Beware upstream text that says "re-engage the subagent with a synchronous message" (`bmad-build-auto` step-04): on this harness that is exactly this anti-pattern — the implement block pre-answers it.
- **Propagating the working directory but not the prohibitions** — an internal subagent (build-auto's handoff, a review layer) that receives Rule 13's path and not the `🚫` git/CI prohibitions does what any capable agent does when it finishes work in a repo with no instruction to the contrary (incident 2026-08-28: commit + push + live deploy from depth 3). Every stage block propagates its prohibitions "in the same breath" as Rule 13, and the layer above verifies self-reports against `git`/`gh` rather than trusting them.
- **Diagnosing liveness from artifacts** — spec `in-progress`/`in-review` + dirty tree describes a live agent as well as a dead one. Liveness belongs to sessions: a fresh sequential lead owns no live stages; an orchestrator asks the runner. Re-spawning on the artifact reading alone puts two agents in one worktree (see Resume Semantics).
- **Returning early** — an interim or progress return while your own subagents are still working strands them as orphan writers that no party can stop (`TaskStop` is owner-scoped and the owner has returned). The same holds for a background test sweep and an armed `Monitor`: "the monitor is armed, I'm holding off" ends a subagent's turn, which IS its return, and the sweep dies or runs on unsupervised. A sweep too long for one foreground call (the cap is 10 minutes) is split into calls that each fit — never backgrounded to get around the cap. Every agent at every depth returns exactly once, when its work is complete; a HALT is a completion, an update is not (Rule 18 (2), (5)).
- **Trusting `ListAgents` for stage discipline** — it reports process liveness, not protocol state; a resumed agent shows `running` after it returned. Use it to tell whether a runner is alive or dead, never to conclude "one active stage agent, which is fine" (Rule 18).
- **TaskCreate / TaskList / TaskUpdate** — Subagents poll TaskList and grab tasks regardless of `blockedBy` or ownership. The task system isn't needed; ignore it.
- **Dispatching `bmad-build-auto` without an explicit anchor** — Without a story key (plan) or spec path (implement) the skill scans for intent, picks the wrong story, or halts `unclear intent`. Under batches or resume that silently plans the wrong story. Always pass the anchor; never let it "find the next story".
- **Using the interactive `bmad-build` as a pipeline stage** — Its checkpoints (`[A]/[E]`, `[S]/[K]`, resume list, push/PR offer) and its `open_spec` editor launch have no place in an unattended run. `bmad-build-auto` is the same workflow with those removed and a machine HALT contract.
- **Folder+id dispatch in sprint mode** — Passing a spec folder / `stories.yaml` to build-auto on an `epics.md` + `sprint-status.yaml` project halts `no epic spec found`. That dispatch mode is for `bmad-spec` projects.
- **Spawning `bmad-build-auto` on a dirty tree** — It halts before planning and again at finalize (Rule 16). Uncommitted cycle-log or tracker writes are the usual culprit: write-ahead → bookkeeping commit → spawn.
- **Disabling every build-auto review layer to save cost** — The skill halts `no active review layers`. Trim by `id` in `_bmad/custom/bmad-build-auto.toml` if telemetry justifies it; keep at least one.
- **Inline skill execution** — Agents interpreting skill logic instead of invoking via the `Skill` tool. Always specify `Skill` explicitly in prompts. For `bmad-build-auto` this also skips the `uv run render_skill.py` bootstrap — the agent then follows unrendered, placeholder-ridden workflow text.
- **Missing context handoff** — Not passing the spec path, `baseline_revision`, and the diff file list between stages; code reviewers can't review effectively.
- **Parent-before-submodule push** — Broken submodule pointers on the remote. Always submodules-first.
- **Normalizing known test failures** — Carrying forward "N pre-existing failures, unrelated" erodes baseline reliability. Fix or formally defer in `deferred-work.md` immediately.
- **Treating a local green as a CI green** — every gate runs the suite on the lead's machine; the run that matters runs on the platform the project ships from, and nobody reads it unless a gate does. Record the run at `committed`, resolve it before the next plan spawn, and never present the merge question on a red head (see "CI Is a Gate").
- **Routing to a key nobody validated** — `owner=<story key>` is a promise that a gate will enumerate the entry; a stale or mistyped key breaks the promise with no error and no count anomaly. `ledger.sh` refuses unknown owners; `ledger_load` repairs or halts on the ones that slipped through (Rule 17 (1)).
- **Routing without planning** — an entry that exists only as `owner=<key>` in the ledger reaches that story's pipeline at `ledger_adjudicated`, after the design is fixed and code review has passed, where the only honest moves are re-own or a Fix Pack. Routed means planned: an acceptance bullet in `epics.md` and the plan spawn's ledger inbox (Rule 17 (1b)).
- **Deferred findings only in spec files** — Without centralized tracking in `deferred-work.md`, deferred items are invisible to the drain (adjudication, burn-down, X.0).
- **Filing without draining** — Every gate writes to `deferred-work.md` and no stage closes an entry, so the pool is monotonic and its open count stops meaning anything (field report: 0 → 1,588 entries in 114 days, never shrinking). A rule that says "periodic burn-down" with no stage that executes it never runs. The drain is a stage with a log entry: `ledger_adjudicated` per story, `ledger_burndown_*` per epic, X.0 as a close (Rule 17).
- **Relabelling instead of closing** — "Defer with rationale", re-owning to "the next X.0", or leaving `owner=burndown` past the burn-down gate moves an entry without deciding it. Every disposition is `resolved-by`, a specific story key, or terminal. (The one sanctioned deferral is a `decision-pending` question the decision sheet sends to the next X.0 as `owner=burndown note=to_x0:…` — it is decided there.)
- **`burndown` owning a LOW** — a LOW is fixed at emission (two-way door → patch / Fix Pack) or closed `wontfix-accepted` with a probe; routing it "to the story that will touch it" turns a 15-line fix into a story bullet nobody wants (field report 2026-08-29: 17 of 30 burn-down entries). "In-footprint" is the epic's footprint, not the story's.
- **Deciding a product call at the burn-down gate** — `escalated` and `decision-pending` entries are the user's; a lead or runner that re-owns, closes, or answers one there has decided for the user and hidden the decision in a trailer line. They go to the decision sheet at SC-4 (Rule 17 (5)); a runner lists them under `decisions_for_user:`.
- **Numbering the burn-down story `N.9`** — 9 was shorthand for "last" and is last only on an epic with eight or fewer stories; "the next unused number" can fill a gap below an existing story. The burn-down story is `N.<M+1>`, M the highest existing story number under Epic N across `epics.md` and `sprint-status.yaml`, reused if it already exists (Rule 17 (3)).
- **A burn-down story that takes everything** — one story with 30 bullets is either enormous or a relabelling in disguise. The charter is bounded by `burndown_story_max`; the overflow is re-owned explicitly and counted, and the drain ratio is logged so an under-sized drain is a number.
- **Recording a pass nobody could have failed** — a test or gate whose red has never been observed is not evidence (Rule 19): expected values forced by the type, subjects never constructed, two empty runs compared. Every AC's pinning test has a named, applied, reverted mutation; the lead's AD gate is the layer that most needs it because it is the layer everyone trusts.
- **Quoting a sha from memory** — every sha in a contract, clarification, or log entry is `git rev-parse`d at the moment of writing, and verified (`git cat-file -e`) by whoever acts on it.
- **Reading or hand-editing the ledger file** — Prose accretes (a field-reported 24 kB entry), owner clauses land on the wrong line, and a closed-family word in a body fools every count. `LEDGER` is the only reader and writer; effective state is the latest trailer line by timestamp.
- **Re-filing an adjudicated finding** — a `by-design`/`wontfix`/`routed` ledger entry re-reported as new work re-purchases triage every epic (pilot: one root cause written up four times). Rule 15: `LEDGER slice all` first; `append occurrence=` on the canonical entry.
- **Expecting the tracker to hold stories `epics.md` lacks** — `SPRINT_PLAN generate` derives every key from the epic files and drops any other key (`dropped_orphans`); a hotfix or cleanup story exists only once it has a `### Story` heading in `epics.md`. A non-empty `dropped_orphans` is a defect to investigate, never noise.
- **Skipping retrospective review before epic start** — Without explicitly reading the previous retro and triaging deferred items, accumulation goes silent.
- **Batching stories inside one checkout** — two `bmad-build-auto` runs in one working tree halt on each other's uncommitted work (see "Smart Parallelism"). One story at a time per checkout; concurrency is the parallel kit's per-epic worktrees.
- **Deferring AD-mandated agent-time verification without surfacing it** — "I can't do X from this environment" for work an AD's Rule commits to specific tooling. The lead executes AD-tooled verifications directly.
- **Treating AD violations as LOW deferrable findings** — AD violations are HIGH (Rule 6).
- **Keeping a decision registry BMAD does not read** — a `docs/adr/` folder, a rules file, a trailer line: none of them reach `bmad-build-auto`'s epic context. The spine's `AD-n` blocks are the registry; a decision with architectural weight goes there, at the moment it is made (Rule 20).
- **Hard-coding upstream prompt text in an override** — an id-keyed `[[workflow.review_layers]]` entry replaces the whole layer, so a pasted copy of last release's instruction silently pins a stale placeholder (`{diff_output}` → `{diff_file}` on BMAD `main`) or a rewritten brief. Derive from the installed `customize.toml` and apply the kit's transformations; Step 4 checks the placeholders.
- **Skipping the per-story smoke** — Test pyramid passing while the deployed product is broken is recurring. Failed smoke is HIGH, never deferrable.
- **Smoke executed by a spawned subagent** — Subagents may lack runtime tooling. Smoke is lead-side.
- **Service-introducing story without an Integration AC** — Producer + consumer ship green with wiring never built. Lead validates integration-AC presence at the spec-validation gate.
- **`on_complete` hooks in BMAD `.toml` customizations** — Verification gates belong in the workflow's spawn-prompt skeleton, not the `.toml`. Keeping both creates two sources of truth.
- **Blocking the pipeline waiting for a "completion message"** — The `Agent` tool returns once when the subagent's run ends. Read the returned message directly.
- **Committing directly to `main`, `master`, or `develop`** — Rule SC-6 forbids this absolutely. Every workflow commit lands on `{TICKET}-epic{N}`; trunks are reached only via the SC-4 merge gate + an out-of-band PR.
- **Creating the feature branch silently** — Rule SC-1 mandates STOPping and asking. Silent defaults produce `feature/undefined_undefined` on remotes.
- **Skipping Source Control gates on a "resume" assumption** — Mid-resume should find the branches in place. If missing, that's a workspace-integrity error per SC-2 — halt; don't silently re-create. Silent recreation orphans prior commits.
- **Forgetting submodules need their own `{TICKET}-epic{N}`** — SC-2 applies per repo. Each submodule has its own.
- **Running parallel epics in a single working directory** — Git allows one HEAD per working tree. Parallel agents use `git worktree add` or separate clones (SC-8).
- **Auto-resolving merge conflicts at SC-4** — Three-way-merge conflicts at end-of-epic must be surfaced to the user. Git's auto-resolution can silently drop intentional changes. (Attended runs. `--unattended` resolves only what its policy allows, as rulings the owner can veto — U9.)
- **Spawning pipeline sub-agents without passing the resolved model** — Omitting the `model` parameter on the `Agent` call makes every stage inherit the lead's model instead of the pipeline's intent (dev/qa → `sonnet`, code-review → `opus` per the Model Strategy map, unless `_bmad/custom/model-overrides.yaml` or a skill's frontmatter re-pins it). Resolve per Agent Invocation Pattern step 1 and pass it on the spawn. Lead-run gate skills are the exception — they run inline on the lead's model by design.
- **Letting a skill's interactive checkpoint block the run** — `bmad-code-review` has four happy-path halts (context confirm, decision-needed resolution, patch menu, next-steps menu) plus a large-diff chunking offer; `bmad-qa-generate-e2e-tests` opens by asking what to test. Pre-answer every checkpoint in the spawn prompt (Rule 9); an un-pre-answered checkpoint is a Clarification, never a silent wait. (`bmad-build-auto` has none — its only exits are HALT statuses.)
- **Expecting `bmad-build-auto` to sync sprint-status.yaml** — It never does (only the interactive `bmad-build` syncs). The lead writes `ready-for-dev` / `in-progress` / `review` through `SPRINT_PLAN generate --set`; `bmad-code-review` writes `done`/`in-progress`; the lead writes `epic-{N}: done`. Any other status write path (hand edits, prose instructions to the stage agent) desyncs tracking (see Status Ownership).
- **Hand-parsing or hand-editing sprint-status.yaml** — `SPRINT_PLAN status` returns per-status counts and the next story machine-readably; `TRACKER_READ` is the one sanctioned per-story read; `SPRINT_PLAN validate` checks integrity; `SPRINT_PLAN generate --set` is the only writer. `/bmad-sprint-status` is a deprecated shim whose `mode=data`/`mode=validate` no longer exist.
- **Reading build-auto's prose instead of the spec** — The spec frontmatter `status` and `## Auto Run Result` are the contract; an agent's summary can claim `done` while the spec says `blocked`. Re-read the spec from disk after every build-auto spawn.
- **Confusing spec status with story status** — Spec `done` = build-auto finished; story `done` = code review passed. Spec `in-review` ≠ sprint-status `review`. Never copy one vocabulary into the other.
- **Backgrounding pipeline subagents** — including by omission: on current Claude Code builds an `Agent` call without `run_in_background: false` IS a background call, whatever its prompt says about synchrony. A backgrounded subagent never hands control back in an unattended run and the pipeline stalls (the reason `bmad-build-auto` mandates synchronous subagent calls); one level down it returns an interim message while its handoff or test sweep runs on with no supervisor (Rule 18 (5)). Pass `run_in_background: false` on every stage spawn, require it in every spawn prompt, and keep tests and commands in the foreground; a parallel batch is N `Agent` calls in ONE message, which still resolve together.
- **Batching any pipeline stage in one working tree** — Two concurrent build-auto runs in one checkout see each other's uncommitted spec/context files or finalize commits and halt (dirty tree / `finalization left repository dirty`) or race on the index. Stories are strictly sequential within a checkout; concurrency is per-epic worktrees (parallel kit).
- **Letting the first plan spawn compile `epic-<N>-context.md`** — build-auto compiles it *before* its own clean-tree check and then halts on its own untracked output. The lead pre-warms and commits the cache (see Epic Context Pre-warm) after every planning-artifact change.
- **Re-dispatching on a `blocked` spec without resetting `status`** — build-auto routes by frontmatter status; a `blocked` spec always halts `blocked spec supplied`. Apply the answer, set `draft` (re-plan), `in-progress` (re-implement), or `in-review` after `git apply`-ing the saved patch of a review-stage intent gap, commit, re-spawn on the spec path — never on the story key (that resumes only a `draft` spec and creates `spec-<key>-2.md` for every other status).
- **Trusting `--set` never to downgrade** — `sprint_plan.py generate --set` is the script's repair path and writes unconditionally. Read the current value first.
- **Unbounded implement↔review rework** — Endless re-spawn ping-pong on unresolved findings is a signal the spec is wrong, not the code. Cap at 3 rework iterations, then surface to the user; a cap override needs a finite `items=` list and closes at the next review without HIGH (see Rework Loop). A re-review that finds no HIGH closes the story — MEDs alone never re-open a reworked story (Rule 15 exit condition).
- **Re-reviewing the whole story after every rework** — resetting `baseline_commit` to the original baseline makes each re-review re-read the entire story diff; fresh eyes on old code always find something tolerable, and under fix-now rules a tolerable finding is a rework spawn (field report 2026-09-11: round 3 re-read 7,247 lines, found 0 HIGH, and the story was re-opened anyway). After a rework the review is scoped to `<rework baseline>..HEAD`; the whole story was reviewed at round 1.
- **Batching test-runner calls in one message against a shared runtime** — tool calls in one message run concurrently; stateful test classes race on shared fixtures (an uninstall racing an install left a database mounted with no file behind it). One class or suite per call, wait, never re-submit on a client timeout — propagated to every depth (Rule 18 (3)); destructive verification belongs on a throwaway environment named at spec validation.
- **Manufacturing next round's findings from this round's fixes** — a rework that changes pinning tests without updating their `mutation:` lines hands the next review a fresh set of Rule 19 findings. Whoever changes a pinning test writes the line in the same pass; a stale line on a test that exists is a LOW the reviewer closes in-pass, never a rework trigger.
- **Skipping the `deferred:` harvest, or harvesting without an owner** — build-auto's deferred findings live only in spec frontmatter until the lead files them with `LEDGER new … <status> <owner> harvest`; unharvested items are invisible to the drain, and ownerless ones have no gate that will ever close them (Rules 15/17).
- **Automating the retrospective** (attended runs — `--unattended` runs it headless by the owner's policy, U9) — Spawning `/bmad-retrospective` as a subagent, pre-answering its elicitation or WAIT points, or passing its `-H` / `--headless` flag. The retrospective is the one deliberate human-in-the-middle gate in the loop; its output is only as good as the human answers it collects. (Distinct from the start-of-epic retro-REVIEW gate, which is lead-automated triage of the previous retro's written artifacts and needs no human.)
- **Mis-scoping or forgetting to restore the IDE file-sync toggle** — On IRIS/ObjectScript projects, disable `objectscript.conn.active` ONLY transiently around each branch-changing git operation (Window A: epic-start branching; Window B: SC-4 merge), and restore it immediately after each window. Do NOT leave the sync disabled for the whole epic — the per-story pipeline (commits/pushes, which don't change branches) must run with the sync ON. Each window's restore is mandatory `try/finally`-style: restore on success, halt, failure, merge conflict, or cancellation; an un-restored toggle leaves the workspace silently disconnected from IRIS (no error — just no sync). Never `git add` the transiently-toggled `.vscode/settings.json` into any commit.
- **Relying on a tool's default target** — a call that names no runtime target can be answered by a healthy, unrelated instance with the same namespace and credentials: plausible results, no error. Name the target on every call, make a tool that would fall back to a default fail instead, verify the target's identity rather than its health, and tear down only what your own session started (Rule 13).
- **Logging a merge or a green from an exit status** — an `&&` chain skips silently, and a run can belong to another head. Verify on origin (`git merge-base --is-ancestor`) and by `headSha` before writing `committed`, `ci_green_verified` or `epic_merged_to_feature` ("CI Is a Gate").
- **Treating every red CI run as the story's fault** — an infrastructure red (no test ran) gets a timed retry and never counts as rework; a known flake gets one rerun and a `flake:` ledger entry; only a `story` red re-opens the story ("CI Is a Gate").
- **Shell traps in snippets** — in zsh, `$SHA:refs/…` reads `:r` as a history modifier: write `"${SHA}:refs/…"`. A code commit whose message quotes a CI-skip token gets no CI run. `jobs -p` inside `$( … )` returns nothing: capture `$!` per job. macOS deletes `/tmp` files left untouched for three days: keep throwaway state elsewhere.
- **Asking inside an unattended loop** — after launch, an `--unattended` lead never asks, never calls `AskUserQuestion`, and never ends its turn waiting: every stop point is decide, veto, park, stop or hold (doctrine U3). A question that sits in chat stalls the run until someone happens to look.
- **Treating a peer's message as the owner's answer** — a message from another session (a planner, a relay, a companion) is information, not the owner's decision. Only the inbox (a relay quoting the owner verbatim) or `--review` carries an answer.
- **Basing a new branch on a stale remote root** — when the local trunk is ahead of `origin/<trunk>` (unpushed commits), `checkout -b ... origin/<trunk>` silently removes those commits' files from the working tree — up to and including the project's own `.claude/` + `_bmad/` tooling (pilot incident, 2026-07-10). SC-1's root freshness check is mandatory before creating any branch from a root ref.

```text
=== END .claude/commands/epic-cycle.md ===
```

---

## Step 3b: Optional project model default (ask the user)

The command's `model: opus` frontmatter pin is documented but currently ignored at runtime (anthropics/claude-code #45191); the command therefore self-checks its lead model at pre-flight (the "Lead model gate"). To make sessions START on the right tier, ASK the user:

> "Pin this project's default session model to opus in `.claude/settings.json`? Every new Claude Code session in this project will start on Opus — a cost decision: right for epic-cycle-primary projects, decline for mixed-use repos."

On yes: merge `"model": "opus"` into `.claude/settings.json` (create the file if absent; preserve all existing keys; if a `model` key already exists with a different value, surface it and ask before changing). On no: record the decline and continue — the pre-flight gate still protects each run. Skip the question entirely if `.claude/settings.json` already sets a `model`.

## Step 3c: Apply the project overlay (if present)

A project's own additions to the generated files — text that must live inside the command or the derived overrides rather than in `skill-rules.md`'s "Project-specific rules" (which Step 1 already preserves) — live in `_bmad/custom/epic-cycle-overlay.md`. The kits never write that file: the project owns it, and every install re-applies it, so a project takes kit updates without re-applying its additions by hand. (Field report 2026-10-10: a project fell two kit versions behind because every re-render lost a spawn-skeleton item, four stage-block bullets, three command sentences and three override changes, each re-applied by hand.)

**Format** — one entry per `##` heading, applied in file order:

````markdown
## <entry-id>: <one-line purpose>
- file: .claude/commands/epic-cycle.md
- anchor: <one exact line of the target file, or a substring that occurs in exactly one line>
- position: after | before | replace
- until: <optional, `replace` only: the first whole line at or after the anchor that ends the replaced block>

```text
<the text to insert, verbatim, as whole lines>
```
````

`file` is `.claude/commands/epic-cycle.md`, `_bmad/custom/bmad-build-auto.toml`, or `_bmad/custom/bmad-code-review.toml`. `after` / `before` insert the text as new lines next to the anchor line; `replace` replaces the anchor line — or, with `until`, the block from the anchor line through the first following line that equals `until` exactly. (Switching a review layer off is a `replace` from its `id = "…"` line through `"""`, with `id`, `name` and `instruction = ""` as the text.)

**Applying it.** For each entry, find the anchor in the target file. Exactly one matching line → apply it, unless the entry's text is already present verbatim at that position (a re-run: skip and report `already applied`). Zero or several matching lines → STOP and report the entry with its match count: the kit text it anchored to has changed, and the project re-anchors the entry — never drop one silently, never guess. Override entries apply right after Step 2 has written the overrides (then re-run Step 4 check 1c on them); command entries apply once the command is final — here, after Step 3b, or, when the parallel kit is installed or about to be re-applied, by the parallel kit after its own patches (its Edit 5.7). Report applied / skipped / stopped entries in the install summary.

## Step 4: Validate

After writing all files, run these checks:

1. **Slash command does NOT reference deprecated multi-agent patterns:**
   ```bash
   grep -E "TeamCreate|TeamDelete|SendMessage|team_name|shutdown_request|shutdown_response|STATUS: completed|STATUS: clarification_needed" .claude/commands/epic-cycle.md
   ```
   Expected: zero matches outside the Anti-Patterns section other than prohibitions. Acceptable: the Anti-Patterns mentions explicitly forbidding these patterns; the Rule 18 prohibitions in the stage blocks, re-dispatch protocol, Handling Clarifications, and the `protocol_violation` stage definition (every one says "never"/"NOT"/"forbidden" next to `SendMessage`); and — when the parallel kit is installed — its sanctioned orchestrator→runner `SendMessage` lines in Parallel Orchestration / Permission Mode (filter with `| grep -vE "orchestrator|runner|TeamCreate|never|NOT|forbidden|returned agent"`). A `SendMessage` that is an *instruction to send* is a defect.

1a. **No pre-6.11 pipeline references remain:**
   ```bash
   grep -cE "bmad-create-story|bmad-dev-story|bmad-dev-auto|bmad-quick-dev|mode=data|mode=validate" .claude/commands/epic-cycle.md
   ```
   Expected: only the mentions that explain the retired path — the `SPRINT_PLAN` shorthand, the Model Strategy policy-file migration note, the Rework Loop, and Anti-Patterns (≤ 6 lines); zero in Task Sequence, Pipeline Flow, or any spawn block.

1b. **Finding disposition bar (Rule 15), Rule 16, and the v6.11 stage doctrine installed:**
   ```bash
   grep -c "Rule 15" .claude/commands/epic-cycle.md          # >= 3 (CR block, rework loop, X.0, SC-4)
   grep -cE "^## Rule (1[3-9]|20)" _bmad/custom/skill-rules.md   # 8 (13–20)
   grep -c "typically \`docs/adr/\`" .claude/commands/epic-cycle.md _bmad/custom/skill-rules.md   # 0 each (the spine is the registry; the only remaining `docs/adr` mention is the anti-pattern forbidding it)
   grep -c "Rule 20" .claude/commands/epic-cycle.md                      # >= 4 (clarification protocol, decision sheet, AD-Aware Layer 3, re-dispatch)
   grep -c "ARCHITECTURE-SPINE" .claude/commands/epic-cycle.md           # >= 3
   grep -c "spine_resolved" .claude/commands/epic-cycle.md               # >= 3 (Task Sequence, skeleton, stage enum)
   grep -c "lint_spine" .claude/commands/epic-cycle.md _bmad/custom/skill-rules.md   # >= 1 each
   grep -c "ledger_stray_bullets" .claude/commands/epic-cycle.md         # >= 2
   grep -c "free-text" .claude/commands/epic-cycle.md                    # >= 1 (dirty-tree row of the blocking-condition table)
   grep -c "1e. Sprint-status sync" .claude/commands/epic-cycle.md       # 1 (CR block pre-answers the YAML rewrite)
   grep -c "git apply" .claude/commands/epic-cycle.md                    # >= 1 (review-stage intent-gap patch restore)
   grep -c "ci=pending" .claude/commands/epic-cycle.md                   # >= 3 (Task Sequence, Pipeline Flow, CI Is a Gate, resume, telemetry)
   grep -c "merge_gate_blocked" .claude/commands/epic-cycle.md           # >= 2 (SC-4 precondition, stage enum)
   grep -c "Ledger inbox" .claude/commands/epic-cycle.md                 # >= 2 (plan block, spec gate)
   grep -c "routed_story_max" .claude/commands/epic-cycle.md             # >= 1
   grep -c "owner_unknown" _bmad/scripts/ledger.sh                       # >= 2 (load reports unknown owners)
   grep -c "LEDGER_OWNER_CHECK" _bmad/scripts/ledger.sh                  # >= 2 (validation + bypass)
   grep -c "Routed means planned" _bmad/custom/skill-rules.md            # 1 (Rule 17 (1b))
   grep -c "Rule 19" .claude/commands/epic-cycle.md                      # >= 4 (Task Sequence, QA block, CR block, AD gate, anti-pattern)
   grep -c "never owns a LOW" _bmad/custom/skill-rules.md                # >= 1 (Rule 15)
   grep -c "decision_sheet_recorded" .claude/commands/epic-cycle.md      # >= 4 (Task Sequence, Pipeline Flow, Ledger Drain, SC-4, stage enum)
   grep -c "highest existing story number" .claude/commands/epic-cycle.md   # >= 3 (Task Sequence, Pipeline Flow, Ledger Drain, anti-pattern)
   grep -c "highest existing story number" _bmad/custom/skill-rules.md      # 1 (Rule 17 (3))
   grep -c "N\.9" .claude/commands/epic-cycle.md                         # 1 (only the anti-pattern forbidding a fixed burn-down number)
   grep -c "N\.9" _bmad/custom/skill-rules.md                            # 0
   grep -c "burndown_story_max" .claude/commands/epic-cycle.md           # >= 3
   grep -c "drain_ratio" .claude/commands/epic-cycle.md                  # >= 3
   grep -c "mutations_demonstrated" .claude/commands/epic-cycle.md       # >= 2 (QA block, telemetry)
   grep -c "decision_dependency" .claude/commands/epic-cycle.md          # >= 2 (spec gate, telemetry)
   grep -c "Supervised, not unattended" .claude/commands/epic-cycle.md   # 1 (Permission Mode)
   grep -c "read, never recalled" .claude/commands/epic-cycle.md         # 1 (Cycle Log Format)
   grep -c "at least ten\|find at least N" _bmad/custom/bmad-build-auto.toml _bmad/custom/bmad-code-review.toml   # 0 each (hunter quota softened — 6.11 and 6.12 wordings)
   LA=$(grep -c '^instruction = """' _bmad/custom/bmad-build-auto.toml)    # active build-auto layers: 4, less any the project overlay switched off (instruction = "")
   LC=$(grep -c '^instruction = """' _bmad/custom/bmad-code-review.toml)   # active code-review layers: 4, likewise
   grep -c "Rule 19" _bmad/custom/bmad-code-review.toml                  # 1 (verification-gap layer; 0 only if the overlay switched it off)
   grep -c "one test class or suite" _bmad/custom/bmad-build-auto.toml   # LA + 1 (handoff + active layers — Rule 18 (3) test serialization)
   grep -c "one test class or suite" _bmad/custom/bmad-code-review.toml  # LC
   grep -c "Exit condition" _bmad/custom/skill-rules.md                  # 1 (Rule 15 — the loop can end)
   grep -c "Rework re-dispatch" .claude/commands/epic-cycle.md           # >= 2 (implement re-spawn block, Rework Loop, Pipeline Flow)
   grep -c "Rework re-review" .claude/commands/epic-cycle.md             # >= 3 (CR block, CR re-spawn block, Pipeline Flow)
   grep -c "rework_opened" .claude/commands/epic-cycle.md                # >= 3 (Pipeline Flow, Rework Loop, stage enum, telemetry)
   grep -c "rework_cap_override" .claude/commands/epic-cycle.md          # >= 3 (Pipeline Flow, Rework Loop, stage enum)
   grep -c "unresolved_high_med" .claude/commands/epic-cycle.md          # >= 4 (CR block, checkpoint, cr_complete, model_tier_checkpoint, telemetry)
   grep -c "mutates_shared_runtime" .claude/commands/epic-cycle.md       # >= 2 (spec gate, telemetry)
   grep -c "no HIGH closes the story" .claude/commands/epic-cycle.md     # >= 2 (Pipeline Flow, Rework Loop, anti-pattern)
   grep -ci "return exactly once" .claude/commands/epic-cycle.md         # >= 4 (plan, implement, QA, CR blocks)
   grep -c "Rule 18" .claude/commands/epic-cycle.md                      # >= 5
   grep -c "Containment (/epic-cycle Rule 18)" _bmad/custom/bmad-build-auto.toml   # LA + 1 (handoff + active layers)
   grep -c "Containment (/epic-cycle Rule 18)" _bmad/custom/bmad-code-review.toml  # LC
   grep -c "Synchronous is stated, never assumed" _bmad/custom/skill-rules.md   # 1 (Rule 18 (5))
   grep -c "run_in_background: false" .claude/commands/epic-cycle.md     # >= 8 (Permission Mode, invocation pattern, plan/implement/QA/CR blocks, pre-warm, anti-pattern)
   grep -h "^Launch a" _bmad/custom/bmad-build-auto.toml _bmad/custom/bmad-code-review.toml | grep -vc "run_in_background: false"   # 0 (every launch line names the flag — transformation (d))
   grep -c "run_in_background: false" _bmad/custom/bmad-build-auto.toml  # LA + 2 (handoff launch + handoff containment + one per active layer launch)
   grep -c "run_in_background: false" _bmad/custom/bmad-code-review.toml # LC (one per active layer launch)
   grep -c "timeout: 600000" _bmad/custom/skill-rules.md                 # 1 (Rule 18 (5) — the foreground cap and batching)
   grep -c "timeout: 600000" _bmad/custom/bmad-build-auto.toml           # 1 (handoff containment)
   grep -c "timeout: 600000" .claude/commands/epic-cycle.md              # >= 4 (plan, implement, QA, CR blocks)
   grep -c "Verification arrangement" .claude/commands/epic-cycle.md     # 1 (Per-Story Smoke: (loop)/(once) default, CI-fresh tiers, lead-owned full run)
   grep -c "(loop)" .claude/commands/epic-cycle.md                       # >= 3 (plan block, implement block, Per-Story Smoke)
   grep -c "fresh_check" .claude/commands/epic-cycle.md                  # >= 4 (Task Sequence, Per-Story Smoke, Pipeline Flow, stage enum)
   grep -c "Shared surfaces:" .claude/commands/epic-cycle.md             # >= 2 (plan block, CR block)
   grep -c "ci_red_classified" .claude/commands/epic-cycle.md            # >= 3 (CI Is a Gate, Pipeline Flow, stage enum)
   grep -c "merge-base --is-ancestor" .claude/commands/epic-cycle.md     # >= 3 (CI Is a Gate, Pipeline Flow, Cycle Log Format, anti-pattern)
   grep -c "story_cycle" .claude/commands/epic-cycle.md                  # >= 3 (Task Sequence, Pipeline Flow, stage enum)
   grep -c "stage_api_error" .claude/commands/epic-cycle.md              # >= 2 (Agent Invocation Pattern, stage enum)
   grep -c "launch_env" .claude/commands/epic-cycle.md                   # >= 3 (pre-flight, Permission Mode, stage enum)
   grep -c "BMAD v7 gate" .claude/commands/epic-cycle.md                 # 1 (pre-flight refuses BMAD v7)
   grep -c "CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH" .claude/commands/epic-cycle.md   # >= 2 (pre-flight, Permission Mode)
   grep -c "Extend, don't copy" _bmad/custom/skill-rules.md              # 1 (Rule 2)
   grep -c "Runtime targets get the same discipline" _bmad/custom/skill-rules.md   # 1 (Rule 13)
   grep -c "Read once, then work from what you hold" _bmad/custom/skill-rules.md   # 1 (Rule 13)
   grep -c "Comments state the contract" _bmad/custom/skill-rules.md     # 1 (Rule 14)
   grep -c "Hermeticity: a test passes for its own reasons" _bmad/custom/skill-rules.md   # 1 (Rule 19)
   grep -c "Read each file once" _bmad/custom/bmad-build-auto.toml       # 1 (handoff containment)
   grep -c "hermeticity" _bmad/custom/bmad-build-auto.toml _bmad/custom/bmad-code-review.toml   # 1 each (verification-gap brief; 0 only where the overlay switched that layer off)
   grep -c "LEDGER_ID_COUNTER" _bmad/scripts/ledger.sh                   # >= 2 (shared id counter for parallel runs)
   grep -c "armed=" .claude/commands/epic-cycle.md                       # >= 2 (checkpoint runs on every project)
   grep -c "ledger_adjudicated" .claude/commands/epic-cycle.md    # >= 5 (Task Sequence, Pipeline Flow, Ledger Drain, resume, stage enum)
   grep -c "ledger_burndown" .claude/commands/epic-cycle.md       # >= 5
   grep -c "LEDGER " .claude/commands/epic-cycle.md               # >= 12
   grep -c "Filing without draining" .claude/commands/epic-cycle.md   # 1
   grep -c "stage_spawned" .claude/commands/epic-cycle.md             # >= 8 (Task Sequence, invocation pattern, Pipeline Flow, resume, enum + definition, examples)
   grep -c "Propagate this prohibition" .claude/commands/epic-cycle.md  # >= 4 (plan, implement, QA, CR blocks)
   grep -c "resuming a pipeline stage agent" .claude/commands/epic-cycle.md   # 1 (anti-pattern)
   grep -c "story_planning" .claude/commands/epic-cycle.md            # 0 (folded into stage_spawned)
   grep -c "Fix Pack" .claude/commands/epic-cycle.md         # >= 2
   grep -c "Root freshness check" .claude/commands/epic-cycle.md  # 1 (SC-1)
   grep -c "Lead model gate" .claude/commands/epic-cycle.md       # 1 (pre-flight)
   grep -c "Halt after planning" .claude/commands/epic-cycle.md   # >= 4 (Task Sequence, skeleton, plan block, Pipeline Flow, X.0)
   grep -c "SPRINT_PLAN" .claude/commands/epic-cycle.md           # >= 10
   grep -c "bookkeeping commit" .claude/commands/epic-cycle.md    # >= 6
   grep -c "Auto Run Result" .claude/commands/epic-cycle.md       # >= 4
   grep -c "baseline_revision" .claude/commands/epic-cycle.md     # >= 5
   grep -cE "blind-hunter|edge-case-hunter|verification-gap|acceptance-auditor" .claude/commands/epic-cycle.md  # >= 3
   grep -c "After any context compaction, re-read this file" .claude/commands/epic-cycle.md   # 1 (near the top, inside what compaction keeps)
   grep -c "note=to_x0" .claude/commands/epic-cycle.md                 # >= 4 (overflow, decision sheet, X.0, anti-pattern — the deferred owner the tracker knows)
   grep -c "rev-parse HEAD\` still equals" .claude/commands/epic-cycle.md _bmad/custom/skill-rules.md   # >= 3 / 1 (subagent-commit checks are HEAD-based)
   grep -c "branches --not --remotes\` and" .claude/commands/epic-cycle.md _bmad/custom/skill-rules.md   # 0 / 0 (no repo-wide unpushed check remains)
   grep -c "epic_merge_started" .claude/commands/epic-cycle.md         # >= 4 (pseudo-code, SC-4, resume table and gates, log format)
   grep -c "record merge" .claude/commands/epic-cycle.md               # >= 3 (SC-3 exception, SC-4, pseudo-code)
   grep -c "UNPUSHED_NEW" .claude/commands/epic-cycle.md               # 1 (resume-mode rows for a log-less new branch)
   grep -c "story_base" .claude/commands/epic-cycle.md                 # >= 4 (Task Sequence, smoke, fresh check, log format)
   grep -c "story_base" _bmad/custom/skill-rules.md                    # 2 (Rule 17 (2): adjudication judges the whole delivered scope; Rule 16: the unattended set-aside)
   grep -c "TRACKER_READ" .claude/commands/epic-cycle.md               # >= 3 (shorthand, resume, anti-pattern)
   grep -c "\*.yaml\` workflow" .claude/commands/epic-cycle.md         # 2 (CI detection: pre-flight and "CI Is a Gate")
   grep -c "_bmad/bmm/config.yaml" .claude/commands/epic-cycle.md      # >= 1 (path-agreement check)
   grep -c "v7_global_accepted" .claude/commands/epic-cycle.md         # 1 (pre-flight reads the install's acceptance)
   grep -c "latest by its UTC timestamp" _bmad/custom/skill-rules.md   # 1 (Rule 15: effective state by timestamp)
   grep -c "planning decisions the same commit records" _bmad/custom/skill-rules.md   # 1 (Rule 16 bookkeeping scope)
   grep -c "No spine file" _bmad/custom/skill-rules.md                 # 1 (Rule 20 without a spine)
   ```

1c. **Every customization override parses and loads the rules file** (the v6.11 renderer HALTs on an unparseable override, so this check is mandatory):
   ```bash
   for s in bmad-build-auto bmad-build bmad-qa-generate-e2e-tests bmad-code-review; do
     uv run --no-cache _bmad/scripts/resolve_customization.py --skill "$PWD/.claude/skills/$s" --key workflow.persistent_facts | grep -c "skill-rules.md"
   done
   uv run --no-cache _bmad/scripts/resolve_customization.py --skill "$PWD/.claude/skills/bmad-build" --key workflow.open_spec
   uv run --no-cache _bmad/scripts/resolve_customization.py --skill "$PWD/.claude/skills/bmad-build-auto" --key workflow.implementation_handoff | grep -c "Rule 18"   # 1
   uv run --no-cache _bmad/scripts/resolve_customization.py --skill "$PWD/.claude/skills/bmad-build-auto" --key workflow.review_layers | grep -c "Rule 18"           # LA (one per active layer; confirms the id-keyed replace merged)
   uv run --no-cache _bmad/scripts/resolve_customization.py --skill "$PWD/.claude/skills/bmad-build-auto" --key workflow.review_layers | grep -c "run_in_background: false"   # LA (every active merged layer launches synchronously)
   uv run --no-cache _bmad/scripts/resolve_customization.py --skill "$PWD/.claude/skills/bmad-code-review" --key workflow.review_layers | grep -c "run_in_background: false"  # LC
   uv run --no-cache _bmad/scripts/resolve_customization.py --skill "$PWD/.claude/skills/bmad-code-review" --key workflow.review_layers | grep -c "Rule 19"          # 1 (verification-gap carries the falsifiability brief)
   uv run --no-cache _bmad/scripts/resolve_customization.py --skill "$PWD/.claude/skills/bmad-code-review" --key workflow.review_layers | grep -c "at least ten\|find at least N"     # 0 (the override replaced the upstream quota)
   uv run --no-cache _bmad/scripts/resolve_customization.py --skill "$PWD/.claude/skills/bmad-build-auto" --key workflow.review_layers | grep -c "at least ten\|find at least N"      # 0 (blind-hunter's quota softened)
   for s in bmad-build-auto bmad-code-review; do echo "$s merged=$(uv run --no-cache _bmad/scripts/resolve_customization.py --skill "$PWD/.claude/skills/$s" --key workflow.review_layers | grep -o '"id":' | wc -l | tr -d ' ') installed=$(grep -c '^\[\[workflow.review_layers\]\]' .claude/skills/$s/customize.toml)"; done   # merged = installed for each (an override id that matches no layer ADDS a layer, quota and all, instead of replacing one)
   ```
   Expected: `1` four times (each merged `persistent_facts` list contains the rules file); the last command prints the JSON object `{"workflow.open_spec": ""}` (an empty string value). Any `ConfigError` / non-zero exit means an override does not parse — fix it before continuing: the rendered skills would HALT on it, and the non-rendered ones (code-review, QA, sprint planning, retrospective) would silently fall back to a manual merge, which is worse.

1d. **Every placeholder in an override exists in the installed skill** (the derive-don't-paste rule — catches an upstream placeholder rename such as `{diff_output}` → `{diff_file}`):
   ```bash
   for s in bmad-build-auto bmad-code-review; do
     comm -23 <(grep -oE "\{[a-z_]+\}" _bmad/custom/$s.toml | sort -u) <(grep -oE "\{[a-z_]+\}" .claude/skills/$s/customize.toml | sort -u)
   done
   ```
   Expected: no output. Any placeholder printed is used by the override but unknown to the installed skill — re-derive that layer from the installed `customize.toml`.

2. **All the customization files exist, and the legacy pair is gone:**
   ```bash
   ls _bmad/custom/
   ls _bmad/custom/bmad-create-story.toml _bmad/custom/bmad-dev-story.toml 2>/dev/null | wc -l   # 0
   ```
   Expected: `skill-rules.md`, `bmad-build-auto.toml`, `bmad-build.toml`, `bmad-qa-generate-e2e-tests.toml`, `bmad-code-review.toml`, `epic-cycle-unattended.md`, `unattended.yaml` (plus `.bak-*` backups, any `.user.toml`, and the project's own files such as `kit-versions.yaml` or the overlay).

2b. **The ledger tool parses and round-trips** (run against a scratch file, never the project's ledger):
   ```bash
   bash -n _bmad/scripts/ledger.sh
   W="$(mktemp -d)" || exit 1; T="$W/ledger.md"
   bash _bmad/scripts/ledger.sh "$T" new "Self-test entry" "kit-step-4" low low in-story "installer self-test" open 0-0-selftest install "delete me"   # prints DW-1
   bash _bmad/scripts/ledger.sh "$T" append DW-1 "status=dropped by=install note=self-test"
   bash _bmad/scripts/ledger.sh "$T" load     # total=1 open=0 ... terminal=1
   bash _bmad/scripts/ledger.sh "$T" slice all | wc -l   # 0
   C="$W/ctr"; echo 7 > "$C"
   LEDGER_ID_COUNTER="$C" bash _bmad/scripts/ledger.sh "$T" new "Counter test" "kit-step-4" low low in-story "self-test" open 0-0-selftest install "x"   # prints DW-7 (shared counter wins over local next)
   cat "$C"   # 8
   echo x > "$C"; LEDGER_ID_COUNTER="$C" bash _bmad/scripts/ledger.sh "$T" new "Bad counter" "kit-step-4" low low in-story "self-test" open 0-0-selftest install "x"; echo "exit=$?"   # ERROR … exit=1 (a broken counter is never silently ignored)
   bash _bmad/scripts/ledger.sh "$T" new "Typo" "kit-step-4" low low in-story "self-test" opne 0-0-selftest install "x"; echo "exit=$?"   # ERROR … not in the grammar … exit=1 (Rule 15 statuses only)
   bash _bmad/scripts/ledger.sh "$T" append DW-1 "by=install note=status=open"   # text after note= is not a field
   bash _bmad/scripts/ledger.sh "$T" load | head -1   # total=2 open=1 … terminal=1 status_unknown=0 (DW-1 stays dropped)
   bash _bmad/scripts/ledger.sh "$T" new "Prefix" "kit-step-4" low low in-story "self-test" decision-pending 0-0-selftest install "note=human=x" >/dev/null; grep -c "note=note=" "$T"   # 0 (a caller's note= prefix is stripped)
   rm -f "$T" "$C"
   # owner validation against a scratch tracker (the ledger's sibling sprint-status.yaml)
   D="$W/tracker"; mkdir -p "$D"; printf 'development_status:\n  1-1-real-story: backlog\n' > "$D/sprint-status.yaml"
   bash _bmad/scripts/ledger.sh "$D/deferred-work.md" new "Owner test" "kit-step-4" low low in-story "self-test" routed 1-1-real-story install "ok"        # prints DW-1
   bash _bmad/scripts/ledger.sh "$D/deferred-work.md" new "Bad owner" "kit-step-4" low low in-story "self-test" routed 1-1-retitled-story install "x"; echo "exit=$?"   # ERROR … exit=1
   LEDGER_OWNER_CHECK=off bash _bmad/scripts/ledger.sh "$D/deferred-work.md" append DW-1 "owner=1-1-retitled-story by=install note=plant"
   bash _bmad/scripts/ledger.sh "$D/deferred-work.md" load | head -1   # … owner_unknown=1
   bash _bmad/scripts/ledger.sh "$D/deferred-work.md" slice unknown | wc -l   # 1
   printf '  1-2-gr\303\266\303\237e-anzeigen: backlog\n' >> "$D/sprint-status.yaml"
   bash _bmad/scripts/ledger.sh "$D/deferred-work.md" new "Unicode owner" "kit-step-4" low low in-story "self-test" routed "$(printf '1-2-gr\303\266\303\237e-anzeigen')" install "ok"   # prints DW-2 (a non-ASCII story key is a key)
   awk 'BEGIN { for (i = 1; i <= 600; i++) printf "  9-%d-padding-key-for-a-large-tracker-self-test: backlog\n", i }' >> "$D/sprint-status.yaml"
   bash _bmad/scripts/ledger.sh "$D/deferred-work.md" new "Large tracker" "kit-step-4" low low in-story "self-test" routed 1-1-real-story install "ok"   # prints DW-3 (a tracker past the pipe buffer still validates its FIRST key)
   bash _bmad/scripts/ledger.sh "$D/deferred-work.md" append DW-1 'by=install note=C:\new\tools and \d+' >/dev/null; grep -cF 'C:\new\tools and \d+' "$D/deferred-work.md"   # 1 (backslashes stored verbatim)
   printf '\n### DW-9: order\n- source: s | severity: low | fix-risk: low | footprint: in-story\n- evidence: e\n- 2026-01-02T00:00:00Z status=dropped by=install note=later\n- 2026-01-01T00:00:00Z status=open owner=burndown by=install note=earlier-but-last\n' >> "$D/deferred-work.md"
   bash _bmad/scripts/ledger.sh "$D/deferred-work.md" slice all | grep -c "DW-9"   # 0 (the latest timestamp wins, not the last line)
   echo 20 > "$D/ctr"; echo "pid=1 acquired_at=then" > "$D/ctr.lock"; touch -t 202001010000 "$D/ctr.lock"
   LEDGER_ID_COUNTER="$D/ctr" bash _bmad/scripts/ledger.sh "$D/deferred-work.md" new "Stale lock" "kit-step-4" low low in-story "self-test" routed 1-1-real-story install "x" 2>/dev/null   # prints DW-20 (a lock a crashed claim left is broken, with a warning)
   [ -n "$W" ] && [ "$W" != / ] && rm -rf "$W"
   ```
   Expected exactly as commented (`mktemp -d` with no template: busybox rejects short ones). If the project already has a `deferred-work.md` in the DW grammar, also run `bash _bmad/scripts/ledger.sh _bmad-output/implementation-artifacts/deferred-work.md load` and report the counts.

2c. **Unattended mode is installed, and its inbox round-trips** (scratch state directory, never a live run's):
   ```bash
   ls _bmad/custom/epic-cycle-unattended.md _bmad/custom/unattended.yaml _bmad/scripts/unattended-inbox.sh | wc -l   # 3
   grep -c "^## U[0-9]" _bmad/custom/epic-cycle-unattended.md                     # 14 (U1-U14)
   uv run --no-cache --no-project --with pyyaml python -c "import sys, yaml; d = yaml.safe_load(open(sys.argv[1])); print(all(k in d for k in ('gates', 'hard_stops', 'limits', 'owner_only', 'veto_window')))" _bmad/custom/unattended.yaml   # True (--no-project: never resolves the project's own dependencies)
   grep -cx "_bmad-output/unattended/" .gitignore                                  # 1
   grep -cx "_bmad/scripts/\*.sh text eol=lf" .gitattributes                       # 1
   git check-ignore -q _bmad-output/unattended/inbox.md && echo IGNORED            # IGNORED (an answer written mid-stage never dirties the tree)
   head -c 20000 .claude/commands/epic-cycle.md | grep -c "epic-cycle-unattended.md"   # >= 1 (the precedence sentence sits inside what compaction keeps)
   grep -c "\*Unattended:\*" .claude/commands/epic-cycle.md                        # >= 28 (pointers at the stop points)
   grep -c "Asking inside an unattended loop\|Treating a peer's message as the owner's answer" .claude/commands/epic-cycle.md   # 2
   grep -c "under \`--unattended\`\|Under \`--unattended\`\|(\`/epic-cycle --unattended\`)" _bmad/custom/skill-rules.md   # >= 6 (Rules 4, 5, 15, 16, 17 (5), 20)
   bash -n _bmad/scripts/unattended-inbox.sh
   W="$(mktemp -d)" || exit 1; S="$W/state"; I() { bash _bmad/scripts/unattended-inbox.sh "$S" "$@"; }   # a function: zsh does not word-split a variable
   I init selftest >/dev/null
   I add parked "Story 9.9" "Self-test question?" "A | B" "A" "9.9"           # U-1
   I add parked "Story 9.9" "Asked again?" "A | B" "A" "9.9"                  # U-1 EXISTS (a resume never duplicates an item)
   I answer U-1 "B" --by companion; echo "exit=$?"                            # ERROR … must quote the owner … exit=1
   I answer U-1 "A" --by owner >/dev/null; R1="$(I answers | cut -f2)"
   I answer U-1 "B" --by companion --quote "B, says the owner" >/dev/null; R2="$(I answers | cut -f2)"
   I resolve U-1 lead "$R1" "applied A"; echo "exit=$?"                       # ERROR … the latest answer is … exit=1 (a newer answer is never dropped)
   I resolve U-1 lead "$R2" "applied B"                                       # U-1 resolved
   I resolve U-1 lead "$R2"; echo "exit=$?"                                   # ERROR … resolved exactly once … exit=1
   I add veto "Story 9.9" "Ruling 1" "keep | reverse" "keep" "-"; I add veto "Story 9.9" "Ruling 2" "keep | reverse" "keep" "-"   # U-2 / U-3 (every veto ruling is its own item)
   I verify                                                                   # ok
   echo "answer: by hand" >> "$S/inbox.md"; I verify; echo "exit=$?"          # MISMATCH … exit=1
   I render 2>/dev/null; ls "$S" | grep -c "inbox.md.edited-"; I verify     # rendered / 1 / ok (the hand edit is saved aside, never applied)
   I hold "owner self-test" >/dev/null; I hold "auto: usage self-test"           # HOLD KEPT: owner self-test (an auto hold never replaces the owner's)
   I hold-clear "relaunch" --usage-only; echo "exit=$?"                          # HOLD KEPT: owner self-test … exit=5 (a relaunch never clears the owner's hold)
   I hold-clear "self-test" >/dev/null; ls "$S/HOLD" 2>/dev/null | wc -l         # 0 (an interactive resume clears a hold)
   T1="$(I run-start t1 lead | sed -n 's/.*token=//p')"; I run-start t2 lead >/dev/null; echo "exit=$?"   # exit=3 (one run at a time)
   T2="$(I run-takeover t1 lead "relaunch" | sed -n 's/.*token=//p')"
   I run-boundary t1 "$T1" "old session"; echo "exit=$?"                      # ERROR … not this session -- stop … exit=4 (a replaced session learns it at once)
   I run-end t1 "$T2" "self-test"                                             # run.lock released (self-test)
   [ -n "$W" ] && [ "$W" != / ] && rm -rf "$W"
   ```
   Expected exactly as commented.

3. **No `.toml` carries an `on_complete` hook:**
   ```bash
   grep -l "on_complete" _bmad/custom/bmad-*.toml
   ```
   Expected: zero matches. Verification gates and closing-summary instructions live in the slash command's spawn-prompt skeleton.

4. **No `.toml` references deprecated envelopes:**
   ```bash
   grep -E "STATUS: completed|STATUS: clarification_needed|SendMessage|shutdown_request" _bmad/custom/bmad-*.toml | grep -vE "(do not|may NOT) .?SendMessage.? any agent"
   ```
   Expected: zero matches. The containment text's own prohibitions — "do not SendMessage any agent" in every layer, "You may NOT `SendMessage` any agent" in the handoff — are filtered out; any other mention (an instruction to send, an envelope token) is a defect.

5. **Each `.toml` loads `skill-rules.md` via `persistent_facts`:**
   ```bash
   grep -c "persistent_facts" _bmad/custom/bmad-*.toml
   ```
   Expected: each file reports `1`.

6. **The slash command contains every section:** open `.claude/commands/epic-cycle.md` and confirm presence of: Pre-flight Runtime Check (with the BMAD + runtime gate and the `SPRINT_PLAN` shorthand), Task Sequence, Permission Mode, Model Strategy, Skill Tool Invocation, Agent Invocation Pattern (with Spawn Prompt Skeleton, Plan / Implement / QA / Code-review stage blocks, Clarification protocol with the build-auto blocking-condition table and re-dispatch protocol, Pipeline Flow), Smart Parallelism (retired), Rework Loop, Status Ownership, Per-Story Smoke, Ledger Drain, Epic Context Pre-warm, Retrospective Review & Story X.0 Creation, Source Control Branching (with the conditional IRIS/ObjectScript IDE file-sync toggle, Rules SC-1 through SC-8, and Tracker format flexibility), Resume Semantics, Sprint Planning Per Epic, Retrospective Per Epic, Lead Validates the Story Spec, Context Handoff Between Stages, Lead Context Management, AD-Aware Execution (the architecture spine, Layers 1–3), When to Pause, Handling Clarifications, Submodule / Sub-Repository Commit Order, Completion Logging, Workflow Telemetry, Anti-Patterns.

7. **Telemetry metadata is documented:**
   ```bash
   grep -c "spawn_at" .claude/commands/epic-cycle.md
   grep -c "model=" .claude/commands/epic-cycle.md
   grep -c "closing_sections_present" .claude/commands/epic-cycle.md
   ```
   First two ≥ 3 matches; third ≥ 2 matches.

8. **The slash command carries the model strategy and passes resolved models on spawns:**
   ```bash
   grep -c "Model Strategy" .claude/commands/epic-cycle.md
   grep -c "model resolved in step 1" .claude/commands/epic-cycle.md
   ```
   Expected: first ≥ 2, second ≥ 1 (the Agent Invocation Pattern resolves frontmatter-override-else-map and passes it on the `Agent` call).

9. **Interactive checkpoints are pre-answered and the rework loop is bounded:**
   ```bash
   grep -c "Pre-answered checkpoint" .claude/commands/epic-cycle.md
   grep -c "Rework Loop" .claude/commands/epic-cycle.md
   grep -c "SPRINT_PLAN status" .claude/commands/epic-cycle.md
   ```
   Expected: first ≥ 2 (QA + code-review stage blocks), second ≥ 3, third ≥ 3 (shorthand definition, resume detection, lead context management).

10. **Status ownership is documented with the v6 enum and the lead's writer:**
   ```bash
   grep -c "epic_status_done" .claude/commands/epic-cycle.md
   grep -c "ready-for-dev" .claude/commands/epic-cycle.md
   grep -c "generate --set" .claude/commands/epic-cycle.md
   ```
   Expected: first ≥ 3 (End of Epic step + Pipeline Flow + telemetry stage list), second ≥ 3, third ≥ 5.

11. **The retrospective human-in-the-middle exception is explicit:**
   ```bash
   grep -c "human-in-the-middle" .claude/commands/epic-cycle.md
   ```
   Expected: ≥ 4 (Execution Guidelines, Retrospective Per Epic heading + body, Pipeline Flow, telemetry stage list, Anti-Patterns).

12. **The slash command includes the conditional IRIS/ObjectScript IDE file-sync toggle:**
   ```bash
   grep -c "objectscript.conn.active\|IDE file-sync toggle" .claude/commands/epic-cycle.md
   ```
   Expected: ≥ 2 matches (the toggle subsection under Source Control Branching + the Pipeline Flow steps + the Anti-Pattern). The toggle must be conditional — it applies only when `.vscode/settings.json` has `objectscript.conn.active: true`, and is skipped entirely on non-ObjectScript projects.

13. **Every project-overlay entry is in place** (skip if `_bmad/custom/epic-cycle-overlay.md` is absent; when the parallel kit is installed, skip the entries whose `file` is `.claude/commands/epic-cycle.md` — Step 3c deferred them to its Edit 5.7, and its Step 7 checks them): for each entry, its target file contains the entry's text verbatim — check its first and last lines with `grep -cF` (≥ 1 each). A missing entry means Step 3c stopped or was skipped; fix the entry's anchor and re-apply before continuing.

---

## Step 5: Done

The workflow is ready to use. Projects can add custom rules to `_bmad/custom/skill-rules.md` (under "Project-specific rules") as retrospectives identify durable patterns, and can pin branch-naming conventions in `_bmad/custom/branch-naming.yaml` if the JIRA-style defaults don't fit. This kit document can be retained as the authoring source for future regeneration, or deleted — the slash command and customizations are self-contained at runtime.

**Upgrading from a pre-6.11 install:** `_bmad/custom/model-overrides.yaml` (if present) is keyed by skill in the old schema (`bmad-dev-story`, `bmad-quick-dev`, `bmad-qa-generate-e2e-tests`); offer to rewrite it to the stage-keyed schema (`implement`, `qa`) with a `history:` entry `action: schema-migrated-to-stages`. The `.bak-*` copies of `bmad-create-story.toml` / `bmad-dev-story.toml` can be deleted once the user is satisfied; restoring them only affects explicit by-name runs of those deprecated shims.

**Mid-epic upgrade (tell the user this whenever Step 1 found an in-flight epic — a `cycle-log-epic-*.md` with entries but no `epic_merged_to_feature` / `epic_merge_skipped`):**

- **Sequential run — safe at a story boundary.** Upgrade after a story's `committed` entry and before the next `stage_spawned` (or at any stage boundary with a clean tree and no spawn in flight — never while a `bmad-build-auto` / QA / code-review agent is running). The install commit lands on the epic branch (a chore commit under SC-3; it reaches the feature branch at the SC-4 merge). On the next `/epic-cycle` invocation resume does the rest: committed stories are skipped; sprint planning re-runs, so `ledger_load` runs; a story at `cr_complete` without `ledger_adjudicated` resumes at adjudication; an epic whose last story is committed but has no `ledger_burndown_*` entry runs the burn-down gate before `epic_status_done`.
- **Ledger migration is not optional mid-epic.** `ledger_load` halts on a legacy-grammar ledger, so the Step 1 migration must be part of the upgrade commit. Migrated entries are triaged by class at migration (human-owned and product calls → `decision-pending`, story-closable → `routed owner=burndown`, non-two-way-door LOWs → `wontfix-accepted`); this epic's burn-down gate is therefore the first real drain of the story-closable backlog and may charter a burn-down story (bounded by `burndown_story_max`), and this epic's SC-4 decision sheet is the first time the `decision-pending` entries reach the user — budget for both.
- **Upgrading to ≥ 2026-10-19.1 (`--unattended` and `--review`).** Attended runs are unchanged: the doctrine (`_bmad/custom/epic-cycle-unattended.md`) is read only under the new flags. The install writes `_bmad/custom/unattended.yaml` from its template when absent — **review it with the user before the first unattended run**: `owner_only` (what always waits for them), `veto_window` (what the loop decides and reports), `hard_stops`, `milestones` (which define "the release"), `product_defaults`, `owner_fixed` and `stage_preamble` are the project's to set. `.gitignore` gains `_bmad-output/unattended/`. Never upgrade under a live unattended run (Step 1 check 7): hold it first (`bash _bmad/scripts/unattended-inbox.sh <state> hold "upgrade"`, where `<state>` is `_bmad-output/unattended` or, for a parallel run, `<worktree_base>/.coordination/unattended`), wait until the run logs `hold_complete` and `run-status` prints `none`, upgrade, then re-invoke the same command to resume — the resume clears the hold.
- **Upgrading to ≥ 2026-10-18.1 (the adversarial review of the kit).** Safe at a story boundary; the orchestrator rule below still applies. (1) `ledger.sh` is replaced: drop any local patch to it — this version validates owners correctly on large trackers and non-ASCII keys, stores backslashes verbatim, reads effective state by timestamp, and breaks a crashed claim's counter lock. (2) An entry an earlier run parked on a not-yet-existing X.0 key (`owner=<N>-0-…` written with `LEDGER_OWNER_CHECK=off`) reads `UNKNOWN` at `ledger_load`: re-own it `owner=burndown note=to_x0:<what to decide>`. (3) The pre-flight now compares the resolver's artifact paths with `_bmad/bmm/config.yaml` and halts on a mismatch — align them before the next run. (4) A user-level v7 preview needs `v7_global_accepted:` in `kit-versions.yaml` (re-run Step 1 to record it), or the pre-flight halts. (5) A project whose CI is not GitHub Actions now reads `ci=unavailable` (the merge gate asks you to confirm CI) instead of `ci=none`. (6) Sequential merges now commit their merge record on the feature branch (`chore(epic-N): record merge`); an epic merged under an earlier version with an uncommitted `epic_merged_to_feature` line should have that line committed on the feature branch now.
- **Upgrading to ≥ 2026-10-14.1 (BMAD 6.12.2; the v7 gate).** BMAD 6.12.2 is a drop-in: no skill text or `customize.toml` changed, so the derived overrides are identical; its helper scripts now write UTF-8 on Windows, and `install --yes` without `--directory` installs into the current directory — `npx bmad-method@6.12 install --action update --modules bmm --tools claude-code --yes` upgrades a project unattended. Its installer now advertises a BMAD v7 preview (`npx skills add bmad-code-org/BMAD-METHOD`, then `bmad setup`). Do not apply it to a project that runs these kits: v7 replaces the review layers, sprint planning and story files they drive, and from this version Step 1 and the command's pre-flight refuse it (BMAD-V7 / V7-GLOBAL). `ledger.sh` now refuses a status outside the Rule 15 grammar (`LEDGER_STATUS_CHECK=off` for a migration), reads a trailer's fields only up to its `note=`, and strips a caller's `note=` prefix; run `LEDGER load` after the upgrade — `status_unknown=` above 0 names entries an earlier typo hid from the status-based gates, and a project that patched these into its own copy can drop the patch. Parallel runs: `parallel.yaml` gains an optional `integrate_conflicts` (absent = `ask`, today's behaviour); a project whose own rule lets runners resolve plain-intent conflicts during integrate-forward sets `resolve-plain` and retires that override. Nothing else to migrate.
- **Upgrading to ≥ 2026-10-10.1 (field report 2026-10-10; BMAD 6.12.1).** Safe at a story boundary. (1) Move every hand addition to the command or the overrides into `_bmad/custom/epic-cycle-overlay.md` before re-running the kit (Step 3c) — from now on they survive each upgrade. (2) The overrides are re-derived from the installed 6.12 `customize.toml` (`{diff_file}`, `{claims_file}`, the softened finding floor, the hermeticity brief); on BMAD 6.12.1, re-run BMAD's own installer first. (3) CI resolution point (1) moves to the next story's implement spawn; an in-flight epic picks it up at its next story. (4) New story-level entries `fresh_check` and `story_cycle`: a story committed before the upgrade is complete without them. Record the project's fresh-environment recipe under Project-specific rules if it has one; otherwise the check runs on a clean checkout. (5) Specs planned before the upgrade carry no `(loop)` / `(once)` marks and run their `## Verification` as before. (6) A project that wrote its own rules for what this version adds — read once, runtime targets, CI at the implement spawn, targeted tests, fresh-environment either-order checks, extend-don't-copy, prose discipline — retires the duplicates or keeps them under Project-specific rules where they say more. Nothing to migrate in the ledger: `LEDGER_ID_COUNTER` is used only when a parallel run sets it.
- **Upgrading to ≥ 2026-10-08.2 (long runs fit the cap).** Rule 18 (5) names the foreground cap (10 minutes, `timeout: 600000` on every test or build call, 2 minutes by default) and the batching rule (one class per call, or batches sized from the project's recorded timings to about 8 minutes), and says that arming `Monitor` and stopping is a return; Rule 18 (3) no longer lets a suite that outlasts the cap run as one call. File 2 is re-derived (the handoff's containment clause changed). Specs already planned with one long `## Verification` command need no edit: the implement stage splits it under the new rule. A project that already moved its full suite to the lead or runner by its own rule keeps it — that is now option (b) of the documented verification arrangement (Per-Story Smoke); if its rules name a timings file, that file is the batching source. Nothing to migrate.
- **Upgrading to ≥ 2026-10-08.1 (synchronous is stated).** Rule 18 gains item 5 — `run_in_background: false` on every `Agent` call at every depth, commands and tests in the foreground, no `Monitor` waits, no turn ended with work in flight — and the old item 5 (`ListAgents`) becomes item 6. File 2 / File 5 overrides are re-derived: transformation (d) puts the flag on every launch line, and the handoff's containment gains a foreground clause. A project that patched Rule 18 or its overrides by hand for the same purpose loses that patch to the overwrite and gets the kit's text in its place; move a project-specific incident note to "Project-specific rules" if it is worth keeping. Safe at a story boundary; nothing to migrate. Until the upgrade lands, an interim return is still handled by Rule 18 (4)'s orphan protocol.
- **Upgrading to ≥ 2026-09-13.1 (burn-down story numbering).** The burn-down story is `N.<M+1>` — one past the epic's highest existing story number across `epics.md` and `sprint-status.yaml`. An "Epic N burn-down" story chartered under an earlier version keeps its number and key; the gate reuses it. Nothing to migrate.
- **Upgrading to ≥ 2026-09-11.1 (the rework loop has an exit).** A story currently inside the rework loop closes at its next review that finds no HIGH: its unpatched MEDs are `routed` / `escalated` with owners (in-epic MEDs to the owning story's key), never re-opened; set its `baseline_commit` to the current rework's `baseline_revision` before that re-spawn (scoped re-review) and append the "Rework re-review" block. A cap override already in force (a `rework_cap_override` without `items=`) is re-recorded with the finite item list before any further spawn — an open-ended "continue until complete" is not a valid override under this version. File 2 / File 5 overrides are re-derived (the handoff and review-layer containment now serialize stateful test runs). `cr_complete` entries from before this version lack `rows=` / `unresolved_high_med=`, so the next model-tier checkpoint reads `high+med` for those stories and logs `metric=gross`. Specs already past `spec_validated` are not re-gated for `mutates_shared_runtime`; the lead applies the throwaway-environment requirement by hand to any in-flight story whose verification mutates a shared runtime.
- **Upgrading to ≥ 2026-08-30.2 (owner validation, ledger inbox, CI gate).** The first `ledger_load` after the upgrade validates every owner against the tracker and repairs (or halts on) `UNKNOWN` ones — run `bash _bmad/scripts/ledger.sh <impl>/deferred-work.md load` now and look at `owner_unknown=`. Entries already `routed` to stories with no `DW-n:` bullet get their bullets at the next gate that touches them (X.0 for the next epic's keys; the orchestrator's post-merge step) — or write them now for the next epic's stories if the pile is large (field case: fifteen on one story). `committed` entries from before the upgrade carry no `ci=`; the next resume treats the branch head's current run as the only one that matters and resolves it at the first CI resolution point.
- **Upgrading to ≥ 2026-08-30.2 (the spine is the registry).** There is no migration: the kit stops naming `docs/adr/` and resolves the project's `ARCHITECTURE-SPINE.md` at the next epic start (`spine_resolved`). A project that kept ADRs in `docs/adr/` should fold them into the spine as `AD-n` blocks with `bmad-architecture`'s Update intent (one interactive run) — the pipeline will not read the folder. Cycle-log identifiers (`adr_verifications_complete`, `adr_constrained_acs`) and `parallel.yaml`'s `adr_verifications` lock stage keep their spelling. Also from this version: File 2 / File 5 overrides are **derived from the installed `customize.toml`**, not pasted — re-run Step 2 for them after every BMAD update (Step 4 check 1d catches a stale placeholder).
- **Upgrading to ≥ 2026-08-30.1 mid-epic (Rule 15 / 17 / 19 changes).** The new disposition rows apply from the next code review; existing `burndown`-owned LOWs and MEDs are re-triaged at this epic's burn-down gate (fix-pack-sized → the burn-down story, else `wontfix-accepted`; MED/≤med in-epic → the burn-down story), and existing `escalated` / `decision-pending` entries reach the user at this epic's decision sheet. Rule 19's mutation requirement applies to stories whose QA has not yet run; stories already past `qa_complete` are not re-opened for it — the next code review's `verification-gap` layer will flag their unpinned ACs as findings if it sees them.
- **Pre-6.11 kit → this kit with a story mid-`bmad-dev-story`:** finish that story to `committed` under the old command (or revert it) BEFORE upgrading; the new resume logic reads spec files and cannot continue a half-done legacy story file.
- **Orchestrator (parallel) mode — do NOT upgrade with runners in flight.** The install commit lands in the main checkout; each runner's worktree is a checkout of its epic branch, which does not contain it, so runners keep the old command and rules — and live runners hold their old doctrine in context anyway. Finish and merge the in-flight epics first, then upgrade before the next dispatch. If waiting is impossible: merge the feature branch (carrying the upgrade commit) into every open epic branch by hand, stop the runners, and re-dispatch them at their resume points from a fresh orchestrator session.
- **Always restart** every session that has loaded `/epic-cycle` (see the warning below).

**Coexistence with `bmad-loop`:** if the project has the `bmad-loop` module installed (`_bmad/bmad-loop/`, `.bmad-loop/policy.toml`), tell the user that `/epic-cycle` and `bmad-loop` both drive `bmad-build-auto` and must not be run over the same stories concurrently — `bmad-loop` dispatches from a `bmad-spec` folder (`SPEC.md` + `stories.yaml`), `/epic-cycle` from `epics.md` + `sprint-status.yaml`; pick one per epic.

Record the install: create/update `_bmad/custom/kit-versions.yaml` with `epic-cycle-base: <this document's Kit-Version>` (companions record their own keys: `skill-optimization`, `parallel-epic-cycle`). The parallel kit's upgrade check reads this registry.

**Commit the OUTPUT, completely.** Stage the output set by path and commit it with a message naming the kit version applied — the output set is the installed files (`.claude/commands/epic-cycle.md`, `_bmad/custom/*` including `kit-versions.yaml`, `epic-cycle-unattended.md` and `unattended.yaml`, `_bmad/scripts/ledger.sh`, `_bmad/scripts/unattended-inbox.sh`, `.gitignore`, `.gitattributes`, a migrated ledger + its `.legacy.md` if Step 1 migrated one, `.claude/settings.json` if Step 3b ran, backups), NOT just the kit source documents. Never `git add -A`: it sweeps unrelated work in progress, and any untracked file that is not gitignored (a secret included), into the install commit. On `main`, `master` or `develop`, ask the user before committing there. Verify afterward that `git status --short` lists none of the output files; any other entry is the user's own work, which the next run's clean-tree gate will also stop on — tell them. (Pilot finding 2026-07-11: an upgrade session committed only the kit documents and left every output file dirty.)

**⚠️ Tell the user explicitly — session restart required.** A session that has already invoked `/epic-cycle` holds the OLD command text in its context; this install/upgrade takes effect only in NEW sessions and invocations. Instruct the user, verbatim: close this session and any session currently mid-`/epic-cycle`, and start a fresh session before the next `/epic-cycle`.

**Companion documents** (normally in the same directory as this kit): `skill-optimization-prompt.md` — the optional per-skill model-tier pass; run it AFTER this kit so the `/epic-cycle` command exists to receive its `model: opus` pin — and `parallel-epic-cycle-workflow-creation.md` — parallel epic execution via worktrees and runner subagents; requires this kit and offers to chain both prerequisites itself. **If either companion was previously applied and you just re-ran this kit, re-apply them now:** this install overwrote `.claude/commands/epic-cycle.md`, removing the model pin and the parallel patches (both companions are idempotent and safe to re-run).
