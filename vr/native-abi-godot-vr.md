---
tags:
  - architecture
  - godot
  - vr
  - abi
---

# Converting a game to VR with a native ABI and Godot

An ABI-driven VR conversion keeps an existing game simulation authoritative and
uses Godot as a new input, presentation, and platform layer. Instead of porting
gameplay rules into Godot, compile or wrap the game as a native library, expose
a small C-compatible interface, and let Godot send player intent and render the
resulting state.

This approach is a strong fit when a game already has valuable, tested behavior
that should not be rewritten: combat rules, AI, progression, random-number
sequences, physics, saved-game semantics, or replay compatibility. It is also
useful when the original renderer or input system cannot support VR cleanly.

The central rule is simple:

> The native game owns outcomes. Godot owns how the player expresses intent and
> how the result is presented in VR.

## Architecture at a glance

```text
OpenXR headset, controllers, hands, and Godot UI
        |
        | tracked poses, buttons, menu choices
        v
Godot frontend
        | input mapping and coordinate conversion
        | native binding and session ownership
        |
        | stable C ABI
        v
Native host adapter
        | wire types <-> engine-neutral game types
        v
Existing game simulation
        | gameplay, AI, RNG, collision, progression, time
        v
Snapshots and one-shot events
        |
        v
Godot rendering, audio, haptics, menus, and platform services
```

The dependency direction should point toward the simulation. The simulation
must not import Godot, OpenXR, or frontend types. Godot should not receive raw
pointers into live game objects or call internal gameplay functions directly.

This produces three replaceable pieces:

- The simulation can continue powering its original desktop frontend, tests,
  servers, replays, or other hosts.
- The ABI can support Godot, test harnesses, and tooling without knowing their
  implementation language.
- The VR frontend can change layout, controls, rendering, and comfort behavior
  without becoming a second implementation of the game.

## Decide what remains authoritative

Before writing bindings, draw a hard ownership boundary.

The native game should normally retain:

- game time and fixed-step advancement;
- player movement, weapons, damage, and collision;
- enemies, spawning, AI, and pathfinding;
- random-number generation;
- inventory, perks, progression, and scoring;
- mode state, win/loss conditions, and save semantics; and
- replay recording or verification, if available.

Godot should normally own:

- OpenXR startup and tracked-device access;
- translating hand, head, and button state into game intent;
- world scale, recentering, seated/standing modes, and comfort settings;
- meshes, sprites, animation presentation, shaders, particles, and lighting;
- spatial audio, haptics, and VR-native menus;
- interpolation between simulation ticks; and
- platform packaging and store integration.

Some systems sit near the boundary. Treat cosmetic particles, camera shake,
reticles, controller vibration, and UI animation as presentation. Treat anything
that can change a hit, spawn, score, RNG draw, or future decision as simulation.

## Make the game embeddable

The cleanest native library exposes a host-neutral runner with a lifecycle like:

```text
create(config) -> session
tick(session, fixed_input) -> small_result
snapshot(session) -> current_state
events(session) -> things that happened this tick
destroy(session)
```

If the existing game is tightly coupled to its window, renderer, or input API,
first extract an engine-neutral simulation seam. Avoid placing Godot calls
inside the old update loop. The ABI should wrap the seam, not become the seam.

Build the adapter as a shared library:

- `.dll` on Windows;
- `.so` on Linux and Android; or
- `.dylib` on macOS, if that platform is supported.

Keep the native host target free of the old renderer and windowing dependencies.
This reduces load-time failures and makes cross-compilation much easier.

## Why use a C ABI

A C ABI is usually the smallest stable bridge between a native simulation and
Godot. C symbols can be consumed by C#, GDExtension, Rust, C++, Zig, and most
FFI-capable languages without exposing compiler-specific object layouts.

The public boundary should use:

- fixed-width integers such as `uint32_t` and `int64_t`;
- `float` only when both sides explicitly agree on 32-bit IEEE representation;
- opaque integer handles rather than object pointers;
- explicit pointer-plus-length pairs for strings and buffers;
- flat structs with documented alignment;
- numeric IDs instead of native enums; and
- return codes plus a separate error-message function.

Avoid C++ classes, exceptions, templates, standard-library containers, native
strings, callbacks with unclear lifetimes, and pointers into movable internal
storage.

### A minimal contract

```c
#include <stdint.h>

#define GAME_HOST_ABI_VERSION 1u
#define GAME_HOST_OK 0
#define GAME_HOST_E_BUFFER_TOO_SMALL (-3)

typedef uint64_t game_session_t;

typedef struct game_input {
    float move_x;
    float move_y;
    float aim_x;
    float aim_y;
    uint32_t buttons;
} game_input_t;

typedef struct game_tick_result {
    uint32_t ticks_advanced;
    uint32_t paused;
    uint32_t game_over;
    int32_t score;
} game_tick_result_t;

uint32_t game_host_abi_version(void);

int32_t game_session_create(const uint8_t *config_json,
                            uint32_t config_len,
                            game_session_t *out_session);

int32_t game_session_tick(game_session_t session,
                          const game_input_t *inputs,
                          uint32_t input_count,
                          game_tick_result_t *out_result);

int32_t game_snapshot(game_session_t session,
                      uint8_t *buffer,
                      uint32_t *inout_len);

int32_t game_events(game_session_t session,
                    uint8_t *buffer,
                    uint32_t *inout_len);

void game_session_destroy(game_session_t session);
int32_t game_last_error(uint8_t *buffer, uint32_t len);
```

This is intentionally procedural. The native side remains free to use rich
internal types; only the adapter performs translation.

## Versioning and layout rules

Treat the ABI as a versioned network protocol that happens to run in-process.
An incorrect layout may not crash immediately; it can produce plausible but
wrong positions, counts, or flags.

Use these rules:

1. Export an ABI version function and check it before creating a session.
2. Put a magic number and version in every variable-length payload header.
3. Define alignment and byte order. Little-endian, 4-byte fields are a simple
   cross-platform choice for common desktop and standalone-VR targets.
4. Append fields where possible. Never silently reorder existing fields.
5. Update the native definition, public C header, and Godot mirror together.
6. Add size and offset assertions on both sides.
7. Fail closed when a native library and frontend disagree.

Opaque handles should contain or reference a generation number so a destroyed
session's stale handle cannot become valid when its slot is reused.

## Bind the ABI to Godot

There are two common binding tracks.

### C# and P/Invoke

Godot .NET can call the C ABI directly with `LibraryImport` or `DllImport`.
Mirror each wire struct with sequential layout:

```csharp
using System.Runtime.InteropServices;

[StructLayout(LayoutKind.Sequential)]
public struct GameInput
{
    public float MoveX;
    public float MoveY;
    public float AimX;
    public float AimY;
    public uint Buttons;
}

internal static partial class NativeGame
{
    private const string LibraryName = "game_host";

    [LibraryImport(LibraryName, EntryPoint = "game_host_abi_version")]
    internal static partial uint AbiVersion();

    [LibraryImport(LibraryName, EntryPoint = "game_session_tick")]
    internal static partial int Tick(
        ulong session,
        ReadOnlySpan<GameInput> inputs,
        uint inputCount,
        out GameTickResult result);
}
```

Add a native-library resolver when development builds store platform binaries
under project-specific directories. On mobile, ensure the library is packaged
where the platform loader expects it.

### GDExtension

Use a small GDExtension wrapper when the project uses GDScript, needs editor-
visible native classes, or cannot depend on Godot .NET on a target platform.
The GDExtension should still call the same C ABI. Do not let it grow into a
second native gameplay layer.

Whichever track is chosen, create a managed Godot-side session class that owns:

- the native handle;
- version negotiation;
- reusable snapshot and event buffers;
- conversion of return codes to clear errors;
- restart and disposal behavior; and
- strict buffer-view lifetimes.

Keep raw FFI calls out of gameplay-facing Godot scripts.

## Design the input path

VR input is not a direct replacement for mouse and keyboard input. Convert
tracked poses into the same intent the simulation already understands.

A typical path is:

1. Sample controller and head poses in Godot.
2. Project a hand ray or position onto a control surface.
3. Convert from Godot metres to the game's coordinate system.
4. Apply dead zones, handedness, button edges, and comfort rules.
5. Produce one immutable input struct for the next simulation tick.
6. Send only that struct through the ABI.

Keep coordinate conversion and input mapping as pure functions. They can then
be tested without a headset or a running native library.

Document every field semantically. For example, `move_x/move_y` might mean an
analog direction, a velocity request, or an absolute destination. Identical
types do not imply identical meaning.

### Coordinate spaces

Name each space explicitly:

- tracking space: poses reported by OpenXR;
- Godot world space: metres and the active scene transform;
- playfield-local space: coordinates relative to a tabletop, cabinet, or room;
- game space: the simulation's original units and axis conventions.

Use one mapper for all transformations and test corners, center points, rotated
playfields, scale changes, and out-of-bounds clamping. A common 2D-to-tabletop
mapping is:

```text
game x -> Godot local x
game y -> Godot local z
game origin -> one playfield corner
game extent -> playfield side in metres
```

Do not feed headset scale, recenter offsets, or rendering interpolation back
into the simulation unless they represent an explicit gameplay input.

## Keep simulation time independent of headset refresh

Most VR displays render faster than an older game's simulation rate. Run the
simulation at its original fixed rate and render independently.

For example:

- simulation: 60 Hz;
- headset: 72, 90, or 120 Hz;
- Godot physics callback: advances fixed simulation ticks;
- Godot render callback: interpolates between the two latest snapshots.

Do not pass variable headset-frame delta into a deterministic fixed-step game.
When a host frame must advance multiple simulation ticks, record and process
each advanced tick correctly. When paused, advance zero ticks instead of
inventing no-op replay rows.

## Separate state from events

The frontend needs both persistent state and one-shot occurrences.

Snapshots describe what is true now:

- player transforms and status;
- enemies and projectiles;
- pickups and persistent effects;
- animation phases;
- game-mode and HUD state; and
- stable presentation identities.

Events describe what happened during a tick:

- shots, impacts, reloads, and voice lines;
- music transitions;
- particles that should be spawned once;
- terrain decals or corpse stamps;
- haptic cues; and
- achievements or platform notifications.

Missing an intermediate full snapshot is usually harmless because the next
snapshot replaces it. Missing an event loses an effect, so event streams must be
drained after every simulated tick.

Keep static session metadata separate as well: map dimensions, terrain seed,
tileset IDs, level identity, or asset-pack selection can often be queried once.

## Give terminal UI explicit ownership

A terminal simulation flag and a visible results panel are different states.
Define who owns ticking, input, audio, and rendering during every transition:

```text
live play -> death animation -> name entry/results -> route or restart
   tick            tick               frozen
   input           neutral            panel only
   events          drained            no gameplay events
```

This prevents a subtle VR failure: the frontend hides the player and presents
the score screen while the simulation continues underneath it. Enemies can keep
attacking the hidden corpse, positional hit sounds continue, and a second death
cue may play even though the run has already ended.

Use one ordered screen-ownership decision near the start of the host frame. A
modal first-run prompt, fatal error, name-entry keyboard, or results panel should
either stop simulation advancement or explicitly define the limited cinematic
updates that remain allowed. Do not rely only on input suppression; neutral
input does not stop AI, collisions, timers, or one-shot audio.

Keep the death cinematic separate from results ownership. It may be desirable
for the world and corpse animation to tick for a short pacing window. At the
handoff to results, capture the final score and statistics, drain the last
intended events, then freeze the simulation until the player chooses a route.

Test terminal paths that bypass ordinary damage. Perks, scripts, timers, debug
commands, disconnects, and quest rules may set health or outcome state directly
without producing the normal damage event or death sound. Prefer a canonical
simulation event or transition record; if presentation supplies a fallback cue,
deduplicate it against recent native events.

The regression matrix should include:

- ordinary lethal damage;
- direct perk or script death;
- death with pending level-up choices;
- pause opened immediately before death;
- ranked and unranked score paths;
- quest success and failure panels; and
- network terminal state with late packets or reconnect activity.

For each case, assert both the visible route and the absence of post-panel
gameplay events.

## Use flat, reusable snapshot buffers

Copying thousands of entities into individual managed objects every tick creates
avoidable allocation and garbage-collection pressure. A practical payload is:

```text
SnapshotHeader
PlayerState[player_count]
EnemyState[enemy_count]
ProjectileState[projectile_count]
PickupState[pickup_count]
EffectState[effect_count]
```

Use a size-then-fill protocol:

- a null or empty buffer returns the required size;
- a short buffer reports the required size and a specific error;
- a large enough buffer receives the payload and written length.

If the maximum snapshot size is bounded, allocate two maximum-size buffers once
and alternate between them. Godot can retain the previous and current snapshots
for interpolation. In C#, `MemoryMarshal.Cast` can expose read-only spans over
the buffer without allocating per-entity wrappers.

Give pooled entities stable presentation identities. An array index alone is
not sufficient if slots are reused. A `(slot, generation)` pair lets the
renderer distinguish "the same enemy moved" from "a dead enemy's slot now holds
a newly spawned enemy."

## Rebuild presentation rather than gameplay

The snapshot is not a draw-command stream. It should expose facts, while Godot
chooses how those facts appear in VR.

Godot may:

- render 2D art as billboards or lifted cards;
- turn a flat map into a tabletop diorama;
- replace a screen-space HUD with world-space panels;
- adapt a cursor into hand reticles or controller rays;
- route sounds through `AudioStreamPlayer3D`;
- derive haptics from damage, firing, or reload events; and
- use `MultiMeshInstance3D` for large sprite populations.

Presentation-only fields such as tint, animation phase, effect age, or muzzle
flash intensity are reasonable ABI additions. Prefer exposing a stable fact
over reproducing gameplay logic in Godot to infer it.

If original assets are used, create a separate import pipeline that reads a
user-owned installation and generates Godot-ready textures, atlases, audio, and
manifests. Keep asset conversion independent from the simulation ABI.

## Replays and differential tests

The strongest proof that the new frontend remains thin is reproducibility.

If the game has replays, record the exact input consumed by every simulation
tick. A replay made through the ABI should verify through the same native path
as a replay made by the original frontend. Useful gates include:

- ABI-driven and directly driven sessions receive identical inputs and produce
  identical final summaries;
- identical configuration and input produce byte-identical snapshots;
- replay verification through the ABI matches the native verifier;
- reading snapshots never changes RNG or simulation state;
- input is recorded per advanced tick, not per rendered frame; and
- long recordings can be detached and encoded away from the render thread.

Without an existing replay system, build a deterministic scripted-input harness
and compare hashes or selected checkpoints. Also retain golden tests for mapper
math and packed-layout tests for the Godot bindings.

## Keep expensive work off the VR frame

VR is sensitive to stalls. Session initialization, level loading, replay
encoding, asset conversion, and large save operations should not run in the
headset's critical render path.

When a native result must outlive its session, move it into a separately owned
opaque handle. The main thread can detach it in constant time, restart gameplay,
and let a worker encode or write it. Specify which functions are thread-safe and
make ownership explicit: every detached handle must have one destroy path,
including after errors.

## Cross-platform loading and packaging

Test native loading on every target early. A desktop library loading successfully
does not prove an Android build is correct.

Check:

- CPU architecture and calling convention;
- exported symbol names;
- C runtime dependencies;
- thread-local storage and pthread requirements;
- minimum Android API level;
- library placement inside the APK;
- ahead-of-time restrictions for managed bindings;
- stripping that accidentally removes exports; and
- an ABI-version check against the exact library being packaged.

For standalone headsets, inspect the final APK rather than trusting the staging
directory. Verify that the native game library and required OpenXR vendor
libraries are present under the correct `lib/<abi>/` directory.

## A phased conversion plan

### Phase 1: prove the boundary

- Extract or identify a headless fixed-step simulation entry point.
- Build a native shared library for the desktop development target.
- Export version, create, one tick, a minimal snapshot, and destroy.
- Call it from a tiny Godot scene and display a status value.

### Phase 2: prove VR input

- Start OpenXR and track both controllers.
- Define tracking, playfield, and game coordinate spaces.
- Implement pure mapping and input functions with tests.
- Move and aim one simulated player without copying movement rules into Godot.

### Phase 3: prove presentation

- Render players, enemies, and projectiles from snapshots.
- Add double-buffered interpolation.
- Drain audio and one-shot effects after every tick.
- Establish stable entity identities and pooling behavior.

### Phase 4: prove determinism

- Drive the ABI with scripted or recorded inputs.
- Compare against the original host path.
- Add replay or checkpoint verification to continuous integration.
- Confirm snapshot reads are observational only.

### Phase 5: complete the product shell

- Build VR-native menus, pause flow, settings, and results screens.
- Add haptics, accessibility, seated/standing modes, and recentering.
- Integrate save data and asset import.
- Package and validate every target architecture.

## Common failure modes

| Failure | Typical cause | Prevention |
|---|---|---|
| Native library loads but state is nonsense | Struct layout or version mismatch | Startup handshake and size/offset tests |
| Replays diverge | Godot duplicated rules, used variable delta, or a read consumed RNG | Single simulation owner and replay gate |
| Movement feels wrong | Target point confused with direction or velocity | Document field semantics and test mapper math |
| Audio or decals disappear | One-shot events were sampled like persistent state | Drain events after every tick |
| Enemies attack beneath the results panel | Terminal UI suppressed input but did not own simulation ticking | Freeze at the cinematic-to-results handoff |
| Entities interpolate into unrelated spawns | Pool slots reused without generations | Stable `(slot, generation)` identities |
| Headset stutters at match end | Replay/save encoding performed synchronously | Detach ownership and process on a worker |
| Desktop works but headset cannot load native code | Wrong ABI, libc, APK location, or stripped exports | Inspect and preflight the final package |
| VR frontend becomes difficult to verify | Gameplay decisions leaked into Godot | Keep Godot limited to intent and presentation |

## Safe extension checklist

When Godot needs new information, classify it first:

- input: something the player intentionally requested;
- state: something currently true;
- event: something that happened once; or
- metadata: something stable for the session.

Then:

1. Add the smallest engine-neutral field or export.
2. Append fields rather than reordering payloads.
3. Update the native declaration, C header, and Godot binding together.
4. Bump the ABI version for layout or semantic changes.
5. Add native raw-buffer tests and Godot-side layout/decoder tests.
6. Rebuild every target library and reject stale binaries.
7. Record and verify a replay for anything that could affect outcomes.
8. Confirm the final desktop package or headset APK contains the tested binary.

The result should remain one game with multiple frontends. The ABI translates;
Godot presents; the existing simulation decides what happens.
