# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

QBox is a SystemC/QEMU co-simulation framework that integrates QEMU into SystemC as a TLM-2.0 model. It enables cycle-approximate simulation of hardware platforms supporting ARM (Cortex-A/M/R, Neoverse), RISC-V (32/64), and Hexagon architectures.

## Build Commands

```bash
# Install dependencies (Ubuntu/macOS)
sudo scripts/install_dependencies.sh

# Configure (using presets: gcc, gcc-debug, clang, clang-debug, clang-lto, mac, mac-debug)
cmake --preset gcc

# Specify target architectures
cmake --preset gcc -DLIBQEMU_TARGETS="aarch64;riscv64;hexagon"

# Build
cmake --build --preset gcc --parallel

# Run all tests
ctest --preset gcc

# Run a single test by name
ctest --preset gcc -R <test_name>

# Install
cmake --install build
```

Without presets: `cmake -B build && cmake --build build --parallel && ctest --test-dir build --output-on-failure`

## Code Style

- C++14 minimum, Google-based clang-format with 4-space indentation, 120-column limit
- Formatting config in `.clang-format`; CI enforces it via `clang-format`
- Braces on new lines after classes and functions; pointer alignment left (`int* p`)
- Constructor initializer break before comma, no space before colon

## Architecture

**Three-layer design:**

1. **libqemu-cxx** (`qemu-components/common/`) — C++ wrapper around QEMU's C API. Provides `LibQemu`, `Cpu`, `Timer`, `MemoryRegion` abstractions. Target-specific code in `include/libqemu-cxx/target/`.

2. **libqbox** (`qemu-components/`) — SystemC TLM-2.0 integration layer. Each subdirectory is a component (CPU models, interrupt controllers, UARTs, timers, PCI, USB, NVMe, etc.). CPU models expose TLM initiator sockets for memory access.

3. **systemc-components** (`systemc-components/`) — Pure SystemC platform components independent of QEMU: router (address-based TLM transaction dispatcher), gs_memory (RAM/ROM), loader (ELF), exclusive_monitor, character backends (stdio/socket/file/TAP), Python binder.

**Platforms** (`platforms/`, `examples/`) combine these layers into complete SoCs configured via Lua scripts.

## Configuration

Platforms are configured via Lua files loaded with `--gs_luafile` / `-l`. Parameters can be overridden with `--param` / `-p`. CCI (Configuration, Control and Inspection) is used for runtime parameter management.

## Key CMake Options

- `LIBQEMU_TARGETS` — semicolon-separated target architectures
- `CPM_DOWNLOAD_ALL` — download all dependencies via CPM
- `ENABLE_PYTHON_BINDER` — Python integration (default: ON)
- `CPM_SOURCE_CACHE` — cache directory for offline builds

## Dependency Management

Uses CPM.cmake for automatic dependency downloads. Versions pinned in `package-lock.cmake`. If a `Packages/` directory exists locally, CPM uses it for offline builds. Key dependencies: SystemC, CCI, Google Test, pybind11, Lua.

## Contribution Requirements

- DCO sign-off required on commits: `git commit -s -m "message"`
- Branch from `main`, PR against `main`
