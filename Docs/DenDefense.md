# Shared Den defense

## Scope and entrypoints

Owner: [`Scripts/DenDefense.fos`](../Scripts/DenDefense.fos), server-authoritative scenario with
client-only rest fades and failure input blocking. It uses the existing shared `den` location/map and its
original Smitty, not a private base or a replacement city. Ordinary inhabitants and map items are not
reset, except that Smitty's workshop door at `303:147` starts open and is reopened for each new attempt.
Smitty's former trader/dialog behavior is intentionally replaced for this scenario.

- [`PlayerRegistration.fos`](../Scripts/PlayerRegistration.fos) calls `OnSuccessfulLogin` with the
  **returned** `Player`, after successful registration/existing-account login and outside rollback
  catches. `Main::OnPlayerEnter` can still be rejected and must not claim a run, kit or squad.
- [`Main.fos`](../Scripts/Main.fos) creates Den in a fresh world and routes initial placement to the
  base. `DenDefense::ResumeController` can create a missing Den in an existing world; it does not
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
On every start, `ResumeController` explicitly makes the Den location persistent, including its map,
inhabitants and items: engine-created locations are transient until opted in. Player critters are
already explicitly persistent in `PlayerRegistration`. Other transient world locations are not opted
in by this scenario. It then cancels the map controller and all rest callbacks, stops scenario
actions, clears wave/spawn/rest progress and marks the attempt Failed. It retains generation and actor
IDs for `StartAttempt` to clean up old companions/enemies on the next confirmed login; that login
increments the generation, so prior kit/reward claims do not apply to the new attempt. No account,
player character or ordinary town content is deleted, and the world is not regenerated.
No existing serialized identifier is removed/renamed and no migration rule/engine revision bump is added.
The `LAN` profile uses persistent SQLite outside the package; see [ClientUpdates.md](ClientUpdates.md)
for storage and restart instructions. Other profiles remain Memory unless overridden.

## Supplies, companions and ground loot

- Briefing gives each participant **one crowbar, one leather jacket and two stimpaks once per generation**. Joining alone
  does not grant them. Any briefed participant may request eligible rest; every participant may claim
  their final reward. Only the primary may issue follow/wait/defend/assist orders.
- Participant and companion placement searches nearby hex rings, rejects occupied hexes and requires
  a route to the workshop through openable unlocked doors. A locally free pocket between cars is not
  a valid spawn; failure to find a connected hex aborts placement instead of trapping an actor.
- The four-person squad has exactly one leader (the first companion), using
  [`Critters/DenDefenseLeader.focr`](../Critters/DenDefenseLeader.focr): the bald leather-armor
  model `hmlthr`, inheriting the ordinary companion's stats and starting sledgehammer.
  The other three use [`Critters/DenDefenseCompanion.focr`](../Critters/DenDefenseCompanion.focr):
  the long-haired leather-jacket model `hmlmax`. All start at level 3 / 3,000 XP and SPECIAL `8/4/5/4/5/10/4`
  (40 points, the five remaining stats split as evenly as integers allow). Tagged skills are Melee
  Weapons, First Aid and Small Guns. All affordable level-up points go through
  `Parameters::SpendSkillPoints` into Melee Weapons using the same tag bonus and cost bands as players:
  30 earned points yield 118% melee, with one point left because the next increase costs two;
  First Aid starts at 38% and Small Guns at 65%.
  Each starts with **five consumable stimpaks** and exactly one equipped weapon, in squad order:
  **sledgehammer, combat knife, crowbar, club**. Smitty does not equip or resupply companions.
  Maximum/starting HP are **103/91/103/91**, another 50% over the previous 69/61/69/61 with integer
  truncation. These values apply once at creation; rest and reconnects do not compound the increase.
  [`Merc.fos`](../Scripts/Merc.fos) retains ownership through death and exempts these companions from
  ordinary dismissal/expiry. Scenario-only Charisma counting in `Merc`/`GlobalmapGroup` does not enlarge
  vehicle seating or exempt unrelated recruits.
- **Wave kill XP is shared**, not awarded to the last hitter. `Combat` suppresses its ordinary
  killer-only award for current-generation wave enemies; `ConfirmEnemyDeath` removes a registered
  enemy and distributes its `KillExperience` once. Every non-dead participant and scenario companion
  on that map gets an equal base share, including knocked-out defenders, regardless of damage dealt
  or whether a player, companion, Smitty or the environment finished the enemy. Integer remainder
  points go to randomly rotated recipients; the base pool is not multiplied or discarded. More
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
  leader prototype. No new RemoteCall or removal/rename migration rule is required.
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
  Combat pauses collection and releases its claim. Follow/wait cancels collection; death, rest/failure,
  map departure, master departure or dismissal also release it. Completion resumes the previous
  follow/wait behavior. These jobs/reservations do not survive server restart; no new RemoteCall or
  serialized-property removal/rename migration is needed. Client gameplay acceptance remains manual.
- Both `hmlthr` and `hmlmax` support Knife, Club and Hammer frames for the starting weapons.
  The narrow companion-only Club-to-Knife fallback in [`CritterAnims.fos`](../Scripts/CritterAnims.fos)
  remains for older tribal companions; it does not admit unsupported weapons for unrelated NPCs.
  Visual suitability still needs in-client acceptance.
- There are **no inventory snapshots or special retained-inventory resurrection**. Participant death
  spills ordinary, non-hidden, non-gag belongings onto the actual map. If no map is available, items
  remain until a safe mapped spill. Nested containers move intact; quantities use `ItemStacks`.
- Before deleting obsolete companions/enemies, reset spills their remaining belongings. It never
  purges map items. Smitty's remaining ordinary inventory is spilled before re-equipping him. Hidden/
  gag system items are not exposed. Failed moves assert before actor deletion.
- Preserve ground loot, acquired rewards and living player inventories across attempts. There is no
  expiry, anti-farming rule, provenance ledger or arbitrary per-wave experience bonus. LAN SQLite
  preserves this state across process shutdown; profiles still using Memory do **not**.

## Global rest and time domains

A rest request persists its next 10:00 target and phase. If that morning passes during the initial fade
or a server pause, it is reselected before the first clock advance. The target is then locked before
calling the setter, so a resumed/partly completed rest cannot advance by an extra day. Every online
client, even outside Den, receives phase/remaining-time state. A mid-rest entrant joins that same
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
| 1 | 25 attackers cycling Prostitute, Drunkard and Madman; original Den models/dialogs, one-shot stock bags 26/11, plus alcohol/rocks for drunkards |
| 2 | 15 tribals: exactly three spears, twelve unarmed; meat/flowers/healing powder |
| 3 | 20 unarmed townspeople and Bobby/Billy Dunton; each brother has brass knuckles and a single-shot `springer_rifle` with **one loaded + four spare rounds**; money/alcohol/miscellaneous loot |

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

The first enemy is admitted when the 10:00 rest transition finishes; subsequent admissions are spaced by
three real seconds. Each wave reinstalls the map controller. Spawn selection rotates explicit indoor
candidate sets in the nearby western houses and checks a route toward the workshop through openable,
unlocked doors. An occupied/unreachable room falls through to the other rooms, never to street hexes
or forced unlocking. If all candidates are blocked, the same cursor retries and logs periodically;
this is not a kill. Visible reachable participants/companions distract attackers; otherwise they
walk toward living Smitty.
Stale/invisible distractions return to that objective. `MainPlanes.CanOpenDoor` is shared by target
reachability, actual NPC movement and `MainPlanes` door reactions: AI permission is required, and
non-openable or locked doors without a key remain blocked (broken locks need no key). Movement
plans admit those doors during path search, then interpret the engine's `HexBusy` door collision as
an opening subplan before resuming the parent task; they do not force doors open at spawn time.
Scenario attackers do not flee because of the ordinary minimum-hit-chance heuristic.

Smitty stays inside an explicit workshop hex set (door `303:147` excluded), cannot pursue or be knocked
outside, and uses a model-compatible 10mm pistol. Admission **and actual attack execution** reject
outside targets, including a target that leaves after acquisition. `Combat.ApplyDamage` checks
`DenDefense.CanDamage` for each actual recipient before attack events or damage/status changes, so
miss redirection, bursts and explosions cannot bypass scenario faction/workshop restrictions. Explicit
self-damage and environmental damage without an attacker retain their existing behavior. Ordinary
city faction help is not recruited into scenario fights. Smitty remembers player/companion aggression
and retaliates while the aggressor is inside the workshop; the controller retains those targets as
well as wave enemies. A new generation clears his aggression memory.

[`MsgStr.fos`](../Scripts/MsgStr.fos) resolves NPC names from `[Dialogs, dialogId, Name]`, falling back
to `[Critters, protoId]`, not numeric hash sums. Its client setter refreshes the displayed name when a
visible NPC changes `DialogId`; combat messages consume that displayed name. Both localized Smitty
names remain authored in his dialog. [`Animation.fos`](../Scripts/Animation.fos) separates one-shot
Fallout death playback from a final-frame corpse idle, avoiding repeated falling/twitching without
reviving or moving the corpse.

`NpcPlanes` and `ItemMovement` prevent synthetic ammo refill for attackers/tagged brother rifles.
Normal weapon selection rejects an exhausted rifle and falls back to knuckles/unarmed. Both brothers
use a model supporting shotgun frames. `Combat` emits their additional randomized localized threats at
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
The mode waits while moving, busy, performing queued actions, using other screens or the console, or
while the defense is resting/frozen. It does not reload, attack with empty/broken weapons or repeatedly
issue unusable attacks with injured arms. Target requests are spaced by at least 500 ms; an unanswered
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
