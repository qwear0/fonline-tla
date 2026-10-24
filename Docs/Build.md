# Build, update and cleanup

## Pipeline and minimum work

Building the game includes **native compilation and resource baking**, followed by packaging when
needed. `Baking/` holds compiled scripts and runtime-ready maps, dialogs, text and processed assets;
unpackaged applications use it directly, and `package.py` turns it into distribution packs. It is
regenerable build output, not character storage or a backup. Keep one working bake for fast updates.

| Changed inputs | Required work |
| --- | --- |
| Documentation | Links and `git diff --check`; no build/restart. |
| Scripts, dialogs, text, maps, prototypes | One incremental `BakeResources`; it already compiles scripts. |
| GUI | Update `.fogui` and matching generated code; regenerate when required by `AGENTS.md`, then bake. |
| Native code / CMake | Build affected targets; rebake and rebuild matching sides when contracts change. |
| Engine, embedded resources or release config | Check matching native outputs and package configuration; follow [ClientUpdates.md](ClientUpdates.md). |

```sh
cmake --build Build/Auto --config RelWithDebInfo --target BakeResources
# Earlier syntax-only feedback, if useful (not required in addition to every bake):
# cmake --build Build/Auto --config RelWithDebInfo --target CompileAngelScript
# Native example; choose only affected targets:
# cmake --build Build/Auto --config RelWithDebInfo --target TLA_ServerHeadless
```

Reuse verified native outputs only while their sources/native configuration remain unchanged.
Matching `.build-hash` files are necessary for packaging, not proof of freshness. Do not routinely
force-bake, regenerate all GUI, run `Verify :: All`, rebuild every target or repeat successful checks.
Warnings count as failures; report actual results and gaps.

## LAN resource update

The server package already contains client resources and the native update feed. Package **Server
only** for ordinary updates; make a separate client distribution only when requested or required by
the host/runtime boundary. After a successful bake and freshness checks, require matching hashes in
`Baking/Resources.build-hash` and the required server/Windows client/runtime outputs:

```sh
# From the game root; never package over the running server.
release="$PWD/tmp/lan-update-$(date +%Y%m%d-%H%M%S)"
build_hash="$(tr -d '\r\n' < Baking/Resources.build-hash)"
python3 Engine/BuildTools/package.py \
  -maincfg "$PWD/TLA.fomain" -buildhash "$build_hash" \
  -devname TLA -nicename FOnlineTLA \
  -target Server -platform Linux -arch x64 -pack Headless+Raw \
  -config LAN -input "$PWD" -output "$release" \
  -expect-client-runtime Windows:win64: -resource-pack-jobs 4
```

`4` bounds packaging workers; adjust for RAM/CPU, without disabling archive validation. Allow enough
time or track a background PID/log/exit status; do not kill long compression with a short timeout or
start duplicate packagers. Packaging still processes large unchanged packs: no persistent archive
cache helper is configured here. Repacking alone does not imply a full client redownload.

Activate using [ClientUpdates.md](ClientUpdates.md#activate-and-use): prepare while the old server
runs, require successful packaging, then stop it gracefully and back up complete stopped storage and
the previous package. Replace the server root and launch from `TLA-LAN/TLA-Server-LAN-Linux/`, keeping
`TLA-Storage/LAN/` and never opening it from two servers. Check SQLite restore, updater-pack loading,
`Start server complete!`, the listening port and zero warnings/errors. Report preparation and downtime
separately. Den siege resets; saved characters remain. This does not refresh a separate client archive.
**Never run `MakePackage-LAN` against a running package: that target deletes `TLA-LAN/`.**

## Retention and cleanup

| Output | Policy |
| --- | --- |
| `Baking/` | One reusable working tree; no copy per check/release. Normal baking removes obsolete outputs within it, not old staging elsewhere. |
| `tmp/`, `/tmp/FOSquad/` | At task completion, remove only finished task-owned bulky staging/isolated bakes; retain compact reports/logs in repository `tmp/`. `/tmp` may be RAM-backed. |
| Generated distributions | Keep the active package and requested distributables; do not retain a full duplicate per release as evidence. |
| `/data/!backups/FOSquad/` | Keep two newest complete verified pre-deployment storage/package rollback sets plus pinned recovery sets. Prune older sets only during approved maintenance after verifying the deployment and retained backups. All backups belong here. |
| `Build/`, `Binaries/` | Keep needed native outputs; prune obsolete configurations only in a scoped cleanup, not every update. |
| `Cache/` | Identify the exact owner/contents and obtain approval before clearing an unused instance; expect recreation/redownload and possible loss of preferences. |

- Before removal, retain commands/results, revision/build/metadata identities and backup location in
  a small report. Review unpinned evidence after 30 days, without age-only deletion; pin unresolved work
  with a reason/review date. Unknown owners and earlier tasks require separate cleanup scope.
- Check resolved paths, working directories and open files. Protect active packages, stdout/logs,
  authored resources, `.git/`, `Engine/`, `TLA-Storage/`, SQLite WAL/SHM and recovery logs. Never follow
  symlinks into another tree, use `git clean -fdx`, or blanket-delete ignored directories.
- Under disk pressure, remove approved obsolete staging first. Complete inactive `Baking/` may then
  be removed with approval after all consumers finish; rebake before packaging/unpackaged use. Do not
  edit individual baked packs/hash markers or remove a live baking lock. Do not stop servers just to clean.
- These rules install no automatic cleaner and grant no deletion authority over unrelated data.
  Report actual bytes recovered and retained exceptions; documentation changes alone need no cleanup.

Measure first: `df -h . /tmp`, `du -sh Baking Build Binaries Cache tmp TLA-LAN TLA-Storage`,
`du -h -d 1 tmp /tmp/FOSquad`. Delete only exact verified paths, not broad wildcards.
