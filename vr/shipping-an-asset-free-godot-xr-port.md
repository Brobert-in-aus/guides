---
tags:
  - architecture
  - ci
  - game-porting
  - godot
  - openxr
  - release-engineering
  - vr
---

# Shipping an asset-free Godot XR port

A source-available port may be safe to publish while its original game assets,
maintainer-built binaries, or signing material are not. Treat those as separate
distribution surfaces instead of trying to solve them with one package.

This guide describes the personal-build model proven while preparing a Godot
.NET and native-library port for Quest and PCVR:

```text
public source repository
        |
        | user creates a real GitHub fork
        v
manual workflow in the user's fork
        |
        | produces an asset-free personal executable
        v
user's Windows PC -------------------- user-owned classic installation
        |                                      |
        +------------- local converter --------+
                          |
                          v
                 integrity-checked asset pack
                          |
              +-----------+-----------+
              |                       |
              v                       v
       Quest app inbox          PCVR user inbox
              |                       |
              +------ atomic import --+
```

The model does not determine whether a particular project may be distributed.
Repository-fork permission, software licences, trademarks, original game
assets, and linked binary distribution are separate questions. Record the
project's actual permissions and obtain legal advice where appropriate.

## Separate the four payloads

Keep these payloads independently auditable:

1. **Source**: code, build scripts, original documentation, and original
   asset-conversion tools that may be published.
2. **Build output**: an APK or PCVR archive assembled in the user's own fork.
3. **Game data**: art, audio, maps, and archives supplied from the user's own
   installation and processed only on their PC.
4. **Identity material**: Android keystores, passwords, certificates, workflow
   secrets, and locally generated manifests.

Do not rely on `.gitignore` as a distribution boundary. A developer workspace
can contain ignored assets that an export preset, generated manifest, or broad
copy step still packages. Test the clean source tree and the final archives.

## Use a real fork for a fork-based release

A renamed standalone copy is still a standalone repository. If the documented
flow requires GitHub's fork network, verify the public repository is registered
as a fork and has the expected parent. Renaming a real fork preserves that
relationship; pushing the same history to a newly created repository does not
create it.

Before launch, record:

- repository visibility;
- default branch;
- exact launch commit;
- fork status and parent;
- enabled workflows;
- tags and Releases that already exist; and
- whether inherited tag workflows have publication side effects.

This last check matters when a fork inherits a workflow that publishes to a
package registry on `v*` tags. A source-only launch may intentionally use a
normal commit without creating a tag or GitHub Release.

GitHub's documentation explains the platform behavior of
[fork permissions and visibility](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/working-with-forks/about-permissions-and-visibility-of-forks).
It is not a substitute for the upstream project's licence or permission terms.

## Make personal-build workflows manual and explicit

Put the personal workflows on the fork's default branch and use
`workflow_dispatch`. Require an explicit input before uploading artifacts:

```yaml
on:
  workflow_dispatch:
    inputs:
      publish_artifact:
        description: Upload this personal build
        required: true
        default: false
        type: boolean
```

The maintainer can run the same workflow with publication disabled to validate
the build contract without creating a downloadable binary. A user can enable
publication in their own fork and receive a short-lived personal artifact.

Pin and verify the toolchain used to create it:

- Godot editor and matching export templates;
- Godot OpenXR vendor plugin;
- .NET SDK;
- native compiler and target;
- Android template, SDK, Java, and signing tools; and
- any archive or patching utility used after export.

The workflow should start from a clean checkout, reject forbidden source
payloads, build, inspect the final package, write a machine-readable manifest,
and upload only the explicitly approved files.

## Gate the source and the final package

Use complementary allow and deny checks.

Source-tree gates should reject:

- original PAQ, PAK, WAD, ROM, or equivalent archives;
- locally generated asset packs;
- baked sprites, audio, and manifests under the Godot project;
- APKs and PCVR archives;
- keystores and secrets; and
- developer recordings or personal test data.

Final-package gates should require the runtime payload:

- executable and Godot PCK;
- managed assemblies and `.NET` runtime data;
- native simulation library for the intended architecture;
- OpenXR loader and required vendor libraries; and
- correct package ID, version, and manifest features.

They should independently reject game assets and raw archives. Inspect the APK
or desktop archive itself; a successful Godot export log is not evidence of the
payload that was ultimately signed or uploaded.

For Quest, also validate native-library architecture and Android page alignment.
For PCVR, verify that the executable remains beside its PCK, managed data,
OpenXR libraries, and `native/<runtime-id>` library.

## Convert user-owned assets locally

The converter should accept an explicit installation directory and also probe
well-known locations. Validate the edition before extraction: a classic bonus
build and a later remake may use different archives despite sharing a product
name.

Produce one versioned pack instead of requesting broad storage permission. A
useful contract is:

```text
asset-pack.json
sprites/sprite-manifest.json
sprites/...
audio/audio-manifest.json
audio/...
```

The root marker should contain:

- schema version;
- deterministic content ID;
- source-edition identity without private paths;
- SHA-256 for every payload; and
- the conversion-tool version.

Reject absolute paths, `..` traversal, duplicate normalized names, undeclared
files, missing files, unsupported schemas, and hash mismatches. Extract into a
staging directory, validate the complete result, then atomically replace the
working asset directory. A failed update must leave the previous installation
usable.

Keep the pack deterministic. Exclude editor cache sidecars and machine-local
metadata, normalize path ordering and timestamps, and compare independently
generated packs byte for byte in tests.

## Use app-owned inboxes

Do not make the runtime search arbitrary storage.

On Quest, copy the pack with ADB to the application's external-files inbox:

```text
/sdcard/Android/data/<package-id>/files/
```

On PCVR, use a per-user inbox under the engine's application-data directory.
At startup:

1. detect the inbox pack;
2. validate and stage it;
3. atomically install it into writable application data;
4. archive or remove the consumed inbox copy; and
5. initialize textures and audio only after import succeeds.

An asset-independent recovery panel is essential. Missing or corrupt assets
must produce readable instructions and a retry action without depending on the
assets that failed to load.

## Preserve Android signing identity

The first personal Quest build can generate a keystore, but the artifact must
include that key and a manifest that explains how to preserve it. Later builds
must restore the same key from a private Actions secret.

A different signer cannot update the installed APK. The user must uninstall
the old package first, which normally removes imported assets, settings,
scores, and replays. Treat the keystore as application identity, not as a
disposable build by-product.

Test two hosted runs:

1. no secret: generate a key and artifact;
2. saved secret: rebuild and prove the signer certificate is identical.

Compare certificate fingerprints, not merely keystore filenames or passwords.

## Make fresh-install testing genuinely fresh

An APK reinstall is not a first run if the existing package data remains.
For a destructive rehearsal, resolve and display the exact device, package,
APK, asset pack, sizes, and hashes before execution. Then:

1. force-stop the exact package;
2. uninstall only that package;
3. install the candidate without launching it;
4. stage the asset pack;
5. verify package and remote-pack hashes;
6. confirm Android reports a new install and a not-yet-launched state; and
7. leave the application stopped.

Launch the first run from the headset library. An ADB launch can race or bypass
Quest controller, permission, and system interstitials, making onboarding look
as though it was skipped.

For ordinary updates, keep the signing key and use an in-place install so the
application can reuse imported assets and user data.

## Test the workflow as a product

Workflow YAML passing lint is not enough. Rehearse the complete user journey in
an isolated temporary repository:

- create the repository with the intended default branch;
- push a clean source copy;
- enable and run the manual workflow;
- wait for completion and download the artifact;
- re-run local package gates against the download;
- preserve the first signing key as a secret;
- run again and compare signer certificates; and
- delete the temporary repository when evidence has been retained elsewhere.

Store a small evidence file with commit, workflow IDs, artifact names, hashes,
certificate fingerprints, tool versions, and pass/fail results. Do not make a
temporary repository or expiring artifact the only copy of release evidence.

## Keep Quest and PCVR instructions separate

The shared policy can be one paragraph, but the installation procedures differ.

Quest documentation should cover:

- personal APK workflow and artifact expiry;
- keystore retention;
- Developer Mode and ADB authorization;
- asset-pack transfer;
- fresh install versus update; and
- first launch from the headset.

PCVR documentation should cover:

- Windows ZIP versus Linux tar archive;
- keeping the complete extracted package together;
- local per-user asset inbox;
- selecting and starting an OpenXR runtime; and
- which OS/runtime/headset combinations were physically tested.

Do not infer runtime support from successful packaging. Record build-only
targets separately from physically tested targets.

## Release checklist

- [ ] Public repository is the intended real fork, with the expected parent.
- [ ] Default branch points to the reviewed commit and the worktree is clean.
- [ ] No forbidden assets, binaries, recordings, packs, or keys are tracked.
- [ ] Manual workflows default to no artifact publication.
- [ ] Toolchains and downloaded dependencies are pinned and hash-verified.
- [ ] Source and final-package gates both pass.
- [ ] Generated-key and restored-key Quest runs use the same signer.
- [ ] PCVR archives contain every adjacent runtime component.
- [ ] Local asset packs are deterministic, integrity-checked, and atomic.
- [ ] Fresh Quest install is left stopped for a headset-initiated first launch.
- [ ] Quest, PCVR, multiplayer, and first-run acceptance evidence names the
      exact commit and package hashes.
- [ ] Existing tag-triggered workflows were reviewed before creating any tag.
- [ ] Temporary test repositories and expiring artifacts are cleaned up after
      durable evidence is saved.
