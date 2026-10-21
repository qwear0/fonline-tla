# The TLA Reanimation & Stabilization Plan

> Master plan to bring **FOnline: The Life After (TLA)** back into working order after years of
> dormancy, designed to be executed as a multi-stage, long-running campaign across many sessions.
> This document is the single coordinating plan; it **subsumes** [Docs/Refactoring.md](Refactoring.md)
> (the existing Scripts-only refactor) rather than duplicating it, and points at the existing tooling
> (`Tools/AiControlMcp/`, `Tools/ScriptQuality/`, `Tools/NullableEstimate/`) and docs as the
> machinery that executes it.

Audience: AI maintainers executing the revival. Read [AGENTS.md](../AGENTS.md) first for repository
practices, then this plan. Working language is Russian; this agent-facing doc stays English per the
repo convention.

---

## 1. Vision & Guiding Principles

### 1.1 The goal

A TLA install where a player can log in, traverse the world map, enter towns, talk to NPCs, accept
and complete quests across all regions, fight, trade, craft, use skills, and have the server run for
hours without exceptions. Concretely: **headless server reaches `"Start server complete!"` with zero
exceptions and stays clean under the `SOAK` profile (§2.2); the four currently-green quests stay
green; quest verification grows region-by-region toward the original 165-flag planning inventory (a future
`catalog.yaml` must be reconciled to current source before becoming its denominator); the three disabled subsystems (GameEvent, Caravan, BulletinBoard) are debugged and
re-enabled (probe-verified) or consciously retired; and the codebase reaches the polish/headers/
structure bar set by [Docs/ScriptStyle.md](ScriptStyle.md).**

This is a stabilization campaign, not a feature project. We are reviving what exists, not designing
new content.

**Out of scope (consciously excluded, so "done" is not silently blocked on them):**

- **Audio polish** — sound/music balance, missing audio events. Audio is `M` (manual) in the
  Section 6.1 matrix and stays that way; it does not gate any stage exit or the Definition of Done.
- **PvP fairness / balance** — PvP combat tuning and fairness are not part of reviving single-player
  and co-op flows; PvP is `M` in Section 6.1 and out of scope for "working order."
- **Rendering feel / visual polish** — GPU-dependent look, animation feel, layout aesthetics. We
  inspect screenshots and the existing live client for functional presentation; subjective "feel" is
  out of scope. No new golden-image test infrastructure is authorized by this plan.
- **New content** — no new quests, NPCs, maps, or items. We revive and stabilize the existing 165
  quest flags and existing systems only.

These exclusions are deliberate: a stage or the DoD is never blocked on an out-of-scope item. If an
out-of-scope item turns out to block a quest or a subsystem, it is logged in `Docs/KnownIssues.md`
and the quest/subsystem is marked `manual`/`blocked`, not held open indefinitely.

### 1.2 Hard constraints (non-negotiable; from AGENTS.md, Refactoring.md, owner decisions)

1. **Server-authoritative.** Gameplay state lives on the server. Client scripts are UI, input,
   presentation, and client-only probes. `Scripts/AiControl.fos` issues *commands* that go through
   the normal validated RPC flow. Some of those are **privileged QA commands** —
   `qa_set_prop`, `qa_set_game_prop`, `qa_give_item`, `qa_teleport_map/hex/global` directly mutate
   server state and are emphatically *not* normal gameplay. They are gated by
   `AiControl.AllowQaCommands` (**off in production**) and exist only for test/automation setup. The
   bridge therefore does grant elevated powers when `AllowQaCommands` is on; the discipline (see §5.2)
   is to reach quest prerequisites through real gameplay where feasible and treat `qa_set_*` setup as
   a documented shortcut, so the catalog can distinguish *verified end-to-end* from
   *verified-from-injected-state*.
2. **Behavior-preserving by default.** Refactors preserve behavior. Any behavior change must be a
   deliberate, code-verified bug fix with the original intent restored (check git history when
   load-bearing). Do not mask invariant failures with broad defensive fallbacks — assert or fail
   loudly.
3. **Never bulk-delete commented-out code.** Some commented blocks are migration breadcrumbs
   (Caravan/GameEvent/BulletinBoard inits, Fallout 2 reference logic). Surface and annotate with a
   reason; delete only per-block after reading git history.
4. **Comment language.** `.fos` code comments and the per-file header block are **Russian** (owner
   decision 2026-06-20). Serialized/contract names stay **English**: `///@ Property/Enum/Setting/
   Event/RemoteCall`, proto ids, text-pack keys, identifiers. Agent docs (`AGENTS.md`, `Docs/*` that
   are agent-facing) and native `SourceExt/` C++ stay English. Player-facing text follows the
   existing localized `russ engl` pack structure.
5. **Generated files are not hand-edited.** `Scripts/Content.fos` and `Scripts/Generated/GuiScreens.fos` are
   generated; `GuiScreens.fos` is regenerated from `Gui/*.fogui` via
   `Tools/InterfaceEditor/generate_gui_screens.py`. Any GUI code fix must be mirrored into the owning
   `.fogui` or it is lost on regen. `VERSION` is generated via `Tools/GenerateVersion`.
6. **Verify every step (warnings are failures).** Compile AngelScript (0 warnings) → Bake Resources
   (**Force Bake** after moving a `///@ Property`) → relevant `Build :: TLA_*` → for behavior
   changes use `TLA_ServerHeadless` startup plus direct exercise of the affected flow when startup
   cannot prove it. Unit suites were removed; no `TLA_UnitTests` gate remains. Documentation-only
   work does not automatically require a build/bake. Do not create/modify automated tests or test
   infrastructure, even temporary/delegated, without an explicit owner request.
7. **Engine is independent.** `Engine/` is a relative link to `../FOSquad_Engine`, not a submodule.
   Game behavior belongs in scripts/content or `SourceExt/`; reusable engine work needs separate
   authorization, verification and commits. `EngineRevision` records an actually published baseline,
   not an uncommitted engine worktree. Cross-check `H:/lf-30` for migration patterns when available.
8. **Follow the standing commit policy.** After code edits, create a Russian-description commit and
   push normally to the configured fork, not upstream or with rewritten history. A coordinator may
   own Git operations for delegated workers. Never stage, revert or overwrite unrelated dirty work.
9. **Single-threaded game server.** `Server.SingleThreadedLogic = True` runs logic on one worker;
   callbacks access reachable entities directly, without server `[[Async]]`, `Game.Sync` or `Sync::Lock*`
   covers. Restoring multithreaded server support is a separate project, not this stabilization plan.

### 1.3 Operating posture

- **Document-as-you-go.** Durable docs (architecture, per-system, quest catalog, known issues) are
  produced alongside verification, in the same session, not deferred to an end sprint.
- **Triage before fix.** Categorize a defect (engine vs SourceExt vs Scripts vs content) and pick the
  narrowest correct layer before touching code.
- **Ratchet, don't regress.** Every quality dimension that is green stays green; baselines only move
  down.

---

## 2. The Standing Regression Gates

These gate **every** stage and every batch. A stage is not "done" until all applicable gates are
green. They are the contract that lets the campaign run for months without silent rot. **Which gates
apply to a given edit is set by the change tier (§2.1), not "all G1–G9 every time"** — that tiering is
what keeps the inner loop affordable instead of training the operator to skip gates. The
parameterized `SOAK` profile (§2.2) is the single definition of "soak clean" referenced throughout.

| # | Gate | Command / tool | Pass criterion | Scope |
|---|------|----------------|----------------|-------|
| G1 | **AngelScript compile** | `Compile AngelScript` | 0 warnings; `TLA_ASCompiler.log` / `Build/_errors.txt` clean | every script/content change |
| G2 | **Resource bake** | `Bake Resources` (Force Bake after `///@ Property` moves) | clean bake, 0 warnings; `TLA_Baker.log`, `Build/_bake.log` clean | every content/script change |
| G3 | **Target builds** | `Build :: TLA_Server`, `:: TLA_Client` (and `:: TLA_ServerHeadless`, `:: TLA_Mapper`, `:: TLA_Baker`, `:: TLA_ASCompiler` as touched) | 0 warnings; links; binaries produced | every code change |
| G4 | **Native-boundary interaction** | Smallest affected existing application CLI/API/UI flow after G3 | observed behavior matches requirement; record gaps | native/engine-facing behavior not established by a build |
| G5 | **Headless startup, 0 exceptions** | `Prepare :: TLA_ServerHeadless` then run `--ApplySubConfig LocalTest` | exactly one `"Start server complete!"`; 0 `EntitySyncException`/`Assertion`/`Fatal`. (Use the extended local `SOAK` profile, §2.2, for sustained-runtime claims.) | any server-side behavior change |
| G6 | **Script quality ratchet** | `Analyze :: Script Quality (Ratchet)` → `validate_scripts.py --ratchet` | no (check,file) pair above `Tools/ScriptQuality/baseline.json` | every `Scripts/*.fos` change |
| G7 | **Nullable contracts & native ABI** | `Analyze :: Nullable All` (`validate_nullable.py` plus read-only analyzers) | exact, no errors | every `?`/`ptr<T>`/`nptr<T>`-touching change |
| G8 | **Formatting** | Narrowest `Format :: *` task for touched authored files; inspect scoped whitespace/diff | intended files formatted; unrelated dirty work preserved | before code/content handoff |
| G9 | **Existing quest playthroughs** | Use shipped `tla_quest_runner.py` specs or direct gameplay on disposable characters | observed stage changes match the exercised requirements; record injected setup | affected quest/content flows; no automatic nightly CI |

Notes:
- G9 starts at the four verified specs (`cassidy_letter`, `arroyo_mynoc_oil`, `den_smitty_robot`,
  `klam_vaccination`). Dated green runs are not a current pass. New automated quest specs or runner
  infrastructure require explicit owner authorization; ordinary gameplay/manual evidence can cover
  additional quests without adding a suite.
- New `Scripts/*.fos` files have no baseline entries, so any violation in them fails G6 — keep new
  files clean.
- The baseline (`Tools/ScriptQuality/baseline.json`, ~299–284 violations: 149 commented-out-code, 56
  redundant-bool-return, 55 textpack-magic-id, 19 banner-tags, 11 file-too-large, 9 hand-rolled-utils)
  may only ratchet *down*, except commented-out-code preserved-breadcrumb and file-too-large
  split-deferred cases, which are annotated rather than forced.

### 2.1 Gate tiering by change class

Running the **full** G1–G9 set on *every* edit is economically impossible at campaign cadence
(regional quest work and bounded refactor batches) and, worse, it trains the operator to
silently skip gates. So the per-change gate set is **tiered by what the change touches**. Pick the
tier from the change class; the inner loop (§5.1) references these tiers.

| Tier | Change class | Required gates (inner loop) | Notes |
|------|--------------|-----------------------------|-------|
| **T0** | Documentation-only | Source/link/whitespace review; existing validator if the documented contract warrants it | No automatic compile/bake or quest run. Quest spec/infrastructure changes need separate explicit authorization. |
| **T1** | Comment/header/format-only `.fos` change | G1 compile + G6 quality + G8 format (+ G7 when nullable syntax is touched); bake changed script resources per AGENTS | No headless run solely for unchanged behavior; verify the diff actually is non-semantic. |
| **T2** | Behavior change, symbol rename, `///@ Property`/contract move, content edit, native change | **Full G-set** applicable to the surface: G1→G2→G3→(G4 if native/engine)→G5 headless→G6→G7→G8→G9 | Bake/build/direct runtime checks follow the changed surface, not a fixed all-target chain. |

**Incremental-bake fast path.** Default to incremental `Bake Resources` in the inner loop. Reserve
**Force Bake** for a `///@ Property` move or concrete evidence of stale incremental outputs, not a
nonexistent nightly CI promise. Do not Force Bake
reflexively — it is the slowest gate in the loop.

**No implicit CI backstop.** Workflows are archived in `.github/disabled-workflows/`; source-side
archival changes remote execution only after publication, without changing remote settings. Required
verification must be performed locally or reported pending/unverified. Choose costly bake/soak runs only
when adequate to the changed surface, and never claim an unrun nightly job covers a gap. Track G records
future authorization/reconciliation work, not an active pipeline.

### 2.2 The SOAK profile (one number, referenced everywhere)

"Headless soak clean" and "multi-hour" appear in several stage exits and in the Definition of Done.
To keep them measurable rather than subjective, **SOAK is a single parameterized profile**, defined
once here and referenced by name from every stage exit and §8:

> **`SOAK` = run `TLA_ServerHeadless` for `N = 4` hours, with `M ≥ 100` NPCs active, rotating the
> active region across the populated towns (Arroyo → Klamath → Den → … ), and producing
> `0` unhandled exceptions (`EntitySyncException` / `Assertion` / `Fatal` / unhandled
> `runtime_exception`), with steady-state RSS (no monotonic growth).**

Any stage exit or DoD clause that says "soak clean" means exactly this `SOAK` run. `N` and `M` are the
knobs (raised for the Stage 4 extended/owner-playtest soak); the *clean* criterion — 0 unhandled
exceptions, region rotation, stable RSS — is fixed. `SOAK` is a proposed local acceptance profile, not evidence of an existing nightly job or a completed run.

---

## 3. Workstreams

Eight parallel tracks. They are not strictly sequential — within a stage, several advance together —
but each has dependencies that order it within the roadmap (Section 4).

### A. Systematic System Testing

**Objective:** Establish and maintain per-system smoke + behavior coverage via the AI control bridge,
and close the bridge-primitive gaps that block deeper testing.

**Tasks:**
- Build a per-system smoke matrix (Section 6.1) using the existing bridge commands documented in
  [Docs/AiControl.md](AiControl.md): movement, dialog, inventory, item-use, skills, combat, crafting,
  barter, screens, registration.
- Use existing runners/direct gameplay one flow at a time and retain their reports. The earlier proposed
  `tla_batch_quest_runner.py` / `aggregate_quest_results.py` harness and JUnit CI aggregation were not
  established; new test infrastructure is blocked pending an explicit owner request.
- Close high-value bridge gaps in priority order: `qa_get_reputation` + faction observation (~50
  quests gate on rep), `chosen.effects`/`perks_active` observation (drugs/radiation/perk verification),
  `chosen.weight`/encumbrance, `qa_advance_time` (time-gated quests), worldmap travel/encounter
  observation. Each new command extends `SourceExt/ClientAiBridge.cpp` + `Scripts/AiControl.fos` +
  the MCP adapter, and follows server-authority (QA-gated).
- Direct per-mechanic observations: combat (kill/exp/loot), barter (offers and prices), crafting
  (skill gates and consumption), skills (first-aid/repair/science). Reuse existing manual bridge tools;
  adding suites or extending runners requires separate explicit authorization.

**Tooling:** `Tools/AiControlMcp/` (smoke_ai_control_mcp.py, tla_mechanics_playtest.py,
tla_quest_runner.py), `save_screenshot` for inspection, `environment_query` for reachability.

**Dependencies:** Available existing tools and current local baseline (Stage 0). Bridge-primitive gaps must land before the quests/systems
that need them (faction rep before faction-gated quests; `qa_advance_time` before time-gated quests).

**Exit criteria:** Every system in Section 6.1 has a recorded direct observation or an explicit blocked/manual
status; priority bridge gaps are closed by separately authorized code work or have a manual fallback.
A new batch harness is not a current acceptance requirement.

### B. Quest Coverage to All 165

**Objective:** Verify quests region by region using existing shipped specs and direct gameplay, with
injected setup recorded separately. The original target was 165 flags; re-enumerate source before an
exhaustive claim. New automated specs are conditional on an explicit owner request, not this plan.

**Tasks:** Use the 17 regional planning buckets in Section 6.2, reconciling their membership to source, and
the per-batch loop (Section 5.2). For each quest: identify giver NPC + map + hex, trace dialog with
`--trace-dialog` to extract Russian answer keywords, encode stage targets + `setup` prerequisites
(annotating injected setup), exercise the actual transitions and record results. Use the manual quest
protocol (§5.6) where no shipped spec exists; author new specs only when explicitly requested.

**Tooling:** `tla_quest_runner.py` (shipped `QUESTS` dict; no `Docs/quest_specs.yaml` input) +
the shipped **`--trace-dialog` discovery mode (§5.2 step 1)**,
`qa_teleport_map/hex/global`, `qa_set_prop`, `qa_set_game_prop`, `qa_give_item`, `qa_get_prop`
(Server-scope fallback for the ~63 Server-scope flags), `environment_query path` for
reachability/skip.

**Dependencies:** Track A existing observation paths and the shipped `--trace-dialog` for discovery;
a source-reconciled inventory before claiming exhaustive completion; a catalog generator is prospective.
Track C for content/code bugs that block a quest; faction-rep and time bridge gaps for
the relevant quests. The former `DenMomSlut` and Arroyo guard-dialog blockers are fixed and live-traced.

**Exit criteria:** Per region batch, exercised quests have recorded pass/fail evidence against a
source-reconciled inventory; no suite expansion is required without authorization; every `manual` quest is `manual-verified` or
`manual-deferred` (§5.6); retain blockers and evidence in the authorized campaign report. The proposed
KnownIssues/quest catalog pages can index that evidence once created, including end-to-end versus
injected-state status; they are not required existing inputs.

### C. Bug Triage & Fixing

**Objective:** Drive the headless server and live playtests to zero exceptions; resolve the documented
breakage backlog with verified fixes.

**Tasks (priority-ordered by the bug survey):**
1. **Re-enable the disabled subsystems** (the biggest playability blockers): debug the fresh-DB
   null-derefs in `GameEvent` (DeclareEvents), `Caravan` (`AddRoutePoint`/`CaravansInit`), and
   `BulletinBoard`, currently commented out in `Main.fos::start()`. This is owner-coordinated and is
   the gate for caravan escort quests and several game events.
2. **Timer/lookup lifecycle review under single-threaded logic.** Check fallible entity-id lookups,
   current ownership/map topology and destructive/transfer behavior when touching a callback. Do not
   add server async markers or locks. The earlier ~114-site sync backlog is not current work.

   Historical migration record retained below describes the superseded multithreaded approach, not
   present code or instructions (full session evidence remains in `Docs/History/Refactoring-2026.md`):

   ```text
   The first
   Item batch (`DeferredDestroyItem`, `AutoCloseDoor`), `ReddGates::CloseGates`, `MapCoast::CoastLoop0`,
   `MapRadiation::MapLoop`, `MapModoc::ResetWaspLoc`, and the full Primal Tribe raid/quest time-event graph are
   fixed. Each callback is `[[Async]]` and owns its complete entity cover in script; reentrant transfer, radiation,
   spawn/destroy, and plane-change paths reacquire and revalidate before continuing. Primal Tribe's synchronous
   map-enter event now only schedules the covered raid job, while deferred quest results lock one player at a time.
   The energy-barrier deferred initializer and NCR/Gecko network-mode callbacks now acquire their item/blocker/map
   covers in `EnergyBarierSync.fos`, revalidate topology, and schedule only one auto-enable per network.
   Behemoth route and camera timers now revalidate the current route target/map/location and cover the viewed
   player/map/location before the Engine call. Caravan preparation, departure, deletion, and return time events
   now run only as async jobs with script-owned covers; the latest transfer contracts are centralized in
   `Sync::LockForTransferToMap` / `LockForTransferToGlobalWithGroup`. The next registry-id wave covers deferred
   destructive calls, Replicator tank/surface cleanup, Explode/SmokeGrenade, Android radio/awakening,
   Replication Bank warning/attack, NCR beggar/brahmin/Kess/Smit timers, TownSupply, and the simple Elevator,
   Jukebox, Navarro, Resources, SignalRocket, and V13Goris callbacks. `LockItemWithParent`,
   `LockMapWithLocation`, and `LockCrittersWithMaps` centralize current-parent/current-map revalidation.
   `MapSfTanker::TankerIdle` was already migrated and has been removed from the stale candidate list. Purgatory
   cleanup, all MainIntro scene callbacks plus the shared Replication path, the complete KlamCowboy timer chain,
   the remaining VcGuardsman/VcLynnet timers, and all four NpcRevenge map-loop/delayed callbacks are now migrated
   too. `MobWave` callbacks now reload their serialized state under the source-map cover, stabilize the complete
   mob/target/current-map graph, and preserve it through covered map transit. A fresh tree scan leaves 64 plain
   `[[TimeEvent]]` callbacks for triage (not all touch entities); prioritize the remaining Hunter/Merc,
   invasion/siege, and combat-pattern callbacks by entity access and
   destructive/transfer risk.
   ```
3. **Recorded content/data repairs:** all authored map bag references now resolve to configured bags: the live
   `repl_bank_den` Bag 6 fault and the four older `intro_init`/`redding_miners`/`sf_hubb` orphans are fixed, and
   the ContentQuality contract has no baseline exceptions. The dialog-parser migration blockers are also fixed:
   Todd's hashed/content script arguments
   and all 51 boolean property demand/results now preserve legacy values, with Todd and `DenMomSlut` live-green.
   The missing English `NrWriKidnap` block 3350–3354 is fixed, parity-ratcheted, and verified through the baked
   runtime pack; the shifted Den Bekky 1100 id and deathclaw-egg block 3490–3495 are fixed and runtime-verified
   too. Active Lynnet 5920–5924, Hub Lab 8030–8031 and race-start 3857–3858 strings are now covered as well.
   The former `repl_*` start-map text item was stale after the dialog-key fix: fresh live and
   screenshot checks were recorded green for all 17 replication dialogs. These are prior repair results,
   not current validator guarantees: the surviving ContentQuality validator only checks map
   `SceneryParams`, not bags or language parity. Recheck touched bilingual keys through source/bake/runtime.
4. **Intermittent `Mob::Idle` "Null assignment to non-nullable handle"** — verify whether recent
   engine MT/sanitizer fixes resolved it; **scheduled as a concrete Stage-4 task** (confirm
   non-recurrence under the `SOAK` profile, §2.2), not left as an open question; audit the
   `TryGoHome → AddWalkPlane → AddPlane` chain in `Mob.fos`/`NpcPlanes.fos` if it recurs under `SOAK`.

**Tooling:** Tiered triage (Section 5.3): static (compile/bake/validators) → headless smoke →
bridge-driven exercise → targeted repro. Bug reports follow the survey's template (file/line/severity/
status/repro/snippet/suspected-fix/test-plan).

**Dependencies:** Track A existing application paths for repro; owner coordination for subsystem re-enable (DB/serialization).

**Exit criteria:** The `SOAK` profile (§2.2) runs clean (0 unhandled exceptions); the documented
backlog items are CONFIRMED-fixed or RESOLVED-by-engine or consciously deferred with a logged reason.

### D. Refactoring (integrates Refactoring.md)

**Objective:** Carry [Docs/Refactoring.md](Refactoring.md) Round 2 to completion and execute the
god-module splits, behavior-preserving, without regressing gates.

**Tasks (Refactoring.md phases, mapped):**
- **R2-1 sign-off:** distinguish owner approval of the target look from completed mechanical work;
  the journal does not establish a separate owner acceptance. Do not repeat the backfill as a new pilot.
- **R2-2 mechanical completion:** the 2026-07-03 journal records whole-tree header backfill and subsequent
  Russian comment polish of all 13 giants. Current samples (`Main`, `ClientMain`, `GlobalmapGroup`,
  `Animation`, `Stdlib`) agree; no new exhaustive count is claimed. Remaining work is semantic/naming/
  structure review, disputed findings and owner playtests, in bounded leaves-first batches.
- **R2-3 retired:** the harness was implemented, expanded and later removed by owner request. Do not
  port `Testing.fos`, add suites or wire CI under this plan.
- **God-module splits** (Phase 4 structural), in safe-order tiers: leaves (Poker, Combat→CombatCore/
  Effects/Tables, Flags, AnimHelpers) → domain (Worldmap 4–5 parts, Caravan 3–4, FixBoy+Parameters) →
  async/timer (GlobalmapGroup, NpcPlanes) → side/infra (ClientMain, MapperMain, AiControl) → core
  (Main, last). **Cross-cutting infra modules with no owning quest batch (Main, Worldmap, Caravan,
  NpcPlanes, Parameters) use the DIFFERENTIAL gate (§5.7): existing quest/direct flows + local `SOAK` before/after,
  diffed. Caravan is split bundled with its Stage-1 re-enable, or deferred until escort quests are
  green — never split independently (§5.7 double-exposure rule).** Save-safety: properties keyed by
  name not regIndex, so splits need no MigrationRule (confirmed in Phase 4); Force Bake after
  `///@ Property` moves.
- **Standing cleanups:** commented-out-code per-block triage (annotate, don't bulk-delete), Game.Log
  audit (diagnostic vs noise; prefix kept ones), redundant-bool-return fixes, and header/comment upkeep
  when touching a module. The old 152/270 missing-header measurement predates the completed backfill.

**Tooling:** `Tools/ScriptQuality/` (validate_scripts.py, render_audit.py, baseline.json, `--ratchet`/
`--fix`), `Tools/NullableEstimate/`, `Build/_audit/` (38 group docs), `Format :: *`.

**Dependencies:** Owner sign-off where needed; leaf splits before core; refactor a system only with an
adequate before/after application baseline, not historical suite coverage. For infra modules with no
owning quest batch, the §5.7 differential gate substitutes for "quests green first."

**Exit criteria:** Outstanding owner sign-off/polish decisions resolved; all god-modules split (each infra split passed the §5.7
differential gate) or consciously deferred; baseline ratcheted down (headers 100%,
redundant-bool-return → ~0, banner-tags → 0); no gate regressions.

### E. Documentation

**Objective:** Produce the durable doc set the revival needs, document-as-you-go, each doc with a
steward.

**Proposed documentation, not existing routes:** Prefer extending `Docs/Systems.md`, `Docs/AiControl.md`
and `Docs/FOSquadWorkspace.md` first. Create a focused page only when source-grounded discovery cost
justifies it; register it in `Docs/README.md`. Original candidates under `Docs/` were: `Architecture.md` (engine/game split, authority, data flow,
subsystem map); `Systems/{Combat,Dialogs,Quests,Worldmap,AI,Crafting,Reputation,Economy}.md`
(contract + architecture + known issues + test coverage, written as each domain is refactored/tested);
`Quests/catalog.yaml` + `Quests/README.md` (all 165 quests: giver/stages/prereqs/test-status,
generated by a `Tools/QuestCatalog/generate_catalog.py` helper from CritterProps + dialogs + specs);
`ContentAuthoring.md` (how to add quest/NPC/dialog/map/item end-to-end); `TestingPlaybook.md` (bridge
setup, spec format, batch runner, troubleshooting — extracted/expanded from AiControl.md);
`KnownIssues.md` (live bug/gap register with severity/status/workaround). Cross-reference the new set
from `AGENTS.md` §Quick Reference and `README.md`.

**Catalog status: proposed, not an implemented denominator.** Before exhaustive quest claims, reconcile
an inventory to current source. A separately authorized `generate_catalog.py` could emit it by
enumerating each `///@ Property Critter ... Group = Quests` flag (165 in the original inventory), writing one
`catalog.yaml` entry per flag (giver, region/batch, Max, scope, prereqs, and a `test_status` field:
`green` / `manual-verified` / `manual-deferred` / `blocked` / `untested`). Exhaustive claims must use
that reconciled source inventory, not force a future generator to emit the old number. A maintained
catalog could then provide the denominator; prose pages follow the domains actually covered.

**Tooling:** Current skill/checklist, direct source reads and existing docs. The catalog generator,
link-checker and index generator are proposals, not available commands or permission to add test infrastructure.

**Dependencies:** Each system doc follows its Track A/D work; the quest catalog follows Track B.

**Exit criteria:** The end-of-run checklist in Section 8 is satisfied; a new contributor can read
`Architecture.md` + a system doc + `ContentAuthoring.md` and start fixing/authoring with confidence.

### F. Content Validation

**Objective:** Add the missing semantic-validation layer so content rot is caught at bake/startup,
not at runtime (or never).

**Current tasks:** Use existing bake/compiler diagnostics, `validate_scenery_params.py` for its known
map contracts, direct reference/text inspection and application observations. Do not infer localization
parity from a bake or the surviving validator. Record gaps that these checks cannot establish.

**Conditional design, not authorized infrastructure work:** The earlier proposed
`Tools/ContentValidator/validate_content.py` would need explicit owner authorization before implementation.
Its proposed scope was
tiers: **Tier 1 (fail build)** — proto-reference integrity (dialog DialogId, map NPC/item spawns,
critter inventory, caravan routes point at defined protos), text-key existence in *both* baked packs,
quest-property definitions referenced by dialogs; **Tier 2 (warn)** — dialog reachability, quest stage
≤ `Max` coherence, special answer-link `///@ Enum DialogAnswerLink` metadata presence, russ/engl
localization parity; **Tier 3/4 (info)** — quest property read/written coverage, orphaned protos/
dialogs/text keys. Integrate post-bake and optionally at headless startup
(`ValidateRuntimeConsistency`). Establish a baseline + ratchet like ScriptQuality.

**Tooling:** new validator + `Tools/ContentValidator/baseline.json`; `SourceExt/DialogBaker.cpp`
(today validates demand/result signatures only — does not check text keys or proto refs).

**Dependencies:** None hard; pairs naturally with Track B (quest authoring surfaces content gaps).

**Exit criteria:** Touched references resolve and relevant bilingual text is inspected/exercised; retain
unverified paths explicitly. The proposed Tier 1+2 pipeline is not an existing gate or a prerequisite that
can silently authorize new infrastructure.

### G. Tooling / CI / Automation

**Objective:** Keep local verification and campaign evidence usable without pretending archived CI is active.

**Current tasks:** Use `.vscode/tasks.json` and existing application tools locally. Record exact commands,
revision/configuration, outcomes and unrun checks. Use disposable worlds/characters, not production data.
Workflow definitions remain in `.github/disabled-workflows/`; no remote settings or publication are implied.

**Retained proposal, blocked pending explicit owner request:** PR smoke, nightly full bake/`SOAK`, quest
matrix/JUnit aggregation, `tla_batch_quest_runner.py`, `aggregate_quest_results.py` and a 15–20 quest critical
path were the earlier automation design. They are not an active backstop, not required repair work under
the current no-new-tests policy, and must not resurrect unit targets. Any authorized future reactivation
must reconcile archived commands, prerequisites, publication and the current testing policy first.

**Tooling:** GitHub Actions, the existing `.vscode/tasks.json` (authoritative task set),
`Build/_audit/` + `Build/_artifacts/` for progress artifacts.

**Dependencies:** Existing local checks and direct application access. Future CI has a separate owner decision.

**Exit criteria:** Applicable local checks have retained, honest results; pending verification is visible.
Campaign notes may live in `Build/_audit/progress.md`, but ignored reports are not portable prerequisites.

### H. Engine-Update Upkeep

**Objective:** Absorb separately authorized external-engine updates with a deterministic protocol so engine churn never
stalls the campaign.

**Tasks:** On each bump, run the 7-step protocol (Section 5.4): inspect upstream + cross-check
`H:/lf-30`/active sibling → recompile → Force Bake → build toolchain → build targets → headless
startup → affected native-boundary interaction; fix fallout by category (new FIXED_SETTING in `TLA.fomain`, `///@ EngineHook`
rename in `SourceExt/`, core type/API/nullability change in Scripts, single-threaded-profile compatibility, baker/
CMake `AddEngineSources` change, shader constraints). Note: effects compile for Direct3D feature level
10.0; level 9.3 code is opt-in (`Baking.Direct3DLevel9Shaders`, off in TLA because baking hangs on
`MapSprite_Contour_1px.fofx` with it), and an effect that does not fit level 9.3 fails the bake when it is
on; engine shader edits belong to the separate engine repository and need separate authorization/publication.

**Tooling:** Independent engine Git checkout, existing bake/build/application checks, `H:/lf-30/AGENTS.md` as migration
reference.

**Dependencies:** Runs out-of-band whenever the owner bumps; pauses other tracks until gates are
green again.

**Exit criteria:** Applicable existing checks pass, limitations are recorded, engine code is separately
committed/pushed to its fork, and only then the published SHA is recorded in the game pin. Follow the
standing commit policy; a delegated worker must not duplicate coordinator-owned Git operations.

---

## 4. Staged Roadmap

Stages are ordered by dependency: an existing-tool baseline, usable tracing and a source-reconciled
inventory before exhaustive quest verification; the subsystem-probe primitives (esp. `qa_advance_time` +
worldmap observation) **before** the subsystem re-enable they verify; bridge-primitive gaps before the
quests that need them; leaf refactors before core; subsystem re-enable before the quests it unblocks.
Relative size is rough effort (S/M/L/XL). Stages overlap at the seams (e.g. region quest batches in
Stage 2 run while leaf refactors in Stage 3 begin), but a stage's exit gates the *next* stage's
heavier dependents.

### Stage 0 — Foundation: local baseline, existing tracing and inventory (size: M)

**Entry:** Establish current compile/bake/build outcomes and exercise relevant shipped flows as needed.
The earlier 296/296 unit count and four green quest runs were historical entry evidence, not live gates.
**Work:**
- Lock the G1–G9 gate set + the **gate tiers** (§2.1) and the `SOAK` profile (§2.2) as the standing
  contract; capture fresh applicable results. ScriptQuality 284/299 and four green quests are old
  measurements, not targets to copy into a new report.
- **Use shipped `--trace-dialog`.** The bounded dialog tracer already exists in the runner. It extracts
  candidate Russian answer-keyword paths ranked by observed stage advance. Reuse it on disposable QA
  characters when discovery is needed; do not reimplement it or infer fresh success from the checkpoint.
  - *Exit criterion:* given a giver NPC + `dialogId`, `--trace-dialog` emits candidate answer-keyword
    sets **ranked by stage-advance**, for at least the 4 seed quests + 3 fresh Arroyo quests.
  - *Checkpoint (2026-08-08):* the bounded CLI/replay implementation, correlated authoritative reads/writes,
    internal wall-clock budget, configurable observed-map timeout, stable-slot replay for authored `@@` variants,
    and unit contracts are in place. Live disposable-character traces cover Cassidy (`Да, конечно.`,
    `ArroyoCassidyLetter` 0 → 1), Mynoc oil (`Я принесу тебе смазку...`, `ArroyoMynocOil` 0 → 1), all four Den
    Smitty/Mr. Handy transitions (`DenSmittyFixit` 0 → 1 → 2 → 3 → 4), Klamath Hish (`KlamVaccination` 0 → 1),
    and the three fresh Arroyo flags: Todd proof (`ArroyoProofOfDeath` 0 → 1), Todd's Linnett letter
    (`ArroyoLetterToLinnett` 0 → 1), and Mynoc defence (`ArroyoMynocDefence` 0 → 1). Every run restores the
    original flag; the trace deliverable meets its **7/7** exit criterion.
- **Reconcile quest inventory to source.** The original design called for `generate_catalog.py` to
  enumerate `Group = Quests` flags into `Docs/Quests/catalog.yaml` with scope/stage/status, using 165
  as its measured starting point. That generator/catalog is prospective, not an available prerequisite
  or proof of current count. Record a source-grounded inventory before an exhaustive acceptance claim;
  a new generator or automated spec expansion needs separate authorization.
- **Use existing runners with manual hygiene.** Start from a disposable QA world/character, inspect
  modal state, close existing dialogs/screens, and return to idle between runs. Never rotate or erase a
  real shard DB under this plan. New batch-driver/aggregation infrastructure remains blocked under Track G.
- **`qa_advance_time` feasibility SPIKE (load-bearing; see §5.2 / §6.1).** Prototype `qa_advance_time`,
  characterize its blast radius on a server-authoritative single-threaded server (mass timer callbacks,
  deferred destruction and ordering), and decide granularity (per-critter local clock vs global world
  clock). **If it cannot be made safe, ALL time-gated quests fall back to `manual`** (recorded in
  `catalog.yaml` + KnownIssues), and Stage 1's caravan/GameEvent re-enable verification adjusts
  accordingly. This spike's outcome gates Stage 1's subsystem re-enable verification.
- Record local required/pending checks; no CI activation, new suites or nightly jobs are authorized.
  The earlier CI proposal remains deferred in Track G.
- Establish progress tracking: `Build/_audit/progress.md` (incl. a **last-known-green marker**, §5.5),
  `Build/_artifacts/` layout.
- Write `Docs/Architecture.md` (first pass) and `Docs/TestingPlaybook.md` (extract from AiControl.md);
  create `Docs/KnownIssues.md` seeded from the bug survey.
**Produces:** A fresh local baseline, existing tracing observations, source-reconciled quest inventory,
manual hygiene and maintained owning docs. Bridge extension proposals need their own authorized code task.
**Exit:** Applicable existing local checks have observed results; inventory and missing observations are
explicit, shipped flows exercised as relevant, and timer/worldmap gaps have either a safe observation path
or a manual fallback. No removed target or inactive CI job is an exit requirement.

### Stage 1 — Per-system smoke + bug triage + subsystem re-enable (size: L)

**Entry:** Stage 0 exit (incl. the `qa_advance_time` spike verdict).

Order within the stage matters: **the bridge primitives that probe a subsystem must land before that
subsystem is re-enabled and declared green.** Re-enabling GameEvent/Caravan/BulletinBoard is not
"done" at `"Start server complete!"` — startup only proves the init no longer null-derefs. Each
subsystem needs a **direct functional probe**, which needs the observation primitives first.

**Work (ordered):**
1. Update the system × observation-method matrix (Section 6.1) with current exercised/blocked/manual status.
2. **Land the subsystem-probe primitives first:** `qa_advance_time` (per the Stage-0 spike verdict;
   if the spike said *unsafe*, time-gated probes degrade to manual and time-gated quests are `manual`)
   + worldmap travel/encounter observation (`worldmap_move`/route-progress read); `qa_get_reputation`
   + faction observation; `chosen.effects`/`weight` observation.
3. **Re-enable GameEvent / Caravan / BulletinBoard** (Track C #1) — owner-coordinated; the central
   playability unblock; gates caravan escort quests. **Re-enable is verified by a DIRECT subsystem
   probe, not just startup:**
   - *Caravan:* spawn a caravan, **`qa_advance_time`**, and verify route/escort progress on the
     worldmap (needs step 2's primitives).
   - *GameEvent:* fire one GameEvent and observe its declared effect.
   - *BulletinBoard:* post one message and read it back.
   If the `qa_advance_time` spike came back *unsafe*, the Caravan probe falls back to an owner manual
   playtest and caravan escort quests stay `manual` (recorded in `catalog.yaml`).
4. Review timer/lookup lifecycle faults when reproducing them under single-threaded logic, not the retired
   entity-cover backlog. Use current SceneryParams diagnostics only for their actual map contracts.
5. Inspect touched references/text in source, bake and application as appropriate (Track F); do not
   create the proposed ContentValidator pipeline without explicit owner authorization.
**Produces:** system smoke coverage, subsystem-probe primitives, three subsystems back online **and
probe-verified**, or explicitly deferred with manual fallbacks; touched content references checked.
**Exit:** `SOAK` (§2.2) clean with subsystems enabled; each re-enabled subsystem passes its direct
probe (or is owner-manual-verified with a logged reason); system matrix has no empty "missing
primitive" cell without an explicit manual/blocked disposition for a planned Stage 2 region.

### Stage 2 — Region-by-region quest coverage + fixes (size: XL)

**Entry:** Stage 1 exit (bridge gaps + subsystem re-enable in place) **and the Stage-0 `--trace-dialog`
deliverable green** — Track B's per-quest loop is built on `--trace-dialog`, so Stage 0 **gates** Stage 2.
**Work:** Execute the region batches (Section 6.2) via the per-batch loop (Section 5.2), ordered
Arroyo → Klamath → Den (split batches) → Modoc → Redding → Vault City → NCR → SF → Navarro →
New Reno → Broken Hills → Vault 13 → GameEvent/Replicator → Tribal/Main long-chain → cross-region
long-chain. Each batch: trace (`--trace-dialog`) → exercise shipped specs/direct gameplay → triage (file blockers to
Track C) → fix → re-verify → ratchet → catalog. Long-chain (Max 8–21) and time-gated (caravan
escorts) handled last; escorts marked `manual` unless re-enabled caravans make them automatable.
Every `manual` quest follows the **manual quest protocol** (§5.6): it gets a recorded
owner-playtest pass and a `manual-verified` (vs `manual-deferred`) `test_status` in `catalog.yaml`.
**Produces:** Recorded per-region quest observations and content/code fixes, measured against the
source-reconciled inventory; a future catalog can index that evidence, not manufacture it.
**Exit:** Every exercised quest has observed transitions and explicit setup/limits; every `manual` quest is `manual-verified` (recorded pass) or
`manual-deferred` with a logged reason; remaining quests are explicitly `blocked` with logged reasons.

### Stage 3 — Deep system hardening + refactor (size: XL)

**Entry:** Stage 2 substantially done (a region's quests green before that region's NPC modules are
restructured, so refactor regressions are caught).
**Work:**
- Reproduce fallible lookup/lifecycle problems in touched GetItem/GetCritter paths; bind nullable locals
  and revalidate actual state as required. Do not add locks to the single-threaded game profile.
- Refactoring Track D: outstanding owner sign-off and semantic polish → reviewed god-module slices
  (leaves → domain → timer → side/infra → Main), each with direct before/after observations.
- **Infra-module splits use the DIFFERENTIAL gate (§5.7).** Cross-cutting god-modules with **no owning
  quest batch** (Main, Worldmap 11.5k, Caravan, NpcPlanes, Parameters) cannot use "refactor after its
  quests are green" — there is no such batch. Instead, **exercise the same existing quest/direct flows
  and local `SOAK` before/after and compare observed state, exceptions and RSS**; the split is accepted only if the diff is clean.
  **Caravan is special:** bundle its split *with* its Stage-1 re-enable (touch the file once) **or**
  defer the split until the caravan escort quests are green — do **not** split it independently
  (double exposure of an already-fragile subsystem).
- Inspect combat/barter/crafting/skills with existing application tools; retain state/log observations.
- Game.Log and commented-code per-block triage; redundant-bool-return fixes; maintain completed headers.
**Produces:** Ratcheted-down findings, reviewed slices or explicit deferrals, and bounded behavioral evidence.
**Exit:** Outstanding sign-off/polish decisions resolved; slices passed their applicable differential checks
or remain deferred; no new quality violations; relevant local `SOAK` observations are clean. No retired
harness restoration is required.

### Stage 4 — Content validation + soak + docs polish (size: L)

**Entry:** Stage 3 exit.
**Work:**
- Inspect outstanding orphan/reference/quest read-write questions with existing tools; resolve or record
  limits. The proposed ContentValidator expansion remains conditional, not a current gate.
- Continue the localization parity pass (russ/engl text packs): active `NrWriKidnap`, Den Bekky, deathclaw egg,
  Lynnet witness, Hub Lab and race-start gaps are fixed and ratcheted, while replication dialogs have complete
  bilingual reachable-text coverage; reduce the remaining explicit legacy gaps.
- **`Mob::Idle` "Null assignment to non-nullable handle" — confirm non-recurrence under `SOAK`.**
  Treat the intermittent as a real task, not an open question: run the extended `SOAK` profile and
  confirm the `Mob::Idle` null does not recur; if it does, audit the `TryGoHome → AddWalkPlane →
  AddPlane` chain in `Mob.fos`/`NpcPlanes.fos` and fix (Track C). Closing this is required for §8 #1.
- Extended `SOAK` (§2.2 with raised `N`/`M`, plus a live owner playtest session); regression-diff
  against the Stage 0 baseline logs.
- Documentation completion: all `Docs/Systems/*`, `Quests/`, `ContentAuthoring.md` finished; cross-
  references in `AGENTS.md`/`README.md`; source/link/whitespace review locally, no implicit CI result.
**Produces:** validated content, clean `SOAK`, `Mob::Idle` confirmed non-recurring, complete doc set.
**Exit:** Section 8 Definition of Done satisfied.

### Ongoing (post-Stage-4)

Engine-update upkeep (Track H) per separately authorized update; appropriate existing local checks for
new content; direct observations and known-issue records. Keep any maintained quest inventory aligned to
source. New automated specs/infrastructure or CI reactivation require an explicit owner request.

---

## 5. The Repeatable Stage / Loop Templates

The campaign resumes across sessions by reading the progress log and the dated artifacts under
`Build/_audit/` and `Build/_artifacts/`, then picking up at the documented resume point.

### 5.1 The fix-verify-document loop (atomic unit of work)

```
1. READ      — understand the system/bug (code + git history + relevant Docs/Systems/*).
2. TRIAGE    — classify layer (engine / SourceExt / Scripts / content); pick narrowest fix.
               Also classify the CHANGE TIER (§2.1: T0 docs, T1 comment/format, T2 behavior),
               which decides the gate set in step 4.
3. CHANGE    — behavior-preserving edit, or deliberate code-verified bug fix. Russian comments,
               English contracts. Never bulk-delete commented code.
4. VERIFY    — run the TIER's gate set (§2.1), not blindly all of G1–G9:
                 T0 → source/link/whitespace review; existing validator when relevant.
                 T1 → G1 compile + G2 bake + G6 ratchet + G8 format (+ G7 if nullable syntax is touched).
                 T2 → G1 → G2 incremental bake (Force Bake only if a ///@ Property moved) → G3 →
                      G5 headless (if server behavior) → G6 → G7 → G8 → G9.
               Use G4 direct native-boundary interaction where needed. No inactive CI job covers an unrun check.
               Warnings = failure.
5. ROLLBACK  — if verification REGRESSES (a previously-green gate/quest goes red) and the root cause
               is not found within this loop: pause the affected change, diagnose and preserve evidence.
               Repair/rerun in scope; ask the owner/coordinator before abandoning or bulk-reverting work.
               Never stash/revert mixed user changes (§1.2 #8). Record the failed attempt in
               Docs/KnownIssues.md (symptom, hypothesis, what was tried), and re-establish green
               (the last-known-green marker, §5.5) before proceeding. Do not stack a second change
               on top of an un-diagnosed regression.
6. DOCUMENT  — update the relevant Docs/* (system doc, KnownIssues, quest catalog) IN THE SAME
               session; annotate any preserved commented block with a reason.
7. RECORD    — append to Build/_audit/progress.md (advance the last-known-green marker on success);
               retain authorized reports/screenshots separately from source. After code edits follow the
               Russian-description commit + fork push policy; delegated Git may belong to the coordinator.
```

### 5.2 The per-region quest batch loop (Track B)

> **Existing route:** `--trace-dialog` already ships; its 2026-08-08 checkpoint remains historical.
> Use disposable QA characters and existing specs/direct gameplay. The authoring steps below apply
> only to separately authorized automated-spec work; they do not authorize new tests or batch infrastructure.

```
PRE-BATCH HYGIENE (manual procedure using existing tools; no batch harness is supplied)
  - Use a disposable QA world/character; do not rotate/erase a real shard DB.
  - Use a FRESH char name this batch (dodges the InactivityDisconnectTime=0 stale-session gotcha).
  - Assert NO residual modal before the first quest (a leftover screen poisons dialog_answer).
PRE-BATCH DISCOVERY
  - tla_mechanics_playtest.py --teleport-map <region> --report region.json → enumerate NPCs/reachability.
  - Map quest-flag prefixes (e.g. DenSmitty*) → dialog ids / giver NPCs; confirm map proto names.
  - Cross-check prefixes against the current source inventory; use a maintained catalog only if it exists.
FOR EACH QUEST (simplest Max first):
  1. TRACE — extract the Russian answer-keyword sets with --trace-dialog (the ~160x core action):
       a. Identify giver NPC proto + dialogId (from the quest-flag prefix → dialog ref).
       b. INPUT : tla_quest_runner.py --trace-dialog --npc <proto> --dialog <dialogId>
                  --flag <Critter.QuestFlag> --report trace.json
       c. OUTPUT: trace.json = candidate answer-keyword sets RANKED by stage-advance (the answers
                  that move <flag> toward its stage target), each with the answer text + node path.
       d. Take the top-ranked set as `prefer[]`; keep alternates as fallbacks.
  2. RECORD  — document observed stages/prerequisites; use the manual protocol when no shipped spec exists.
               Only if explicitly authorized to author specs: add a QUESTS stage dict
               {map, npc, npc_hex?, target, prefer[], setup[]}.
               Server-scope flags → verify via qa_get_prop; prereqs via qa_set_prop/game_prop/give_item.
               SETUP ANNOTATION (end-to-end vs injected): prefer reaching prereqs by real gameplay.
               Every qa_set_*/qa_give_item/qa_teleport_* used for setup is a privileged shortcut
               (§1.2 #1) and MUST be recorded in the result as injected setup, so observations distinguish
               "verified end-to-end" from "verified-from-injected-state". Do not assume the shipped
               runner consumes a proposed `setup_injected` field.
  3. RUN     — tla_quest_runner.py --quest <name> --report run.json → expect ok:true, property advance.
  4. TRIAGE  — on failure: content/code bug? → file in KnownIssues + known_quest_issues.md, hand to
               Track C. Brittle keywords? → grow prefer[] from trace.json alternates. Unreachable/
               guard NPC? → probe + flag.
  5. FIX     — apply Track-C fix if in scope; re-verify gates.
  6. RE-VERIFY — exercise the quest again via the same existing route and retain actual stage evidence.
  7. RATCHET — G6/G7/G8 still green; baseline only moves down.
  8. DOCUMENT — record status, injected setup and limits in the authorized report; update a maintained
               catalog if present. Mark manual-verified/manual-deferred/blocked as needed.
  PER-QUEST TEARDOWN (manual; use the existing bridge)
  - Drain any open modal: while screen.modalActive → close_screen.
  - Reset to a known idle state (no dialog, no barter, character not mid-action) before next quest.
END
BATCH REPORT — summarize exercised flows, blockers, outcomes and limitations; retain at the authorized path.
```

Mark time-gated escorts (`Redd*CaravanEscort`, `NcrReddingCaravanEscort`) and long-chains as
`manual` (then `manual-verified`/`manual-deferred` per §5.6); revisit escorts only after caravans are
re-enabled and probe-verified (Stage 1). If the Stage-0 `qa_advance_time` spike returned *unsafe*, all
time-gated quests are `manual` by default.

### 5.3 The bug triage tiers (Track C)

```
TIER 1 static    — Compile/Bake/validators; catches missing symbols, bad refs, magic ids, nullability.
TIER 2 headless  — Prepare :: TLA_ServerHeadless; "Start server complete!" + 0 exceptions; map symptom
                   ("Bag N not found"→NpcBags; "Null assignment"→Mob::Idle/plane; EntitySyncException→
                   unexpected non-single-threaded configuration or an engine-boundary regression; do not add locks).
TIER 3 bridge    — login → qa_teleport to high-NPC map → exercise (idle loops, item lifecycle, dialog,
                   quest path) → watch event stream for runtime_exception + server log.
TIER 4 repro     — minimal direct application reproduction; apply fix; repeat the same flow and inspect
                   state/logs. Do not turn the reproduction into a new automated test suite.
```

### 5.4 The engine-update protocol (Track H)

```
PRE   — Inspect independent /apps/FOSquad_Engine status/recent upstream changes; preserve dirty work.
        Check H:/lf-30 for migration patterns when available; record current verification baseline.
BUMP  — Separately authorized engine update at a coherent published boundary, no submodule commands.
GATES (in order, fix fallout at each):
  1 Compile AngelScript        (core type/API/nullability change → Scripts)
  2 Force Bake Resources       (new FIXED_SETTING → TLA.fomain; proto layout → SubProto/MigrationRule)
  3 Build TLA_ASCompiler/Baker (engine header/API drift)
  4 Build TLA_Server/Client    (EngineHook rename / native API → SourceExt; AddEngineSources → CMake)
  5 Headless startup           (single-threaded-profile compatibility; init crash)
  6 Direct affected interaction (native/runtime behavior not established by build/startup)
POST  — Local SOAK/quest observation when relevant; preserve warnings/failures and unrun checks.
        Publish verified engine code to its fork first (Russian-description commit, normal push),
        then update EngineRevision to that published SHA and publish the game change separately.
        Delegated workers respect coordinator-owned Git operations.
```

### 5.5 Resuming across sessions

- **Source of truth for "where are we":** `Build/_audit/progress.md` (per-stage status, current
  sub-step, blockers, and the **last-known-green marker**) + dated `Build/_artifacts/` reports.
- **Last-known-green marker:** progress.md records the most recent state where the tier's gates + the
  exercised quest flows were observed green (which stage/batch/step, plus the SHA/spec snapshot). The ROLLBACK rule
  (§5.1 step 5) uses this marker for comparison while preserving partial work/evidence and seeking an
  owner/coordinator decision on unresolved regressions; it does not authorize a mixed-worktree rollback.
- **Resume rule:** re-run the cheapest gate that proves the current state (compile + incremental bake
  + headless for relevant runtime claims, not automatically for docs), confirm the last-known-green marker still holds, read progress.md, continue at the
  documented resume point of the active stage/batch.
- **Quest evidence is the memory of Track B:** shipped QUESTS entries describe runnable flows, not fresh
  results. Retain dated observations with injected setup and manual/blocked status. A future catalog can
  index them; no nightly regression is currently proving freshness.

### 5.6 The manual quest protocol (`manual-verified` vs `manual-deferred`)

Some quests are not automatable by the bridge: long-chain quests (Max 8–21 — the NCR chains, the
Tribal **MainQuest** (Max 21, the game's main quest), the SF Imperator/ZAX arcs) and time-gated
escorts. Marking them `manual` is only meaningful if "manual" is **defined** — otherwise the MAIN
QUEST has no path to "done". Each manual quest therefore needs a status in its retained evidence (and
in a maintained catalog if one is later created):

- **`manual-verified`** — has at least one *recorded owner-playtest pass* against the checklist below.
- **`manual-deferred`** — known-not-yet-verified, with a logged reason (e.g. blocked on a subsystem,
  scheduled for a later session). Allowed during the campaign; **not** allowed at the Definition of
  Done for any quest in scope.

**Per-quest manual-verification checklist template** (one filled-in copy per manual quest, stored with
the catalog entry; screenshots into `Build/_artifacts/manual/<quest>/`):

```
QUEST: <name>   GIVER: <npc proto @ map/hex>   Max: <stage cap>
1. ENTRY STATE   — exact starting conditions (char level, items held, prior flags set, faction rep).
2. STEPS         — numbered player actions to drive the chain start → finish (each a concrete UI act).
3. EXPECTED      — per-step expected outcome (dialog node reached, flag value, item granted, screen).
4. EVIDENCE      — screenshot per key step into Build/_artifacts/manual/<quest>/NN.png.
5. RESULT        — PASS/FAIL + the final flag value vs Max; date + who ran it.
```

A `manual-verified` quest is "done" for §8 #3; a `manual-deferred` quest is a tracked open item.

### 5.7 The differential gate (infra-module splits with no owning quest batch)

For cross-cutting god-modules (Main, Worldmap 11.5k, Caravan, NpcPlanes, Parameters) there is no
"owning quest batch", so "refactor after its quests are green" is undefined. These splits use a
pre/post **differential** gate instead:

```
BEFORE the split — capture a baseline:
  - Exercise the relevant shipped specs and direct gameplay flows (G9) → save actual transition results.
  - Run SOAK (§2.2) → save the exception/RSS profile.
PERFORM the split (behavior-preserving; leaves before core; Force Bake on ///@ Property moves).
AFTER the split — re-capture and DIFF:
  - Repeat the same existing flows/configuration → compare authoritative state and messages to baseline.
  - Re-run SOAK → diff exception/RSS profile (no new exceptions, no new growth).
ACCEPT only if the diff is clean; else ROLLBACK (§5.1 step 5) and re-attempt smaller.
```

**Caravan is special (double-exposure rule):** Caravan is both a Stage-1 re-enable target and a
god-module split. Do **not** split it independently. Either (a) bundle the split *with* its Stage-1
re-enable so the file is touched once under one verification, or (b) defer the split until the caravan
escort quests are green and use the differential gate then. Splitting it on its own exposes an
already-fragile subsystem twice.

---

## 6. Coverage Matrices

### 6.1 System × observation method

This matrix preserves the earlier campaign's capability/observation baseline, not a current runtime pass.
`A` = an existing automated/manual-run tool path was recorded exercised, `P` = partial observation,
`M` = manual-only, `gap` = missing observation capability. Revalidate touched flows; no entry authorizes
new automated suites or guarantees availability in the current world state.

| System | Status | Method / primitive | Missing primitive |
|--------|--------|--------------------|-------------------|
| Movement | A | `move_to_hex`, `environment_query path` | — |
| Dialogs | A | `talk_to`, `dialog_answer`; `dialog.*` observation + replication-dialog screenshot | — |
| Inventory ops | A | `move_item`, `drop_item`, `operate_container` | encumbrance: `chosen.weight` (gap) |
| Item use | A | `use_item`, `reload`, `unload` | — |
| Skills | A | `use_skill` | — |
| Combat | A | `attack_entity`; `inCombat`/`isAlive`/exp | PvP fairness = M; mob-AI path trace (gap) |
| Crafting (FixBoy) | A | `craft` | recipe-list observation (gap) |
| Barter/economy | A | dialog + `move_item`; screen obs | price-trace over campaign = P |
| Quests | A (4) / P (Server-scope) | runner + `qa_get_prop` | faction-rep gates → `qa_get_reputation` (gap) |
| Faction/reputation | gap | — | `qa_get_reputation` + faction observation |
| Buff/debuff (drugs/rad/perk) | gap | — | `chosen.effects` / `perks_active` observation |
| Party/followers | gap | — | `qa_recruit_npc` + party observation |
| Worldmap/encounters | P (canonical only) | `qa_teleport_global` (no-op from sandbox) | `worldmap_move` + encounter observation |
| Time-gated content | gap | — | `qa_advance_time` (Stage-0 feasibility spike; if unsafe → all time-gated quests `manual`) |
| Screens/GUI | A (verify M) | `show/hide/close_screen`; `screen.modalActive` | layout/animation feel = M |
| Rendering | M | Existing `save_screenshot` and manual composed-frame inspection | GPU-dependent visuals = M; no active golden-image CI |
| Audio | M | — | audio-event emit (engine) |
| Performance/soak | P | the `SOAK` profile (§2.2): headless 0-exception + region rotation + RSS tracking | profiler counters (gap) |
| Registration/login | A | `register`, `login` | — |

### 6.2 Quest region-batches (order, counts, done)

The original planning inventory counted **165** distinct `Group = Quests` Critter flags. The proposed
`generate_catalog.py` / `catalog.yaml` is not an existing authority; reconcile an exhaustive inventory
against current source before claiming completion. The regional allocation and arithmetic below preserve
the earlier plan's measurements, not a fresh census or proof that overlapping prefixes partition source.
Use it to order discovery by geography and stage complexity; dated seed runs require fresh observation.

| Batch | Region cluster | Flag prefixes | Count | Already green (seeds) | Notes |
|-------|----------------|---------------|-------|-----------------------|-------|
| 1 | Arroyo | `Arroyo*`, `arroyo_*` | 8 | `cassidy_letter`, `arroyo_mynoc_oil` | 6 new traces (ProofOfDeath, LetterToLinnett, MynocDefence, …) |
| 2 | Klamath | `Klam*` | 12 | `klam_vaccination` (Server-scope) | SmilyModoc cross-town; FindTrappers/BugenLure long-chain |
| 3 | Den — batch A (foundation) | `Den*` (Max≤4) | 13 | `den_smitty_robot`, `DenMomSlut` live trace | Max≤4 |
| 4 | Den — batch B (medium) | `Den*` (Max 3–5) | 13 | — | medium chains |
| 5 | Den — batch C (deep) | `Den*` (deep/long) | 12 | — | remaining Den flags; deeper chains |
| 6 | Modoc | `Mod*` | 10 | — | ModBaltasArmor multi-prop; brahmin/ghost chain |
| 7 | Redding | `Redd*` | 13 | — | ReddDocRadio sub-tasks; `Redd*CaravanEscort` = `manual` (time-gated) |
| 8 | Vault City | `VC*` | 20 | — | high giver density; `cassidy_letter` already exercises vc_cindy |
| 9 | NCR | `Ncr*`, `NCR*` | 19 | — | many long-chains deferred (21/15/11); `NcrReddingCaravanEscort` = `manual` |
| 10 | San Francisco | `SF*` | 13 | — | good diversity; Imperator/ZAX long ones → long-chain batch |
| 11 | Navarro | `Nav*` | 3 | — | all automatable; quick round |
| 12 | New Reno | `NR*` | 3 | — | incl. NrWriKidnap (English 3350–3354 block fixed and runtime-verified) |
| 13 | Broken Hills | `BH*` | 4 | — | BHRocketBase silo integration |
| 14 | Vault 13 | `V13*` | 2 | — | small, late-game |
| 15 | GameEvent / Replicator | `GERepl*` | 5 | — | **gated on GameEvent re-enable** (Stage 1) |
| 16 | Tribal / Main long-chain | `MainQuest`, `QChosen`, Tribal `Q*` | 7 | — | MainQuest = the Max-21 main quest → `manual` (§5.6) |
| 17 | Misc / core / cross-region | `MailDelivery`, `BarterLourensRats*`, other `Q*`-sub-flags, core | 8 | — | core/contract sub-flags + cross-region; caravan escorts = `manual` |

**Arithmetic (sums to 165):**
`8 (Arroyo) + 12 (Klamath) + 13 + 13 + 12 (Den A/B/C = 38) + 10 (Modoc) + 13 (Redding) + 20 (Vault City)
+ 19 (NCR) + 13 (SF) + 3 (Navarro) + 3 (New Reno) + 4 (Broken Hills) + 2 (Vault 13) + 5 (GERepl) +
7 (Tribal/Main) + 8 (Misc/core)`
= `8+12+38+10+13+20+19+13+3+3+4+2+5+7+8` = **165**. ✓
(Named-region subtotal `8+12+38+10+13+20+19+13+3+3+4+2 = 145`; plus `5+7+8 = 20` for the
GameEvent/Replicator, Tribal/Main, and Misc/core buckets = 165.)

The four shipped seed specs (`cassidy_letter`, `arroyo_mynoc_oil`, `den_smitty_robot`,
`klam_vaccination`) correspond to batches 1–3. Their earlier green runs do not establish current success.

Structural blockers carried into batches: ~63 Server-scope flags need `qa_get_prop`; the caravan
escorts (`Redd*CaravanEscort`, `NcrReddingCaravanEscort`, + the cross-region escorts in batch 17) are
time-gated `manual` until caravans are re-enabled **and** the `qa_advance_time` spike succeeds;
batch 15 (`GERepl*`) is blocked until GameEvent is re-enabled (Stage 1); Russian dialog keyword
extraction is per-quest via `--trace-dialog` (grow `prefer[]` from its ranked alternates); content
the remaining content-validator backlog hands off to Track C.

---

## 7. Risk Register

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| **Intermittent lifecycle/null faults** (including recorded Mob::Idle null) are hard to reproduce | High | High | Local soak and direct bridge exercise; inspect fallible lookups/topology and Mob::Idle. Under SingleThreadedLogic, unexpected sync throws require configuration/engine-boundary investigation, not restoration of script locks. |
| **Disabled subsystems block features** (GameEvent/Caravan/BulletinBoard fresh-DB null-derefs) | Certain (present) | High | Owner-coordinated re-enable in Stage 1; treat as DB/serialization investigation, not a script patch; gate caravan escort quests behind it. |
| **Content rot** (dangling proto/text/quest refs may survive baking) | High | Medium | Inspect touched references and both languages, bake and exercise resolution. Existing SceneryParams validation is narrow; proposed ContentValidator/catalog tooling is not an active guarantee. |
| **Engine churn** (frequent external-engine updates break compile/bake/native; shader-profile + FIXED_SETTING traps) | High | Medium | Deterministic 7-step protocol (Section 5.4); cross-check `H:/lf-30`/sibling; pause other tracks until gates green; keep engine shaders within the minimal profile. |
| **Automation fragility** (dialog keyword matching brittle to text changes; sandbox vs canonical world; fresh-server hygiene; Russian client requirement) | Medium | Medium | `--trace-dialog` (Stage-0 deliverable) extracts ranked keyword sets and grows `prefer[]` from its alternates; manual disposable-world hygiene + per-quest teardown (§5.2) reduce stale-session/modal poisoning; document gotchas in TestingPlaybook + known_quest_issues; quest catalog records keyword sets and end-to-end-vs-injected status. |
| **`qa_advance_time` unsafe on a server-authoritative server** (mass timer callbacks, ordering and deferred destruction; load-bearing for time-gated quests, escorts, GameEvent, effect decay) | Medium | High | Stage-0 feasibility spike characterizes blast radius and picks granularity (per-critter vs global clock); explicit fallback: if unsafe, ALL time-gated quests + the Caravan subsystem probe go `manual`/owner-playtest (recorded in catalog.yaml + KnownIssues). |
| **Scope / never-ending campaign** (165 quests + full refactor + docs is months) | High | Medium | Stage gates with explicit exit criteria; the **manual quest protocol** (§5.6) gives even the Max-21 MainQuest a concrete `manual-verified` path; mark long-chain/time-gated quests `manual` rather than forcing; ratchet + last-known-green marker prevent backsliding; progress.md keeps the run measurable and resumable. |
| **Refactor regressions in big splits** (Worldmap 11.5k, Caravan 4k) | Medium | High | Infra modules with no owning quest batch use the **differential gate** (§5.7): same existing quest/direct flows + local `SOAK` before/after, compared; Caravan split bundled with its Stage-1 re-enable or deferred until escort quests are green (no independent split); ROLLBACK (§5.1) on any un-diagnosed regression; leaves before core; behavior-preserving discipline. |
| **Commented-code deletion regret** (breadcrumbs lost) | Low–Med | Medium | Never bulk-delete; annotate with reason; per-block triage with git-history check. |
| **Publishing unrelated work / overwriting user changes** | Low | High | Read dirty files first, stage only task-owned changes, preserve user work. Follow Russian-description commit + fork push policy; keep repositories separate and respect coordinator-owned Git in delegated work. |

---

## 8. Definition of Done

"In working order" is measured, not asserted. The campaign is done when **all** of the following hold
and are demonstrated by the gates/artifacts:

1. **Server runs clean.** `TLA_ServerHeadless` reaches `"Start server complete!"` and the `SOAK`
   profile (§2.2: `N=4h`, `M≥100` NPCs, region rotation, 0 unhandled exceptions, stable RSS) passes.
   Relevant timer/lookup lifecycle faults are resolved under single-threaded logic, and the `Mob::Idle`
   null is not observed in the recorded `SOAK` (Stage 4). This bounds the observation, not a proof of
   impossibility outside the exercised state. [G5; Track C]
2. **Disabled subsystems resolved.** GameEvent, Caravan, BulletinBoard are debugged and re-enabled
   **and probe-verified** (each passes its Stage-1 direct subsystem probe, or is owner-manual-verified)
   — or consciously retired with a logged owner decision. Caravan escort quests are reachable or
   explicitly `manual` (per §5.6). [Track C #1]
3. **Quest verification.** Measured against a source-reconciled inventory (the original plan counted
   165 flags), not an assumed generated catalog: in-scope quests across the regional batches have
   recorded existing-tool/direct playthroughs, each marked **verified end-to-end** or
   **verified-from-injected-state** with actual prerequisites/shortcuts; every `manual` quest is **`manual-verified`** (at least one recorded manual-verification
   pass, §5.6) — no in-scope quest is left `manual-deferred`; the remaining quests are explicitly
   `blocked` with reasons in retained campaign evidence (indexed by a maintained catalog if present). The four seed quests
   stay green throughout. [G9; Track B]
4. **System observations.** Each in-scope system has a recorded direct observation and explicit
   remaining gaps. Priority bridge gaps are resolved by separately authorized work or have a documented
   manual path; unavailable flows remain blocked rather than silently accepted. [Track A]
5. **Content integrity.** Touched proto/text/quest references resolve through existing source/bake/
   application checks and expected russ/engl keys are verified. Unchecked domains stay explicit;
   no absent ContentValidator pipeline is claimed. [Track F]
6. **Code health.** Completed headers/comments are maintained; remaining owner sign-off/polish and
   structural slices are resolved or consciously deferred. Existing quality/nullable checks and direct
   observations pass on the retained state. Removed harnesses are not required. [G6/G7; Track D]
7. **Verification is recorded.** Applicable existing local gates have exact results and limitations.
   Archived CI is not an acceptance result; reactivation/new infrastructure remains a separate explicit
   owner decision, not a campaign completion dependency. [Track G]
8. **Docs are usable and current.** Owning maps/features cover maintained behavior and known gaps,
   are indexed from `Docs/README.md`, and reflect source. Proposed pages/catalogs are added only when
   justified, not advertised as existing commands/routes. [Track E]
9. **Engine upkeep is deliberate.** Independent engine state and published `EngineRevision` are
   distinguished; updates follow separate verification/commit/publication boundaries. [Track H]

The standing gates (G1–G9) remain green at all times throughout — that is the invariant that makes
"working order" durable rather than a one-time snapshot.

---

## Appendix: Key references

- [AGENTS.md](../AGENTS.md) — repository practices, build/verify/commit policy, engine boundary.
- [Docs/Refactoring.md](Refactoring.md) — historical Phase 0–4 outcomes, completed R2 header/comment
  backfill and remaining owner sign-off/polish; retired harness decision. Integrated by Track D.
- [Docs/ScriptStyle.md](ScriptStyle.md) — headers, Russian comments, structure, idioms, formatting.
- [Docs/AiControl.md](AiControl.md) — AI control bridge protocol, commands, observation, verified quests.
- [../Nullability.md](../Nullability.md) — script `T?` and native `ptr<T>`/`nptr<T>` strong-nullable rules.
- `Tools/AiControlMcp/` — bridge MCP adapter, `tla_quest_runner.py`, `tla_mechanics_playtest.py`,
  `smoke_ai_control_mcp.py` (+ planned `tla_batch_quest_runner.py`, `aggregate_quest_results.py`).
- `Tools/ScriptQuality/` — `validate_scripts.py`, `baseline.json` (`--ratchet`/`--fix`), `render_audit.py`.
- `Tools/NullableEstimate/` — `validate_nullable.py` and the surviving script-side nullable analyzers; unit suite removed.
- `.vscode/tasks.json` — authoritative build/bake/format/launch/analyze tasks.
- `Build/_audit/` — per-module audit docs + `progress.md`; `Build/_artifacts/` — test reports/screenshots.
- `H:/lf-30` (and active `lf-NN` sibling) — migration reference on the same engine.
