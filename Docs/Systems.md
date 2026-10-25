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

1. `InputHandler` / `ClientMain` / `Gui/Game.fogui` route input to feature modules and `ChosenActions`.
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
| `Parameters` | S/C | Derived property getters/setters and character rules; shared server-side skill/perk spending |
| `CritterActions` | S | Authoritative actions and AP processing |
| `CritterState` | S | Alive, knockout and dead transitions |
| `NpcPlanes` / `MainPlanes` / `Pattern*` | S | NPC plan processing, weapon selection and behavior patterns; `MainPlanes` reuses a usable cached weapon but retries failed searches against current inventory/ammo/state on the next selection |
| `Animation` | C/M | `InitAnimation` subscribes to critter animation events; Fallout/tactics selection; sounds only on C |
| `ChosenActions` | S/C | Client action queue and server request handlers; reused for autoattack, priority-preserving self-care and exact-item ground-pickup batches |
| `SelfCare` / `AutoFirstAid` | C | Alt+F1 First Aid / Alt+F2 Doctor on self; Alt+Shift+F1 toggles automatic First Aid below 50% HP; normal queued skills/AP/cooldowns, healing before automatic attacks; [ClientControls.md](ClientControls.md), [AutoFirstAid.md](AutoFirstAid.md) |
| `DenAutoAttack` / `InputHandler` / `ChosenTabs` | S/C | Ctrl+A selects usable Den enemies with positive HP first, then distance; Ctrl+Shift+A repeats when AP allows, with a HUD status tab; melee-only approach, no aiming; [DenDefense.md](DenDefense.md#player-autoattack) |
| `PlayerRegistration` | S/C | Login and registration; confirmed returned Player commits Sierra siege entry |
| `MercProgression` | S/C | Owner-only NPC Character/SPECIAL sheet, manual skills/perks; dialog `Show`, snapshot and server spending; [DenDefense.md](DenDefense.md) |
| `DenDefense` | S/C | Main Sierra entrance start/re-entry, generation/primary, companions, rest/randomized waves, loot and failure; restart resets siege, not saved characters; `OnSuccessfulLogin`, `ResumeController`, `SpawnTick`; [DenDefense.md](DenDefense.md) |
| `DenCleanup` | S/C | Authored Todd by Sierra's truck at `50:56`; post-wave corpse collection, return-to-truck unloading, leftover loot/workshop supplies and persistent processed-corpse counter with client presentation; `Init`, `Setup`, `Idle`; [DenDefense.md](DenDefense.md#todd-and-post-wave-cleanup) |
| `DenWander` | S | 30% of wave-1 enemies first walk to a reachable yard hex, retaliate only against their own attackers, say an arrival line and resume normal attacks; `TryStart`, `Process`, `OnAttacked`, `WalkEnd`; [DenDefense.md](DenDefense.md#combat-and-completion) |
| `Merc` / `MercFormation` / `MercLoot` / `MercStorage` / `FavoriteItem` / `CritterAnims` | S / shared helpers | Companion ownership, leader-to-squad dialog behavior propagation (`SetBehaviorParameter`, `ROLE_LEADER`), distinct reserved Den follow positions, transient loot collection and categorized workshop-unloading jobs (`StoreLoot`, personal reserves, immediate-engagement pause), four usable workshop containers, persistent weapon readiness maintained outside combat, and weapon-animation admission; narrow Den exceptions, normal recruitment/animation policy otherwise |
| `Replication` | S | Ordinary death/respawn/permanent-death flow; Den participants use their scenario's base resurrection instead |
| `TownSupply` | S | Disabled city bounty-hunter retaliation; `CallTownSupply`, pending callbacks and direct spawns are inert; legacy guards are removed |

See [Animation.md](Animation.md) for the native baked root-motion flow, distinct from the game animation selector.

### Interaction, items and UI

| Owner | Side | Responsibility / entry to inspect |
| --- | --- | --- |
| `Dialogs` | S/C | Dialog contexts, speech/answer transport and close flow |
| `Dialog` / `NpcDialog` | S | Authored demand/result handlers and NPC-specific helpers |
| `Barter` | S | Authoritative NPC barter; free two-way item transfer with the primary's Den companions |
| `FixBoy` | S/C | Crafting UI and validated crafting; recipes in `Resources/ServerData/CraftRecipes.json` |
| `ClientItems` | S/C | Inventory/container presentation and synchronization |
| `GroundPickup` | C | Plain G snapshots all eligible items at the current hex, otherwise the nearest reachable hex within five; `Request` -> `ChosenActions::QueueGroundPickup`, exact approach and per-ID revalidation; [GroundPickup.md](GroundPickup.md) |
| `Item` / `ItemMovement` | S | Doors, holodisks, deferred destruction and move rules |
| `ItemStacks` | S/C | Game-owned `Count`/`Stackable`: add, count, destroy, split and move units; client count/copy helpers |
| `EnergyBarier` | S | Location-wide force-field networks, blockers/triggers and terminals; `BarierInit` -> `InitBariers` constructs two non-null blocker paths (both empty for short fields), then `CBarier` |
| `MoveCursor` / `GuiScreensExt` / `DropMenuHandler` | C | Move hex, cursor cycling and held-click action menu |
| `Gui/*.fogui` | C | Authored screens and embedded callbacks, including `Gui/Game.fogui` input |
| `Core/Gui` and companion Core GUI modules | C | Game-owned retained GUI infrastructure |

`CritterItem::GetActive` resolves the selected `HandsProtoItemId` when a hand is empty;
`GetActiveMode` reads `HandsItemMode` rather than the immutable prototype's `Mode`.
`Gui/Game.fogui` redraws the main-hand plate from `OnDraw`, including empty-hand attacks,
AP cost and aiming. `ClientItems::OnCritterAction` refreshes cached inventory cells after
authoritative slot moves; the engine must apply slots before raising that event.
`ClientItems::OnChosenInventoryChanged` refreshes inventory and equipment views on
`OnItemInvIn` / `OnItemInvOut`, after the engine adds or removes the owned instance.
`ClientMain::CritterAction` plays the chosen's drop animation from the server action:
`ChosenActions` sends the drop request without predicting that animation locally.
`ChosenTabs::DrawChosenTabs` shows red Doctor/First Aid tabs while the chosen's owner-synced
`TimeoutSkDoctor` / `TimeoutSkFirstAid` is active, using the same sprite/color as the injury tab and
existing localized skill names. The tabs disappear when their synchronized-time cooldown expires.

Merc following keeps one `MercFollow` walk plan, not a queue of historical player positions.
Ordinary mercs target the master's latest hex with `MercMasterDist` as the stop radius. Den companions
use `MercFormation` to reserve distinct reachable positions: weighted 1–5-hex idle rings and compact
1–3-hex moving rings, preferring pairwise separation within the same bounds. They walk to the exact
reserved hex. Unchanged active walk routes are not restarted every AI tick; attack plans retain priority.
`MercLoot` temporarily replaces following with reserved-target collection walks; completion or explicit
follow/wait releases the job. `Merc::TrySelfFirstAid` uses the ordinary skill/AP path below 50% HP,
with the standard First Aid cooldown and a localized healing line; it runs before loot/follow work.
`Merc::TrySelfDoctor` takes priority for injuries outside combat (no battle timeout or attack plan),
using ordinary skill rolls/AP and the standard Doctor cooldown after every attempt, with a localized
spoken line or emote. Doctor's cooldown does not block subsequent First Aid.
In combat only (battle timeout or current attack plan), if First Aid is unavailable below 50% HP,
`Merc::TrySelfStimpak` uses an owned ordinary stimpak (otherwise super stimpak) through the normal
item-use/AP/drug path. `Merc::TryCombatSelfHeal` gates `NpcPlanes::OnCritterIdle` before busy/plan
processing and also runs from `MercIdleImpl`: it stops movement and holds the existing plan/target
while waiting for the selected heal's AP. First Aid has priority whenever off cooldown and within
maximum AP; low current AP does not switch it to a stimpak. Unreachable AP costs are not waited for.
No AI attack, reload, weapon change or plan movement runs until
healing is performed or neither option remains available. Outside combat, only skill healing runs.
`DenDefense` no longer spends companions' stimpaks separately; `PatternCommon` and `FighterPattern`
skip hired NPC healing so these paths cannot bypass priority or create extra stimpaks for mercs.
New Den companions get a random localized personal name once at creation, stored as a `merc_name`
text-pack tag in existing public/persistent `Critter.TextArgs`. `MsgStr::NpcName`, its DialogId/TextArgs
setters and `ClientMain::CritterDescription` resolve that same identity. The leader inherits the normal
companion model and has only a personal name; `MapText::DrawCritterText` colors his overhead name
(not speech) green by prototype identity, so all clients see it without exposing owner-only role data.
See [DenDefense.md](DenDefense.md#supplies-companions-and-ground-loot).

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
| `UserOptions` / `ClientZoom` | C | Persist qualified `Group.Name` local settings without dropping unrelated options; keep zoom across maps/restarts through `ClientZoom.Level`; [ClientZoom.md](ClientZoom.md) |
| `ClientHealthDisplay` | S/C | Plain F10 toggles player/merc HP overhead (initially off, session-only); server publishes merc classification/maximum HP, client renders it; [ClientHealthDisplay.md](ClientHealthDisplay.md) |
| `InputHandler` / `MapText` | C | F11 toggles overhead NPC names, initially shown; the choice lasts until client restart. Player names and speech remain visible. F11 no longer toggles fullscreen. |
| `AiControl` | S/C | Client observation/control and explicitly gated server QA commands; [AiControl.md](AiControl.md) |
| `MapperMain` | M | Editor tabs/tools and mapper lifecycle |
| `Migration` | S metadata | Current property remove/rename `///@ MigrationRule` declarations |

[ClientControls.md](ClientControls.md) summarizes the convenience hotkeys and their shared focus gate:
active Game screen, no console/text focus/modal screen. `SelfCare::CanUse` checks queue eligibility,
not currently accumulated AP/busy; `Request(skill, true)` preserves waiting manual commands and
`IsPending()` covers queued/in-flight treatment. Server `CritterActions` and `Main` retain skill
execution and cooldown authority. `ChosenProcess` gives `AutoFirstAid::Process` the first opportunity
to enqueue healing before movement and after AP regeneration; `DenAutoAttack` respects that priority.

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
- New-character shared start/Sierra siege: `PlayerRegistration` -> `DenDefense` and `Dialogs/den_defense_*`;
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
