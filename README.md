# SeriesBridge

**A research-driven Xbox Series X|S to Windows compatibility project**

> **Project status: Planning / early architecture.** This repository description is a project starting point, not a claim that a working Xbox compatibility runtime or GTA VI PC port exists.

SeriesBridge is a proposed Windows application made of two separate components:

1. A C++ desktop launcher for game selection, hardware detection, configuration, shader preparation, cache management, diagnostics, and launch control.
2. A C++ compatibility runtime that loads supported, lawfully obtained Xbox Series X|S title files and implements the Xbox-facing services required by a title.

The project is intended to investigate a practical Windows compatibility architecture, with GTA VI as a possible research target where the user already has lawful access to the required files. Compatibility must be demonstrated through testing; it must never be assumed from the fact that an executable can be loaded.

## Project goals

- Build a clean Windows launcher that opens before the game.
- Use C++20 for the launcher and runtime.
- Run compatible x64 title code natively where possible, rather than adding an unnecessary CPU instruction emulator.
- Implement Xbox-facing system services incrementally and report unsupported behavior clearly.
- Translate supported Xbox RDNA2 shader programs into a host-supported representation.
- Precompile shaders and pipeline states that can be discovered before launch, then compile newly encountered items at runtime.
- Preserve Xbox guest-memory semantics while mapping memory efficiently to discrete and integrated PC GPUs.
- Measure memory use, GPU budgets, transfer volume, shader preparation time, frame pacing, and compatibility failures.
- Make security, reproducible builds, and dependency provenance part of the design from the beginning.

## Non-goals and boundaries

- No decryption, DRM circumvention, entitlement bypass, or extraction of protected console packages.
- No distribution of game executables, copyrighted game assets, firmware, private keys, or proprietary SDK components.
- No bypassing of anti-cheat, online protections, or server-side game logic.
- No promise that every Xbox title or every game version will work.
- No claim that all shaders can be compiled before the game runs.
- No assumption that a PC with 16 GB of VRAM is equivalent to an Xbox with 16 GB of unified memory.

The runtime should accept only files the user is legally entitled to use and has made available through an authorized process. Game files should remain separate from the SeriesBridge source and release packages.

## Design overview

```text
┌───────────────────────────────────────────────┐
│             SeriesBridge Launcher             │
│                   C++ / Qt 6                  │
│                                               │
│  Library · Profiles · Hardware · Settings     │
│  Shader Preparation · Cache · Diagnostics     │
└──────────────────────┬────────────────────────┘
                       │ Versioned local IPC
                       ▼
┌───────────────────────────────────────────────┐
│              SeriesBridge Runtime             │
│                  C++20 / Win32                │
│                                               │
│  PE Loader & Import Router                     │
│  Xbox API Compatibility Services              │
│  Guest Memory Manager                          │
│  Graphics Compatibility Layer                 │
│  RDNA2 Shader Translation                      │
│  D3D12 Backend                                 │
└──────────────────────┬────────────────────────┘
                       ▼
┌───────────────────────────────────────────────┐
│             Windows / GPU Platform            │
│       WDDM · DXGI · Direct3D 12 · DXC          │
└───────────────────────────────────────────────┘
```

The launcher and runtime are separate processes. The launcher is responsible for the user experience and preparation workflow; the runtime owns executable loading, compatibility services, memory mappings, graphics execution, and game process state.

## Key engineering principles

### 1. Unified memory is a guest behavior to preserve

Xbox Series consoles use a unified physical memory architecture. A typical PC with a discrete GPU instead has system RAM and dedicated VRAM. SeriesBridge should preserve the game's expected guest addresses, page mappings, aliases, visibility rules, and synchronization behavior without forcing the host to reproduce the console's physical layout.

The memory manager should support:

- A guest address space independent of host pointers and D3D12 resource objects.
- Lazy page commitment rather than eager reservation of the entire guest range.
- Explicit page and alias tracking.
- CPU/GPU visibility and coherency tracking.
- UMA and discrete-GPU backing strategies.
- DXGI local/non-local memory budget monitoring.
- Residency, streaming, eviction, and safe resource lifetime management.
- Accounting for committed bytes, duplicated backing, transfers, and fence waits.

### 2. Shader preparation is both offline and runtime

The pre-launch pipeline should scan supported shader inputs, decode and translate known programs, compile generated HLSL with DXC, and store versioned cache entries. It should use bounded parallel workers, progress reporting, cancellation, and safe cache writes.

Some shaders and pipeline states may only be discovered after the game begins executing. The runtime therefore needs an on-demand compilation path that shares the same cache format and records misses for later preparation.

DXC compiles HLSL; it does not itself decode Xbox RDNA2 shader machine code. A separate decoder, semantic representation, and translator are required before DXC can be used.

### 3. C++ is an implementation choice, not a shortcut

C++20 is selected for its integration with Win32, Direct3D 12, Qt, and low-level Windows development. Rewriting another project from Rust to C++ does not automatically improve correctness, performance, or security. Every subsystem must be justified by tests and profiling.

### 4. Treat game and shader inputs as untrusted

Parsers for executable images, shaders, archives, textures, and command streams must validate bounds and reject malformed inputs. The runtime should not require administrator privileges. The shader worker should have narrowly scoped access where practical. Crash reporting and external network activity should be disabled by default.

## Proposed technology stack

| Area | Planned technology | Purpose |
|---|---|---|
| Language | C++20 | Launcher and runtime |
| Compiler | MSVC; `clang-cl` in CI | Windows build and additional diagnostics |
| Build | CMake + Ninja | Reproducible project configuration |
| Launcher UI | Qt 6 Widgets | Desktop interface and settings |
| OS integration | Win32 + Windows SDK | Process, memory, file, and synchronization APIs |
| Graphics backend | Direct3D 12 | Host graphics execution |
| Shader compiler | DirectX Shader Compiler (DXC) | Compile generated HLSL to DXIL |
| Tests | CTest + GoogleTest or Catch2 | Unit and integration tests |
| Fuzzing | libFuzzer or equivalent | Parser robustness testing |
| Dependencies | Locked manifest / pinned versions | Dependency review and repeatable builds |

The initial implementation should avoid adding a CPU JIT unless a measured compatibility requirement establishes a need for one. Compatible x64 title code should run natively where possible, with missing Xbox services supplied by the compatibility layer.

## Planned application features

### Launcher

- Game-library view and manual folder selection.
- Read-only file validation and executable/import inspection.
- CPU, GPU, driver, RAM, and adapter capability detection.
- Per-title profiles and supported settings.
- Fast-launch and full-preparation modes.
- Shader and pipeline preparation queue with progress, pause, and cancellation.
- Persistent cache status, cleanup, and repair.
- Runtime logs, compatibility warnings, and local diagnostic export.
- Runtime start, stop, restart, and crash detection.

### Runtime

- PE loader and import routing.
- Xbox kernel and system-service compatibility modules.
- Filesystem overlay with strict path containment.
- Guest memory address, page, and alias model.
- D3D12 resource, descriptor, queue, fence, and barrier translation.
- RDNA2 shader decoder and translation pipeline.
- Audio, input, storage, and presentation compatibility modules as required by the selected title.
- Clear reporting for unsupported APIs and incomplete features.

## Repository structure (proposed)

```text
SeriesBridge/
├── CMakeLists.txt
├── CMakePresets.json
├── README.md
├── LICENSE
├── SECURITY.md
├── CONTRIBUTING.md
├── docs/
│   ├── architecture.md
│   ├── memory-model.md
│   ├── graphics-and-shaders.md
│   ├── security-model.md
│   └── compatibility-matrix.md
├── apps/
│   └── launcher/
├── runtime/
│   ├── loader/
│   ├── services/
│   ├── memory/
│   ├── graphics/
│   ├── shaders/
│   ├── storage/
│   ├── audio/
│   └── input/
├── tools/
│   ├── pe-inspect/
│   ├── shader-inspect/
│   └── cache-inspect/
├── tests/
│   ├── fixtures/
│   ├── unit/
│   ├── integration/
│   ├── graphics/
│   ├── memory/
│   └── fuzz/
├── cmake/
└── .github/
    └── workflows/
```

This is a proposed layout. Directories and build instructions should be updated as implementation is added; this README does not imply that these targets already exist.

## Development roadmap

| Milestone | Focus | Completion evidence |
|---|---|---|
| M0 | Target discovery, legal review, threat model | Exact test target and lawful input requirements documented |
| M1 | CMake workspace, CI, launcher/runtime shells | Both apps build; versioned local IPC works |
| M2 | Read-only PE inspection and minimal loader | Synthetic executable fixtures pass loader tests |
| M3 | Guest memory and host mapping | Aliasing, CPU/GPU visibility, and budget tests pass |
| M4 | D3D12 path and shader translation | Validated shader tests render correctly on supported GPUs |
| M5 | Pre-launch shader/PSO manager | Repeatable cold/warm cache measurements and safe cancellation |
| M6 | Xbox service compatibility and title integration | Measured API coverage and defined title smoke-test checkpoint |
| M7 | Launcher completion and diagnostics | A non-developer can configure, prepare, launch, and diagnose |
| M8 | Security, performance, and release candidate | Audits pass; known limitations and reproducible build instructions are published |

## Testing and performance measurements

Compatibility claims should be based on repeatable tests, not a single successful launch or FPS figure.

Track at least:

- Executable loading and import-resolution coverage.
- Xbox API calls implemented, unsupported, or partially implemented.
- Shader translation correctness and cache hit/miss rates.
- Cold-cache and warm-cache preparation duration.
- Time to first frame and time to interactive menu.
- p50, p95, and p99 frame times over a documented test interval.
- Peak process commit and working set.
- DXGI local and non-local memory usage and budget.
- Guest pages mapped, alias count, backing duplication, bytes transferred, and fence waits.
- GPU device-removal events, runtime crashes, and reproducibility of failures.

Test discrete and UMA GPUs where available. Test multiple vendors and driver versions before making broad compatibility claims. Record title build, hardware, driver, resolution, settings, and cache state alongside every published benchmark.

## Security

SeriesBridge should assume that game files and shader inputs can be malformed or hostile.

Required controls include:

- No administrator requirement for normal launch.
- Strict path canonicalization and overlay containment, including symlink/junction/reparse-point tests.
- Bounds-checked parsers and fuzzing for PE, shader, texture, archive, and command-stream formats.
- Versioned, validated local IPC with restrictive access controls.
- A separate shader-worker process where practical.
- Minimal GitHub Actions permissions and pinned third-party actions.
- Locked and reviewed dependencies.
- Release hashes, signed artifacts, and an SBOM.
- No silent crash-dump upload or telemetry; external diagnostics are opt-in and previewable.
- A public security policy and private vulnerability-reporting route.

**Important runtime limitation:** Running a game executable in the same security context as the user is not sandboxing it. Process separation alone does not make untrusted code safe. Any sandbox claim must be backed by an explicit Windows isolation design and security testing.

See [`SECURITY.md`](SECURITY.md) for the intended disclosure process once that file is created.

## Legal and project integrity

This is an independent compatibility research project. It is not affiliated with or endorsed by Microsoft, Xbox, Rockstar Games, or Take-Two Interactive. Product and game names are used only to identify compatibility research targets.

Do not commit third-party game files, proprietary SDK material, console firmware, private keys, or other restricted content. Review licenses before borrowing or adapting code from existing compatibility projects. In particular, source-code reuse may carry obligations that differ from independently implementing documented behavior.

## Research references

These public sources informed the proposed architecture. Re-check the exact repository revision and documentation before implementation because upstream projects and APIs can change.

- [XCL repository](https://github.com/xcl-dev/xcl)
- [XCL README](https://github.com/xcl-dev/xcl/blob/main/README.md)
- [XCL implementation handoff](https://github.com/xcl-dev/xcl/blob/main/HANDOFF.md)
- [XCL security policy](https://github.com/xcl-dev/xcl/blob/main/SECURITY.md)
- [Xbox Series X specifications](https://www.xbox.com/en-us/consoles/xbox-series-x)
- [Direct3D 12 memory management](https://learn.microsoft.com/en-us/windows/win32/direct3d12/memory-management)
- [Direct3D 12 residency](https://learn.microsoft.com/en-us/windows/win32/direct3d12/residency)
- [DXGI video-memory budget information](https://learn.microsoft.com/en-us/windows/win32/api/dxgi1_4/ns-dxgi1_4-dxgi_query_video_memory_info)
- [Microsoft DirectX specifications: D3D12 Resource Heaps](https://microsoft.github.io/DirectX-Specs/d3d/ResourceHeaps.html)
- [DirectX Shader Compiler (DXC)](https://github.com/microsoft/DirectXShaderCompiler)
- [Qt 6 and CMake](https://doc.qt.io/qt-6/cmake-get-started.html)
- [GitHub Actions security hardening](https://docs.github.com/en/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions)
- [LLVM libFuzzer](https://llvm.org/docs/LibFuzzer.html)
- [MSVC AddressSanitizer](https://learn.microsoft.com/en-us/cpp/sanitizers/asan)

## Related documentation

- [Full development plan](Xbox_Series_to_Windows_Development_Plan.md)

---

**Current next step:** create the C++20/CMake workspace, add CI and the security policy, then implement a synthetic PE-inspection and D3D12 test-renderer vertical slice before attempting title-specific compatibility.
