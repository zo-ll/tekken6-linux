# tekken6-linux

Native Linux port of **Tekken 6 Native Online (Xbox 360)**.

The upstream release is a Windows build. It uses ReXGlue (an open-source Xbox 360 static
recompiler). This project rebuilds the game-specific code that the release does not publish,
then builds a Linux ELF binary with the ReXGlue Vulkan backend. No Wine, no Proton.

Read [SPEC.md](SPEC.md). It holds the full plan: architecture, work breakdown, interface
contracts, risks, and verification.

## Status

Draft 1. Not started.

## Quick facts

- Engine: ReXGlue SDK `v0.10.0` (BSD-3), Vulkan backend on Linux.
- Input to the build: your own Tekken 6 (USA, Europe) Rev 1 for Xbox 360, plus the
  `tekken6_native_online.zip` release package (for the codegen config and the patch tools).
- This repo ships no game code and no game data.
- Netplay: GGPO, direct peer-to-peer. No official lobby, no cross-play with it.

## Layout

```
SPEC.md              the spec
CMakeLists.txt       project build (to add)
tekken6_config.toml  codegen config (to copy from the release package)
src/                 main.cpp, tekken6_app.h, t6_netplay.cpp, t6_watch.cpp, t6_status.cpp
tools/               codegen patch scripts (to copy from the release package)
generated/           codegen output, patched. Not in git.
```
