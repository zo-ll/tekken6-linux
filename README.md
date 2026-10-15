# tekken6-linux

Play **Tekken 6 Native Online (Xbox 360)** on Linux, against Windows players.

[Tekken 6 Native Online](https://github.com/Ekspym/Tekken6-native-online) is a Windows release.
This repo does not port it. It runs the unchanged Windows launcher and game under Proton, and it
applies the fixes they need. Because the files are unchanged, the exe hash, the lobby and the
rollback netcode are the same as on Windows.

This repo ships no game code and no game data. You need your own Tekken 6 (USA, Europe) Rev 1
disc image (`.iso`).

## Requirements

- Proton: GE-Proton (newest is used) or Proton Experimental. Install GE-Proton with ProtonUp-Qt,
  or Proton Experimental from Steam. Steam does not need to run.
- A Vulkan GPU driver.
- `curl`, `python3`.
- About 10 GB of free disk space.

## Use

```sh
./t6 install      # download the latest release, verify its SHA-256, extract, apply the fixes
./t6 run          # start the launcher
./t6 desktop      # optional: add a menu entry
```

The first time, in the launcher, open **Profile -> BUILD GAME** and pick your `.iso`. The launcher
builds `native/tekken6.exe` with the bundled Windows compiler (under Proton). After that, the
launcher starts the game for each match. The launcher updates itself.

Other commands:

- `./t6 doctor`: show the setup and check the fixes.
- `./t6 fix`: apply the fixes only (`run` does this every time).
- `./t6 check <iso>`: check that the `.iso` is the supported version.
- `./t6 build <iso>`: build the game from the terminal (the same as BUILD GAME in the launcher).
- `./t6 hash`: print the SHA-256 of the built game. It must be the same as on a Windows PC.

Settings (environment variables):

| Variable | Default | Meaning |
|----------|---------|---------|
| `T6_HOME` | `~/Games/Tekken6NativeOnline` | The release package folder. |
| `T6_DATA` | `~/.local/share/tekken6-linux` | The Proton prefix and `launcher.log`. |
| `T6_GPU` | `auto` | `auto`: use the NVIDIA GPU if there is one (PRIME offload). `nvidia`: always. `default`: do nothing. |
| `PROTON` | auto | A Proton folder (the one that holds the `proton` script). |

## The fixes

| Problem | Fix | Source |
|---------|-----|--------|
| The launcher does not start: `Qt6Core.dll` needs `icuuc.dll`, which the package does not ship. | Copy Proton's `icu.dll` as `icuuc.dll`. It exports all 21 ICU functions that Qt imports. | upstream issue #19 |
| The launcher aborts in `mscms.dll.WcsCreateIccProfile` (a Wine stub). | Rename the sRGB color profile in the prefix. | upstream issue #10 |
| The 3D scene is black (the launcher forces the `rtv` render path under Wine). | Add `execute_unclipped_draw_vs_on_cpu = true` to `native/tekken6.toml`. | upstream issue #18 |

## Status

Tested on Fedora 44, GE-Proton10-34, release v2.2.1, 16 cores, 15 GB RAM:

- Done: install (download and SHA-256 check), prefix creation, all fixes, the launcher starts.
- Done: the game build from the ISO under Proton: 106 files, 327 s. It used most of the RAM
  and about 6 GB of zram swap (the builder runs one compiler job per core, minus one).
- Done: Practice mode on the RTX 4060: 60 FPS (16.67 ms average frame time), a few single hitches
  of about 85 ms, no crashes, correct 3D rendering.
- Not tested yet: an online match and the exe hash against a Windows player. The
  launcher compares the game files with the opponent's and reports "your game files differ"
  on a mismatch.

## Layout

```
t6                          the tool
docs/native-port-spec.md    the abandoned native-rebuild plan (research)
refs/                       downloads for research. Not in git.
```
