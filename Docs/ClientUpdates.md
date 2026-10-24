# Windows client updates over LAN

## Ownership and package

The game uses FOnline's built-in updater, not a separate launcher or HTTP update feed.
`FOnlineTLA.exe` is the client host; `FOnlineTLA.dll` is the replaceable native runtime.
`Resources/*.fores` are the client resource packs. The host executable itself is not
self-updated; an incompatible host/runtime ABI requires a new client distribution.
See the engine's [runtime/updater contract](../Engine/Docs/en/explanation/runtime/client-updater.md).

- `TLA.fomain`, subconfig `LAN`: the development endpoint `192.168.1.168:4008`, TCP,
  the base development channel key and SQLite storage. This is not a public-server profile.
  `FO_DATABASE_SQLITE` is enabled by the game build. From the standard packaged server root,
  `Server.DbStorage = DbSQLite ../../TLA-Storage/LAN` stores accounts, characters and world data
  in `TLA-Storage/LAN/Storage.sqlite`, outside the disposable `TLA-LAN/` package. Den siege progress
  resets at every server start; the next confirmed login starts a fresh attempt with the saved character.
  Other profiles retain their base Memory setting unless explicitly overridden.
- `CMakeLists.txt`, package `LAN`: Windows win64 client and Linux x64 server, including
  the headless server. Keeping both entries together makes the engine packager require
  the Windows native-runtime payload rather than silently omit it.
- The packaged server owns `ServerResources/`, client `Resources/`, and
  `PlatformBinaries/Windows-win64/`. The latter contains the packaged runtime DLLs and
  matching PDB files. Its `.exe` is not a client update payload.
- The engine packager patches the same LAN client config into the distributed client
  and the server's downloadable DLLs. Do not copy an unprocessed build DLL into the feed.

## Build and assemble

Start with [FastBuild.md](FastBuild.md) to select the minimum adequate work. Resource-only updates can
reuse verified unchanged native outputs and package the server/feed alone; a separate Windows client
distribution is not needed for every restart. When native outputs need rebuilding, build matching
sources/configuration in both native build trees before packaging. For the local Linux build and an
already configured Windows cross-build, for example:

```sh
cmake --build Build/Auto --config RelWithDebInfo --target BakeResources TLA_Server TLA_ServerHeadless
cmake --build <windows-build-directory> --target TLA_Client
cmake --build Build/Auto --config RelWithDebInfo --target MakePackage-LAN
```

A Windows cross-build needs the SDK/LLVM environment from the engine's
[build-tool workspace instructions](../Engine/BuildTools/README.md).
`MakePackage-LAN` assembles existing outputs; it does not build the other platform
for you. It first **deletes `TLA-LAN/`**: never run it while a server is running from
that directory. For a later release, use the engine packager's CLI with a different
`-output tmp/<release>` staging root, then switch servers after approval. Build hashes
of the bake, server and Windows client/runtime must agree.
The resulting roots are `TLA-LAN/TLA-Client-LAN/` and
`TLA-LAN/TLA-Server-LAN-Linux/`. `TLA-LAN/` is a generated package, not source;
it is covered by the repository's `/TLA-**` ignore rule. For a separate staging root,
invoke the engine packager with `-output tmp/...`.
For redistribution, archive the complete client root including its DLL, PDBs and
`Resources/`; do not ship a source tree, a server secret, or another client's cache.

## Activate and use

1. Assemble a complete server package before stopping the old process. Do not replace
   packs underneath a running packaged server: its descriptors and open files were
   loaded at startup. Every feed publication requires a coordinated server restart.
2. When switching an older Memory server to SQLite, get explicit approval before stopping it:
   its in-memory accounts and progress disappear and are not imported into SQLite. Thereafter,
   keep the existing `TLA-Storage/LAN/` database when restarting or refreshing the package. Never
   run two servers against that same storage. Back up a stopped server's complete storage directory
   before replacing binaries or metadata; copying only a live SQLite file omits possible WAL data.
3. Stop the previous server gracefully and start `./TLA_ServerHeadless` **with the
   packaged server directory as its working directory**. Do not run the development
   `Binaries/Server-Linux-x64/TLA_ServerHeadless` as the updater backend: unpackaged
   mode deliberately skips its initialization. The package embeds `LAN`; no source
   `TLA.fomain` or `--ApplySubConfig` is needed. The relative database path assumes this exact
   directory depth. When launching a package from another staging root, explicitly pass
   `--Server.DbStorage 'DbSQLite /absolute/path/to/TLA-Storage/LAN'` so it opens the intended database.
4. Confirm the log reaches `Start server complete!` and includes `Load client data
   packs for synchronization`, not `Skip updater backend initialization in unpackaged mode`.
5. Extract the client archive into a new writable folder and run `FOnlineTLA.exe`
   directly. No batch launcher is needed. Updates use the same authenticated game
   connection on TCP port 4008. HTTP download hosting only distributes the initial ZIP.
6. Resource changes are synchronized before gameplay. A native update is staged
   when client/server network compatibility versions differ; when the built-in
   updater asks for a restart, close/relaunch the **client** to load the new DLL.
   This does not restart the server or erase its SQLite DB.

After successful activation, follow [DiskCleanup.md](DiskCleanup.md) for staging cleanup and rollback
retention. Do not retain a full staging copy as the only verification evidence or delete an active feed/log.

A changed DLL hash alone does not enter native-update mode. For a native-only release
whose generated compatibility remains unchanged, deliberately advance
`Network.ForceCompatibilityVersion` in the `LAN` subconfig on **both** packaged sides
(for example to a unique release tag), then rebake/repackage together. If used, that
override becomes a release contract: advance it for every subsequent native release,
including generated-contract changes; never leave incompatible builds on the same tag.
The current LAN release uses `tla-lan-20261024-merc-loot` to deliver the native equipment-snapshot
and AI-bridge shutdown fixes together with the mercenary command/loot resources. The prior
`tla-lan-20261023-equipment` tag must not be reused for this runtime. Advance this tag for
every later LAN native release; do not reuse it for a different native runtime. Other profiles
leave the override empty when the generated compatibility version is sufficient. The host/runtime
ABI and updater-protocol boundaries still apply and may require a fresh distribution.

If the LAN address changes, update the `LAN` subconfig, rebake and repackage both
sides. Use the existing `PublicGame` key provisioning for a public deployment;
the deliberately public development secret must never protect a production server.

## Verification and failure clues

- Compare the packaged server/client metadata version, not just Git or file dates.
  Old archives can pass native compatibility but still carry outdated gameplay metadata.
- A server package must contain the Windows target DLL and PDB files as well as
  client resources; native build success alone does not establish an update feed.
- Check `TLA_ServerHeadless.log`, `FOnlineTLA.log` and `FOnlineTLA.prev.log` for the
  handshake, resource synchronization and native-update result. The client retains
  one previous run's log.
- Windows GUI, actual resource downloads and DLL replacement need direct client
  observations. Packaging and a server startup do not prove those client-side paths.
- Existing SQLite storage is reopened on restart; logs must show `Connect to DbSQLite data base`
  and `Restore world`, rather than `Connect to Memory data base` and a fresh `Generate world`.
  Den startup reset is logged separately; it does not clear accounts or rebuild the world.
- No serialized entity properties or migration rules change with this storage selection. Do not
  point current binaries at obsolete SQLite formats; see the engine's
  [storage upgrade boundary](../Engine/Docs/en/explanation/persistence/index.md#storage-formats-and-upgrade-boundary).
