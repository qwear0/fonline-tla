# Persistent client map zoom

## Owner and scope

- [ClientZoom](../Scripts/ClientZoom.fos) owns the local-map zoom preference on the **client**.
- Existing `Gui/Game.fogui` input still performs Z + mouse-wheel zoom and Z + middle-button reset.
  `InputHandler`, generated GUI code and the Engine are not modified by this feature.
- World-map zoom is separate and is not persisted here. No server state or RPC is added.

## Input and map lifecycle

1. `InitClientZoom` loads `Settings.ClientZoom.Level` and subscribes to Game events.
2. `OnMapLoad` restores the preference with `CurMap.ChangeZoom(level)` when a local map exists
   and `Settings.View.MapZoomEnabled` is true. Global-map contexts have no `CurMap`.
3. `OnMouseDown`, at `EventPriority::Lowest`, observes `CurMap.SpritesZoomTarget` **after** the
   normal GUI input handler. It requires the foreground screen to be `GuiScreen::Game`,
   zoom-in-progress to be enabled and the console to be inactive. It neither consumes input
   nor changes screens, focus, cursor or the zoom itself.
4. A changed preference schedules a write one second after the last change. `OnLoop` only checks
   that deadline; it does not write every frame or sample intermediate animated zoom values.
5. `OnMapUnload` and `OnFinish` flush an outstanding change without waiting for the deadline.
   Explicit `UserOptions::Save` also saves the current preference and clears the pending write.

The supported script API is read-only `Map.SpritesZoom` / `Map.SpritesZoomTarget` plus
`Map.ChangeZoom(float)`, not a writable zoom property or hypothetical `GetZoom` / `SetZoom` methods.
The preference stores the **target**, not the interpolated current zoom. Restoration uses normal
Engine interpolation and scroll-area constraints. A small map's automatic minimum zoom does not
replace the player's preference; the next map can therefore restore it again.

The game clamps saved/restored values to the Engine's current `GameSettings::MIN_ZOOM` /
`MAX_ZOOM` range, **0.05–20.0**, declared in Engine `Source/Common/Common.h`.
Missing/zero, negative or NaN preferences use **1.0**. Positive out-of-range values are clamped.
Legacy game declarations `View.SpritesZoomMin/Max` are not the current Engine's zoom limits.
Review the constants when updating the Engine; individual map constraints remain Engine-owned.

## Persistence contract

- `ClientZoom.Level` is a script-declared client float setting in the cached `LocalSettings.focfg`.
  Normal Engine startup applies that config before script initialization.
- `Game.ClientZoomLevel`, `ClientZoomSavedLevel` and `ClientZoomSaveAt` are client-only mutable
  properties. They retain the preference and debounce state across map destruction, without
  mutable namespace globals, network synchronization or server-save migration.
- [UserOptions](../Scripts/UserOptions.fos) owns `SaveValues(dict<string, string>)`.
  Engine `SetUserConfig` replaces the entire config, so `SaveValues` instead reads the existing
  entry with `Game.GetCacheText` and updates it through `Game.SetCacheText` under the same name.
- Only supplied root keys are replaced, including repeated assignments and `+=` continuations.
  Other raw entries, comments and named sections are retained. New root assignments are prepended
  so an incomplete continuation at EOF cannot consume them. The writer skips unchanged output.
- Zoom autosave supplies **only** `ClientZoom.Level`; it does not snapshot partially edited proxy
  fields or other foreground options. The explicit options Save retains its normal snapshot while
  preserving keys outside that snapshot, including preferences owned by other modules.
- A crash/forced termination before the debounce expires can lose the last pending change;
  normal map changes and shutdown flush it. No per-frame disk-write guarantee is claimed.

## Verification and maintenance

After integrating script changes, use the existing incremental `BakeResources` workflow in
[Build.md](Build.md); it compiles scripts too. Source/whitespace validators do not establish runtime
persistence, GUI event ordering or visual interpolation. Do not add tests, launch servers or run
E2E checks without the owner's authorization.

Update this page when the setting key, supported zoom API/range, input ownership, event priority,
debounce policy or UserOptions merge contract changes.
