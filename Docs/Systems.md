# Game systems and ownership

TLA's gameplay ownership map: where to start reading, which side owns a change, and which authored
inputs accompany it. This is not a line-count inventory or a claim of test coverage. For repository
rules and proportionate verification, read [AGENTS.md](../AGENTS.md); for the external engine boundary,
read [FOSquadWorkspace.md](FOSquadWorkspace.md).

## Wiring and authority

- **No includes.** Baking sees `.fos` modules from explicitly listed resource-pack directories.
  `TLA.fomain` lists `Scripts`, `Scripts/Core`, `Scripts/Generated` and `Scripts/Json` for both
  `Metadata` and `Scripts`; input discovery does not recurse.
- **Game owns its libraries.** `Scripts/Core/` contains GUI and utility modules such as `Gui`,
  `Math`, `Time`, `Color`, `Sprite` and `Tween`, formerly bundled with the engine. `Scripts/Json/`
  is also game-owned. Neither is an engine source tree.
- **Entrypoints.** `Main` (server), `ClientMain` (client) and `MapperMain` (mapper) route broad lifecycle
  events. Subsystems also subscribe to their own events from `[[ModuleInit]]` functions; inspect the
  actual subscription, not only the central dispatchers.
- **Server authority.** Client input queues intent; server RPC handlers validate and execute gameplay
  changes. Server events and timers also change state; not every state change originates in an RPC.
- **Single-threaded server.** `Server.SingleThreadedLogic = True` fixes logic to one worker. Scripts
  take no entity cover; do not restore the removed `Sync` module or server `[[Async]]` cover pattern.
  The engine's reusable multithreaded mode is not supported by the current game scripts.

## Trace one player action

1. `ClientMain` / `Gui/Game.fogui` route input to `ChosenActions`.
2. Client `ChosenActions` queues intent and sends the appropriate remote call.
3. Server `ChosenActions` validates the request and delegates execution to `CritterActions`.
4. `CritterActions` spends AP and performs pickup, item use, skills, reload or attack.
5. `Combat` resolves attacks; `Parameters` supplies derived stats.
6. Replicated state/events return to the client; `Animation` selects presentation animations.

## System owners

`S` = server, `C` = client, `M` = mapper. Module names below refer to `Scripts/<name>.fos` unless a
subdirectory is explicit. Russian module headers give finer-grained scope and entrypoints.

### World, travel and locations

| Owner | Side | Responsibility / entry to inspect |
| --- | --- | --- |
| `Worldmap` | S | Encounter tables and world-map rules; `WorldmapInit` |
| `GlobalmapGroup` | S/C | Group membership and travel; global processing and movement |
| `GlobalMapLocations` | S/C | Known-location lists and synchronization |
| `Location` | S | Location creation and garbage collection |
| `Entrance` | S | Authored entry selection and critter transfer |
| `Caravan` | S | Caravan routes and schedules; initialization remains a separate migration concern |

Encounter/caravan data extraction is tracked in [AuditPlan.md](AuditPlan.md), not implied complete by this map.

### Combat, critters and lifecycle

| Owner | Side | Responsibility / entry to inspect |
| --- | --- | --- |
| `Combat` | S | Hit resolution, damage and effects; `CombatAttack`, `ApplyDamage` |
| `Parameters` | S/C | Derived property getters/setters and character rules |
| `CritterActions` | S | Authoritative actions and AP processing |
| `CritterState` | S | Alive, knockout and dead transitions |
| `NpcPlanes` / `Pattern*` | S | NPC plan processing and behavior patterns |
| `Animation` | C/M | `InitAnimation` subscribes to critter animation events; Fallout/tactics selection; sounds only on C |
| `ChosenActions` | S/C | Client action queue and server request handlers; reused for the one-shot autoattack approach/attack |
| `DenAutoAttack` / `InputHandler` | S/C | Ctrl+A prioritizes usable enemies with positive HP, then distance; approaches for melee only and attacks without aiming; [DenDefense.md](DenDefense.md#one-shot-player-autoattack) |
| `PlayerRegistration` | S/C | Login and registration; confirmed returned Player commits Den defense entry |
| `DenDefense` | S/C | Shared Den generation/primary, companions, rest/waves, ground loot and failure; `OnSuccessfulLogin`, `ResumeController`; [DenDefense.md](DenDefense.md) |
| `Merc` / `CritterAnims` | S / shared helpers | Companion ownership and weapon-animation admission; narrow Den exceptions, normal recruitment/animation policy otherwise |
| `Replication` | S | Ordinary death/respawn/permanent-death flow; Den participants use their scenario's base resurrection instead |
| `TownSupply` | S | Disabled city bounty-hunter retaliation; `CallTownSupply`, pending callbacks and direct spawns are inert; legacy guards are removed |

See [Animation.md](Animation.md) for the native baked root-motion flow, distinct from the game animation selector.

### Interaction, items and UI

| Owner | Side | Responsibility / entry to inspect |
| --- | --- | --- |
| `Dialogs` | S/C | Dialog contexts, speech/answer transport and close flow |
| `Dialog` / `NpcDialog` | S | Authored demand/result handlers and NPC-specific helpers |
| `Barter` | S | Authoritative NPC barter |
| `FixBoy` | S/C | Crafting UI and validated crafting; recipes in `Resources/ServerData/CraftRecipes.json` |
| `ClientItems` | S/C | Inventory/container presentation and synchronization |
| `Item` / `ItemMovement` | S | Doors, holodisks, deferred destruction and move rules |
| `ItemStacks` | S/C | Game-owned `Count`/`Stackable`: add, count, destroy, split and move units; client count/copy helpers |
| `MoveCursor` / `GuiScreensExt` / `DropMenuHandler` | C | Move hex, cursor cycling and held-click action menu |
| `Gui/*.fogui` | C | Authored screens and embedded callbacks, including `Gui/Game.fogui` input |
| `Core/Gui` and companion Core GUI modules | C | Game-owned retained GUI infrastructure |

`CritterItem::GetActive` resolves the selected `HandsProtoItemId` when a hand is empty;
`GetActiveMode` reads `HandsItemMode` rather than the immutable prototype's `Mode`.
`Gui/Game.fogui` redraws the main-hand plate from `OnDraw`, including empty-hand attacks,
AP cost and aiming. `ClientItems::OnCritterAction` refreshes cached inventory cells after
authoritative slot moves; the engine must apply slots before raising that event.

Merc following keeps one `MercFollow` walk plan and retargets it to the master's latest
hex with `MercMasterDist` as the stop radius. It does not queue historical player positions.
Unchanged active walk routes are not restarted every AI tick; attack plans retain priority.

The engine handles individual item instances and never merges stacks. Use `ItemStacks::` for quantity
operations rather than raw engine add/move/destroy/clone calls. `Gui/Cursor.fogui` refreshes weapon
contours against current death state each frame: corpses have no red outline or hit-chance label.
`Gui/Game.fogui` rejects dead weapon targets before aiming or queuing an attack; non-weapon corpse
interactions remain available. Generated callbacks are not authored owners.

### Infrastructure and presentation

| Owner | Side | Responsibility / entry to inspect |
| --- | --- | --- |
| `Tla` and domain metadata modules | S/C | Shared enums/action codes and game contracts; inspect side guards per declaration |
| `Stdlib` | S/C/M | Shared string/array conversions; server-only rolls, transfers, healing, dialog and quest helpers |
| `Messaging` / `MsgStr` | S/C | Player messages and text-pack key construction |
| `Sounds` | shared helpers, C playback | Map legacy/case-insensitive sound names and series to exact baked paths; call `Sounds::Play` |
| `DayColor` | shared helpers, C/M updates | Evaluate authored day curves and push `Map.SetDayColors` each frame |
| `MapTime` / `GameTime` | shared helpers, S clock setter | One scaled TLA calendar, future dates and forward-only shared dawn advance; frame callbacks retain their real delays |
| `UserOptions` | C | Persist player choices as qualified `Group.Name` local settings |
| `InputHandler` / `MapText` | C | F11 toggles overhead NPC names, initially shown; the choice lasts until client restart. Player names and speech remain visible. F11 no longer toggles fullscreen. |
| `AiControl` | S/C | Client observation/control and explicitly gated server QA commands; [AiControl.md](AiControl.md) |
| `MapperMain` | M | Editor tabs/tools and mapper lifecycle |
| `Migration` | S metadata | Current property remove/rename `///@ MigrationRule` declarations |

`Items/Custom/base_tile.foitem` owns the shared Fallout tile offset `-24 26` for floors
and roofs; roof height remains `Geometry.MapRoofElevation` in `TLA.fomain`. Do not shift
individual city maps to compensate for a missing base-tile offset.

Without `DayColor` updates maps use the engine's white default. Without `Sounds` name resolution,
`Game.PlaySound` requires an exact resource path. Neither behavior is implicitly engine-owned anymore.

## Authored content and generation

| Input | Owner / contract |
| --- | --- |
| `Dialogs/*.fodlg` | Location-prefixed dialogs; `Dialogs` runtime and `Dialog` demand/result consumers |
| `Items/`, `Critters/`, `Maps/` | Prototypes, map placements and locations; inspect the consuming gameplay rule |
| `Texts/*.russ.fotxt`, `Texts/*.engl.fotxt` | Localized packs; preserve expected bilingual keys for touched content |
| `Gui/*.fogui` | `Tools/InterfaceEditor/generate_gui_screens.py` emits `Scripts/Generated/GuiScreens.fos` |
| `Resources/ServerData/` | Server-owned JSON/data consumed by gameplay modules |
| Location/quest `.fos` modules | Regional behavior, usually matching dialog/location naming prefixes |

Baking `russ engl` is not a key-parity proof. The surviving `Tools/ContentQuality/validate_scenery_params.py`
checks map `SceneryParams`, not localization. Inspect touched keys in both language surfaces and use the
baked application when resolution matters. Generated `Scripts/Content.fos` is not hand-maintained.

## Where to begin and how to verify

- Damage/skills/AP: `Parameters`, then `Combat` or `CritterActions`.
- Player abilities: both sides of `ChosenActions`, then authoritative execution.
- Place/NPC behavior: the regional module, matching dialogs/protos, then `NpcPlanes`/`Pattern*` as needed.
- New-character shared start/Den defense: `PlayerRegistration` -> `DenDefense` and `Dialogs/den_defense_*`;
  see [DenDefense.md](DenDefense.md). Gameplay acceptance and safe runtime replacement remain user-owned.
- Dialog-callable checks/effects: `Dialog`, then `.fodlg` consumers and localized text.
- GUI fixes: owning `.fogui` and `GuiScreensExt`; preserve matching generated behavior as described in
  [AGENTS.md](../AGENTS.md#guiscreensfos-pitfall).
- Verification: choose compile, bake, affected target build, existing validator or direct application
  interaction using [AGENTS.md](../AGENTS.md#build-and-verify). `Verify :: All` is optional and does not run
  removed unit suites; archived workflows are not an active gate. No new tests/infrastructure without an
  explicit owner request.

Update this map when ownership, side, entrypoint, resource inputs or generation direction changes, not when
line counts change. Historical test counts belong in dated evidence, not in this ownership map.
