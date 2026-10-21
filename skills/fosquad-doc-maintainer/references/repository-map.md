# Game documentation and evidence map

Paths below are relative to the target game checkout unless explicitly marked external. Use them as navigation candidates, then check the current source. This is not a substitute for `AGENTS.md` or an exhaustive file inventory.

## Reading order

| Question | Maintained entry | Source of facts |
| --- | --- | --- |
| How should an agent work? | `AGENTS.md` | Owner decisions and current repository instructions |
| Where are the guides? | `Docs/README.md` | Existing maintained pages and their links |
| Which module owns gameplay? | `Docs/Systems.md` | `Scripts/`, callers and authored content references |
| How are game and engine connected? | `Docs/FOSquadWorkspace.md` | `Engine`, `EngineRevision`, both Git checkouts, `CMakeLists.txt` |
| How do I build, bake or launch? | `README.md`, `AGENTS.md` | `.vscode/tasks.json`, `.vscode/launch.json`, `CMakePresets.json`, `CMakeLists.txt`, `TLA.fomain` |
| How are scripts written? | `Docs/ScriptStyle.md`, `Nullability.md` | Actual `.fos` modules and the connected engine/compiler API |
| How can a real client be observed or controlled? | `Docs/AiControl.md` | `SourceExt/ClientAiBridge.cpp`, `Scripts/AiControl.fos`, `Tools/AiControlMcp/` |
| What is planned versus completed? | `Docs/AuditPlan.md`, `Docs/Refactoring.md`, `Docs/ReanimationPlan.md` | Explicit status and dated evidence; never infer implementation from a plan |
| What happened previously? | `Docs/History/` | Historical reports, not current acceptance evidence |

## Ownership and generation boundaries

- `Scripts/`: authoritative server gameplay versus client UI/presentation; respect side guards and event/attribute wiring.
- `Scripts/Core/` and `Scripts/Json/`: game-owned libraries, not engine source just because they originated upstream. `Core/` owns the GUI/Math/Time/Color/Sprite/Tween utility surface; both script resource packs explicitly list it.
- `Scripts/ItemStacks.fos`: game-owned item quantities, merging/splitting and movement; raw engine item operations handle instances, not units. `Scripts/Sounds.fos` resolves legacy names to exact paths; `Scripts/DayColor.fos` pushes client/mapper tint.
- `Scripts/Migration.fos`: current server property remove/rename metadata. Do not route maintainers to absent `SourceExt/ContentMigration.cpp` or assume these rules cover all historical conversions.
- `SourceExt/`: game-specific native extensions. Verify `AddEngineSources(...)` roles in `CMakeLists.txt` and the actual engine annotations/signatures.
- `Gui/*.fogui` -> `Tools/InterfaceEditor/generate_gui_screens.py` -> `Scripts/Generated/GuiScreens.fos`. `Scripts/GuiScreensExt.fos` is the hand-written companion. Do not mistake a generated callback for its source of truth.
- `Scripts/Content.fos`: generated content declarations, not an authored API to patch by hand.
- `TLA.fomain`: runtime profiles and resource-pack input directories. Baking does not recurse into unlisted script directories.
- `Dialogs/`, `Texts/`, `Items/`, `Critters/`, `Maps/`, `Resources/`: authored game data. Cross-check identifier consumers, both expected language surfaces and serialization/migration implications.
- `Build/`, `Binaries/`, `Baking/`, `Cache/`, `TLA-Dev/` and runtime logs: local outputs/evidence, not canonical input. Local `CMakeUserPresets.json` is not a portable checked-in prerequisite.
- `Engine/`: relative link to the independent engine checkout. Resolve the real target before engine inspection; do not silently recurse into it as game-owned documentation.

## Fork-specific verification facts

- TLA pins `Server.SingleThreadedLogic = True`; server scripts use no `Sync::Lock*`/`Game.Sync` cover or server `[[Async]]` pattern. Do not apply engine multithreaded guidance to this game profile.
- The game and engine's own unit suites were removed at the owner's request. Historical suite counts and test filenames do not establish current coverage.
- Manual gameplay tools such as `Scripts/Test.fos`, client-control tools and runtime diagnostics are not automatically unit tests. Verify purpose before proposing removal.
- `.github/disabled-workflows/` is an archive, not an active verification route. Old workflow commands need reconciliation before any authorized reactivation.
- `EngineRevision` can intentionally lag an uncommitted engine worktree. Record that distinction in compatibility claims.
- An upstream version number or a prior successful build is not fresh evidence for a changed checkout.

## Skill ownership

- Game skill source: `skills/fosquad-doc-maintainer/`.
- Engine skill source: `skills/fosquad-engine-doc-maintainer/` in the independent engine repository.
- Keep each skill self-contained; neither should depend on `/qsd`, a sibling skill installation, or a production server to run its documentation workflow.
