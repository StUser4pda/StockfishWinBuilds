# Stockfish Windows Builds

Automated GitHub Actions workflow to build Windows Stockfish binaries (CLANG/MSYS2) for multiple CPU architecture variants and publish them as workflow artifacts.

## What this does
This repository contains a GitHub Actions workflow that checks out the official Stockfish repository, builds Windows executables (Stockfish.exe) using MSYS2 CLANG64 and Intel SDE for profile-guided builds, strips symbols, and uploads per-architecture artifacts for download.

See the workflow: [.github/workflows/build.yml](.github/workflows/build.yml)

## Features
- Builds multiple CPU-targeted Stockfish variants (AVX2, AVX512, BMI2, SSE4.1, etc.)
- Uses MSYS2 CLANG64 toolchain
- Uses Intel SDE to emulate instruction sets for profile-guided builds
- Produces and uploads a single src/Stockfish.exe artifact per matrix entry
- Workflow is manually triggerable via workflow_dispatch (optionally target a specific commit)
- Artifacts retained for 30 days

## Matrix architectures
The workflow builds the following ARCH variants:
- x86-64-universal
- x86-64-avx512icl
- x86-64-vnni512
- x86-64-avx512
- x86-64-avxvnni
- x86-64-bmi2
- x86-64-avx2
- x86-64-sse41-popcnt
- x86-64

Artifact naming format:
Stockfish-dev-YYYYMMDD-<short-sha>-windows-<arch>
(The workflow composes a version string like `dev-YYYYMMDD-<sha>` using git date + short SHA.)

## How it works (high-level)
- The job runs on windows-latest.
- It checks out official-stockfish/Stockfish (submodules: recursive) — you can pass a `commit` via the workflow input to build a specific commit.
- Sets up MSYS2 (CLANG64) and installs the Mingw Clang toolchain.
- Installs and configures Intel SDE (the workflow uses petarpetrovt/setup-sde).
- In an MSYS2 shell it runs `make ... profile-build` with COMP=clang and ARCH set per matrix entry, using SDE as RUN_PREFIX to generate profile data for instruction-set emulation.
- Strips debug symbols with `llvm-strip` and uploads src/Stockfish.exe as an artifact.

Key workflow file: `.github/workflows/build.yml`

## Run the workflow (manual)
1. On GitHub, go to this repository → Actions → "Build Stockfish — CLANG Windows" workflow and click "Run workflow".
2. Optionally provide a `commit` input to build a specific commit SHA/ref. If left empty, the workflow builds the current default checked-out ref from official-stockfish/Stockfish.

## Download artifacts
After a run completes, download artifacts from that workflow run in the Actions UI.

Using the GitHub CLI:
1. List recent runs:
   gh run list --repo StUser4pda/StockfishWinBuilds
2. Download a specific run's artifact:
   gh run download --repo StUser4pda/StockfishWinBuilds <run-id> --name "Stockfish-dev-YYYYMMDD-<sha>-windows-<arch>"

Replace `<run-id>` with the run identifier from `gh run list`.

## Build locally (rough steps)
To reproduce the CI flow locally on Windows:
1. Install MSYS2 and open the MSYS2 MinGW CLANG64 shell.
2. Install required packages:
   pacman -Syu
   pacman -S --needed mingw-w64-clang-x86_64-toolchain make git
3. Obtain Intel SDE and note the path to sde.exe (SDE_PATH). In the MSYS2 shell you can convert a Windows path with `cygpath -u`.
4. Clone Stockfish:
   git clone --recurse-submodules https://github.com/official-stockfish/Stockfish.git
   cd Stockfish/src
5. Build (example):
   export SDE_PATH="C:/path/to/sde"
   SDE_BIN=$(cygpath -u "$SDE_PATH")/sde.exe
   make -j$(nproc) COMP=clang ARCH=x86-64-avx2 profile-build RUN_PREFIX="$SDE_BIN -future --"
6. Strip:
   llvm-strip src/Stockfish.exe

Notes:
- The workflow uses Intel SDE for emulation during profile-guided builds; you need a local SDE binary (sde.exe).
- Use the MSYS2 CLANG64 environment to match the CI setup.

## Troubleshooting
- If the workflow fails saying `sde.exe not found`, ensure the `SDE_PATH` provided by the setup action is valid in the MSYS2 environment (the workflow converts it with `cygpath`).
- If builds fail due to missing toolchain packages, confirm `mingw-w64-clang-x86_64-toolchain` and `make` are installed in MSYS2.
- For unexpected build failures, view the full Actions run log in the GitHub UI for the failing matrix entry.

## Contributing
This repository only contains CI workflow(s). If you want the workflow to trigger different configurations, please open an issue or PR with the desired changes. If you want me to add the README directly to this repository, tell me and I will create the commit.

## License
No license file is present in this repository. Add a LICENSE file if you want to make the repo contents explicitly reusable.
