# ScatlettVIctory (work in progress)

[ScatlettVIctory](https://github.com/cr0nkers/ScatlettVIctory) is a
work-in-progress fork of
[fxceless/victory-fork](https://github.com/fxceless/victory-fork), built on
[VIctory](https://github.com/Marvisak/VIctory), a compatibility layer that runs
an Xbox Series title natively on Windows from files the user supplies.
The goal is to get the game's world rendering correctly and playably.

The executable and Cargo package are named `victory`. Configuration uses
`victory.toml` and the `VICTORY_*` environment variables shown below.

**This repository contains no game files, code or assets.** You need your own
copy of the game. It is not affiliated with or endorsed by the game's
developer or publisher.

## Current progress (2026-11-25)

**Not playable yet.** The opening cutscene and first gameplay scene render,
but world performance is around 2–4 fps and shadowed areas are black.

Gameplay results below are inherited from the source fork's report dated
2026-11-25; they have not been revalidated in ScatlettVIctory. That report
used Windows 11, an Intel i7-14700KF, an NVIDIA RTX 4070 12 GB, 32 GB RAM,
and the Series X profile.

| Area | Reported status |
|------|-----------------|
| Boot and menus | Start screen and menus run at 30 fps |
| Opening cutscene | Plays through in order, with lighting |
| First gameplay scene | Characters, textures, sky and sunlit exterior render; world rendering remains incomplete |
| Input and interface | HUD, pause menu and controller input work |
| Saves | Autosaves a checkpoint and loads it again |
| Lighting | Direct sunlight renders; ray-traced bounce light and lamp/sun shadows do not run end to end |
| Ray tracing | 49/52 builder and 59/64 trace programs translate; experimental and off by default |
| Stability | No driver hang or device loss reported in normal runs |

### Main blockers

- **Lighting:** areas without direct sunlight remain black. See the
  [ray-tracing progress and known gaps](docs/fork-ray-tracing.md).
- **Performance:** reported default frame times are 420–450 ms, with around
  24 GB of memory in use during play.
- **Rendering coverage:** the first gameplay room can be missing after the
  cutscene, and some shaders still fail to translate.
- **Hardware coverage:** the source fork's gameplay results cover one PC.
  Some implementations approximate console behavior and are marked
  `shortcut:` in the code.

### Development progress

The inherited rendering work includes fixes for meshlet culling, descriptor
exhaustion, lighting-buffer copies, texture sampling and several shader
translation failures. Memory and submission work includes GPU buffer copies,
shader optimization, asynchronous submissions and pipeline-cache fixes.

Most newer performance paths are experimental and off by default. The source
fork reported 240–350 ms per frame with either `VICTORY_RELAY=2` or the
combination `VICTORY_LATE_WAIT=3 VICTORY_WIDE_BACK=1
VICTORY_CARD_BUFFERS_MIN_MB=1 VICTORY_CARD_BUFFERS_MB=1024`.
`VICTORY_INPLACE` is also experimental. The wave64 barrier form is off unless
`VICTORY_WAVE_WIDE=1`, following a reported GPU hang. Each switch is described
where it is read in the code.

Recent ScatlettVIctory changes:

- Persistent data uses `XDG_DATA_HOME` or `~/.local/share` when
  `LOCALAPPDATA` is unavailable, keeping it out of the default cache directory.
- `translate --jobs 0` is rejected instead of silently skipping all shaders.
- `census`, `shader-info` and `translate` report directory traversal errors
  instead of silently producing incomplete reports.

- **Save integrity:** the newest save version's blobs are checked before older backups are pruned.
  Missing or unreadable blobs now report errors instead of being counted as zero
  bytes or silently omitted from quota and container reports.

For the implementation layout, see [Architecture](docs/architecture.md).

<details>
<summary>Historical upstream status (2026-11-23)</summary>

This is VIctory's earlier report, retained for reference. The current fork
progress is summarized above.

**Not playable yet.** The title boots, runs its menus at its own 30 fps and
loads the world, where rendering fails.

| Stage | State |
|-------|-------|
| Boot, first-boot screens (accessibility, language, settings, legal) | works, 30 fps |
| Intro video with sound | works |
| Start screen, profile and saves | works |
| Loading into the world (about 50 s) | works |
| The world (first mission's intro) | reached and runs with indirect draws; its title card renders, the scene after it is still dark; 0.5-1.8 s per frame |

Tested on one machine: Windows 11, AMD RX 6700 XT.

> **Warning.** Entering the world on a real GPU can currently hang it, and
> Windows then resets the display driver (every monitor goes black for a
> moment). To look further, use a diagnostics build with
> `VICTORY_D3D12_WARP=1`, which renders on the software adapter.

What each part does today and what it still needs:

| Subsystem | Today | Needs |
|-----------|-------|-------|
| Loader | Runs the title's executable and its DLLs in our process; crash reports name the faulting code and are kept in `crashes.log` | – |
| Runtime services | One signed-in offline user, licences, packages, saves that survive a kill at any step, text entry dialog | Speech is silent; the dialog and relaunch are untested live |
| Network | Offline by design: the title cannot reach the network | – |
| Files and streaming | Archives and streamed data read, zlib and BCPack decoded in parallel | Loading is limited by writes into title memory |
| Input | XInput pads, keyboard, keyboard stand-in pad | Trigger rumble |
| Audio | Menu, video and loading sound; XMA voices and Opus streams decode | Reverb is silent; Opus results unverified against the title |
| GPU: menus | Draws, compute and presentation on host D3D12; only changed title memory is copied | Frame pacing drifts on one screen in release builds |
| GPU: world | Mesh pipelines, shader-written buffers, descriptor lifetimes, indirect execution (arguments clamped), pipelines made by worker threads and kept on disk | Most of the scene is still missing (untranslated shaders drop draws); frames take seconds; real queries, depth bounds, HDR and scaling are missing |
| Shaders | Console GPU code recompiled to DXIL at first use, cached on disk. Menus: all. World: mesh 8,191 of 8,506, vertex/pixel 10,617 of 12,346, compute 1,643 of 2,539 | 64-bit image atomics, merged geometry stages, the ray-tracing builder's shaders; some compute programs need a GPU with 64-lane waves |
| Textures | Console tiled layouts and detiling for the modes the title uses | Render-target metadata planes |
| Series S mode | Runs through the menus to the loading screen | The world, as above |

</details>

## Getting started

Requirements: Windows 10/11 x64, a CPU with AVX2, a Direct3D 12 GPU with
shader model 6.6 (and mesh shaders for the world), Rust (pinned in
`rust-toolchain.toml`).

The title also needs several GiB of available Windows commit at startup.
Check **Committed** under **Task Manager → Performance → Memory**. If
`crashes.log` reports Windows error 1455, close other applications or increase
the Windows paging file size, then retry.

```powershell
git clone https://github.com/cr0nkers/ScatlettVIctory.git
cd ScatlettVIctory
cargo xtask install-hooks                # asset-policy check on every commit
copy victory.example.toml victory.toml   # then set your paths (outside the repo)
cargo test --workspace                   # needs no game data
cargo run -p victory -- config           # check your paths
```

Run the title:

```powershell
cargo build --release -p victory
target\release\victory.exe run <game dir>\<the title's exe> --execute
```

The window caption shows the framerate. Saves, settings and `crashes.log`
are kept under the data folder (default `%LOCALAPPDATA%\victory\data`), derived
data under the cache folder.

Input: XInput pads. With none connected the keyboard is a pad while the
title's window has focus:

| Keys | Pad |
|------|-----|
| W A S D / I J K L | left / right stick |
| arrows | D-pad |
| Enter, Backspace, X, Y | A, B, X, Y |
| Q / E, Z / C | shoulders, triggers |
| F / H | stick clicks |
| Esc, Tab | Menu, View |

## Commands

```
victory inspect <exe|dll> [--imports] [--exports]
victory census [paths...] [--missing] [--strict]
victory lib-exports <import.lib> [--list]
victory shader-info [paths...] [--verbose] [--disasm]
victory translate [paths...] [--validate] [--pipelines] [--list] [--jobs N]
victory run <exe> [--execute] [--unimplemented trap|zero] [--strict]
            [--console series-x|series-s] [-- <title arguments>]
victory config
```

Logging: `VICTORY_LOG=victory_loader=trace,info` (a release build logs
info and above).

## Debugging

A release build has no debugging aids. Build them in with

```powershell
cargo build --profile diag -p victory --features diagnostics   # target\diag\victory.exe
```

| Variable | Effect |
|----------|--------|
| `VICTORY_LOG=<target>=debug,...` | debug and trace output |
| `VICTORY_STACKS=20,60` | every thread's stack at those seconds |
| `VICTORY_WATCH=<rva>,...` | logs a title u32 when it changes |
| `VICTORY_KEYS=30:Enter,35:X` | presses keys at those seconds, focus or not |
| `VICTORY_CRASH_SAVE=<rva>:<len>,...` | saves title memory to the cache folder on a crash |
| `VICTORY_CAPTURE=<dir>` | saves presented frames (powers of two, or `VICTORY_CAPTURE_FRAMES=<list>`) |
| `VICTORY_DUMP_TARGETS=1` | with the two above, saves every render target of those frames |
| `VICTORY_D3D12_DEBUG=1` | host debug layer |
| `VICTORY_D3D12_GBV=1` | with the above, GPU-based validation (bad descriptor use reported, not faulted; slow) |
| `VICTORY_D3D12_DRED=1` | on a GPU failure, the allocation hit and the commands in flight |
| `VICTORY_PIPELINE_WAIT_MS=24` | reduce the per-frame wait for background pipeline creation from the default of 500 ms. A draw can be omitted until its pipeline is ready |
| `VICTORY_SKIP_PIPELINES=<hash>,...` | omit draws and dispatches, including indirect ones, whose pipeline contains a listed 16-digit shader hash |
| `VICTORY_GPU_TIMING=1` | with the above, each list of a submission run and timed on its own; slow lists logged with their slowest commands |
| `VICTORY_D3D12_WARP=1` | host device on the software adapter (slow; cannot reset the driver) |
| `VICTORY_SHADER_LOOP_LIMIT=<n>` | translated shader loops stop after `n` iterations per lane instead of 2^20 (own shader cache entries); if a lower `n` makes world frames much faster, loops hitting the limit are what makes them slow |
| `VICTORY_AUDIO_CAPTURE=<file.wav>` | records what the title plays (our process only) |
| `VICTORY_XMA_DUMP=<dir>` | saves the first 32 XMA voices' context and input |

Every build appends crashes and host GPU failures to `crashes.log` in the
title's data folder.

## Credits

- **[VIctory](https://github.com/Marvisak/VIctory)** by Marvisak and its
  contributors is the base of everything here: the loader, the runtime
  services, the Direct3D 12 back end and the shader recompiler. Licensed
  GPL-3.0-or-later; so is ScatlettVIctory. Upstream's history is preserved.
- **[fxceless/victory-fork](https://github.com/fxceless/victory-fork)** is the
  source fork for ScatlettVIctory's inherited rendering and performance work.
- **[XCL](https://github.com/ivall/xcl)** by ivall and its contributors
  (GPL-2.0-only) was used as a reference for how the same title behaves under
  another layer. Fixes inspired by it were written anew in VIctory's code.
- The source fork's changes were developed with [Claude Code](https://claude.com/claude-code).

## License

GPL-3.0-or-later. See [LICENSE](LICENSE).
