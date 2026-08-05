# Folia compatibility audit

## Targets and limits

This audit covers the server plugin compiled for Luminol/Folia 1.21.11 and Canvas/Folia 26.2. It is a static source review backed by unit tests and compilation against Paper API 1.21.11 and Paper API 26.2. It is not a runtime test on a real Luminol, Canvas, Paper, or Folia server. Canvas-specific behavior that differs from the Paper/Folia API used for compilation remains `uncertain` until deployment testing or the matching Canvas core source confirms it.

`folia-supported: true` only allows the plugin to load. It is not treated as proof of thread safety or functional completeness.

## Pre-modification findings

| Classification | Entry point and call chain | Current context and real owner | Boundary values | Termination and cleanup | Rules |
|---|---|---|---|---|---|
| `must_fix` | Client screen draw -> `ScreenRenderer.textureIdentifier` -> `TextureManager.register` | Minecraft client/render thread; the video backend owns the raw GL texture | Raw GL integer ID crossed into a Minecraft texture wrapper | The old raw-ID cache had no release path and survived screen cleanup, disconnect, resource reload, and client stop | 2, 8 |
| `must_fix` | `VideoQuad.cleanup` / `MpvVideoBackend.cleanupTexture` / `AbstractCameraPlayer.cleanup` -> GL texture deletion | Minecraft render context, MPV shared context, or camera framebuffer owner | Raw GL integer ID | The backend deleted the GL object, but the Minecraft wrapper and cached render layer remained registered | 8 |
| `suggested_fix` | Resource reload -> texture manager reload -> next screen render | Minecraft resource apply/client context | Resource identifiers and immutable render snapshots | No explicit external-texture registry invalidation existed | 8 |
| `defer` | `VideoPlayerPaperPlugin.onEnable` -> `FoliaScheduler.initialize` -> one-time `RegionizedServer` detection | Plugin enable/global lifecycle context | Cached boolean only | Scheduler owner is cleared in `onDisable`; the detected server model remains process-stable across plugin reloads | 1, 3 |
| `defer` | Plugin message -> cloned byte array -> `DataHolder.runStateForPlayer` -> entity scheduler -> `ServerPacketHandler.handle` | Plugin messaging callback hands off to the Player entity owner | Cloned packet bytes, receive timestamp | ByteBuf is released in `finally`; state tasks are cancelled and maps cleared on disable | 1, 6, 7, 8 |
| `defer` | Player join -> entity scheduler -> fixed-rate `DataHolder.updatePlayer` | Player entity scheduler owns Player, World, and Location reads | UUID and immutable `PlayerPosition` snapshot enter the locked state tables | Quit, entity retirement, disable, and epoch mismatch cancel the task and remove references | 2, 6, 7, 8 |
| `defer` | World discovery in entity context -> immutable `WorldDescriptor` -> async read -> global locked apply | World access occurs in the owner context; file I/O occurs on the async scheduler; mutation returns through the global state executor | Dimension string and Path snapshot, then parsed configuration data | Request IDs and lifecycle epochs reject stale completion; queues are flushed or cancelled on disable | 2, 6, 8 |
| `defer` | State mutation -> lazy serialized snapshot -> `WorldSaveQueue` -> async file writer | Locked global state captures the snapshot; async tasks own file I/O | Strings, paths, generation numbers, and serialized snapshots only | Per-world slots are bounded by active dimensions, coalesced, retried, flushed with a timeout, and cancelled on disable | 2, 6, 8 |
| `defer` | Residence flag/lifecycle event -> global delayed refresh -> per-player entity refresh | Plugin lifecycle/global context enumerates players; each permission read is handed to its Player entity owner | UUID/player task target and immutable permission contexts | Listeners and task handles are cancelled and unregistered in bridge shutdown | 1, 2, 6, 7, 8 |

## Implemented resource lifecycle

The client now assigns every active raw GL texture a process-local generation. An active raw ID reuses its registration, while release followed by OpenGL ID reuse receives a new identifier such as `videoplayer:external_texture/7/2`. The registry stores integers and generation values only; it does not retain Minecraft, Player, Entity, World, Inventory, backend, or GL wrapper objects.

The following owner cleanup paths release the external registration before deleting or rebuilding their textures:

- `VideoQuad.cleanup`
- `MpvVideoBackend.cleanup` and `MpvVideoBackend.cleanupTexture`
- `AbstractCameraPlayer.cleanup`
- `AbstractCameraPlayer.updateTexture` before framebuffer resize

Disconnect, protocol reset, client stop, and client resource reload clear all remaining registrations. The 1.21.11 renderer also clears related `RenderType` entries. The 26.2 renderer clears its immutable `FrameRenderSnapshot` so a submission cannot retain the previous generation.

## Post-modification three-pass review

### 1. Functionality and scope

- Plugin name, mod ID, main classes, public commands, permissions, configuration keys, packet channel, internal version, wire revision, and unrelated playback behavior remain unchanged.
- `-26.2` is a distribution filename suffix only. It is not added to the internal version or handshake token.
- The resource change is limited to external texture registration, release, reload, and the directly owned GL/framebuffer cleanup chain.

### 2. Folia and performance

- `FoliaScheduler` is initialized once early in `onEnable`, caches Folia detection, and uses official Paper/Folia scheduler APIs after detection.
- Business code contains no direct legacy `BukkitScheduler` scheduling entry. Legacy scheduler calls exist only inside the wrapper's non-Folia path.
- Entity and region wrapper delays are clamped to at least one tick.
- No synchronous `teleport` call exists. The plugin currently has no teleport operation requiring `teleportAsync`.
- No `Bukkit.isPrimaryThread()` check is used as a Folia ownership test.
- Player, World, and Location reads occur in the Player entity owner context. Cross-context state uses UUIDs, strings, paths, byte arrays, configuration objects, or immutable position/permission snapshots.
- File and network operations run on async workers. The bounded shutdown flush occurs only during plugin disable and does not hold `DataHolder.LOCK` while waiting.
- No region callback uses `Future.get`, `CompletableFuture.join`, a blocking cross-region wait, or a synchronous database/network/file operation.

### 3. Lifecycle and resources

- Player tracking, reload handshake, world save debounce/retry, Residence retry/cache refresh, native runtime, and yt-dlp task handles have explicit cancellation paths.
- `DataHolder` clears Player UUID/name state, world state, handshake state, persistence state, and task tables on disable.
- `ClientVersionTracker` uses concurrent collections and cancels per-player timeout tasks during shutdown or session replacement.
- `DataHolder` maps and sets remain under the single `DataHolder.LOCK`; this preserves atomic mutations spanning multiple tables and avoids concurrent iteration. They are not replaced independently with concurrent collections.
- Provider concurrency is bounded by `VideoProviders.RESOLUTION_LIMIT`. World save slots are coalesced per active dimension. Client external texture registrations now have owner release and global cleanup paths.
- Plugin and Residence listeners are unregistered on disable. Native and scheduler executors are stopped, and lifecycle epochs reject stale callbacks after reload.

## Unavailable Folia events

No listener for `PlayerRespawnEvent`, `PlayerTeleportEvent`, `PlayerChangedWorldEvent`, `WorldLoadEvent`, or `WorldUnloadEvent` exists in the Paper plugin source for either target. Therefore, there is no class/listener to list as using one of these unavailable events. This result is specific to the reviewed Luminol/Folia 1.21.11 and Canvas/Folia 26.2 targets and must be rechecked after a server API upgrade.

## Runtime verification still required

Deployment verification should cover connection and handshake, screen creation and deletion, MPV/VLC playback, camera and 360 rendering, resource reload, disconnect/reconnect, plugin disable/reload, world changes discovered through player tracking, and concurrent players in separate Folia regions. Any conclusion that conflicts with the exact target server core is superseded by that core and should be recorded as a version-specific difference.
