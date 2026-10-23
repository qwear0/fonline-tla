# Documentation index

Game-side documentation for FOnline: The Life After. The engine keeps its own set under
`Engine/Docs/`, indexed by `Engine/Docs/README.md`; nothing here duplicates it.

Start with [../AGENTS.md](../AGENTS.md) for repository practices, then with the map below.

## Engine build configuration

TLA enables `FO_DATA_SOURCE_DAT`, `FO_DATA_SOURCE_ZIP`, `FO_IMAGE_FRM`, `FO_IMAGE_RIX` and
`FO_AUDIO_ACM` in `CMakeLists.txt` for its authored Fallout archives, images and sounds. PNG, TGA and FOFRM
importers are built in. `Server.DbStorage = Memory` needs no optional database provider.
To select persistent storage, enable its positive `FO_DATABASE_JSON`, `FO_DATABASE_SQLITE`
or `FO_DATABASE_MONGO` CMake option and reconfigure. Engine `2026.1.26-dev` requires new
empty storage for JSON and SQLite; it does not convert their old formats.
See the Engine's [component selection](../Engine/Docs/en/how-to/build/project-configuration.md#optional-native-components)
and [storage upgrade boundary](../Engine/Docs/en/explanation/persistence/index.md#storage-formats-and-upgrade-boundary).
After an Engine update, rebuild and rebake matching client/server outputs before running them.

## Orientation

| Document | What it answers |
| -------- | --------------- |
| [FOSquadWorkspace.md](FOSquadWorkspace.md) | Two-repository setup, engine baseline versus local state, build commands and bounded historical evidence. |
| [ClientUpdates.md](ClientUpdates.md) | Built-in Windows resource/DLL updates over LAN, matching client/server packaging and Memory-safe activation. |
| [Systems.md](Systems.md) | Durable gameplay ownership, sides, entrypoints and authored-content boundaries. Read before opening a `.fos`. |
| [ScriptStyle.md](ScriptStyle.md) | How a `.fos` module is written: header block, comments, naming, structure. |
| [Animation.md](Animation.md) | Baked root-motion import, sprite loading and native 2D walk-cycle alignment. |
| [DenDefense.md](DenDefense.md) | Shared Den introduction: confirmed-login ownership, companions, waves, global rest, ground loot and failure. |
| [AiControl.md](AiControl.md) | AI control bridge/MCP adapter, normal actions versus privileged QA, existing live-client diagnostics. |
| [../Nullability.md](../Nullability.md) | Script `T?`, native `ptr<T>`/`nptr<T>`, generated GUI ownership and existing checks. |

## Plans and status

| Document | What it answers |
| -------- | --------------- |
| [AuditPlan.md](AuditPlan.md) | The 2026-09-02 audit findings: original measurements, reconciled status and remaining work under current fork policy. |
| [Refactoring.md](Refactoring.md) | Standing decisions and the open remainder of the Scripts refactoring rounds. |
| [ReanimationPlan.md](ReanimationPlan.md) | The stabilization campaign: applicable existing checks, remaining work, historical milestones and proposals requiring owner authorization. |

## History

Session narrative is kept out of the plans so that a plan stays readable as a plan.

| Document | What it holds |
| -------- | ------------- |
| [History/Refactoring-2026.md](History/Refactoring-2026.md) | Batch reports, engine-bump follow-ups and bug hunts from refactoring rounds R2 and R3. |

## Conventions

- Maintainer documentation is English, like `AGENTS.md`, except the established Russian
  [ScriptStyle.md](ScriptStyle.md). Script comments and module headers are Russian.
- A plan records decisions and what is still open. What happened in a session goes to `History/`.
- When behaviour changes, the owning document changes in the same commit.
