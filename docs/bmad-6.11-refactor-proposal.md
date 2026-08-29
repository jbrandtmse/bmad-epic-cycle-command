# Refactor proposal: `/epic-cycle` kits for BMAD Method v6.11

**Status:** Option A accepted 2026-08-26 and applied to all three kits at Kit-Version `2026-08-26.1` (see §8 for what the spikes showed and what remains unverified). **Baseline reviewed:** kits at Kit-Versions `2026-07-19.1` (base), `2026-07-11.3` (parallel), `2026-07-19.1` (model pass); BMAD v6.11.0 installed in this repo (`_bmad/_config/manifest.yaml`), `bmad-loop` v0.11.1; CHANGELOG + `src/` on GitHub `main`; docs.bmad-method.org workflow map.

---

## 1. Summary

BMAD v6.11.0 collapses Phase 4 into `bmad-sprint-planning → bmad-build → bmad-code-review`. Everything the base kit's per-story pipeline is built on has moved:

| Kit dependency | v6.11.0 state | Impact |
| --- | --- | --- |
| `/bmad-create-story` (lead-side story creation gate) | Deprecated shim, retained in full under `v6-shims/`; removed at v7. Prints a deprecation notice on every run. | Works today, dead at v7. Story creation is now `bmad-build`'s planning step (`spec-{slug}.md`). |
| `/bmad-dev-story` (Sonnet dev stage, review-continuation rework) | Same: deprecated, retained, removed at v7. | Works today, dead at v7. Rework mode has no successor as such (see §4.6). |
| `/bmad-sprint-status mode=data` / `mode=validate` (resume detection, lead context re-anchor, parallel status aggregation) | Shim to `bmad-sprint-planning`; **`data`/`validate` modes deleted** ("zero callers"). | **Broken now.** Replace with `sprint_plan.py status|validate` JSON or the skill's `action=status` headless view. |
| `_bmad/custom/bmad-create-story.toml`, `bmad-dev-story.toml` | Customization key = skill directory name; these still resolve, but only for the deprecated skills. `bmad-build`/`bmad-build-auto` load `_bmad/custom/bmad-build.toml` / `bmad-build-auto.toml`. | **`skill-rules.md` is not loaded by the new pipeline** until those two files exist. |
| `persistent_facts = ["file:{project-root}/…"]` | Unchanged schema; lists append to defaults; `file:` + `{project-root}` still supported. Renderer now **halts on an unparseable override**. | Kit TOMLs stay valid; add a parse check to validation. |
| `bmad-code-review` lanes "Blind Hunter / Edge Case Hunter / Acceptance Auditor" | Kept, not deprecated. Layers are now keyed by `id`: `blind-hunter`, `edge-case-hunter`, **`verification-gap` (new default)**, `acceptance-auditor` (`when review_mode = "full"`). Layer `instruction` is overridable per project. Still four happy-path HALTs (context confirm, decision-needed, patch menu, next-steps menu). | Update lane names/ids in the CR spawn block, `review_tier: mixed` logic, Rule 13. Pre-answers still needed and still match. |
| `bmad-qa-generate-e2e-tests` | Unchanged; `preceded-by: bmad-build`. | None. |
| `bmad-retrospective` | Rebuilt: `-H <epic>` headless, verdicts `accepted / accepted-with-open-items / rejected`, appends `action_items:` to `sprint-status.yaml`, `sprint_status.py detect-epic --epic N` unfinished-story gate. Sprint-mode artifact is still a dated file in `implementation-artifacts` (`epic-{N}-retro-*.md` glob still used by the skill itself). | Interactive human-in-the-middle run still valid. Story X.0 triage gains a second source: open `action_items`. |
| `bmad-sprint-planning` | Absorbs `check-implementation-readiness`: opens with a PASS/CONCERNS/FAIL readiness gate; headless JSON `{status, intent, gate, status_file, findings, warnings}`; deterministic `scripts/sprint_plan.py generate|status|validate`. | Sprint-planning gate should run headless and branch on `gate`. |
| `sprint-status.yaml` | **Format and story/epic vocabulary unchanged** (`backlog → ready-for-dev → in-progress → review → done`). `drafted`/`contexted` normalized. `action_items:` block added. | None on the file; ownership table changes (§4.3). |
| `deferred-work.md` (Rule 15 ledger) | Still the sink for `bmad-build` (`- source_spec / summary / evidence` bullets) and `bmad-code-review` (`## Deferred from: code review (date)` headings). **`bmad-build-auto` writes a `deferred:` list into spec frontmatter instead.** | Rule 15 needs a harvest step + format reconciliation (§4.5). |
| Output paths (`_bmad-output/implementation-artifacts`, `planning-artifacts`) | Unchanged; `_bmad/config.toml` `[modules.bmm]` keys. | None. |
| Runtime prerequisites | `uv` + Python 3.11 are **hard requirements**: `bmad-build`/`bmad-build-auto` render via `uv run render_skill.py` and HALT without it; every other skill calls `uv run resolve_customization.py`. New gitignored `_bmad/render/`. | Add a `uv` pre-flight to the command and the kits' Step 1; README prerequisite. |
| `model:` SKILL.md frontmatter | Still absent upstream, still not an official field. New upstream model-routing levers: `implementation_handoff` and `review_layers[].instruction` are free prose an override can point at any model/tool. | Model pass list needs renames; the stage→model map stays the enforcement path; new lever available for build-auto's internal subagents. |
| `bmad-quick-dev`, `bmad-dev-auto`, `bmad-review-*`, `bmad-editorial-review*`, research trio, `bmad-document-project`, `bmad-generate-project-context`, `bmad-sprint-status` | Forwarding shims. `bmad-check-implementation-readiness`, `bmad-agent-tech-writer`, `bmad-index-docs`, `bmad-shard-doc` **removed** (deleted from `.claude/skills` by the installer). | Model pass list is stale (§6). |
| Nested subagents / backgrounding | `bmad-build-auto` **mandates** subagents (HALT `no subagents`), forbids `run_in_background`, and requires a **clean working tree at step-01** and at finalize (it commits its own work and HALTs `finalization left repository dirty`). | Two hard constraints on how the kit spawns it (§4.2, §4.4). |

**Recommendation in one paragraph:** rebuild the per-story pipeline on `bmad-build-auto` as the implementation primitive (two spawns per story: *plan* at Opus with `Halt after planning.`, then *implement* at Sonnet), keep `bmad-qa-generate-e2e-tests` and an Opus `bmad-code-review` as the independent gate, move status ownership of `ready-for-dev`/`review` to the lead via `sprint_plan.py`, adopt "write-ahead → commit → spawn" so build-auto's clean-tree checks pass, and harvest build-auto's `deferred:` frontmatter into the Rule 15 ledger. Rename the customization files, retire `mode=data`, add a `uv` pre-flight, and refresh the model pass and parallel kit for the renamed/removed skills. Everything git-side (SC-1…SC-8, worktrees, merge queue, resume, IDE-sync toggle, telemetry) survives with small edits.

---

## 2. Strategic choice (decide first)

| Option | What it means | Verdict |
| --- | --- | --- |
| **A. Refactor onto `bmad-build-auto`** (recommended) | Per-story stages become build-auto (plan) → lead gates → build-auto (implement + self-review + commit) → ADR → QA → CR → smoke → commit. Keeps every kit value-add (branching doctrine, ADR gates, lead-side smoke, model tiering, Story X.0, telemetry, resume, parallel worktrees). | Upstream-sanctioned unattended primitive (it is what `bmad-loop` drives); explicit machine halt contract replaces Rule 9 pre-answer hacks for the dev stage; survives v7. |
| B. Adopt `bmad-loop` and shrink `/epic-cycle` to a wrapper | Python orchestrator drives `bmad-build-auto` over a `stories.yaml` folder (stories mode), with its own gates, worktree isolation and deferred-work sweep. | Loses sprint-status/epics.md flow (it dispatches from `SPEC.md` + `stories.yaml`), the SC-* branching doctrine, ADR gates, per-story smoke, Opus code-review gate, submodule handling. Worth a later evaluation; not a replacement for the kit today. Note the overlap in the README so users don't run both on the same stories. |
| C. Pin to the shims (`bmad-create-story`/`bmad-dev-story`) and change nothing else | Only fix `mode=data` and lane names. | Buys time until v7, but every run prints deprecation notices, the shims no longer receive upstream fixes, and story creation misses the new epic-context/continuity/matrix-test machinery. Acceptable as a stop-gap branch only. |

The rest of this document assumes **A**. Where C is trivially reachable (a "legacy pipeline" flag), it is called out.

Why `bmad-build-auto` rather than `bmad-build` as the spawned primitive: `bmad-build` is structurally interactive (`CHECKPOINT 1 [A]/[E]`, `[S]/[K]` multi-goal halt, resume-list halt, push/PR offer, and `open_spec` runs `code -r …` — opening VS Code from inside a subagent). Pre-answering all of that in a spawn prompt is the same fragile Rule 9 trick the kit already tolerates for code-review, but here it is avoidable: build-auto is the same step files with those checkpoints removed and a terminal `status` contract instead. (If a project insists on `bmad-build`, ship `_bmad/custom/bmad-build.toml` with `open_spec = ""` and pre-answer the four halts — document as unsupported.)

---

## 3. Target per-story pipeline (v6.11 shape)

```
Per epic (unchanged): clean-tree gate → resume detection → SC-1/SC-2 → checkout epic branch
  → sprint planning (headless; gate PASS/CONCERNS/FAIL) → retro-review + Story X.0 gate

Per story:
  0. Lead asserts branch; lead COMMITS bookkeeping (cycle log, sprint-status.yaml) — tree must be clean before any build-auto spawn
  1. PLAN     Agent(bmad-build-auto, model=opus, prompt: story key + "Halt after planning.")
              → spec-{epic}-{story}-{slug}.md, status ready-for-dev          logs: story_created (path=…, build_status=ready-for-dev)
  2. GATES    Lead: Integration-AC validation (Rule 1/2) on the spec; ADR mapping for ACs
              Lead: sprint_plan.py generate --set <key>=ready-for-dev ; commit spec + bookkeeping
  3. IMPL     Agent(bmad-build-auto, model=sonnet, prompt: spec path)
              → implement (handoff subagent) → 4-layer self-review at Sonnet → Finalize: status done, LOCAL COMMIT
              Lead: sprint_plan.py generate --set <key>=in-progress before spawn, =review after `done`
              Lead harvests spec frontmatter `deferred:` → deferred-work.md (Rule 15)   logs: dev_complete (build_sha=…, review_loop_iteration=…, followup_review_recommended=…, deferred=N)
              `blocked` + blocking condition → Clarification protocol (§4.4)
  4. ADR      Lead-side ADR-tooled verifications (unchanged)                    logs: adr_verifications_complete
  5. QA       Agent(bmad-qa-generate-e2e-tests, model=sonnet) (unchanged)      logs: qa_complete
  6. CR       Agent(bmad-code-review, model=opus, spec path ⇒ review_mode=full, diff = baseline_revision..HEAD + working tree)
              → sets story done | in-progress                                    logs: cr_complete
  7. REWORK   ≤3 iterations (§4.6)
  8. SMOKE    Lead-side (unchanged)                                              logs: smoke_complete
  9. COMMIT   Lead commits QA/CR/smoke changes + bookkeeping; pushes epic branch (unchanged SC-3)   logs: committed (sha=…)
```

Model tiers are unchanged in spirit: the story spec (highest-leverage judgment) at Opus, implementation at Sonnet, independent adversarial gate at Opus. build-auto's internal handoff subagent and review layers inherit the spawn's model ("same model capability as the current session"), so the *implement* spawn at Sonnet keeps its self-review at Sonnet — cheap, and the Opus `bmad-code-review` remains the gate. This also honours BMAD's own note in the sprint-status template: "Dev moves story to 'review', then runs code-review (fresh context, different LLM recommended)."

Cost note: per story you now pay one Sonnet 4-layer self-review (inside build-auto) plus the Opus 4-layer code-review. If telemetry shows the Sonnet self-review adds little, `_bmad/custom/bmad-build-auto.toml` can trim layers by `id` (empty `instruction` disables one; at least one must remain or build-auto halts `no active review layers`).

---

## 4. Design decisions that need explicit doctrine

### 4.1 Two spawns per story (plan / implement)

`bmad-build-auto` honours `Halt after planning.` in the invocation prompt: it writes the spec, sets `status: ready-for-dev`, and HALTs. Re-invoking it with the spec path resumes at step-03 (its resume table: `ready-for-dev`/`in-progress` → implement, `in-review` → review, `done` → follow-up review pass, `blocked` → halt `blocked spec supplied`). This restores the kit's "lead validates the story before dev" gate and the Opus-plan / Sonnet-implement split without any shim. It is also exactly the `spec_checkpoint` semantics `stories.yaml` defines, so the doctrine matches upstream vocabulary.

Dispatch form for the plan spawn: the sprint-status story key (`3-2-digest-delivery`) as the intent — build-auto's epic-story path resolves `{epic_num}`/`{story_num}`, compiles/reuses `epic-<N>-context.md`, and loads the previous `done` story in the epic for continuity. Do **not** use folder+id dispatch (that needs a `bmad-spec` folder with `SPEC.md` + `stories.yaml`; the kit stays in sprint mode).

### 4.2 Clean tree before every build-auto spawn ("write-ahead → commit → spawn")

build-auto step-01 item 3: "require a clean working tree … HALT on a dirty tree". Finalize: commits reviewed-diff files, then "Verify the version-controlled working copy is clean. Otherwise HALT … `finalization left repository dirty`". The kit's write-ahead cycle log (uncommitted `cycle-log-epic-{N}.md`) trips both. Doctrine change:

- Every cycle-log / sprint-status write that precedes a build-auto spawn is followed by a **bookkeeping commit** on the epic branch (`chore(epic-N): cycle log` style; SC-3 branch assertion applies). Same for the spec after the plan spawn.
- The IDE file-sync toggle rule already forbids staging `.vscode/settings.json`; add: bookkeeping commits stage only `_bmad-output/implementation-artifacts/{cycle-log-*,sprint-status.yaml,deferred-work.md,spec-*.md}`.
- build-auto's own finalize commit is recorded as `build_sha=` on `dev_complete`; the story's final `committed sha=` remains the lead's post-smoke commit. Resume integrity check 1 (`committed sha` reachable) is extended to `build_sha`.
- Parallel kit: `.worktrees/` is gitignored and the coordination dir lives under it, so worktrees stay clean; `dispatch.yaml` writes never touch the worktree.

### 4.3 Status ownership (rewrite the table)

build-auto never touches `sprint-status.yaml`; `bmad-build` would, but we don't use it. New table:

| Transition | Written by |
| --- | --- |
| file generation, `backlog` seeding, epic keys, `epic-N-retrospective: optional` | `bmad-sprint-planning` (headless) via `sprint_plan.py generate` |
| story `backlog → ready-for-dev` (after plan spawn) | **lead**: `sprint_plan.py generate … --set <key>=ready-for-dev` |
| story `→ in-progress` (+ epic `backlog → in-progress` lift) | **lead**, immediately before the implement spawn (`--set <key>=in-progress --set epic-N=in-progress`) |
| story `in-progress → review` | **lead**, after build-auto returns `done` |
| story `review → done` (or back to `in-progress`) | `bmad-code-review` step-04 §6 (unchanged; needs `{story_key}` set in the spawn prompt) |
| `epic-N-retrospective: optional → done`, `action_items` append | `bmad-retrospective` via `sprint_status.py update` (unchanged) |
| epic `in-progress → done` | **lead** (unchanged) |

Mechanics: `uv run --no-cache .claude/skills/bmad-sprint-planning/scripts/sprint_plan.py generate --epic-file <planning_artifacts>/epics.md --status-file <impl>/sprint-status.yaml --stories-dir <impl> --project <name> --date "<MM-DD-YYYY HH:MM>" --set <key>=<status>` — deterministic, never downgrades, preserves comments, JSON result. (`generate` is the only subcommand with `--set`; the story-file floor only recognises `<key>.md`, not `spec-<key>-*.md`, so the lead's explicit `--set` is required.) Resume snapshot: `sprint_plan.py status --status-file …`; integrity: `sprint_plan.py validate --status-file …`. Both replace every `/bmad-sprint-status mode=data|validate` reference (resume detection, Lead Context Management re-anchor, parallel Status aggregation, Anti-Patterns bullet).

Vocabulary note for the command text: the **spec frontmatter** status enum is `draft | ready-for-dev | in-progress | in-review | done | blocked`; the **sprint-status** enum is unchanged. `in-review` (spec) ≠ `review` (sprint-status). Never copy one into the other.

### 4.4 Contract mapping: build-auto HALT → kit protocol

build-auto ends every run with the HALT protocol: `status: <done|blocked|ready-for-dev>` in spec frontmatter plus a `## Auto Run Result` section (`Status:` / `Blocking condition:`). The lead reads the spec file (not the agent's prose) as the source of truth:

| build-auto result | Lead action |
| --- | --- |
| `ready-for-dev` (plan spawn) | log `story_created`; run gates |
| `done` | harvest `deferred:`; set `review`; log `dev_complete` |
| `blocked` — `intent gap`, `unclear intent`, `missing previous-story continuity decision`, `matrix ambiguity`, `spec failed ready-for-development standard` | `## Clarification Needed` path: surface blocking condition + `## Auto Run Result` to the user; on answer, amend the spec (Design Notes / non-frozen sections — or the `<intent-contract>` block only with the user's decision) and re-spawn on the spec |
| `blocked` — `version-control metadata not writable`, `finalization left repository dirty`, `no subagents` | kit/environment defect: halt loudly, do not re-spawn until fixed (usually a missed bookkeeping commit) |
| `blocked` — `review repair loop exceeded 5 iterations (non-convergence)`, `implementation verification failed`, `patch verification failed`, `matrix test audit failed` | treat like a failed rework cap: surface to user with the spec's Review Triage Log |
| `blocked` — `no epic spec found`, `no stories.yaml found`, `story id not found …` | folder+id dispatch was used by mistake — kit bug |

Rule 4's closing-summary sections are **not** required of the build-auto spawn (its contract is the spec); keep Rule 4 for the QA and CR spawns. Files-modified handoff to QA/CR comes from `git diff --name-only <baseline_revision>..HEAD` plus `git status --short`, not from prose.

### 4.5 Rule 15 ledger reconciliation

Three writers, two shapes:

- `bmad-build-auto` → spec frontmatter `deferred: [{summary, evidence, location?, severity?}]`
- `bmad-code-review` → `deferred-work.md` under `## Deferred from: code review (<date>)`, one bullet per finding
- kit Rule 15 → canonical entries with `status:` / fix-risk / occurrences / appended annotation lines

Proposal: keep `deferred-work.md` as the single ledger (Story X.0 triage, merge-gate sweep, `merge=union` attribute all depend on it) and make the lead the harvester: after each `dev_complete`, copy every new `deferred:` item into `deferred-work.md` as a Rule 15 canonical entry (`source_spec`, `summary`, `evidence`, `status: routed|open`, severity from the item, fix-risk assessed by the lead) — the same harvest `bmad-loop` does. Code-review's bullets keep their upstream heading; Rule 15 fields are appended as annotation lines beneath each bullet rather than reformatting upstream output. Rule 15's "consult the ledger before filing" instruction moves into the CR spawn block only (build-auto has no ledger access by design).

### 4.6 Rework loop after code-review

`bmad-dev-story`'s review-continuation mode has no successor. When `bmad-code-review` leaves the story `in-progress` (unresolved high/medium, or patches left as action items):

1. Lead sets the spec frontmatter `status: in-progress` and appends the unresolved findings as unchecked items under `## Tasks & Acceptance` (non-frozen section; the `<intent-contract>` block is untouched).
2. Bookkeeping commit; re-spawn build-auto (Sonnet) on the spec — resume table routes `in-progress` → implement → self-review → finalize.
3. Re-run ADR verifications if touched; re-spawn `bmad-code-review`.
4. Cap stays at 3 iterations, then surface.

The Fix Pack path (trivial LOW) becomes: reviewer patches inline (already pre-answered "apply every patch"); anything needing a dev iteration takes the loop above. **Needs a pilot** — the alternative (spawn build-auto with a freeform "fix these findings" intent) creates a second spec and is rejected.

### 4.7 Sprint planning gate

Invoke `/bmad-sprint-planning` headless (`-H` / "headless" in the Skill args) with intent sprint-planning. Parse the trailing JSON: `gate: FAIL` → halt and surface `findings` (+ saved `implementation-readiness.md`); `CONCERNS` → log `sprint_planning_complete gate=CONCERNS` and surface once, continue; `PASS` → continue; `status: blocked` → halt (duplicate epic versions / orphans need a human). Re-run on resume is still idempotent.

### 4.8 Retrospective and Story X.0

Keep the retrospective fully interactive (human-in-the-middle exception unchanged); invoke with the explicit epic number so its `detect-epic --epic N` gate runs. New for the retro-review gate: read `action_items` with `status: open` from `sprint-status.yaml` as a third triage source alongside `epic-{N-1}-retro-*.md` and `deferred-work.md`; Story X.0 closes items via `sprint_status.py update --set-action-status` only with the user's confirmation (the skill's own rule). Story X.0 creation: build-auto plan spawn with a freeform intent (title + the triage table as the description) and `Halt after planning.`; the lead appends the triage table to the resulting spec.

### 4.9 Epic context cache

`epic-<N>-context.md` is compiled by build-auto's subagent on the first story and reused while no file in `planning-artifacts` is newer. Rule 5 (NFR tripwire amends planning artifacts) therefore invalidates it — expected; note it so a recompile on the next story isn't mistaken for a bug. Optionally pre-warm it at epic start with a lead-side Opus subagent running `compile-epic-context.md` (cheaper than compiling inside the Sonnet spawn).

### 4.10 Spawn depth and the parallel kit

Orchestrator → runner (background) → build-auto agent → handoff subagent / review layers = depth 3 nested `Agent` calls. The parallel kit's 2026-07-09 spikes verified depth 2. Two options, pick after a spike:

- **Spawned** (default proposed): runner spawns build-auto as an `Agent` exactly as the classic lead does. Needs a depth-3 spike.
- **Inline fallback**: runner invokes `bmad-build-auto` via `Skill` (depth 2 for its subagents) and routes the implementation tier through `_bmad/custom/bmad-build-auto.toml` — `implementation_handoff` prose that says "launch the subagent with `model: sonnet`" — and, if desired, `review_layers[].instruction` likewise. This is the upstream-sanctioned model-routing lever and keeps the Opus runner as the planner. Cost: the runner's context absorbs the workflow text (against Lead Context Management doctrine).

---

## 5. Concrete change list

### 5.1 `epic-cycle-workflow-creation.md` (base kit) — bump to a new Kit-Version

**"What this installs" table + Step 2 files**

- Remove `bmad-create-story.toml`, `bmad-dev-story.toml`. Add `bmad-build-auto.toml` and `bmad-build.toml` (both `[workflow] persistent_facts = ["file:{project-root}/_bmad/custom/skill-rules.md"]`; `bmad-build.toml` additionally `open_spec = ""` so a stray manual/inline `bmad-build` never launches VS Code from a pipeline session). Keep `bmad-code-review.toml`, `bmad-qa-generate-e2e-tests.toml`. Optional: `bmad-sprint-planning.toml` and `bmad-retrospective.toml` with the same fact file (both accept `persistent_facts`) — *not shipped in 2026-08-26.1; projects can add them by hand.*
- Step 1 detection: add `ls _bmad/custom/bmad-create-story.toml _bmad/custom/bmad-dev-story.toml` → on presence, back up and **delete** (or offer to keep for explicit shim runs); detect BMAD version from `_bmad/_config/manifest.yaml` and refuse `< 6.11.0` (point at the pre-6.11 kit tag); add `uv --version` and Python ≥ 3.11 checks; check `_bmad/scripts/render_skill.py` exists.
- Step 4 validation: add `uv run _bmad/scripts/resolve_customization.py --skill .claude/skills/bmad-build-auto --key workflow.persistent_facts` and the same for `bmad-build`, `bmad-code-review`, `bmad-qa-generate-e2e-tests` — expected: the rules file appears in the merged list, exit 0 (this is the only way to prove the override parses, since the renderer halts on a bad TOML). Replace grep expectations that reference `bmad-dev-story`, `ready-for-dev` counts, `mode=data`.

**Command (`.claude/commands/epic-cycle.md`) sections**

| Section | Change |
| --- | --- |
| Pre-flight Runtime Check | Add: `uv --version` gate (HALT with install pointer); BMAD version gate ≥ 6.11.0 from manifest; verify `.claude/skills/bmad-build-auto/SKILL.md` exists. |
| Task Sequence — Per Story | Rewrite to §3 (plan spawn → gates → implement spawn → ADR → QA → CR → rework → smoke → commit). |
| Execution Guidelines | Lead-side skills are now: sprint planning (headless), retrospective (interactive). Story planning is a **spawned** Opus build-auto stage (the race-ahead rationale is gone: build-auto never advances to another story). |
| Permission Mode | Add build-auto's own rule: it mandates subagents and forbids backgrounding — the pipeline-stage agent must have the `Agent` tool available (`subagent_type: general-purpose`). |
| Model Strategy table | Rows: Plan → `bmad-build-auto` → `opus`; Implement → `bmad-build-auto` → `sonnet` (internal handoff + self-review inherit); QA → `sonnet`; Code review → `opus`. Note the new `implementation_handoff` / `review_layers[].instruction` levers as the sanctioned alternative for `review_tier: mixed` and for the inline fallback. Resolution order unchanged (overrides file → frontmatter → map). Model-tier checkpoint escalation targets become `bmad-build-auto` (implement stage) and `bmad-qa-generate-e2e-tests`; drop `bmad-quick-dev`. |
| Agent Invocation Pattern / Spawn Prompt Skeleton | Two build-auto prompt skeletons (plan: story key + `Halt after planning.` + Rule 1/2 + ADR path; implement: spec path only + Rule 5/6/13/14). Step 3 "reads closing summary" becomes "reads spec frontmatter `status` + `## Auto Run Result`" for build-auto stages. |
| Stage-specific rule blocks | Replace the **Dev spawn** block with **Plan spawn** and **Implement spawn** blocks (no closing-summary directive; no git prohibition — build-auto *must* commit; instead: "do not push"). **Code-review spawn**: pre-answer 1 becomes "explicit argument = spec path `<path>` → review_mode `full`; diff = `git diff <baseline_revision>` plus untracked files; story key `<key>`"; lane names → ids (`blind-hunter`, `edge-case-hunter`, `verification-gap`, `acceptance-auditor`); `review_tier: mixed` re-tiers `blind-hunter`/`edge-case-hunter`/`verification-gap` and leaves `acceptance-auditor` inherited; next-steps menu option 3 (Done) unchanged. Rule 15 ledger instructions per §4.5. |
| Clarification protocol | Add the build-auto `blocked` mapping table (§4.4). |
| Pipeline Flow | Rewrite the per-story block; add bookkeeping commits; replace `/bmad-sprint-status mode=data` with `sprint_plan.py status`; sprint planning headless + gate branch. |
| Smart Parallelism | "Story-file creation stays sequential" → plan spawns may run as a batch too (each build-auto invocation is self-contained), but keep sequential by default because each plan spawn reads the previous `done` story for continuity. Batch implement spawns unchanged (disjoint files). Bookkeeping commits serialize. |
| Rework Loop | Replace with §4.6. |
| Status Ownership | Replace with §4.3 table + the `sprint_plan.py generate --set` mechanics + spec-vs-sprint vocabulary note. |
| Retrospective Review & Story X.0 | Add `action_items` source; X.0 via build-auto plan spawn (§4.8). |
| Sprint Planning Per Epic | Headless + gate handling (§4.7). |
| Retrospective Per Epic | Pass explicit epic number; note the verdict vocabulary and that `rejected` on unfinished stories is expected if the range ended early. |
| Lead Creates Story Files | Rename "Lead validates the story spec" — the gate content (Integration AC check) is unchanged; the creation is the Opus plan spawn. |
| Context Handoff | Story file path → spec path; file lists from `baseline_revision..HEAD` + working tree. |
| Lead Context Management | `mode=data` → `sprint_plan.py status`; re-anchor also reads the current spec's frontmatter `status`. |
| Resume Semantics | Resume-point computation gains build-auto's spec status as evidence (a `story_created` entry + spec `done` + `build_sha` reachable but no `dev_complete` → write the missing entry, don't re-spawn); integrity check 1 covers `build_sha`; new "dirty tree at resume" note: a crashed build-auto can leave a partial implementation uncommitted — surface, never auto-clean. |
| Cycle Log Format | `story_created` gains `build_status=ready-for-dev spec_tokens=`; `dev_complete` gains `build_sha= review_loop_iteration= followup_review_recommended= deferred= baseline_revision=`; add `bookkeeping_committed sha=` (optional, for resume forensics); `sprint_planning_complete gate=`. |
| Anti-Patterns | Update: "Story-creator agent races ahead" → reword (build-auto never advances; the anti-pattern is *omitting the explicit story key*); "Double-writing story statuses" → invert (lead owns `ready-for-dev`/`in-progress`/`review`; never let build-auto's absence of sync surprise you); "Hand-parsing sprint-status" → `sprint_plan.py`; "Letting dev-story self-select" → "invoking build-auto without a story key / spec path"; add "Spawning build-auto on a dirty tree", "Using `bmad-build` (interactive) as a pipeline stage", "Using folder+id dispatch in sprint mode", "Disabling every build-auto review layer". Remove the `bmad-dev-auto` reference in the backgrounding bullet (now `bmad-build-auto`). |

**Step 3b / Step 5**: unchanged except the companion re-apply note and a new line telling users to delete `_bmad/custom/bmad-create-story.toml` / `bmad-dev-story.toml` backups once satisfied.

### 5.2 `_bmad/custom/skill-rules.md`

| Rule | Edit |
| --- | --- |
| 1, 2 (Integration ACs, Consumed-by) | Applies-to → `bmad-build-auto` planning step (and `bmad-build`). Wording: "the spec's Tasks & Acceptance / I-O matrix must include …"; the `## Consumed-by` / `## Consumes` sections go under Design Notes (non-frozen) so later amendments don't fight the `<intent-contract>`. |
| 3 | Unchanged. |
| 4 (closing summary) | Scope to QA and CR spawns; build-auto spawns use the HALT/`## Auto Run Result` contract instead. |
| 5, 6 (NFR tripwire, ADRs) | Applies-to → `bmad-build-auto`, `bmad-code-review`. Rule 5 halt: build-auto has no "amend planning artifact and continue" path — halting with `blocked` + `intent gap`/questions is the correct behaviour; the lead amends the planning artifact, then re-spawns. Rewrite accordingly. |
| 7 | Unchanged (add: build-auto's handoff subagent is depth +1 from the stage agent). |
| 8 | Unchanged. |
| 9 (unattended menu protocol) | Now applies to `bmad-code-review` and `bmad-qa-generate-e2e-tests` only; build-auto has no menus; `bmad-build` is not a pipeline stage. Retro exception unchanged. |
| 10–12 (parallel) | Unchanged. |
| 13 | Lane names → layer ids; also names build-auto's handoff/review subagents. |
| 14 | Applies-to → `bmad-build-auto`, QA, CR. |
| 15 | Add the harvest rule (§4.5) and the code-review heading compatibility note. |
| New Rule 16 — Clean tree before build-auto | The "write-ahead → commit → spawn" invariant and the bookkeeping-commit file allow-list. |

### 5.3 `skill-optimization-prompt.md`

- Preflight: add `uv` check; BMAD ≥ 6.11 check; note that `removals.txt` skills are gone.
- Tier lists: `bmad-quick-dev` → `bmad-build` (sonnet, with the same "standalone runs inline on the session model" caveat), `bmad-dev-auto` → `bmad-build-auto` (**pin `opus`** — the plan spawn is what the pin documents; the implement spawn is forced to `sonnet` by the stage map, matching the existing "lead-at-opus + stage-map tiers" guidance), `bmad-dev-story`/`bmad-create-story` → shims: pin to `sonnet`/`opus` respectively for explicit-name runs, flagged "removed at v7"; `bmad-review` → opus (new core skill; the three `bmad-review-*` and three `bmad-editorial-review*` become shims pinned to opus to match their target); `bmad-deep-recon` → sonnet (research trio become shims → sonnet); `bmad-project-context` → sonnet (document-project / generate-project-context shims → sonnet); `bmad-sprint-status` → shim → haiku (target `bmad-sprint-planning` is haiku); delete `bmad-check-implementation-readiness`, `bmad-agent-tech-writer`, `bmad-index-docs`, `bmad-shard-doc`; add `bmad-loop-setup` / `bmad-loop-sweep` / `bmad-loop-resolve` (sonnet — sweep is mechanical but reads a whole ledger; resolve edits frozen specs).
- Rule 4 policy file: `overrides:` keys become `bmad-build-auto` (implement) and `bmad-qa-generate-e2e-tests`; mention `implementation_handoff` as the upstream-sanctioned enforcement path for the inline/runner case.
- Strategy note: upstream still ships no `model:` frontmatter (verified: zero hits in `src/`).

### 5.4 `parallel-epic-cycle-workflow-creation.md`

- Step 1: add `uv` + BMAD ≥ 6.11 checks; `.claude/skills` and `_bmad/` must be **tracked** in the target repo (a worktree without them cannot render `bmad-build-auto`) — halt if `.gitignore` excludes them.
- Edit 5.2 Runner-Mode Deltas: "create-story, stage spawns (dev/QA/CR)" → "plan/implement build-auto spawns, QA, CR"; add the bookkeeping-commit rule (worktree-local, on the epic branch); Status aggregation → `sprint_plan.py status` per worktree.
- Edit 5.6: lane names → layer ids; add build-auto's handoff/review subagents to the working-directory rule; the spawn-depth decision (§4.10) — insert a "runner build-auto mode: spawned | inline" key in `parallel.yaml` (`build_auto_mode: spawned`).
- SC-4-P step 4: `sprint-status.yaml` conflict resolution "re-run `/bmad-sprint-planning`" → headless `sprint_plan.py generate` (deterministic merge, never downgrades).
- Rule 12 stage names unchanged; add a note that build-auto's Matrix Test Audit runs tests during the implement stage — if the project's runtime lock covers `qa`, decide whether `implement` must also hold it (`stages: [all-runtime]` already exists for this).
- `.gitattributes` union merge: keep for `deferred-work.md`; add `_bmad-output/implementation-artifacts/sprint-status.yaml merge=union`? **No** — YAML union merges corrupt; keep the re-generate policy.

### 5.5 `README.md`

- Compatibility note → "v6.11.0+ (rendered `bmad-build-auto` pipeline); the last v6.10-compatible kits are tagged `<tag>`."
- Prerequisites: add `uv` (Python 3.11+ provisioned by uv), git ≥ 2.34.
- Kit 1 description: pipeline is "sprint planning → build-auto plan (Opus) → lead gates → build-auto implement (Sonnet) → QA → adversarial code review (Opus) → rework → smoke → commit"; customization files list.
- New "Relationship to `bmad-loop`" paragraph (§2 option B) so users don't run both over the same stories.

---

## 6. Open questions / spikes before implementation

1. **Depth-3 nesting** (orchestrator → runner → build-auto → handoff/review subagents) in this Claude Code build — decides §4.10's default.
2. **`Halt after planning.` + re-dispatch on the same spec** end-to-end, including that the plan spawn leaves the tree dirty only with the new spec (then bookkeeping commit) and the implement spawn passes step-01's clean-tree check.
3. **Rework loop via `status: in-progress` + appended tasks** (§4.6) — confirm build-auto re-implements only the appended tasks and the self-review doesn't re-open the frozen block.
4. **`sprint_plan.py generate --set`** as the lead's status writer: confirm it accepts `epic-N=in-progress` and story keys from a `--set` without regenerating anything else destructively (dry-run first).
5. **Code-review with `spec_file` = a build-auto spec**: the acceptance-auditor reads "acceptance criteria" from the spec's Tasks & Acceptance / I-O matrix — verify it doesn't expect the old story-file layout (`## Acceptance Criteria`, Dev Agent Record → File List). Rule 8/QA "File List completeness" (parallel kit Edit 5.6c) references the old Dev Agent Record section — needs a new home (spec `## Verification` or `## Design Notes`).
6. **Deferred harvest format** — confirm `bmad-loop-sweep`'s `### DW-<n>:` shape is not required by anything the kit calls (it is bmad-loop-only), so Rule 15's shape can stay.
7. Whether to keep a `pipeline: legacy` switch (option C) for projects mid-epic on 6.10 that upgrade BMAD before finishing — cheap to add now, expensive to add later.

---

## 7. Suggested sequencing

1. **Spike** items 1–4 above on a scratch project (half a day). Record results in the kit's design notes.
2. **Base kit** rewrite (§5.1, §5.2) → new Kit-Version; run it on this repo's own v6.11 install; validate with the new `resolve_customization.py` checks.
3. **Model pass** list refresh (§5.3) → new Kit-Version.
4. **Parallel kit** deltas (§5.4) → new Kit-Version; keep the anchors the base kit rewrite changes in sync (Edit 5.2/5.6 anchors move).
5. **Pilot** one real epic sequentially, then one parallel pair; feed the telemetry (`review_loop_iteration`, `followup_review_recommended`, double-review cost) back into the layer-trimming decision (§3 cost note).
6. **README** + tag the pre-6.11 kits before merging.

---

## 8. Applied 2026-08-26 — spike results and residual risk

**Spikes run (this repo, BMAD 6.11.0, uv 0.12.5):**

- **Depth-3 nesting (§6 item 1): PASS.** lead → L2 → L3 general-purpose agents all spawned and returned. `build_auto_mode: spawned` is the parallel-kit default.
- **`sprint_plan.py generate --set` (§6 item 4): PASS with two corrections.** Sets story and epic keys; `status`/`validate` emit clean JSON; the on-disk story-file floor recognises `<key>.md` but **not** `spec-<key>-*.md`, so the lead's explicit `--set` is required (as designed in §4.3). Corrections from the adversarial review: (1) `--set` is the script's *repair* path and **does downgrade** (`review → ready-for-dev` was written; the first read of the spike output was wrong) — the kit now makes the lead read-before-write; (2) `--set` on a key absent from the epic files **fails** (`--set key … is not in the generated plan`), so Story X.0 is first inserted into `epics.md`.

**Not yet spiked (need a real pipeline run on a scratch project):**

- §6 item 2 — `Halt after planning.` → bookkeeping commit → re-dispatch on the same spec passing build-auto's clean-tree check.
- §6 item 3 — rework via `status: in-progress` + appended `[Review]` tasks.
- §6 item 5 — `bmad-code-review`'s acceptance-auditor against a build-auto spec layout.
- §4.8 Story X.0 registration: whether `SPRINT_PLAN generate --set <new-key>=…` keeps a key absent from `epics.md` or drops it as an orphan (the kit text tells the lead to check and fall back to adding the story to `epics.md`).

**Ledger drain (Kit-Version 2026-08-27.1, from the `lore2` field report "The Pool Has No Drain", `bug.md`):** Rule 15 gained a fixed entry grammar (`### DW-n` heading, body written once, append-only single-line trailer; effective status/owner = last trailer line; `wontfix-accepted` + `dropped` added to the vocabulary; every entry born with an owner), Rule 17 made the drain a set of logged stages (`ledger_load` before sprint planning, `ledger_adjudicated` per story between `cr_complete` and `smoke_complete`, `ledger_burndown_complete|skipped` per epic before `epic_status_done` with an optional Story N.9 burn-down, Story X.0 as a close), and the base kit ships `_bmad/scripts/ledger.sh` so nobody reads or hand-edits the file. Deviations from the report's proposal: append-only kept (parallel `merge=union`) with last-line-wins semantics instead of rewriting owner lines; Rule 15's vocabulary reused; adjudication placed after code review (no playtest stage in this pipeline); burn-down runs as a real story through the pipeline; owners assigned at harvest; a deterministic tool added because a mature ledger cannot be read by the lead.

**Orchestration liveness (Kit-Version 2026-08-28.1, from the DragonWar field report of 2026-08-28):** `stage_spawned` write-ahead entry before every stage `Agent` call (folds `story_planning`; carries `agent_name`); resume step 4 branches on session ownership instead of asserting "died mid-run" (a fresh sequential lead owns no live stages; an orchestrator asks the runner and never diagnoses a worktree from artifacts); every stage block propagates its `🚫` git/CI prohibitions into internal subagents in the same breath as Rule 13, with verify-don't-trust; `SendMessage`-resuming a stage agent is a named anti-pattern (lead/runner/orchestrator→stage all forbidden); the orchestrator's notification handling gains an interim-status bucket (`runner_interim_status`) and re-dispatches a runner only when `ListAgents` confirms it dead; the runner contract forbids interim returns. Deviation from the report: the liveness rule keeps "died mid-run" as the correct diagnosis for a fresh sequential lead (subagents die with their session) rather than halting in every ambiguous case.

**Containment (Kit-Version 2026-08-29.1, from the second DragonWar report of 2026-08-28):** Rule 18 (returned agents are finished — universal `SendMessage` rule; return exactly once; prohibitions propagate to every depth; orphan protocol; `ListAgents` limitation); the real trigger was upstream `bmad-build-auto` step-04's "re-engage the implementation subagent with a synchronous message", which on Claude Code is an async `SendMessage` — pre-answered as "cannot be re-engaged" in the implement block, and the containment text is written into `implementation_handoff` and every review layer via `_bmad/custom/bmad-build-auto.toml` / `bmad-code-review.toml` so depth-3 prompts carry it by construction; parallel kit: non-runner notification bucket, `TOOLING:` `escalated_to_user` outcome, structural `epics.md` diff instead of a raw hash, orchestrator write discipline (absolute paths + cwd assertion); model-tier checkpoint evaluates on every project with a pre-flight `telemetry_gate` for unapplied recommendations.

**Design changes made while applying (beyond §5):** Smart Parallelism is retired entirely — build-auto's clean-tree check runs after it writes the spec/epic-context, so even plan spawns cannot share a checkout (§4.2); the lead pre-warms and commits `epic-<N>-context.md` (new "Epic Context Pre-warm" section) because build-auto would otherwise halt on its own untracked cache file; a re-dispatch protocol (reset spec `status` to `draft`/`in-progress`, re-spawn on the spec path) replaces naive re-spawns after `blocked`; the lead mirrors `baseline_revision` into `baseline_commit` because `bmad-code-review` reads the latter; the pre-flight requires `_bmad-output/implementation-artifacts` to be tracked; `bmad-sprint-planning` moved haiku → sonnet in the model pass (it absorbed the readiness gate); `model-overrides.yaml` `overrides:` keys are stages, not skills, because `bmad-build-auto` runs at two tiers; a new Rule 16 (clean tree before dispatch) carries the bookkeeping-commit doctrine.
