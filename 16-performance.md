# Performance Targets

---

## Design Philosophy

The game must run well on a Steam Deck at 60 FPS with appropriate settings. Higher-end
hardware unlocks visual quality and render distance. The rendering pipeline is designed
in layers — each visual feature can be independently disabled without breaking the game's
look or feel. The voxel light grid ensures the game looks good even with all post-processing
disabled.

---

## Target Hardware Tiers

| Tier | Hardware | Resolution | Target | Example hardware |
|------|----------|-----------|--------|-----------------|
| **Minimum (Steam Deck)** | Handheld APU, 16 GB unified | 720p-800p | 60 FPS | Steam Deck (Zen 2, RDNA 2, 16 GB) |
| **Low** | Integrated GPU, 4-core, 8 GB | 1080p | 60 FPS | Intel i5-8250U + UHD 620, Ryzen 5 5600G |
| **Recommended** | Mid-range discrete GPU, 6-core, 16 GB | 1080p | 60 FPS | Ryzen 5 3600 + RTX 2060, RX 6600 |
| **High** | Current-gen GPU, 8-core, 32 GB | 1440p | 60 FPS | Ryzen 7 5800X + RTX 3070, RX 6800 |
| **Ultra** | High-end GPU, 8+ core, 32 GB | 4K | 60 FPS | Ryzen 9 7900X + RTX 4080, RX 7900 XT |

### Steam Deck baseline

The Steam Deck is the minimum spec reference device. All design decisions must account
for it:

- **GPU:** RDNA 2, 8 CUs, ~1.6 TFLOPS — modest by desktop standards
- **CPU:** Zen 2, 4 cores / 8 threads, 2.4-3.5 GHz — limited single-thread
- **RAM:** 16 GB unified (shared CPU/GPU) — memory efficiency matters
- **Display:** 800p (1280×800) — lower resolution helps GPU budget significantly
- **TDP:** 4-15W envelope — thermal throttling is real, sustained load must be managed
- **Storage:** NVMe SSD — chunk loading from disk is fast

**Steam Deck target profile:**
- 800p, Low preset, render distance 8-10
- SDFGI off (too expensive for 8 CUs)
- Voxel light grid provides all illumination (still looks good)
- Simplified water, reduced particles
- LOD aggressive
- 60 FPS sustained

---

## Render Distance

Player-configurable. Default varies by detected hardware tier.

| Setting | Distance | Chunks loaded (approx) | Default for |
|---------|----------|----------------------|-------------|
| 6 chunks (96 blocks) | Short | ~1,800 | — |
| 8 chunks (128 blocks) | Moderate | ~3,200 | Steam Deck, Minimum |
| 12 chunks (192 blocks) | Comfortable | ~7,200 | Low, Recommended |
| 16 chunks (256 blocks) | Open | ~12,800 | High |
| 24 chunks (384 blocks) | Expansive | ~28,800 | Ultra |
| 32 chunks (512 blocks) | Extreme | ~51,200 | Ultra (high-end) |
| 48 chunks (768 blocks) | Maximum | ~115,000 | Ultra (enthusiast hardware) |

**LOD is mandatory at high distances.** The godot_voxel plugin provides chunk LOD —
distant chunks render at reduced geometric resolution. Without LOD, render distances
above 16 are impractical on any hardware.

**LOD levels:**

| Distance from player | LOD level | Block resolution | Mesh detail |
|---------------------|-----------|-----------------|-------------|
| 0-8 chunks | LOD 0 | Full (1 block) | Full geometry |
| 8-16 chunks | LOD 1 | 2× (2-block voxels) | ~25% triangles |
| 16-24 chunks | LOD 2 | 4× | ~6% triangles |
| 24-48 chunks | LOD 3 | 8× | ~1.5% triangles |

LOD transitions use distance-based blending to avoid visible pop-in.

---

## Frame Rate Targets

| Target | Frame time | When |
|--------|-----------|------|
| 60 FPS | 16.6ms | All tiers, all settings |
| 30 FPS | 33.3ms | Fallback — never targeted, but acceptable during heavy load spikes |

**The game targets 60 FPS on all hardware tiers, including Steam Deck.** Quality
settings scale down to meet this target. There is no "30 FPS mode" — instead, settings
auto-adjust to maintain 60.

**Uncapped mode:** Players can disable the frame cap for high-refresh displays
(90, 120, 144, 240 Hz). Vsync on by default, toggleable.

### Adaptive quality (optional)

An optional dynamic quality system that adjusts settings in real-time to maintain
60 FPS:

1. Monitor frame time over a rolling 2-second window
2. If average exceeds 15ms (risk of dropping below 60): reduce quality one step
3. If average is below 12ms (headroom available): increase quality one step
4. Adjustable settings (in priority order):
   a. Particle density
   b. Shadow resolution
   c. SSAO sample count
   d. Entity draw distance
   e. Volumetric fog resolution
   f. Render distance (last resort)

Player can enable/disable adaptive quality. When disabled, settings are fixed at
whatever the player chose.

---

## Frame Time Budget

At 60 FPS, total budget is **16.6ms per frame**. Budget allocation:

### Main thread

| System | Budget | Notes |
|--------|--------|-------|
| Scene tree / Godot overhead | 1.0ms | Node processing, signals |
| Physics (Jolt) | 2.0ms | Collision, character movement, rigid bodies |
| AI / Lua scripts | 3.0ms | Entity ticks, mod scripts, event processing |
| Entity systems | 1.5ms | Buff ticks, ownership, skill cooldowns |
| Fluid simulation | 0.5ms | Cellular automaton for active fluid chunks |
| Event bus dispatch | 0.5ms | Command validation, event routing |
| UI update | 0.5ms | Control tree, binding refresh, HUD |
| Networking | 0.5ms | Packet send/receive, serialization |
| Main thread total | **9.5ms** | Leaves 7ms for GPU sync and overhead |

### Worker threads

These systems run on background threads and never block the main thread:

| System | Thread | Notes |
|--------|--------|-------|
| Chunk meshing | Worker pool | Greedy mesh + collision shape generation |
| Chunk generation | Worker pool | Terrain gen, structure placement, decorators |
| Light propagation | Worker | BFS flood fill, data texture upload queued |
| Navigation rebuild | Worker | Chunk portal recalculation |
| Asset loading | Worker | GLB import, texture loading |
| Audio | Godot AudioServer thread | Separate from main thread |

### GPU

| Pass | Budget (recommended) | Budget (Steam Deck) |
|------|---------------------|---------------------|
| Chunk rendering | 4.0ms | 6.0ms |
| Entity rendering | 1.0ms | 1.5ms |
| Shadow maps | 1.5ms | 0ms (off) |
| SDFGI | 2.0ms | 0ms (off) |
| SSAO | 0.5ms | 0ms (off) |
| SSR | 0.5ms | 0ms (off) |
| Volumetric fog | 0.5ms | 0ms (off) |
| Water shader | 0.5ms | 0.5ms (simple) |
| Particles | 0.5ms | 0.3ms (reduced) |
| Post-processing | 0.5ms | 0.3ms |
| UI rendering | 0.3ms | 0.3ms |
| GPU total | **11.8ms** | **8.9ms** |

---

## Quality Presets

### Preset definitions

| Setting | Deck | Low | Medium | High | Ultra |
|---------|------|-----|--------|------|-------|
| Resolution | 800p | 1080p | 1080p | 1440p | 4K |
| Render distance | 8 | 10 | 12 | 16 | 24-48 |
| SDFGI | Off | Off | Half-res | Full | Full + cascades |
| Shadows | Off | Low-res (1024) | Medium (2048) | High (4096) | High + soft (4096) |
| SSAO | Off | Off | Low (4 samples) | Medium (8) | High (16) |
| SSR | Off | Off | Off | Half-res | Full |
| Volumetric fog | Off | Off | Low-res | Medium | High |
| Water | Flat tinted | Animated surface | Surface + refraction | Full + caustics | Full + caustics |
| Particles | 25% | 50% | 75% | 100% | 100% |
| Entity draw distance | 32 blocks | 48 blocks | 64 blocks | 96 blocks | 128 blocks |
| LOD aggressiveness | Aggressive | Aggressive | Normal | Relaxed | Minimal |
| Light data texture | 16³ | 16³ | 18³ (padded) | 18³ | 18³ |
| Texture filtering | Bilinear | Bilinear | Trilinear | Anisotropic 4× | Anisotropic 16× |
| Anti-aliasing | FXAA | FXAA | TAA | TAA | TAA + MSAA 4× |

### Auto-detection

On first launch, the game detects hardware and selects the appropriate preset:

1. Query GPU vendor, model, VRAM
2. Query CPU core count, frequency
3. Query total system RAM
4. Match against a hardware database (embedded, periodically updated)
5. If no match, run a 10-second benchmark (generate and render a test scene)
6. Select preset that targets 60 FPS on detected hardware
7. Player can override at any time

### Individual overrides

Each setting in the preset is individually adjustable. The preset is a starting point.
Changing any setting switches the preset label to "Custom."

---

## Memory Budgets

### Target: 2 GB (Steam Deck / minimum), 4 GB (recommended and above)

| Resource | 2 GB budget | 4 GB budget | Notes |
|----------|------------|------------|-------|
| Chunk block data | ~80 MB | ~200 MB | Palette + RLE, scales with render distance |
| Chunk meshes (GPU) | ~120 MB | ~400 MB | Vertex data, LOD reduces distant chunks |
| Light data textures (GPU) | ~30 MB | ~75 MB | 23 KB × loaded chunk count |
| Entity data | ~30 MB | ~60 MB | Active + dormant entities |
| Mod Lua VMs | ~64 MB | ~128 MB | 32 MB per mod default, configurable |
| Audio buffers | ~60 MB | ~100 MB | Streaming music, loaded SFX |
| Asset cache (mod GLBs) | ~80 MB | ~250 MB | LRU cache, evicts when full |
| SQLite page cache | ~20 MB | ~50 MB | Read cache for chunk loading |
| Godot engine overhead | ~150 MB | ~200 MB | Scene tree, resources, renderer state |
| SDFGI probes | 0 MB | ~100 MB | Only when SDFGI enabled |
| Shadow maps | 0-20 MB | ~50 MB | Scales with shadow quality |
| OS / headroom | ~350 MB | ~400 MB | Breathing room |
| **Total** | **~2 GB** | **~4 GB** | |

### Memory pressure handling

When memory usage approaches the budget:
1. Reduce asset cache size (evict least-recently-used mod assets)
2. Unload dormant entity visual data (keep game state, free mesh/texture)
3. Reduce chunk load distance by 1 ring (temporary, restores when memory frees)
4. Log warning in debug overlay

Never crash due to memory — gracefully degrade render distance and cache size.

---

## Entity Performance

| Budget | Value | Notes |
|--------|-------|-------|
| Max active entities (ticking AI) | 512 | Default, configurable per world |
| Max dormant entities (saved, not ticking) | 4096 | In loaded chunks, no AI cost |
| Max entities per chunk | 32 active | Spawn system respects this cap |
| AI tick budget per entity | 0.1ms | Lua script execution |
| AI tick budget total per frame | 3.0ms | Across all mods and entities |
| Entity draw distance | 32-128 blocks | Varies by quality preset |
| Entity LOD | 2 levels | Reduced mesh at distance, billboard at far |

### Entity LOD

| Distance | Render mode |
|----------|------------|
| 0-32 blocks | Full mesh, full animation |
| 32-64 blocks | Simplified mesh, reduced animation rate (half keyframes) |
| 64-128 blocks | Billboard sprite (pre-rendered from mesh) |
| >128 blocks | Not rendered |

---

## Chunk Loading Performance

| Operation | Target time | Thread |
|-----------|------------|--------|
| Chunk generation (terrain + ores + caves) | <10ms | Worker |
| Chunk meshing (greedy mesh + collision) | <5ms | Worker |
| Light propagation (single source BFS) | <1ms | Worker |
| Chunk load from SQLite | <2ms | Worker |
| Chunk save to SQLite | <2ms | Worker |
| Data texture upload | <0.1ms | Main (GPU upload) |
| Collision shape upload | <0.1ms | Main |

### Chunk loading rate

| Scenario | Rate | Notes |
|----------|------|-------|
| Player login (initial load) | 20 chunks/sec | Ring 0 priority, rest background |
| Player moving (streaming) | 8-12 chunks/sec | Load ahead in movement direction |
| Teleport (bulk load) | 20 chunks/sec | Same as login |

Player can enter the world when Ring 0 (1 chunk) is ready. Full render distance
populates over 5-30 seconds depending on distance setting.

### Worker thread pool

Thread count auto-detected from CPU core count:

| CPU cores | Worker threads | Allocation |
|-----------|---------------|------------|
| 4 (Steam Deck) | 2 workers | 1 meshing, 1 generation/loading |
| 6 | 3 workers | 2 meshing, 1 generation/loading |
| 8 | 4 workers | 2 meshing, 2 generation/loading |
| 12+ | 6 workers | 3 meshing, 3 generation/loading |

Worker threads handle: chunk generation, chunk meshing, light propagation, navigation
rebuild, asset loading. Main thread never blocks on these operations.

---

## Networking Performance

| Budget | Value |
|--------|-------|
| Server tick rate | 20 ticks/sec (50ms per tick) |
| Server tick budget | 50ms (all game logic) |
| Bandwidth per client (typical) | ~50 KB/sec |
| Bandwidth per client (peak) | ~200 KB/sec (initial join, entering modified areas) |
| Bandwidth total (16 players) | ~800 KB/sec typical, ~3.2 MB/sec peak |
| Entity update rate | 20/sec per entity in interest area |
| Chunk stream throttle | 10 chunks/sec per client |
| Max packet size | 1400 bytes (MTU-safe) |

### Server hardware targets

| Players | CPU | RAM | Bandwidth |
|---------|-----|-----|-----------|
| 1-4 (listen server) | Any modern 4-core | 4 GB available | Home internet |
| 5-8 | 4-core 3+ GHz | 6 GB | 10 Mbps up |
| 9-16 | 6-core 3+ GHz | 8 GB | 25 Mbps up |

---

## Profiling and Monitoring

### Debug overlay (F3)

Real-time performance information available in development and release builds:

```
┌─ Performance ─────────────────────────┐
│ FPS: 62 (16.1ms)   GPU: 10.2ms       │
│ CPU main: 8.4ms    CPU workers: 3/4   │
│                                       │
│ Chunks: 3,247 loaded  42 meshing      │
│ Entities: 186 active  824 dormant     │
│ Draw calls: 1,842  Triangles: 2.4M   │
│                                       │
│ Memory: 2.8 GB / 4.0 GB              │
│   Chunks: 1.2 GB  Entities: 45 MB    │
│   Assets: 180 MB  Lua VMs: 86 MB     │
│                                       │
│ AI budget: 2.4ms / 3.0ms             │
│   core: 1.2ms (48 entities)          │
│   crystal_mod: 0.8ms (12 entities)   │
│   mymod: 0.4ms (6 entities)          │
│                                       │
│ Network: ↑4.2 KB/s  ↓48.1 KB/s       │
│ Ping: 34ms                            │
└───────────────────────────────────────┘
```

### Performance logging

Frame time spikes (>20ms) are logged with a breakdown of which system exceeded budget.
Logs are written to `user://logs/performance.log` and available for bug reports.

### Mod performance attribution

Each mod's CPU time is tracked separately:
- AI tick time per mod
- Lua script execution time per mod
- Event handler time per mod
- Custom component tick time per mod

Mods exceeding their budget are flagged in the debug overlay. Repeat offenders can be
force-throttled (reduced tick rate) by the server.

---

## Optimization Strategies

### Rendering

- **Greedy meshing:** Reduces triangle count by 80-90% vs naive per-face
- **Chunk LOD:** Distant chunks at reduced resolution via godot_voxel
- **Frustum culling:** Godot built-in — chunks outside camera view not rendered
- **Occlusion culling:** Godot 4 OccluderInstance3D — chunks behind mountains not rendered
- **Instanced rendering:** Identical decorators (flowers, grass) use GPU instancing
- **Texture atlas:** Block textures packed into a single atlas to minimize draw calls
- **Entity LOD:** Simplified meshes at distance, billboards at far distance

### World data

- **Palette + RLE:** Chunk block data compressed 10-50× vs raw array
- **Homogeneous chunk skip:** All-air chunks stored as a flag, not allocated
- **Dirty tracking:** Only modified chunks are re-serialized
- **SQLite page cache:** Hot chunks served from memory, not disk

### Game logic

- **Dormancy:** Entities beyond active distance stop ticking (zero CPU)
- **Budget enforcement:** Lua scripts throttled when exceeding budget
- **Spatial indexing:** Entity queries use spatial hash, not linear scan
- **Fluid settling:** Settled fluid chunks removed from tick list
- **Batched light updates:** Multiple light changes per frame in single BFS pass
- **Navigation caching:** Path results cached per chunk until terrain changes

### Memory

- **LRU asset cache:** Mod GLB meshes evicted when cache is full
- **Streaming audio:** Music streamed from disk, not loaded into memory
- **Texture streaming:** Distant chunk textures at reduced mip level
- **Entity visual unloading:** Dormant entities free mesh/texture, keep game state
