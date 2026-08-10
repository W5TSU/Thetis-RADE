# CI build notes — gaps found and how they were closed

The repo as checked in cannot be built by a plain `msbuild Thetis_VS2026.sln`
on a stock GitHub-hosted runner. This documents what was broken and what the
`build.yml` workflow does about it, so the next person debugging a red build
knows where to look.

## Fixed, with confidence

- **`PlatformToolset=v145` everywhere.** Every `.vcxproj` in the tree pins an
  unreleased VS toolset ("v145" / informally "VS 18"/2026), unavailable on
  `windows-latest` (which ships VS2022 / v143). The workflow overrides this on
  every `msbuild` call with `/p:PlatformToolset=v143 /p:VisualStudioVersion=17.0`.
  This pattern is already proven working in the sibling
  `W5TSU/OpenHPSDR-Thetis-Hermes-Lite2` repo's own `build.yml`.

- **Missing `portaudio.vcxproj`.** `Thetis_VS2026.sln` references
  `Project Files/lib/portaudio-19.7.0/build/msvc/portaudio.vcxproj`, which
  doesn't exist in this repo — `build/` is git-ignored, so it's a locally
  generated file nobody committed. Vendored it into this branch from the
  `OpenHPSDR-Thetis-Hermes-Lite2` repo: same portaudio version, and its
  `ProjectGuid` (`{0A18A071-125E-442F-AFF7-A3F68ABECF99}`) matches exactly what
  `Thetis_VS2026.sln` already expects, and its `PA19.lib` output path matches
  what `ChannelMaster.vcxproj` expects (`Project Files/build/x64/Release/`).

- **`$(HPSDR_PLATFORM)` is unset.** `ChannelMaster.vcxproj` uses it (as a raw
  environment variable, not an MSBuild property) for several output/include
  paths, e.g. `../../lib/NR_Algorithms_$(HPSDR_PLATFORM)`. It's never defined
  anywhere in the repo — it must exist in the maintainer's shell. The workflow
  sets `HPSDR_PLATFORM: x64` at the job level.

- **Native libs ChannelMaster links against that aren't in the `.sln`.**
  `rnnoise`, `libebur128`, `WebRTC_AGC`, and `radae_c` all have committed
  `.vcxproj` files, but none are referenced by `Thetis_VS2026.sln`, so a plain
  solution build never builds them and ChannelMaster fails to link. The
  workflow builds each of them explicitly, in dependency order, before the
  main solution build.

## Best-effort / unverified — check here first if the build goes red

- **`opus_dnn`'s CMake flag.** There's no committed `.vcxproj` for it, only
  `Project Files/lib/opus_dnn/CMakeLists.txt` (a vendored xiph/opus checkout).
  Its DNN/LPCNet code only compiles in when `OPUS_DEEP_PLC`, `OPUS_DRED`, or
  `OPUS_OSCE` is set. The workflow guesses `-DOPUS_OSCE=ON` because the README
  specifically calls out a "FARGAN vocoder", which is OSCE's feature — but this
  wasn't verified against radae_c's actual symbol requirements. If
  `radae_c.vcxproj` or `ChannelMaster.vcxproj` fails on unresolved LPCNet/FARGAN
  symbols, try `-DOPUS_DRED=ON` and/or `-DOPUS_DEEP_PLC=ON` instead.

- **`radae_c`'s build script mentions a missing file.**
  `Project Files/lib/radae_c/msvc/build_radae_c.bat` says to run
  "after the vendored opus_dnn is built (`thetis_rade_opuslib.bat`)" — that
  script doesn't exist anywhere in this repo. It's possible opus_dnn needs
  more than a plain CMake build (a specific post-build step, model download,
  etc.) that this workflow doesn't replicate. `Project Files/lib/opus_dnn/dnn/download_model.bat`
  exists and may be relevant if radae_c expects a bundled LPCNet model.

- **x86 is not attempted.** The `.sln`'s own `ProjectConfigurationPlatforms`
  exclude `portaudio` from the `Release|x86` build (no `Build.0` line), which
  means even the maintainer's own solution config doesn't build a working x86
  audio backend. This workflow only targets `Release|x64`.

- **The WiX installer is intentionally skipped.** `Thetis-Installer.wixproj`
  has no `Build.0` entry for `Release|x64` in the `.sln` (only for
  `Release|Mixed Platforms`), so it won't build here and no WiX toolset was
  installed. Building the installer would need `Release|Mixed Platforms` and
  WiX Toolset v3.x on the runner.
