I need you to update the model configuration for all BMAD Method skills to optimize performance against token cost for an InterSystems ObjectScript stack.

**Kit-Version:** 2026-07-19.1

## Strategy (why these assignments)

Current model landscape (verify pricing at platform.claude.com/docs/en/pricing if this file is old):

| Alias    | Resolves to     | Cost (in/out per MTok)          | Context | Fit |
|----------|-----------------|----------------------------------|---------|-----|
| `opus`   | Claude Opus 4.8 | $5 / $25                         | 1M      | Deepest judgment: architecture, adversarial review, long-horizon autonomy |
| `sonnet` | Claude Sonnet 5 | $3 / $15 (intro $2 / $10 **through 2026-08-31**) | 1M      | Near-Opus quality on coding and agentic work at ~60% of the cost |
| `haiku`  | Claude Haiku 4.5| $1 / $5                          | **200K**| Mechanical, status, routing tasks with small context |

Above-Opus tiers (Fable/Mythos, $10 / $50) are deliberately absent from these assignments: Opus is the price/performance point for every opus-tier role below, including the `/epic-cycle` lead — ~2x the cost for marginal gain on pipeline work. Reserve them for exceptionally hard one-off sessions, never as a standing pin.

The optimization pattern is **expensive planner/reviewer + efficient implementer**:

- **Opus** is reserved for (a) high-judgment, low-frequency skills where quality errors cascade (architecture, spec, PRD, story-context assembly), and (b) adversarial review/critique skills that act as the quality gate over Sonnet-generated work. Since Sonnet 5 reached near-Opus coding quality, routine implementation no longer needs Opus — the Opus-tier review layer catches what slips through, which matters most on an uncommon codebase like ObjectScript.
- **Sonnet** carries all implementation, structured authoring, research, documentation, and facilitation work — the highest-frequency skills, so this is where most of the savings come from.
- **Haiku** only runs mechanical parse/route/status skills and deprecated routing shims. Never assign Haiku to a skill that may ingest a whole codebase or large multi-doc context — its window is only 200K tokens (the other two have 1M). Expectation-setting: Haiku pins pay off only on *standalone* invocations — inside `/epic-cycle`, the sprint-planning/status/story gates run lead-side, where per-skill model switching does not apply. Do not expect pipeline savings from this tier; it is correct, free future-proofing.
- Skills that orchestrate subagents matter more than they look: `bmad-code-review`'s internal review lanes (Blind Hunter, Edge Case Hunter, Acceptance Auditor) run at the parent agent's model because nested subagent spawns **inherit** the parent's model when no `model` parameter is passed — do not rely on any "same model capability as the current session" wording inside the skill (verified 2026-07-19: the installed v5/v6-era `bmad-code-review` carries no such language). The tier is actually enforced by the `/epic-cycle` stage map passing `model: opus` on the code-review spawn; the frontmatter pin documents the same intent.
- **Cost concentration (pilot-measured, 2026-07):** the Opus review layer — parent + three internal lanes, each re-reading the diff, story, and rules — is ~75-80% of per-story pipeline spend and is the only material cost lever in this document. Keep it at Opus by default: it is the safety net that makes Sonnet implementation safe, especially on an uncommon stack. A telemetry-gated de-escalation experiment exists (see rule 4's `review_tier: mixed`), but never run any review lane on a lighter tier than the implementation model.

Note: the `model:` frontmatter field is documented for both skills and slash commands but is currently IGNORED at runtime for both (anthropics/claude-code #45191, #44385 — verified 2026-07-11). Treat every pin in this document as declared intent + future-proofing for when the bug is fixed, NOT as enforcement. Enforcement paths that work today: explicit `model` parameters on Agent tool spawns (the epic-cycle stage→model map uses these), the project model-policy file `_bmad/custom/model-overrides.yaml` (read by epic-cycle's model resolution; lives in `_bmad/custom/` so BMAD upgrades never touch it — see rule 4), the epic-cycle command's pre-flight lead-model gate, and session-level model selection (`claude --model`, `/model`, or a `.claude/settings.json` `"model"` default — the base kit's Step 3b offers this).

## Task

**Preflight:** this pass expects BMAD and the epic-cycle base kit to be installed already. If `.claude/skills/` contains no `bmad-*` skills, BMAD itself is missing — stop and have the user run `npx bmad-method install` first. If the skills exist but `.claude/commands/epic-cycle.md` does not, the base kit hasn't been installed — OFFER to install `epic-cycle-workflow-creation.md` (normally in the same directory as this document) before proceeding; if the user declines, apply the skills-only pass now, skip rule 1's command pin, and note that this document should be re-run after the kit is installed.

**Version-drift reconciliation (mandatory, before applying any rule):** inventory `.claude/skills/` and reconcile against the lists below — real installs routinely diverge (verified 2026-07-19: iris-couch ships 55 skills and iris-execute-mcp-v2 ships 57, including whole families this document must cover while lacking several skills it names). Three rules:

1. An installed skill not listed here is classified by the generic tier rules; the explicit family entries below (TEA `bmad-testarch-*`, BMB builders, extra personas) exist to remove the ambiguous cases.
2. Where a listed consolidated skill (`bmad-prd`, `bmad-architecture`, `bmad-ux`, `bmad-spec`) is NOT installed, the legacy skills (`bmad-create-prd`, `bmad-edit-prd`, `bmad-validate-prd`, `bmad-create-architecture`, `bmad-create-ux-design`) are NOT deprecated shims on that install — they are the real skills. Assign them the same tier directly and ignore the shim framing for them.
3. Skills listed here but absent from the install (e.g. `bmad-dev-auto`, `bmad-forge-idea`) are simply skipped — do not create anything.

**Preserve project re-pins:** if `_bmad/custom/model-overrides.yaml` exists (see rule 4), its `overrides:` entries are deliberate per-project decisions that WIN over this document's defaults — write the override's tier into that skill's frontmatter instead of the default, and never delete or "correct" the file.

Scan the `.claude/skills/` directory (where the BMAD skills are installed) and add or update the `model:` attribute in the YAML frontmatter of each skill's `SKILL.md` according to these exact rules. Also update the one custom slash command listed in rule 1 — it lives in `.claude/commands/`, not `.claude/skills/`:

### 1. Set `model: opus` — high-judgment, correctness-critical, or adversarial-review skills (up to 21, install-dependent) + 1 custom command

Architecture, canonical contracts, context fusion, adversarial critique, and unattended autonomy. These are low-frequency or small-context skills where the quality delta is worth 2x the cost — and several propagate their model to parallel review subagents. Deprecated shims are included here at the model of the skill they forward to, so the redirected work runs at the correct tier even if the target skill's own `model:` override doesn't re-switch mid-conversation.

- bmad-architecture            (flagship architecture judgment; spawns parallel reviewer subagents; huge brownfield context)
- bmad-code-review             (adversarial quality gate; the internal lanes — Blind Hunter / Edge Case Hunter / Acceptance Auditor — inherit this model via subagent spawn inheritance; a project's `review_tier: mixed` in model-overrides.yaml can re-tier the hunter lanes, see rule 4)
- bmad-testarch-test-review    (TEA module, if installed — adversarial test-suite review gate; same logic as bmad-code-review)
- bmad-create-ux-design        (legacy installs only, where bmad-ux is absent — it is the real UX skill there; same rationale as bmad-ux)
- bmad-customize               (moved from sonnet: rare invocation so the cost is nil, but its output — behavior overrides applied to every future skill run — has exactly the propagating blast radius this tier exists for; meta-config errors are the expensive kind, per the pilot's upgrade-pass defect)
- bmad-review-adversarial-general  (cynical review — critique depth is the entire deliverable)
- bmad-review-edge-case-hunter (exhaustive branch/boundary enumeration — completeness is reasoning-bound)
- bmad-advanced-elicitation    (critique engine: socratic, pre-mortem, red-team; tiny context, pure judgment)
- bmad-check-implementation-readiness  (full-loads PRD+UX+architecture+epics; adversarial traceability gap-hunting)
- bmad-correct-course          (cross-artifact change-impact analysis; mistakes ripple across the whole plan)
- bmad-spec                    (canonical SPEC kernel consumed by every downstream skill; preservation errors are expensive)
- bmad-create-story            (highest-leverage context-fusion step — the story spec carries the judgment so dev skills can safely run Sonnet)
- bmad-dev-auto                (IF INSTALLED — absent from current v5/v6-era installs. Unattended dev loop with no human safety net: the loop's lead/gating context runs opus, but do NOT let the pin blanket-propagate to implementation subagents — if the skill spawns dev/QA stages, pass the stage map's tiers on those spawns (dev/qa sonnet, review opus) exactly as /epic-cycle does. A blanket-Opus unattended loop pays Opus rates for code the review layer already checks, ~+28%/story)
- bmad-prd                     (high-stakes PRD authoring with reviewer-gate subagent orchestration)
- bmad-ux                      (DESIGN.md/EXPERIENCE.md contracts; design judgment degrades visibly on weaker models)
- bmad-forge-idea              (adversarial persona-driven idea pressure-testing)
- bmad-prfaq                   (Working Backwards gauntlet — adversarial coaching and honest verdicts)
- bmad-create-prd              (DEPRECATED shim → forwards to bmad-prd, an opus skill — match the target's model)
- bmad-edit-prd                (DEPRECATED shim → bmad-prd; match the target's model)
- bmad-validate-prd            (DEPRECATED shim → bmad-prd; match the target's model)
- bmad-create-architecture     (DEPRECATED shim → bmad-architecture, an opus skill; match the target's model)
- **/epic-cycle** — custom slash command at `.claude/commands/epic-cycle.md`, NOT a skill (orchestration lead for the whole epic pipeline: sprint planning, story creation — the highest-leverage context-fusion step — retrospective, ADR verifications, and per-story smoke all run inline on the lead's model, and the command's own Model Strategy section mandates an Opus-tier lead. Pinning `model: opus` in the command frontmatter enforces that instead of relying on whatever model the session happens to run. The pipeline stays cheap regardless: dev/qa stages are spawned as `sonnet` subagents per the command's stage→model map. Note: `model` is officially documented for slash commands too — and equally ignored at runtime today (#45191). Keep the pin as declared intent; actual lead-tier enforcement is the command's own pre-flight Lead model gate plus the optional settings.json project default. Above-Opus tiers: the gate accepts Fable/Mythos, but Opus is the intended price/performance point for the lead — ~2x the cost for marginal lead-side gain; reserve for exceptionally hard epics, never a standing default.)

*Rule:* Any custom skill responsible for system architecture, adversarial/critical review, canonical contract or spec authoring, massive cross-artifact context fusion, or fully unattended code generation must also be set to `opus`. A deprecated routing shim always takes the model of the skill it forwards to.

### 2. Set `model: sonnet` — implementation, authoring, research, documentation, facilitation (up to 39, install-dependent)

The high-frequency workhorses. Sonnet 5 is near-Opus on coding and agentic work, so per-story implementation runs here with the Opus review layer as the safety net.

- bmad-agent-analyst           (persona loader/router — real work happens in dispatched skills)
- bmad-agent-architect         (persona loader/router)
- bmad-agent-dev               (persona loader/router into coding work)
- bmad-agent-pm                (persona loader/router)
- bmad-agent-tech-writer       (persona loader/router)
- bmad-agent-ux-designer       (persona loader/router)
- bmad-dev-story               (story implementation — the Opus-authored story spec supplies the hard judgment)
- bmad-quick-dev               (most-run general coding entry point; near-Opus coding at lower cost. Note: invoked standalone it runs INLINE on the session model — this pin applies only when spawned as a stage. Policy: run high-priority/complex bug-fix sessions from an Opus-tier session, because a hotfix has neither the Opus spec upstream nor the Opus review downstream that make Sonnet implementation safe)
- bmad-qa-generate-e2e-tests   (bounded test implementation)
- bmad-checkpoint-preview      (diff walkthrough on a bounded change; frequent per-story use)
- bmad-create-epics-and-stories (structured decomposition against clear rules)
- bmad-document-project        (whole-codebase scan — needs 1M context, rules out Haiku)
- bmad-generate-project-context (codebase distillation into project-context.md)
- bmad-technical-research      (web research fan-out can accumulate large context)
- bmad-domain-research         (cited research synthesis)
- bmad-market-research         (cited research synthesis)
- bmad-editorial-review-structure (document-architecture judgment beyond Haiku, below Opus)
- bmad-editorial-review-prose  (comprehension-vs-preference discrimination needs real linguistic judgment)
- bmad-brainstorming           (long facilitation sessions — high output volume makes Opus costly; ideation is a Sonnet strength)
- bmad-party-mode              (multi-persona facilitation; its own design already pushes subagents to cheaper models)
- bmad-retrospective           (large multi-story context — needs 1M window; synthesis is Sonnet-grade knowledge work)
- bmad-product-brief           (short-output coaching upstream of the Opus-tier PRD)

TEA test-architect module (`bmad-testarch-*` family + companions, present on current installs):

- bmad-testarch-atdd           (writes test code — never Haiku; the uncommon-stack constraint applies to tests too)
- bmad-testarch-automate       (writes test code)
- bmad-testarch-framework      (test-framework scaffolding — code)
- bmad-testarch-ci             (CI pipeline authoring — config-as-code)
- bmad-testarch-test-design    (structured test-architecture authoring)
- bmad-testarch-nfr            (NFR test planning)
- bmad-testarch-trace          (requirements-to-test traceability authoring)
- bmad-tea                     (test-architect persona loader/router)
- bmad-teach-me-testing        (teaching/facilitation)

BMB builder module (meta-authoring with a human in the loop, present on current installs):

- bmad-agent-builder           (authors agent definitions)
- bmad-bmb-setup               (builder-module setup)
- bmad-module-builder          (authors BMAD modules)
- bmad-workflow-builder        (authors workflows)
- bmad-distillator             (content distillation into module inputs)

Additional per-install skills:

- bmad-agent-qa / bmad-agent-sm / bmad-agent-quick-flow-solo-dev  (persona loaders/routers, where installed)
- bmad-init                    (project initialization — writes docs/config; too consequential for Haiku, too rare for the tier to matter)

*Rule:* Any custom skill that writes code implementations or tests, generates documentation, creates epics/stories/tickets, runs research or brainstorming workflows, or hosts an agent persona must also be set to `sonnet`.

### 3. Set `model: haiku` — mechanical, status, and routing skills (5)

All small-context (well under Haiku's 200K limit) and deterministic skills.

- bmad-sprint-status           (parse sprint-status.yaml, count, route — frequently polled, ideal for cheap model)
- bmad-sprint-planning         (deterministic epic parse → YAML emit)
- bmad-help                    (catalog lookup + next-step routing)
- bmad-index-docs              (mechanical folder indexing)
- bmad-shard-doc               (shells out to a CLI — the model barely touches content)

*Rule:* Any custom skill that only reads logs, checks workflow status, updates YAML trackers, routes intents, or splits/indexes documents must also be set to `haiku` — unless it could plausibly ingest more than ~150K tokens of content, in which case use `sonnet` (Haiku's context window is only 200K). Deprecated routing shims are NOT haiku — they take the model of the skill they forward to (see the opus rule). Reminder: these pins only take effect on standalone invocations; inside `/epic-cycle` these skills run lead-side on the lead's model (see Strategy).

### 4. Per-project model policy — `_bmad/custom/model-overrides.yaml` (uncommon-stack escalation + review de-escalation)

The tiers above are calibrated on mainstream-stack evidence (the pilot ran Node.js). On a project whose primary language is low-resource for LLMs (InterSystems ObjectScript, COBOL, ABAP, ...), the Sonnet-implementer bet is unproven — do not pre-emptively escalate, but arm the telemetry valve:

**Escalation rule (dev/qa → opus, per project):** keep `bmad-dev-story` / `bmad-quick-dev` / `bmad-qa-generate-e2e-tests` at `sonnet` for the project's first epic. At each end of epic, `/epic-cycle`'s model-tier checkpoint evaluates the cycle log; if EITHER (a) mean `cr_complete` high+med findings ≥ 2 per story with dev at a Sonnet-tier model, OR (b) ≥ 2 stories entered the rework loop for language-semantics defects, the lead recommends re-pinning those three skills to `opus` for this project. Rationale: two rework iterations (extra Sonnet dev + extra Opus review) already cost more than Opus-dev.

**De-escalation experiment (review lanes, opt-in):** after ≥ 2 consecutive epics on a stack where Opus review produced zero high and ≤ 1 medium finding per epic, the project MAY set `review_tier: mixed` — the code-review agent then spawns its bug-hunting lanes at `sonnet` while the acceptance/verification lane stays at the parent's Opus-tier model (est. −24%/story at Sonnet intro pricing, ~−15% at standard pricing). Hard rollback rule: any subsequent `smoke_complete defects_caught>0` while mixed reverts to `full-opus` immediately. Never set review lanes below the implementation tier.

**Config schema** (`_bmad/custom/model-overrides.yaml` — the installer never touches `_bmad/custom/`, so this survives BMAD upgrades; epic-cycle's model resolution reads it with top priority):

```yaml
# Project-local model policy. overrides win over SKILL.md frontmatter and the stage map.
stack_risk: uncommon            # uncommon | standard (default). uncommon arms the end-of-epic checkpoint.
review_tier: full-opus          # full-opus (default) | mixed
overrides:                      # optional: skill -> tier re-pins
  bmad-dev-story: opus
  bmad-quick-dev: opus
  bmad-qa-generate-e2e-tests: opus
history:                        # append-only; every change records its evidence
  - date: 2026-07-19
    action: escalate-dev-to-opus
    evidence: "epic-2: high+med avg 2.5/story with dev=claude-sonnet-5; 2 rework loops (ObjectScript $ORDER semantics)"
    approved_by: user
```

**Telemetry is mandatory:** every tier change must be visible in the cycle log, with the reason. `/epic-cycle` logs `model_tier_checkpoint` on each end-of-epic evaluation and `model_tier_changed` (with `direction=`, `skills=`, `from=`, `to=`, `reason=`, `evidence=`) whenever the policy file changes — see the command's Workflow Telemetry section. A tier change that isn't in the cycle log and the `history:` block didn't happen. When this pass is re-run (upgrade chain), it reads the file and preserves every recorded re-pin (see Preflight).

## Changes vs the previous version of this configuration

**2026-07-19.1** (model-tier optimization review — see `docs/model-tier-optimization-report.md` in the authoring repo):

- **Version-drift reconciliation added to Preflight** — real installs (iris-couch 55 skills, iris-execute-mcp-v2 57) diverge from this list; legacy `bmad-create-prd`/`bmad-create-architecture`/`bmad-create-ux-design` are real skills there, not shims.
- **Explicit tiers for the TEA (`bmad-testarch-*`), BMB builder, and extra-persona families** — all sonnet except `bmad-testarch-test-review` → opus (review gate).
- **`bmad-customize` moved sonnet → opus** (propagating blast radius of behavior overrides).
- **`bmad-dev-auto` marked install-conditional** and its guidance changed from blanket opus to lead-at-opus + stage-map tiers for spawned implementation subagents.
- **Corrected the `bmad-code-review` propagation mechanism** — subagent model inheritance, not in-skill capability-match language.
- **New rule 4:** per-project `_bmad/custom/model-overrides.yaml` — telemetry-gated uncommon-stack escalation (dev/qa → opus) and opt-in `review_tier: mixed` de-escalation, both logged via `model_tier_checkpoint` / `model_tier_changed`.
- **Pricing notes:** Sonnet intro pricing dated (ends 2026-08-31); above-Opus tiers explicitly out of scope.
- **Haiku expectations scoped** — standalone-invocation savings only; no effect inside `/epic-cycle`.

**2026-07-11.1:**

- **Deprecated shims pinned to their target skill's model**: bmad-create-prd, bmad-edit-prd, bmad-validate-prd, and bmad-create-architecture only print a deprecation notice and forward to their consolidated replacement (bmad-prd, bmad-architecture — both opus). They take the target's model so the forwarded work runs at the correct tier regardless of whether the target's own `model:` override re-switches mid-conversation.
- **Added skills missing from the old list**: bmad-architecture (the real architecture engine → opus), bmad-dev-auto (unattended dev loop → opus), bmad-forge-idea (adversarial idea forge → opus).
- **Added the custom /epic-cycle slash command → opus** (`.claude/commands/epic-cycle.md`): the lead runs all gate skills inline on its own model, so pinning the command guarantees the Opus-tier lead its Model Strategy section requires; stage subagents still run per its stage→model map.
- **Removed bmad-investigate** — no such skill is installed in `.claude/skills/`.
- **Implementation and PRD-adjacent work moved from opus to sonnet** (checkpoint-preview, editorial reviews, retrospective, document-project, generate-project-context, e2e tests, epics-and-stories, agent personas): Sonnet 5's near-Opus coding/agentic quality makes the Opus premium unjustified there, while the Opus-tier adversarial review skills remain the quality gate.
- **bmad-sprint-planning moved from sonnet to haiku** — it is a deterministic parse-and-emit task.

If you find a skill that isn't explicitly listed here, use the generic rules to categorize it into Opus (architecture/adversarial review/spec/context-fusion/unattended autonomy), Sonnet (coding/docs/research/facilitation/personas — including anything that writes test code), or Haiku (admin/status/routing/shims, small context only) and update it accordingly. When in doubt between sonnet and haiku, choose sonnet; when in doubt between opus and sonnet for a review/gate skill, choose opus.

Please read the files, apply the updates, and provide a brief summary of the files you modified. After applying, spot-check one skill from each tier to confirm the `model:` override actually takes effect in this Claude Code version.

## Record the pass

Create/update `_bmad/custom/kit-versions.yaml` with `skill-optimization: <this document's Kit-Version>` — the parallel kit's upgrade check reads this registry.

**⚠️ Tell the user explicitly — session restart required.** Model pins apply when a skill or the `/epic-cycle` command is next loaded; a session already mid-`/epic-cycle` keeps its old tiers. Commit ALL modified SKILL.md files + the command (`git add -A`; verify `git status --short` is clean), then instruct the user to start a fresh session before the next `/epic-cycle`.
