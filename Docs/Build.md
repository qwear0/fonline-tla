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
separately. Sierra siege resets; saved characters remain. This does not refresh a separate client archive.
**Never run `MakePackage-LAN` against a running package: that target deletes `TLA-LAN/`.**

## Retention and cleanup

| Output | Policy |
| --- | --- |
| `Baking/` | One reusable working tree; no copy per check/release. Normal baking removes obsolete outputs within it, not old staging elsewhere. |
| `tmp/`, `/tmp/FOSquad/` | Mandatory closeout: remove finished disposable staging, isolated bakes and runtime trees; retain compact reports/logs only. Include verified inactive generated leftovers from completed earlier tasks under the standing authority in `AGENTS.md`. `/tmp` may be RAM-backed. |
| Generated distributions | Keep the active package and requested distributables; do not retain a full duplicate per release as evidence. |
| `/data/!backups/FOSquad/` | Keep two newest complete verified pre-deployment storage/package rollback sets plus pinned recovery sets. Prune older sets only during approved maintenance after verifying the deployment and retained backups. All backups belong here. |
| `Build/`, `Binaries/` | Keep needed native outputs; prune obsolete configurations only in a scoped cleanup, not every update. |
| `Cache/` | Identify the exact owner/contents and obtain approval before clearing an unused instance; expect recreation/redownload and possible loss of preferences. |

### Mandatory task closeout

**Run this checklist before every final response or handoff, not only after a build or when storage is
full.** Documentation-only tasks must account for any scratch files/processes they created too. If
there is nothing disposable, say so; do not manufacture cleanup work or launch a build/server for it.
The standing authority is in [AGENTS.md](../AGENTS.md#mandatory-task-closeout), not a blanket permission
to delete everything under a temporary directory.

1. **Inventory and classify.** Track the exact temporary paths and process identities you create.
   At closeout inspect `tmp/`, `/tmp/FOSquad/` and your launched processes. Check for completed earlier
   task leftovers as well: release copies, isolated bakes, compiler outputs, expanded diagnostic clients
   and disposable environments. A temporary-looking name or old timestamp alone is not proof of
   ownership/completion. Verify against task reports, invocation/output records and current consumers.
2. **Measure and preserve compact evidence.** Before substantial deletion, record exact candidate paths,
   sizes, commands/results, revision/build/metadata identities and existing backup locations in a small
   report under repository `tmp/`. Keep useful logs/manifests rather than an entire package or bake.
   Review unpinned evidence after 30 days, without age-only deletion; pin unresolved evidence with a
   reason/review date. That review period does not justify keeping redundant multi-gigabyte output trees.
3. **Release disposable processes.** If a task was explicitly authorized to launch a diagnostic server,
   client, virtual display or helper, stop only that exact task-owned process after use and confirm exit.
   Recheck command, working directory and executable: a saved PID may be stale/reused. Then remove its
   disposable runtime/cache/session directory and stale PID files. Do not leave a Memory server or its
   files behind merely because it was called a smoke check. This rule grants no permission to start
   server tests. Do not stop a deployed LAN/HTTP service, user-requested persistent process or another
   agent's job; a requested background build stays active and must be reported as such.
4. **Recheck safety immediately before removal.** Resolve candidate paths without following directory
   symlinks; inspect active working directories, open/mapped files and nested mounts. Protect active
   packages/stdout/logs, authored resources, `.git/`, `Engine/`, `TLA-Storage/`, SQLite WAL/SHM and recovery
   logs. Keep the current canonical `Baking/`, needed `Build/`/`Binaries/`, installed SDK/toolchains and
   requested/served downloads. Installer caches are disposable only if retained tooling has no dependency
   on them. If a concurrent job changes identities or starts using a candidate, stop that deletion and
   reclassify it; do not cancel the job or restart a server to make cleanup possible.
5. **Remove verified disposable data now.** Delete exact inactive generated roots, including confirmed
   leftovers from completed earlier tasks, without asking the user to repeat the cleanup request.
   After a successful deployment, remove its redundant staging package once the active copy is verified
   and required rollback backups are in the designated backup root. A stopped diagnostic process, a
   pushed commit or a successful bake does not excuse leaving its disposable trees behind. Unknown
   ownership/use requires a separately agreed scope; preserve and report those paths. Never use
   `git clean -fdx`, broad deletion wildcards or blanket deletion of ignored directories.
6. **Verify and report closeout.** Confirm the selected paths are gone and disposable processes exited;
   remeasure space and check Git status without touching unrelated edits. The final response must state
   what was removed and measured bytes recovered (distinguish disk from RAM-backed `/tmp`), or explicitly
   that no disposable artifacts were created. Name retained bulky/active artifacts, why they remain and
   the condition for later cleanup. If cleanup is blocked, report the blocker instead of claiming full
   completion. Perform this check again after any final build/package operation creates new outputs.

This is a mandatory agent action, not an installed automatic cleaner. It grants no deletion authority
over unrelated data or backup sets. Routine closeout does not purge the canonical bake/build/cache;
under disk pressure, removing a complete inactive canonical `Baking/` still requires separate approval
and a subsequent rebake before use. Never edit individual baked packs/hash markers or remove a live
baking lock. Backups follow the retention table and require their own approved maintenance scope.

For substantial cleanup, measure first with `df -h . /tmp`,
`du -sh Baking Build Binaries Cache tmp TLA-LAN TLA-Storage` and
`du -h -d 1 tmp /tmp/FOSquad`; use a bounded inventory for a task that created no bulky outputs.
