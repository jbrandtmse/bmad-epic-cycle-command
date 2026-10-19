# `/epic-cycle --unattended` and `--review`: design

Status: **approved by the owner, 2026-10-18**, including the three review-driven changes in §18. **Not implemented yet:** implementation starts when the owner says so. The field-trial project is undecided; the owner is first watching how OcuTalon does with its hand-written workaround.
Sources: the owner's requirements of 2026-10-18; the field-tested OcuPilot practice (`contests/contest-49/ocupilot-unattended-rules.md`, including its ten hindsight items); OcuTalon's distilled "Rule 22" prompt; the kit's own stop points (about 40 across the base command and the parallel kit).

## 1. What the owner asked for

- **The development loop runs unattended.** It makes decisions to the best of its ability, logs them, and keeps running.
- **Setup may still stop.** Pre-flight, branches and workspace, and the up-front planning gates keep their attended stops.
- **The design is generic.** It assumes no planner, liaison, relay or any other designated agent; one session must be enough. Nothing assumes a stack or project either.

Decisions taken on 2026-10-18:
- AD conflicts are handled **comply first** (§8).
- The retrospective runs **headless** in unattended mode (§9).
- Stages **decide small ambiguities themselves**, and escalate to the lead freely (§7).
- **`--review`** is in scope (§13).

## 2. Scope boundary

| Phase | Covers | Under `--unattended` |
|---|---|---|
| **Setup** (attended) | **Pre-flight:** lead model gate, BMAD/runtime/v7 gates, launch environment, telemetry gate. **Branches and workspace:** SC-1/SC-2, root freshness, resume-mode and integrity checks. **Up-front planning:** parallel dependency-graph approval, `spine_resolved`, `ledger_load`, the sprint-planning readiness gate, retro review and Story X.0 | **Unchanged:** stops and asks. The flag adds launch checks (§3). |
| **Development loop** (unattended) | From an epic's first `stage_spawned` to its merge: plan, implement, QA, code review, rework, AD gates, smoke, fresh check, commit and CI, the burn-down gate, and the epic-close gates (model tier, retrospective, decision sheet, merge to the feature branch). Parallel: dispatching onward, notification handling, the merge queue | **Never waits for input.** Every stop point resolves to one of the outcomes in §5. |

**Setup for the whole range runs at launch.** At launch the attended setup gates run for every epic in the range, not just the first, so a problem surfaces while the owner is present. These are the readiness gate per epic, epic-branch provisioning checks, and `ledger_load` repairs. When a gate has to run again at a later epic's start (the readiness gate, because earlier epics changed the plan):
- PASS continues;
- CONCERNS continues, with a veto item;
- FAIL parks that epic and gap-fills (§10);
- an integrity error is a hard stop for that epic.

An unattended run therefore never stops at an epic boundary to wait. *(Review finding 1.1: without this, a sequential `--unattended 3-7` run would halt after every epic. Approved by the owner, 2026-10-18.)*

## 3. Activation and launch

- **Invocation:** `/epic-cycle <range> --unattended`. The flag is logged once as `unattended_started policy=<sha256 of unattended.yaml> range=<range>`. It is carried into every runner spawn prompt and every resume prompt, so a re-invocation stays unattended.
- **Launch checks**, added to the attended pre-flight, so a failure stops while the owner is still there:
  - `_bmad/custom/unattended.yaml` exists and parses;
  - the session runs in `bypassPermissions` or `auto` mode, because a permission prompt stalls an unattended loop;
  - the launch environment covers spawn depth, `CLAUDE_CODE_RETRY_WATCHDOG=1`, and the Bash cap if the project needs it;
  - parallel runs only: the dependency graph is approved and its hash is current.
- **Re-entry:** re-invoking the same command resumes, through the kit's existing resume semantics. A crashed session is relaunched by the owner or by external tooling; the kit does not supervise its own process.
- **Compaction:** a line near the top of the command (inside what survives compaction) says that an unattended run re-reads `epic-cycle-unattended.md` after any compaction.

## 4. The policy file: `_bmad/custom/unattended.yaml`

Project-owned. The kit writes it from a generic template only when it is absent, and never overwrites it. The categories are **data, not prose**, because OcuPilot's top regret was a grant scattered across chat, logs and a stale rule. The template, with its generic defaults:

```yaml
version: 1
product_defaults:      # tie-break principles for calls made alone, most important first (the project edits these)
  - The architecture spine is the contract; where it is silent, choose the simplest option consistent with it.
  - When a choice trades safety for convenience, keep the safe behaviour and make it convenient.
  - Measure, don't guess; anything not yet measured is labelled (inference).
  - Smaller is better; split rather than stretch a story.
  - Text claims only what the product does.
owner_fixed: []        # values the owner chose; quoted verbatim in every stage, runner and resume prompt; never re-derived
stage_preamble: ""     # the project line that must head every stage prompt (e.g. "one test-runner call per message")
milestones: []         # [{date: 2026-11-30, scope: "release 1: epics 1-6"}] - the run report measures pace and risk against these
veto_window:           # decide now and report; the owner may reverse before the next release
  - security posture
  - secrets and credentials
  - privileges, roles and access grants
  - data loss, or a change to how stored data is kept
  - weakening a guarantee stated in the PRD, the spine or a spec's intent
  - public-facing text NOT already specified in the UX or PRD; claims about what the product does; licence and attribution notices; project or owner identity
  - a new third-party dependency under a permissive licence, pinned to an exact version
  - moving an acceptance criterion to a later story in the same release (never one a release gate, milestone or end-to-end acceptance test depends on)
  - amending an architecture decision (AD)
owner_only:            # park: record in the inbox, continue with independent work. Park ONLY on an explicit match here.
  - releases, tags, release pull requests, anything landing on the trunk
  - anything outward-facing (publishing, posting, filing external reports, DNS or hosting, repository or service settings, making anything public)
  - dropping scope from a release, or deferring it out of the release
  - changing a number stated as a product target in the PRD or epics (implementation details such as timeouts, batch sizes and retry counts are decide)
  - money, accounts, keys, quota; a dependency under a copyleft or unclear licence
  - agent instruction files (CLAUDE.md, AGENTS.md, anything under .claude/)
  - runtime resources this session did not start
  - rewriting pushed history (force-push); deleting branches or tags other than the kit's own merged epic branches
hard_stops:            # stop the affected unit (story, epic or parallel slot); record; continue elsewhere
  - a red CI head that is not a known flake and cannot be fixed inside its story, nor fixed forward
  - a conflict that touches an owner_only category, or cannot be classified
  - agent liveness that cannot be established; a live orphan writer that cannot be stopped
  - a tooling need that cannot be met
  - a security incident (a secret in a commit, log, trace or ANY printed output; anything pushed somewhere public) - never rewrite history
  - a workspace integrity error
gates:
  retrospective: headless      # headless | skip (skip for quota-tight runs)
  model_tier: record_only      # offers are recorded, never applied (tier changes are spend)
  merge_to_feature: when_green # when_green | hold
  merge_conflicts: resolve     # resolve (plain unions decided; behaviour choices ruled as veto items) | ask
  gap_fill: burndown           # burndown | none; name standing cleanup stories in `gap_fill_stories` to plan the supply
  gap_fill_stories: []
ci: {queue_minutes: 60, blocked_hours: 6}
hold: {triggers: [usage_warning, usage_limit], checkpoint_minutes: 30, runner_return_minutes: 45}
notify: {push: true, companion_session: ""}
```

## 5. Outcomes, and how a decision is classified

| Outcome | Meaning | Record |
|---|---|---|
| **decide** | Rule using `product_defaults`, the spine and the rules, then continue | `ruling … category=decide`; inbox FYI |
| **veto** | Decide now; the owner may reverse before the next release | `ruling … category=veto reversal=<what undoing it takes>`; inbox veto item |
| **park** | Record the question with options and a recommendation; the unit waits; independent work continues (§10) | `parked`; ledger `decision-pending … note=human=<id>`; inbox item; push notification |
| **hard stop** | Stop the affected unit; record; continue elsewhere, or end the run with its report | `unattended_stop`; inbox; push notification |
| **hold** | Checkpoint everything and end cleanly (§11) | `hold_ordered`, `hold_complete`; resume prompts; push notification |

Classification: compare the decision against `owner_only`, then `veto_window`, then `hard_stops`. **Park only on an explicit `owner_only` match.** Ambiguity between decide and veto resolves to **veto**, never to park. Parking stalls real runs, because size splits make siblings depend on each other, and the veto window exists precisely to keep work moving. *(Review finding 3.1.)* A `ruling` line always names the question, the choice, the rejected options and why, the category, and the policy hash. **`decide` outcomes stay out of the inbox:** they go to the log and the run report only. The inbox holds veto items, parked items, stops, holds and held merges. *(Review finding 5.1: the field owner declined to review routine rulings.)*

## 6. The loop's stop points, and their unattended behaviour

| Stop point (today) | Under `--unattended` |
|---|---|
| A stage returns `## Clarification Needed` | Classify, then the lead rules or parks. A ruling is written into the spec and the stage re-spawned (the re-dispatch protocol). Pipeline or environment defects (dirty tree, no subagents, failed pre-warm) are still fixed, never "answered". |
| Rule 5, tier-2 amendment (changes what the product does) | Size splits → decide. Moving an AC within the release → veto. Dropping or deferring out of the release, product numbers → park. |
| Decision dependency at spec validation (builds on an undecided item) | The undecided item is ruled if its category allows; otherwise the story parks. |
| AD conflict | Comply first (§8). |
| Rework cap reached (3 iterations) | Park the story with the outstanding findings. No cap override unattended. |
| Smoke or fresh-check failure | The rework path (already the kit's option b). |
| AD-tooled verification fails | The rework path. |
| Red CI on the story just committed | The rework path. A red head that can't be fixed in the story → hard stop for it. |
| Red matching a ledgered flake | One rerun and an `occurrence=` bump. Green continues; red takes the rework path. |
| Inherited red CI | Rerun once. If the cause is identifiable and within the lead's footprint, charter a fix-forward story (a veto item). Otherwise hard stop for the epic. |
| Orphan writer or protocol violation (Rule 18) | Apply the orphan protocol and log `protocol_violation`. A live writer that can't be stopped → hard stop for that slot or epic. |
| Merge or integrate-forward conflict | With `merge_conflicts: resolve`: a plain union is decided; a behaviour choice is ruled as a veto item; an `owner_only` match parks; an unclassifiable conflict is a hard stop. With `ask`: hard stop. |
| Infrastructure reds, three in a row / CI queued over `queue_minutes` / blocked over `blocked_hours` | Timed retries, and continue locally without pushing the next head. An inbox item once blocked past the limit. |
| CI unavailable at the merge gate | Merge held (it can't be confirmed). Inbox item. |
| Burn-down gate | Unchanged; it needs no human. |
| Model-tier checkpoint | Evaluate and log; offers recorded, never applied. |
| Retrospective question | Run headless (§9). |
| Decision sheet | Each `escalated` / `decision-pending` entry is classified: owner-only items stay `decision-pending` (inbox), veto items decided with a veto entry, the rest decided with `by=unattended`. |
| Merge to the feature branch (SC-4) | Merge when `when_green` holds (§9). Otherwise hold the epic unmerged; inbox item. Never the trunk, never a release. |
| Spend or usage limit, or the first usage warning | Hold (§11). |
| Rate limit or overload | The existing backoff, plus the retry watchdog. |
| Agent tool permission prompt, or `AskUserQuestion` | Must not occur. An unattended lead never asks; anything that would ask is one of the outcomes above. |

## 7. Stages under unattended

When the flag is set, every stage prompt gains a short **Unattended** block rendered from the policy: the product defaults, the owner-fixed values, and the category lists. The stage rule is:

- **Decide small ambiguities yourself.** "Small" means within the spec's stated intent, reversible inside the story, touching no `veto_window` or `owner_only` category, and contradicting no AD or owner-fixed value. Record each choice: plan, in the spec's `## Design Notes` under "Decided in-stage"; implement, in the spec; QA and review, in `## Decisions`. The reviewer sees them.
- **Escalate anything else with `## Clarification Needed`, and escalate freely.** When in doubt, escalate. The lead classifies and rules or parks; a stage never parks or asks the owner itself.

## 8. AD conflicts: comply first

1. **Comply.** Reshape the story's approach to meet the existing AD. If an AC can't be met within it, move that AC to a later story in the same release (a veto item). Never move one that a release gate, a milestone or an end-to-end acceptance test depends on; that case goes straight to step 2 or 3.
2. **Amend, with a veto window**, if compliance is impossible. Amend the AD on Rule 20's quick path: Rule text, an `[AMENDED <date>, <story>, unattended ruling]` marker, `memlog.py append`, `lint_spine.py`, a bookkeeping commit, the pre-warm re-run. Log `spine_updated reason=unattended_ruling`; inbox veto item.
3. **Park** if the amendment touches an `owner_only` category, or contradicts an AD in a way only a re-architecture resolves (the conflict `bmad-architecture`'s Update intent exists for). `--review` offers the interactive Update.

## 9. Epic close

- **Retrospective:** `/bmad-retrospective -H <epic>`, the skill's stable orchestrator interface. Headless, it writes its document (marked `headless: true`), with an Assumptions section and an acceptance verdict, and **does not apply the status changes it proposes**. `--review` shows the findings and the proposed changes for the owner to confirm. Log `epic_retro_complete mode=headless`.
- **Decision sheet:** as in §6.
- **Merge to the feature branch, `when_green`.** All of these must hold:
  - every story `done`, none parked;
  - the burn-down gate logged;
  - CI green on the exact head, verified on origin;
  - no conflicts, or only conflicts `integrate_conflicts` allows resolving.

  Then the merge is verified on origin. Otherwise the epic is held unmerged and the run continues with independent work.
- **Never** the trunk, a release, or anything outward-facing.

## 10. Park and continue

- **A parked story** keeps its tracker status. It gets a `decision-pending` ledger entry (`note=human=<inbox id>`) and a `story_parked` log line.
- **The lead takes the next independent story** in epic order. "Independent" means it doesn't depend on a parked story, by any of these tests:
  - `epics.md` dependency statements;
  - the `Consumes:` lists;
  - the parked story's id appearing in its acceptance criteria;
  - split children are dependent on a parked sibling by default, because `Consumes:` lists are often not updated at split time;
  - its own plan stage, which is told which stories are parked and escalates if it needs one.
- **If no story in the epic is independent:**
  1. Run the next epic whose dependencies are met (`epic-dependencies.yaml` when present; sequential order otherwise).
  2. Failing that, with `gap_fill: burndown`, take a story named in `gap_fill_stories`, or charter a cleanup story from closable ledger entries (Rule 17's chartering rules). Reactive cleanup alone runs dry; the field project planned standing cleanup stories as its gap-filler.
  3. Failing that, the run ends as `stopped: all remaining work blocked` with its report. It never sits waiting.
- **Un-parking:** at every story boundary the lead reads the inbox answers (§12). An answered item is logged `owner_decision via=inbox`, applied through the re-dispatch protocol, and the story returns to the queue.

## 11. Hold and resume

- **Triggers:**
  - the first usage warning the session can see;
  - a usage- or spend-limit error (`stage_api_error error=usage_limit`);
  - a `HOLD` request (a file in the run's state directory, or an inbox line).

  The agent usually cannot see quota, so holds trigger on warnings and errors rather than percentages.
- **Steps:**
  1. Start no new stage.
  2. Let each running stage reach its checkpoint, for at most `checkpoint_minutes` (30). After that, WIP-commit what exists and record the stage as interrupted. Field holds took 30–60 minutes, and several started too late.
  3. Make `WIP(hold)` commits by path, and push bookkeeping.
  4. Leave in-flight CI running, and record which throwaways are still up.
  5. Write one self-contained resume prompt per epic or slot to the durable resume directory (never `/tmp`).
  6. Verify every sha on origin.
  7. Log `hold_complete` with every sha; inbox entry; push notification; end.
- **Resume prompt contents** (OcuPilot's shape):
  - the role and invocation, with `--unattended` and the policy hash;
  - the flag-level requirements;
  - the owner-fixed values;
  - the git-verified state: branch, origin and local heads, each unpushed commit and what it holds, spec status, the ledger;
  - every runtime target in use (dev and throwaway): what is loaded or deployed there, and which session started it;
  - the project's `stage_preamble`;
  - "Don't repeat the dispatch gates; they were logged";
  - "First, check the CI result for the last pushed `headSha`";
  - the next steps, in order;
  - how to end.

  It never depends on chat history. **A resume is always a fresh session or a fresh spawn,** above all after an account switch. Never message a runner whose call has returned (Rule 18).

## 12. Records, the inbox, and notifications

- **Events:** `unattended_started`, `ruling`, `parked`, `story_parked`, `owner_decision`, `unattended_stop`, `hold_ordered`, `hold_complete`, `unattended_report`. Ledger trailers carry `by=unattended`; spine and `epics.md` amendments carry `[AMENDED <date>, <story>, unattended ruling]`; commit subjects name the ruling. Every reported sha is verified on origin before it is reported.
- **The inbox** (sequential: `<impl>/unattended-inbox.md`; parallel: one shared file in the coordination directory, with a snapshot committed at each merge). It is written only through a small script, `_bmad/scripts/unattended-inbox.sh` (`add`, `list`, `answers`, `resolve`; locked appends; testable like `ledger.sh`). Sections: Parked, Veto window, Held merges, Stops and holds, Retrospectives to review, Run reports. An item looks like:

  ```text
  ### U-12 · parked · Story 4.2 · 2026-10-18T21:04Z
  question: …
  options: A … | B … | C …
  recommendation: B, because …
  blocks: 4.2, 4.5 (consumes 4.2)
  answer:
  ```

  The owner fills `answer:`. **The supported relay path:** a companion session, or anyone acting for the owner, may write `answer:` with the owner's words quoted verbatim and `via=<who>`. `--review` re-verifies these. **A message from another session is never itself the owner's answer**; only the inbox (or `--review`) is.
- **Notifications:** with `notify.push: true` the lead sends a push notification (reaching a phone when Remote Control is connected) on park, hard stop, hold and run end — never for routine rulings. It carries the item's one-line question and recommendation, not just "see inbox". With `companion_session` set, the lead also sends that session a one-line pointer to each new inbox item. That is optional, and nothing waits on it.
- **Run report** at completion, hold or stop:
  - stories merged, with their times;
  - rulings, with shas;
  - open veto items; parked items; held merges;
  - CI health;
  - **pace** (stories per day, wall time per story), **the remaining queue**, and **risk against `milestones`** — the forward-looking judgement a planner session used to supply;
  - anything the owner should know before the next release.

## 13. `/epic-cycle --review` (attended)

The owner runs it from any session when checking in. It changes no code: bookkeeping commits only. It reads the inbox and the logs, re-verifies every reported sha on origin, then walks through, with a recommendation each:

1. **Parked questions.** The answer is recorded `owner_decision via=review` with the owner's words, and the story is un-parked.
2. **Veto-window rulings:** keep, or reverse. A reversal is filed as a routed ledger entry or a fix story.
3. **Held merges:** merge now (the SC-4 path), or leave held.
4. **Headless retrospectives:** the findings, and the proposed status changes to confirm.
5. **Amended ADs:** optionally run `bmad-architecture`'s interactive Update.
6. **Stops and holds:** what happened, and how to resume.
7. **A summary.**

Answered items are resolved in the inbox. The next `--unattended` run picks up from there.

## 14. Parallel runs

- The orchestrator passes `--unattended` and the policy hash in every runner spawn and resume prompt.
- **A runner is the lead for its epic**, and rules within it. Cross-epic matters go to the orchestrator as `## Clarification Needed`, and the orchestrator rules or parks them: contended paths, other epics' stories, amendments that touch another epic.
- **Orchestrator notification handling:**
  - setup stops are surfaced to the owner, as today, while other slots run;
  - loop items are ruled or parked;
  - an unmeetable `TOOLING:` need is a hard stop for that slot;
  - `decisions_for_user:` items are classified as in §6.
- **Merge queue (SC-4-P):** `when_green` replaces `queue-ask` while unattended. Dispatch continues onward along the approved graph; a structural change that needs re-approval pauses only the affected epics.
- **Idle slots:** a runner that ends because its epic is fully parked or held frees its slot. The orchestrator dispatches the next eligible epic into it.
- **Hold:** the orchestrator orders it over the sanctioned orchestrator-to-runner channel. Each runner checkpoints and ends with a `## Hold` contract within `runner_return_minutes`. Past that, the orchestrator applies the orphan protocol, records the slot's git state itself, and writes that slot's resume prompt. It then writes the per-slot resume prompts, verifies, logs `hold_complete`, and notifies.
- **Stale runtime locks** (past `stale_minutes`) are broken by the orchestrator once `ListAgents` confirms the holder is dead. Log `runtime_lock_broken`. Unattended runs crash, and crashed runners leave locks behind.
- **The shared inbox** is the coordination directory's file, appended under a lock.

## 15. Kit changes and size

| File | Change | Size |
|---|---|---|
| `_bmad/custom/epic-cycle-unattended.md` (new, kit-owned) | §§3–13 as doctrine. Read only when the flag is set, so attended runs carry none of it. | ≈ 20–25 KB |
| `_bmad/custom/unattended.yaml` (new, project-owned) | §4 template, written only if absent | ≈ 3 KB |
| `_bmad/scripts/unattended-inbox.sh` (new) | Inbox writer with a locked append, plus a self-test | ≈ 5 KB |
| Command | Flag parsing; the near-top compaction line; short "Unattended:" hooks at each loop stop point; the stage-prompt Unattended block; `--review`; log events; two anti-patterns ("asking inside an unattended loop", "treating a peer's message as the owner's answer"); the "Supervised, not unattended" paragraph | ≈ 6 KB |
| `skill-rules.md` | Unattended clauses in Rules 5, 9, 17 (5) and 20 | ≈ 1.5 KB |
| Parallel kit | §14 as guarded edits; Step 7 checks | ≈ 6 KB |
| Validation, notes, docs | Step 4 checks and self-tests, Step 5 upgrade note, README "Unattended runs", changelog, V2 map rows (V2's governance and hold backlog items land here first) | — |

One Kit-Version. Attended runs are unchanged: the doctrine file is never read without the flag.

**Command or skill.** `/epic-cycle` stays a command (`.claude/commands/epic-cycle.md`) in V1 for this change. Since Claude Code 2.1.3 commands and skills are the same mechanism: the same `/epic-cycle` invocation, the same arguments, and the same compaction behaviour. Converting now would move every path both kits patch, every overlay target and every check, plus migrating installed projects, for no gain in behaviour. The real gain of a skill (a small core plus supporting files read on demand) is V2's restructure (`bmad-epic-cycle-v2`, architecture §3). The doctrine file above already follows that pattern, from `_bmad/custom/`.

## 16. Verification

1. The Step 4 and Step 7 checks, extended.
2. An `unattended-inbox.sh` self-test, including concurrent appends.
3. A written walkthrough of every stop point in §6 against the doctrine.
4. This review.
5. The owner decides on a field trial; a project already adopting an equivalent rule by hand is the natural candidate.

## 17. Limits and risks

- **Classification errors.** Mitigated by stricter-wins, the veto window and `--review`.
- **A crashed session is not restarted by the kit.** Re-invocation is safe, but needs the owner or external tooling. The README documents an optional external relaunch loop.
- **Quota is mostly invisible** to the agent, hence the warning- and error-triggered holds.
- **An owner who rarely reviews** accumulates parked work. The run report says so, and the run ends rather than idling.
- **Headless retrospectives** are weaker than attended ones. `--review` surfaces them.
- **Auto-merging to the feature branch** can carry a ruling the owner later vetoes. Reversal is then rework, not a revert of the merge.

## 18. Review record

The Contest Planner reviewed this design on 2026-10-18, drawing on OcuPilot's and OcuTalon's runs. Every finding is adopted above.

**Three findings changed something the owner had decided. The owner approved all three on 2026-10-18:**
- **1.1 Setup for the whole range at launch.** Per-epic re-runs of the readiness gate: PASS continues, CONCERNS is a veto item, FAIL parks the epic. This replaces "a setup gate between epics stops as today".
- **1.2 `merge_conflicts: resolve` under `--unattended`.** Plain unions are decided and behaviour choices ruled as veto items, overriding `integrate_conflicts: ask` for the run. The owner approved `ask` as the attended default on 2026-10-14.
- **3.1 Park only on an explicit `owner_only` match.** Ambiguity resolves to veto. This replaces "stricter wins" as presented earlier.

**Adopted refinements:**
- 1.3 known-flake row; 1.4 inherited red (rerun, then fix forward); 1.5 orphans; 1.6 AC-move guard;
- 2.1–2.5 policy template changes;
- 3.2 split siblings; 3.3 planned gap-fill supply;
- 4.1–4.3 hold time box, resume-prompt contents, fresh resume;
- 5.1 no routine rulings in the inbox; 5.2 the relay path; 5.3 pace and milestone risk; 5.4 informative pushes;
- 6.1–6.3 idle slots, hold deadline, stale locks;
- 7.1 skip kept as an option; 7.2 documented relaunch loop.

**Left to the owner:** 7.3, the field trial. The owner is first watching how OcuTalon does with its hand-written workaround; another project may be chosen instead.
