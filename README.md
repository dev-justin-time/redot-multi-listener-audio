# Multiple Current AudioListener3D — Split-Screen Audio for Redot Engine

**Proposal**: [Redot Engine Feature Request](https://github.com/Redot-Engine/redot-engine/issues)
**Implements**: Multiple simultaneous `AudioListener3D` nodes for per-player spatial audio in split-screen multiplayer.

## Overview

This feature allows multiple `AudioListener3D` nodes to be active simultaneously, solving the long-standing problem of broken 3D audio in split-screen multiplayer modes. Each player can now hear sound effects correctly spatialized relative to their own camera/listener.

## Design

Two minimal, backward-compatible additions:

### 1. `AudioListener3D.solo` (new boolean property)

| State | Behavior |
|---|---|
| `solo = true` (default) | Enabling this listener disables all others — **same as current engine behavior**. Existing projects are unaffected. |
| `solo = false` | Allows this listener to be current without disabling other listeners, enabling split-screen multi-listener setups. |

*When `solo` is toggled back to `true` on an already-current listener, it immediately enforces solo mode by disabling all others.*

### 2. `AudioStreamPlayer3D.listener_path` (new NodePath property)

| State | Behavior |
|---|---|
| `listener_path = NodePath()` (default empty) | Falls back to original behavior — spatialize relative to the viewport's current listener via the existing camera iteration loop. |
| `listener_path = ^"Player2/Listener"` | Spatializes **only** relative to that specific `AudioListener3D`. Other listeners (e.g. Player 1's) are skipped for this sound. |

## Files Changed

| File | Change |
|---|---|
| `scene/3d/audio_listener_3d.h` | Added `solo` member (bool, default `true`), `set_solo()`, `is_solo()`, `_find_listeners()` |
| `scene/3d/audio_listener_3d.cpp` | Modified `make_current()` to respect `solo`; added `set_solo()`, `is_solo()`, `_find_listeners()`; updated bindings |
| `scene/3d/audio_stream_player_3d.h` | Added `listener_path` member (NodePath), `set_listener_path()`, `get_listener_path()` |
| `scene/3d/audio_stream_player_3d.cpp` | Modified `_update_panning()` to use explicit listener when `listener_path` is set; added bindings |

## Usage Example (GDScript)

```gdscript
# Split-screen setup — two cameras with independent audio listeners
func _setup_split_screen():
    # Camera for Player 1 (left half)
    var cam1 = Camera3D.new()
    cam1.viewport = $ViewportContainer1/Viewport
    add_child(cam1)
    
    var listener1 = AudioListener3D.new()
    listener1.solo = false  # Allow other listeners
    listener1.make_current()
    cam1.add_child(listener1)
    
    # Camera for Player 2 (right half)
    var cam2 = Camera3D.new()
    cam2.viewport = $ViewportContainer2/Viewport
    add_child(cam2)
    
    var listener2 = AudioListener3D.new()
    listener2.solo = false  # Don't disable Player 1's listener
    listener2.make_current()
    cam2.add_child(listener2)
    
    # Bind Player 1's engine sound to their listener
    $Player1Engine.listener_path = listener1.get_path()
    
    # Bind Player 2's engine sound to their listener
    $Player2Engine.listener_path = listener2.get_path()
    
    # Global sounds (race countdown) use default — heard by all
    $RaceCountdown.play()
```

## Backward Compatibility

- **`solo` defaults to `true`** — any existing project that enables a single `AudioListener3D` works exactly as before.
- **`listener_path` defaults to empty** — any existing `AudioStreamPlayer3D` spatializes normally through the viewport's listener.
- **No new dependencies, no API breaks, no signal changes.**

## Testing

1. Compile the engine: `scons platform=windows target=editor`
2. Create a test scene with two Camera3D nodes in separate Viewports
3. Add an AudioListener3D as a child of each camera, set `solo = false`, `current = true`
4. Add AudioStreamPlayer3D nodes with `listener_path` pointing to each listener
5. Verify sound attenuates/pans correctly per-player
