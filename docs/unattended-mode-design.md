# `/epic-cycle --unattended` and `--review`: design

Status: **revised after an adversarial review (§19), re-approved by the owner on 2026-10-18, and implemented in Kit-Version 2026-10-19.1.**
- The owner approved the first version on 2026-10-18.
- The review found it not implementable as written. The owner approved four of the revisions the same day (§19), then the revised design as a whole.
- **Implemented in Kit-Version 2026-10-19.1** (base and parallel), after the kit fixes it depends on (§16) shipped in 2026-10-18.1. The doctrine is `_bmad/custom/epic-cycle-unattended.md` (sections U1–U14). Not yet field-tested.
- The field-trial project is undecided; the owner is first watching how OcuTalon does with its hand-written workaround.

Sources:
- the owner's requirements of 2026-10-18;
- the field-tested OcuPilot practice (`contests/contest-49/ocupilot-unattended-rules.md`, including its ten hindsight items);
- OcuTalon's distilled "Rule 22" prompt;
- an inventory of every stop point in the base command and the parallel kit (§6).

## 1. What the owner asked for

- **The development loop runs unattended.** It makes decisions to the best of its ability, logs them, and keeps running.
- **Setup may still stop.** Pre-flight, branches and workspace, and the up-front planning gates keep their attended stops.
- **The design is generic.** It assumes no planner, liaison, relay or any other designated agent; one session must be enough. Nothing assumes a stack or project either.

Decisions taken on 2026-10-18:
- AD conflicts are handled **comply first** (§8).
- The retrospective runs **headless** in unattended mode (§9).
- Stages **decide small ambiguities themselves**, and escalate to the lead freely (§7).
- **`--review`** is in scope (§13).
- After the review: holds wait for the current stage to finish, with no time limit (§11); the inbox is written only through its script and checked around every stage (§12); a sequential run whose epic is blocked fills the gap inside that epic, then ends (§10); the headless retrospective's `rejected` verdict holds the merge, and its action items go forward (§9).

## 2. Scope boundary

| Phase | Covers | Under `--unattended` |
|---|---|---|
| **Setup** (attended) | **Pre-flight:** lead model gate, BMAD/runtime/v7 gates, launch environment, telemetry gate, the launch checks in §3. **Branches and workspace:** SC-1/SC-2 for the first epic, root freshness, resume-mode and integrity checks, IDE-toggle crash recovery. **Up-front planning:** parallel dependency-graph approval, `spine_resolved`, the first epic's `ledger_load`, retro review and Story X.0, and the sprint-planning readiness gate for **every** epic in the range | **Unchanged:** stops and asks |
| **Development loop** (unattended) | Everything after the first `stage_spawned`: the per-story pipeline, CI, the epic-close gates (burn-down, model tier, retrospective, decision sheet, merge), **and every later epic's start** (SC-2, `ledger_load`, retro review, Story X.0, the readiness re-check). Parallel: dispatching, notification handling, the merge queue | **Never waits for input.** Every stop point resolves to one of the outcomes in §5 |

**Later epics' starts are part of the loop.** Story X.0 for epic N needs epic N−1's retrospective and ledger, so it cannot run at launch:
- **Retro review** is already lead-automated triage.
- **X.0's step 5** follows the classification in §5, with three rules:
  - owner-only `decision-pending` entries carry forward unchanged (§9);
  - other undecided entries are ruled;
  - open action items are included as today, including the headless retrospective's.
- **`ledger_load`:** an owner it cannot repair is re-owned to `burndown` with `note=owner_unresolved`, and logged as a `decide` ruling.
- **The readiness re-check:**
  - PASS continues;
  - CONCERNS continues, with a veto item;
  - FAIL, or an integrity error, stops that epic (§5).

An unattended run therefore never stops at an epic boundary to wait.

## 3. Activation and launch

- **Invocation grammar.** `/epic-cycle <range> --unattended` runs a range unattended. `/epic-cycle --review` takes no range, and reads the run state (§12). `--review` with a range is an argument error.
- **Logging.** The flag is logged once as `unattended_started run=<id> policy=<sha256 of unattended.yaml> doctrine=<sha256 of epic-cycle-unattended.md> range=<range>`.
  - `run=<id>` stays the same across holds and resumes. "This run" (not "this session") is the unit of provenance.
  - The flag and both hashes are carried into every runner spawn prompt and every resume prompt.
- **Precedence.** One sentence near the top of the command, inside what survives compaction:

  > "Under `--unattended`, `_bmad/custom/epic-cycle-unattended.md` overrides every passage it names. Re-read that file, and this command, after any compaction."

  Every passage it overrides carries a short "Unattended:" pointer (§15).
- **Launch checks**, added to the attended pre-flight, so a failure stops while the owner is still there:
  - `_bmad/custom/unattended.yaml` exists and parses;
  - **permission mode:** the owner confirms that the session runs in `bypassPermissions` or `auto` mode, logged as `permission_mode_attested` (the agent cannot reliably detect it);
  - **session type:** an interactive session (terminal, IDE or desktop) is the supported type. A `claude -p` launch needs `CLAUDE_CODE_PRINT_BG_WAIT_CEILING_MS=0` for the parallel kit;
  - **launch environment:** spawn depth, `CLAUDE_CODE_RETRY_WATCHDOG=1`, and the Bash cap if the project needs it;
  - **CI mode.** If CI is `none` or `unavailable`, the owner chooses one of:
    - `ci.without_ci: local_green` (merge on the full suite plus the smoke);
    - `hold` (merges wait; a sequential run then ends at the first epic close).

    A run never discovers this mid-loop.
  - **repository visibility:** read with `gh repo view --json visibility`, or asked, and logged. It informs the security hard stop (§4);
  - **push notifications:** check whether a push-notification tool is available. If not, log `notify_unavailable` and continue with the inbox only;
  - **run lock:** no live run already holds the run state (§12);
  - **parallel runs only:** the dependency graph is approved and its hash is current.
- **Re-entry:** re-invoking the same command resumes, through the unattended resume semantics in §11. A crashed session is relaunched by the owner or by external tooling; the kit does not supervise its own process.

## 4. The policy file: `_bmad/custom/unattended.yaml`

Project-owned. The kit writes it from a generic template only when it is absent, and never overwrites it. The categories are **data, not prose**, because OcuPilot's top regret was a grant scattered across chat, logs and a stale rule. The template, with its generic defaults:

```yaml
version: 2
product_defaults:      # tie-break principles for calls made alone, most important first (the project edits these)
  - The architecture spine is the contract; where it is silent, choose the simplest option consistent with it.
  - When a choice trades safety for convenience, keep the safe behaviour and make it convenient.
  - Measure, don't guess; anything not yet measured is labelled (inference).
  - Smaller is better; split rather than stretch a story.
  - Text claims only what the product does.
owner_fixed: []        # values the owner chose; quoted verbatim in every stage, runner and resume prompt; never re-derived
stage_preamble: ""     # the project line that must head every stage prompt (e.g. "one test-runner call per message")
milestones: []         # [{date: 2026-11-30, scope: [1, 2, 3, 4, 5, 6], name: "release 1"}] - defines "the release";
                       # the run report measures pace and risk against these. With none, every AC move is owner_only.
licences_allowed: [MIT, BSD-2-Clause, BSD-3-Clause, Apache-2.0, ISC]
hard_stops:            # checked FIRST. Stop the affected unit (story, epic or parallel slot); record; continue elsewhere
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
  - amending an architecture decision (AD), within the bounds of §8
gates:
  retrospective: headless      # headless | skip (skip for quota-tight runs)
  model_tier: record_only      # offers are recorded, never applied (tier changes are spend); the mandatory rollback still applies
  merge_to_feature: when_green # when_green | hold
  merge_conflicts: resolve     # resolve | ask. Overrides parallel.yaml integrate_conflicts for the run (§9)
  gap_fill: burndown           # burndown | none
  gap_fill_stories: []         # standing cleanup stories, per epic, that plan the gap-fill supply
limits:
  clarification_rounds: 3      # per stage per story; then the story stops
  split_children: 6            # stories one original story may become; past that the story stops
  fix_forward_per_cause: 1     # per first-red sha; then the epic stops
  veto_open_max: 20            # open veto items; at the limit, merges to the feature branch are held until --review
  unverified_stories: 1        # stories that may be committed past an unresolved CI result during an infrastructure outage
ci: {queue_minutes: 60, blocked_hours: 6, without_ci: ""}   # without_ci is set at launch when CI is none or unavailable
hold: {at: "", after_hours: 0} # an owner-set hold: a clock time (ISO 8601) or hours of run time; quota itself is invisible
notify: {push: true}
```

## 5. Outcomes, and how a decision is classified

| Outcome | Meaning | Record |
|---|---|---|
| **decide** | Rule using `product_defaults`, the spine and the rules, then continue | `ruling … category=decide`; log and run report only |
| **veto** | Decide now; the owner may reverse before the next release | `ruling … category=veto reversal=<what undoing it takes>`; inbox veto item |
| **park** | Only on an explicit `owner_only` match: record the question with options and a recommendation; the unit waits; independent work continues (§10) | `story_parked`; inbox item; push notification |
| **stop** | The unit cannot proceed, but nothing owner-only is at stake: stop the affected story, epic or slot; record; continue elsewhere, or end the run with its report | `unattended_stop unit=<…> reason=<…>`; inbox item; push notification |
| **hold** | Finish the current stage, record everything and end cleanly (§11) | `hold_ordered`, `hold_complete`; push notification |

**Classification order:** `hard_stops` first, then `owner_only`, then `veto_window`. Anything else is decide.
- **Park only on an explicit `owner_only` match.** Ambiguity between decide and veto resolves to **veto**, never to park.
- Every other blocker is a **stop** of the smallest unit that contains it: the rework cap, a readiness FAIL, a re-architecture, an unmeetable tooling need. A stopped unit is listed in the inbox with options, and `--review` can restart it.
- A `ruling` line always names the question, the choice, the rejected options and why, the category, and the policy hash.
- **`decide` outcomes stay out of the inbox:** they go to the log and the run report only.

**Setting a story aside.** A story that parks or stops **after its implement stage has produced code** is set aside, so that its code never sits under later work:
1. Save its source changes since `story_base`, including any uncommitted QA tests and review patches, as `<impl>/set-aside/<story>.patch`. `story_base` is the commit before its first implement spawn, logged on that spawn.
2. Restore the story's source paths to `story_base` in one commit, `chore(epic-N): set aside <story> (<inbox id or reason>)`. This is a forward commit, never a reset, so it is safe whether or not the story's commits were pushed. It is Rule 16's second sanctioned source-file bookkeeping commit.
3. Set the spec and the tracker back to `ready-for-dev`. Log `story_set_aside patch=<path> sha=<commit>`.
4. If the story had been pushed, its set-aside head goes through CI like any head. Green continues; red stops the epic.

*Changed 2026-10-19 (owner decision): a story that changed a submodule is not set aside; its epic stops instead. A set-aside also needs every later story's code to be absent from its range, and the patch to have been saved and checked before anything is restored (doctrine U4).*

**Picking a set-aside story up again:**
1. Apply the patch three-way on the current head, and set the spec's review baseline to the commit before the apply, so review sees the whole story.
2. Continue at the stage the answer calls for.
3. If the patch does not apply cleanly, re-plan the story with the patch kept for reference.

## 6. The loop's stop points, and their unattended behaviour

Line references are to the base command (`B`) and the parallel kit (`P`) at Kit-Version 2026-10-14.1. Each row becomes an "Unattended:" pointer at that passage.

**Stage results**

| Stop point (today) | Under `--unattended` |
|---|---|
| A stage returns `## Clarification Needed`, or a build-auto clarification HALT (B 1104–1108, 1116) | Classify, then the lead rules (written into the spec; the stage re-spawned through the re-dispatch protocol) or parks. At most `limits.clarification_rounds` rounds per stage per story; then stop the story. Pipeline or environment defects (dirty tree, no subagents, a failed pre-warm) are still fixed, never "answered" |
| Build-auto non-convergence: `review repair loop exceeded`, `implementation verification failed`, `patch verification failed`, `matrix test audit failed`, `handoff conflicts with spec` (B 1119) | As the rework cap |
| Rework cap reached (B 1224, 1284) | Stop the story and set it aside. No cap override unattended. CI and smoke reworks count toward the cap, as today |
| A HIGH the stage asks to waive (B 246) | Park (owner-only) and set the story aside |
| Rule 5 tier-2 amendment (B 182) | A split into siblings planned next in the same epic, within `limits.split_children`, is decide. Moving an AC to a not-yet-planned story in the defined release is veto. Dropping or deferring out of the release, product numbers, epic boundaries and dependency statements park |
| Decision dependency at spec validation | The undecided item is ruled if its category allows; otherwise the story parks |
| Missing Integration AC at spec validation (B 1727–1729) | Option (a), re-plan, once. Still missing: option (b) only if a later story in this epic is named as its consumer; otherwise stop the story. Never option (c) |
| A destructive check with nowhere disposable to run it (B 1737) | Stop the story (a tooling need); inbox item |
| Epic-context compilation fails (B 1403) | Retry once; then stop the epic |
| AD conflict | §8 |
| Smoke, fresh-check or AD-tooled verification failure (B 1327, 1782) | The rework path (option b) |
| Orphan writer or protocol violation (Rule 18) | Apply the orphan protocol and log `protocol_violation`. A live writer that can't be stopped stops that slot or epic |
| Stage API error: rate limit or overload | The existing backoff, plus the retry watchdog |
| Stage API error: usage or spend limit (B 932) | Hold (§11) if the lead can still act; otherwise the session ends, and the resume prompt from the last story boundary carries the run |
| Model-tier checkpoint (B 904, 1243) | Evaluate and log; offers recorded, never applied. The mandatory mixed-tier rollback (B 906) still applies |

**CI**

| Stop point (today) | Under `--unattended` |
|---|---|
| Red CI on the story just committed (B 1347) | The rework path. Unfixable within the cap: stop the story and set it aside |
| Red matching a ledgered flake (B 1345) | One rerun and an `occurrence=` bump. Green continues; red takes the rework path |
| Three infrastructure reds in a row; CI queued past `queue_minutes` (B 1345) | Timed retries. Resolution point 1 is relaxed for at most `limits.unverified_stories` story: that story's successor may be committed locally, unpushed. When CI returns, push the stacked heads one at a time, oldest first, resolving each before the next push, so a red is charged to the right story. Blocked past `blocked_hours`: stop the epic, with an inbox item |
| Inherited red CI (B 1347) | Rerun once. Then one fix-forward story per root cause (keyed by the first red sha), if the failing code lies in this epic's own changes (`git diff --name-only <feature>...HEAD`); it is a veto item. Otherwise, or after that one, stop the epic |
| CI unavailable at the merge gate (B 1544) | As chosen at launch (`ci.without_ci`): merge on local green, or hold the merge |

**Epic start and close**

| Stop point (today) | Under `--unattended` |
|---|---|
| `ledger_load` cannot repair an owner (B 1357) | Re-own to `burndown`, `note=owner_unresolved`; a decide ruling |
| Readiness gate at a later epic's start (B 1170, 1700) | §2 |
| Retro review and Story X.0 at a later epic's start (B 1174, 1416) | §2 and §9 |
| Burn-down gate | Unchanged; it needs no human |
| Retrospective question (B 1245, 1710) | Headless (§9) |
| Decision sheet (B 1248, 1546) | §9 |
| Final-epic ledger sweep (B 1395) | Classified like the decision sheet; the residue is listed in the run report |
| Merge to the feature branch (B 1250, 1544–1557) | §9 |
| Merge conflicts (B 1602) | §9: integrate forward first; conflicts per `gates.merge_conflicts` |
| A red feature head after a merge | Classify as in B 1345: flake or infrastructure is retried. A story red means no new epic starts: sequential, the run ends with its report; parallel, no dispatch, though running runners finish. Inbox item |
| `veto_open_max` reached | Merges are held until `--review`; work inside open epics continues |

**Workspace and resume**

| Stop point (today) | Under `--unattended` |
|---|---|
| SC-7: unsure where to commit (B 1580) | Stop the run (workspace integrity) |
| Resume: an open clarification, a dead stage's dirty tree, divergence (B 1641–1649) | §11, unattended resume |
| Integrity checks 1–3 fail; ambiguous liveness (B 1642, 1647–1649) | Stop that epic (integrity error) |
| Agent tool permission prompt, or `AskUserQuestion` | Must not occur. An unattended lead never asks; anything that would ask is one of the outcomes in §5 |

**Parallel kit**

| Stop point (today) | Under `--unattended` |
|---|---|
| Runner `## Clarification Needed` (P 480) | The orchestrator classifies. It rules and resumes the runner with the ruling, or parks and resumes it with "parked; continue with independent work" |
| Runner `TOOLING:` need (P 481) | The orchestrator attempts the verification itself, as today. Unmeetable: stop that slot's story; the runner continues if it can |
| Runner liveness unknown (P 484) | Stop that slot |
| Red feature head blocks dispatch (P 454) | As the red feature head above |
| Bootstrap fails at provisioning (P 457) | Stop that epic; dispatch the next eligible one |
| Dependency-graph change needs re-approval (P 441) | A split inside one epic re-records the hash (`reason=split`). Any other structural change pauses only the affected epics; inbox item; `--review` re-approves |
| Merge queue question (P 490–499) | `when_green` (§9) |
| An AD whose Rule both sides amended, or a `lint_spine` finding, at SC-4-P (P 497) | Hold that merge; inbox item. §8's parallel rule prevents the double amendment |
| Retrospective gate after the merge (P 500) | Headless, run by the runner before its contract (§14) |
| Resume finds a half-done conflicted merge (P 522) | Hold that merge; inbox item |

## 7. Stages under unattended

When the flag is set, every stage prompt gains a short **Unattended** block rendered from the policy: the product defaults, the owner-fixed values, and the category lists. Rule 4 gains one clause: under an unattended run, the stage prompt's Unattended block defines which preference choices the stage decides itself; everything else still halts. The stage rule is:

- **Decide small ambiguities yourself.** "Small" means:
  - within the spec's stated intent;
  - reversible inside the story;
  - touching no `veto_window` or `owner_only` category;
  - contradicting no AD or owner-fixed value.

  Record each choice: plan, in the spec's `## Design Notes` under "Decided in-stage"; implement, in the spec; QA and review, in `## Decisions`. The reviewer sees them.
- **Escalate anything else with `## Clarification Needed`, and escalate freely.** When in doubt, escalate. The lead classifies and rules or parks; a stage never parks or asks the owner itself.
- **Never touch the run state directory** (§12) or run the inbox script. This prohibition is propagated to every subagent, like the commit prohibition.

## 8. AD conflicts: comply first

1. **Comply.** Reshape the story's approach to meet the existing AD.
   - If an AC can't be met within it, move that AC to a not-yet-planned story in the same release (a veto item).
   - Never move one that a release gate, a milestone or an end-to-end acceptance test depends on; that case goes to step 2 or 3.
2. **Amend, with a veto window, if compliance is impossible and the amendment is not a re-architecture.**
   - Use Rule 20's light path: the Rule text, an `[AMENDED <date>, <story>, unattended ruling]` marker, `memlog.py append`, `lint_spine.py` (which must return `ok: true`), a bookkeeping commit, and the pre-warm re-run.
   - Log `spine_updated reason=unattended_ruling`; inbox veto item.
3. **Otherwise stop the story** (§5), or park it if the amendment touches an `owner_only` category. `--review` offers `bmad-architecture`'s interactive Update.

**A re-architecture** is what Rule 20 says it is ("a decision that contradicts an existing AD"). It is made testable as any amendment that does one of these:
- (a) changes the AD's Binds or Prevents;
- (b) removes or reverses something its Rule requires or forbids, rather than extending the Rule to a case it does not address;
- (c) touches an AD that code already merged to the feature branch, or another in-flight epic's spec, relies on: its id appears in a `done` story's spec, or in another open epic's specs.

An amendment that only extends or tightens the Rule, on an AD nothing merged relies on, is step 2.

**Parallel runs:** a runner never amends an AD on its own. The question goes to the orchestrator.
- The orchestrator allows an amendment only when no other in-flight epic's specs cite the AD.
- If allowed, the runner writes the amendment in its worktree (the kit's spine-write path), with `orchestrator` in the marker. Otherwise the story waits, set aside, until the epics that cite the AD have merged.
- This keeps two runners from amending one AD.

## 9. Epic close

- **Retrospective:** `/bmad-retrospective -H <epic>`, the skill's stable orchestrator interface. What headless mode actually does:
  - it writes the retrospective document (`headless: true`, with an Assumptions section and an acceptance verdict in the frontmatter);
  - marks the retro key `done` (which means "the retrospective ran", not "the epic passed");
  - appends its proposed action items as `open`;
  - does **not** change the status of earlier action items (`--set-action-status` needs the user).

  The lead reads the verdict from the document's frontmatter, since the sprint status does not carry it:
  - `rejected` holds the merge, with an inbox item;
  - `accepted` and `accepted-with-open-items` continue.

  New action items go into the next X.0 as today, and are listed in the inbox under "Retrospectives to review" so the owner can drop them. Log `epic_retro_complete mode=headless verdict=<…>`. Sequential: the retrospective already runs before the merge. Parallel: §14.
- **Decision sheet:** each `escalated` / `decision-pending` entry is classified (§5):
  - Owner-only entries stay `decision-pending`, with a trailer `note=inbox=<id>`, under the owner token the kit's deferred-decision path uses (kit fix F02).
  - Veto entries are decided with a veto item; the rest are decided `by=unattended`.
  - The decision sheet, the burn-down gate and X.0 skip entries whose latest note is `inbox=<id>`. They close only when the owner answers.
  - Human-only entries (`note=human=`) move to `action_items` as today.
- **Merge to the feature branch, `when_green`.** All of these must hold:
  - every story `done`, none parked or set aside;
  - the burn-down gate logged;
  - a retrospective verdict that is not `rejected` (or the retrospective skipped by policy);
  - the decision sheet recorded;
  - `veto_open_max` not reached.

  Then:
  1. **Integrate forward first** if the feature branch has moved since the epic branched or last integrated. Merge `origin/<feature>` into the epic branch and resolve conflicts per `gates.merge_conflicts`:
     - plain unions are decided;
     - behaviour choices are ruled as veto items;
     - code an AD governs goes through §8;
     - an `owner_only` match parks;
     - an unclassifiable conflict holds the merge.

     Then run the full suite and the smoke on the merged tree, push, and require CI green on that exact head.
  2. **Merge** the epic branch into the feature branch. It is now conflict-free. Push, verify on origin, and record the post-merge bookkeeping (kit fix F03).
  3. **Resolve CI for the new feature head.** A story red is the red-feature row in §6.

  If any condition fails, the epic is held unmerged, with an inbox item, and the run continues with independent work.
- **Never** the trunk, a release, or anything outward-facing, whatever an answer says. Such answers are recorded for an attended session.

## 10. Park, stop and continue

- **A parked or stopped story** is logged (`story_parked` / `unattended_stop`), gets an inbox item, and is set aside (§5) if it has code. It gets no ledger entry: the log, the inbox and the tracker carry it, so the drain machinery never sees it.
- **The lead takes the next independent story** in epic order. A story is *not* independent if any of these say it depends on a parked or stopped story:
  - `epics.md` dependency statements;
  - the `Consumes:` lists;
  - the parked story's id appearing in its acceptance criteria;
  - a split sibling of the parked story (split children depend on their parked siblings by default, because `Consumes:` lists are often not updated at split time);
  - its own plan stage, which is told which stories are parked and escalates if it needs one.
- **If no story in the epic is independent:**
  1. With `gap_fill: burndown`, take a story named for this epic in `gap_fill_stories`, or charter a gap-fill cleanup story from closable ledger entries owned by this epic (Rule 17's chartering rules).
     - A gap-fill story is numbered like any inserted story.
     - It is titled "Epic N gap-fill", and is never reused as the burn-down story.
  2. Failing that:
     - **sequential:** the run ends as `stopped: remaining work blocked`, with its report;
     - **parallel:** the runner ends and frees its slot (§14).

     A sequential run never moves to another epic while this one is blocked: one checkout cannot carry two unmerged epic branches safely. Moving across epics is a parallel-kit feature.
  3. A run never sits waiting.
- **Picking up answers:** at every story boundary the lead runs the inbox script's `answers` (§12).
  - Each new answer is logged `owner_decision via=inbox by=<who>`.
  - It is applied through the re-dispatch protocol (a set-aside story through §5's pick-up steps).
  - It is marked resolved, and the story returns to the queue.

## 11. Hold and resume

**Triggers:**
- a `HOLD` request (`unattended-inbox.sh hold`, or a `HOLD` file in the run state directory);
- the first usage warning the session can see;
- a usage- or spend-limit error the lead can still act on;
- the owner-set `hold.at` or `hold.after_hours`.

Quota itself is mostly invisible to the agent, hence the owner-set times.

**Steps.** A hold waits for the running stage to finish, however long it takes; it never interrupts it:
1. Start no new stage.
2. When the running stage returns, do its normal bookkeeping.
   - Work the stage leaves uncommitted by design (QA's tests, review patches) is saved as `<impl>/hold/<story>-<stage>.patch` and the tree restored to HEAD.
   - On resume, the patch is applied and the pipeline continues at the next stage. This is the re-dispatch protocol's patch path, so the review baseline is unchanged.
   - **There are no WIP commits.**
3. Push bookkeeping under the normal push rules. Leave in-flight CI running.
4. Refresh the resume prompts, and record which throwaway runtimes are up, with their run provenance.
5. Verify every sha on origin. A sha that exists only locally is listed as local-only.
6. Log `hold_complete` with every sha; inbox entry; push notification; release the run lock; end.

**The resume prompt is kept current, not written at the end.** At every story boundary the lead (or runner) rewrites `<state>/resume/<epic or slot>.md` from the log. A session that dies without warning, such as a hard usage limit or a crash, therefore leaves a current resume prompt and needs no hold steps. It contains:
- the role and invocation, with `--unattended`, the run id and both hashes;
- the owner-fixed values;
- the git-verified state: branch, origin and local heads, each unpushed commit and what it holds, spec status, the ledger, parked and set-aside stories;
- every runtime target in use (dev and throwaway): what is loaded or deployed there, and which run started it;
- the project's `stage_preamble`;
- "First, check the CI result for the last pushed `headSha`";
- the next steps, in order;
- how to end.

**Unattended resume semantics,** overriding B 1640–1649 and 1673 under the flag:
- The resume point is the earliest story whose anchor is not `committed`, **skipping parked, stopped and set-aside stories.**
- An open clarification is classified and ruled or parked, not surfaced.
- **A dead stage's dirty tree** (a fresh lead, no orchestrator) is saved as `<impl>/resume/<story>-<stage>.patch` as evidence, the tree restored to HEAD, and the stage re-run from its spec status.
- Divergence, branch drift, a diverged head and ambiguous liveness stop that epic.
- Idempotent steps (sprint planning, `spine_resolved`) re-run. The attended setup gates this run already logged are not asked again.
- **The log beats the resume prompt.** The prompt is a convenience; where the two disagree, the log wins.
- A resume re-logs `unattended_resumed run=<id> policy=<hash> doctrine=<hash>`, which is how an owner edit to the policy during a hold is accepted.
- **A resume is always a fresh session or a fresh spawn,** above all after an account switch. Never message a runner whose call has returned, except through the parallel kit's sanctioned resume.

## 12. Run state, the inbox, and notifications

**The run state directory** `<state>` is outside git's view, so nothing written there dirties a tree:
- sequential: `_bmad-output/unattended/`, gitignored by the kit;
- parallel: `<worktree_base>/.coordination/unattended/`.

It holds `inbox.md`, `inbox.journal`, `run.lock`, `HOLD` and `resume/`. A snapshot of the inbox is committed to `<impl>/unattended-inbox-snapshot.md` in each merge's bookkeeping commit and at run end, so the record also lives in git.

**Events:**
- `unattended_started`, `unattended_resumed`, `ruling`, `story_parked`, `story_set_aside`, `owner_decision`, `unattended_stop`, `hold_ordered`, `hold_complete`, `unattended_report`.
- Ledger trailers carry `by=unattended`. Spine and `epics.md` amendments carry `[AMENDED <date>, <story>, unattended ruling]`. Commit subjects name the ruling.
- Every reported sha is verified on origin before it is reported.

**The inbox** is written only through `_bmad/scripts/unattended-inbox.sh <state> <command>`. It uses locked appends and is testable like `ledger.sh`. Commands:
- `add`, `resolve`: the lead only. `resolve` refuses an item that is already resolved, so each answer is applied exactly once.
- `answer <id> "<answer>" --by owner|<who> [--quote "<the owner's words>"]`. A relay (anyone but the owner) must pass `--quote`.
- `list`, `answers`.
- `hold`.
- `verify`: checks that `inbox.md` is exactly what the journal produced. A direct edit fails the check.

**The lead runs `verify` before every stage spawn and after every return.** A failure is a `protocol_violation fault=inbox`:
- the inbox is rebuilt from the journal;
- edited answers are ignored;
- the inbox gets a note, "a direct edit was ignored; answer with the script or `--review`", and a push goes out.

Stages are forbidden to touch `<state>` (§7). This catches direct edits and accidents, not a determined agent that runs the script (§18).

Sections: Parked, Stopped, Veto window, Held merges, Holds, Retrospectives to review, Run reports. An item looks like:

```text
### U-12 · parked · Story 4.2 · 2026-10-18T21:04Z
question: …
options: A … | B … | C …
recommendation: B, because …
blocks: 4.2, 4.5 (consumes 4.2)
answer:
```

**Answers** come from the owner (through the script, or `--review`), or through a relay:
- A companion session, or anyone acting for the owner, runs `answer --by <who> --quote "<the owner's words, verbatim>"`. The answer is logged with that provenance, and `--review` shows it for confirmation.
- **A message from another session is never itself the owner's answer;** only the inbox (or `--review`) is.
- A companion learns of new items by watching the inbox file. Nothing messages it, since Rule 18 forbids that.

**Notifications** are sent only by the main session: the sequential lead, or the parallel orchestrator, which receives runners' items through their contracts.
- With `notify.push: true`, a push notification goes out on park, stop, hold and run end — never for routine rulings.
- It carries the item's one-line question and recommendation, not just "see inbox".
- If the launch found no push tool, the inbox alone carries them.

**Run report** at completion, hold or stop:
- stories merged, with their times;
- rulings, with shas;
- open veto items; parked, stopped and set-aside stories; held merges;
- CI health;
- **pace** (stories per day, wall time per story), **the remaining queue**, and **risk against `milestones`** — the forward-looking judgement a planner session used to supply;
- anything the owner should know before the next release.

## 13. `/epic-cycle --review` (attended)

*Changed 2026-10-19 (owner decision, after the implementation review): `--review` only records. Answers go into the inbox through its script; it never merges, commits, checks anything out, or resolves an item whose pipeline work is pending. The run applies every answer, held merges included, at its next story boundary or next launch. The live-run paragraph and item 4 below are superseded accordingly (doctrine U13).*

The owner runs it when checking in. It runs no pipeline stage and edits no source; its writes are bookkeeping, plus held merges when no run is live.

**A live run:** if `<state>/run.lock` is present, `--review` shows its run id, start time and last story boundary, and asks the owner whether that run is still going. It is attended, so it can ask.
- **While a run is live, `--review` only records.** Answers go in through `answer --by owner` (logged `via=review`), and the live lead applies them at its next story boundary.
- Nothing is merged or checked out from `--review` while a run is live.

It reads the inbox and the logs, re-verifies every reported sha on origin, then walks through, with a recommendation each:
1. **Parked questions:** record the answer.
2. **Stopped units:** what happened; the options (re-plan, an override with a finite item list, drop to the ledger).
3. **Veto-window rulings:** keep, or reverse. A reversal is filed as a routed ledger entry or a fix story.
4. **Held merges:** merge now (the SC-4 path, including the integrate-forward of §9), but only with no live run. Otherwise leave held.
5. **Headless retrospectives:** the verdict, the findings, the new action items (keep or drop), and the proposed status changes to earlier items (confirm or not).
6. **Amended ADs:** optionally run `bmad-architecture`'s interactive Update.
7. **Dependency-graph changes** awaiting re-approval (parallel).
8. **Relayed answers:** confirm each.
9. **A summary.**

Pipeline work for the answers (un-parking, re-planning, rework) runs in the live run, or in the next `--unattended` or attended run. `--review` never runs it itself.

## 14. Parallel runs

- **The orchestrator passes** `--unattended`, the run id and both hashes in every runner spawn and resume prompt.
- **A runner is the lead for its epic,** and rules within it. Cross-epic matters go to the orchestrator as `## Clarification Needed`: contended paths, other epics' stories, AD amendments (§8).
  - The orchestrator rules, or parks, and resumes the runner through the sanctioned orchestrator-to-runner message.
  - An edit to a contended path waits, with the story set aside, until the epic that owns the path has merged.
- **Orchestrator notification handling:**
  - setup stops are surfaced to the owner, as today, while other slots run;
  - loop items are classified as in §5;
  - `decisions_for_user:` items are classified as at the decision sheet (§9);
  - on **every** notification, the orchestrator also reads the inbox answers and re-dispatches any parked epic whose answer has arrived.
- **Retrospective:** the runner runs it headless as the last step before its completion contract, inside the worktree. It holds the runtime lock (Rule 12) if the retrospective exercises the runtime. A `rejected` verdict is reported in the contract, and the orchestrator holds the merge.
- **Merge queue (SC-4-P):** `when_green` replaces `queue-ask` while unattended.
  - If the feature branch moved after the runner integrated forward, the orchestrator resumes the runner for one more integrate-forward (§9), rather than resolving anything itself.
  - Dispatch continues along the approved graph. A structural change that needs re-approval pauses only the affected epics; a split inside one epic re-records the graph hash with `reason=split`.
- **Idle slots:** a runner that ends because its epic is fully parked, stopped or held frees its slot, and the orchestrator dispatches the next eligible epic into it. When no runner remains and nothing is dispatchable, the run ends with its report; it never idles.
- **Hold:** the orchestrator writes `HOLD` in the run state.
  - Each runner checks for it at every stage boundary, finishes its bookkeeping, refreshes its slot's resume prompt, and ends with a `## Hold` contract.
  - The orchestrator waits for every runner's `## Hold`, however long the current stages take. It never stops a live runner or writes in its worktree.
  - It then verifies, logs `hold_complete`, notifies and ends.
- **Stale runtime locks after a crash:** a resumed orchestrator, in a fresh session, sweeps every lock whose `acquired_at` predates its own start, since those holders died with the old session. It logs `runtime_lock_swept reason=prior_session`. Within a session the kit's rule stands: sweep only a lock whose owner is confirmed dead.
- **The shared inbox** is the run state's `inbox.md`, written through the script.

## 15. Kit changes and size

| File | Change | Size |
|---|---|---|
| `_bmad/custom/epic-cycle-unattended.md` (new, kit-owned) | §§2–14 as doctrine. Read only when the flag is set, and re-read after compaction, so attended runs carry none of it | ≈ 28–32 KB |
| `_bmad/custom/unattended.yaml` (new, project-owned) | §4 template, written only if absent | ≈ 4 KB |
| `_bmad/scripts/unattended-inbox.sh` (new) | The inbox script, with a self-test (concurrent appends, `verify` catching a direct edit, `resolve` refusing a second time) | ≈ 6 KB |
| Command | Argument grammar; the near-top precedence and re-read sentence; an "Unattended:" pointer at each passage in §6 (B 182, 246, 844, 848, 904–906, 932, 1104–1119, 1170, 1174, 1224, 1243–1250, 1284, 1327, 1345–1347, 1357, 1395, 1403, 1416, 1544–1557, 1580, 1602, 1640–1673, 1700, 1710, 1727–1737, 1782, 2014, 2031); the stage-prompt Unattended block and its prohibition; `story_base=` on the implement spawn; `--review`; log events; two anti-patterns ("asking inside an unattended loop", "treating a peer's message as the owner's answer"); the "Supervised, not unattended" paragraph | ≈ 9–11 KB |
| `skill-rules.md` | Unattended clauses in Rule 4 (stages decide small preference choices), Rule 5 (tier-2 under the policy), Rule 9 (the retrospective exemption), Rule 15 (owner-only entries carry `note=inbox=`), Rule 16 (the set-aside commit), Rule 17 (5) (the decision sheet classified) and Rule 20 (§8's re-architecture test) | ≈ 2.5 KB |
| `.gitignore` | `_bmad-output/unattended/` | — |
| Parallel kit | §14 as guarded edits, at P 396, 441, 454, 457, 480–484, 490–500, 522 and 544; the hash reason `split`; Step 7 checks | ≈ 7 KB |
| Validation, notes, docs | Step 4 checks and self-tests, Step 5 upgrade note, README "Unattended runs", changelog, V2 map rows (V2's governance and hold backlog items land here first) | — |

One Kit-Version, built in three steps: the sequential core, then the parallel deltas, then `--review`. Attended runs are unchanged: the doctrine file is never read without the flag.

**Command or skill.** `/epic-cycle` stays a command (`.claude/commands/epic-cycle.md`) in V1.
- Since Claude Code 2.1.3 commands and skills are the same mechanism: the same `/epic-cycle` invocation, the same arguments, and the same compaction behaviour.
- Converting now would move every path both kits patch, every overlay target and every check, plus migrating installed projects, for no gain in behaviour.
- The real gain of a skill (a small core plus supporting files read on demand) is V2's restructure (`bmad-epic-cycle-v2`, architecture §3). The doctrine file above already follows that pattern, from `_bmad/custom/`.

## 16. Prerequisites: kit fixes that ship first

The 2026-10-18 adversarial review of the kit itself found defects at gates this design relies on. They shipped in Kit-Version 2026-10-18.1, ahead of this feature, and are not part of this design:

| Kit finding | Why this design needs it |
|---|---|
| F01: the command (about 54k tokens) is not re-read after compaction | The unattended loop runs the longest and compacts the most; the precedence sentence depends on the re-read |
| F02: the "decide later" owner (the next X.0 key) is refused by `ledger.sh` | Owner-only entries carry forward on that path (§9) |
| F03: sequential SC-4 has no sanctioned post-merge commit and no write-ahead marker | The automatic merge (§9), and resuming after a crash at the merge |
| F04: pre-flight log entries dirty the tree before the first clean-tree gate | Launch checks (§3) add more pre-flight entries |
| F06: the unpushed-commit checks span every branch | Set-aside, hold and runner contracts verify what is pushed |
| F07, F08: `ledger.sh` refuses valid owners on large trackers, and non-ASCII keys | Long unattended runs produce large trackers |
| F09: no sanctioned status downgrade | Set-aside returns a story to `ready-for-dev` |
| F10: `baseline_revision` is overwritten by every rework | Set-aside and adjudication need the story's real start (`story_base`) |
| F11: the bookkeeping-commit scope excludes `epics.md` and the spine | Rulings write both in bookkeeping commits |
| F12: the patch re-dispatch cannot pass the lead's clean-tree gate | Set-aside pick-up and hold resume use that patch path |

## 17. Verification

1. The Step 4 and Step 7 checks, extended.
2. The `unattended-inbox.sh` self-test.
3. A written walkthrough of every row in §6 against the doctrine.
4. Scripted scratch-repo trials of the mechanics an agent can get wrong:
   - setting a story aside and picking it up again, before and after a push;
   - a hold while a stage runs;
   - a resume after a simulated crash with a dirty tree;
   - the integrate-forward-then-merge sequence.
5. This review and the owner's re-approval.
6. The owner decides on a field trial; a project already adopting an equivalent rule by hand is the natural candidate.

## 18. Limits and risks

- **Classification errors.** Mitigated by checking hard stops first, the veto window, `veto_open_max` and `--review`.
- **The inbox check is not authentication.** It catches direct edits and accidents. A determined agent with a shell could still run the script. Outward-facing and release actions never execute unattended whatever an answer says, and `--review` shows every relayed answer.
- **A crashed session is not restarted by the kit.** Re-invocation is safe, but needs the owner or external tooling. The README documents an optional external relaunch loop.
- **Quota is mostly invisible.** Holds trigger on warnings, errors and the owner's times. A hard usage limit ends the session at once; the resume prompt, kept current at every story boundary, carries the run.
- **A sequential run ends on the first fully blocked epic,** after gap-fill. Moving across epics needs the parallel kit.
- **An owner who rarely reviews** accumulates parked work. The run report says so, and the run ends rather than idling.
- **Headless retrospectives** are weaker than attended ones. `--review` surfaces them, and a `rejected` verdict holds the merge.
- **Auto-merging to the feature branch** can carry a ruling the owner later vetoes. Reversal is then rework, not a revert of the merge.

## 19. Review record

**First review: the Contest Planner, 2026-10-18,** drawing on OcuPilot's and OcuTalon's runs. Every finding was adopted.
- **Three changed something the owner had decided, and the owner approved all three:**
  - setup for the whole range at launch (1.1, since narrowed by U-02 below);
  - `merge_conflicts: resolve` under `--unattended` (1.2);
  - park only on an explicit `owner_only` match (3.1).
- **Refinements:**
  - 1.3 known flakes; 1.4 inherited red; 1.5 orphans; 1.6 the AC-move guard;
  - 2.1–2.5 policy template changes;
  - 3.2 split siblings; 3.3 the planned gap-fill supply;
  - 4.1–4.3 hold and resume (4.1's time box since replaced by U-05);
  - 5.1–5.4 the inbox and reports;
  - 6.1–6.3 idle slots, the hold deadline (since replaced) and stale locks;
  - 7.1 `skip`; 7.2 the relaunch loop.

**Second review: an adversarial review of this design against the kit, 2026-10-18.** It found 30 findings (5 Critical, 14 High, 10 Medium, 1 Low), and the maintainer checked every Critical and High against the kit, BMAD and harness files. Two held only in part:
- U-10: the `note=human=` items do not leak;
- U-15: CI reworks were already capped.

**The owner approved four revisions on 2026-10-18:**
1. Holds wait for the current stage, with no time limit and no WIP commits, and the resume prompt is kept current at every story boundary (U-04, U-05, U-19).
2. The inbox is kept outside the git tree, written only through its script, and checked around every stage, while keeping the relay path (U-03, U-13).
3. A sequential run with its epic blocked fills the gap inside the epic, then ends (U-16).
4. A headless `rejected` verdict holds the merge, and its action items go forward and are listed for the owner (U-18).

The owner then re-approved the revised design as a whole, and approved the kit release its §16 prerequisites shipped in (2026-10-18.1).

**Third review: an adversarial review of the implementation (Kit-Version 2026-10-19.1, before its release), 2026-10-19.** Two reviewers, one on the doctrine's logic and one on the mechanics (the inbox script on bash 3.2 with BSD awk, gawk, mawk and busybox awk; the install steps; the git procedures on git 2.34 and 2.50), found 55 defects between them (1 Critical, 13 High). Among them: a second set-aside of the same story wiped later stories' code; a set-aside committed files the story's `.gitignore` had hidden; a HOLD that no resume cleared; `--review` resolving answers before the run applied them; runners running the launch and releasing the orchestrator's lock; a blocked runner re-spawned forever; parallel answers applied twice; the runner's retrospective written after its final push. All were fixed, and a re-review by two fresh reviewers then found 43 more (4 High — among them a parallel run that could not resume from a hold, and a Step 4 check that could run `rm -rf /` on a busybox system), all fixed before release; a third round, walking whole runs through the text and attacking the mechanics again, found 38 more (5 High — among them a hold that lost a code review's work, a pre-flight check that stopped every relaunch and runner, and a conflicted pick-up that wiped an answer just applied), and those were fixed too; a fourth, limited to hold, resume and pick-up, found 21 more (6 High — among them a dead stage's evidence patch that blocked every later answer, a usage-limit cut-off whose re-run point was lost so QA's tests could vanish, a staged pick-up a dead stage or later commit could lose or split, a resumed hold that committed files no stage reported, and `apply.whitespace` rewriting or refusing patches), and those were fixed: a pick-up is committed at once, a hold returns its work with a plain `git apply`, every apply uses `--whitespace=nowarn`, dead-stage evidence lives apart and logs where to re-run, holds rank so an automatic one never replaces the owner's, and runners check the time and usage triggers themselves. One residual is accepted: a session whose lock was taken over and that compacts before its next heartbeat could read the new token from the log; a takeover of a live session needs the owner's confirmation or a relaunch loop started beside it, which the README forbids. The owner decided three on 2026-10-19: submodule stories stop their epic instead of being set aside; `--review` only records (above); and parallel runs ship with this release rather than later. The implementation also departs from the design in three small ways: hashes are `git hash-object`, a relaunch is marked by `EPIC_CYCLE_RELAUNCH=1`, and runners report parks at once through `## Clarification Needed`.

Where each finding landed:

| Finding | Severity | Where it landed |
|---|---|---|
| U-01 a parked story's code under later work | Critical | §5 setting a story aside |
| U-02 X.0 cannot run at launch | Critical | §2, §9 |
| U-03 unauthenticated inbox answers | Critical | §12, §18 (owner decision 2) |
| U-04 WIP commits escape review | Critical | §11 (owner decision 1) |
| U-05 unenforceable hold limits | Critical | §11, §14 (owner decision 1) |
| U-06 uncovered stop points | High | §6, rebuilt from an inventory of both kits |
| U-07 attended resume semantics | High | §11 |
| U-08 an untested merged tree; red feature head | High | §9, §6 |
| U-09 kit text the doctrine contradicts | High | §3 precedence; §15 pointers |
| U-10 owner-only leaks (HIGH waiver, licences) | High (partly) | §4 |
| U-11 self-serving or parallel AD amendments | High | §8 |
| U-12 agents rewriting the policy or `_bmad/` | High | §4, §3 hashes |
| U-13 the inbox inside the git tree | High | §12 (owner decision 2) |
| U-14 `--review` during a live run | High | §13 |
| U-15 unbounded fix-forward | High (partly; Medium) | §4 limits, §6 |
| U-16 sequential cross-epic continuation | High | §10 (owner decision 3) |
| U-17 the parked marker destroyed by gates | High | §9 `note=inbox=`; §10 no ledger entry for parked stories |
| U-18 the headless retrospective misstated | High | §9 (owner decision 4) |
| U-19 holds start too late | High | §11 (owner decision 1) |
| U-20 splits trigger graph re-approval | Medium | §6, §14 `reason=split` |
| U-21 infrastructure reds and the CI gates | Medium | §6 |
| U-22 contradictory classification rules | Medium | §4 order, §5 park and stop |
| U-23 runaway clarifications, splits and vetoes | Medium | §4 limits |
| U-24 parallel un-parking and contended paths | Medium | §14 |
| U-25 over-broad runtime rule; locks after holds | Medium | §4, §14 |
| U-26 "the release" undefined | Medium | §4 milestones |
| U-27 public repositories; irreversible veto items | Medium | §3, §4 |
| U-28 notification feasibility | Medium | §3, §12 |
| U-29 launch checks miss fatal configurations | Medium | §3 |
| U-30 naming, grammar and size | Low | §3, §4, §13, §15 |
