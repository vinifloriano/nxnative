# Factorio (`01004200189F4000`)

Update 2.0.77 (v2097152), SDK 21.4.2. Wube's own engine on SDL. It renders with **OpenGL 3.3 core** through EGL, not NVN: the host's GL layer implements it on Metal (`dm/gfx/gl/hle_gl.m`). Modules `main` and `sdk`; `subsdk0` (NVIDIA's OpenGL driver) is never loaded. `main` also imports Vulkan and the RomFS has `.metal` shaders, which this build does not use.

On the Mac (dev build and Mac app, 2026-10-29): the changelog shown at first start, the main menu with its live background map, the new-game screen and the tutorial's first level, with the game's sprite atlases compressed on the GPU, at 60 fps; `Factorio initialised` 9 to 14 s after start. A controller was connected during these runs. Gameplay beyond the tutorial's start is not tested; the iPhone is not tested.

## What it needs

| Setting | Where | Why |
|---|---|---|
| `modules main sdk` | [`game.tsv`](game.tsv) | `subsdk0` is NVIDIA's OpenGL driver, which programs the GPU through NvRm: never loaded, the GL layer answers `main`'s EGL imports |
| `sys_resource 0x0`; `mem_reserve 0x100000000`, `mem_alias 0x10000000`, `mem_stack 0x10000000`, `app_pool 0xCD500000` | `game.tsv` | no system resource, so no virtual address memory; `main` sets a heap of at most the pool − 64 MB (a 3328 MB heap region). The game logs `RAM: 3284/3285 MB` |
| No NVN shader tables | `game.tsv` `shader_note` | no NVN shaders: `gl_shaders.py` writes the GL library and its table from the RomFS, not the profile |

Nothing is cut in `main` or `sdk`.

## Start-up

- **Resolved: abort 0xe15 at `sdk+0xcfdd4`** with `[RED] sf: the game connects to service fsp-srv itself`: SDK 21's `fs::QueryMountRomCacheSize(size_t*, int)` / `MountRom(name, int, …)` (rom of program index 0), `MountRamDisk` (`ramdisk:`, 32 MB asked) and `EnableDefaultGlobalFileDataCache` had no hooks ([Rom of a program index, RAM disk and the default file data cache](../../../docs/reference/services.md#rom-of-a-program-index-ram-disk-and-the-default-file-data-cache)).
- **Resolved: `std::terminate` at the first `steady_clock::now()`** (`main+0x1c412e8`): SDK 21's monotonic clock reads `time::GetStandardSteadyClockCurrentTimespec` ([`clock_gettime(CLOCK_MONOTONIC)` through the steady clock](../../../docs/reference/services.md#clock_gettimeclock_monotonic-through-the-steady-clock)).
- The game writes its own log to the host log (`[dm] [game]` lines): atlas sizes, `Sprites loaded`, `Factorio initialised`, the menu level's `level.dat`.

## Rendering (OpenGL)

- **Shaders:** the RomFS's `data/core/graphics/shaders/*.vert` / `*.frag` (34 + 68 files, 97 distinct sources, `#version 330`, written by SPIRV-Cross from HLSL) are what the game compiles, verbatim; `gl_shaders.py` translates them from the RomFS, so no capture list is needed (every shader two capture runs collected, 51, is in the library). Every constant is in a named `std140` block (75 blocks).
- **Resolved: most draws skipped** (`uniform block … is not the default block`): named uniform blocks, sampler objects, buffer textures (`samplerBuffer` in the tile and sprite vertex shaders) and `usampler2D` ([GL uniform blocks, sampler objects and buffer textures](../../../docs/reference/rendering.md#gl-uniform-blocks-sampler-objects-and-buffer-textures)).
- **Resolved: a second draw buffer and integer targets**: sprites write `lightColor` to colour attachment 1; the GPU texture compressor renders into `RG32UI` / `RGBA32UI` ([GL draw buffers and integer render targets](../../../docs/reference/rendering.md#gl-draw-buffers-and-integer-render-targets)).
- **Resolved: a jump to address 0 at `main+0x9ab528`** (`glCopyImageSubData`, loaded only with `GL_ARB_copy_image`) and the GPU compressor's self-test failing (`Test FAILED. … BC3 PSNR: 7.97`, so every atlas was compressed on the CPU, 11.3 s to `Sprites loaded`): [GL texture copies, readback and timer queries](../../../docs/reference/rendering.md#gl-texture-copies-readback-and-timer-queries). The test now passes (PSNR 35.8 / 33.8), 8.9 s.
- The game sets `GL_TEXTURE_LOD_BIAS` −0.484375 on its atlases, which Metal samplers do not have (`[RED]` once): open, not visible in the menus.
- About 140 draws and 14 passes a frame in the main menu, 60 to 80 draws in the tutorial; 26 pipelines.

## Found on this game

General host changes first needed here; each applies to every game:

- **SDK 21 file system calls** (rom of a program index, RAM disk, default file data cache): [services.md](../../../docs/reference/services.md#rom-of-a-program-index-ram-disk-and-the-default-file-data-cache).
- **SDK 21's monotonic clock through the steady clock**: [services.md](../../../docs/reference/services.md#clock_gettimeclock_monotonic-through-the-steady-clock).
- **The GL layer's uniform blocks, sampler objects, buffer textures, draw buffers, integer formats, image copies, readback with BC decoding and timer queries** (`hle_gl.m`), and **named blocks and texture buffers in `gl_shaders.py`**: [rendering.md](../../../docs/reference/rendering.md#gl-uniform-blocks-sampler-objects-and-buffer-textures) and the two entries after it.

## Open

- Gameplay past the tutorial's first minutes, saving and loading, and the iPhone build are not tested.
- `GL_TEXTURE_LOD_BIAS` is not applied (Metal samplers have no bias).
