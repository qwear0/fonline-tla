# Shared Den defense

## Scope and entrypoints

Owner: [`Scripts/DenDefense.fos`](../Scripts/DenDefense.fos), server-authoritative scenario with
client-only rest fades and failure input blocking. It uses the existing shared `den` location/map and its
original Smitty, not a private base or a replacement city. Ordinary inhabitants and map items are not
reset. Smitty's former trader/dialog behavior is intentionally replaced for this scenario.

- [`PlayerRegistration.fos`](../Scripts/PlayerRegistration.fos) calls `OnSuccessfulLogin` with the
  **returned** `Player`, after successful registration/existing-account login and outside rollback
  catches. `Main::OnPlayerEnter` can still be rejected and must not claim a run, kit or squad.
- [`Main.fos`](../Scripts/Main.fos) creates Den in a fresh world and routes initial placement to the
  base. `DenDefense::ResumeController` can create a missing Den in an existing world; it does not
  regenerate that world or select a new primary on load.
- [`den_defense_smitty.fodlg`](../Dialogs/den_defense_smitty.fodlg) owns briefing, supplies, rest and
  final claims. [`den_defense_companion.fodlg`](../Dialogs/den_defense_companion.fodlg) owns orders.
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
stay dead until reset. Disconnecting, ordinary Active/Victory login and loading the server do not
select another primary. A replacement
generation after primary death still sets up Smitty and the squad anew.

`Game` properties persist generation, IDs, stage, wave cursor, outstanding/history IDs and rest target/
phase. Critter markers persist participation and once-per-generation kit/reward claims. Frame-time
deadlines and client presentation are transient. Controller/rest/death callbacks validate generation;
resume stops/replaces the map controller rather than multiplying it. An interrupted Starting setup
becomes Failed on load, without automatically starting another attempt. These are additive properties;
no existing serialized identifier is removed/renamed and no migration rule/engine revision bump is added.

## Supplies, companions and ground loot

- Briefing gives each participant **one crowbar and two stimpaks once per generation**. Joining alone
  does not grant them. Any briefed participant may request eligible rest; every participant may claim
  their final reward. Only the primary may issue follow/wait/defend/assist orders.
- Participant and companion placement searches nearby hex rings, rejects occupied hexes and requires
  a route to the workshop through openable unlocked doors. A locally free pocket between cars is not
  a valid spawn; failure to find a connected hex aborts placement instead of trapping an actor.
- Each of the four companions starts with one crowbar and two actual consumable stimpaks.
  Their maximum and starting HP are 150% of the normal prototype-derived maximum, applied once at
  creation; rest and reconnects do not compound the increase.
  [`Merc.fos`](../Scripts/Merc.fos) retains ownership through death and exempts these companions from
  ordinary dismissal/expiry. Scenario-only Charisma counting in `Merc`/`GlobalmapGroup` does not enlarge
  vehicle seating or exempt unrelated recruits.
- Ordinary and super stimpaks declare both `IsCanUse` (inventory self-use) and `IsCanUseOnSmth`
  (use on another critter). `CritterActions.UseItemOn` checks these capabilities before dispatching
  to `Main.CritterUseItem` -> `Drugs.UseDrug` / `UseDrugOn`; one unit is consumed through `ItemStacks`.
- The tribal models lack Club frames. The server's narrow companion exception in
  [`CritterAnims.fos`](../Scripts/CritterAnims.fos) admits the existing `Animation` Club-to-Knife fallback
  for these companions; it does not allow other NPCs to use unsupported weapons. Visual suitability
  still needs in-client acceptance.
- There are **no inventory snapshots or special retained-inventory resurrection**. Participant death
  spills ordinary, non-hidden, non-gag belongings onto the actual map. If no map is available, items
  remain until a safe mapped spill. Nested containers move intact; quantities use `ItemStacks`.
- Before deleting obsolete companions/enemies, reset spills their remaining belongings. It never
  purges map items. Smitty's remaining ordinary inventory is spilled before re-equipping him. Hidden/
  gag system items are not exposed. Failed moves assert before actor deletion.
- Preserve ground loot, acquired rewards and living player inventories across attempts. There is no
  expiry, anti-farming rule, provenance ledger or arbitrary per-wave experience bonus. Memory storage
  does **not** preserve this state across process shutdown.

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

Before deploying matching baked resources/binaries, protect any running Memory server: shutdown loses
its world/progress, and publication alone does not deploy the change. Use isolated bake output for
non-gameplay verification rather than replacing resources of a valuable live session.

Update this page when login ownership, stage/death policy, item claims/spills, time domains, map hex
sets, wave equipment, presentation or the script/content boundary changes. See [Systems.md](Systems.md)
for adjacent owners and [README.md](README.md) for navigation.
