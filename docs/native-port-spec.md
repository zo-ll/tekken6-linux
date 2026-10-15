# Tekken 6 Native Online — Linux Port (native rebuild)

> **Abandoned on 2026-10-15.** The goal is to play against Windows players. A native rebuild cannot
> do that: the lobby pairs players by the SHA-256 of `tekken6.exe`, the netplay code is closed, and
> `rexruntime.dll` is a private fork of ReXGlue (T6-specific code), not stock v0.10.0. The project now
> runs the unchanged Windows release under Proton. See the README. This file stays as research.

**Project spec. Status: draft 1. Owner: az.**

---

## 1. Purpose

Run the Windows release "Tekken 6 Native Online (Xbox 360)" natively on Linux.

"Native" means: a Linux ELF binary, no Wine, no Proton, no Windows compatibility layer.
The Linux binary uses the Vulkan GPU backend of ReXGlue.

The upstream release ships no source for its game-specific code. This project rebuilds the
missing code against the open-source ReXGlue SDK, then reuses the code generator, the code
generation config, and the patch tools from the release package.

---

## 2. Goals

| ID | Goal | Priority |
|----|------|----------|
| G1 | Build the ReXGlue SDK for Linux (Vulkan backend) | must |
| G2 | Translate `Default.xex` with the shipped codegen config on Linux | must |
| G3 | Build a Linux `tekken6` binary that boots the game and starts a match | must |
| G4 | Play offline modes: VS, Arcade, Practice, Ghost | must |
| G5 | Rebuild the sound boundary and the guest state snapshot on Linux | should |
| G6 | Rollback netplay over GGPO, direct peer-to-peer (no official lobby) | should |
| G7 | Input mapping and a small launcher for Linux | could |
| G8 | Keep the release package as a build input (no source copy) | must |

## 3. Non-goals

- **No account system and no official lobby.** The official lobby is a closed server.
  The game binary itself does no matchmaking. Use direct peer-to-peer GGPO.
- **No cross-play with the official Windows lobby.** The official launcher pairs players by
  a hash of `tekken6.exe`. A Linux build has a different hash. See risk R6.
- **No game data in the repo.** The user must own Tekken 6 (USA, Europe) Rev 1 for Xbox 360.
- **No redistribution of derived assets.** `tools/shader_cache/4E4D07FC.xsh` and
  `tools/menu_vs.bin` derive from the game data. Keep them out of the repo.
- **No emulator.** Do not add Xenia, RPCS3, or a JIT. This project is a static recompilation.

---

## 4. What the Windows release is (verified)

Download: release `v2.1.5`, asset `tekken6_native_online.zip`.
Size 196 860 598 bytes. SHA-256
`00a22f4cc4ea3d8948480264bf7acea44bdd062426bf9f5b84b748a0c05d269b`.
7753 files, 690 MB unpacked. The Git repo itself holds only `README.md`.

### 4.1 Pipeline

`builder/build_native.py` runs six steps:

1. Check the copy. `Default.xex` SHA-256 must be
   `ee138a4fe8d32a0edc131451377f4377041a49626153fa992db24f5420a7f15c`
   ("Tekken 6 (USA, Europe) Rev 1").
2. Copy the game data to `native/game`. ISO input is extracted with `builder/bin/xdvdfs.exe`.
3. Translate the code. `builder/bin/rexglue.exe -f codegen native/work/tekken6_manifest.toml`.
   Output: `native/work/generated/default/tekken6_recomp.*.cpp`, `tekken6_pch.h`, `tekken6_init.cpp`,
   `tekken6_register.cpp`.
4. Apply four patch tools in order: `pagemap_patch`, `rawstore_patch`, `sndcall_patch`,
   `faststore_patch`. They edit the generated C++.
5. Compile with the bundled llvm-mingw clang 23 (`builder/llvm-mingw/bin/clang++.exe`).
6. Link `native/tekken6.exe` from the object files plus the prebuilt libraries.

### 4.2 Code generation inputs

- `builder/project/tekken6_config.toml` — codegen hints: `skip_*` flags, 20 forced function
  starts, 15 vtable thunks, 11 switch tables, 1 invalid instruction pattern, 4 `[[midasm_hook]]`
  entries.
- `builder/compile.json` — the compiler flags. See section 7.1.
- `builder/project/tools/*.py` — 324 lines of pure Python. Platform independent.

### 4.3 The midasm hooks (from `tekken6_config.toml`)

| Address | Name | Registers | Purpose (from the comments) |
|---------|------|-----------|-----------------------------|
| `0x82219900` | `T6FrameHook` | `r3` | Frame boundary. Head of the main loop `sub_822198D8`. Snapshot save/restore point. |
| `0x82268A0C` | `T6InputHook` | `r1` | Input point. After both `0x4C` input entries are built, before `sub_823FAA08`. |
| `0x822658F0` | `T6SeedHook` | `r3` | Seed of the game LCG at `0x828B49E0`. |
| `0x826A0D78` | `T6CrtSeedHook` | `r3` | CRT `srand`, per-thread rand state at `_getpdt()+20`. |

### 4.4 The prebuilt library — the porting blocker

`builder/lib/libt6prebuilt.a` is a Windows/COFF archive. `ar t` lists 37 members:

- SDK code, compiled for Windows and D3D12: `d3d12_*.cpp.obj`, `surface_win.cpp.obj`,
  `window_sdl.cpp.obj`, `immediate_drawer.cpp.obj`, `imgui_*.obj`, `presenter.cpp.obj`,
  `graphics_*.obj`, `keybinds.cpp.obj`, `windowed_app*.obj`, `rex_app.cpp.obj`.
- **Game-specific code with no published source:**
  | Object | Size | Defined symbols |
  |--------|------|-----------------|
  | `main.cpp.obj` | 1 020 071 | 4430 |
  | `t6_netplay.cpp.obj` | 899 787 | 2254 |
  | `t6_watch.cpp.obj` | 48 748 | 182 |
  | `t6_status.cpp.obj` | 19 990 | 96 |

  `main.cpp.obj` includes `tekken6_app.h`. `t6_netplay.cpp.obj` includes
  `C:/Users/Volum/t6pkg/src/src/t6_netplay.cpp`.

The SDK graphs code in this archive duplicates open SDK sources. The four game files do not.
This project must rewrite those four files. Section 7.3 gives the symbols they must provide.

### 4.5 The runtime DLLs

`builder/runtime/`: `rexruntime.dll`, `rexgpu-xenos.dll`, `libc++.dll`, `libunwind.dll`,
`mmjoystick.ini`. These are build outputs of the SDK. `rexgpu-xenos` is a plugin
(`src/graphics/plugin_main.cpp` in the SDK). The SDK builds both for Linux
(`rexruntime`, `rexgpu-xenos.so`). These DLLs are **not** a blocker. The archive in 4.4 is.

### 4.6 Shader caches

| File | Magic | Backend |
|------|-------|---------|
| `tools/shader_cache/4E4D07FC.xsh` | `XESH` (`0x48534558`) | **Shared.** Both backends read it. |
| `tools/shader_cache/4E4D07FC.rov.d3d12.xpso` | `XEPSDXRO` | D3D12 only. Not portable. |

Evidence in the SDK:

- `src/graphics/d3d12/pipeline_cache.cpp:330` and `src/graphics/vulkan/pipeline_cache.cpp:504`
  both open `<cache_root>/shaders/shareable/<title_id>.xsh`.
- `d3d12/pipeline_cache.cpp:253` writes `<title_id>.<rov|rtv>.d3d12.xpso`.
- `vulkan/pipeline_cache.cpp:421` writes `<title_id>.<fsi|fbo>.vk.xpso`.
- The stored header holds only the Xenos ucode hash and the ucode, with `kVersion = 0x20201219`
  on both backends (`include/rex/graphics/vulkan/pipeline_cache.h:102`,
  `include/rex/graphics/d3d12/pipeline_cache.h:101`).

**Conclusion:** reuse the shipped `.xsh`. Discard the `.xpso`. The Vulkan backend builds its
own pipeline cache.

### 4.7 The launcher

`Tekken6 Native Online.exe` is a Qt6 Windows application (Qt6Core/Gui/Widgets/Network/Svg,
`platforms/qwindows.dll`, `tls/qschannelbackend.dll`, MSVCP140, VCRUNTIME140).

Tasks: login, lobby, matchmaking, build orchestration, update, and starting
`native/tekken6.exe` with `T6_*` environment variables. It writes
`runs/online_%1/tekken6.log`. The lobby server is overridable with `T6_LOBBY_SERVER`.

The game binary has no lobby strings and no HTTP code. Matchmaking is launcher-side.

---

## 5. Target architecture

```
tekken6-linux/
  CMakeLists.txt                 # from rexglue init, adapted
  tekken6_manifest.toml          # [project] name, sdk_version, game_root / [entrypoint]
  tekken6_config.toml            # copy of the release file (codegen hints + hooks)
  src/
    main.cpp                     # REX_DEFINE_APP(tekken6, Tekken6App::Create)      [rewrite]
    tekken6_app.h                # class Tekken6App : public rex::ReXApp            [rewrite]
    t6_netplay.cpp               # GGPO session, snapshot, resim, hooks             [rewrite]
    t6_netplay.h                 # hook declarations + snapshot API
    t6_watch.cpp / t6_watch.h    # value watches (t6_watch_on, t6_rwatch)           [rewrite]
    t6_status.cpp                # status overlay                                   [rewrite]
  tools/
    pagemap_patch.py             # copy from the release package
    rawstore_patch.py
    sndcall_patch.py
    faststore_patch.py
  generated/default/             # codegen output, then patched. Not in git.
  thirdparty/rexglue-sdk/        # SDK source tree, tag v0.10.0 (or an installed SDK)
```

Runtime data (not in git):

```
<prefix>/
  game/                          # extracted ISO: Default.xex + DATA*.BIN
  cache_root/shaders/shareable/4E4D07FC.xsh   # from the release package
  tools/menu_vs.bin              # from the release package, if needed
```

Pin the SDK to **v0.10.0**. The release manifest declares `sdk_version = "0.10.0"`
(`build_native.py` writes it into `tekken6_manifest.toml`).

---

## 6. Work breakdown

| Phase | Work | Exit criterion |
|-------|------|----------------|
| W0 | Clone and build the SDK for Linux: `cmake --preset linux-amd64`, `cmake --build --preset linux-amd64-release`. Run `scripts/check_vulkan_stack.py`. | `rexglue`, `librexruntime.so`, `librexgpu-xenos.so` exist. |
| W1 | Create the project (`rexglue init`), copy `tekken6_config.toml`, run codegen on the XEX. | `generated/default/tekken6_recomp.*.cpp` exists, no codegen error. |
| W2 | Port the flags to Linux, compile the generated code for the first time. | All generated objects compile. |
| W3 | Write a minimal `src/main.cpp` + `tekken6_app.h` (a `rex::ReXApp` subclass, no hooks, no netplay). Link and boot. | The game reaches the first frame. |
| W4 | Add the four hook functions (empty bodies first), then the snapshot machinery: `t6_dirty`, `t6_pagemap`, `t6_cap`, `t6_timebase`, `t6_watch`. Run `tools/*.py`, recompile. | Offline VS match runs. |
| W5 | Verify offline modes and the sound boundary. | Practice and Arcade work. |
| W6 | Implement `t6_netplay.cpp`: GGPO session, save/load, resim, input hook. | A peer-to-peer rollback match works. |
| W7 | Launcher: build helper, prefix setup, input map, `T6_*` env plumbing. | One command builds and starts the game. |

Do W3 before W4. A booting binary finds the missing symbols faster than a full rewrite.

---

## 7. Interface contracts

### 7.1 Build flags

`builder/compile.json` for Windows. The Linux translation:

| Windows flag | Linux action |
|--------------|--------------|
| `-DREX_HAS_D3D12=1` | Drop. The SDK CMake defines the backend. |
| `-municode`, `-mwindows` | Drop. |
| `-march=x86-64-v2`, `-O3`, `-DNDEBUG`, `-std=c++23`, `-msse4.1` | Keep. |
| `-fasync-exceptions` | Drop. Windows-only. |
| `-gline-tables-only` | Keep. |
| `-lws2_32 -luser32 -lgdi32 -lwinmm -limm32 -lole32 -loleaut32 -lversion -luuid -ladvapi32 -lsetupapi -lshell32 -ldinput8 -ldxgi -ldxguid -lpthread` | Drop. Use the CMake targets. |
| `{BUILDER}/lib/*.a` (except the game archive) | Drop. Use `find_package(rexglue)` targets. |
| `-Wl,--whole-archive libt6prebuilt.a` | Drop. Replace with the four rewritten source files. |

Prefer the SDK CMake integration (`rexglue_setup_target`). It applies the recomp options and
registers the codegen custom command. `resources/templates/init/rexglue_cmake.inja` is the
reference.

`generate_exception_handlers` is absent from `tekken6_config.toml`, and the SDK default is
`false` (`include/rex/codegen/config.h:99`). The generated code does not emit `__try/__except`.

### 7.2 Codegen contract

`tekken6_manifest.toml`:

```toml
[project]
name = "tekken6"
sdk_version = "0.10.0"
game_root = "../game"

[entrypoint]
file_path = "../game/Default.xex"
out_directory_path = "generated/default"
includes = [
  "tekken6_config.toml",
]
```

Codegen emits: `tekken6_pch.h`, `tekken6_funcs.h`, `tekken6_init.h`, `tekken6_init.cpp`,
`tekken6_register.cpp`, `sources.cmake`, and `tekken6_recomp.*.cpp`. The patch tools then
modify `tekken6_pch.h` and `tekken6_recomp.*.cpp`.

Codegen emits `extern` declarations for the midasm hooks. The signature follows the
`registers` list (`src/codegen/function_graph.cpp:422`):

```cpp
extern void T6FrameHook(PPCRegister& r3);
extern void T6InputHook(PPCRegister& r1);
extern void T6SeedHook(PPCRegister& r3);
extern void T6CrtSeedHook(PPCRegister& r3);
```

`PPCRegister` is `rex::ppc::Register` (`include/rex/ppc/context.h:213`).
A hook that returns `bool` comes from `return_on_false` / `jump_address_on_*`. The T6 config
does not use those keys.

### 7.3 Symbol contract for the rewritten files

The patch tools inject these declarations into `tekken6_pch.h`. The generated code calls them.
The Linux sources must define every symbol below with the exact name and signature.

**`pagemap_patch.py`**

```cpp
extern "C" unsigned char t6_pagemap[0x100000];
extern "C" constinit thread_local unsigned char t6_thread_bit;
extern "C" unsigned char t6_fine_heap[0x480000];   // 0xA7290000-0xABA90000
extern "C" unsigned char t6_fine_data[0x69000];    // 0x82726000-0x82DB8000
extern "C" unsigned char t6_fine_virt[0x400000];   // 0x40000000-0x44000000
extern "C" unsigned char t6_dirty[0x100000];       // per 4 KB page
extern "C" unsigned char t6_dirty_l1[0x4000];      // per 64 pages
extern "C" unsigned char t6_fine_page[0x100000];
extern "C" unsigned char t6_fine_changed;
extern "C" int t6_watch_on;
extern "C" void t6_watch(u32 addr, unsigned long long value, int size, const char* fn);
extern "C" int t6_rwatch_on;
extern "C" unsigned t6_rwatch(u32 addr, unsigned value, int size, const char* fn);
extern "C" uint64_t t6_timebase();
```

Page kinds: `1` game, `2` MMC sound, `4` CnuSound2, `8` XAudio2, `16` Direct3D,
`32` written by a non-main thread, `128` worker thread infrastructure.

The tool also writes `t6_pagemap` to the file named by `T6_PAGEMAP=<file>`. `t6_timebase()`
backs `REX_QUERY_TIMEBASE()`.

**`sndcall_patch.py`**

```cpp
extern "C" constinit thread_local int t6_snd_capture;
extern "C" void t6_cap(u32 a, u32 n);
extern "C" void t6_snd_call(PPCContext& ctx, uint8_t* base, u32 addr, PPCFunc* fn);
extern "C" void t6_lib_alloc(PPCContext& ctx, uint8_t* base);
extern "C" void t6_lib_free(PPCContext& ctx, uint8_t* base);
```

Library ranges: CnuSound2 `0x82570000-0x82590000`, XAudio2 `0x824EE000-0x82503000`.
Capture also covers CRT `0x826A0000-0x826B8000` and the heap allocator
`0x8223B000-0x8223E000`. The tool rewrites calls to `sub_824C64B8` and `sub_824C6550`
inside the library, and routes direct and vtable calls from game code through `t6_snd_call`.

**`rawstore_patch.py`**

No new symbols. It wraps `simde_mm_store_si128`, `dcbz`/`dcbzl` memset, and the
`stwcx`/`stdcx` compare-and-swap sites with `T6_MARK` from `pagemap_patch.py`.

**`faststore_patch.py`**

No new symbols. It rewrites `REX_STORE_U*` through the stack pointer into `T6_STORE_SP_*`.
It also disables the value watches unless the build sets `-DT6_WATCH_BUILD`, and it replaces
`REX_LOAD_U8/U16/U32` with direct reads.

**Also required (from `t6_netplay.cpp.obj`):** `T6NetFrame`, `T6NetResim`, `T6Resimulating`,
`ParkSession`, `t6_frame_ctx`, `t6_snd_capture`, plus these GGPO symbols imported by the object:
`ggpo_start_session`, `ggpo_add_player`, `ggpo_add_local_input`, `ggpo_synchronize_input`,
`ggpo_advance_frame`, `ggpo_idle`, `ggpo_close_session`, `ggpo_set_frame_delay`,
`ggpo_set_max_prediction`, `ggpo_set_disconnect_notify_start`, `ggpo_set_disconnect_timeout`,
`ggpo_get_network_stats`.

### 7.4 App contract

The SDK supplies the whole application skeleton. `include/rex/rex_app.h` defines
`rex::ReXApp` with the phases `SetupEnvironment`, `SetupPresentation`, `OnFinalizePaths`,
`ConstructRuntime`, `LaunchModule`, and these overridable hooks: `OnPostInitLogging`,
`OnPreSetup(RuntimeConfig&)`, `OnLoadXexImage(std::string&)`, `OnPostLoadXexImage`,
`OnPostSetup`, `OnCreateDialogs(ImGuiDrawer*)`, `OnShutdown`, `OnConfigurePaths(PathConfig&)`,
`OnFinalizePaths`, `OnConfigureFonts`, `OnConfigureStyle`, `OnPreLaunchModule`.

`PathConfig` fields: `game_data_root`, `user_data_root`, `update_data_root`, `cache_root`,
`metadata_root`, `config_path`. Command line accepts `--game_data_root=`, `--cache_root=`,
`--fullscreen=`, `--gpu`, and the draw-resolution flags.

Reference skeleton: `resources/templates/init/app_header.inja`.

---

## 8. Environment variables

Read from the game binary (`t6_netplay.cpp.obj` strings and the `rexruntime.dll` strings).
The launcher sets the subset marked "launcher".

**Netplay**

`T6_NET` (enable), `T6_NET_REMOTE` (remote endpoint), `T6_NET_LOCAL_PORT`, `T6_NET_PLAYER`,
`T6_NET_SEED`, `T6_NET_DELAY`, `T6_NET_INPUT`, `T6_NET_KEYS`, `T6_NET_KEYS_GLOBAL`,
`T6_NET_CMD`, `T6_NET_STATUS`, `T6_NET_LOCKSTEP`, `T6_NET_LOCKSTEP_DELAY`,
`T6_NET_FRAMESKIP`, `T6_NET_VSYNC`, `T6_NET_AUDIO_ASYNC`, `T6_NET_AUDIO_LATE`,
`T6_NET_AUTOPARK`, `T6_NET_FASTBOOT`, `T6_NET_SKIPINTRO`, `T6_NET_PRIO`, `T6_NET_LOCK`,
`T6_NET_WAIT`, `T6_NET_HOLD`, `T6_NET_HOLD_STEP`, `T6_NET_DUMP_DIR`, `T6_NET_DUMP`,
`T6_NET_HEAPDUMP`, `T6_NET_RAWDUMP`, `T6_NET_PAGEHASH`, `T6_NET_AREA_HASH`,
`T6_NET_DIRTYSTAT`, `T6_NET_TAILSTAT`, `T6_NET_SAMPLE`, `T6_NET_ALIGN`, `T6_NET_CLOCK_PUSH`,
`T6_NET_PHASE`, `T6_NET_PHASE_STEP`, `T6_NET_FIND_POS`, `T6_NET_RESIM_SAVE_ALL`,
`T6_NET_SAVE_FIRST`, `T6_NET_SAVE_TWO_PASS`, `T6_NET_WORKER_DIAG`, `T6_NET_TEST_MASH`,
`T6_NET_TEST_STALL`.

**Offline and test modes**

`T6_PLAY`, `T6_PLAY_UNTIL`, `T6_PRACTICE`, `T6_STAGE`, `T6_CHAR_P1`, `T6_CHAR_P2`,
`T6_NAME_P1`, `T6_NAME_P2`, `T6_ROUNDS`, `T6_ROUND_TIME`, `T6_ARCADE_CHALLENGE`,
`T6_AUTO_CONFIRM`, `T6_TEST_CHAR`, `T6_TEST_PAD`, `T6_TEST_RANDOM_STAGE`.

**Determinism and seed**

`T6_FIX_SEED`, `T6_VIRTUAL_TB`, `T6_COSMETIC_RAND_SHARED`, `T6_COSMETIC_TRACE`.

**Input and video**

`T6_KEYMAP`, `T6_PAD_MAP`, `T6_NO_BLUR`, `T6_NO_VIBRATION`, `T6_NO_SHADER_CACHE`,
`T6_SOUND_SEPARATE`.

**Snapshot and diagnostics**

`T6_SNAP_EXCLUDE_FILE`, `T6_DUMP_AT_BATTLE`, `T6_DUMP_DATA`, `T6_DUMP_IMAGE`,
`T6_FRAME_DATA`, `T6_FRAME_DATA_DEBUG`, `T6_FP_DUMP`, `T6_MODEDUMP`, `T6_MOVE_DUMP`,
`T6_MOVE_DUMP_RAW`, `T6_HOLD_DUMP`, `T6_HOLD_IMAGE`, `T6_LOAD_HOLD_GPU`,
`T6_LOAD_NO_GPU_WAIT`, `T6_GROUP_PROBE`, `T6_HASH`, `T6_HASH_RANGES`, `T6_EXCL_INC`,
`T6_CAL_ASYNC`, `T6_LOG_MODE`, `T6_HANG_MS`, `T6_PAGEMAP`, `T6_WATCH`, `T6_RWATCH`.

**Threading and performance**

`T6_JOB_WORKERS`, `T6_JOB_SEM`, `T6_NO_GLOBAL_LOCK`.

**Launcher only**

`T6_LOBBY_SERVER`, `T6_UPDATE_API`, `T6_LAUNCHER_LOG`, `T6_LAUNCHER_NOFOCUS`,
`T6_LAUNCHER_SHOW_GAME`, `T6_LAUNCHER_WINDOWED`.

**Runtime**

`REX_FILE`, `REX_LOG_LEVEL`, `REX_ROUND_ROBIN_CPU`. From `rexruntime.dll`: `T6_CS_SPIN_US`,
`T6_LOOKUP_CACHE`, `T6_SEM_SPIN_US`.

> The name list is verified. Most semantics are not. Confirm each variable against
> behaviour during W4 and W6. Add the findings to this table.

---

## 9. GGPO

`builder/lib/libggpo.a` is a Windows archive with 14 members: `p2p`, `spectator`, `synctest`,
`bitvector`, `game_input`, `input_queue`, `log`, `main`, `udp`, `udp_proto`, and 4 more. The
API is the classic GGPO C API. The library is BSD-licensed and portable.

Work:

1. Take the GGPO source, build it for Linux, and link it as a static library.
2. Match the upstream version. The wire protocol must match the Windows build for
   peer-to-peer play against Windows players.
3. `main.cpp.obj` inside the archive is GGPO's own test entry point. Do not link it.

GGPO handles transport and rollback timing. `t6_netplay.cpp` handles the game-specific part:
the snapshot, the resim loop, the hooks, and the sound boundary.

---

## 10. Risks

| ID | Risk | Impact | Mitigation |
|----|------|--------|------------|
| R1 | Four game files have no source. | High. Core work. | Rewrite from the documented hook addresses and the patch-tool symbols. Boot first (W3), then add behaviour. |
| R2 | The snapshot may not be deterministic on Linux. | High. | The patch tools already track dirty pages, sound writes, and foreign threads. Compare the sim against the Windows build frame by frame (see section 11). |
| R3 | The Vulkan backend may differ from D3D12 in guest GPU behaviour. | Medium. | The Xenos command processor is shared. Report backend bugs upstream. |
| R4 | Missing symbols at link time. | Low. | Section 7.3 is the checklist. |
| R5 | GGPO version mismatch. | Medium. | Diff the Windows archive symbols against the Linux build. |
| R6 | No cross-play with the official lobby. | Medium. | Accept it. Use direct peer-to-peer. Document the pairing rule (exe hash). |
| R7 | The `.xpso` cache cannot be reused. | Low. | Expected. The first run rebuilds it. Keep the `.xsh`. |
| R8 | Closed driver or Vulkan feature gap. | Medium. | Run `scripts/check_vulkan_stack.py`. Test on a second GPU vendor. |
| R9 | Legality of derived assets. | Medium. | Never commit game data, `.xsh`, or `menu_vs.bin`. Keep them gitignored. |

---

## 11. Verification

**V1 — SDK.** `cmake --preset linux-amd64` then `cmake --build --preset linux-amd64-release`
from the SDK root. Check the presence of `rexglue`, `librexruntime.so`, `librexgpu-xenos.so`.
Run `scripts/check_vulkan_stack.py`.

**V2 — Codegen.** Run codegen on the XEX. Compare the generated file count and the function
count against the Windows build log (`native/build.log` in the release, if available) or
against a Windows run.

**V3 — Boot.** The binary opens a window and reaches a frame. Log the guest RIP at the first
`T6FrameHook` call.

**V4 — Offline match.** Play a VS match to the end. Check video, audio, and input.

**V5 — Determinism.** Use `T6_FIX_SEED` and a fixed input sequence. Run twice. The guest state
hash (`T6_NET_PAGEHASH`, `T6_HASH`) must match.

**V6 — Netplay.** Two Linux peers, one match, one rollback event (force with a packet drop).
Then one Linux peer against one Windows/Proton peer.

**V7 — Optional oracle.** Run the Windows build under Proton to get reference behaviour and
reference frame data. Use it only as a reference. It is not the deliverable.

---

## 12. Open questions

1. What does `T6_NET_REMOTE` accept — `ip:port`, or a bare address with a fixed port?
2. What is the exact GGPO version in `libggpo.a`?
3. Does `menu_vs.bin` affect the offline path only?
4. Which `T6_*` variables does the launcher set per match, and with which values? Capture the
   launcher's process environment on Windows to answer this.
5. Are the `T6Watch`/`T6Status` overlays needed, or are they debug-only?
6. Does the release ship a `.pdb` or a build log that maps `main.cpp.obj` symbols to source
   lines? (The launcher has a `.pdb` path string. The package does not contain the file.)

---

## 13. References

| Source | Path or URL |
|--------|-------------|
| Release package | `Tekken6 Native Online` v2.1.5, `tekken6_native_online.zip` |
| Build script | `builder/build_native.py` |
| Codegen config | `builder/project/tekken6_config.toml` |
| Patch tools | `builder/project/tools/*.py` |
| Build flags | `builder/compile.json` |
| ReXGlue SDK | https://github.com/rexglue/rexglue-sdk (tag `v0.10.0`, BSD-3) |
| SDK app base | `include/rex/rex_app.h` |
| SDK project template | `resources/templates/init/*.inja` |
| SDK GPU plugin | `src/graphics/plugin_main.cpp`, `src/graphics/vulkan/*` |
| SDK SEH for POSIX | `src/core/seh_posix.cpp` |
| XDVDFS | https://github.com/antangelo/xdvdfs (Rust) |

License note: ReXGlue is BSD-3-Clause. The Xbox 360 and Tekken 6 trademarks belong to their
owners. This project ships no game code and no game data. The user supplies their own copy of
Tekken 6 (USA, Europe) Rev 1.
