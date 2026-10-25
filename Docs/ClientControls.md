# Client controls and convenience actions

Client-side controls use the ordinary gameplay path, not privileged healing or item-grant commands.
`Scripts/InputHandler.fos` owns hotkey dispatch; feature modules own their state and eligibility.
See [Systems.md](Systems.md) for the ownership map and [Build.md](Build.md) for verification policy.

## Controls

| Input / feature | Behavior | Owner |
| --- | --- | --- |
| Map zoom | Keep the selected zoom across map loads and client restarts; save the qualified `ClientZoom.Level` option without dropping other local options. | `ClientZoom`, `UserOptions`; [ClientZoom.md](ClientZoom.md) |
| **F10** | Toggle overhead HP for players and hired mercs; initially off, session-only; ordinary NPCs are not included. | `ClientHealthDisplay`; [ClientHealthDisplay.md](ClientHealthDisplay.md) |
| **Alt+F1** | Use First Aid (Sanitary) on the chosen character. | `SelfCare`, `ChosenActions` |
| **Alt+F2** | Use Doctor on the chosen character's injuries. | `SelfCare`, `ChosenActions` |
| **Alt+Shift+F1** | Toggle automatic self First Aid below **50% HP**, with healing priority over automatic attacks. | `AutoFirstAid`; [AutoFirstAid.md](AutoFirstAid.md) |
| **G** | Pick up the ground-item snapshot at the current hex; if empty, approach the nearest eligible hex within **5 hexes** and pick up its items. | `GroundPickup`, `ChosenActions`; [GroundPickup.md](GroundPickup.md) |

These hotkeys require the active **Game** screen, no console, no focused text input and no modal
screen. Modifier combinations are exact: Ctrl does not alias these actions, Alt+Shift+F1 does not also
request manual First Aid, and Alt+Shift+F2 does not invoke Doctor. Unmodified F1 still toggles help;
unmodified F2 keeps its existing behavior. Shift+F10 retains the existing atlas-dump shortcut.
F11 remains the separate session-only NPC-name toggle.

## Self-care queue contract

- Client API: `bool SelfCare::Request(CritterProperty skill, bool priority = false)`,
  `bool SelfCare::CanUse(CritterProperty skill)`, and `bool SelfCare::IsPending()`.
- Only `SkillFirstAid` and `SkillDoctor` are accepted. First Aid requires missing HP; Doctor requires
  an injury as defined by `Tla::IsCritterInjured`. Both check the chosen character, current local map,
  alive state, gameplay UI, scenario freeze/rest, their independent cooldowns and reachable maximum AP.
- `CanUse` means **eligible to queue**. Busy state and insufficient *current* AP do not reject a queued
  request: ordinary `ChosenProcess` waits for them. This lets automatic First Aid reserve the next AP
  instead of repeatedly losing them to an attack. It is not a server permission or a success result.
- Ordinary manual requests replace the queue as other manual actions do. A `priority` request inserts
  one normal `ChosenUseSklOnCritter` action at the front, or promotes an existing matching self-use,
  without deleting waiting manual actions. Current client movement stops; its queued command remains
  available to resume afterward. Already submitted server actions are not interrupted.
- The queue rechecks eligibility before prediction and submission. `IsPending` covers queued self-care
  and a bounded two-second client wait after submission. The wait reduces duplicate requests before
  cooldown replication; it is not an acknowledgement that treatment succeeded. A rejected request can
  be retried after that wait. Cooldowns remain authoritative and continue to block retries afterward.
- Flow: `InputHandler` / `AutoFirstAid` -> `SelfCare` -> client `ChosenActions` -> ordinary `UseSkill`
  RPC -> server `ChosenActions` -> `CritterActions::UseSkill` -> `Main::CritterUseSkill`.
  The server checks life/scenario/AP, resolves the target, applies the normal healing/injury roll,
  kit bonuses and cooldown. There is no self-heal cheat RPC or client health mutation.

## Ground pickup queue contract

`ChosenActions::QueueGroundPickup(mpos hex, ident[] itemIds)` accepts the selected snapshot. Its
movement reaches the exact hex (`cut = 0`), without the ordinary nearby-hex path fallback. Each item
is revalidated against the snapshot before submission; disappeared or moved items are not replaced
by another instance with the same prototype. Manual queue replacement cancels the snapshot.
The existing `PickItem` RPC adds an exact-item-ID argument: tagged G actions send it,
legacy actions send `ZERO_IDENT`. Server `CritterActions` validates and picks up the exact instance;
AP, carrying restrictions and normal item behavior remain authoritative.

## Persistence and verification boundary

Zoom is saved automatically after a one-second debounce, with a flush on map unload/client finish;
`UserOptions::SaveValues` merges it into existing local settings. F10 is a session-only choice.
Automatic First Aid starts disabled and resets on map unload/load, disconnect, chosen/map changes
or death. Disabling it stops new automatic requests; a skill already in the ordinary queue or submitted
to the server may finish.

Self-care requests, their in-flight guards and pickup snapshots are transient client state; they add
no persistent character property or new healing RPC. HP presentation uses server-published
`HealthDisplayMerc` / `HealthDisplayMaxHp`, not client inference about an invisible owner.
Consult the owning feature pages for details.

After merging these modules, run the existing incremental `BakeResources` to compile the complete
script/config set. Static whitespace or source review alone does not demonstrate input focus,
replication timing, rendering, pathing or gameplay success. Do not start servers/E2E or add a test suite
without explicit authorization. Update this page when hotkeys, queue handoffs or persistence change.
