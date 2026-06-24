# TLBB Codebase Overview

## Project Topology
- `Common/`: shared engine-agnostic code used by both servers and client; defines data schemas, network packets, utilities, and share-memory APIs.
- `Server/`: multi-service backend solution (Login, World, Game, Billing, ShareMemory) plus a `Common/` mirror tuned for server-only helpers.
- `Game/`: Windows client, proprietary engine (`WXEngine`), game runtime (`WXClient`), scripting, tooling, and bundled third-party libraries (OGRE, CEGUI, wxWidgets, DirectX, FMOD, LuaPlus, zlib, etc.).
- `_Pre-Built/`: binary dependencies delivered with the tree.

## Shared Foundation (`Common/`)
- **Game definitions:** `GameDefine*.h`, `GameStruct*.h`, and `GameUtil` encode core gameplay enums, stat blocks, item schemas, pet data, guild/city models, and helper utilities. These headers are tightly coupled to save formats and database rows.
- **Network layer:** `Net/` supplies cross-platform sockets, buffered streams, and the abstract `Packet` base. Packets live in `Packets/`, organised by prefix: `CG*` (client -> game), `GC*` (game -> client), `GW*` (game -> world), `WG*` (world -> game), `LB/LW/LC` (login/billing). Every packet implements `Read`, `Write`, and `Execute` to keep encode/decode logic close to message handling.
- **Database system:** `DBSystem/DataBase` wraps ODBC via `ODBCInterface`, rich request structs (e.g., `DBCharFullData`, `DBAbilityList`), and managers that translate between SQL schemas and in-memory `GameStruct` models.
- **Share memory:** `ShareMemAPI` and `SMUPool` offer cross-process shared-memory buffers. Servers use these to publish replicated state (player shops, rankings, mail queues) without constant DB round-trips.
- **Combat and gameplay helpers:** `Combat/`, `BuffImpactMgr`, `SkillDataMgr`, and related managers centralise skill templates, buff stacking, and damage resolution logic consumed by multiple services.

## Server Stack (`Server/Server`)
The backend is split into cooperating executables, each with its own solution/vcxproj. All services depend on `Server/Common` for logging, config parsing, threading, and packet factories.

- **LoginServer:** Authenticates users, enumerates characters, and brokers initial hand-off. `Main/LoginMain.cpp` instantiates `g_Login`, which drives `ConnectManager`, DB tasks, and packet handlers under `Login/`.
- **WorldServer:** Holds global state: player directory (`OnlineUser`, `AllUser`), chat, guilds (`GuildManager`), cities, mails, cross-scene matchmaking, and world timers. `World/Main/World.cpp` initialises managers, loads configuration tables (`WorldTable`, `SceneInfo`), and coordinates downstream GameServers.
- **GameServer:** Heavy-lift gameplay simulation. The `Server/` folder is partitioned into subsystems: `ActionModule`, `AI`, `Scene`, `Player`, `Mission`, `Skills`, etc. `Main/Server.cpp` wires up singletons such as `SceneManager`, `ThreadManager`, `ClientManager`, `ItemManager`, and `DaemonThread`, then enters the frame loop. GameServers own real-time scenes, physics, combat resolution, and client session management.
- **ShareMemory service:** Keeps persistent shared-memory segments alive, handles command input, and bridges to the DB (`ShareMemory/ShareData`, `ShareDBManager`). Critical for features like player stalls or offline mail that require durability but also low-latency access from multiple processes.
- **BillingServer:** Minimal transaction processor used when the game integrates with an external billing/login provider. Handles packet families `BW*`, `WB*`, and DB writes for account status.
- **Common infrastructure:** `Server/Common/ServerBase` implements logging (`Log`), INI config, thread abstractions, and a simple script bridge (FoxLua). `Server/Common/Packets` mirrors the shared packet catalogue but compiles with server-specific handlers. The `SMU` folder exposes helpers for integrating share-memory units inside the gameplay loop.

### Runtime Flow
1. Client connects to **LoginServer** to authenticate and fetch character lists (`LCRet*` packets).
2. After selection, LoginServer introduces the client to the **WorldServer** (`LWAsk*`, `WLRet*`).
3. WorldServer assigns a **GameServer** instance based on scene/shard load and replies with connection credentials (`WGRet*`, `GWAsk*`).
4. GameServer runs the session, exchanging high-frequency `CG*/GC*` traffic, while synchronising cross-scene/cross-feature data back to WorldServer and ShareMemory (`GW*/WG*`, `SS*`).
5. Periodic persistence tasks write through the **DBSystem** or publish to ShareMemory for asynchronous save workers.

## Database & Persistence
- Uses Microsoft ODBC (`ODBCInterface`) with SQL templates in `SqlTemplate.*`.
- Each logical data set (equipment, bag, cooldowns, abilities, missions, settings) has a paired request/response struct mirroring the DB schema; transactions are orchestrated through manager classes (`DBManager`, `ShareDBManager`).
- Share-memory units (`HumanSMU`, `PlayerShopSMU`, etc.) act as fast caches. Managers such as `SMUManager`, `ShareMemManager`, and `ShareMemAO` handle attach/detach, locking, and dumping to disk for recovery.

## Networking & Packet Handling
- Packet IDs are defined in `PacketDefine.h`; headers store both logical type and payload length using helper macros (`GET_PACKET_INDEX`, `SET_PACKET_LEN`, etc.).
- Server-side packet handling is spread across subsystem-specific directories (e.g., `GameServer/Server/Packets`, `World/Packets`, `Login/Packets`). Each handler interprets the request and delegates to gameplay managers.
- The client mirrors this layout under `WXClient/Network/PacketHandler`, making it easy to cross-reference client expectations with server behaviour.
- Connection management is abstracted by `ServerSocket`, `SocketInputStream`, and `SocketOutputStream`, with platform-specific glue inside `SocketAPI`.

## Client Stack (`Game/Client`)
- **WXClient:** The actual game executable. Entry point `WXClient.cpp` boots global state, selects the active `CGameProcedure`, and runs the main loop. Modular subsystems live under subfolders (e.g., `Action`, `Interface`, `Sound`, `World`). `Procedure/` implements a state-machine for login, character select, scene transitions, and gameplay.
- **WXEngine:** Proprietary rendering and systems framework, wrapping DirectX 9. Key components include `Gfx` (render device and post-processing), `UI` (CEGUI integration), `Input`, `Sound` (FMOD), `Script` (LuaPlus), `ResourceProvider`, and `Kernel` (core loop, time, plugin loading).
- **Common client framework:** `Game/Client/Common` (`TD*` headers) standardises math, profiling, object hierarchies, input dispatch, and resource management shared by both WXClient and engine modules.
- **Networking:** `WXClient/Network/NetManager` owns sockets and dispatches packets to the handler catalogue that mirrors server message types. Client also loads packet factories from `WXNetPackets`.
- **Data & assets:** `DataPool`, `DBC`, and `Res` manage configuration tables, asset caches, and resource lookups. Script bindings under `Script/` expose gameplay logic to Lua.
- **User experience:** `Interface/` covers HUD windows, dialog flows, and UI event wiring. `Procedure/GamePro_*` orchestrates transitions between login, character creation, and in-world states.

## Tooling and Auxiliary Projects
- `SceneEdit/`: in-house world editor leveraging OGRE/wxWidgets for editing terrain, quests, and placements.
- `Helper/`, `Debuger/`, `CrashReport/`: utilities for profiling, debugging, and crash dump handling on the client.
- Bundled middleware (`FreeType`, `Opcode`, `ParticleFX2`, `SndShell`, `wxDockIt`, etc.) sits alongside the client sources to simplify building without external downloads.

## Build & Configuration Notes
- Solutions target Windows (Visual Studio VC8/VC9 projects are present). Some server code supports Linux builds guarded by `__LINUX__` macros, but Windows paths, registry calls, and Win32 APIs are prevalent.
- Runtime configuration relies on INI files parsed by `ServerBase/Ini` and `WXSystem.cfg`. Logs are emitted per service (`./Log/*.log`).
- Comments are predominantly Chinese; some characters render garbled unless the editor uses GB2312/CP936 encoding.

## Getting Oriented
- Start from the packet definitions in `Common/Packets` to understand gameplay features and cross-server responsibilities.
- Trace a feature end-to-end by matching client handlers (`WXClient/Network/PacketHandler/CG*`) with server counterparts (`GameServer/Server/Packets/CG*Handler.cpp`).
- For persistence-heavy systems (shops, mail, guilds), inspect the corresponding SMU manager plus DB task in `DBSystem`.
- When modifying gameplay flow, audit both the WorldServer coordinator and GameServer scene logic to keep state in sync.
