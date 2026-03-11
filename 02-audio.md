# Audio Architecture

---

## Decision: Layered Two-Library System

| Layer | Library | Responsibility |
|-------|---------|---------------|
| Layer 1 — Core game audio | Godot AudioServer (built-in) | All SFX, ambient loops, 3D positional audio, UI sounds, mod audio |
| Layer 2 — Adaptive soundtrack | FMOD Studio (fmod-for-godot plugin) | Biome-reactive music, combat stingers, seamless transitions |
| Layer 3 — Mod audio (Lua) | Godot AudioServer via Lua API wrapper | Mod-supplied OGG/WAV files loaded at runtime |

---

## Layer 1 — Godot AudioServer

The built-in Godot 4 audio system handles the majority of audio needs with zero additional dependencies.

**Capabilities used:**
- `AudioStreamPlayer3D` — positional audio with distance attenuation, Doppler effect, and area-based reverb zones
- `AudioBus` system — route sounds through effects chains (reverb, EQ, compressor, limiter)
- `AudioStreamRandomizer` — natural variation for repeated sounds (footsteps, impacts, ambient)
- Background streaming for long soundtrack files — no memory spike
- `AudioEffectCapture` — real-time DSP for procedural audio effects
- Mouse capture and audio focus handled by Godot's OS abstraction

**Mod audio via AudioServer:**
Mods supply OGG/WAV/MP3 files in their mod pack. The game's Lua API exposes AudioServer functionality:

```lua
-- Exposed Lua audio API for mods
Audio.play_sound("res://mods/mymod/sounds/swing.ogg", position)
Audio.play_music("res://mods/mymod/music/theme.ogg", true)  -- loop = true
Audio.set_volume("master", 0.8)
Audio.register_ambient("mymod:cave_wind", "res://mods/mymod/sounds/cave.ogg")
```

---

## Layer 2 — FMOD Studio

FMOD Studio handles the adaptive music system. The audio designer works in FMOD Studio's
visual tool — no code required for music authoring.

**Plugin:** `fmod-for-godot` by Alessandro Fama (github.com/alessandrofama/fmod-for-godot)

**Capabilities used:**
- **Parameters** — map gameplay state to audio properties: biome, health, combat intensity, weather
- **Adaptive music** — seamless loop transitions, stingers, and layering between states
- **Spatializer** — HRTF, occlusion, distance curves, reverb zones for cinematic sound
- **Banks** — efficient audio asset packaging and streaming

**Example parameter-driven music states:**
- `biome` parameter: `forest` → `desert` → `cave` → `crystal_tundra`
- `combat_intensity` parameter: `0.0` (peaceful) → `1.0` (boss fight)
- `weather` parameter: `clear` → `rain` → `storm`

**Licensing:** Free for revenue ≤ $200K/year; ~$1,500/year beyond that.

**Mod audio and FMOD:** Mod audio does NOT use FMOD. FMOD is only for the core adaptive
soundtrack. Modders use the Godot AudioServer API (Layer 1) for their audio content.

---

## Audio Format Strategy

| Format | Use case | Notes |
|--------|----------|-------|
| OGG Vorbis | Everything — default | Best size/quality. Godot-native. All SFX and music. |
| WAV (PCM 16-bit) | Short latency-critical SFX | Footsteps, hits, UI clicks — where decode latency matters |
| MP3 | Mod audio fallback | Widely known to modders; Godot 4 supports it |
| FMOD Bank (`.bank`) | Adaptive soundtrack only | FMOD's packaged format; not exposed to modders |

**Recommended mod audio format:** OGG Vorbis. Accept MP3 as fallback. Document this explicitly in modding guide.

---

## Input → Audio Connection

Audio triggered by input events (sword swing, footstep on block type) is handled in
entity/item Lua behavior scripts:

```lua
function CrystalSword.on_hit(player, item, target, damage)
  Audio.play_sound("mymod:sounds/crystal_hit.ogg", target:position())
end
```

FMOD combat parameter is updated by the game's combat system:
```gdscript
# GDScript — update FMOD parameter when combat state changes
FMODStudio.set_parameter("combat_intensity", current_combat_intensity)
```

---

## Rejected Options

- **Wwise** — more powerful spatial audio (geometry-based reverb) but proprietary `.wem` format makes mod audio nearly impossible; Godot integration less mature; licensing complex. Skip.
- **OpenAL Soft** — Godot 4 no longer uses it; no adaptive music; no tooling. Skip.
- **miniaudio** — considered as a Lua-side mod audio layer (direct FFI calls), but Godot AudioServer exposed via Lua API covers the same need more cleanly.
