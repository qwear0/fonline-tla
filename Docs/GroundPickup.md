# One-shot ground pickup

`GroundPickup` owns the client-side **G** request and its fixed target snapshot.
`ChosenActions` owns movement, the action queue, AP prediction and the existing server
`PickItem` call. `CritterActions` remains the authoritative server action owner.

## Selection

1. If the chosen's current hex has eligible visible items, select that hex without moving.
2. Otherwise examine only hexes within **5 hexes of the position at the key press**.
3. Select the reachable candidate with the smallest `Game.GetDistance` from that position.
   Distance means geometric hex distance, not walking-path length. Equal distances use
   ascending `x`, then `y`; a blocked nearer candidate does not exclude farther reachable ones.
4. Snapshot the eligible item ids on that one hex. Queue an exact approach, if necessary,
   followed by one ordinary pickup attempt per snapshot item.

An eligible item is a known visible client map item with `MapHex` ownership on the current
map, a positive count and `IsCanPickUp`, without `Hidden`, `IsGag`, `IsScenery` or `IsWall`.
Doors, containers and cars are excluded: ordinary `PickItem` interacts with them instead of
picking them up. G does not open nearby containers or loot corpses.

Reachability requires a free destination footprint (including the chosen's multihex), no
other living occupant there, and a nonempty engine path ending at the **exact** target hex.
The own hex requires no path. `IsNoMove` excludes remote candidates, not items underfoot.
The explicit destination check matters because the engine path API has an adjacent-hex
shortcut which alone does not establish that the destination is unblocked.
A route may be longer than five steps; the radius limits the target, not the route length.

**Nonstandard multihex limitation:** the current engine's critter-path overload may return
an empty or one-step shortcut for nearby targets when `Multihex > 0`. An endpoint mismatch
is conservatively rejected, even if that hex is physically reachable. Ordinary player
characters with `Multihex = 0` are unaffected. This does not substitute a neighbouring
pickup; full support for these larger characters requires an engine exact-path fix and
matching queue integration, not a new script path planner.

## Client contract

- `GroundPickup::Request()` is the public CLIENT entry point. `InputHandler` binds plain G
  only when gameplay input is appropriate, with UI/text-focus checks owned by that caller.
- `bool ChosenActions::QueueGroundPickup(mpos hex, ident[] itemIds)` consumes the fixed
  snapshot. It preserves the request state already set by `Request`, uses exact movement
  without widening `cut`, and stops rather than picks from a neighbouring hex when the
  approach fails. If queue admission returns false, `Request` clears its snapshot.
- `GroundPickup::IsQueuedDestinationValid(mpos hex)` rechecks the original chosen/map,
  freeze/rest, surviving visible snapshot targets and current exact reachability.
- `GroundPickup::IsQueuedItemValid(ident itemId, mpos hex)` rechecks snapshot membership,
  current chosen/map, visibility, eligibility and original target hex before execution.
- `GroundPickup::CancelRequest()` invalidates the snapshot. Manual queue replacement must
  invalidate it; the module also clears it on map load and disconnection.

Transient state lives only in CLIENT Game properties: `GroundPickupChosenId`,
`GroundPickupMapId`, `GroundPickupHex`, `GroundPickupItemIds`. No namespace mutable-global
allowlist entry, persistent save property or new pickup RPC is introduced.

There is no rescan after reaching the selected hex, after a rejection, or after the last
queued item. Items spawned later and items on other hexes require another key press.
Each snapshot id is attempted at most once; a weight-limited partial stack or a blocked
callback is not retried indefinitely. Disappeared, moved, hidden or transferred items are
skipped, not replaced by another item with the same prototype.

## Server authority

The existing `PickItem(mpos hex, hstring pid, ident itemId)` RPC uses a nonzero item id for
G and `ZERO_IDENT` for legacy prototype-based interactions. Its server helper is
`CritterActions::PickItem(Critter cr, mpos hex, hstring pid, ident itemId = ZERO_IDENT)`.

For an exact id the server resolves that same map item and checks map ownership, original
hex, prototype, eligibility and `cr.IsSee(item)`. Rejection never falls back to another id.
The legacy dynamic-item lookup also checks ownership and visibility; it no longer accepts
the engine's unfiltered first matching prototype. Legacy scenery, door and container
interaction remains on the existing path.

AP and freeze/rest authority remain in `CritterActions::ProcessAp` / `DenDefense::CanAct`.
The action still fires `OnCritterUseSkill` with `SpecialSkillPickOnGround`, then follows
`Main::CritterUseSkill` -> `ItemMovement::MoveItem` -> `ItemStacks::MoveItem`.
This preserves server weight/partial-stack rules and blocking move/skill callbacks; no
client inventory mutation or direct stack transfer is used.

## Source and verification

- [GroundPickup.fos](../Scripts/GroundPickup.fos): selection, snapshot and client guards.
- [ChosenActions.fos](../Scripts/ChosenActions.fos): queue integration and existing RPC.
- [CritterActions.fos](../Scripts/CritterActions.fos): exact/legacy server resolution and AP.
- [Main.fos](../Scripts/Main.fos), [ItemMovement.fos](../Scripts/ItemMovement.fos),
  [ItemStacks.fos](../Scripts/ItemStacks.fos): ordinary authoritative pickup chain.
- [Build.md](Build.md): supported bake and verification workflow. Static validators and
  whitespace checks do not establish runtime movement, replication or AP behaviour.

Update this page when the selection radius/order, eligibility, queue cancellation,
exact-target RPC contract or normal server pickup chain changes.
