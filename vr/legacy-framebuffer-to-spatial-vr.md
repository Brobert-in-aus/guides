---
tags:
  - architecture
  - game-porting
  - godot
  - vr
  - preservation
---

# Converting a legacy framebuffer game to spatial VR

A safe VR conversion does not have to begin by separating every gameplay
system from a legacy renderer. For an old fixed-step game whose update and draw
code are tightly interleaved, a better first move is often to host the original
game intact, intercept its completed framebuffer, and put that framebuffer in
VR. Once the game is playable through the new host, replace presentation one
category at a time with structured snapshots and spatial rendering.

This approach is useful when the original game has:

- a software-rendered or indexed-color framebuffer;
- simulation and drawing mixed inside one large loop;
- menus and cinematics that use the same drawing primitives as gameplay;
- timing behavior that must remain exact;
- useful demo, replay, or state-hash facilities; and
- distinctive 2D art that should survive the conversion.

The working rule is:

> Preserve the complete legacy presentation as a verified fallback, then
> spatialize only the parts you can identify, export, and test.

## Architecture at a glance

```text
OpenXR headset, tracked hands, controllers, and Godot shell
        |
        | abstract buttons and spatial targets
        v
Native host boundary
        |
        | input exchange at legacy present points
        v
Original game thread
        | fixed-step simulation
        | original menus, timing, audio, and renderer
        |
        +----> indexed framebuffer ------------------------+
        |                                                  |
        +----> semantic presentation records               |
               sprites, layers, text, sound, metadata      |
                                                           v
Godot presentation
        | legacy lane or fallback surface
        | spatial sprites and tile geometry
        | interpolation, palette shaders, audio, UI
        v
Headset at native refresh rate
```

This is a migration architecture, not merely a final architecture. The intact
framebuffer gets the game into VR early and remains the safety net while the
structured presentation path grows.

## Start at the present boundary

Most software-rendered games eventually call one function that means “show the
frame,” even if the preceding update loop is tangled. That function is the
highest-leverage interception point.

Turn each legacy present into a synchronization point:

1. The game finishes drawing into its normal framebuffer.
2. The native host publishes a copy or immutable view of that frame.
3. The host applies the latest abstract input state.
4. The game thread waits or continues according to its original pacing model.
5. Godot uploads the frame and presents it in VR.

This catches gameplay, menus, title screens, transitions, and cinematics with
very little initial refactoring. It also avoids the common trap of building a
beautiful gameplay scene that cannot reach the game's menus, save flow, or
level transitions.

Do not assume every present marks a completed gameplay tick. Pause screens and
in-game menus may present halfway through an interrupted tick. Export both a
monotonic publication number and the gameplay tick number so the frontend can
distinguish a new menu frame from a new simulation state.

## Host the original game on its own thread

Legacy game entry points often expect to own the process. They may block in a
menu, use thread-local state, allocate a large stack, call exit-like functions,
or pace themselves with sleeps. Calling that entry point from Godot's render
thread is unsafe.

Run the game on a dedicated native thread and keep Godot responsive. The host
boundary should own:

- startup configuration and data paths;
- the game thread and its required stack size;
- input exchange;
- frame and snapshot publication;
- orderly quit and a bounded shutdown timeout;
- error text that remains available after startup fails; and
- synchronization primitives with explicit ownership.

Where possible, replace process termination with a controlled unwind at the
outer game-thread boundary. If a timed destroy cannot stop the game, do not
silently free memory the detached thread might still access. Fail closed and
prevent a replacement session from racing the old one.

## Define a small versioned host contract

The first contract only needs configuration, input, framebuffer acquisition,
and lifetime control. Structured presentation can be appended later.

```c
typedef uint64_t legacy_session_t;

typedef struct legacy_config {
    uint32_t struct_size;
    uint32_t abi_version;
    uint32_t flags;
    char data_dir[260];
    char user_dir[260];
} legacy_config_t;

typedef struct legacy_input {
    uint32_t struct_size;
    uint32_t buttons;
    int16_t analog_dx;
    int16_t analog_dy;
    int16_t target_x;
    int16_t target_y;
    uint8_t use_target;
    uint8_t target_speed;
} legacy_input_t;

typedef struct legacy_frame {
    uint32_t struct_size;
    uint32_t publication_number;
    uint32_t gameplay_tick;
    uint8_t menu_present;
    uint8_t indexed_pixels[320 * 200];
    uint32_t palette_rgba[256];
} legacy_frame_t;

uint32_t legacy_host_abi_version(void);
int32_t legacy_session_create(const legacy_config_t *config,
                              uint32_t config_size,
                              legacy_session_t *out_session);
int32_t legacy_session_submit_input(legacy_session_t session,
                                    const legacy_input_t *input,
                                    uint32_t input_size);
int32_t legacy_session_acquire_frame(legacy_session_t session,
                                     legacy_frame_t *frame,
                                     uint32_t frame_size,
                                     uint32_t timeout_ms);
int32_t legacy_session_destroy(legacy_session_t session);
```

Use struct sizes and an ABI version from the start. The boundary will evolve
quickly once sprite records, layer maps, sound events, and editor metadata are
added. Reject a stale frontend or native library before either can interpret a
plausible but incorrect layout.

## Prove equivalence before changing presentation

The first milestone is not “it looks correct in the headset.” It is “the hosted
game behaves exactly like the original build.”

Create a deterministic baseline before suppressing any legacy drawing:

- record per-tick state hashes in the standalone game;
- run the same demo, replay, or scripted input through the hosted library;
- compare every tick, not only the final score;
- include framebuffer hashes while the original renderer is intact;
- measure input-to-frame latency at the native boundary; and
- verify create, quit, destroy, and restart behavior.

Keep a fast verification mode that disables rendering delays and audio while
preserving simulation order. A long deterministic run should be cheap enough
to execute after any change near the update loop.

If the game has no replay or demo system, capture abstract inputs and hash a
carefully selected state vector: RNG state, entity pools, player state, level
position, score, and mode flags. Avoid hashing pointer values, padding, or
presentation-only data.

## Put the framebuffer in VR first

Upload the indexed frame and palette to Godot and render it on a simple surface:
a tabletop, arcade cabinet, vertical screen, or tilted rhythm-game lane. This
early prototype answers questions that architecture diagrams cannot:

- Is the scale readable?
- Is the play posture comfortable?
- Does forward scrolling cause discomfort?
- Does the original tick rate look acceptable at headset refresh rate?
- Which input metaphor fits the game?
- Which parts truly need depth?

Preserve nearest-neighbor character without accepting headset shimmer. Pixel
art in VR often benefits from mipmaps, multisample antialiasing, render-scale
tuning, and a shader that keeps texels crisp while smoothing subpixel movement.

Treat the legacy surface as a product feature during development. It is the
fallback for unsupported effects, menus not yet rebuilt, unusual levels, and
diagnostics when the spatial path fails.

## Convert tracked motion into simulation intent

Do not move the player ship only in Godot. The simulation must still own its
position, collisions, firing origin, and replay input.

For a hand-steered playfield, map a tracked hand into a bounded control
rectangle and send an absolute target in game coordinates. Let the simulation
pursue that target during its own tick:

```text
tracked hand position
        |
        v
playfield-local rectangle
        |
        | clamp and normalize
        v
legacy game-space target
        |
        | fixed-step pursuit inside the simulation
        v
authoritative ship position
```

This is usually more stable than turning hand motion into host-frame velocity.
It does not overshoot when the headset and game run at different rates, and the
exact target consumed by each simulation tick can be recorded.

Retain digital or thumbstick input as an accessibility and debugging path.
Input should be expressed as abstract actions rather than raw OpenXR button
names so menus and automated harnesses can use the same contract.

## Record presentation semantics at legacy draw sites

Once the hosted framebuffer works, add a presentation recorder beside the
legacy renderer. Each relevant draw call continues to draw normally but also
emits a compact semantic record.

```c
typedef struct presentation_sprite {
    uint8_t category;
    uint8_t sheet_id;
    uint16_t cell_index;
    int16_t x;
    int16_t y;
    uint16_t source_id;
    uint16_t entity_type;
    uint8_t flags;
    uint8_t palette_effect;
} presentation_sprite_t;
```

Useful categories include ground enemies, airborne enemies, player, shots,
pickups, shadows, explosions, debris, and in-play text. A category is not just
a renderer convenience: it is the first durable statement about what a legacy
blit means.

Record facts already known at the draw site:

- source sheet and cell;
- simulation-space position;
- semantic category;
- stable source identity;
- palette filtering, blending, or darkening flags;
- animation or composition information; and
- entity type for authored presentation metadata.

Avoid re-identifying sprites later from framebuffer pixels. The draw call has
better information and can preserve unusual cases explicitly.

## Suppress by category, not all at once

After Godot can reproduce one category, suppress that category in the legacy
frame and let the spatial version show through. Keep everything else flat.

A practical sequence is:

1. player and common enemies;
2. player and enemy shots;
3. shadows, explosions, and debris;
4. scrolling background layers;
5. in-play text and HUD icons; and
6. menus and full-screen effects, if rebuilding them is worthwhile.

Suppression should be controlled by explicit host flags and recomputed each
tick. If a level enables an unsupported post-process or unusual rendering mode,
disable suppression for that tick and publish a `legacy_fallback` flag. Godot
must then hide spatial layers and show the complete framebuffer.

This gives every experimental renderer a clean rollback path and makes visual
differences mechanically inspectable: the suppressed framebuffer should contain
only the categories still assigned to it.

## Export background maps as data

Scrolling tile layers benefit more from real geometry than from flat overlay
planes. Export each layer's static map and tile atlas when a level loads, then
publish only its changing scroll and draw state each tick.

Separate:

- static map dimensions and tile indices;
- static indexed tile pixels and opacity;
- per-tick tile offset and sub-tile position;
- blend mode;
- draw-order or over-mode flags; and
- a level or sheet epoch telling the host to refresh caches.

Do not infer depth solely from layer number. Legacy engines frequently move a
layer before or after entities according to level data. Preserve those modes
until an authored spatial interpretation replaces them.

Add a standalone raster verifier: reconstruct each exported layer from the map,
atlas, and scroll record, hash it, and compare it with the layer drawn by the
native renderer. This catches off-by-one rows, wrap errors, stale maps, and
incorrect sub-tile offsets before a headset is involved.

## Reset temporal presentation state at boundaries

Interpolation buffers, apron ghosts, prior runs, pairing tables, and cached
sheet cells are all histories of one presentation timeline. They must not
survive a level, episode, asset-sheet, or game-mode boundary.

Export enough identity to recognize a discontinuity: a sheet or level epoch,
episode and mode IDs, plus a monotonic gameplay tick. Before rotating current
cells into the previous-frame buffer, clear every temporal cache when an epoch
or mode changes, or when the tick moves backwards. Clearing only the most
obvious ghost list is insufficient if it can immediately be reseeded from a
previous-mode cell buffer.

Test transitions, not just cold starts. Menu to story, arcade to story, level
restart, episode change, and return to title can each expose stale sprites that
never existed in the new simulation state. A raw tick-one record dump is a
useful discriminator: if the alleged object is absent from the native records,
the bug belongs to retained presentation history.

## Treat de-parallax as a gameplay migration

Old games often fake depth by moving background layers opposite the player's
horizontal motion. In spatial VR, real layer separation makes that fake motion
look like the world bending or shots curving.

Removing visual parallax is not always a presentation-only change. It may be
coupled to:

- enemy and pickup spawn windows;
- culling bounds;
- player travel limits;
- projectile motion or player-follow codes;
- collision coordinates; and
- scripted level events.

A staged approach is safer:

1. Export the original parallax deltas.
2. Rebase presentation records in Godot while leaving simulation untouched.
3. Document visual-versus-hitbox discrepancies this creates.
4. Add a native simulation mode that pins parallax and widens affected bounds.
5. Verify that mode independently with deterministic tests.

After an intentional simulation change, determinism and equivalence are
different claims. The modified game can remain perfectly deterministic while
no longer matching the upstream game's per-tick hashes. Version the new mode,
establish a new golden baseline, and state clearly which configurations are
expected to match upstream. Do not describe a migrated mode as tick-for-tick
equivalent merely because its own replays are repeatable.

If correcting a curved projectile only in the renderer would move it away from
its real hitbox, leave the visible curve until the simulation migration is
ready. Honest artifacts are safer than invisible collisions.

## Interpolate identities, not array slots

A 35 Hz game looks unpleasant when its world motion is displayed directly at
90 Hz. Publish structured state at the game rate and interpolate presentation
at headset refresh rate.

Give each record a stable identity. Pool indices are not sufficient because a
slot may be destroyed and reused within a few ticks. Use a generation counter,
spawn serial, or category-specific identity. Pair current and previous records
only when identity and semantics agree.

Some short-lived effects should not interpolate at all. Explosion slots, for
example, may recycle so quickly that pairing them creates long translucent
smears. A small amount of stepping can be less noticeable than incorrect
motion.

Menus need a separate publication rule. A mid-tick menu present must not expose
a half-written sprite-record buffer or replay one-shot sounds. Keep the last
complete gameplay snapshot, remove categories that should not float over the
menu, and mark the frame as a menu presentation immediately.

## Preserve indexed-art semantics

An indexed framebuffer contains more information than an RGBA screenshot. Keep
the palette pipeline explicit.

Export sprite pixels and opacity separately because palette index zero may be a
real opaque color. Reproduce legacy operations in shaders:

- palette lookup;
- hue/value filtering;
- color-key suppression;
- blend-table behavior;
- darkening and shadows; and
- palette animation or full-screen grading.

Pack sprites into atlases and batch them, but test exact slot boundaries. A
floating-point decode just below an integer can sample the previous atlas cell.
Round discrete values before division and modulo.

In multiview VR shaders, discrete per-instance data must not be smoothly
interpolated across a triangle. Mark flags, atlas indices, palette selectors,
and packed integers as flat varyings. Desktop flat mode can appear correct
while stereo multiview introduces tiny interpolation errors that flip a flag
bit or select the wrong cell.

Clamp composed-sprite UVs at their edges. Using `fract` to select a quadrant
can wrap a slightly negative multisample edge coordinate to the far side of the
texture, producing hairline artifacts visible only in the headset.

## Make depth authored and editable

Legacy draw order is evidence, not a complete spatial model. Start with broad
height bands, then let entity or asset metadata refine them.

Useful metadata includes:

- grounded, hovering, airborne, or foreground classification;
- base height and shadow plane;
- multi-cell assembly rules;
- whether an object rides the terrain or an elevated platform;
- projectile height overrides; and
- presentation exceptions for specific entity types.

Build a flat-screen height editor that runs the real game, lets a developer
select visible entities, adjusts metadata live, and saves a small external data
file. Editing while the encounter is running is faster and more reliable than
repeatedly rebuilding hard-coded tables.

Surface queries should use actual pixel occupancy when tile art is sparse. A
tile being present does not mean every pixel within that tile represents an
elevated platform. Give decals a small real geometric lift when coplanar depth
bias behaves differently under multiview; verify that the lift remains
imperceptible from normal play angles.

“Under platform” is an occlusion relationship, not a single global draw layer.
An enemy hanging from a rail may need its upper pixels hidden by the rail while
its lower body remains visible. Preserve the platform's real occupied shape and
compare sprite fragments against it, or split the sprite at the surface. A
single center-point height test will make partly occluded objects pop wholly in
front of or behind the platform.

## Put first-run VR onboarding before game startup

A legacy game may enter its own blocking menu loop as soon as the native
session starts. If players need recentering guidance, controller-ray selection,
or a safe controls tutorial, gate native startup until onboarding completes or
is skipped.

The first panel should be readable at the player's expected eye height even
when the initial origin is wrong. Explain the recenter action there, accept
laser input from either controller, and keep later tutorial panels clear of the
play area. Teach spatial controls through observable consequences: let the
real ship pursue the hand target at its actual game speed, require full-range
movement, and respawn a missed practice pickup until success. Persist completion
but retain a developer override for replaying the tutorial.

## Build diagnostics around exact frames

Headset-only bugs are common. Make reports reproducible without relying on
memory or prose.

Useful tools include:

- exact-frame capture in deterministic demos;
- a spectator viewport for XR captures, because the headset swapchain may not
  be readable from the main viewport;
- environment flags that force a ship, weapon, level, effect, or fallback;
- a suppressed-frame dump showing anything that leaked past category gates;
- trace lines for classification changes and slot reuse;
- top-down orthographic captures that map directly to legacy pixels; and
- an in-game checklist or metadata editor usable in the headset.

Save the frame number, gameplay tick, active modes, and relevant entity IDs
with every capture. “The third purple enemy looked transparent” becomes much
easier to solve when it means “demo frame 18420, entity type 73, source 19.”

Do not treat a non-XR editor run as proof of PCVR behavior. Exercise the exported
binary through a real OpenXR runtime and test mode transitions in the headset.
A recording captured by a standalone headset may still depict the streamed
PCVR application, so identify the executing build and runtime rather than
inferring the platform from the capture device.

## Build releases from one exact source state

Make desktop and standalone-headset packages identify the same clean commit.
Write the commit, version, dirty flag, build time, and signing identity into a
small build record; generate adjacent SHA-256 files; inspect the final ZIP and
APK rather than trusting staging directories; and verify the APK with the
platform signing tool. Use a persistent release key for sideloaded Android
builds so upgrades preserve package identity.

Documentation bundled inside an archive is part of the artifact. If install or
playtesting instructions change during release review, rebuild the affected
package instead of updating only the repository page. A private repository and
draft prerelease provide a useful final review surface. Publish the prerelease
before changing repository visibility so source and downloads can become
public together.

## A phased conversion plan

### Phase 1: host the untouched game

- Build the legacy core as a shared library.
- Run it on a dedicated thread with a headless video driver.
- Intercept the final present call.
- Exchange abstract input and the indexed framebuffer.
- Verify demos or replays tick for tick against the standalone build.

### Phase 2: prove the VR product shape

- Put the framebuffer on a comfortable spatial surface.
- Add OpenXR, recentering, flat fallback, and diagnostics.
- Test scale, tilt, posture, controls, and headset performance.
- Implement hand-target steering inside the simulation tick.

### Phase 3: add semantic snapshots

- Record sprites and one-shot sounds at legacy draw sites.
- Export stable identities, categories, and palette effects.
- Render one category spatially while preserving the full legacy fallback.
- Interpolate safe record types between simulation ticks.

### Phase 4: replace the world in layers

- Export tile maps, atlases, and per-tick scroll state.
- Reconstruct and hash layers in a standalone harness.
- Suppress verified legacy categories individually.
- Add authored depth metadata and a live editor.

### Phase 5: retire coupled 2D assumptions

- Migrate parallax-related simulation rules deliberately.
- Handle full-screen effects and exceptional levels.
- Rebuild only the menus and HUD that benefit from VR interaction.
- Keep the legacy path available for unsupported content and recovery.

## Common failure modes

| Failure | Typical cause | Prevention |
|---|---|---|
| Godot freezes in menus | Legacy game called on the render thread | Dedicated game thread and asynchronous host contract |
| Gameplay works but title screens do not | Integration began inside the gameplay loop | Intercept the common present boundary first |
| A pause menu shows floating gameplay fragments | Mid-tick present published a partial record buffer | Separate publication cursor, completed-tick snapshot, and menu flag |
| Hosted replay diverges | Input or timing applied at a different point | Per-tick hashes against the standalone path |
| Spatial layer scrolls one row off | Map wrap or sub-tile offset reconstructed incorrectly | Standalone layer raster hashes |
| Shots look straight but collide elsewhere | Visual de-parallax changed presentation only | Migrate coupled simulation rules or preserve the artifact |
| Old enemies appear on tick one of a new mode | Previous cells or ghosts survived a presentation epoch | Reset all temporal caches before rotating frame buffers |
| Effects smear across the screen | Short-lived pool slots paired as stable identities | Generations and category-specific interpolation policy |
| Sprite turns translucent only in-headset | Packed instance flags interpolated in multiview | Flat varyings and rounded discrete decoding |
| Hairlines appear on composed sprites | `fract` wrapped extrapolated edge UVs | Clamp quadrant UVs |
| Ground object ghosts over platforms | Coplanar or incorrectly classified depth | Pixel-aware surface query and small geometric lift |
| Hanging object is wholly above or below a rail | Under-platform treated as one global layer | Fragment- or occupancy-aware partial occlusion |
| Fallback frame becomes white or opaque | Suppression color is also affected by palette animation | Explicit opacity/key plane and transition tests |

## Safe-extension checklist

When spatializing another legacy draw category:

1. Identify the authoritative draw sites and all of their variants.
2. Record semantic data without suppressing the original pixels.
3. Add layout assertions and bump the ABI when the contract changes.
4. Verify records mechanically against a matching legacy frame.
5. Give records stable identities and decide whether they interpolate.
6. Reproduce palette, opacity, filtering, and composition behavior.
7. Render the category spatially with suppression disabled.
8. Enable category suppression and inspect the residual framebuffer.
9. Run the deterministic state-hash gate.
10. Reset temporal caches and test entry from every adjacent mode.
11. Test flat rendering, an exported PCVR build, and headset multiview.
12. Add a forced fallback for unsupported effects.
13. Capture exact frames for any remaining visual discrepancy.

The result is a sequence of working products rather than one long rewrite. The
original game remains playable at every phase; the framebuffer provides
coverage; semantic records provide structure; and Godot gradually turns a flat
legacy presentation into a spatial one without taking ownership of the game.
