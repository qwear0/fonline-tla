# Generated files and disk cleanup

Owner: game workspace operations. Read together with [FastBuild.md](FastBuild.md) and
[ClientUpdates.md](ClientUpdates.md). These are retention rules, not a scheduled cleanup service or
permission to delete unrelated user/task data. No automatic cleaner is installed by this document.

## What Baking is

`TLA.fomain` selects `Baking/` as `Baking.BakeOutput`. The baker turns authored scripts, dialogs,
text, prototypes, maps and assets into runtime-ready resources there. `package.py` reads that output
and assembles compressed `.fores` archives and patched binaries for distribution. Unpackaged tools
and applications also consume the baked resources.

Baking is **regenerable output, not character storage or a backup**. Decoded/processed graphics can
be much larger than their packaged archives. Keep one reusable working bake for incremental builds;
do not copy the entire tree for every verification. Deleting it saves space once but makes the next
bake expensive again. Successful normal baking already removes resources no baker claims; it does
not clean old staging directories outside its output.

## Retention policy

| Location | Keep / cleanup rule |
| --- | --- |
| `Baking/` | Keep one current working output. Do not archive each bake. A task-specific isolated bake is disposable after its consumers finish and necessary evidence is retained. |
| Repository `tmp/` | Keep compact plans, reports and verification logs. Remove task-owned bulky staged packages, copied resource trees and disposable outputs at task completion, after successful activation/verification or abandonment. |
| `/tmp/FOSquad/` | Remove task-owned disposable files and isolated outputs when the task ends and no process uses them. `/tmp` can be RAM-backed; do not use it as a long-term asset/package store. |
| `TLA-LAN/` and other generated distributions | Keep the active package and any specifically requested distributable. Do not accumulate a full duplicate for every release. Never delete a running/served package. |
| `/data/!backups/FOSquad/` | Default rollback retention: the two newest complete, verified pre-deployment storage/package sets. Preserve explicitly pinned incident/recovery sets. Prune older sets only during approved maintenance after confirming the new deployment and retained backups; never infer deletion rights from age/name alone. |
| `Build/`, `Binaries/` | Keep native outputs needed for the active Linux server and supported client update feed. Remove obsolete configurations/build trees only in an explicitly scoped cleanup; routine updates must not force costly native rebuilds. |
| `Cache/` or a client's own cache | Regenerable only after identifying the exact owner and contents. Clear an unused instance only with approval; expect redownload/recreation and possible loss of local preferences. Never treat every directory named Cache as disposable. |

Review unpinned evidence older than 30 days; age triggers review, not blind deletion. Keep unresolved
investigation evidence until resolution or explicit handoff. Pin retained exceptions with a reason
and review date in a compact report. Backups belong only under `/data/!backups/FOSquad/`, not `tmp/`,
`Baking/` or another full copy beside the active package.

## Finish each build/deployment task

1. Confirm the result first: successful bake/package, and successful startup/storage restore for a
   deployment. Keep the previous rollback package and a complete stopped-storage backup before activation.
2. Retain commands, exit status, relevant logs, revision/build/metadata identities and backup location
   in a small `tmp/<task>/` report. A multi-gigabyte resource tree is not verification evidence by itself.
3. Remove only this task's disposable artifacts whose consumers are finished. If a staged package was
   moved into the active location, remove only the leftover staging material, not its new active target.
   Keep any log still used by a running process; do not unlink the directory holding its stdout.
4. Inspect older large directories and propose exact cleanup scope separately. Earlier tasks, unknown
   owners, unknown contents and user edits are not covered by routine task cleanup.
5. Report bytes recovered and material retained. Do not claim cleanup happened when only policy was added.

## Safety and disk-pressure recovery

- Never clean authored `Resources/`, `Scripts/`, `Gui/`, maps/prototypes/texts, `.git/`, the external
  `Engine/` checkout, or persistent `TLA-Storage/` as generated output. Packaged `Resources/` inside an
  active server/client root is also protected despite its similar name.
- Never delete SQLite databases, `-wal`/`-shm` companions or recovery operation logs as cache. Back up
  the complete stopped storage directory; do not copy just a live `Storage.sqlite`.
- Check exact resolved paths, process working directories and open files before deletion. Package/tool
  use of a directory can outlast the command that created it. Do not follow symlinks into another tree.
- Under pressure, remove approved obsolete staging copies first. If that is insufficient, an approved
  cleanup may remove the complete inactive `Baking/` output after all its consumers finish; the next
  successful `BakeResources` must recreate it before unpackaged use or packaging. Do not selectively
  remove packs/hash markers, edit baked files, or force-remove a live baking lock to manufacture validity.
- Do not use `git clean -fdx`, blanket `rm -rf tmp/*`, or an age-only recursive deletion rule. Git-ignored
  does not mean unused or unowned. Never stop a live server solely to satisfy a cleanup target without approval.

Start with read-only measurements (from the game root):

```sh
df -h . /tmp
du -sh Baking Build Binaries Cache tmp TLA-LAN TLA-Storage /tmp/FOSquad /data/\!backups/FOSquad
du -h -d 1 tmp /tmp/FOSquad
```

No broad deletion command is prescribed: select exact, verified paths and authorization first.
