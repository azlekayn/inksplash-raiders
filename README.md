# InkSplash Raiders

**A Linux-first + Steam Deck fork of [SPR Recomp](https://github.com/Elie-0066/SPR-Recomp-Project)**, the unofficial
static recompilation of *Splatoon Raiders* (Nintendo Switch 2). Upstream targets Windows first; this fork builds,
tests and tunes on Linux (native Vulkan via RADV, Wayland/X11 through SDL 3) with the Steam Deck (1280x800, 60 fps)
as the handheld target. Fixes that are not Linux-specific are offered back upstream.

Same rules as upstream: no game code or data is included. You need your own copy of the game, and the executable you
build from it is for your own use only. Do not share builds.

### What this fork changes

- **Shadow blocks fixed** (Linux shadow storage, `runtime/vk_memory.c`): screen tiles no longer lose their sunlight
  for a frame (translucent dark rectangles flickering in every scene); frame interval 24 ms -> 16.7 ms on the test
  machine.
- **Cheaper GPU write-back**: large written storage spans are compared only up to the extent the GPU actually writes
  (53 MiB -> ~2 MiB a frame; ~3.5 ms of CPU per frame saved, important on the Deck's 4 cores).
- **Gyro on controllers that appear twice** (e.g. Flydigi Vader 5 Pro on its receiver: SDL's driver with motion
  sensors plus the kernel's xpad device without): the one with sensors is used.
- **Rumble no longer freezes Flydigi receivers** (changed rumble values at most every 30 ms): no more "stuck swimming".
- **Shader pre-compile loading screen** at boot, plus test/debug knobs (`TOTK_WINDOW_POS`, `TOTK_WINDOW_NOFOCUS`,
  `TOTK_PRESENT_SHOTS`, `TOTK_NO_SHADOW`, `TOTK_INPUT_TRACE`).

### Steam Deck status

Work in progress. A CPU profile on desktop puts the port at ~27 ms of CPU per frame across all threads (game code ~7.6
ms; the rest is the NVN -> Vulkan replay, write-back and thread-scheduler emulation). The plan: cut the per-frame
overheads, run the replay on its own thread (`TOTK_REPLAY_ASYNC=1`) and build against the Steam Runtime (sniper)
sysroot so the binary runs on SteamOS's older glibc.

Mirrors: [GitHub](https://github.com/azlekayn/inksplash-raiders) ·
[GitLab](https://gitlab.com/azlekayn/inksplash-raiders)

---

*Upstream README follows.*

# SPR Recomp

<a href="https://recomplauncher.com/game/spr-recomp-project"><picture><source media="(prefers-color-scheme: dark)" srcset="https://recomplauncher.com/assets/press/launcher/badge-light.svg"><img alt="Get it on RecompLauncher" src="https://recomplauncher.com/assets/press/launcher/badge-dark.svg" height="48"></picture></a>

## Dont forget to follow me on twitter for updates on futures releases ! : https://x.com/Lyriiiie

An unofficial static recompilation of *Splatoon Raiders* (Nintendo Switch 2) into a native Windows x86-64 executable
with a Vulkan renderer. It is a **fork of [DKB Recomp](https://github.com/Lucas-blip-png/dkb-pc-recomp)**, retargeted
from *Donkey Kong Bananza*; DKB Recomp is itself built on [TotK Recomp](https://github.com/valentderah/totk-pc-recomp).
The lifter, Horizon host, and Vulkan backend are largely retained from those projects; this fork adds the Raiders
target, its SDK instruction support, and the work needed for its RomFS, NVN 2 calls, and shaders. Bananza
documentation remains as reference material for shared parts of the pipeline.

The ARM64 game code is translated to C ahead of time and compiled with Clang. At run time, the host implements the
Switch services and answers NVN 2 graphics calls through Vulkan.

**This project is not affiliated with or endorsed by Nintendo. It contains no game code or data: you need your own
copy of the game, and the executable you build from it is for your own use only. Do not share builds.**

**If you are playing on AMD GPUs please see [AMD graphics cards](#amd-graphics-cards).**

(This is also HEAVILY Vibecoded and experimental just so you know, if there is any problem let me know, also yes it was played and tested on a very cheap PC)

## Status

**[Download the latest release](https://github.com/Elie-0066/SPR-Recomp-Project/releases/latest)** (Windows setup, no game data). **Early, playable in parts.** The title, the opening cutscene, the character creator, the hub and the first gameplay
zone render and play at about 60 fps on the test machine (Ryzen 5 4500, RTX 3050). Known open issue: the game's
neural-network upscaler is stood in for by FSR 1 EASU.

**Ultrawide.** The F1 menu's *Ultrawide* setting (Screen tab, after a restart) renders 21:9 (2560 x 1080, 3440 x 1440
or 3840 x 1600) or 32:9 (3072 x 864 or 3840 x 1080) with the camera's view widened; the HUD keeps its 16:9 layout in the middle. Docked mode only. The F1 menu
is in English, Russian and Korean.

**Game versions.** Update 65536 is the only version SPR Recomp is developed and tested on. The setup also builds the base
game (1.0.0) and later updates, but that support is **experimental**: it is not certain that they build, start or play
correctly. The setup says so with a warning when it sees another version. Report problems on another version only after
checking them on update 65536.

| Phase | State |
| --- | --- |
| Dump intake | Extracted base and update ExeFS/RomFS are read in place; no keys needed. |
| Lifter | 259,913 functions in update 65536 across three modules; no unsupported instructions. |
| Horizon host | Boots through `nnMain`, the RomFS mount, HID, audio, saves and the character creator. |
| NVN 2 | Draws, passes, compute and presents work; a few procs (DLSS, dirty tile maps) are still missing. |
| Shaders | Eglantine lowers all 3,690 seed stages. |
| Gameplay | The first zone plays at about 60 fps, with the player's body. |

The detailed, current phase plan is [docs/raiders-plan.md](docs/raiders-plan.md). Shared architecture, build instructions,
and graphics notes are in [docs/architecture.md](docs/architecture.md), [docs/building.md](docs/building.md), and
[docs/maxwell.md](docs/maxwell.md). Bananza-specific notes remain in the `bananza-*` documents for reference.

## Game content boundary

This is a source-code fork; it does not include game files. Use your own legally obtained dump. Do not add game data,
keys, extracted assets, generated game code, or derived shader files to commits, issues, or pull requests. Local paths
belong in the ignored `config.local.toml`; generated and extracted files belong under ignored `build/` or `generated/`.
Instructions for obtaining game files are not provided.

## Requirements

- Windows 10 or 11 on an x86-64 CPU with AVX2. Linux and macOS are not supported.
- A GPU with Vulkan 1.3 and a current driver. Tested on NVIDIA only; see [AMD graphics cards](#amd-graphics-cards).
- 16 GB of RAM plus a large page file (the process commits about 11 GB); 32 GB is comfortable.
- About 6 GB free for the build, besides the dump.
- Your own dump of *Splatoon Raiders* update 65536, as a `.nup` file or as extracted ExeFS and RomFS folders. No keys are
  needed.
- To build from source: the toolchain in [docs/building.md](docs/building.md) (MSYS2 UCRT64 Clang, Ninja, Visual
  Studio Build Tools, Python 3.12 and 3.14, the .NET SDK for the shader translator).

### AMD graphics cards

Whether SPR Recomp runs on AMD (Radeon) GPUs is **unknown**. Every test so far was on an NVIDIA RTX 3050, and the
maintainer has no machine with an AMD GPU to test on for now. The renderer only uses standard Vulkan, so it may work,
but the upstream projects saw AMD-specific problems (DKB Recomp had a device loss on AMD only), and some workarounds in
this code are tuned for NVIDIA's driver.

Version 0.1.8 has an **experimental** fix for AMD: a textured array the game never samples as a cube is no longer made
cube compatible, a combination AMD's Windows driver is known to sample wrongly (it matches the glitching sky texture
reported on a ROG Ally). Version 0.1.9 adds two more for AMD, from a report on an RX 9070 XT: the recorder thread is
off by default (it crashed at launch), and the compressed sky cube gets no block views (its sky and water came out
posterized). These fixes are **experimental**: they were checked on NVIDIA only, so they are not a guarantee that every
AMD problem is solved.

If you have an AMD card, a report is very welcome either way: open an
[issue](https://github.com/Elie-0066/SPR-Recomp-Project/issues) with your GPU, driver version, how far the game got
and the log (`logs\dkb.log`), even if everything works.

## Install

Download `SPR-Setup-<version>-windows.zip` from the [latest release](https://github.com/Elie-0066/SPR-Recomp-Project/releases/latest), unpack it into an empty
folder and run `SPR-Setup.exe`. It needs nothing else installed: the compiler, Python and the shader translator are in
the zip. The first install recompiles the whole game on your PC and takes about 30 minutes on a 6-core CPU (lifting,
then compiling about 280 MB of generated C, then preparing the shaders); later starts are immediate.

From a source checkout, the same setup runs with the repository's Python environment. It builds the game in one go from a `.nup` file or the extracted update folders, identifies the dump's version first
(warning when it is not the tested update 65536), and leaves a ready folder with `spr-recomp.exe` ([docs/installer.md](docs/installer.md)):

```powershell
.venv\Scripts\python.exe -m installer --game raiders --exefs <update-exefs> --romfs <complete-romfs> --dest <install-folder>
.venv\Scripts\python.exe -m installer.gui
```

## Build

The project uses the toolchain documented in [docs/building.md](docs/building.md): MSYS2 UCRT64 Clang and Ninja, plus
the repository's Python environments. With the Raiders update's ExeFS and RomFS directories already extracted:

```powershell
.venv\Scripts\python.exe recomp\gen.py --exefs <update-exefs> --out generated\raiders
.venv\Scripts\python.exe tools\build_game.py --version raiders --exefs <update-exefs> --romfs <update-romfs>
```

The build writes the executable and its derived index under `build\raiders`. Raiders' loose RomFS directory is indexed
in place, so it does not need to be repacked into a second multi-gigabyte image. Running it:

```powershell
build\raiders\spr-recomp.exe --boot
```

Running, settings and logs are covered in [docs/dkb-testing.md](docs/dkb-testing.md); controls in
[docs/controls.md](docs/controls.md).

## Reporting problems and contributing

Bugs go to the [issue tracker](https://github.com/Elie-0066/SPR-Recomp-Project/issues), with the template that fits: the commit you built, your GPU, driver and
Windows version, the log (`logs\dkb.log` next to `spr-recomp.exe`) and the steps to reproduce. Never attach game files,
keys, builds, shader caches or screenshots of the game. Issues about Raiders do not belong on the DKB Recomp or TotK
Recomp trackers.

Pull requests are welcome; read [CONTRIBUTING.md](CONTRIBUTING.md) first, and enable the content check before your first
commit:

```sh
git config core.hooksPath .githooks
```

Everyone taking part follows the [Code of Conduct](CODE_OF_CONDUCT.md).

## Credits

SPR Recomp stands on the work of [DKB Recomp](https://github.com/Lucas-blip-png/dkb-pc-recomp) and
[TotK Recomp](https://github.com/valentderah/totk-pc-recomp) and their contributors, whose commits and authors are kept
in this repository's history, and on the projects listed in [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md), among them
LAS Recomp, MacNativeGames, Eglantine (Eden), Ryujinx, Dear ImGui and SDL.

## Origins and licenses

This fork retains code from DKB Recomp and, through it, TotK Recomp. The relevant upstream attributions and terms are
in [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md). The repository as a whole is GPL-3.0-or-later because it includes
Eglantine's GPL shader lowering ([LICENSE](LICENSE)); files originating in TotK Recomp retain their MIT terms
([LICENSE-MIT](LICENSE-MIT)). Third-party code keeps its own license. The game and its trademarks are not licensed by
this repository. This is an independent, non-commercial fan project and is not affiliated with or endorsed by
Nintendo.
