# Mirror Networking Library — Agent SKILLS Document

> **Version**: Auto-generated from source analysis
> **Repository**: NEXTINDIE/Mirror
> **Root**: `Assets/Mirror/`
> **Runtime**: Unity (C#, IL Weaver post-processor)

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Directory Map](#2-directory-map)
3. [Core Systems](#3-core-systems)
4. [Transport Layer](#4-transport-layer)
5. [Serialization & Messages](#5-serialization--messages)
6. [Synchronized State (SyncVar / SyncCollections)](#6-synchronized-state)
7. [Remote Procedure Calls (Command / ClientRpc / TargetRpc)](#7-remote-procedure-calls)
8. [Network Transform & Rigidbody Sync](#8-network-transform--rigidbody-sync)
9. [Client-Side Prediction](#9-client-side-prediction)
10. [Interest Management (AOI)](#10-interest-management-aoi)
11. [Snapshot Interpolation & Time Sync](#11-snapshot-interpolation--time-sync)
12. [Lag Compensation](#12-lag-compensation)
13. [Authentication](#13-authentication)
14. [IL Weaver (Code Generation)](#14-il-weaver-code-generation)
15. [Components Reference](#15-components-reference)
16. [Utilities & Tools](#16-utilities--tools)
17. [Profiling & Diagnostics](#17-profiling--diagnostics)
18. [Network Discovery](#18-network-discovery)
19. [Editor Tooling](#19-editor-tooling)
20. [Examples Catalog](#20-examples-catalog)
21. [Tests](#21-tests)
22. [Coding Conventions](#22-coding-conventions)
23. [Common Patterns & Recipes](#23-common-patterns--recipes)
24. [Key Type Quick-Reference](#24-key-type-quick-reference)

---

## 1. Architecture Overview

Mirror is a **high-level networking library for Unity** (MMO-scale capable). It follows a **Server-Authoritative** model with optional **Client Authority** for specific objects.

### Core Concepts

| Concept | Description |
|---------|-------------|
| **Host Mode** | Server + Client running in the same Unity process via `LocalConnection` pairs |
| **Server Authority** | Server owns game state; clients send `[Command]`s, server broadcasts `[ClientRpc]`s |
| **Client Authority** | Per-object opt-in via `SyncDirection.ClientToServer`; client sends state, server relays |
| **IL Weaving** | Mono.Cecil post-processes compiled DLLs at compile-time to inject serialization, RPC stubs, and SyncVar accessors |
| **Transport Abstraction** | Pluggable transport layer (KCP/UDP, Telepathy/TCP, SimpleWeb/WebSocket, Encryption, Multiplex) |
| **Interest Management** | Server-side visibility culling so clients only receive data for nearby/relevant objects |
| **Snapshot Interpolation** | Client-side buffering + interpolation of remote entity states for smooth movement |
| **Prediction** | Client-side physics prediction with server reconciliation for responsive gameplay |

### Data Flow

```
Client                              Server
  │                                   │
  │── [Command] ──────────────────▶   │  (client→server RPC)
  │                                   │── process + validate
  │   ◀────────────── [ClientRpc] ──  │  (server→all clients)
  │   ◀────────────── [TargetRpc] ──  │  (server→one client)
  │   ◀───── SyncVar / SyncList ────  │  (automatic state sync)
  │   ◀──────── SpawnMessage ───────  │  (object spawning)
  │                                   │
  │── TimeSnapshot ──▶ ◀──────────    │  (time synchronization)
```

---

## 2. Directory Map

```
Assets/Mirror/
├── Core/                          # Networking core (static classes, base types)
│   ├── Batching/                  # Message batching (Batcher, Unbatcher)
│   ├── LagCompensation/           # Server-side hit rewind
│   ├── Prediction/                # Physics prediction interfaces
│   ├── SnapshotInterpolation/     # Client interpolation system
│   ├── Threading/                 # Thread-safe utilities (ThreadLog, ConcurrentPool)
│   └── Tools/                     # Helpers (Pool, Compression, DeltaCompression, Extensions)
│       └── CustomStructs/         # Half, Vector2Byte, Vector3Long, etc.
├── Components/                    # MonoBehaviour components
│   ├── NetworkTransform/          # Position/rotation/scale sync (Reliable, Unreliable, Hybrid)
│   ├── NetworkRigidbody/          # Physics sync (3D + 2D variants)
│   ├── PredictedRigidbody/        # Client-side prediction system
│   ├── Discovery/                 # LAN server discovery (UDP multicast)
│   ├── InterestManagement/        # AOI systems (Distance, Scene, Match, Team, SpatialHashing, Hex)
│   ├── LagCompensation/           # HistoryCollider, LagCompensator components
│   └── Profiling/                 # Bandwidth/Ping/FPS graphs, RuntimeProfiler
├── Transports/                    # Transport implementations
│   ├── KCP/                       # KCP over UDP (default, recommended)
│   ├── Telepathy/                 # TCP transport
│   ├── SimpleWeb/                 # WebSocket transport (WebGL compatible)
│   ├── Multiplex/                 # Run multiple transports simultaneously
│   ├── Latency/                   # Latency/jitter/loss simulation
│   ├── Middleware/                 # MiddlewareTransport base (decorator pattern)
│   ├── Encryption/                # AES256-GCM encrypted transport
│   ├── Edgegap/                   # Edgegap relay + lobby integration
│   └── Threaded/                  # ThreadedTransport base for worker threads
├── Editor/                        # Unity Editor scripts
│   └── Weaver/                    # IL post-processor (Mono.Cecil)
├── Authenticators/                # Auth implementations (Basic, Device, Timeout, UniqueName)
├── Plugins/                       # Mono.CecilX DLLs (for Weaver)
├── CompilerSymbols/               # Auto-adds MIRROR define symbols
├── Presets/                       # NetworkTransform presets
├── Hosting/                       # Edgegap hosting integration
├── Examples/                      # Demo scenes (38 examples)
└── Tests/                         # Unit & integration tests
```

---

## 3. Core Systems

### 3.1 NetworkManager (`Core/NetworkManager.cs`)

**Class**: `NetworkManager : MonoBehaviour`

Central entry point. Manages lifecycle, scenes, player spawning, and configuration.

| Property / Method | Type | Description |
|-------------------|------|-------------|
| `singleton` | static | Global NetworkManager instance |
| `transport` | Transport | Active transport reference |
| `networkAddress` | string | Server address for client connection |
| `maxConnections` | int | Max simultaneous clients |
| `sendRate` | int | Reliable messages per second (default 30) |
| `unreliableBaselineRate` | int | Unreliable baseline per second (default 1) |
| `unreliableRedundancy` | int | Unreliable redundancy count |
| `offlineScene` / `onlineScene` | string | Auto scene management |
| `playerPrefab` | GameObject | Default player prefab |
| `autoCreatePlayer` | bool | Auto-spawn player on connect |
| `spawnPrefabs` | List\<GameObject\> | Registered spawnable prefabs |
| `StartServer()` | void | Start as dedicated server |
| `StartClient()` | void | Start as client and connect |
| `StartHost()` | void | Start as host (server + client) |
| `StopServer()` / `StopClient()` / `StopHost()` | void | Shutdown |
| `OnServerConnect(conn)` | virtual | Override: client connected |
| `OnServerDisconnect(conn)` | virtual | Override: client disconnected |
| `OnClientConnect()` | virtual | Override: connected to server |
| `OnClientDisconnect()` | virtual | Override: disconnected from server |
| `ServerChangeScene(sceneName)` | void | Change scene for all clients |

**Enums**: `PlayerSpawnMethod { Random, RoundRobin }`, `NetworkManagerMode { Offline, ServerOnly, ClientOnly, Host }`, `HeadlessStartOptions`

### 3.2 NetworkServer (`Core/NetworkServer.cs`)

**Class**: `NetworkServer` (static partial)

Server-side state management, connection tracking, object spawning, and message routing.

| Property / Method | Type | Description |
|-------------------|------|-------------|
| `active` | static bool | Server is running |
| `connections` | static Dict\<int, NetworkConnectionToClient\> | All client connections |
| `spawned` | static Dict\<uint, NetworkIdentity\> | All spawned objects by netId |
| `localConnection` | static NetworkConnectionToClient | Host mode local connection |
| `maxConnections` | static int | Connection limit |
| `tickRate` | static int | Server tick rate |
| `sendRate` / `sendInterval` | static int/double | Send frequency |
| `Listen(maxConns)` | static void | Start listening |
| `Shutdown()` | static void | Stop server |
| `Spawn(obj, conn?)` | static void | Spawn object (optionally with owner) |
| `Destroy(obj)` | static void | Destroy networked object |
| `Unspawn(obj)` | static void | Unspawn without destroying |
| `AddPlayerForConnection(conn, obj)` | static void | Assign player to connection |
| `ReplacePlayer(conn, obj, options)` | static void | Replace player object |
| `RemovePlayer(conn, options)` | static void | Remove player |
| `SendToAll(msg, channel?)` | static void | Broadcast to all clients |
| `SendToReady(msg, channel?)` | static void | Send to ready clients |
| `RebuildObservers(identity, init)` | static void | Refresh visibility |
| `RegisterHandler<T>(handler)` | static void | Register message handler |

**Enums**: `ReplacePlayerOptions`, `RemovePlayerOptions`

### 3.3 NetworkClient (`Core/NetworkClient.cs`)

**Class**: `NetworkClient` (static partial)

Client-side connection, entity observation, and message handling.

| Property / Method | Type | Description |
|-------------------|------|-------------|
| `active` | static bool | Client is connected/connecting |
| `isConnected` | static bool | Fully connected |
| `connection` | static NetworkConnectionToServer | Server connection |
| `localPlayer` | static NetworkIdentity | Local player object |
| `spawned` | static Dict\<uint, NetworkIdentity\> | Observed objects |
| `ready` | static bool | Client is ready |
| `Connect(address)` | static void | Connect to server |
| `Disconnect()` | static void | Disconnect |
| `Ready()` | static void | Signal ready to server |
| `AddPlayer()` | static void | Request player spawn |
| `RegisterHandler<T>(handler)` | static void | Register message handler |
| `Send<T>(msg, channel?)` | static void | Send message to server |

**Enums**: `ConnectState { None, Connecting, Connected, Disconnected }`

### 3.4 NetworkIdentity (`Core/NetworkIdentity.cs`)

**Class**: `NetworkIdentity : MonoBehaviour` (sealed)

The network identity component — one per networked GameObject.

| Property | Type | Description |
|----------|------|-------------|
| `netId` | uint | Unique network ID (assigned at spawn) |
| `sceneId` | ulong | Scene object ID (for scene objects) |
| `isServer` / `isClient` / `isHost` | bool | Runtime context flags |
| `isLocalPlayer` | bool | This is the local player |
| `isOwned` | bool | This client owns this object |
| `connectionToClient` | NetworkConnectionToClient | Owner connection (server-side) |
| `observers` | Dictionary\<int, NetworkConnectionToClient\> | Who can see this object |
| `visibility` | Visibility | `Default`, `ForceHidden`, `ForceShown` |
| `networkBehaviours` | NetworkBehaviour[] | Child behaviours |

**Lifecycle Callbacks**: `OnStartServer()`, `OnStopServer()`, `OnStartClient()`, `OnStopClient()`, `OnStartLocalPlayer()`, `OnStopLocalPlayer()`

### 3.5 NetworkBehaviour (`Core/NetworkBehaviour.cs`)

**Class**: `NetworkBehaviour : MonoBehaviour` (abstract)

Base class for all networked game logic components.

| Property | Type | Description |
|----------|------|-------------|
| `syncDirection` | SyncDirection | `ServerToClient` or `ClientToServer` |
| `syncMode` | SyncMode | `Observers` (all viewers) or `Owner` (only owner) |
| `syncMethod` | SyncMethod | `Reliable` or `Hybrid` |
| `syncInterval` | float | Min seconds between syncs (0 = every frame) |
| `isServer` / `isClient` / `isHost` | bool | Runtime context flags |
| `isLocalPlayer` / `isOwned` | bool | Ownership flags |
| `netId` / `netIdentity` | uint / NetworkIdentity | Identity reference |
| `connectionToClient` | NetworkConnectionToClient | Owner connection |

**Virtual Callbacks**: `OnStartServer()`, `OnStopServer()`, `OnStartClient()`, `OnStopClient()`, `OnStartLocalPlayer()`, `OnStopLocalPlayer()`, `OnStartAuthority()`, `OnStopAuthority()`

**Enums**: `SyncDirection { ServerToClient, ClientToServer }`, `SyncMode { Observers, Owner }`, `SyncMethod { Reliable, Hybrid }`

### 3.6 NetworkConnection (`Core/NetworkConnection.cs` + variants)

| Class | Role | Key Properties |
|-------|------|----------------|
| `NetworkConnection` (abstract) | Base connection | `isAuthenticated`, `isReady`, `identity`, `owned`, `lastMessageTime` |
| `NetworkConnectionToClient` | Server → Client | `connectionId`, `address`, `rtt`, `observing`, `remoteTimeline` |
| `NetworkConnectionToServer` | Client → Server | Routes sends to `Transport.active.ClientSend()` |
| `LocalConnectionToClient` | Host mode (server side) | In-process queue, no real transport |
| `LocalConnectionToServer` | Host mode (client side) | In-process queue, no real transport |

---

## 4. Transport Layer

### 4.1 Transport Base Class (`Core/Transport.cs`)

**Class**: `Transport : MonoBehaviour` (abstract)

| Member | Description |
|--------|-------------|
| `active` (static) | Current transport instance |
| `Available()` | Platform availability check |
| **Client Events** | `OnClientConnected`, `OnClientDataReceived`, `OnClientDisconnected`, `OnClientError` |
| **Server Events** | `OnServerConnectedWithAddress`, `OnServerDataReceived`, `OnServerDisconnected`, `OnServerError` |
| **Client Methods** | `ClientConnect(address)`, `ClientSend(data, channel)`, `ClientDisconnect()` |
| **Server Methods** | `ServerStart()`, `ServerSend(connId, data, channel)`, `ServerDisconnect(connId)`, `ServerStop()` |
| `GetMaxPacketSize(channel)` | Max message size for channel |
| `Shutdown()` | Full cleanup |

**Interface**: `PortTransport` — `Port { get; set; }` for transports using a network port.

**Enum**: `TransportError { DnsResolve, Refused, Timeout, Congestion, InvalidReceive, InvalidSend, ConnectionClosed, Unexpected }`

### 4.2 Transport Implementations

| Transport | Directory | Protocol | Default Port | Key Features |
|-----------|-----------|----------|--------------|--------------|
| **KcpTransport** | `Transports/KCP/` | KCP over UDP | 7777 | Fast, game-optimized; NoDelay, configurable window/timeout; **Recommended** |
| **ThreadedKcpTransport** | `Transports/KCP/` | KCP over UDP | 7777 | Multi-threaded variant for high-CCU |
| **TelepathyTransport** | `Transports/Telepathy/` | TCP | 7777 | Reliable/ordered; Nagle toggle; simple fallback |
| **SimpleWebTransport** | `Transports/SimpleWeb/` | WebSocket (ws/wss) | 27777 | **WebGL compatible**; SSL/TLS support |
| **EncryptionTransport** | `Transports/Encryption/` | AES256-GCM | (wraps inner) | Wraps any transport with encryption; RSA key exchange |
| **MultiplexTransport** | `Transports/Multiplex/` | Multiple | (varies) | Listen on multiple transports simultaneously |
| **LatencySimulation** | `Transports/Latency/` | (wraps inner) | — | Simulates latency, jitter, packet loss, scrambling |
| **EdgegapKcpTransport** | `Transports/Edgegap/` | Relay/UDP | — | Edgegap cloud relay + lobby services |

### 4.3 Middleware Pattern

`MiddlewareTransport` (abstract) wraps an `inner` transport via decorator pattern. Used by:
- `EncryptionTransport` — adds encryption layer
- `LatencySimulation` — adds simulated network conditions

### 4.4 Threaded Transport

`ThreadedTransport` (abstract) elevates transport work to a background thread. Used by:
- `ThreadedKcpTransport` — KCP on worker thread for high performance

---

## 5. Serialization & Messages

### 5.1 NetworkWriter / NetworkReader (`Core/NetworkWriter.cs`, `Core/NetworkReader.cs`)

High-performance binary serializers with auto-resizing buffers.

| Writer Method | Reader Method | Type |
|---------------|---------------|------|
| `WriteByte()` | `ReadByte()` | byte |
| `WriteInt()` | `ReadInt()` | int |
| `WriteFloat()` | `ReadFloat()` | float |
| `WriteString()` | `ReadString()` | string (UTF-8, length-prefixed) |
| `WriteBool()` | `ReadBool()` | bool |
| `WriteVector3()` | `ReadVector3()` | Vector3 |
| `WriteQuaternion()` | `ReadQuaternion()` | Quaternion |
| `WriteNetworkIdentity()` | `ReadNetworkIdentity()` | NetworkIdentity (by netId) |
| `WriteNetworkBehaviour<T>()` | `ReadNetworkBehaviour<T>()` | NetworkBehaviour (by netId + index) |
| `WriteVarInt()` | `ReadVarInt()` | Variable-length int |
| `WriteVarUInt()` | `ReadVarUInt()` | Variable-length uint |
| `WriteList<T>()` | `ReadList<T>()` | List\<T\> |
| `WriteArray<T>()` | `ReadArray<T>()` | T[] |
| `WriteBlittable<T>()` | `ReadBlittable<T>()` | Any unmanaged struct |
| `WriteNetworkMessage<T>()` | `ReadNetworkMessage<T>()` | Any NetworkMessage |

**Pooling**: `NetworkWriterPool.Get()` / `NetworkReaderPool.Get()` — use `using` for auto-return.

**Allocation limit**: Reader has 16 MB `AllocationLimit` for collections to prevent abuse.

### 5.2 Built-In Messages (`Core/Messages.cs`)

| Message Struct | Direction | Purpose |
|----------------|-----------|---------|
| `TimeSnapshotMessage` | Bidirectional | Periodic time synchronization |
| `ReadyMessage` | Client → Server | Client ready to receive spawns |
| `NotReadyMessage` | Client → Server | Client temporarily not ready |
| `AddPlayerMessage` | Client → Server | Request player object spawn |
| `SceneMessage` | Server → Client | Scene load/unload instruction |
| `CommandMessage` | Client → Server | [Command] RPC payload |
| `RpcMessage` | Server → Client | [ClientRpc] / [TargetRpc] payload |
| `RpcOptimizedMessage` | Server → Client | Optimized RPC variant |
| `SpawnMessage` | Server → Client | Spawn networked object |
| `UnspawnMessage` | Server → Client | Unspawn (return to pool) |
| `ObjectDestroyMessage` | Server → Client | Permanently destroy object |
| `UpdateVarsMessage` | Bidirectional | SyncVar delta updates |

**Enums**: `SceneOperation { Normal, LoadAdditive, UnloadAdditive }`, `SpawnFlags`

### 5.3 Message Batching (`Core/Batching/`)

| Class | Purpose |
|-------|---------|
| `Batcher` | Groups outgoing messages into batches with timestamps; threshold-based sizing |
| `Unbatcher` | De-batches received data, extracts individual messages and timestamps |

Each batch starts with an 8-byte timestamp header. Messages within a batch have varint-length headers.

### 5.4 Custom Type Serialization

The **Weaver** auto-generates `Read<T>()` / `Write<T>()` for custom structs implementing `NetworkMessage`. For custom serialization, define static extension methods:

```csharp
public static void WriteMyType(this NetworkWriter writer, MyType value) { ... }
public static MyType ReadMyType(this NetworkReader reader) { ... }
```

The Weaver discovers these via `ReaderWriterProcessor` and registers them automatically.

---

## 6. Synchronized State

### 6.1 [SyncVar] (`Core/Attributes.cs`, processed by Weaver)

Automatically synchronizes a field from owner to observers.

**Usage**:
```csharp
[SyncVar] int health = 100;
[SyncVar(hook = nameof(OnHealthChanged))] int health = 100;
```

**Properties**:
- Direction controlled by `syncDirection` on NetworkBehaviour
- Hook callback: `void OnHealthChanged(int oldValue, int newValue)`
- Weaver generates getter/setter IL code and dirty-bit tracking
- Only changed fields are sent (delta updates)

### 6.2 SyncList\<T\> (`Core/SyncList.cs`)

Networked list with per-operation delta sync.

| Operation | Callback |
|-----------|----------|
| `Add(item)` | `OnAdd(index)` |
| `Insert(index, item)` | `OnInsert(index)` |
| `this[index] = item` | `OnSet(index, oldItem)` |
| `RemoveAt(index)` | `OnRemove(index, oldItem)` |
| `Clear()` | `OnClear()` |
| Any change | `OnChange(op, index, item)` |

### 6.3 SyncDictionary\<TKey, TValue\> (`Core/SyncDictionary.cs`)

Networked dictionary. Also: `SyncSortedDictionary<TKey, TValue>`.

| Operation | Callback |
|-----------|----------|
| `Add(key, value)` | `OnAdd(key)` |
| `this[key] = value` | `OnSet(key, oldValue)` |
| `Remove(key)` | `OnRemove(key, oldValue)` |
| `Clear()` | `OnClear()` |

### 6.4 SyncHashSet\<T\> / SyncSortedSet\<T\> (`Core/SyncSet.cs`)

Networked set (unordered or sorted).

| Operation | Callback |
|-----------|----------|
| `Add(item)` | `OnAdd(item)` |
| `Remove(item)` | `OnRemove(item)` |
| `Clear()` | `OnClear()` |

### 6.5 SyncObject Base (`Core/SyncObject.cs`)

Abstract base for all sync collections. Provides:
- `OnDirty` callback
- `IsRecording()` / `IsWritable()` authority checks
- `OnSerializeAll()` / `OnSerializeDelta()` / `OnDeserializeAll()` / `OnDeserializeDelta()`

---

## 7. Remote Procedure Calls

### 7.1 [Command] — Client → Server

```csharp
[Command]
void CmdFireWeapon(Vector3 direction) { /* runs on server */ }
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `channel` | `Channels.Reliable` | Transport channel |
| `requiresAuthority` | `true` | Must be object owner to call |

### 7.2 [ClientRpc] — Server → All Clients

```csharp
[ClientRpc]
void RpcPlaySound(string soundName) { /* runs on all clients */ }
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `channel` | `Channels.Reliable` | Transport channel |
| `includeOwner` | `true` | Also execute on owning client |

### 7.3 [TargetRpc] — Server → One Client

```csharp
[TargetRpc]
void TargetShowMessage(NetworkConnectionToClient target, string msg) { /* runs on target client */ }
```

| Parameter | Default | Description |
|-----------|---------|-------------|
| `channel` | `Channels.Reliable` | Transport channel |

### 7.4 Guard Attributes

| Attribute | Effect |
|-----------|--------|
| `[Server]` | Method only executes on server (logs warning if called on client) |
| `[ServerCallback]` | Same as `[Server]` but no warning |
| `[Client]` | Method only executes on client (logs warning if called on server) |
| `[ClientCallback]` | Same as `[Client]` but no warning |

### 7.5 Runtime Registration (`Core/RemoteCalls.cs`)

RPCs use 16-bit stable hashes (`FNV-1a` via `GetStableHashCode16()`) for function identification. `RemoteProcedureCalls` maintains a static dictionary of registered delegates.

---

## 8. Network Transform & Rigidbody Sync

### 8.1 NetworkTransform Variants (`Components/NetworkTransform/`)

| Class | Channel | Strategy |
|-------|---------|----------|
| `NetworkTransformReliable` | Reliable | Delta compression; precision rounding; bandwidth optimal |
| `NetworkTransformUnreliable` | Unreliable | Snapshot buffering; smooth under packet loss |
| `NetworkTransformHybrid` | Both | Reliable full-state + unreliable deltas; balanced |

**Base class**: `NetworkTransformBase` (abstract)
- Selective sync: position (X/Y/Z), rotation (X/Y/Z), scale
- Coordinate space: World or Local
- `onlySyncOnChange` optimization
- Snapshot interpolation for smooth remote movement
- Compression via `Changed` bitflags

**Data types**:
- `TransformSnapshot` — `(remoteTime, localTime, position, rotation, scale)`
- `SyncData` — compact bitfield-based sync data with `Changed` enum flags

### 8.2 NetworkRigidbody Variants (`Components/NetworkRigidbody/`)

| Class | Physics | Channel |
|-------|---------|---------|
| `NetworkRigidbodyReliable` | 3D Rigidbody | Reliable |
| `NetworkRigidbodyUnreliable` | 3D Rigidbody | Unreliable |
| `NetworkRigidbodyReliable2D` | 2D Rigidbody2D | Reliable |
| `NetworkRigidbodyUnreliable2D` | 2D Rigidbody2D | Unreliable |
| `NetworkRigidbodyHybridCORE` | 3D Rigidbody | Hybrid |

All set non-authority rigidbodies to kinematic. Extend the corresponding `NetworkTransform*` base.

---

## 9. Client-Side Prediction

### 9.1 PredictedRigidbody (`Components/PredictedRigidbody/`)

| Class | Purpose |
|-------|---------|
| `PredictedRigidbody` | Main component: client predicts physics locally; server sends corrections |
| `PredictedSyncData` | Blittable struct (deltaTime, position, rotation, velocity, angularVelocity) |
| `RigidbodyState` | State snapshot with delta tracking for interpolation |
| `PredictionUtils` | Utilities: ghost creation, collider copying, rigidbody moving |
| `PredictedRigidbodyPhysicsGhost` | Component on ghost object for raycasts |

**PredictionMode**: `Smooth` (ghost object with visual interpolation) or `Fast`

**Workflow**:
1. Client predicts physics locally and stores state history
2. Server simulates authoritatively and sends corrections
3. Client compares predicted vs actual state
4. If divergence exceeds threshold → reconcile (snap or smooth correction)
5. Ghost object shows physics state; visual object interpolates toward it

### 9.2 Prediction Core (`Core/Prediction/`)

| Interface / Class | Purpose |
|-------------------|---------|
| `PredictedState` (interface) | Defines state fields: timestamp, position, rotation, velocity, angular velocity (+ deltas) |
| `Prediction` (static) | `Sample()` — interpolate/extrapolate state history |

---

## 10. Interest Management (AOI)

### 10.1 Base Classes (`Core/InterestManagementBase.cs`, `Core/InterestManagement.cs`)

| Method | Description |
|--------|-------------|
| `OnCheckObserver(identity, connection)` | Return true if connection should see identity |
| `OnRebuildObservers(identity, observers)` | Populate observer set |
| `SetHostVisibility(identity, visible)` | Show/hide in host mode (renderers, lights, audio, etc.) |

**NetworkIdentity.visibility**: `Default` (use AOI), `ForceHidden` (never visible), `ForceShown` (always visible)

### 10.2 Implementations (`Components/InterestManagement/`)

| Implementation | Directory | Strategy | Key Config |
|----------------|-----------|----------|------------|
| `DistanceInterestManagement` | `Distance/` | Brute-force distance check | `visRange` (float) |
| `DistanceInterestManagementCustomRange` | `Distance/` | Per-object custom range | Attach to NetworkIdentity |
| `SceneInterestManagement` | `Scene/` | Same Unity scene = visible | Scene-based isolation |
| `SceneDistanceInterestManagement` | `SceneDistance/` | Same scene AND within distance | Combined check |
| `MatchInterestManagement` | `Match/` | Same `matchId` (Guid) = visible | `NetworkMatch` component |
| `TeamInterestManagement` | `Team/` | Same `teamId` (string) = visible | `NetworkTeam` component |
| `SpatialHashingInterestManagement` | `SpatialHashing/` | 2D grid (XZ or XY), 9 neighbors | `visRange`, `CheckMethod` |
| `SpatialHashing3DInterestManagement` | `SpatialHashing/` | 3D grid, 27 neighbors | `visRange` (XYZ) |
| `HexSpatialHash2DInterestManagement` | `SpatialHashing/` | 2D hex grid, 7 cells | `visRange`, hexagonal tiling |
| `HexSpatialHash3DInterestManagement` | `SpatialHashing/` | 3D hex grid, 19 cells | `visRange`, `cellHeight` |

**Lazy Rebuilding**: Most implementations track "dirty" objects/scenes/matches and only rebuild when needed, plus periodic rebuild intervals.

---

## 11. Snapshot Interpolation & Time Sync

### 11.1 NetworkTime (`Core/NetworkTime.cs`)

| Property | Description |
|----------|-------------|
| `localTime` | Local system time (unscaled) |
| `time` | Estimated server time (via interpolation) |
| `rtt` | Round-trip time (latency) |
| `PingInterval` | 0.1s between pings |
| `PingWindowSize` | 50 samples for averaging |

### 11.2 Snapshot Interpolation (`Core/SnapshotInterpolation/`)

| Class | Purpose |
|-------|---------|
| `SnapshotInterpolation` (static) | Core algorithm: `Timescale()`, `Interpolate()`, `TargetBufferTime()` |
| `Snapshot` (interface) | `remoteTime`, `localTime` |
| `TimeSnapshot` (struct) | Lightweight time-only snapshot |
| `SnapshotInterpolationSettings` | `bufferTimeMultiplier` (2x), `bufferLimit` (32), catchup speeds |

**Algorithm**:
1. Server sends periodic `TimeSnapshotMessage` to clients
2. Client buffers snapshots sorted by remote time
3. Client interpolates between two snapshots using adjusted `localTimeline`
4. Dynamic catchup/slowdown keeps buffer at target size
5. Entity transforms interpolate between position/rotation snapshots

### 11.3 Client Time Interpolation (`Core/NetworkClient_TimeInterpolation.cs`)

Partial class on `NetworkClient` managing:
- `localTimeline` — client's interpolated time
- `localTimescale` — speed adjustment (catchup/slowdown)
- `snapshots` — `SortedList<double, TimeSnapshot>`

---

## 12. Lag Compensation

### 12.1 Core Algorithm (`Core/LagCompensation/`)

| Class | Purpose |
|-------|---------|
| `LagCompensation` (static) | `Insert()` — store state; `Sample()` — interpolate at past timestamp |
| `Capture` (interface) | `timestamp`, `DrawGizmo()` |
| `LagCompensationSettings` | `historyLimit` (6), `captureInterval` (0.1s) |
| `HistoryBounds` / `MinMaxBounds` | Bounding box structures for physics history |

### 12.2 Components (`Components/LagCompensation/`)

| Class | Purpose |
|-------|---------|
| `HistoryCollider` | Maintains axis-aligned bounds history for a collider; projects to trigger collider for raycasting |
| `LagCompensator` | Tracks multiple collider histories; `Sample()` estimates client time and returns interpolated capture |

**Workflow**: Server captures collider bounds at intervals → On hit request, rewind to estimated client time → Interpolate bounds → Raycast against historical position.

---

## 13. Authentication

### 13.1 Base Class (`Core/NetworkAuthenticator.cs`)

**Class**: `NetworkAuthenticator : MonoBehaviour` (abstract)

| Method | Side | Description |
|--------|------|-------------|
| `OnStartServer()` | Server | Register auth message handlers |
| `OnServerAuthenticate(conn)` | Server | Authenticate connecting client |
| `ServerAccept(conn)` | Server | Approve client |
| `ServerReject(conn)` | Server | Reject client |
| `OnStartClient()` | Client | Register auth message handlers |
| `OnClientAuthenticate()` | Client | Begin auth handshake |
| `ClientAccept()` / `ClientReject()` | Client | Auth result |

**Events**: `OnServerAuthenticated` (UnityEvent\<NetworkConnectionToClient\>), `OnClientAuthenticated` (UnityEvent)

### 13.2 Implementations (`Authenticators/`)

| Authenticator | Mechanism |
|---------------|-----------|
| `BasicAuthenticator` | Username + password validation |
| `DeviceAuthenticator` | `SystemInfo.deviceUniqueIdentifier` |
| `TimeoutAuthenticator` | Wraps another authenticator; adds timeout deadline |
| `UniqueNameAuthenticator` | Validates unique player names (HashSet tracking) |

---

## 14. IL Weaver (Code Generation)

### 14.1 Overview

The Weaver is a **Mono.Cecil-based IL post-processor** that modifies compiled C# assemblies at compile-time. It enables Mirror's declarative API (`[SyncVar]`, `[Command]`, `[ClientRpc]`).

**Location**: `Editor/Weaver/`
**Dependency**: `Plugins/Mono.Cecil/` (CecilX DLLs)

### 14.2 Entry Points

| File | Unity Version | Mechanism |
|------|---------------|-----------|
| `ILPostProcessorHook.cs` | ≥ 2020.3 | Unity's `ILPostProcessor` API (preferred) |
| `CompilationFinishedHook.cs` | < 2020.3 | `CompilationPipeline.assemblyCompilationFinished` callback |

### 14.3 Processing Pipeline

1. **Assembly Resolution**: `ILPostProcessorAssemblyResolver` resolves Mirror + Unity references
2. **Reader/Writer Discovery**: `ReaderWriterProcessor` finds custom `Read<T>()` / `Write<T>()` methods
3. **MonoBehaviour Validation**: `MonoBehaviourProcessor` ensures non-NetworkBehaviour classes don't use Mirror attributes
4. **NetworkBehaviour Processing** (per class): `NetworkBehaviourProcessor` coordinates:
   - `SyncVarAttributeProcessor` → generates getter/setter IL, hook delegates, dirty bits
   - `SyncObjectProcessor` → validates SyncList/Dictionary/Set fields
   - `CommandProcessor` → replaces `[Command]` method body with network send; creates server invoke handler
   - `RpcProcessor` → replaces `[ClientRpc]` method body with broadcast; creates client invoke handler
   - `TargetRpcProcessor` → replaces `[TargetRpc]` method body with targeted send
   - `ServerClientAttributeProcessor` → injects `isServer`/`isClient` guard checks
5. **Method Substitution**: `MethodProcessor` renames original methods (e.g., `CmdFire` → `UserCode_CmdFire`) and injects network stubs
6. **SyncVar Access Replacement**: `SyncVarAttributeAccessReplacer` replaces direct field access with generated getters/setters

### 14.4 Key Weaver Classes

| Class | Purpose |
|-------|---------|
| `Weaver` | Main orchestrator; coordinates all processing |
| `WeaverTypes` | Caches references to Mirror runtime types/methods |
| `Readers` / `Writers` | Generate/register `Read<T>()` / `Write<T>()` methods |
| `SyncVarAccessLists` | Tracks replacement getter/setter properties and dirty bit counts |
| `Extensions` | Type checking: `Is<T>()`, `IsDerivedFrom<T>()`, `ImplementsInterface<T>()` |

### 14.5 WeaverFuse (`Core/WeaverFuse.cs`)

`WeaverFuse.Weaved()` returns `true` if the weaver ran successfully. Used as a safety check to prevent running un-weaved code.

---

## 15. Components Reference

| Component | File | Purpose |
|-----------|------|---------|
| `NetworkAnimator` | `Components/NetworkAnimator.cs` | Syncs Animator parameters (int/float/bool/trigger) across network |
| `NetworkName` | `Components/NetworkName.cs` | Syncs GameObject name for debugging |
| `NetworkPingDisplay` | `Components/NetworkPingDisplay.cs` | OnGUI displaying RTT in milliseconds |
| `NetworkStatistics` | `Components/NetworkStatistics.cs` | Tracks bandwidth (bytes/packets per second) |
| `RemoteStatistics` | `Components/RemoteStatistics.cs` | Server syncs performance stats to authenticated clients |
| `GUIConsole` | `Components/GUIConsole.cs` | Runtime console (errors/warnings/logs) |
| `NetworkDiagnosticsDebugger` | `Components/NetworkDiagnosticsDebugger.cs` | Logs network message events |
| `NetworkRoomManager` | `Components/NetworkRoomManager.cs` | Room system with slots, ready states, scene transitions |
| `NetworkRoomPlayer` | `Components/NetworkRoomPlayer.cs` | Per-player room state (ready, index) |
| `NetworkManagerHUD` | `Core/NetworkManagerHUD.cs` | Runtime GUI for server/client/host controls |

---

## 16. Utilities & Tools

### 16.1 Core Utilities (`Core/Tools/`)

| Class | Purpose |
|-------|---------|
| `Pool<T>` | Generic object pool (Stack-based) |
| `Compression` | Quaternion compression, VarInt encoding, float↔long scaling |
| `DeltaCompression` | Delta compress/decompress byte arrays |
| `Extensions` | `ToHexString()`, `GetStableHashCode()` (FNV-1a), `GetStableHashCode16()` |
| `Utils` | Connection creation helpers, address parsing |
| `ExponentialMovingAverage` | Sliding window average (for RTT, jitter, buffer) |
| `AccurateInterval` | High-precision interval timing |
| `Mathd` | Double-precision math (`Lerp`, `Clamp`, etc.) |

### 16.2 Custom Structs (`Core/Tools/CustomStructs/`)

| Struct | Size | Purpose |
|--------|------|---------|
| `Half` | 2 bytes | 16-bit float for bandwidth savings |
| `Vector2Byte` / `Vector3Byte` / `Vector4Byte` | 2-4 bytes | Byte-range vectors |
| `Vector2SByte` / `Vector3SByte` / `Vector4SByte` | 2-4 bytes | Signed byte-range vectors |
| `Vector3Long` / `Vector4Long` | 24-32 bytes | Long-precision vectors (large worlds) |

### 16.3 Threading (`Core/Threading/`)

| Class | Purpose |
|-------|---------|
| `ThreadLog` | Thread-safe logging from background threads |
| `WorkerThread` | Worker thread lifecycle management |
| `ConcurrentPool<T>` | Thread-safe object pool |
| `ConcurrentNetworkWriterPool` | Thread-safe writer pool for threaded transports |

### 16.4 Pooling (`Core/NetworkReaderPool.cs`, `Core/NetworkWriterPool.cs`)

| Pool | Factory | Usage |
|------|---------|-------|
| `NetworkWriterPool` | `NetworkWriterPooled` | `using var writer = NetworkWriterPool.Get(); ...` |
| `NetworkReaderPool` | `NetworkReaderPooled` | `using var reader = NetworkReaderPool.Get(data); ...` |

---

## 17. Profiling & Diagnostics

### 17.1 NetworkDiagnostics (`Core/NetworkDiagnostics.cs`)

Static events for monitoring all network traffic:
- `OutMessageEvent` — fired on every sent message
- `InMessageEvent` — fired on every received message
- `MessageInfo` struct: `{ message, channel, bytes, count }`

### 17.2 Profiling Components (`Components/Profiling/`)

| Component | Displays |
|-----------|----------|
| `NetworkBandwidthGraph` | In/Out bandwidth (B/s, KiB/s, MiB/s) |
| `NetworkPingGraph` | RTT average + variance |
| `FpsMinMaxAvgGraph` | FPS with min/max/avg curves |
| `NetworkRuntimeProfiler` | Per-message-type statistics (count, bytes) |
| `BaseUIGraph` | Abstract graph renderer (shader-based, circular buffer) |

**Aggregation modes**: Sum, Average, PerSecond, Min, Max

### 17.3 ConnectionQuality (`Core/ConnectionQuality.cs`)

| Quality | Color |
|---------|-------|
| `ESTIMATING` | — |
| `POOR` | — |
| `FAIR` | — |
| `GOOD` | — |
| `EXCELLENT` | — |

**Methods**: `Simple` (RTT + jitter based), `Pragmatic` (buffer time ratio based)

---

## 18. Network Discovery

### 18.1 Components (`Components/Discovery/`)

| Class | Purpose |
|-------|---------|
| `NetworkDiscoveryBase<Req, Resp>` | Generic UDP multicast discovery (port 47777) |
| `NetworkDiscovery` | Concrete implementation with ServerRequest/ServerResponse |
| `NetworkDiscoveryHUD` | OnGUI: Find Servers, Start Host, Start Server buttons |
| `ServerRequest` | Empty discovery request message |
| `ServerResponse` | Response with `Uri`, `serverId`, `IPEndPoint` |

**Usage**: Add `NetworkDiscovery` + `NetworkDiscoveryHUD` to scene. Clients broadcast on LAN, servers respond with connection info.

---

## 19. Editor Tooling

### 19.1 Custom Inspectors (`Editor/`)

| Editor | Target | Features |
|--------|--------|----------|
| `NetworkBehaviourInspector` | NetworkBehaviour | Shows sync settings (direction, mode, method, interval), SyncObject collections |
| `NetworkManagerEditor` | NetworkManager | Prefab list management, bulk scan/clear |
| `NetworkInformationPreview` | NetworkIdentity | Preview: Asset/Scene IDs, observers, owner |
| `SyncVarAttributeDrawer` | [SyncVar] fields | "SyncVar" label indicator |
| `ReadOnlyDrawer` | [ReadOnly] fields | Disables editing |
| `SceneDrawer` | [Scene] fields | Scene picker from build settings |
| `SyncObjectCollectionsDrawer` | SyncList/Set/Dict | Foldable collection display |
| `LagCompensatorInspector` | LagCompensator | Preview warning |

### 19.2 Build Processing (`Editor/`)

| Class | Purpose |
|-------|---------|
| `NetworkScenePostProcess` | Validates scene object IDs during builds |
| `AndroidManifestHelper` | Adds INTERNET + MULTICAST permissions for Android |
| `PreprocessorDefine` | Adds MIRROR version symbols to PlayerSettings |
| `Welcome` | One-time editor startup welcome message |

---

## 20. Examples Catalog

| Example | Description |
|---------|-------------|
| `Basic` | Minimal client/server setup |
| `Chat` | Text chat with NetworkMessages |
| `Tanks` / `TanksHybrid` | Tank game (reliable / hybrid sync) |
| `Pong` | Classic Pong networked |
| `Room` | Room/lobby system with NetworkRoomManager |
| `Billiards` / `BilliardsPredicted` | Physics sync + prediction |
| `Benchmark` / `BenchmarkIdle` / `BenchmarkPrediction` | Performance testing |
| `RigidbodyPhysics` / `RigidbodyBenchmark` | Physics synchronization |
| `AdditiveLevels` / `AdditiveScenes` / `MultipleAdditiveScenes` | Additive scene loading |
| `MultipleMatches` | Match-based interest management |
| `Discovery` | LAN server discovery |
| `CharacterSelection` | Character selection flow |
| `SyncDirection` | Client authority demonstration |
| `PickupsDropsChilds` | Pickup/drop mechanics |
| `TopDownShooter` | Top-down shooter with lag compensation |
| `LagCompensation` | Lag compensation demonstration |
| `HexSpatialHash` | Hex-grid interest management |
| `CouchCoop` | Couch co-op (multiple local players) |
| `StackedPrediction` | Stacked physics prediction |
| `EdgegapLobby` | Edgegap cloud lobby |
| `TankTheftAuto` | Vehicle enter/exit mechanics |
| `AssignAuthority` | Dynamic authority assignment |
| `VR` | VR networking |
| `CCU` | Concurrent user testing |
| `Snapshot Interpolation` | Snapshot interpolation demo |
| `PlayerTest` | Player spawn/destroy testing |
| `AutoLANClientController` | Auto-connect LAN client |
| `BenchmarkStinkySteak` | Third-party benchmark |

---

## 21. Tests

| Directory | Contents |
|-----------|----------|
| `Tests/Editor/` | Unit tests for Weaver, serialization, core systems |
| `Tests/Runtime/` | Integration tests requiring Unity runtime |
| `Tests/EditorBehaviours/` | Tests for editor-specific behaviors |
| `Tests/Common/` | Shared test utilities and helpers |

---

## 22. Coding Conventions

| Rule | Details |
|------|---------|
| Naming | PascalCase types/methods, camelCase private fields |
| Braces | Allman style (opening brace on new line) |
| `var` | **Not used** — always use explicit types |
| `this.` | **Not used** — no `this.` qualification |
| Private fields | camelCase (e.g., `connectingSendQueue`) |
| Static classes | Used for core systems (NetworkServer, NetworkClient) |
| `.editorconfig` | Enforced at repo root |
| Attributes | `[SyncVar]`, `[Command]`, `[ClientRpc]`, `[TargetRpc]`, `[Server]`, `[Client]` |

---

## 23. Common Patterns & Recipes

### 23.1 Creating a Networked Object

1. Add `NetworkIdentity` component to GameObject
2. Create a script inheriting `NetworkBehaviour`
3. Use `[SyncVar]` for synchronized fields
4. Use `[Command]` for client→server calls
5. Use `[ClientRpc]` for server→client calls
6. Register prefab in `NetworkManager.spawnPrefabs`
7. Spawn with `NetworkServer.Spawn(obj)`

### 23.2 Player Spawning Flow

```
Client connects → NetworkManager.OnServerConnect() → 
Client sends ReadyMessage → Client sends AddPlayerMessage →
NetworkManager.OnServerAddPlayer() → NetworkServer.AddPlayerForConnection()
```

### 23.3 Custom Message Flow

```csharp
// Define message
public struct MyMessage : NetworkMessage { public string text; }

// Server: register handler
NetworkServer.RegisterHandler<MyMessage>(OnMyMessage);

// Client: send
NetworkClient.Send(new MyMessage { text = "hello" });

// Server: handler
void OnMyMessage(NetworkConnectionToClient conn, MyMessage msg) { ... }
```

### 23.4 Interest Management Setup

1. Add an `InterestManagement` implementation to the NetworkManager GameObject
2. Assign it to `NetworkManager.interestManagement` (or leave null for global visibility)
3. For per-object overrides, use `NetworkIdentity.visibility = ForceShown/ForceHidden`

### 23.5 Scene Management

```csharp
// Server changes scene for all clients
NetworkManager.ServerChangeScene("GameScene");

// Additive scene loading
SceneManager.LoadSceneAsync("Level2", LoadSceneMode.Additive);
NetworkServer.SendToAll(new SceneMessage { sceneName = "Level2", operation = SceneOperation.LoadAdditive });
```

---

## 24. Key Type Quick-Reference

### Core Types

| Type | Namespace | Category |
|------|-----------|----------|
| `NetworkManager` | `Mirror` | Lifecycle management |
| `NetworkServer` | `Mirror` | Server-side logic (static) |
| `NetworkClient` | `Mirror` | Client-side logic (static) |
| `NetworkIdentity` | `Mirror` | Object identity (one per GO) |
| `NetworkBehaviour` | `Mirror` | Networked component base |
| `NetworkConnection` | `Mirror` | Connection base |
| `NetworkConnectionToClient` | `Mirror` | Server→client connection |
| `NetworkConnectionToServer` | `Mirror` | Client→server connection |
| `Transport` | `Mirror` | Transport abstraction |
| `NetworkAuthenticator` | `Mirror` | Auth base |
| `InterestManagement` | `Mirror` | Visibility base |
| `NetworkTime` | `Mirror` | Time synchronization |

### Serialization Types

| Type | Category |
|------|----------|
| `NetworkWriter` / `NetworkWriterPooled` | Write binary data |
| `NetworkReader` / `NetworkReaderPooled` | Read binary data |
| `NetworkMessage` (interface) | Message marker |

### Sync Types

| Type | Category |
|------|----------|
| `SyncList<T>` | Networked list |
| `SyncDictionary<TKey,TValue>` | Networked dictionary |
| `SyncHashSet<T>` | Networked hash set |
| `SyncSortedSet<T>` | Networked sorted set |
| `SyncSortedDictionary<TKey,TValue>` | Networked sorted dictionary |

### Transport Types

| Type | Protocol |
|------|----------|
| `KcpTransport` | KCP/UDP |
| `ThreadedKcpTransport` | KCP/UDP (threaded) |
| `TelepathyTransport` | TCP |
| `SimpleWebTransport` | WebSocket |
| `EncryptionTransport` | AES256-GCM wrapper |
| `MultiplexTransport` | Multi-transport |
| `LatencySimulation` | Test simulation |

### Component Types

| Type | Category |
|------|----------|
| `NetworkTransformReliable` | Reliable position sync |
| `NetworkTransformUnreliable` | Unreliable position sync |
| `NetworkTransformHybrid` | Hybrid position sync |
| `NetworkRigidbodyReliable` | 3D physics sync |
| `NetworkRigidbodyUnreliable` | 3D physics sync |
| `NetworkRigidbodyReliable2D` | 2D physics sync |
| `PredictedRigidbody` | Prediction system |
| `NetworkAnimator` | Animator sync |
| `NetworkDiscovery` | LAN discovery |
| `NetworkRoomManager` | Room/lobby system |
| `DistanceInterestManagement` | Distance AOI |
| `SpatialHashingInterestManagement` | Grid AOI |
| `MatchInterestManagement` | Match-based AOI |
| `TeamInterestManagement` | Team-based AOI |
| `SceneInterestManagement` | Scene-based AOI |

### Attributes

| Attribute | Target | Purpose |
|-----------|--------|---------|
| `[SyncVar]` | Field | Auto-sync field |
| `[Command]` | Method | Client → Server RPC |
| `[ClientRpc]` | Method | Server → All Clients RPC |
| `[TargetRpc]` | Method | Server → One Client RPC |
| `[Server]` | Method | Server-only guard (warning) |
| `[ServerCallback]` | Method | Server-only guard (silent) |
| `[Client]` | Method | Client-only guard (warning) |
| `[ClientCallback]` | Method | Client-only guard (silent) |
| `[ShowInInspector]` | Field | Show SyncList in inspector |
| `[ReadOnly]` | Field | Read-only in inspector |
| `[Scene]` | Field | Scene picker in inspector |

---

*This document was generated by analyzing all source files in the Mirror networking library repository. It is intended for AI agent consumption to enable accurate code generation, modification, and analysis tasks.*
