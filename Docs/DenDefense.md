# Sierra siege (Осада Сьерры)

## Scope and entrypoints

Owner: [`Scripts/DenDefense.fos`](../Scripts/DenDefense.fos), server-authoritative scenario with
client-only rest fades and failure input blocking. The main shared start and re-entry resurrection
use `sad_enter`, the entrance of `sierra_army_depot`. This replaces Den as the scenario base without
renaming the existing `DenDefense*` properties, script owners or dialog resource IDs.
[`Maps/sad_enter.fomap`](../Maps/sad_enter.fomap) authors Smitty at `37:60` inside the workshop
on the right of the truck (`51:51`), Todd beside the truck at `50:56`, and supplies at `35:60`.
Initial participant placement searches around `55:64`; companions use the same connected-hex search
near the player. Both retain the small outdoor spawn yard (`x=49..61`, `y=56..72`);
expanding the first-wave behavior field does not expand player/companion placement.
The workshop door at `40:57` is reopened for each attempt. Its interior is `x=33..47`, `y=58..68`.
The two visible workshop pairs are four named usable containers: screen-left melee/self-use supplies
and screen-right ranged weapons/ammunition/other loot. Left is map `35:59`, right `33:59` (increasing
hex x projects left); upper shelves use draw sublayer **1**, lower drawers **2**, with highlighting.
Their interaction anchors sit one row in front of the old wall-side `y=58` anchors; `Offset=-16 -12`
preserves the artwork's screen position while drawing it in front of the occluding wall. The left
pair used to be the single non-container scenery `generic_4282` (`SEWRKBNCH.FRM`) at `35:58`;
it is replaced, not duplicated. `MercStorage::Setup`, called by `DenCleanup::Setup`, supplies missing
containers on older maps, removes that exact decoration and sorts the original right pair's contents
before removing the old furniture. Other scenery and its blocking footprint are unchanged.
The seven original AutoCannons stay visible but use `DenDefense::DisableTurret`: no scanning timer,
appearance/attack-plane subscriptions or active plans, no enemy stack, and invulnerable.
Attempt setup also disables saved turrets and removes other entrance NPCs except Smitty and Todd,
spilling their belongings first. Map items are not purged. The old Den location is not deleted.

- [`PlayerRegistration.fos`](../Scripts/PlayerRegistration.fos) calls `OnSuccessfulLogin` with the
  **returned** `Player`, after successful registration/existing-account login and outside rollback
  catches. `Main::OnPlayerEnter` can still be rejected and must not claim a run, kit or squad.
- [`Main.fos`](../Scripts/Main.fos) creates Sierra in a fresh world and routes initial placement to the
  base. `DenDefense::ResumeController` creates a missing Sierra in an existing world; it does not
  regenerate that world or select a new primary on load. Every server start resets the siege
  to wave zero and marks the old attempt Failed, so the first confirmed login starts a new generation.
- [`den_defense_smitty.fodlg`](../Dialogs/den_defense_smitty.fodlg) owns briefing, supplies, rest and
  final claims. [`den_defense_companion.fodlg`](../Dialogs/den_defense_companion.fodlg) owns individual
  orders; [`den_defense_leader.fodlg`](../Dialogs/den_defense_leader.fodlg) exposes the same commands
  for the whole squad.
  Neither requires a minimum Intelligence/Charisma. Russian and English text are authored together.
  Client approach and server dialog validation use the NPC's `TalkDistance` plus the initiator's
  `Multihex`, not the player's own talk radius; Smitty's five-hex radius must work on both sides.
  Answer text keys use `{Speech <id> Answer <link>}` in both languages, matching `Dialogs.cpp`;
  repeated links within a speech use `Exit*`, `Exit**`, etc. in both structure and text keys.
  Ordinal `{Answer <speech> <index>}` keys do not resolve and can disconnect a talking client.
- [`Parameters.fos`](../Scripts/Parameters.fos), `CritterGetIsNoMove`, keeps the current generation's
  Smitty stationary on the server. `IsNoMove` is a computed getter without a setter; scenario setup
  must not assign it. Initialization failures retain `GetExceptionInfo()` in the outer login error.
- Actor creation passes `BagId` and `TeamId` as integers in `dict<CritterProperty, any>`; boxing their
  enum values directly serializes names rather than the numeric values these properties require.
  The controller constructs its `PatternCommon::CommonPattern` helper before calling it.
- `Scripts` and `Dialogs` are already resource-pack inputs in `TLA.fomain`; no new engine capability,
  generated-content edit or resource-pack directory is needed.

## One run, one primary

The first confirmed entrant starts a generation and owns four tribal companions. All later entrants
join that same generation without receiving another squad. Connected players also join a replacement
run as secondaries; living inventories, builds, experience and levels are retained.

| Stage | Transition |
| --- | --- |
| Starting | Reserve generation/primary before setup and reentrant callbacks; create the four companions, then Waiting |
| Waiting | A living participant with their briefing kit may request rest; the previous wave must already be cleared |
| Resting | Close conflicting dialogs, block actions, globally fade out, advance to 10:00, heal living defenders, fade in, start the next wave |
| Fighting | Spawn the full wave gradually; complete only after every required enemy has actually died |
| Victory | Stop waves; each participant may claim one `car_fuel_injection` |
| Failed | Stop progression, actions, revival and claims; **only the next successful login** starts a new generation and selects its primary |

Primary death fails the run, including after Victory. Smitty's death does **not** fail/freeze the run,
show a movie, revive him or trigger a reset on login. An already active wave can finish, but no living
Smitty means no briefing, next rest/wave or reward dialog. Death during Resting cancels that rest and
clears the blackout without starting another wave. Secondary death does not fail the run, but the
character stays dead until confirmed re-entry; there is no timed automatic revival. Dead companions
stay dead until reset. Disconnecting and ordinary Active/Victory login do not select another primary.
Restarting the server always ends the old attempt, including Victory; the first confirmed login then
selects the new primary. A replacement generation after primary death or restart sets up Smitty and
the squad anew. Saved characters retain their builds, levels, experience and remaining inventory;
reset does not restore items lost on death.

`Game` properties persist generation, IDs, stage, wave cursor, outstanding/history IDs and rest target/
phase. Critter markers persist participation and once-per-generation kit/reward claims. Frame-time
deadlines and client presentation are transient. Controller/rest/death callbacks validate generation.
On every start, `ResumeController` explicitly makes the Sierra location persistent, including its maps,
inhabitants and items: engine-created locations are transient until opted in. Player critters are
already explicitly persistent in `PlayerRegistration`. Other transient world locations are not opted
in by this scenario. It then cancels the map controller and all rest callbacks, stops scenario
actions, clears wave/spawn/rest progress and marks the attempt Failed. It retains generation and actor
IDs for `StartAttempt` to clean up old companions/enemies on the next confirmed login; that login
increments the generation, so prior kit/reward claims do not apply to the new attempt. No account,
player character or old Den town content is deleted, and the world is not regenerated.
A saved Den controller/spawn timer is stopped before switching bases; missing Smitty/Todd are created
on an already-saved Sierra entrance rather than requiring world regeneration. Old scenario companions
and wave enemies are cleaned up by ID on the next confirmed login, even if they remain on Den.
No existing serialized identifier is removed/renamed and no migration rule/engine revision bump is added.
The `LAN` profile uses persistent SQLite outside the package; see [ClientUpdates.md](ClientUpdates.md)
for storage and restart instructions. Other profiles remain Memory unless overridden.

## Supplies, companions and ground loot

- Briefing gives each participant **one crowbar, one leather jacket and two stimpaks once per generation**. Joining alone
  does not grant them. Any briefed participant may request eligible rest; every participant may claim
  their final reward. Only the primary may issue follow/wait/defend/assist orders.
- Participant and companion placement searches nearby hex rings within the outdoor `DenWander` yard,
  rejects occupied hexes and requires a route to the workshop through openable unlocked doors.
  A locally free pocket between cars is not a valid spawn; failure to find a connected hex aborts
  placement instead of trapping an actor.
- The four-person squad has exactly one leader (the first companion), using
  [`Critters/DenDefenseLeader.focr`](../Critters/DenDefenseLeader.focr), inheriting the ordinary
  companion's stats, long-haired leather-jacket model `hmlmax`, and starting sledgehammer.
  The other three use [`Critters/DenDefenseCompanion.focr`](../Critters/DenDefenseCompanion.focr).
  All start at level 3 / 3,000 XP and SPECIAL `8/4/5/4/5/10/4`
  (40 points, the five remaining stats split as evenly as integers allow). Tagged skills are Melee
  Weapons, First Aid and Small Guns. Their 30 starting skill points and one level-3 perk remain
  unspent; neither creation nor later level-ups automatically choose skills or perks. The owner uses
  **“Let's see what you can do”** in each companion's visible order menu (`Speech 2`, including the
  leader; `Speech 1` is a hidden entry router, not a player-facing menu) to open the ordinary
  Character/SPECIAL sheet and manually allocate skills or choose an available perk. SPECIAL values
  remain read-only, as on the player's sheet. Confirming skills uses the ordinary tag bonus and cost
  bands; canceling leaves pending skill allocations unspent. A perk is applied only after explicit
  selection and confirmation, independently of pending skills.
  [`MercProgression.fos`](../Scripts/MercProgression.fos) sends an owner-only snapshot of NPC stats and
  available perks; each server request rechecks ownership, life, scenario eligibility, map and talking
  distance before spending that NPC's points through `Parameters`. Leader progression never propagates
  to the squad. Existing companions retain previously spent points; no automatic respec is performed.
  The same sheet entry is available in the ordinary `all_mercenary` dialog. Deploy matching client/server
  Scripts and Metadata packs for the new RemoteCalls; snapshot state is client-local and requires no
  saved-property migration. `CreateCompanion` restores the prototype level and scenario HP reserve
  after `Main::CritterInit` regenerates registration fields, before calculating unspent points/perks;
  otherwise the common initializer resets the level to 1 and discards the scenario's HP reserve.
  Each newly created companion, including the leader, gets one random personal name from the
  29-entry `Merc/Name1..29` pool in `Texts/Text.{russ,engl}.fotxt`. `CreateCompanion` stores its
  `@text Text Merc NameN@` tag under `merc_name` in the existing public, persistent `Critter.TextArgs`;
  the name is not rerolled on map entry/reconnect and is localized on each client. Independent draws
  may repeat. `MsgStr::NpcName` and the DialogId/TextArgs setters keep captions and look text consistent;
  old companions without name arguments retain their dialog/prototype fallback until replaced.
  The leader has no title appended to his personal name. `MapText::DrawCritterText` colors only his
  overhead name green by prototype identity (not the owner-only `NpcRole`); speech colors and F11
  name visibility remain unchanged. Deploy matching Scripts, Texts and Protos packs.
  Each starts with **five consumable stimpaks** and exactly one equipped weapon, in squad order:
  **sledgehammer, combat knife, crowbar, club**. Smitty does not equip or resupply companions.
  Maximum/starting HP are **103/91/103/91**, another 50% over the previous 69/61/69/61 with integer
  truncation. These values apply once at creation; rest and reconnects do not compound the increase.
  [`Merc.fos`](../Scripts/Merc.fos) retains ownership through death and exempts these companions from
  ordinary dismissal/expiry. Scenario-only Charisma counting in `Merc`/`GlobalmapGroup` does not enlarge
  vehicle seating or exempt unrelated recruits.
- **Wave kill XP is shared**, not awarded to the last hitter. `Combat` suppresses its ordinary
  killer-only award for current-generation wave enemies; `ConfirmEnemyDeath` removes a registered
  enemy and distributes its `KillExperience` once. `SpawnEnemy` doubles that pool for every siege
  enemy, including the brothers, without changing shared prototypes or unrelated enemies.
  Every non-dead participant and scenario companion
  on that map gets an equal base share, including knocked-out defenders, regardless of damage dealt
  or whether a player, companion, Smitty or the environment finished the enemy. Integer remainder
  points go to randomly rotated recipients; sharing does not further multiply or discard the pool. More
  eligible defenders means less XP each. Dead/off-map actors and Smitty do not take a share. Ordinary
  experience setters, level-ups and personal XP perks still apply; unrelated kills are unchanged.
- The primary can press **Barter** in a living companion's dialog to transfer items freely in either
  direction, without payment or a matching offer. `Barter` sends inventory and equipped belongings,
  excluding hidden, gag and bad items; server checks retain item ownership/counts, distance, session,
  movement vetoes and both actors' final carry weight. Another participant cannot access the squad's
  inventory. `ClientItems.SetBarterPricing` now includes a server-supplied `companionTransfer` flag:
  the window shows zero prices and accepts take-only/give-only offers, but the server independently
  rechecks companion control. Matching client/server Scripts and Metadata packs must be deployed
  together. Ordinary NPC trading still requires payment.
- Following companions use [`MercFormation.fos`](../Scripts/MercFormation.fos), called from
  `Merc::MercIdleImpl`, instead of converging on the same master hex. Each reserves a distinct free,
  reachable target and walks to it with zero stop radius. At rest, target rings are 1–5 hexes from
  the master, with decreasing weight for farther rings, and selection minimizes pairwise separation
  beyond five hexes. During movement, both bounds tighten to three hexes. Existing targets remain
  stable until the master moves/stops, a target becomes occupied or its route fails. Reservations
  are transient `Critter.MercFormation*` properties. Combat and loot jobs take priority; waiting
  companions hold their orders. Obstacles and scattered/catching-up actors may temporarily prevent
  the preferred spacing; formation never teleports or displaces critters. Unrelated mercs retain
  ordinary `MercMasterDist` following.
- Ordinary and super stimpaks declare both `IsCanUse` (inventory self-use) and `IsCanUseOnSmth`
  (use on another critter). `CritterActions.UseItemOn` checks these capabilities before dispatching
  to `Main.CritterUseItem` -> `Drugs.UseDrug` / `UseDrugOn`; one unit is consumed through `ItemStacks`.
- Follow/wait/defend/assist and weapon-readiness commands use
  [`Dialogs/den_defense_leader.fodlg`](../Dialogs/den_defense_leader.fodlg) and the same `DenDefense`
  callbacks as ordinary companions. `Merc::SetBehaviorParameter` applies the leader's **final value**
  to every currently owned NPC mercenary, including those waiting on other maps; differing prior
  toggle states therefore converge. Only the changed setting is copied. Wait also drops each NPC's
  current plans. Commands through an ordinary companion remain individual. No new attack-target orders,
  automatic leader replacement or ongoing synchronization of later recruits is introduced.
  `Merc::ROLE_LEADER` reserves `NpcRole = 1000001` in the existing persistent role property; ordinary
  `Merc::SetMercParameter` / `WaitMaster` / `FollowMaster` dialogs use the same propagation path.
  Master identity, contract type and expiry are not squad behavior settings. New attempts create the
  leader prototype. Squad-command propagation itself needs no additional RemoteCall or removal/rename
  migration rule.
- Every companion dialog shows only the weapon order that changes its current readiness setting:
  **holster outside combat** when ready, or **keep weapons ready** when holstered. `CanHolsterWeapons`
  and `CanReadyWeapons` gate the answers; their plain `CanChangeWeaponReadiness` helper also guards
  the results. Combat's temporary weapon drawing does not change the saved preference. Leader wording
  addresses the whole squad; ordinary wording addresses only that NPC. The server-only persistent
  `Critter.MercWeaponReady` property (group `Mercs`, default false) stores this choice.
  `FavoriteItem::OnCritterIdle` delegates a hired NPC's main hand to `Merc::UpdateWeaponReady`, even
  while following/walking or with favorite-item maintenance disabled; other equipment slots retain
  their ordinary behavior. Outside combat, false holsters the main-hand item; true retains the held
  weapon or draws a usable last-used, favorite or inventory weapon. It does not create weapons or ammo.
  Dead/unconscious, unmapped and action-frozen companions are not re-equipped. Active attack plans and
  the battle timeout defer either order to the end of combat, leaving normal AI weapon selection intact.
  The existing second-based equipment timer reapplies the choice after combat; the command also
  attempts an immediate update. Old entities without the new property default to holstering.
- Hired NPC mercenaries (including the leader) try **First Aid on themselves below 50% HP** from
  `Merc::MercIdleImpl`, before loot/follow processing. They must be alive, on a local map, free to act
  under siege rules, not busy, off `TimeoutSkFirstAid`, and have the ordinary skill-use AP available.
  `CritterActions::UseSkill` runs the normal healing path; a successful heal sets the same
  `GameTime::FirstAidTimeout` used by players and emits one random localized line from
  `Texts/Text.{russ,engl}.fotxt` (`Merc/FirstAid1..9`). No drugs are required or consumed.
- **In combat only**, if First Aid is unavailable and HP is still **strictly below 50%**,
  `Merc::TrySelfStimpak` tries one owned ordinary stimpak, or a super stimpak only when no ordinary
  one is present. Combat means an active battle timeout or current attack plan. Outside combat,
  companions never spend stimpaks automatically. The same alive/local-map, siege and busy restrictions
  and ordinary item-use AP cost apply. `CritterActions::UseItemOn` -> `Main.CritterUseItem` ->
  `Drugs.UseDrug` consumes one unit through `ItemStacks` and schedules the existing drug effect
  (including super-stimpak aftereffects). It neither creates items nor changes the First Aid cooldown.
  The siege controller no longer separately spends companions' stimpaks; `PatternCommon` and
  `FighterPattern` also leave hired NPC self-healing to `Merc`. Smitty retains his existing healing path.
- **Combat healing takes priority over AI actions** through `Merc::TryCombatSelfHeal`, called by
  `NpcPlanes::OnCritterIdle` before its busy check/plan processing and by `Merc::MercIdleImpl`.
  Below 50% HP it selects First Aid whenever off cooldown and within the maximum AP budget,
  otherwise an available stimpak. An action costing more than maximum AP is not waited for. The merc
  stops movement, preserves the current plan/target, and waits for ordinary action completion and
  enough AP through `CritterActions::ProcessAp`; it does not attack, reload, change weapons or walk
  while healing is pending. Low AP never substitutes a stimpak for ready First Aid. Waiting consumes
  no item/AP and starts no skill cooldown; the selected action executes only after AP is available.
  If neither healing option is available, AI continues instead of waiting for a skill cooldown.
- Injured hired NPC mercenaries first try **Doctor on themselves outside combat** through
  `Merc::TrySelfDoctor`, before First Aid, loot or following. An active battle timeout or attack plan
  blocks Doctor; the ordinary alive/local-map, siege, busy, AP and `TimeoutSkDoctor` checks also apply.
  `CritterActions::UseSkill` retains the normal per-injury skill rolls. Every attempt, including a failed
  one, starts `GameTime::DoctorTimeout` and picks one localized `Merc/Doctor1..15` entry from
  `Texts/Text.{russ,engl}.fotxt`: ten spoken lines and five emotes. First Aid remains independent and
  can run on a later idle tick while Doctor is cooling down. No new properties or drug requirements.
- Every companion also offers a one-shot **collect loot** order through `DenDefense::CollectLoot`.
  [`MercLoot.fos`](../Scripts/MercLoot.fos) owns target selection, reservations and collection.
  The ordinary dialog orders only that NPC; the leader starts himself immediately and the other
  living owned mercenaries on this map at two-second intervals. The ten-hex search radius is fixed
  around the speaker's position at command time (shared by the leader's batch, not a drifting radius).
  Eligible targets are current-generation wave corpses and loose pickable map items; no living NPC,
  player/companion corpse, hidden/gag or no-loot item is harvested. Containers are carried intact.
  No value/category filter discards ordinary loot. Collected items stay in the collecting mercenary's
  inventory, using ordinary carry weight and stack splitting/merging; nothing is cloned or teleported.
  A corpse is looted in one **take-all** batch with one AP charge/action, not one item per idle tick.
  Before moving anything, the collector checks the total weight of all eligible corpse items with
  overflow-safe remaining-capacity arithmetic. If the whole batch would exceed carry weight, nothing
  is taken: the corpse is skipped for that order, its claim is released and the NPC says one random
  localized refusal from `Texts/Text.{russ,engl}.fotxt` (`MercLoot/Overweight1..4`). Other collectors may
  still take that corpse if they have capacity. Ordinary item-movement vetoes remain in force; ground
  pickups retain the existing ordinary partial-stack behavior.
  Exact target IDs are claimed in each collector's transient `MercLoot*` properties. Other collectors
  on the map skip a reserved corpse/item; a corpse stays reserved throughout its take-all batch. Workers
  wait for claimed targets to be released rather than racing for the same loot. Thirty seconds without
  a successful transfer releases/skips a stalled target for that order. Real NPC walk plans handle
  routes/doors; adjacent pickups spend ordinary AP and use normal item-movement checks/events.
  Collection ignores the general battle timeout and uses the same `MercStorage::IsEngaged` check as
  unloading: a valid outgoing/incoming NPC attack plan or the two-second guard after an actual attack
  involving this merc pauses collection and releases its claim. Attack events immediately stop the loot
  walk and release the target; combat plans and the collection order survive, so an unengaged merc can
  resume collecting during an ongoing wave. Follow/wait cancels collection; death, rest/failure,
  map departure, master departure or dismissal also release it. Completion resumes the previous
  follow/wait behavior. These jobs/reservations do not survive server restart; no new RemoteCall or
  serialized-property removal/rename migration is needed. Client gameplay acceptance remains manual.
- **Store loot at the base** uses `DenDefense::StoreLoot` -> [`MercStorage.fos`](../Scripts/MercStorage.fos).
  The ordinary dialog orders that companion; the leader orders himself and every living owned siege
  companion on the same map. It replaces a collection order (and collection replaces storage), without
  changing the saved follow/wait setting. Each NPC walks to the relevant workshop containers and puts
  only surplus belongings into them: upper-left melee weapons, upper-right ranged/throwing weapons and
  all ammunition, lower-left self-usable items (including drugs and medical kits), lower-right the rest.
  Containers/bags are carried intact rather than recursively unpacked. Equipped non-weapons and hidden,
  gag or no-loot items stay with the NPC. All four storage prototypes have localized labels and capacity
  `1000000`; no global tool-board/workbench prototype or unrelated map is changed.
  The personal reserve is **up to 30 ordinary stimpaks total**, **one usable weapon per tagged or trained
  combat skill**, and all carried ammunition compatible with those retained weapons. Trained means a
  value above the natural SPECIAL starting formula, adjusted for Good Natured; natural skill alone is
  not investment. Selection reuses `MainPlanes::CheckBattleWeapon` (animation, condition, arms, ammo),
  prefers the highest min+max damage for that skill and the main-hand weapon on a tie; duplicate weapons
  and excess thrown-weapon units are stored. Worn armor is not stripped.
  Storage ignores the general battle timeout: it pauses only for a valid outgoing attack plan, another
  living NPC's attack plan targeting the merc, or the two-second guard after an actual attack involving
  him. The attack events cover player fire too. Pausing removes only the storage walk, never combat
  plans; an unengaged merc can unload during an ongoing wave. Transfers require adjacency, ordinary AP,
  open-container/capacity/cycle checks and `ItemMovement` vetoes. Thirty seconds of stalled travel or a
  missing/full/closed crate ends the order without discarding items. Follow/wait, rest/failure, death,
  dismissal or map/master departure cancels it; completion resumes the previous follow/wait behavior.
  `MercStorage*` job/threat properties are transient; no new RemoteCall or property removal/rename is
  needed. Offline compilation/baking cannot establish in-client visibility and gameplay acceptance.
- Weapon stance comes from the actual main-hand item on the client (`Animation::GetStateAnim`).
  `TLA.fomain` sets `Critter.CritterSlotSendData = False True True True` in slot order
  Inventory/Main/Secondary/Armor: other critters' equipped items are synchronized both on entering
  view and after slot changes; backpack contents stay private. Disabling Main makes an armed
  companion appear unarmed between action animations, even when its model has the required frames.
  The shared `hmlmax` model supports Knife, Club and Hammer frames for the starting weapons.
  The narrow companion-only Club-to-Knife fallback in [`CritterAnims.fos`](../Scripts/CritterAnims.fos)
  remains for older tribal companions; it does not admit unsupported weapons for unrelated NPCs.
  Visual suitability still needs in-client acceptance.
- There are **no inventory snapshots or special retained-inventory resurrection**. Participant death
  spills ordinary, non-hidden, non-gag belongings onto the actual map. If no map is available, items
  remain until a safe mapped spill. Nested containers move intact; quantities use `ItemStacks`.
- Before deleting obsolete companions/enemies, reset spills their remaining belongings. It never
  purges map items. Smitty's remaining ordinary inventory goes into his crate before re-equipping him.
  At attempt setup, loose pickable non-container items within 12 hexes of `40:60` also move into
  that crate, without copying stacks. Hidden/gag system items are not exposed. Failed moves assert
  before actor deletion.
- Preserve ground loot, acquired rewards and living player inventories across attempts. There is no
  expiry, anti-farming rule, provenance ledger or arbitrary per-wave experience bonus. LAN SQLite
  preserves this state across process shutdown; profiles still using Memory do **not**.

## Todd and post-wave cleanup

Owner: `Scripts/DenCleanup.fos` (server cleanup and client truck label); authored inputs:
`Maps/sad_enter.fomap`, `Items/DenDefenseSupplies.foitem`, `Dialogs/den_defense_todd.fodlg`
and `Texts/Text.{russ,engl}.fotxt`.

- Todd is authored at `50:56`, behind the truck (`51:51`). Attempt setup reuses him (or creates him
  on older saved Sierra maps) and cancels his old walk without duplicating the NPC. Setup and each
  confirmed wave start uniformly choose an available hex within **hex distance 3 of the original
  `50:56`**, with a route to the workshop, and teleport him there with a random direction. This is
  not a drifting radius around his last position; occupied hexes never displace other critters.
- Once a wave is fully spawned and its registered enemies are all dead, Todd waits a uniformly
  selected **7–15 real seconds** before selecting a reachable corpse. The safe meaning of “all
  corpses” is **registered, lootable wave enemies of the current generation on this siege map**.
  Players, hired mercenaries, companions, unrelated inhabitants, other generations/maps and no-loot
  bodies are excluded. Ordinary collection respects `MercLoot` reservations both at selection and
  immediately before removal.
- Todd walks with ordinary door-aware NPC plans to corpse-looting distance (one hex plus both
  multihex radii), faces the body and removes it immediately. He then walks back behind the truck,
  preferably to `50:56`, otherwise to a reachable available hex within three hexes. At the exact
  unloading hex he faces the truck and waits **one real second**, credits one body to the truck,
  then may select the next corpse. His `nmlosr` model has no pickup/use frames, so the pause retains
  his standing sprite. A blocked return is retried without losing the carried body; it never
  teleports during ordinary collection.
- Removing a body has a **33%** chance of one uniformly selected bark from the ten requested lines
  (`100..109`, Russian and English). These use literal `Dialogs/den_defense_todd/100..109` keys,
  matching DialogBaker, without the obsolete numeric offset. A 250 ms idle timer handles real-time
  deadlines; busy/dialog time can delay completion. Unreachable bodies are ignored during ordinary
  collection; a stalled selected corpse is skipped after 30 seconds for that wave. Rest, failure or
  Todd's death cancels his transient walk but retains a pending carried body.
- `DenDefense::StartAfterRest` calls `DenCleanup::BeforeWave` only after defender/Smitty validation,
  before advancing the wave or spawning its first enemy. All remaining eligible bodies, including
  unreachable or reserved ones, are removed and counted individually. Only `MercLoot` targets for
  those bodies are released; collectors, their inventories and other jobs/items are not deleted.
  A previously carried body is credited once via its separate pending flag, not counted again in
  the map corpse scan. Repeated calls cannot recount already removed bodies or a cleared pending
  flag. A canceled rest does not run this cleanup.
- Remaining ordinary corpse items are spilled intact onto the corpse's hex before body deletion,
  through `DenDefense.SpillLoot` and `ItemStacks`; failed moves assert instead of deleting unclaimed
  loot. Todd does not move these items into Smitty's crate. The crate at `35:60` still holds workshop
  supplies gathered during setup and Smitty's remaining inventory before re-equipping him. Players
  can take these supplies normally, individually or with Take All (subject to ordinary carry-weight
  and action checks). Setup uses `Tla::RootContainerStack()` for new placements. An empty stack filter
  enumerates every section of the container, so individual transfers accept the same sections as the
  displayed snapshot and Take All, including legacy numeric-`0` supplies; no save wipe or item migration
  is needed. The crate inherits the wooden-crate presentation with an enlarged capacity; old saves
  without it create exactly one crate on demand.
- `Map.DenCleanupCorpseCount` is **Common, PublicSync, Persistent int64**. The CLIENT branch draws
  `DenCleanup/TruckCorpses` above the truck on every interface frame: **“В грузовике X трупов”** /
  **“Bodies in the truck: X”**, including zero. The above-truck anchor is calibrated to the original
  label position at the client's displayed **60%** zoom (`SpritesZoom = 100 / 60`), then lowered by
  78 map pixels (about 1.3 standing human sprite heights). The entire vertical offset is converted
  from map to screen coordinates with the current `Map.SpritesZoom`, not just the added lowering;
  changing Z+scroll therefore preserves its map-relative position. Text size remains unchanged.
  It is independent of NPC names, speech expiry and `MapText`; map entry/reconnect and count changes
  use ordinary property synchronization.
  `Map.DenCleanupCarrying` is server-only Persistent and denotes exactly one already removed body
  awaiting unloading. `DeliverCarried` clears it when credited, including on attempt setup and
  server-start reset; the cumulative count itself is never reset by a new wave or generation.
  A saved pending delivery is finalized at restart because the siege itself intentionally resets.
  Existing saves without these properties start at zero; previously deleted bodies are not reconstructed.
- Persistence requires the same saved Sierra map in a **persistent storage provider**, such as the
  LAN profile's SQLite, with its storage retained across graceful server restarts. **Memory cannot
  preserve either the count or pending delivery across process restart**, even with `Persistent`
  metadata; selecting Memory is not a durable mode. No config/provider switch is made here.
  Transient `DenCleanup*` critter properties hold the walk/return hex, start/deadlines, generation
  and skips. This is an additive property contract, with no removal/rename migration or RemoteCall.
  Rebake and deploy matching client/server Scripts, Metadata and both localized Texts together.

## Global rest and time domains

A rest request persists its next 10:00 target and phase. If that morning passes during the initial fade
or a server pause, it is reselected before the first clock advance. The target is then locked before
calling the setter, so a resumed/partly completed rest cannot advance by an extra day. Every online
client, even outside Sierra, receives phase/remaining-time state. A mid-rest entrant joins that same
transition; disconnect/failure/reset clears its local blackout. Both fade phases block client intent
and authoritative server actions; black pixels are not authorization. Rest and failure also stop any
global-map group containing an action-frozen member, regardless of its leader. `GlobalmapGroup` clears
the route and pending encounter, synchronizes the stopped state and blocks autonomous travel/encounter
processing before callbacks, fuel consumption or encounter selection. Unfreezing does not resume the
old route or pending encounter; travel requires a new command.

[`MapTime.fos`](../Scripts/MapTime.fos) owns scaled-calendar conversion using `WorldTime.StartYear`,
October 30 and `WorldTime.Multiplier`. [`GameTime.fos`](../Scripts/GameTime.fos) uses it for date readers,
formatting and future-date calculations. `AdvanceCalendarTo` is forward-only and refreshes date fields;
client `MapTime`/`DayColor` follow synchronized time for lighting.

- Synchronized deadlines may expire across the global jump: skill/battle/healing, bag or replication
  deadlines retain their existing comparison semantics.
- Frame-time callbacks retain real delays: wave cadence, NPC AI, fades and previously scheduled
  patrol/appointment callbacks are **not** fast-forwarded or globally rescheduled.
- Subsequent calendar appointments use the corrected conversion. Do not promise replay of every
  skipped appointment, introduce an offset clock, or rewind world time when resetting a run.
- Sleep restores HP/AP of living participants, companions and Smitty, including unconscious living
  defenders; it does not revive dead companions or replenish inventory.

## Combat and completion

The city retaliation mechanism in `TownSupply` is disabled globally: neither new nor pending calls
spawn bounty hunters, and legacy guards are removed on initialization/idle. This does not remove
unrelated caravan/encounter NPCs that happen to use bounty-hunter prototypes.

| Wave | Composition and equipment |
| --- | --- |
| 1 | 50 groups of 1–3 attackers (uniform size; 50–150 total, mean 100), cycling Prostitute, Drunkard and Madman; original Den models/dialogs, one-shot stock bags 26/11, plus alcohol/rocks for drunkards |
| 2 | 30 tribals in ten simultaneous groups of three: exactly six spears, twenty-four unarmed; meat/flowers/healing powder |
| 3 | 42 unarmed townspeople and Chuck/Buck Dunton; each brother has brass knuckles and a `10mm_pistol` with **one loaded + four spare rounds**; money/alcohol/miscellaneous loot |

New wave-2/3 enemies, including the Dunton brothers, start at 50% of their normal maximum HP.
Wave 1 halves that already reduced maximum once more (50% of its previous balance); both maximum
and current HP are reduced to 25% of the prototype maximum, rounding down with a minimum of one.
`Combat.CombatAttack` passes the full unarmed/melee
damage range (weapon plus melee bonus) through `DenDefense.ScaleEnemyMeleeDamage`, keeping 34% before
critical and armor calculations. Ranged damage, Smitty, players and NPCs outside this scenario are
unchanged. Health and melee scaling round down to whole points, with a minimum of one. Existing
spawned actors keep their HP until a new attempt creates replacements; the melee modifier is applied
at attack time. Wave 1 reuses `Loser2`/`Loser` models (`nftrmp`/`nmlosr`) and their existing animation
and sound selection, with `all_prostitute`, `all_alco` and `den_crazy_man` names/descriptions/dialog
lines. Its stock bags supply knives/switchblades/shivs, jet, flowers and caps; drunkards additionally
carry beer/booze and rocks. Bags are filled once, not refreshed during combat; city quest state is
not copied to these attackers.

[`Scripts/DenWander.fos`](../Scripts/DenWander.fos) owns wave-1 distractions. One uniform roll per
new enemy assigns mutually exclusive modes: **30% lost**, **10% object fascination**, **10% rock
thrower** (only a model with throwing animation), **10% brawl initiator**, **5% circling runner**
(only a model with running animation), otherwise normal siege AI. An ineligible animation roll
falls back to normal AI, not another distraction. A brawl initiator can forcibly recruit an unpaired
living wave enemy, overriding that enemy's earlier mode; these are initiation probabilities, not
final population quotas. Waves 2/3 are unchanged.

All random destinations use the expanded central approach field (`x=49..99`, `y=33..128`): between
the workshop and left bunker, from the outer entrance approach to the base facade, excluding the
lateral yards, buildings and base interior. Free hexes require the actual NPC's door-aware path
and exact stopping position; unreachable/occupied goals are reconsidered every five real seconds.
Unavailable destinations wait without teleporting or enabling ordinary aggression. The small
player/companion spawn yard is independent of these bounds.

- **Lost:** walks to a random destination, retaliating only against its recorded direct aggressors
  (including missed attacks). A high-priority attacked handler records them before `Main` admits
  retaliation. With no usable aggressor it resumes walking. Exact arrival, from its own walk-end
  event or controller fallback, says one `DenWander/Arrival1..9` line and restores ordinary AI.
- **Fascinated:** approaches a reachable free hex beside an authored turret, barrel, truck, tire or
  tree, then stands facing its anchor for **60–180 real seconds**. During this mode it does not
  attack. Every independently randomized **30–60 seconds** it says a `Stare1..14` line, including the
  three emotes; the first due line is processed before expiry, guaranteeing at least one. Afterwards
  ordinary AI resumes.
- **Rock thrower:** `nftrmp` has `NFTRMPAS`; `nmlosr` lacks the throw animation, so it is excluded.
  A finite **3–6 rock** ground stack is seeded once on a reachable field hex. The NPC walks to pick
  up real ground rocks, throws them using their primary attack and recovers thrown rocks through
  normal pickup. No inventory refill is generated during combat. Only a target already adjacent
  permits an unarmed attack; without rocks it never chases a distant target into melee.
- **Brawl:** both members are forced to adjacent reachable random field destinations and wait for
  each other before fighting. Only this mutually linked pair bypasses same-side attack/damage
  rejection, and `MainPlanes` forces unarmed primary attacks. Each member speaks a `Brawl1..13` line
  every **30–60 real seconds** after assembly. Below **10% maximum HP**, a member stops attacking
  and repeatedly chooses distant escape destinations; its opponent pursues, using running only
  when supported. If both are below the threshold, both flee. Losing the partner restores normal
  AI; an unpaired initiator waits for later batches, but gives up when spawning is finished and
  no partner is available. Damage/HP balance and lethal-hit rules are unchanged.
- **Runner:** randomly chooses a visible reachable participant, companion or Smitty admitted by
  `DenDefense.CanSelectTarget` (the same faction/map/workshop rules as attack selection, without
  granting permission to attack). It retains that target while valid and runs successive arcs in
  one randomly chosen direction, normally **5–9 hexes** away. Every **20–40 real seconds** it tries
  to approach to **two hexes**, then immediately retreats towards an **8–12 hex** outer destination;
  after opening an eight-hex gap it resumes circling. Distances include both actors' multihex radii.
  `OnCritterMoved` initiates the retreat on the actual two-hex crossing, without waiting for the
  one-second controller tick. Moving targets, blocked endpoints and invalid targets trigger
  replanning/reselection. Destinations stay in the central field; if the inner ring is inaccessible,
  it skips that approach and continues outside, without entering melee or choosing allies/neutral
  NPCs instead. With no suitable target it waits and retries. Eligibility uses `CritterTypes.CanRun`
  and `IsNoRun`; no missing run animation is requested. Runners still never attack.

Both languages define all `DenWander` lines in `Texts/Text.{russ,engl}.fotxt`. Plan admission,
actual attacks and damage use the same `DenDefense.CanAttack` gate, with the narrow reciprocal
brawl-pair exception. Mode, goals, partner, runner target/phase/direction, aggressor IDs and deadlines
are transient server-only critter properties; no public property, RemoteCall or rename/removal
migration is added.

The first batch is admitted when the 10:00 rest transition finishes. Wave 1 admits **1–3 enemies
at once**, each size equally likely, over the original **50 batches**. Sizes are rolled once at wave
start into the server-only persistent `DenDefenseSpawnGroups` queue; their sum is the exact enemy
completion total, and blocked batches retain their selected size. A new attempt/server-start reset
clears the queue. Wave 3 still admits one enemy per batch. Both waves keep the uniform
**3000–9000 real ms** delay (mean **6000 ms**), independent of group size. Wave 2 admits **three
enemies at once**, with its unchanged uniform **9000–27000 real ms** delay between groups (mean
**18000 ms**). A separate generation-guarded `SpawnTick` map timer schedules once per batch; the
one-second combat controller does not quantize the interval. Each wave reinstalls the controller
and replaces its spawn timer. Batches uniformly sample distinct free/reachable hexes across the
entire authored global ExitGrid, its anchor and `MultihexMesh` (`x=67..95`, `y=129..139`, tapering
at the south end). The separate grid into `sad_level1` is excluded. Every position must be available
before any group member is created, otherwise the whole group waits for the next randomized retry.
Counts and equipment still advance per enemy, not per batch. No existing property is renamed or removed.
Every candidate needs a route toward the workshop through openable, unlocked doors. Blocked exits
never force an unrelated spawn or unlocking. If all candidates are blocked, the same
cursor retries with another randomized delay and logs periodically;
this is not a kill. Outside the special wave-1 behaviors, visible reachable participants/companions
distract attackers; otherwise they walk toward living Smitty.
Stale/invisible distractions return to that objective. `MainPlanes.CanOpenDoor` is shared by target
reachability, actual NPC movement and `MainPlanes` door reactions: AI permission is required, and
non-openable or locked doors without a key remain blocked (broken locks need no key). Movement
plans admit those doors during path search, then interpret the engine's `HexBusy` door collision as
an opening subplan before resuming the parent task; they do not force doors open at spawn time.
Scenario attackers do not flee because of the ordinary minimum-hit-chance heuristic.

The shared [`NpcPositioning`](../Scripts/NpcPositioning.fos) policy lets NPCs pass through occupied
hexes but forbids an NPC attack while its footprint overlaps another non-dead actor. `CritterActions`
checks before spending attack AP; `Combat` also guards direct scripted attacks. When a combat plan
needs to approach, clear its firing line or leave an overlap, it chooses an exact reachable free
position on the nearest available ring around its target, respecting minimum weapon distance and
ranged line of fire. Other actors' current positions and active route endpoints reserve space;
corpses do not. If no attack position is available, it looks for waiting space on the next eight
rings and retries with the same target. If those rings are full, an overlapping NPC also tries to
step aside locally. A free valid firing position is retained rather than forcing every
ranged NPC into melee. Walk plans also choose distinct endpoints; an occupied exact goal can be
waited for nearby without reporting successful arrival. Idle NPCs without a plan leave overlaps,
except immobile/talking actors. No teleporting, engine collision changes or persistent reservation
properties are introduced; player-controlled attacks are unchanged.

Smitty stays inside an explicit workshop hex set (door `40:57` excluded), cannot pursue or be knocked
outside, and uses a model-compatible 10mm pistol. Admission **and actual attack execution** reject
outside targets, including a target that leaves after acquisition. `Combat.ApplyDamage` checks
`DenDefense.CanDamage` for each actual recipient before attack events or damage/status changes, so
miss redirection, bursts and explosions cannot bypass scenario faction/workshop restrictions. Explicit
self-damage and environmental damage without an attacker retain their existing behavior. Ordinary
city faction help is not recruited into scenario fights. Smitty remembers player/companion aggression
and retaliates while the aggressor is inside the workshop; the controller retains those targets as
well as wave enemies. A new generation clears his aggression memory.

[`MsgStr.fos`](../Scripts/MsgStr.fos) resolves NPC names from `[Dialogs, dialogId, Name]`, falling back
to `[Critters, protoId]`, not numeric hash sums. Named Den companions instead resolve their personal
name from `TextArgs`; client setters refresh names when `DialogId` or `TextArgs` changes. Combat
messages consume that displayed name. Both localized Smitty
names remain authored in his dialog. [`Animation.fos`](../Scripts/Animation.fos) separates one-shot
Fallout death playback from a final-frame corpse idle, avoiding repeated falling/twitching without
reviving or moving the corpse.

`NpcPlanes` and `ItemMovement` prevent synthetic ammo refill for attackers/tagged brother pistols.
Normal weapon selection rejects an exhausted pistol and falls back to knuckles/unarmed. Both brothers
use Klamath's authored Dunton prototype `MasterTrader` / model `nmpeas`, with the names Chuck and Buck
from `Dialogs/klam_danton.fodlg`. That model has pistol frames (`E`), not rifle frames (`J`), so their
former rifles are replaced by pistols without increasing the ammunition supply. Historical dialog
resource IDs `den_defense_bobby` / `den_defense_billy` remain unchanged. `Combat` emits their additional randomized localized threats at
actual firearm ammo expenditure. Wave actors select existing voiced AI packets (45/11 for wave 1,
1 for tribals, 13/14 for wave 3), rather than silent packet 25; the shared bark tables and ordinary
combat-text path remain unchanged.

Only confirmed deaths of current-generation enemies on the scenario map retire outstanding IDs.
Incomplete spawning, map loss, disappearance, escape and unrelated deaths do not clear a wave; a lost
outstanding actor fails diagnostically instead of awarding a false victory.

## Player autoattack

`Ctrl+A` on the Game screen asks `DenAutoAttack` to select a visible, usable target from
**current-generation Den defense enemies only**. Eligible enemies with `CurrentHp > 0` take priority
over those with zero/negative HP; within each group the nearest wins, with critter ID breaking ties.
Living enemies with nonpositive HP remain fallback targets if no positive-HP enemy is usable.
Server checks require a living scenario participant on the defense map. Smitty, companions, players,
ordinary inhabitants and enemies outside this scenario are never automatic targets. There is no
monster/faction heuristic or general-purpose combat mode.

The active main-hand weapon/use is retained, including the selected empty-hand attack. The command
executes one unaimed weapon use without changing the saved aiming mode. Unarmed/melee selects a
reachable, unoccupied attack hex satisfying both weapon range and a clear trace, then approaches that
exact hex with the existing action queue and pathfinder. A reachable radius behind a wall is not sufficient;
enemies without a usable position are skipped. The approach retains a usable destination while moving
and reevaluates it if it becomes blocked or no longer has range/trace to the target. Ranged/throwing
only selects an in-range, unobstructed target and never approaches. Holding the key does not repeat
attacks in this one-shot mode; there is no automatic reload or retarget.
If no eligible target exists, unrelated queued actions are left alone. Weapon changes, lost/dead targets,
failed paths and stale target-selection replies cancel the automatic command. The server rechecks
scenario eligibility, range and trace before the actual attack. The client also rechecks range/trace
for melee and ranged attacks before weapon animation, AP prediction and dispatch.
Ordinary `A`, aimed mouse attacks and damage rules are unchanged; bursts/explosions still follow the
existing collateral-damage rules.

`Ctrl+Shift+A` toggles a client-session repeat mode, shown by a green **Autoattack / Автоатака**
status tab using the same HUD presentation as Sneak. With the Game screen active, it requests the same
one-shot command once enough current AP is available for the active main-hand weapon/use (including
empty-hand attacks), calculating the cost without aiming. Every repeat selects a target anew under
the same Den-only rules. Changing weapon/use affects the next request; the saved aiming mode is untouched.
The mode waits while knocked out, moving, busy, performing queued actions, using other screens or the
console, or while the defense is resting/frozen. Knockout keeps the mode and its HUD tab enabled;
automatic target selection and attacks resume after standing up, once AP and busy-state checks allow.
It does not reload, attack with empty/broken weapons or repeatedly issue unusable attacks with injured
arms. Target requests are spaced by at least 500 ms; an unanswered
request expires after 2 seconds. Manual actions invalidate pending selection replies and take priority.
Toggling off also cancels queued automatic attacks/approaches. Map load, disconnect, death or changing
the controlled critter disables the mode; it is not saved in user options.

Owners: `Scripts/InputHandler.fos` (hotkeys), `Scripts/DenAutoAttack.fos` (repeat state, selection RPC
and execution guard), `Scripts/ChosenActions.fos` (queue), `Scripts/ChosenTabs.fos` (HUD),
`Scripts/MsgStr.fos` and `Texts/Game.{engl,russ}.fotxt` (localized status title),
`Scripts/Tla.fos` (client action codes).
Role/generation properties remain server-only;
client/server outputs must be rebaked/rebuilt together for the added RemoteCalls.

## Failure presentation and verification limits

All online participants are action-frozen on actual failure; Smitty's death alone is not failure.
Primary death leaves the body visible on the map: no fade, movie, black failure overlay or music
replacement. Participant inventory still spills at the death hex. The next confirmed login after
primary death creates a fresh attempt and revives the returning participant without restoring lost
items. The old failure-presentation property declarations remain reserved for metadata/save
compatibility; no movie callback is scheduled. Rest retains its separate dawn fade.

Use the existing script compiler, resource baker, nullable/quality validators and affected native
build targets from [AGENTS.md](../AGENTS.md#build-and-verify). Technical checks do not establish live
routes, door boundaries, balance, corpse targeting, death presentation or animation fallback. Gameplay
acceptance belongs to the user; do not substitute a client/AiControl session without their instruction.

Before deploying matching baked resources/binaries, protect any older Memory server: shutdown loses
its world/progress and requires explicit approval. SQLite survives restarts, but the siege deliberately
does not resume. Keep its external storage directory and stop gracefully before backup/deployment;
publication alone does not deploy the change. Use isolated bake output for non-gameplay verification
rather than replacing resources of a valuable live session.

Update this page when login ownership, stage/death policy, item claims/spills, time domains, map hex
sets, wave equipment, presentation or the script/content boundary changes. See [Systems.md](Systems.md)
for adjacent owners and [README.md](README.md) for navigation.
