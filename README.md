# Driveclub Native

[![YouTube](https://img.shields.io/badge/YouTube-FF0000?style=for-the-badge&logo=youtube&logoColor=white)](https://youtube.com/@blacklineinteractive)
[![Windows Auto Build](https://github.com/fzfalcon/Driveclub-Port/actions/workflows/windows-build.yml/badge.svg?branch=main)](https://github.com/fzfalcon/Driveclub-Port/actions/workflows/windows-build.yml)
[![Latest Windows Build](https://img.shields.io/badge/Windows-Latest%20Build-blue?style=for-the-badge&logo=windows)](https://github.com/fzfalcon/Driveclub-Port/releases/tag/auto-latest)

![LOGO](media/logo.png)

![Driveclub In-Game](media/2026-10-14_22-34-39.png)

A native port of **Driveclub** (PS4, CUSA00003) to **macOS** and **Windows**.

## Windows Downloads

Download the latest automatically published Windows launcher builds from GitHub Releases.

| Edition | Download |
|---|---|
| Windows — FSR | [DriveClub-FSR.exe](https://github.com/fzfalcon/Driveclub-Port/releases/download/auto-latest/DriveClub-FSR.exe) |
| Windows — No FSR | [DriveClub-NoFSR.exe](https://github.com/fzfalcon/Driveclub-Port/releases/download/auto-latest/DriveClub-NoFSR.exe) |
| All Windows builds | [Latest automatic release](https://github.com/fzfalcon/Driveclub-Port/releases/tag/auto-latest) |

**Automatic Windows builds:** GitHub Actions builds the launcher editions after pushes to `main` and updates the rolling `auto-latest` pre-release after a successful build.

> **Important:** These downloads are launcher executables, not a complete playable game package. You must provide your own legally acquired game files and configure the required runtime separately. Download links work after the automatic release workflow has uploaded the renamed executables.

## Platform Overview

| Platform | Game version | Renderer | Location | Maintainer |
|---|---|---|---|---|
| macOS (Intel) | v1.05 | Metal 3 | This repository's `src/` | [BlackLineInteractive](https://github.com/BlackLineInteractive) |
| Windows 10/11 | v1.28 | Vulkan (shadPS4's renderer), FSR 3.1 / FSR 4 / FSR 4.1.1, frame generation | [`windows/`](windows/README.md) | [qnfn](https://github.com/qnfn) |

**Windows highlight:** DCport 0.3.0 supports building AMD's **FSR 4.1.1** for Vulkan using your own DirectX 12 DLL on your PC. See the [Windows documentation](windows/README.md#-new-in-030-fsr-411-built-on-your-own-pc-in-one-click).

The sections below describe the macOS backend. The Windows backend (DCport) uses a native PS4 runtime based on [Supermedo/bloodborne_pc](https://github.com/Supermedo/bloodborne_pc), documented in [`windows/README.md`](windows/README.md).

### Windows — DCport

![DCport launcher on Windows](windows/media/launcher.png)

![DRIVECLUB BIKES on Windows, FSR 4](windows/media/ingame-bike.png)

![Event preview on Windows, FSR 4](windows/media/ingame-track.png)

## Repository Layout

| Path | Backend | Platform | What it is |
| --- | --- | --- | --- |
| `src/` | Metal (`src/gpu/metal`) | macOS | Our native runtime: ELF loader, HLE libraries, PM4 command processor, GCN → MSL |
| `windows/` | Vulkan (DCport) | Windows | Separate native runtime (bloodborne_pc + `runtime/driveclub-runtime.patch`), launcher, community patches |

`src/` and `windows/` are independent: each has its own runtime and build, and changes to one do not affect the other. Fixes found on one side are worth checking on the other (both translate the same GCN/PM4 stream).

## Architecture

```
eboot.bin (x86-64, natively executed)
   │  imports of 922 functions from 57 libraries
   ▼
Sony HLE libraries ── libkernel, libc, Pad, AudioOut, VideoOut, Network/PSN -> stubs
   │  sceGnmSubmitCommandBuffers / sceGnmDingDong
   ▼
GPU frontend ── PM4 packet parsing, register state, GCN shaders -> IR
   │  src/gpu/renderer.h
   └──► Metal 3 (macOS) - src/gpu/metal, IR -> MSL directly
```

Gnm/Gnmx are embedded directly into eboot: the game writes PM4 packets into the command buffer itself and passes only the finished buffer to the driver (`sceGnmDraw*` are not imported). Therefore, the GPU layer operates at the PM4 + GCN shader level.

### Performance Principles

- Shader IR -> MSL directly, without SPIR-V/SPIRV-Cross; disk cache for compiled pipelines.
- Argument buffers Tier 2 (bindless) - minimal resource binding overhead per draw.
- Up to 3 frames in flight; CPU waits for GPU only when running 3 frames ahead.
- On discrete AMD GPUs, memory is not unified: dirty page tracking and uploading modified ranges only.
- Explicit discrete GPU selection on dual-GPU MacBooks (`DC_GPU=<partial_name>` override).

## Requirements

**macOS**

- Intel Mac (x86_64), Metal 3-capable GPU (tested: AMD Radeon Pro 5500M)
- Xcode, CMake >= 3.24, Ninja, SDL3 (`brew install cmake ninja sdl3`)
- Your own copy of the game as a decrypted dump (fSELF/fPKG)

**Windows**

- Windows 10/11 64-bit, a CPU with AVX, a Vulkan 1.3 GPU (FSR 4 needs a recent AMD or NVIDIA GPU)
- Your own decrypted dump of Driveclub v1.28; no Python or build tools needed for the release builds

## Build

**macOS**

```bash
cmake -S . -B build -G Ninja
cmake --build build
./build/driveclub
```

**Windows**: download a release build, or build the launcher and runtime from [`windows/`](windows/README.md).

## Automatic Windows Releases

The workflow is located at `.github/workflows/windows-build.yml`.

- Runs on pushes to `main` and manual workflow dispatch.
- Runs launcher tests before building.
- Builds the FSR and non-FSR launcher editions.
- Uploads the executables as workflow artifacts.
- Updates the rolling `auto-latest` GitHub pre-release after a successful build.

See [Windows build workflow](https://github.com/fzfalcon/Driveclub-Port/actions/workflows/windows-build.yml).

## Game Files

Game files **are not included** in the repository and must never be committed (see `.gitignore`).
Unpacking PKG:

```bash
tools/extract.sh <path_to_pkg> extracted
```

eboot analysis (imports by library):

```bash
tools/elfinfo.py extracted/Image0/eboot.bin
tools/elfinfo.py extracted/Image0/eboot.bin --lib libSceGnmDriver
```

## Legal Disclaimer

- **No Proprietary Game Files**: This project does not distribute, host, or link to any copyrighted material, proprietary game assets, PKG dumps, ROMs, executables (`eboot.bin`), libraries (`.prx`), engine archives (`.dat`), audio, or textures. Users must provide their own legitimately acquired copy of Driveclub (CUSA00003).
- **Intellectual Property**: Driveclub is a registered trademark of Sony Interactive Entertainment LLC. All trademarks, service marks, game titles, and copyrighted assets belong to Sony Interactive Entertainment and Evolution Studios.
- **Educational and Preservation Purpose**: This repository is an independent reverse-engineering project conducted strictly for interoperability, software preservation, and educational purposes. It is not affiliated with, endorsed by, or sponsored by Sony Interactive Entertainment, Evolution Studios, or any associated entities.

## License

This project is licensed under the **GNU General Public License v2.0 or later (GPL-2.0-or-later)** - see the [LICENSE](LICENSE) file for details.

## Acknowledgments

- [shadPS4](https://github.com/shadps4-emu/shadPS4) - PS4 emulator, the primary source of knowledge on ELF loading, HLE libraries, PM4, and GCN shader recompilation. Its Vulkan renderer is the reference for GCN/PM4 semantics in the Metal backend, and the renderer of the Windows backend.
- [OpenGNM](https://ps4-opengnm.github.io/opengnm/) - documentation on PS4 Gnm and GCN shaders.
- Windows backend: deadinside28 and Supermedo (native PS4 runtime), FireBurn (FSR-Vulkan, FSR 4 assets), akitaonrails and the patch authors; see [`windows/README.md`](windows/README.md).
