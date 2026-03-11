# Asset Pipeline

The full pipeline from authoring tool to in-game asset — for both internal development
and community modders.

---

## Pipeline Overview

```
Model          Texture          Animate          Export       Import / Load
──────         ───────          ───────          ──────       ─────────────
Blender    →   Krita        →   Blender      →   .glb     →   Godot editor (pre-import)
MagicaVoxel    Substance        Mixamo           (.glb)        GLTFDocument (runtime / mods)
               Blender Paint    (retarget)
```

**Single enforced format:** glTF 2.0 binary (`.glb`) for all 3D content — meshes, materials,
armatures, animations, and blend shapes in one file. This is the format for both internal
assets and mod-supplied assets.

---

## Static Model Tools

### Blender 4.x — Primary Tool (Recommended for modders)
- **Cost:** Free (GPL)
- **Platforms:** Windows, macOS, Linux
- **Export:** glTF 2.0 — lossless round-trip with Godot's importer
- **Key features:**
  - Geometry Nodes for procedural prop variation (foliage, rocks) without per-asset work
  - Cycles/EEVEE for baking lightmaps and ambient occlusion to texture
  - Python scripting for batch export pipelines
  - Largest tutorial ecosystem specifically for Godot+Blender workflows
- **Workflow:** File → Export → glTF 2.0 (`.glb`) → drag into Godot project

### Godot CSG Nodes — Prototyping Only
- Built-in constructive solid geometry for blockouts and very simple mod props
- Not for production assets; convert to `MeshInstance3D` before shipping
- No UV control, no LOD authoring

---

## Voxel-Specific Tools

### MagicaVoxel — Recommended for modders
- **Cost:** Free
- **Platforms:** Windows, macOS (no Linux — recommend Goxel for Linux users)
- **Export:** glTF 2.0, OBJ, PLY
- **Notes:** Industry-standard free voxel editor; built-in path tracer for marketing renders; 256-colour palette maps well to voxel material registries. Point modders here explicitly in documentation.
- **Workflow:** MagicaVoxel → Export glTF → Import Godot → assign PBR materials or use vertex colours

### Goxel — Linux / browser fallback
- **Cost:** Free (GPL)
- **Platforms:** Windows, macOS, Linux, **Web browser**
- **Export:** glTF, OBJ, PLY
- **Notes:** Less polished than MagicaVoxel but runs in browser (zero install). Recommend to Linux modders.

### Qubicle — Internal artists only
- **Cost:** ~$30 (Steam)
- **Export:** OBJ, FBX (no native glTF — requires Blender cleanup step)
- **Notes:** Better multi-model management; used by commercial voxel titles. Pipeline: Qubicle → OBJ → Blender → glTF → Godot

---

## Animation & Rigging

### Blender Armatures — Primary rigging tool
- glTF 2.0 export carries skeleton, skin weights, shape keys (blend shapes), and all
  animation actions in a single file
- Godot imports as `AnimationPlayer` + `Skeleton3D` nodes automatically
- **Rigify add-on** provides production-ready humanoid rigs in minutes
- **NLA Editor** for non-destructive animation clip authoring
- Action system maps 1:1 with Godot's `AnimationPlayer` clips
- Shape keys (blend shapes) for facial animation and morph targets

### Godot AnimationPlayer — In-engine animation
- Use for: procedural animation, UI animations, state machine setup, shader parameter tweens
- `AnimationTree` + blend spaces for character locomotion blending
- `AnimationRetargeting` (Godot 4.2+) for applying third-party clips to custom skeletons
- Not suitable for complex authored character animation — use Blender for that

### Mixamo (Adobe) — Animation library
- **Cost:** Free (Adobe ID required)
- Hundreds of free motion-captured humanoid animations
- **Workflow:** Mixamo → FBX → Blender (retarget to custom rig) → glTF → Godot
- Note: Mixamo skeleton does not match Godot's humanoid layout exactly — retargeting step is required
- Only for humanoid characters; Adobe could restrict access (no offline fallback)

---

## Texture & Skin Tools

### Krita 5.x — Recommended for modders
- **Cost:** Free (GPL)
- **Platforms:** Windows, macOS, Linux
- **Output:** PNG, EXR, TIFF
- **Notes:** Best free option for painted textures and character skins. Wrap-around mode for seamless tileable textures. Python scripting for batch texture generation.
- **Workflow:** Krita → PNG exports (albedo, normal, roughness, metallic) → Godot `BaseMaterial3D`

### Substance 3D Painter — Internal artists
- **Cost:** ~$220/year (indie)
- Smart materials with procedural wear/damage/rust layers
- Bakes AO, normal, and curvature from high-poly reference meshes
- **Godot export preset** available — outputs correct PBR channel packing
- Do not require this of modders; Krita is the community standard

### Blender Texture Paint — Single-tool pipeline
- Paint directly on the 3D mesh surface in the viewport
- Zero additional software for modders already in Blender
- Less refined brush engine than Krita for 2D texture work

### Aseprite / Libresprite — Pixel art
- **Cost:** $19.99 (Aseprite) / Free (Libresprite fork)
- For pixel-art style textures, UI icons, and inventory art
- Animated sprite sheets export directly to Godot `SpriteFrames`

---

## PBR Texture Map Convention

All 3D materials use Godot's `BaseMaterial3D` (or custom shaders) with standard PBR maps.

| Map | Channels | Notes |
|-----|----------|-------|
| Albedo | RGB | Base colour; no lighting baked in |
| Normal | RGB (OpenGL convention) | Godot expects OpenGL-style normals |
| Roughness | R (greyscale) | 0 = mirror, 1 = fully rough |
| Metallic | R (greyscale) | 0 = dielectric, 1 = metal |
| Emission | RGB | Self-illumination; used for crystal glow, lava, etc. |
| AO | R (greyscale) | Baked ambient occlusion; optional |

**File naming convention (enforced in mod validator):**
```
block_name.albedo.png
block_name.normal.png
block_name.roughness.png
block_name.metallic.png
block_name.emission.png   (optional)
```

---

## Godot Import Pipeline

### Editor import (pre-shipped assets)
1. Drop `.glb` file into project directory
2. Godot auto-imports — creates `.import` metadata file
3. Configure import settings: generate LOD, create collision mesh, animation clips to import
4. Asset is available as a `PackedScene` or `ArrayMesh` resource

### Runtime import (mod assets)
Uses `GLTFDocument.append_from_file()` — available since Godot 4.2.

```gdscript
# ModAssetLoader.gd
func load_glb(path: String) -> PackedScene:
    if _cache.has(path):
        return _cache[path]

    var doc  = GLTFDocument.new()
    var state = GLTFState.new()
    var err  = doc.append_from_file(path, state)

    if err != OK:
        push_error("Failed to load mod asset: " + path)
        return null

    var scene = doc.generate_scene(state)
    _cache[path] = scene
    return scene
```

**Caching:** First load from disk; subsequent spawns reuse the cached `PackedScene`. Mod
assets are evicted from cache when the mod is disabled or the game exits.

### Accepted mod asset formats

| Format | Accepted | Notes |
|--------|----------|-------|
| `.glb` | ✅ Required | Primary format |
| `.png` | ✅ | Textures, icons, UI |
| `.ogg` | ✅ | Audio — preferred |
| `.wav` | ✅ | Short SFX |
| `.mp3` | ✅ | Fallback audio |
| `.yaml` | ✅ | Data definitions |
| `.lua` | ✅ | Behavior scripts |
| `.fbx` | ❌ | Internal only; not mod-facing |
| `.blend` | ❌ | Source files; not distributed |
| `.psd` / `.kra` | ❌ | Source files; not distributed |
| Any executable | ❌ | Rejected by validator |
