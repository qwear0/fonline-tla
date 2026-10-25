# Client health display

Client-only presentation of current/maximum HP above player and hired-merc names.
It does not change healing, combat, targeting or NPC-name preferences.

## Owners and entrypoints

- [`Scripts/ClientHealthDisplay.fos`](../Scripts/ClientHealthDisplay.fos) owns the toggle,
  localized labels and the small authoritative replication bridge.
- [`Scripts/MapText.fos`](../Scripts/MapText.fos) owns rendering and text layout.
- Client API: `void ClientHealthDisplay::Toggle()` and
  `bool ClientHealthDisplay::IsEnabled()`.
- `InputHandler` owns the F10 binding and UI/text-focus admission. It must call `Toggle()`
  only after those checks; this module does not subscribe to keyboard events.
- `Game.HealthDisplayEnabled` is a `Property Game Client bool`, initially false.
  It survives map changes within the client process but is not saved across restarts.
  No mutable namespace-global state or configuration allowlist entry is needed.

## Targets and replicated values

- Only `ControlledByPlayer` critters and currently hired mercs qualify; ordinary NPCs do not.
  The existing non-dead render pass includes knocked-out critters, not corpses.
- [`Merc::IsMerc`](../Scripts/Merc.fos) tests `MercMasterId != ZERO_IDENT` on the server.
  `MercMasterId` is server-only, so the bridge publishes the derived boolean
  `HealthDisplayMerc` with `PublicSync`, without publishing the master's identity.
  This covers any hired merc, not only the chosen player's squad or Den prototypes.
- `CurrentHp` is already `PublicSync` in
  [`Scripts/CritterProps.fos`](../Scripts/CritterProps.fos).
- `MaxLife` is virtual: [`Scripts/Parameters.fos`](../Scripts/Parameters.fos) computes
  `MaxLifeBase + StrengthBase + 2 * EnduranceBase`, adds the `MaxLifeBase` drug effect
  and clamps to 1–9999. [`Scripts/Drugs.fos`](../Scripts/Drugs.fos) keeps `DrugEffects`
  owner-only, so another critter's client-side `MaxLife` is not a reliable denominator.
- The bridge publishes server-evaluated `HealthDisplayMaxHp` with `PublicSync` instead.
  Both new critter properties are transient (not `Persistent`). `OnCritterInit` rebuilds
  them after creation/load; post-set callbacks refresh the merc flag on `MercMasterId`
  changes and maximum HP on `MaxLifeBase`, `StrengthBase`, `EnduranceBase` or `DrugEffects`
  changes. Reactive setter signatures have no value-reference parameter and therefore
  observe the committed value. There is no polling timer or extra client RPC.
- A missing/uninitialized maximum (`<= 0`) suppresses the overlay rather than displaying
  an invented denominator. Existing current HP is displayed unchanged, including negative HP.

## Layout and localization

- Enabled targets show `HP: current/max` (`engl`) or `ОЖ: current/max` (`russ`), selected
  through `CurrentLanguage`; toggle feedback uses the same language selection.
- HP sits above the measured name block. If speech is active, the original speech stays
  at its existing anchor and retains its lifetime; name and HP stack above it with spacing.
  The overlay never writes `TextOnHead` or its expiry and does not use map-text messages.
- Enabled mercs show their name and HP even when F11 NPC names are off. Disabling HP
  restores the ordinary F11 admission without changing that preference.
- Name colors remain unchanged, including karma shading and the green Den leader name;
  speech and HP use the normal text color. Every block uses the critter's current alpha.
  Hidden sprites, fully transparent sprites and unavailable text anchors are skipped.

## Verification and update triggers

Use existing script quality/format checks and `git diff --check`; see [Build.md](Build.md)
for the normal resource-bake workflow. A static check does not establish runtime layout,
F10 focus handling or replication across login/load/hire/fire/drug changes. Those require
matching baked client/server outputs and an explicitly authorized application check.

Update this page and the bridge callbacks when merc membership or the `MaxLife` formula
changes. Changes to property replication require matching client/server metadata and a bake;
no save migration is introduced because the added properties are not persistent.
