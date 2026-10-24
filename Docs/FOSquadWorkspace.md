# FOSquad two-repository workspace

## Layout and ownership

```text
/apps/
  FOSquad/                  # qwear0/fonline-tla: game code and resources
    Engine -> ../FOSquad_Engine
    EngineRevision          # last published engine baseline
  FOSquad_Engine/            # qwear0/fonline: reusable engine
```

`Engine` is a **relative symbolic link**, replacing the original gitlink. The conversion is published in game commit `47c2b49e5a5a7cb045013b894d349e165fbcefe5`; fresh clones of the fork contain this layout. The supported local layout uses an independent engine checkout, not submodule operations. This preserves the existing paths in CMake, `TLA.fomain` resource packs, formatters and editor tasks without copying engine sources or rewriting upstream build tools. Both repositories retain their own `master` branch and SSH `origin`. Run engine Git operations in `/apps/FOSquad_Engine`; `git add` in the game repository does not stage engine source changes.

Do not run `git submodule update` in this fork. `.gitmodules` has been removed. The symbolic link requires the sibling directory to be named `FOSquad_Engine`. On Windows, local clones require Git symbolic-link support (`core.symlinks=true`) and permission to create links (for example, Developer Mode). The archived runner checkout mechanism described below did not require Windows symlink privileges; it is not active CI.

## Recreate the workspace

Clone the two published forks into a common parent directory. The branches can advance independently;
use the game's `EngineRevision` to reproduce its published engine baseline rather than assuming that
the latest engine branch tip always matches the game.

```sh
git clone git@github.com:qwear0/fonline.git FOSquad_Engine
git clone git@github.com:qwear0/fonline-tla.git FOSquad
cd FOSquad
readlink -f Engine
git -C ../FOSquad_Engine rev-parse HEAD
```

Compare the engine HEAD with `EngineRevision`. To reproduce that exact version in a fresh, clean engine checkout:

```sh
git -C ../FOSquad_Engine checkout --detach "$(head -n 1 EngineRevision)"
```

Create a branch before developing engine changes. The existing `/apps/FOSquad_Engine` checkout was **not** detached or reset during setup; it remains on `master`.

## Linux build configuration

Toolchain recorded during workspace setup: Debian Linux x64, CMake 3.31.6, Ninja 1.12.1 and Clang 21.1.8. The preset files use schema version 10, which requires CMake 3.31 even though `CMakeLists.txt` declares a 3.22 minimum. The engine supports Clang >= 20 or GCC >= 13 on Linux.

Installed build prerequisites include Ninja, pkg-config and the X11, Wayland, EGL/OpenGL, udev and PulseAudio development libraries. ALSA development headers and Clang 21 were already present. Engine third-party libraries are built from its vendored sources.

Local machine settings live in the ignored `CMakeUserPresets.json`. To reproduce this machine's preset, create:

```json
{
  "version": 10,
  "configurePresets": [
    {
      "name": "fosquad-linux",
      "displayName": "FOSquad Linux (external engine, Clang 21)",
      "inherits": "auto",
      "generator": "Ninja Multi-Config",
      "cacheVariables": {
        "CMAKE_C_COMPILER": "/usr/bin/clang-21",
        "CMAKE_CXX_COMPILER": "/usr/bin/clang++-21"
      }
    }
  ],
  "buildPresets": [
    {
      "name": "fosquad-linux",
      "configurePreset": "fosquad-linux",
      "configuration": "RelWithDebInfo",
      "jobs": 4
    }
  ]
}
```

Configure and build from `/apps/FOSquad`:

```sh
cmake --preset fosquad-linux
cmake --build --preset fosquad-linux --target TLA_ServerHeadless TLA_Client TLA_ASCompiler
cmake --build --preset fosquad-linux --target CompileAngelScript
```

This uses `Build/Auto`, keeping existing VS Code build tasks compatible with the configured build tree. Binaries are written under `Binaries/`. Four compilation jobs limit memory pressure; increase only after observing available RAM. No game identity, gameplay settings or persistence defaults were changed.

Before running a real local shard, bake resources:

```sh
cmake --build --preset fosquad-linux --target BakeResources
./Binaries/Server-Linux-x64/TLA_ServerHeadless --ApplySubConfig LocalTest
# In another terminal, from the same game directory:
./Binaries/Client-Linux-x64/TLA_Client --ApplySubConfig LocalTest
```

The base `Server.DbStorage = Memory` remains for local development, not durable online-player progress.
The packaged `LAN` profile overrides it with SQLite in `TLA-Storage/LAN/`, outside the package; saved
characters survive restarts while Den siege progress resets. See [ClientUpdates.md](ClientUpdates.md)
for activation and storage paths. Public production keys and deployment remain separate work.

## Verification policy after unit-suite removal

The owner requested removal of game and engine unit suites. The game and engine instructions adopt the owner's policy (recorded here for context, with [AGENTS.md](../AGENTS.md) authoritative for game work): no new or modified automated tests/infrastructure without an explicit request, including temporary or delegated tests; use the least costly adequate existing checks or direct application use. Missing new tests are not a defect.

Game script suites and their harness, native dialog tests and Python unit suites are removed. Engine native/managed/Python/debugger unit suites and generated unit/coverage targets are removed. Third-party sources, manual gameplay tooling and runtime diagnostics are retained. Do not enable `FO_UNIT_TESTS` or `FO_CODE_COVERAGE`: the current engine rejects enabled values rather than providing those removed targets.

The test-removal work recorded successful CMake configuration and `CompileAngelScript`; these are prior results, not a rerun for subsequent documentation/integration changes. Evidence paths: `Build/configure-without-tests.log` and `Build/compile-without-tests.log`. The latter rebuilt the compiler and its native dependencies. This does not establish fresh client/server runtime behavior or a full resource bake.

The engine test removal and documentation repairs are published at `fd4877d81e55a89e7734f5b4dc3fff0c1b08f419` (`2026.1.65-dev`). `EngineRevision` records that published commit; the engine working tree is no longer required to carry unpublished repairs.

## Historical compatibility evidence (before removal)

Game baseline: `ec1ce12fd0f645979ca64bcb0bb0cc4def5c382c` (TLA 0.3.14).

- Original submodule revision: `e1a3815ff96c7d88f47314984e9d9242e37bc585` (engine `2026.1.30-dev`).
- Initially connected engine: `aafc7fe74bb8fa35e6395e04d8c6d2a66a91831a` (engine `2026.1.57-dev`), the historical pin before the published repair range.
- The original engine revision is an ancestor of the connected one; there are 29 intervening commits.

The initial workspace setup, before the owner requested test removal, recorded:

| Check | Result | Local evidence |
| --- | --- | --- |
| CMake configuration and generation | Passed | `Build/configure-fosquad.log` |
| Client, headless server, AngelScript compiler, unit-test binary | Built successfully | `Build/build-fosquad.log`, `Build/build-fosquad-resume.log` |
| All game AngelScript compilation | Passed; engine version and revision confirmed by the compiler | `Build/scripts-fosquad.log` |
| Native unit tests | 622 test cases, 733,804 assertions passed; exit 0 | `Build/unit-tests-fosquad.log` |

The initial native build was interrupted by the command time limit and resumed incrementally; the resumed build exited 0. The initial build emitted two Wayland scanner XML-validation warnings: Debian's scanner does not recognize the `frozen` attribute in SDL's vendored `wayland.xml`. Generation and linking completed, but a warning-free fresh build is not established; updating the scanner is separate follow-up work. No compiler/configuration diagnostics were found in the resumed build, CMake configuration or AngelScript compilation logs. Unit-test output includes missing-effect/sprite diagnostics from test fixtures, so the passing test count is not a claim that the entire log is diagnostic-free. Full resource baking, real game-server startup and an interactive client session were **not** verified during this setup. The checks establish native and script build compatibility, not complete gameplay compatibility. Engine sources were not modified.

## Archived CI and future engine updates

GitHub workflow definitions are now archived in `.github/disabled-workflows/` in both repositories. They are not active workflows and still contain historical references to removed tests. No remote GitHub setting was changed; the archive only affects remote execution after publication. Do not re-enable them without an explicit request and reconciliation.

The preserved `.github/actions/checkout-engine/action.yml` reads the full SHA from `EngineRevision`. In the archived game workflow, all nine engine-consuming jobs checked out `qwear0/fonline` at that SHA; the artifact-only release job did not need engine sources. On disposable CI runners, the action removes only the `Engine` symlink (or Git for Windows' text placeholder) and checks out an ordinary directory at `Engine/`. It never deletes the symlink target or replaces an existing directory. This preserves existing Windows/Linux/macOS build commands without requiring a sibling checkout on runners. The action's Python bootstrap was locally checked with six isolated fixtures (symlink, Windows text placeholder, absent path, existing directory, unrelated file, invalid revision); external checkout sentinel files were preserved. Workflow wiring was checked for all nine engine-consuming jobs. GitHub Actions itself has not been run locally.

For future updates:

1. Inspect and update the engine repository independently; do not reset local engine work.
2. Use the smallest adequate configure/build, script compilation, resource bake or direct shard interaction for the changed behavior.
3. Commit verified engine changes with a Russian description and push normally to the engine fork, not upstream; do not rewrite published history or include unrelated dirty changes.
4. Record that actually published full engine SHA in `EngineRevision`, never a provisional SHA for uncommitted changes.
5. Verify and publish the task-owned game changes separately under [AGENTS.md](../AGENTS.md#commit-policy). A coordinator can own Git mutations for delegated workers.

`EngineRevision` records a published baseline, not the mutable local checkout: always compare it with `git -C Engine rev-parse HEAD` when investigating differences.

### Published game-side migration

Game commit `47c2b49e5a5a7cb045013b894d349e165fbcefe5` publishes the gitlink-to-symlink conversion, unit-suite removal, agent policy and corrected game documentation to the fork. The temporary local `submodule.Engine.ignore=dirty` workaround was removed after that conversion. Fresh clones need no submodule workaround; inspect engine changes in its own repository.

The documentation-repair integration reran CMake configuration and `CompileAngelScript` successfully against the local dirty engine, with no compiler warnings. `git diff --check` and the existing nullable/native ABI validator also passed. The post-push `TLA_ServerHeadless` / `TLA_Client` build completed successfully; `BakeResources` subsequently completed against the prepared `2026.1.58-dev` engine state without matched warning/error diagnostics. Logs are retained locally under `Build/DocumentationRepair/`. These checks do not establish gameplay, rendered-site or deployment acceptance. These are historical checks against the prepared tree, not a claim that a later engine revision has automatically been rebuilt.

### Published engine repair baseline

Engine commit `fd4877d81e55a89e7734f5b4dc3fff0c1b08f419` is published to the engine fork's `master` and recorded in `EngineRevision`. Its eight first-parent updates (`2026.1.58-dev` through `2026.1.65-dev`) were validated as a complete range against `aafc7fe74bb8fa35e6395e04d8c6d2a66a91831a`, with dated EN/RU migration notes for every step. The final sources preserve the previously checked repair; only version/changelog publication records changed during publication.

Fresh engine documentation preparation, validation (411 Markdown entries), localization, description translations, snippets and API/reference/CMake/native parity checks passed on that final commit. Local publication/build evidence is under `Build/DocumentationRepair/Publication/`. HTML rendering remains unverified because Bundler (`bundle`) is unavailable; source checks do not establish site rendering or deployment. No production shard or gameplay acceptance is implied.
