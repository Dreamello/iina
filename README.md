# IINA

Automated CI builds of upstream [IINA](https://github.com/iina/iina) with the
patch set in `patches/` applied.

This repository contains no IINA fork — each build clones upstream at the
requested ref (default: `develop`), applies every `patches/*.patch` in order,
and builds with Xcode on an arm64 GitHub Actions runner.

## Current patches

| Patch | Target | Purpose |
|---|---|---|
| `0001-xcodeproj-drop-x86-only-dylib-refs.patch` | `iina.xcodeproj/project.pbxproj` | build prerequisite: drop `libstdc++.6`/`libgcc_s.1.1` from the Copy Dylibs phase — x86-only GCC runtime libs that only exist in the universal dylib set, which break arm64-only builds |
| `0002-mpv-render-h-add-render-param-backend.patch` | `deps/include/mpv/render.h` | declare the custom `MPV_RENDER_PARAM_BACKEND = 21` render parameter |
| `0003-mpvcontroller-request-gpu-next-backend.patch` | `iina/MPVController.swift` | request the libmpv `gpu-next` render backend when creating the render context |
| `0004-screenshot-keep-jxl-on-gpu-next.patch` | `iina/MPVController.swift` | keep JPEG XL screenshots on the VO path (gpu-next handles HDR tone mapping; the `screenshot-sw` workaround for mpv #15107 causes color mismatch) |

The current set enables mpv's newer `gpu-next` (libplacebo-based) render
backend, which has substantially better HDR handling — in particular,
screenshots of HDR/HLG content keep correct tone/transfer mapping instead of
coming out washed-out SDR. libmpv upstream does not expose a backend selector
through the render API, so this takes a small patch on each side of the API:
the app side (patches here) and the libmpv side (a libmpv build implementing
the custom `MPV_RENDER_PARAM_BACKEND` parameter).

**A build from this repo only changes the app side.** Against the stock libmpv
dylibs it ships with, the extra render parameter is ignored harmlessly (you
keep the classic `gpu` backend). To actually get gpu-next, pair the app with a
patched libmpv by replacing the dylibs in `IINA.app/Contents/Frameworks`.

## Builds

The **Build** workflow runs:

- on every push to `main`
- every 30 minutes, building upstream `develop` — a preflight job skips
  (silently, in seconds) when a release for the current upstream commit
  already exists, so a real build only happens when upstream moves
- manually via workflow dispatch, with inputs:
  - `iina_ref` — upstream branch, tag, or SHA to build (default `develop`)
  - `publish_release` — publish the result as a release

After every successful publish the workflow notifies a downstream
repository via `repository_dispatch` (secret `IINA_AVS_TOKEN`).

Each build produces `iina-<version>-<sha>.tar.xz` (e.g.
`iina-1.4.3-a25ed13.tar.xz`, where `<version>` is the app's marketing version
and `<sha>` the upstream IINA commit built) as a workflow artifact; scheduled
and push builds also publish it to a GitHub release with the same tag.

Only the 10 newest releases are kept; older ones (and their tags) are pruned
automatically after each publish.

Builds are ad-hoc signed (not notarized): on first launch use right-click →
Open, or `xattr -dr com.apple.quarantine IINA.app`.

## References

The gpu-next patches are adapted from prior work in
[nilaoda/iina-avs](https://github.com/nilaoda/iina-avs).

## License

The patches are derivative of IINA and are licensed GPLv3, same as IINA.
