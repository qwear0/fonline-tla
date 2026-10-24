# Fast local builds and LAN updates

Read before choosing build, verification, packaging or server-restart commands. Use the smallest
adequate route; this guide does not replace the activation/storage rules in [ClientUpdates.md](ClientUpdates.md).
For the purpose of `Baking/`, output retention and end-of-task cleanup, read [DiskCleanup.md](DiskCleanup.md).

## Choose the work, not the full chain

| Changed inputs | Minimum relevant work |
| --- | --- |
| Documentation only | Check links and `git diff --check`; no build, bake, package or restart. |
| Scripts, dialogs, text, prototypes, maps | One incremental `BakeResources`; it already compiles scripts. Use `CompileAngelScript` separately only when an earlier syntax check is useful. |
| GUI | Update the owning `.fogui` and matching generated code; regenerate when required by `AGENTS.md`, then bake once. |
| `SourceExt`, native engine code or native build configuration | Build affected native targets; rebake/rebuild matching client/server outputs when their contracts change. |
| LAN config, Embedded resources, native runtime or engine baseline | Follow the complete matching build/package procedure in [ClientUpdates.md](ClientUpdates.md); do not assume an old patched binary is current. |

For ordinary script/content edits, reuse already verified native outputs **only if their sources and
native/embedded configuration have not changed**. Matching `.build-hash` files are necessary for packaging,
not proof that a native output is up to date. Do not run `Verify :: All`, force-bake, rebuild every target,
regenerate all GUI, or repeat a successful bake by habit. Warnings still fail verification.

```sh
cmake --build Build/Auto --config RelWithDebInfo --target BakeResources
# Syntax-only feedback instead of a bake, when no deployment is being prepared:
# cmake --build Build/Auto --config RelWithDebInfo --target CompileAngelScript
```

## Fast existing route for a LAN resource update

The packaged **server already includes client Resources and the native update feed**. An ordinary
server/feed update does not also need a separately packaged Windows client distribution. Build that
separately only when a fresh downloadable client is requested or required by the host/runtime boundary.

After a successful bake, check the native-source prerequisite above and matching hashes in
`Baking/Resources.build-hash`, the required server outputs and Windows client/runtime outputs. With
those prerequisites satisfied, package **Server only**, outside the live package:

```sh
# Run from the game root. Select a fresh staging directory for each release.
release="$PWD/tmp/lan-update-$(date +%Y%m%d-%H%M%S)"
build_hash="$(tr -d '\r\n' < Baking/Resources.build-hash)"
python3 Engine/BuildTools/package.py \
  -maincfg "$PWD/TLA.fomain" -buildhash "$build_hash" \
  -devname TLA -nicename FOnlineTLA \
  -target Server -platform Linux -arch x64 -pack Headless+Raw \
  -config LAN -input "$PWD" -output "$release" \
  -expect-client-runtime Windows:win64: -resource-pack-jobs 4
```

`4` is a bounded packaging-worker example, not a measured speedup guarantee; reduce it for available
CPU/RAM. It does not change native compilation parallelism. Keep normal archive/payload validation.
Use an adequate command timeout or track a background PID, log and exit status: a short timeout that
kills compression and forces a rerun wastes the completed work. Do not start duplicate packagers.

Before activation, require successful packaging and complete `ServerResources/`, `Resources/` and
`PlatformBinaries/Windows-win64/`. Then follow [ClientUpdates.md](ClientUpdates.md#activate-and-use):

1. Keep the old server running during preparation; never run `MakePackage-LAN` against its directory
   (that target deletes `TLA-LAN/`). Never replace its packs in place.
2. Stop the exact old process gracefully. Back up the stopped complete `TLA-Storage/LAN/` and preserve
   the previous server package under `/data/!backups/FOSquad/` before switching.
3. Move the staged server root to the usual `TLA-LAN/TLA-Server-LAN-Linux/` location and launch there;
   keep the original SQLite path and never open it from two servers. Replacing just this server root
   does **not** refresh a separate downloadable client directory/archive.
4. Confirm `Connect to DbSQLite data base`, `Load client data packs for synchronization`, `Restore world`,
   `Start server complete!`, the expected listening port and no warnings/errors. Den siege resets on restart;
   saved characters remain. Report preparation and downtime separately.
5. Follow [DiskCleanup.md](DiskCleanup.md): keep compact evidence, one working bake and the required
   rollback sets; remove only finished task-owned staging copies, not the active package or its logs.

## Current limit: packaging is not incremental baking

`package.py` still constructs and validates the requested archives, including large art/music packs.
Unchanged `.fores` can be byte-identical after that work; repackaging does not itself mean a client must
redownload all assets. Inspect pack identities and actual updater logs before claiming download behavior.

The engine supports `FO_RESOURCE_ARCHIVE_CACHE_HELPER`, but this checkout has no configured game-owned
persistent helper. Local reuse inside one packager invocation is **not** a persistent cross-run cache.
Do not claim that unchanged packs are skipped automatically or invent an unsupported changed-pack-only
CLI. A persistent validated archive cache would be separate implementation work, not part of reading this
runbook. See the engine's [packaging/cache contract](../Engine/Docs/en/how-to/release/packaging.md#reproducibility-and-provenance).
