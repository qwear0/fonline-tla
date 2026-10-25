# Automatic self First Aid

Client-side convenience for the chosen character; it does not change First Aid balance, AP costs,
skill rolls, cooldowns or server authority.

## Controls and ownership

- **Alt+Shift+F1** toggles the mode through `InputHandler` -> `AutoFirstAid::Toggle()`.
- `Scripts/AutoFirstAid.fos` owns the mode, character/map identity and request pacing.
- `Scripts/ChosenTabs.fos` shows a dark-green status tab, using the same presentation as Sneak:
  **Авто-санитар** / **Auto First Aid**. `MsgStr::StrAutoFirstAidTitle` is Game text key `453`.
- `Scripts/SelfCare.fos` owns ordinary self-skill queueing and outstanding-request tracking.
- `Scripts/ChosenActions.fos` owns queue execution and the authoritative skill RPC path.
- `Scripts/DenAutoAttack.fos` yields both new attacks and delayed target replies to self-care.

## Trigger and priority

1. `ChosenActions::ChosenProcess` calls `AutoFirstAid::Process()` before processing ordinary actions
   and before waiting for movement to finish. A standalone `OnLoop` subscription would not establish
   this priority: an already queued attack could otherwise run first.
2. The mode is eligible only for its original living chosen on its original local map.
3. Healing requires `int64(CurrentHp) * 2 < int64(MaxLife)`, with positive `MaxLife`.
   Exactly 50% HP does **not** trigger it; odd maximum HP values are not rounded down.
4. `SelfCare::CanUse(SkillFirstAid)` must permit queueing: valid game UI/session, no console,
   no scenario freeze/rest, expired First Aid cooldown and a reachable maximum AP cost.
   Insufficient **current** AP or an unfinished action does not make the skill impossible.
5. `SelfCare::Request(SkillFirstAid, true)` inserts/promotes one self-skill at the front and preserves
   the remaining action queue. It can stop a local approach, but never resets server busy state,
   clears an animation or bypasses ordinary execution/AP checks. The queued heal waits for AP and
   for the current action to finish, rather than repeatedly spending AP on attacks.
6. Automatic queue requests are paced at no more than one per 500 ms. `SelfCare::IsPending()` also
   suppresses duplicate queued/in-flight requests while waiting for authoritative state.

`AutoFirstAid::ShouldPrioritize()` is a side-effect-free eligibility check used by `DenAutoAttack`.
The attack guards also check `SelfCare::IsPending()`: this protects an accepted queued/in-flight heal
when the automatic eligibility predicate is false, including the short pacing window. Priority
queueing cancels the outstanding target lookup. A late reply, including a one-shot Ctrl+A reply,
must pass these guards before it can replace the queue. An already sent server attack completes
normally; automatic First Aid never retracts it.

First Aid cooldown, impossible maximum AP cost and unavailable UI do not reserve combat priority.
Temporary UI/freeze/rest pauses the mode rather than disabling it. Pending ordinary self-care follows
the same UI/freeze/rest execution guards as manual self-care.

## Lifetime and contracts

- Death/knockout, a different chosen, map unload/load or disconnection resets the mode.
  Identity checks also apply immediately to HUD visibility and eligibility, before lifecycle polling.
- Disabling stops future automatic requests. An already queued or sent ordinary self-skill may finish;
  toggling does not cancel a legitimate skill or interrupt its animation.
- State is held in non-persistent client `Game.AutoFirstAid*` properties, not mutable namespace globals
  or the `TLA.fomain` globals allowlist. There are no new server RPCs or save-data migrations.
- Public client API: `void Toggle()`, `bool IsEnabled()`, `bool ShouldPrioritize()`, `void Process()`.
- Required SelfCare client API: `bool Request(CritterProperty skill, bool priority = false)`,
  `bool CanUse(CritterProperty skill)`, `bool IsPending()` (queued or in-flight self-care).
  `CanUse` checks eligibility to queue, not accumulated current AP or completed busy time.
  `Request(..., true)` must deduplicate/promote rather than replace the rest of the queue.

## Verification boundary

See [Build.md](Build.md) for the existing verification workflow and [Systems.md](Systems.md) for
client/server ownership. Script/text changes require a combined resource bake with the matching
`SelfCare`, `ChosenActions` and `InputHandler` changes. An isolated child compile or a whitespace/
static-validator pass cannot establish integration, server-authoritative healing or HUD rendering.
Do not create new tests or start servers/E2E merely to verify this convenience feature.

Update this page when the trigger, key chord, queue/RPC contract, lifetime or status text changes.
