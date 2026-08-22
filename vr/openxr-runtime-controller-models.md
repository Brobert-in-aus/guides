---
tags:
  - godot
  - meta-quest
  - openxr
  - input
  - vr
---

# Loading and animating OpenXR controller models in Godot

Accurate controller rendering has three separate requirements:

1. load the model supplied for the active hardware;
2. attach it to the correct tracked pose; and
3. animate its controls from the application's OpenXR actions.

Completing only the first requirement produces a model that follows the hand
but whose trigger, grip, buttons, and thumbstick remain static.

## Support both render-model paths

Godot's `OpenXRRenderModelManager` uses
`XR_EXT_interaction_render_model`, the portable path shown in the
[official OpenXR render-model demo](https://github.com/godotengine/godot-demo-projects/tree/master/xr/openxr_render_models).

Quest may instead expose its high-quality Touch model through
`XR_FB_render_model`, provided by the Godot OpenXR Vendors plugin as the
dynamically registered `OpenXRFbRenderModel` node. On affected runtimes, using
only the portable manager yields no runtime model even though render-model
permission is available.

Use a selection order such as:

1. promoted `OpenXRRenderModelManager` result;
2. Meta FB runtime model with a local animator; and
3. an application-provided animated fallback.

Keep the fallback visible while runtime loading is in progress. Switch only
after the chosen model and its animator have initialized, and never show two
representations for one controller at the same time.

## Export the required Quest capability

Enable both project extensions when Meta Quest is supported:

```ini
[xr]

openxr/enabled=true
openxr/extensions/render_model=true
openxr/extensions/meta/render_model=true
```

Inspect the final APK manifest for Meta's render-model permission and optional
feature:

```xml
<uses-permission android:name="com.oculus.permission.RENDER_MODEL" />
<uses-feature android:name="com.oculus.feature.RENDER_MODEL"
              android:required="false" />
```

Keep the feature optional when the application has a fallback. Validate the
final APK rather than relying only on project settings.

## Parent the model to the grip pose

Create one `XRController3D` for each hand and track its grip pose. The model
provider and presentation fallbacks belong below that controller:

```text
XRController3D (left_hand or right_hand, pose=grip)
└── ControllerModelContainer
    ├── OpenXRRenderModelManager
    ├── OpenXRFbRenderModel
    ├── RuntimeModelAnimator
    └── ProceduralFallback
```

For `OpenXRRenderModelManager`, select the matching hand tracker and make the
model local to the `grip` pose. Applying a guessed translation or 90-degree
correction can appear to fix one model while displacing another. Let the
runtime and pose relationship define the transform.

In C#, the vendor node is a GDExtension class and may not have a compile-time
managed type. Instantiate it dynamically:

```csharp
if (ClassDB.ClassExists("OpenXRFbRenderModel") &&
    ClassDB.Instantiate("OpenXRFbRenderModel").AsGodotObject() is Node3D model)
{
    model.Set("render_model_type", isLeft ? 0 : 1);
    container.AddChild(model);
}
```

Add it after the OpenXR session has started. Loading is asynchronous; poll the
provider or connect its loaded signal before looking for the generated scene.

## Treat the runtime animation as calibration

A runtime glTF may contain a skeleton and a reference `AnimationPlayer`, but
the reference clip is not necessarily wired to live input. Playing the clip
only demonstrates every control in sequence.

Instead:

1. find the generated `Skeleton3D` and `AnimationPlayer` recursively;
2. stop and disable reference playback;
3. find tracks for trigger, grip, thumbstick, face buttons, and menu button;
4. store key zero as the neutral pose;
5. choose the key farthest from neutral as the fully actuated pose; and
6. blend from neutral to actuated using live action values every frame.

This extracts model-specific endpoints without hardcoding trigger angles or
button travel. Use analog values for trigger and grip, and boolean action state
for button depression.

The action map must expose every value the animator consumes. Typical Godot
reads are:

| Action | Read | Presentation |
|---|---|---|
| `trigger` | `GetFloat` | index trigger |
| `grip` | `GetFloat` | grip trigger |
| `primary` | `GetVector2` | thumbstick X/Y |
| `primary_click` | button state | stick click |
| `ax_button` | button state | A or X |
| `by_button` | button state | B or Y |
| menu/recenter | button state | system/menu control where exposed |

Use the same actions that drive gameplay. A second model-only input mapping is
easy to invert or leave incomplete.

## Calibrate thumbstick axes per hand

Some runtime clips provide one combined stick sweep instead of independent X
and Y tracks. In that case, derive the neutral stick pose and apply modest local
rotations for live X and Y input.

Do not assume the same signs for both skeletons. A mirrored right-hand skeleton
may require both axes to change sign. Also verify which local axes create
visible tilt: rotating around the shaft produces twist rather than directional
deflection.

For example:

```csharp
float handSign = isLeft ? -1.0f : 1.0f;
Quaternion vertical = new(
    Vector3.Up,
    Mathf.DegToRad(12.0f * handSign * stick.Y));
Quaternion horizontal = new(
    Vector3.Right,
    Mathf.DegToRad(12.0f * handSign * stick.X));

skeleton.SetBonePoseRotation(
    thumbBone,
    neutralThumbRotation * vertical * horizontal);
```

This is a starting point, not a universal calibration. Test left, right, up,
and down independently on both physical controllers. A correct-looking idle
model and a working trigger do not validate stick axes.

## Switch to hands only when hand tracking is active

Controller models and optical hand models should represent the active tracking
source, not be cosmetic choices that can contradict input. When optical hand
tracking becomes valid:

- hide the controller representation;
- show the tracked hand skeleton;
- apply every reported joint pose one-to-one in the correct tracking space; and
- restore controllers promptly when hand tracking is lost.

Use Godot's
[OpenXR hand-tracking demo](https://github.com/godotengine/godot-demo-projects/tree/master/xr/openxr_hand_tracking_demo)
as the reference hierarchy and joint path. A static glove mesh under an
`XRController3D` is not hand tracking.

## Validate physically

Initialization logs prove model selection, not visual correctness. On every
supported runtime and controller pair, check:

- full six-degree grip pose;
- trigger and grip over their analog ranges;
- A/B/X/Y and thumbstick-click depression;
- thumbstick left, right, up, and down on both hands;
- correct visibility during controller-to-hand transitions;
- recovery after focus loss, controller reconnect, and tracking loss; and
- fallback visibility when no runtime model is available.

Keep a procedural fallback even after Quest succeeds. PCVR runtimes differ in
which render-model extensions and model structures they expose.
