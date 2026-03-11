# Networking & Multiplayer Architecture

---

## Overview

Voxelite uses a server-authoritative architecture with client-side prediction for
movement. The server runs all game logic (AI, Lua scripts, state machines, event trees).
Clients are thin renderers that send input commands and receive the minimum state
needed to update visuals.

Both listen server (host-as-player) and dedicated server modes are supported. They
share identical game logic — a listen server is a dedicated server with a local client
attached.

---

## Server Model

### Listen server (default)

One player hosts. The host's game instance runs the server internally alongside their
client. Other players connect to the host.

- Default for "play with friends" — start a world, invite 1-15 others
- World persists on the host's machine
- World is offline when host is not playing
- Host has zero-latency advantage (mitigated by server-authoritative validation)

### Dedicated server

Headless server binary with no rendering. Runs on a machine or VPS.

- For communities that want persistent always-on worlds
- Same binary, launched with `--dedicated` flag (or separate stripped build)
- Configured via `server.yaml` instead of in-game UI
- Console-only interface (no GUI) with the same command set as the in-game console
- Can run on Linux without a display server

### Architecture identity

The listen server and dedicated server run identical code paths. The only difference
is whether a local client is attached:

```
Listen server:
  ┌──────────────────────────────┐
  │  Server (game logic, Lua,   │
  │  world state, entity AI)    │
  │         ↕ direct            │
  │  Local client (render,      │◄──── Remote clients
  │  input, audio)              │      connect via network
  └──────────────────────────────┘

Dedicated server:
  ┌──────────────────────────────┐
  │  Server (game logic, Lua,   │
  │  world state, entity AI)    │◄──── All clients
  │                             │      connect via network
  │  No client. No rendering.   │
  └──────────────────────────────┘
```

---

## Protocol

### Primary — ENet (Godot built-in)

ENet is the default transport protocol via Godot's `ENetMultiplayerPeer`.

- UDP-based with reliable and unreliable channels
- Built into Godot — zero additional dependencies
- Handles connection management, packet ordering, fragmentation, congestion control
- Direct connections — requires port forwarding or relay for NAT traversal

**Channel allocation:**

| Channel | Mode | Contents |
|---------|------|----------|
| 0 | Reliable ordered | Commands (player actions), chunk deltas, inventory changes, event tree state |
| 1 | Unreliable sequenced | Entity position/rotation updates (high frequency, latest-wins) |
| 2 | Reliable ordered | Chat messages, system notifications, quest updates |
| 3 | Reliable unordered | Chunk generation acknowledgments, mod validation |

### Optional — Steam Networking Sockets

When the game ships on Steam, `GodotSteam` provides Steam Networking Sockets as an
alternative transport layer.

- Valve relay network — NAT traversal solved, no port forwarding needed
- DDoS protection via Valve infrastructure
- Opt-in: only used when both host and client are running through Steam
- Falls back to ENet for non-Steam builds and direct-connect scenarios

**Steam integration is opt-in.** The game must be fully functional without Steam for
development, testing, and non-Steam distribution. Steam-specific features (relay,
lobbies, matchmaking) are behind a compile-time flag and runtime availability check.

### LAN play

LAN play works with zero configuration:

- Server broadcasts presence via UDP multicast on the local network
- Client's "LAN Games" browser listens for broadcasts and lists available servers
- Connection is direct ENet — no internet, no account, no port forwarding
- Broadcast includes: server name, player count, mod list hash, game version

```yaml
# LAN broadcast packet (UDP multicast, every 3 seconds)
server_name: "Alex's World"
game_version: "1.2.0"
players: 3
max_players: 16
mod_hash: "a7f3c2e8"        # quick mod list compatibility check
port: 25565
```

---

## Connection Flow

### Client connecting to server

```
1. Client sends: game version, mod list (ids + versions)
2. Server checks:
   - Version compatible?
   - Mod list matches exactly?
3. If mismatch:
   - Send rejection with specific mismatch details
   - "Server requires crystal_creatures@1.3.0, you have 1.2.0"
   - "Server has mod magic_system which you are missing"
4. If match:
   - Server sends: world seed, world settings, time of day, weather
   - Server sends: player's saved data (or creates new player)
   - Server begins streaming chunks around player position
   - Client enters world when Ring 0 chunk is ready
```

### Disconnection

```
Clean disconnect:
  1. Client sends disconnect message
  2. Server saves player data immediately
  3. Server saves chunk player was in (if dirty)
  4. Server despawns player entity
  5. Server notifies other clients

Timeout (crash, network loss):
  1. Server detects no packets for timeout period (default: 15 seconds)
  2. Player entity remains in world briefly (configurable grace period: 30 seconds)
  3. After grace period: save player, despawn, notify others
  4. Grace period prevents exploit: can't pull the plug to avoid death
```

---

## Client-Side Prediction

### Movement prediction

Player movement is predicted client-side for immediate responsiveness. The server
reconciles authoritatively.

```
Frame N:
  1. Client reads input (WASD, jump, sprint)
  2. Client simulates movement locally (position, velocity)
  3. Client renders at predicted position (immediate, no latency)
  4. Client sends InputCommand to server:
     { sequence: 42, inputs: [forward, sprint], delta: 0.016, rotation: (0, 1.2, 0) }
  5. Client stores prediction: { sequence: 42, predicted_position: (104.5, 65, -38.2) }

Server receives InputCommand:
  1. Server applies input to authoritative player state
  2. Server simulates physics with server-side world state
  3. Server sends StateUpdate to client:
     { last_processed_sequence: 42, position: (104.5, 65, -38.2), velocity: ... }

Client receives StateUpdate:
  1. Compare server position to predicted position for sequence 42
  2. If match (within tolerance): no correction needed (common case)
  3. If mismatch:
     a. Snap authoritative state to server position
     b. Re-simulate all unacknowledged inputs (sequences 43, 44, ...)
     c. Smoothly blend visual position toward corrected position
```

### What is predicted (client-side)

- Player position and velocity
- Jump arc
- Sprint state
- Collision with terrain (client has the terrain data)

### What is NOT predicted

- Combat damage and health changes
- Block placement/breaking (wait for server confirmation)
- Inventory mutations
- Entity spawning/death
- Status effects and buffs
- Skill activations and cooldowns
- Event tree progression
- Any Lua script execution

For non-predicted actions, the client sends a command and waits for the server event
before updating local state. A brief delay (~50-150ms on typical connections) is
acceptable for these actions. Visual feedback (animation start, sound effect) can
play immediately as a hint, then be corrected if the server rejects the command.

### Block interaction feedback

Block breaking/placing has a perceived latency issue if purely server-authoritative.
Mitigation:

- **Block breaking:** Client plays break animation and particles immediately on input.
  Server confirms with `BlockBrokenEvent`. If server rejects (block already gone, wrong
  tool, protected area), client reverts the visual.
- **Block placing:** Client shows ghost block at target position immediately. Server
  confirms with `BlockPlacedEvent`. If rejected, ghost disappears. Typical confirm
  time is 1-2 frames on LAN, 3-5 frames on internet.

---

## Entity Replication

### Server → Client updates

The server sends only the data clients need to render entities correctly. No AI state,
no Lua internals, no pathfinding data.

**Per-entity update packet:**

| Field | Frequency | Channel |
|-------|-----------|---------|
| Position | Every tick (20/sec) | Unreliable sequenced |
| Rotation | Every tick | Unreliable sequenced |
| Velocity | Every tick (for interpolation) | Unreliable sequenced |
| Animation state | On change | Reliable ordered |
| Health (for health bar display) | On change | Reliable ordered |
| Equipment / held item (visual) | On change | Reliable ordered |
| Active buff VFX | On change | Reliable ordered |
| Entity spawn (type, initial state) | Once | Reliable ordered |
| Entity death (death animation, loot) | Once | Reliable ordered |

### Entity interpolation

Clients receive entity position updates at a fixed rate (default: 20 updates/second).
Between updates, entity positions are interpolated for smooth visual movement.

```
Client receives position updates for entity at times T0, T1, T2...
Client renders entity position at current_time - interpolation_delay
Interpolation delay = 2 × update interval (100ms at 20/sec)
This ensures there is always a future position to interpolate toward
```

**Extrapolation fallback:** If an update is late (packet loss), the client briefly
extrapolates using last known velocity. If updates stop entirely (entity left interest
area or network issue), extrapolation stops after 500ms and entity freezes until
updates resume.

### Interest management

The server only sends entity data relevant to each client:

```
For each client:
  interest_radius = client_render_distance × chunk_size

  For each entity in world:
    if distance(entity, client_player) < interest_radius:
      → Send updates to this client
    else:
      → Do not send (client doesn't know entity exists)

  When entity enters interest radius:
    → Send spawn packet (type, position, visual state)

  When entity leaves interest radius:
    → Send despawn packet (client removes entity locally)
```

**Priority:** When bandwidth is constrained, prioritize updates for:
1. Entities the player is looking at or targeting
2. Entities within close range (combat distance)
3. Other players
4. Named NPCs
5. Generic mobs at medium range
6. Distant entities (lowest update rate)

---

## Chunk Streaming

### Seed + delta model

Clients generate terrain locally from the world seed. The server sends only the delta
of player-made modifications for each chunk. For unmodified chunks (the vast majority),
the network cost is near zero.

```
Client needs chunk at (4, 2, -7):
  1. Client requests chunk from server
  2. Server checks: has this chunk been modified?

     If unmodified:
       → Server sends: { chunk: (4,2,-7), status: "pristine" }
       → Client generates chunk locally from seed (deterministic)

     If modified:
       → Server sends: { chunk: (4,2,-7), status: "modified", delta: [...] }
       → Client generates base terrain from seed
       → Client applies delta (list of position → block_id changes)
       → Result matches server state exactly
```

**Delta format:**

```yaml
# Chunk delta — only blocks that differ from seed-generated terrain
chunk: [4, 2, -7]
modifications:
  - { pos: [3, 12, 7], block: core:oak_planks }    # player placed
  - { pos: [3, 13, 7], block: core:oak_planks }
  - { pos: [5, 10, 2], block: core:air }            # player broke
  - { pos: [5, 10, 3], block: core:air }
entity_data: [...]   # entities in this chunk (if any)
block_entity_data: [...]  # block entities (if any)
```

**Bandwidth savings:** A player exploring unmodified terrain receives essentially zero
chunk data. Only areas with player construction or destruction generate network traffic.

### Chunk request prioritization

Chunks are requested by the client in the same concentric ring order as local chunk
loading:

```
Ring 0: chunk containing player           → highest priority
Ring 1: 8 surrounding chunks              → high priority
Ring 2-3: expanding ring                   → medium priority
Ring 4+: up to render distance             → background, interleaved
```

Server processes chunk requests in the priority order reported by each client.

### Chunk modification broadcast

When a player modifies a block, all clients with that chunk loaded receive the update:

```
Player breaks block at (67, 54, -12):
  → Server validates (tool, permissions, block exists)
  → Server applies change to world state
  → Server sends to all clients with chunk loaded:
    { event: block_broken, pos: [67, 54, -12], prev_block: core:stone }
  → Each client updates local chunk mesh
  → Delta log for chunk is updated (for future client joins)
```

---

## Tick Rate and Bandwidth

### Server tick rate

| System | Rate | Notes |
|--------|------|-------|
| Server game tick | 20 ticks/sec (50ms) | AI, physics, Lua, state machines |
| Entity position broadcast | 20/sec | Matches game tick |
| Chunk delta broadcast | Immediate on change | Event-driven, not polled |
| Player input processing | As received | Not tick-aligned — processed ASAP |

### Bandwidth budget (per client)

| Direction | Budget | Contents |
|-----------|--------|----------|
| Server → Client | ~50 KB/sec typical | Entity updates, chunk deltas, events |
| Client → Server | ~5 KB/sec typical | Input commands, chunk requests |

**Peak bandwidth:** During initial join (chunk streaming) or entering a heavily modified
area, bandwidth spikes to ~200 KB/sec briefly. Chunk streaming is throttled to prevent
saturating the connection.

### Player count: 16

Target ceiling for first release: **16 concurrent players**. Architecture designed for
single-process server.

| Players | Server CPU (estimated) | Bandwidth (estimated) |
|---------|----------------------|----------------------|
| 1-4 | Minimal (listen server on player's machine) | < 100 KB/sec total |
| 5-8 | Moderate (any modern desktop) | ~400 KB/sec total |
| 9-16 | Significant (dedicated recommended) | ~800 KB/sec total |

**Future scaling beyond 16:** Large-scale worlds (32-64+ players) would require
architectural changes: spatial partitioning of the server tick, multiple server
processes with region handoff, or a distributed entity system. This is explicitly
out of scope for v1 but the message bus architecture does not preclude it — events
can be routed across process boundaries.

---

## Server Configuration

### server.yaml (dedicated server)

```yaml
# server.yaml — dedicated server configuration

server:
  name: "Voxelite Community Server"
  port: 25565
  max_players: 16
  password: null                    # null = no password
  whitelist_enabled: false
  whitelist: []

  network:
    tick_rate: 20                   # server ticks per second
    timeout: 15                     # seconds before disconnect on no packets
    disconnect_grace_period: 30     # seconds player entity stays after disconnect
    chunk_stream_throttle: 10      # max chunks sent per second per client
    max_packet_size: 1400           # bytes, MTU-safe

  steam:
    enabled: false                  # opt-in Steam Networking Sockets
    lobby_type: friends             # friends, public, private, invisible

  lan:
    broadcast: true                 # advertise on LAN
    broadcast_interval: 3           # seconds

  world:
    path: "./saves/server_world"
    autosave_interval: 300
    backup_count: 3

  mods:
    path: "./mods"
    # Mod list and load order determined by mods in path + dependencies

  permissions:
    default_role: player
    roles:
      admin:
        commands: all
        can_ban: true
        can_kick: true
        can_modify_world_settings: true
      moderator:
        commands: [tp, kick, mute, inspect]
        can_kick: true
      player:
        commands: [spawn_point, home]
      guest:
        commands: []
        can_break_blocks: false
        can_place_blocks: false

    players:
      "a7f3c2e8-9b41-4d2a": admin
      "b8e4d1f2-3c56-4e7b": moderator
```

---

## Anti-Cheat

The server-authoritative model provides baseline anti-cheat:

| Cheat type | Prevention |
|------------|-----------|
| Speed hacking | Server validates movement distance per tick against max speed |
| Teleporting | Server rejects position jumps beyond tolerance |
| Fly hacking | Server simulates gravity; rejects airborne movement without jump |
| Item duplication | Inventory is server-authoritative; client cannot modify |
| Damage hacking | Damage calculated server-side from weapon stats and skills |
| X-ray / wallhack | Cannot prevent (client has chunk data), but blocks behind walls are sent regardless — no data to withhold |
| Reach hacking | Server validates interaction distance |
| Spam attacks | Command rate limiting per client |

**Philosophy:** Server-authoritative design prevents the most impactful cheats (items,
damage, movement). Visual-only cheats (x-ray, ESP) cannot be fully prevented in a
voxel game where the client must have block data for rendering. This is an accepted
tradeoff — the same as Minecraft.

---

## Multiplayer-Specific Events

Additional events on the message bus for multiplayer:

```yaml
events:
  - id: core:player_connected
    fields: [player_id, display_name]

  - id: core:player_disconnected
    fields: [player_id, reason]

  - id: core:player_kicked
    fields: [player_id, reason, by]

  - id: core:player_banned
    fields: [player_id, reason, by, duration]

  - id: core:chat_message
    fields: [player_id, message, channel]

  - id: core:server_announcement
    fields: [message]
```

### Chat system

```yaml
chat:
  channels:
    - id: global
      prefix: "[Global]"
      color: "#ffffff"
      range: unlimited

    - id: local
      prefix: "[Local]"
      color: "#aaaaaa"
      range: 64              # blocks — only nearby players see it

    - id: party
      prefix: "[Party]"
      color: "#55ff55"
      range: unlimited
      requires: party_membership

    - id: admin
      prefix: "[Admin]"
      color: "#ff5555"
      range: unlimited
      requires: admin_role
```

Mods can register additional chat channels. Chat messages flow through the event bus
and can be intercepted by mod Lua scripts (e.g., chat commands, profanity filter).

---

## Lua Multiplayer API

```lua
-- Player queries
Server.get_players()                         -- all connected players
Server.get_player(uuid)                      -- specific player
Server.get_player_count()                    -- current count
Server.get_max_players()                     -- configured max

-- Messaging
Server.broadcast(message)                    -- server announcement to all
Server.send_message(player, message)         -- direct message to player
Server.send_chat(player, channel, message)   -- chat message as server

-- Administration
Server.kick(player, reason)
Server.ban(player, reason, duration)
Server.set_role(player, role)
Server.get_role(player)

-- Network state (read-only for mods)
player:get_ping()                            -- round-trip time in ms
player:is_local()                            -- true if this is the listen server host
```
