# Nullability

Convention and runtime enforcement for nullable values across AngelScript and the native engine boundary.

## Core principle

> Better to not pass `null` at all than to defensively check inside and bail out.
>
> A parameter or return is nullable when absence is a meaningful part of the contract. Do not add nullable inputs merely to hide invariant failures. Guarding a genuinely fallible lookup with an early return is valid; requiring a non-null value and fixing its caller is appropriate when absence is not supported.

This applies symmetrically on both sides of the script-engine boundary.

## Script side: `T?` suffix

AngelScript modules in [Scripts/](Scripts/) use a Kotlin/C#-style `?` suffix on the type to mark nullability. Default is **non-nullable**.

```angelscript
// Return may be null
Location? GetCritterLocation(Critter cr)
{
    if (cr.MapId == ZERO_IDENT) {
        return null;
    }
    Map? map = cr.GetMap();
    return map != null ? map.GetLocation() : null;
}

// Parameter may be null — body handles both cases
void OnCritterUseWeapon(Critter cr, WeaponUseMode useMode, HitLocation aim, Critter? target, mpos targetHex)
{
    mpos resolvedTargetHex = target != null ? target.Hex : targetHex;
    // ...
}
```

The `?` suffix is now **parsed natively by the AngelScript compiler** (FOnline "strong nullable" patch in [Engine/ThirdParty/AngelScript/sdk/angelscript/source/as_parser.cpp](Engine/ThirdParty/AngelScript/sdk/angelscript/source/as_parser.cpp) and `as_compiler.cpp`). It is no longer stripped by a preprocessor — the compiler tracks nullability through type checking and emits **compile-time diagnostics** (see «Compile-time diagnostics» below). The parser distinguishes type-suffix `?` from the ternary operator `?` by scanning forward at the same nesting level: a type-suffix is followed by an identifier/`[`/`,`/`)` boundary; a ternary is followed by `:` after the truthy expression.

### Compile-time diagnostics

The compiler now reports three nullability problems as warnings. Treat them as errors to fix, not noise:

| Warning | Meaning | Fix |
|---------|---------|-----|
| `Redundant null comparison: 'T@' is a non-nullable handle and can never be null` | A `== null` / `!= null` test on a value the compiler knows is non-null. | Remove the test. If the source *can* actually be null, mark the script source `T?` or the native API handle `nptr<T>`. |
| `Dereference of nullable handle 'T@?' without a null-check` | A field/method access on a `T?` value that has not been narrowed. | Narrow it first (see narrowing below), or — if it was over-marked — drop the `?`. |
| `Redundant '?': initializer of type 'T@' cannot be null` | A `T? x = <non-null expr>;` local where the initializer is provably non-null. | Drop the `?`: `T x = <expr>;`. |

**Flow-sensitive narrowing.** Inside a guarded scope the compiler narrows a `T?` **local** (not a member access or repeated call) to non-nullable `T`. Recognized forms — one operand must be the literal `null`, the other the local:

- `if (x != null) { /* x is T here */ }` and `if (x == null) { return; } /* x is T after */`
- the same in a ternary: `x != null ? x.foo() : ...`
- short-circuits: `x != null && x.foo()`, `x == null || x.foo()`

Not recognized: multi-condition conjunctions in a block head (`if (a != null && b != null) { ... }` does **not** narrow `a`/`b` in the block body — bind/guard each separately), member-access targets (`obj.Field != null` does not narrow `obj.Field`; bind it to a local first), and a fresh call each time (`Game.GetCritter(id) != null && !Game.GetCritter(id).IsX` — the second call is unnarrowed; bind once to a `T?` local).

**Reference casts.** `cast<T>(x)` is typed **non-nullable** `T@` (the "assume the cast succeeds" contract), so `cast<T>(x) != null` warns as redundant even though a failed downcast returns `null` at runtime. For the type-test idiom, spell the cast nullable: `cast<T?>(x) != null`, or `T? y = cast<T?>(x); if (y != null) { ... }`.

**Formatting.** clang-format treats `?` as the conditional operator and mangles the nullable suffix (`Item? item` → `Item ? item`, `cast<Item?>(x)` → `cast < Item ? > (x)`, `T?[] a` even splits across lines). Always format via the project formatter — VS Code `Format :: *` tasks, [Tools/Formatter/format_project.py](Tools/Formatter/format_project.py), or [FormatSource.bat](FormatSource.bat) — which post-processes clang-format output to restore the suffix in both `.fos` and `.fogui`. Do **not** run `Tools/clang-format-20.exe` directly on these files; it re-mangles the suffix. The GUI generator ([Tools/InterfaceEditor/generate_gui_screens.py](Tools/InterfaceEditor/generate_gui_screens.py)) applies the same repair, so regenerated `Scripts/Generated/GuiScreens.fos` retains `T?` spelling in the normal formatted generation route.

### `///@ Event` and `///@ RemoteCall` declarations

The same `?` suffix is supported in `///@ Event` and `///@ RemoteCall` tag declarations, and the [`MetadataBaker`](Engine/Source/Tools/MetadataBaker.cpp) propagates the per-arg nullable bit into the baked engine metadata (`ArgDesc::Nullable` on `EntityEventDesc::Args` / `RemoteCallDesc::Args`).

```angelscript
///@ Event Server Game OnCritterDamaged(Critter cr, Critter? attacker, int32 damage)
///@ Event Server Game OnCritterDead(Critter critter, Critter? killer)
///@ RemoteCall Server SwitchCharacter(Critter? newCritter)
```

The declaration is the contract. Every `[[Event]]` subscriber and every `[[ServerRemoteCall]]` / `[[ClientRemoteCall]]` / `[[AdminRemoteCall]]` implementation that matches the event/call name must use the same `?` marker on each argument. [`validate_nullable.py`](Tools/NullableEstimate/validate_nullable.py) walks all `.fos` files, pairs declarations with their handlers by function name, and fails on any per-arg nullable mismatch.

```angelscript
// Matches the OnCritterDamaged declaration above.
[[Event]]
void OnCritterDamaged(Critter cr, Critter? attacker, int32 damage) { ... }

// Would be rejected by validate_nullable.py — declaration has `Critter?`,
// handler drops the `?`:
[[Event]]
void OnCritterDamaged(Critter cr, Critter attacker, int32 damage) { ... }
```

The AngelScript compiler now parses `?` natively and emits the compile-time diagnostics above, so per-arg nullability is checked while compiling. `validate_nullable.py` still enforces declaration↔handler parity across files (its source scan covers matching declarations and handlers across files), and the engine's runtime null guards on entity meta-types back it up — see «Runtime enforcement» below.

## Engine side: `ptr<T>` and `nptr<T>`

Native script bindings express the contract in the type itself: `ptr<T>` is a borrowed non-null handle, while `nptr<T>` is a borrowed nullable handle. This applies to `///@ ExportMethod`, exported `///@ ExportRefType` members, script callback signatures (`FindFunc` / `CheckFunc`), exported events, and other generated script-facing surfaces. The former empty `FO_NULLABLE` macro has been removed.

Current declaration excerpts from `Engine/Source/Scripting/ServerCritterScriptMethods.cpp` and
`ServerPlayerScriptMethods.cpp` (definitions omitted):

```cpp
///@ ExportMethod PassOwnership
FO_SCRIPT_API FO_RETURNS_PARENT nptr<Map> Server_Critter_GetMap(ptr<Critter> self);

///@ ExportMethod
FO_SCRIPT_API void Server_Player_SwitchCritter(ptr<Player> self, FO_REQUIRES_COVER nptr<Critter> cr);
```

The cover/ownership annotations express additional engine contracts; they do not change `ptr` versus
`nptr` nullability. Callback templates use the same wrappers, for example
`server->FindFunc<bool, ptr<Critter>, nptr<Item>>(funcName)` for a matching callback signature.

Bare handle pointers (`T*`) no longer carry a nullable contract at this boundary. Codegen rejects them; use `ptr<T>` or `nptr<T>`, including inside handle containers such as `vector<ptr<Item>>`. Raw pointers can still be valid in internal C++, low-level/C ABIs, and non-script engine hooks. See [Engine/Docs/SmartPointers.md](Engine/Docs/SmartPointers.md) for the full native pointer vocabulary.

## Runtime enforcement

Runtime validation is plumbed through codegen-generated `MethodDesc::Call` lambdas, **not** the AS-to-native bridge. [Engine/BuildTools/codegen.py](Engine/BuildTools/codegen.py) emits per-method calls to `NativeDataProvider::CheckArgNotNull` / `CheckReturnNotNull` (defined in [Engine/Source/Common/ScriptSystem.h](Engine/Source/Common/ScriptSystem.h)) for non-null `ptr<T>` contracts right before/after the native invocation:

```
MethodDesc::Call(call)
  → NativeDataProvider::CheckArgNotNull(call, i, "Server_Game_UnloadCritter", "cr", "Critter")   // for each non-nullable entity arg
  → native invocation
  → NativeDataProvider::CheckReturnNotNull(call, "...", "...")                                  // for non-nullable entity return
```

These checks protect dispatch through the generated `MethodDesc::Call` wrapper independently of the
script backend. They do not wrap arbitrary direct C++ calls to the native implementation. This game
uses AngelScript; the managed backend remains disabled in its CMake configuration.

Violation surface: `ScriptException` with the method name, parameter name and type, propagated to the calling AngelScript context.

**Scope of enforcement:** every **script handle** crossing the script ↔ native boundary is validated. Concretely codegen emits the check when the meta-type is one of:
- a `///@ ExportEntity` name (`Critter`, `Item`, `Map`, `Location`, `Player`, `Game`, `ImGui`) or the generic `Entity`,
- an entity relative (`Abstract<Entity>`, `Proto<Entity>`, `Static<Entity>` — currently `AbstractItem`, `ProtoCritter`, `ProtoItem`, `ProtoLocation`, `ProtoMap`, `StaticItem`),
- a `///@ ExportRefType` class (`MovingContext`, `MapSpriteHolder`, `SpritePattern`, `VideoPlayback`, `ScriptImGui`).

On the C++ engine side the matching spellings are `ptr<Critter>` / `nptr<Critter>`, `ptr<CritterView>` / `nptr<CritterView>`, `ptr<Map>` / `nptr<Map>`, `ptr<ProtoItem>`, `ptr<StaticItem>`, `ptr<MovingContext>`, and so on. The membership test lives in `is_validated_pointer_meta_type(...)` in [Engine/BuildTools/codegen.py](Engine/BuildTools/codegen.py).

The project [validate_nullable.py](Tools/NullableEstimate/validate_nullable.py) gate rejects raw pointers in `ExportMethod`/`FO_SCRIPT_API`, exported `ExportRefType` members, and `FindFunc`/`CheckFunc` template arguments before baking reaches codegen. It deliberately ignores ordinary internal raw pointers and `SetupBakersHook`/other `///@ EngineHook` declarations.

**Boundary limit:** direct script-to-script calls do not pass through `MethodDesc::Call`; this native
wrapper does not establish their runtime validation. Compiler nullability diagnostics and source analyzers
are separate checks with their own limits (see [Tooling](#tooling)). Do not infer a runtime null guard on
every script invocation from a native ABI validator pass.

## TLA migration notes

The current engine enforces these contracts during normal runtime, not only during static checks:

- Bind nullable dictionary/engine results as nullable locals. For example, `dict.get(id, null)` must assign to `T?`, not `T`, when the key can be absent.
- Generated component accessors are not nullable probes. Check `Has<Component>` first (`cr.HasDialogContext`, `item.HasRadio`, etc.), then use the component accessor.
- TLA runs with `Server.SingleThreadedLogic = True`: server callbacks access reachable entities directly, without `Sync::Lock*`, `Game.Sync` or server `[[Async]]` cover markers. Do not restore the removed server pattern. The reusable engine's multithreaded mode needs a separate migration; client async attributes are not server entity-cover guidance.
- Dialog metadata participates in baking. Special answer links used by `.fodlg` files need globally visible `///@ Enum DialogAnswerLink ...` declarations, otherwise the dialog baker cannot resolve them even if AngelScript compilation succeeds.
- The client `Chosen` accessor (`Game.Chosen`) is **non-nullable and throws** when there is no chosen critter. Guard with `HasChosen` instead of a null-check: replace `if (Chosen == null) return;` with `if (!HasChosen) return;`, and place the `HasChosen` guard *before* any `Critter chosen = Chosen;` assignment (the assignment itself throws when absent). Fallible native lookups such as `Game.GetCritter`, `Game.GetItem`, `Critter.GetItem`, and `Map.GetCritterAtScreenPos` return `nptr<T>` and surface as `T?` in script; bind and narrow them accordingly.
- GUI warnings surface in generated [Scripts/Generated/GuiScreens.fos](Scripts/Generated/GuiScreens.fos), but the fix belongs in the owning [Gui/*.fogui](Gui/) embedded code. The normal generator runs clang-format and then `fix_fos_nullable_suffix`, restoring `Type? name` and nullable casts; it does not intentionally leave `Type ? name`. `--no-format` skips that formatting/repair stage. Follow [AGENTS.md](AGENTS.md#guiscreensfos-pitfall) for matching embedded-code fixes versus full regeneration; compile after updating the authored and generated surfaces.

## Tooling

The active tools are documented in [Tools/NullableEstimate/README.md](Tools/NullableEstimate/README.md):

| Tool | Purpose |
|------|---------|
| `validate_nullable.py` | Read-only gate for native `ptr<T>`/`nptr<T>` script ABI, obsolete `FO_NULLABLE`, script primitive `T?`, and Event/RemoteCall declaration-handler parity. |
| `apply_nullables.py` | Script-side analyzer/cleanup. `--check` verifies that its output would be unchanged. |
| `estimate_nullables.py` | Read-only script nullability coverage report. |

`apply_native_nullable.py` remains only as a legacy helper for checking out pre-`ptr`/`nptr` engine revisions. It does not define the current native contract and is not part of the active gate.

The validator deliberately does not infer whether an API should be nullable. Authors choose `T` versus `T?` in script and `ptr<T>` versus `nptr<T>` in native declarations; the tools verify that the chosen spelling is legal and consistent.

## Workflows

The VS Code `Analyze :: Nullable Placement` task runs `validate_nullable.py`; `Analyze :: Nullable All` combines it with the read-only script analyzer and coverage report. Unit suites were removed at the owner's request. [.github/disabled-workflows/build.yml](.github/disabled-workflows/build.yml) is an inactive archive, not a CI gate; do not restore tests or reactivate workflows to follow this guide.

Manual equivalents:

```bash
python Tools/NullableEstimate/validate_nullable.py
python Tools/NullableEstimate/apply_nullables.py --check
python Tools/NullableEstimate/estimate_nullables.py
```

## Adding or editing contracts

Choose the contract explicitly at the declaration:

1. Use `T` / `ptr<T>` when the handle must exist.
2. Use `T?` / `nptr<T>` only when absence is a real state that the function handles.
3. Use the same wrappers in exported ref-type accessors and `FindFunc`/`CheckFunc` template signatures.
4. Run `validate_nullable.py` and compile the affected scripts; bake when the changed contract participates in baked content. Native ABI changes also require the smallest affected application build. These checks do not infer nullable intent or establish unexercised runtime behavior.

## See also

- [Docs/Systems.md](Docs/Systems.md) — game module ownership and content boundaries.
- [Docs/ScriptStyle.md](Docs/ScriptStyle.md) — AngelScript conventions.
- [AGENTS.md](AGENTS.md#native-extensions) — native extension registration and verification policy.
- [Engine native extensions guide](Engine/Docs/en/how-to/native-extensions.md) — reusable binding/codegen contract.
