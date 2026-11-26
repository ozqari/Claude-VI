# Xbox Series X|S → Windows Compatibility Project
## Development Plan: C++ Runtime, Pre-launch GUI, Shader Pipeline, Unified-Memory Model, and Security

**Document status:** Research-based development blueprint  
**Research checked:** 26 November 2026  
**Working project name:** `SeriesBridge` (placeholder; do not use Xbox or Rockstar trademarks as the product name)  
**Primary target:** A legally obtained, user-owned, unpacked Xbox Series X|S GDK game build; GTA VI is the intended target, but target-specific compatibility must be demonstrated rather than assumed.

---

## 1. Executive summary

Build a Windows application consisting of a polished C++ launcher and a separate C++ compatibility runtime. The launcher handles game selection, hardware detection, profiles, settings, diagnostics, shader preparation, cache management, and launch. The runtime loads supported x64 game code, implements the Xbox-facing system interfaces required by the title, translates Xbox RDNA2 GPU programs to a PC graphics representation, and maps the Xbox title-memory model onto the host machine.

The project should **not** begin by assuming that every Xbox instruction must be recompiled to PC CPU instructions. Xbox Series X|S uses an x86-64 CPU environment; a compatibility layer can run compatible x64 title code natively and provide the missing Xbox runtime APIs. The major compatibility work is the executable loader, Xbox system-library behavior, graphics command/resource translation, shader translation, memory semantics, storage, input, audio, and title-specific gaps.

The project should also **not** assume that C++ by itself fixes performance or correctness. A Rust implementation can be technically sound, and a C++ rewrite can reproduce the same problems. C++ is selected here because it is the requested implementation direction and fits the Windows/D3D12 ecosystem; architecture, tests, profiling, and correctness are the deciding factors.

### Core product flow

1. User opens `SeriesBridge Launcher.exe`.
2. Launcher detects the CPU, GPUs, drivers, available RAM, and per-GPU memory budgets.
3. User selects an already-unpacked game folder and an executable/profile.
4. Launcher validates files without modifying the game directory.
5. Launcher scans discoverable shader containers and performs supported offline shader translation/compilation.
6. Launcher displays preparation progress, cache status, warnings, and any unsupported features.
7. Launcher starts the runtime in a separate process.
8. Runtime loads the title, performs remaining on-demand shader/PSO compilation, and writes new cache entries.
9. Launcher remains available for logs, performance metrics, stop/restart, and diagnostics.

**Important:** Not every shader or pipeline can necessarily be known before game execution. The design must support both pre-launch compilation and safe on-demand compilation. Precompilation must not be a promise that all first-run stutter disappears.

---

## 2. Research findings and current-project baseline

This plan was informed by the public XCL repository, its README, handoff notes, security policy, Microsoft Direct3D 12 documentation, the DirectX Shader Compiler project, and Qt documentation. These are useful references, not proof that XCL's implementation is suitable for every title.

### 2.1 What XCL already does

As checked on 26 November 2026, the public XCL README describes an Xbox Series X|S-to-Windows compatibility layer with these design choices:

- No CPU emulation for the title's x64 code; Xbox-facing console libraries are reimplemented.
- Console RDNA2 shader programs are recompiled through an HLSL-to-DXIL route.
- PE loading, kernel/runtime services, file-system overlay, DirectStorage, input, audio, D3D12.X, resource handling, and shader caches are part of the documented architecture.
- Its README describes title graphics memory shared between CPU and GPU without copies, up to a 16 GB title-memory range using 4 GiB windows.
- The README reports translation coverage for 26,698 programs from a retail shader pack, background PSO creation, and disk caching.
- Running `xcl.exe` from Explorer opens a game chooser; it also supports command-line inspect/run operations.

Sources: [XCL README](https://github.com/xcl-dev/xcl/blob/main/README.md), [XCL handoff](https://github.com/xcl-dev/xcl/blob/main/HANDOFF.md).

**Consequence for this project:** a launcher, shader cache, shader precompiler, and unified-memory abstraction are not novel features by themselves. The development objective should be to build a more usable and measurable product, and to solve demonstrated correctness/performance problems—not to claim that current projects have none of these components.

### 2.2 Problems worth designing around

XCL's public handoff notes through 23 November 2026 describe a bring-up of an unnamed newer retail title. The document does not identify that title as GTA VI, so this plan does not treat it as proof of GTA VI compatibility.

The handoff records several useful engineering lessons:

- A memory configuration initially reached about 28.7 GB of process commit and failed during driver pipeline compilation.
- The notes describe duplicated backing for mapped title pages and GPU-memory tile allocations; a later sparse-page approach reduced the observed commitment to roughly 16–17 GB in that test.
- The title reached a first menu at about 30 FPS in a later update, while other sections still list incomplete paths such as GPU-visible memory gaps, newer command queues/lists, MSAA layouts, and other title-specific behavior.
- A shader-heavy load involved roughly 3,500 shaders, including around 2,076 compute programs. Compilation and pipeline creation could dominate startup.
- The handoff measures a performance cost from repeatedly switching memory windows in shader code and discusses a different addressing design as a possible improvement.
- A crash reporter was observed sending minidumps externally; the notes recommend running that test with networking disabled. This project should make diagnostics opt-in and network-disabled by default.

Source: [XCL handoff — retail bring-up, memory and shader notes](https://github.com/xcl-dev/xcl/blob/main/HANDOFF.md).

### 2.3 Security lessons from XCL's policy

XCL's current `SECURITY.md` explicitly says the game runs with the same rights as the user and that XCL is not a sandbox. It also calls out overlay escape, incorrect console-path resolution, malformed shader/archive/texture/PE inputs, and unintended log disclosure as reportable issues. Its policy says only `main` is supported and security reports are sent privately to a maintainer through Discord.

This project should improve on that disclosure and release model: publish a stable supported-release policy, provide a dedicated private vulnerability-reporting channel, document response expectations, and never describe process separation as a sandbox unless operating-system isolation is actually enforced and tested.

Source: [XCL SECURITY.md](https://github.com/xcl-dev/xcl/blob/main/SECURITY.md).

### 2.4 Hardware and API facts that shape the design

- Microsoft's Xbox Series X specification lists 16 GB GDDR6, with 10 GB at 560 GB/s and 6 GB at 336 GB/s. This is one console's physical memory configuration, not a memory limit that should be hard-coded for every PC.
- Microsoft describes D3D12 memory management as a complex problem across UMA and discrete adapters and recommends a “classify, budget and stream” approach.
- Windows exposes GPU memory budgets through DXGI `QueryVideoMemoryInfo`; exceeding a budget can cause stutter or other penalties.
- DXC compiles HLSL to DXIL (and can also emit SPIR-V); it does not, by itself, decode Xbox RDNA2 machine code. The project still needs a console-shader decoder and translator before DXC.
- Qt 6 supports a C++ Widgets application built with CMake, which is suitable for the launcher.

Sources: [Xbox Series X specifications](https://www.xbox.com/en-us/consoles/xbox-series-x), [D3D12 memory management](https://learn.microsoft.com/en-us/windows/win32/direct3d12/memory-management), [DXGI video-memory budget](https://learn.microsoft.com/en-us/windows/win32/api/dxgi1_4/ns-dxgi1_4-dxgi_query_video_memory_info), [DXC](https://github.com/microsoft/DirectXShaderCompiler), [Qt 6 + CMake](https://doc.qt.io/qt-6/cmake-get-started.html).

---

## 3. Product goals and non-goals

### 3.1 Goals

- Provide a clean Windows launcher that opens before the game.
- Use C++20 as the baseline language for the launcher and runtime.
- Support user-selected, already-unpacked Xbox Series X|S GDK title files.
- Run compatible x64 title code natively, without an unnecessary CPU instruction emulator.
- Implement Xbox-facing APIs and system-library behavior incrementally, with explicit compatibility reports.
- Translate supported Xbox RDNA2 shader programs to a host-supported shader representation.
- Precompile discoverable shaders and pipelines before launch, then compile missing items on demand.
- Model guest unified memory correctly while adapting to discrete and integrated PC GPUs.
- Track host memory budgets, resource residency, aliasing, synchronization, and transfer costs.
- Keep the original game directory read-only and write logs, profiles, saves/overlays, and caches to controlled locations.
- Make security, reproducibility, and diagnostics part of the architecture from the beginning.

### 3.2 Non-goals

- Do not decrypt, unpack, or bypass DRM/package protections. The application accepts only files the user has lawfully obtained and made available through an authorized process.
- Do not distribute game executables, game assets, console firmware, private keys, proprietary SDK headers/libraries, or copyrighted shader packs.
- Do not promise every Xbox title will work. Compatibility is title- and version-specific.
- Do not claim that all shaders can be compiled without running the title.
- Do not force all Xbox memory into one PC VRAM allocation or blindly reserve 16 GB at startup.
- Do not patch online services, anti-cheat, entitlement checks, DRM, or server-side game logic.
- Do not treat a launcher GUI as a replacement for a compatibility runtime.
- Do not rewrite a proven subsystem merely because it is currently written in Rust. Reuse or interoperate only after a license and security review.

---

## 4. Recommended technology stack

| Area | Recommendation | Reason / notes |
|---|---|---|
| Main language | C++20 | Mature Windows APIs, RAII, low-level memory control, broad tooling. |
| Compiler | MSVC; add `clang-cl` to CI | MSVC is a practical Windows baseline; a second compiler catches portability and undefined-behavior issues. |
| Build system | CMake + Ninja | Explicit multi-target build, IDE-independent, test and packaging integration. |
| Launcher GUI | Qt 6 Widgets | Native-feeling desktop application, settings pages, progress UI, tables and diagnostics; Qt has documented CMake integration. |
| Runtime platform | Win32 + Windows SDK | Native PE/process, file, synchronization, and D3D12 integration. |
| Graphics backend | Direct3D 12 first | Directly matches the Windows target and allows explicit resource, heap, barrier, and fence management. |
| Shader compiler | DXC (`dxcompiler.dll` / `dxil.dll`) | Compile generated HLSL to DXIL; keep the console-to-HLSL translation as a separate tested subsystem. |
| CPU execution | Native x64 title code + compatibility loader/shims | Avoid building a CPU emulator unless a specific title requirement proves it necessary. |
| Shader translation IR | Small explicit project-owned IR | Separates RDNA2 decoding from HLSL emission and makes instruction-level tests possible. |
| Testing | CTest + GoogleTest/Catch2; GPU integration tests | Keep parsers, memory, shader semantics, command translation, and title tests independently testable. |
| Fuzzing | Clang libFuzzer or equivalent + AddressSanitizer | Fuzz PE, shader, archive, texture, and command-stream parsers. |
| Dependency management | vcpkg manifest or another locked C++ dependency system | Pin and review dependency versions. |
| CI | GitHub Actions with least privilege and pinned action SHAs | Avoid mutable workflow dependencies and unnecessary write permissions. |

### 4.1 Do we need LLVM ORC/JIT?

Not in the initial design. Since the Xbox Series title CPU code is x64, a CPU JIT translator is not the default requirement. LLVM may still be useful for tools or future translation needs, but it should not be added just to make the project appear more advanced. DXC is based on LLVM/Clang and is the shader compiler; the console RDNA2-to-HLSL translation remains project code.

### 4.2 Why use C++ without rewriting everything at once?

Start with a modular C++ codebase, but keep launcher and runtime separate. If a proven external subsystem is reused, review its license and provenance first. XCL is GPL-2.0; copying or adapting its code may impose GPL obligations. A clean-room implementation based on documented behavior and independently written tests is a different engineering/legal decision from copying source code. Obtain legal advice before distribution if code or assets from other projects are incorporated.

---

## 5. High-level architecture

```text
SeriesBridge Launcher (Qt 6 / C++)
  ├── Game Library & File Validator
  ├── Hardware / Driver Detection
  ├── Per-title Settings & Profiles
  ├── Shader / PSO Preparation Manager
  ├── Cache Manager
  ├── Logs / Metrics / Diagnostics
  └── Runtime Process Controller
                │ local authenticated IPC
                ▼
SeriesBridge Runtime (C++ / Win32)
  ├── Title PE Loader + Import Router
  ├── Xbox API Compatibility Layer
  │    ├── Kernel / Memory / Thread services
  │    ├── XGameRuntime-compatible services
  │    ├── Filesystem overlay
  │    ├── DirectStorage compatibility
  │    ├── Input compatibility
  │    └── Audio compatibility
  ├── Guest Memory Manager
  │    ├── Guest virtual address model
  │    ├── Physical page / alias tracking
  │    ├── CPU-visible mappings
  │    ├── GPU-visible resource mappings
  │    └── Residency / budget / eviction policy
  ├── Graphics Compatibility Layer
  │    ├── Xbox D3D12.X API mapping
  │    ├── Command-list translation / replay
  │    ├── Descriptor and resource mapping
  │    ├── Fence / barrier / queue synchronization
  │    └── Direct3D 12 host backend
  ├── RDNA2 Shader Translator
  │    ├── Container decoder
  │    ├── RDNA2 ISA decoder
  │    ├── Semantic IR and validator
  │    ├── HLSL generator
  │    └── DXC integration
  └── Telemetry / Crash Reporting (off by default)
                │
                ▼
Windows / WDDM / GPU driver / DXGI / D3D12 / DXC
```

### Process boundaries

- **Launcher:** UI only; it should not load game modules into itself.
- **Runtime:** separate process that owns the game loader and graphics device.
- **Shader worker:** optional separate worker process for parsing/translation/compilation of untrusted shader inputs. It should have narrowly scoped access to input files and cache directories.
- **Crash reporter:** separate, opt-in component. Never silently upload memory dumps or file paths.

IPC should use a local named pipe or equivalent with a restrictive ACL, protocol versioning, request validation, and no unauthenticated network listener.

---

## 6. Memory architecture: preserve Xbox semantics, adapt to the PC

This is one of the most important systems in the project.

### 6.1 The correct mental model

Do not try to make a PC literally become an Xbox unified-memory machine. Instead, maintain a **guest memory model** that preserves the title's expected address, page, aliasing, visibility, and synchronization behavior, then map it onto whatever host resources are available.

On a discrete PC GPU, CPU system RAM and local GPU VRAM are physically distinct resources. On an integrated/UMA adapter, host and GPU memory may be shared. The implementation must detect the host architecture and choose the correct mapping strategy.

### 6.2 Required subsystems

1. **Guest address space** — represent the title's virtual/physical address ranges separately from host pointers and D3D12 resource objects.
2. **Page allocator** — track guest page IDs, page size, allocation owner, permissions, mappings, aliases, and lifetime.
3. **Backing store** — assign each guest page to a host backing type: CPU-visible, GPU-local, shared/UMA, sparse/tiled, or staged.
4. **Alias map** — allow multiple guest virtual/resource views to refer to the same guest backing pages where Xbox semantics require it.
5. **Visibility/coherency tracker** — track CPU writes, GPU writes, dirty ranges, and visibility transitions.
6. **GPU mapping manager** — create/update D3D12 resources and heap/tile mappings; serialize mapping updates against GPU queue use with fences.
7. **Budget manager** — query local and non-local DXGI budgets; track system commit and process usage; react to budget-change notifications.
8. **Eviction/streaming policy** — preserve resources needed by in-flight work and evict only safe, recreatable, or reloadable contents.
9. **Diagnostics** — report guest bytes allocated, host bytes committed, GPU-local usage, non-local usage, duplicated backing, alias counts, page faults/invalid accesses, transfer bytes, and fence waits.

### 6.3 Allocation strategy

- Maintain a logical Xbox title-memory address space; do not eagerly commit the full console-sized range.
- Allocate host backing lazily when guest pages are committed or first used.
- Prefer shared or zero-copy mappings only when the selected adapter and D3D12 resource type genuinely support them and correctness tests pass.
- For a discrete GPU, use GPU-local memory for GPU-dominant resources when beneficial; use upload/readback or staging paths only where required; avoid keeping redundant copies without a clear coherence policy.
- For UMA adapters, exploit shared host/GPU memory where beneficial, but still monitor system-memory pressure.
- Preserve Xbox aliasing behavior explicitly; two resources that alias the same guest pages must not silently become unrelated host allocations.
- Handle sparse/reserved/tiled resource mappings where they provide a correct and efficient fit. Do not assume every resource format or heap class supports the same mapping path.
- Never assume a 16 GB host VRAM card is equivalent to the Xbox's 16 GB shared pool. The Xbox bandwidth split, OS reservation, and host GPU/driver budgets differ.

### 6.4 Required memory tests

- CPU writes → GPU reads and GPU writes → CPU reads.
- Aliasing: two resources map the same guest pages and observe writes correctly.
- Page remapping while command queues are in flight.
- 4 KiB and 64 KiB alignment, boundary crossing, and large sparse ranges.
- Resource destruction while references or fences remain outstanding.
- Out-of-memory and budget-reduction scenarios.
- UMA integrated GPU and discrete AMD/NVIDIA/Intel GPU test matrix.
- Stress test with shader precompilation running concurrently with title loading.

### 6.5 Performance acceptance criteria

Record a reproducible baseline before optimizing. Track:

- Peak process commit and peak host RAM.
- Peak local/non-local GPU-memory usage and budget percentage.
- Guest-memory bytes vs host backing bytes (duplication ratio).
- CPU↔GPU transfer bytes per frame and per minute.
- Time spent waiting on page-map and resource fences.
- Number of allocations, deallocations, remaps, evictions, and failed allocations.
- Frame-time percentiles (p50/p95/p99), not just average FPS.

The acceptance target is **correct behavior within the current host budgets**, not an arbitrary “zero-copy everywhere” claim.

---

## 7. Shader and pipeline preparation

### 7.1 Pipeline

```text
Xbox shader container / RDNA2 program
              │
              ▼
Container parser + bounds validation
              │
              ▼
RDNA2 decoder / disassembler
              │
              ▼
Semantic shader IR
              │
              ▼
HLSL generation
              │
              ▼
DXC compilation
              │
              ▼
DXIL object + validation metadata
              │
              ▼
Host GPU driver pipeline creation (when full PSO data is available)
              │
              ▼
Versioned disk cache
```

DXC is the HLSL compiler, not the Xbox RDNA2 decoder. Keep shader decoding, semantic translation, HLSL emission, DXC compilation, and host pipeline-state creation as separate stages with separate diagnostics.

### 7.2 Pre-launch shader workflow

The launcher should provide a **Prepare Shaders** step:

1. Scan only the selected game directory and explicitly approved shader packs.
2. Parse containers with strict size/count/offset limits.
3. Identify each shader's stage, ISA, hash, and required features.
4. Translate supported RDNA2 programs into HLSL.
5. Compile HLSL to DXIL using the installed/supported DXC version.
6. Build PSOs only when the required root signature, formats, blend/depth/raster state, descriptors, and pipeline stream are known.
7. Store each successful stage separately so a DXC failure does not require redoing decoding and translation.
8. Display skipped/unsupported entries and reasons.
9. Allow pause/cancel and configurable CPU/GPU concurrency.
10. Launch the runtime when the user chooses; remaining items compile on demand.

### 7.3 Cache key and invalidation

Cache identity must include at least:

- Original console shader byte hash.
- Translator version and IR schema version.
- HLSL emitter version.
- DXC compiler version/build.
- Shader target/model and compilation options.
- Host GPU vendor/device/architecture and relevant driver identity for driver-specific pipeline cache entries.
- Relevant pipeline state and root-signature identity for PSO entries.

Use content-addressed, atomic writes. Never overwrite a valid cache entry until the new entry has passed validation. Add a “clear shader cache” action and show cache size, last-use date, and invalidation reason.

### 7.4 Correctness requirements

- Per-instruction semantic tests for decoded instructions.
- Differential tests against a trusted reference where legally available.
- Wave-size tests, including Wave64 behavior on Wave32 GPUs and partial-wave edge cases.
- Tests for LDS/shared memory, atomics, image formats, samplers, texture swizzles, barriers, and control flow.
- Shader compiler error reports include shader hash and stage, never game secrets or full proprietary data by default.
- Fuzz every shader-container parser and decoder.

### 7.5 Startup and stutter goals

- Show progress by stage: scan, decode, translate, DXC compile, PSO build, ready.
- Schedule heavy compilation in a bounded worker pool.
- Offer “fast launch” (compile only required/known shaders) and “full preparation” modes.
- Support runtime background compilation with a safe placeholder/fallback only when semantically correct; never render a wrong shader as a fake success.
- Measure cold-cache and warm-cache startup separately.

---

## 8. Launcher specification

### 8.1 Pages

| Page | Main functions |
|---|---|
| Home | Selected title, Play, Prepare, last session, compatibility status, recent logs. |
| Game Library | Add/remove path references, scan, executable selection, file validation. Never modify title files. |
| Graphics | GPU selection, resolution, window mode, frame pacing, VSync, supported scaling/upscaling options. Show unsupported settings as disabled with explanations. |
| Memory | Host RAM, local/non-local GPU budget, guest committed memory, cache/mapping usage, warnings. Advanced controls behind an expert toggle. |
| Shader Preparation | Scan status, discovered shader count, compiled count, failed/unsupported count, cache size, worker limit, pause/cancel. |
| Compatibility | Implemented, partial, stubbed, and unsupported APIs/features for the selected title. |
| Controls & Audio | Gamepad, keyboard/mouse mapping, output device, audio diagnostics. |
| Profiles | Per-title settings, graphics profiles, cache profiles, reset-to-default. |
| Logs & Diagnostics | Searchable logs, session report, copy diagnostics, opt-in crash dump. |

### 8.2 Play button flow

- Validate title path and selected executable.
- Check supported OS, DX12 feature level, shader model, required driver features, and available memory budgets.
- Warn about missing runtime dependencies; never silently download unknown DLLs.
- Verify cache compatibility; ask whether to rebuild stale entries.
- Run preflight and shader preparation according to the selected launch mode.
- Start runtime as a child process with a validated command line and restricted environment.
- Stream status and structured log events back to launcher.
- On exit, show exit code, session duration, frame-time summary if available, and a safe diagnostic bundle preview.

### 8.3 Settings storage

- Store app settings under the user's local application data directory.
- Use versioned JSON or another documented format.
- Validate settings before use; reject unknown types and out-of-range numeric values.
- Never accept arbitrary command fragments from a profile.
- Treat game paths and shader-cache paths as untrusted user input; canonicalize paths and prevent traversal.

---

## 9. Compatibility layer modules

Implement one subsystem at a time. Every imported API should be classified as `implemented`, `partial`, `stubbed`, `unsupported`, or `not observed` for each target build.

### 9.1 PE loader and import router

- Parse x64 PE headers, sections, relocations, imports, delay imports, TLS, exception/unwind metadata, and exports.
- Validate all offsets, lengths, alignment, arithmetic, and section overlap before mapping.
- Resolve imports to project-owned compatibility shims, safe Windows APIs, or explicit unsupported stubs.
- Do not silently return success for an unsupported API if that would alter game state. Log and report the call with enough context for reproduction.
- Keep loader tests independent of the full game runtime.

### 9.2 Xbox system services

Prioritize by actual import/call telemetry from the target:

1. Kernel, virtual memory, thread, synchronization, module, and timing behavior.
2. XGameRuntime and title identity/session/save behavior needed by the offline title path.
3. File-system path mapping, persistent/local storage, temporary storage, and write overlay.
4. DirectStorage queues, decompression, alignment, and error semantics.
5. Input and controller mapping.
6. Audio services and codec handling.
7. Network-facing APIs only where lawful and required; default network behavior should be disabled until explicitly configured.

Do not recreate proprietary online services or bypass entitlements, DRM, anti-cheat, or authentication.

### 9.3 Graphics API compatibility

- Map resource creation, heap placement, descriptor heaps, command lists, draws, dispatches, copies, clears, barriers, fences, and presentation.
- Keep console API structures separate from D3D12 host structures; translate explicitly.
- Track console GPU virtual addresses in a table rather than treating them as host pointers.
- Make unsupported command IDs fail clearly and report once per callsite/hash where possible.
- Validate queue ordering, command allocator/list reset-close-execute semantics, and fence values.

---

## 10. Security and software-supply-chain plan

Security is a release gate, not a later polish task.

### 10.1 Runtime threat model

Inputs that may be malformed or hostile include executable images, DLLs, shader packs, texture containers, archive structures, pipeline descriptors, file paths, settings, profile files, and IPC messages.

Controls:

- Run without administrator rights.
- Keep launcher and runtime in separate processes.
- Do not treat separate processes as a sandbox by themselves.
- Use Windows process mitigations where compatible (DEP, ASLR, CFG and suitable image-load/dynamic-code policies); test the real game path before enabling restrictive options by default.
- Consider a restricted token or AppContainer for the shader worker, with read-only input and write access only to a temporary work/cache directory. Do not promise full-game sandboxing unless all required capabilities are explicitly granted and isolation is independently tested.
- No network access by default for title execution, shader compilation, crash reporting, or telemetry. If a feature needs network access, expose it clearly and make it opt-in.
- Crash dumps are local and opt-in; show exactly what will be collected before export or upload.
- Keep game files read-only and route title writes through a validated overlay.
- Canonicalize paths and reject path traversal, device-path surprises, junction/reparse-point escapes, and overlay escape attempts.
- Redact usernames, full local paths, tokens, account IDs, and other personal data from logs by default.

### 10.2 Parser and memory safety

- Fuzz PE, shader, texture, archive, pipeline-stream, and settings parsers.
- Use checked arithmetic for all sizes/offsets and hard limits for counts, recursion, allocations, and decompression ratios.
- Prefer span-based parsing and explicit endian/layout conversions.
- Avoid raw pointer arithmetic where checked offsets or spans work.
- Use RAII for handles, COM objects, mapped resources, fences, and locks.
- Run ASan/UBSan-compatible test configurations where supported; use static analysis and compiler warnings as errors for owned code.
- Maintain regression corpora for every parser crash and malformed-input bug.

### 10.3 Supply-chain controls

- Pin third-party dependencies to reviewed versions and maintain a lockfile/manifest.
- Pin GitHub Actions to full commit SHAs; keep workflow permissions least-privilege.
- Protect `main` and release branches; require reviewed pull requests for loader, memory, shader, and CI workflow changes.
- Require multi-factor authentication for maintainers and separate source-code access from release-signing access.
- Do not expose release secrets to untrusted pull-request workflows.
- Generate an SBOM for every release.
- Sign release binaries and publish SHA-256 hashes and build provenance/attestations.
- Build releases from tagged commits in a clean environment; preserve the exact source commit, compiler versions, dependency lock, and build logs.
- If reproducible builds are not yet achievable, document the non-reproducible steps and make binary provenance verifiable.

Sources: [GitHub Actions secure use](https://docs.github.com/en/actions/reference/security/secure-use), [Windows process mitigation policies](https://learn.microsoft.com/en-us/windows/win32/api/winnt/ne-winnt-process_mitigation_policy), [libFuzzer](https://llvm.org/docs/LibFuzzer.html), [MSVC AddressSanitizer](https://learn.microsoft.com/en-us/cpp/build/reference/fsanitize?view=msvc-160).

### 10.4 Responsible disclosure

Publish `SECURITY.md` with:

- Supported release branches and versions.
- Private GitHub vulnerability reporting or a dedicated security email.
- Expected acknowledgement and remediation targets.
- Safe-harbor language for good-faith research, reviewed by counsel.
- Instructions not to upload game executables, copyrighted assets, account tokens, or private dumps.
- A clear policy for when an issue becomes public.

---

## 11. Repository structure

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
│   ├── threat-model.md
│   ├── memory-model.md
│   ├── shader-pipeline.md
│   ├── compatibility-matrix.md
│   ├── test-plan.md
│   ├── release-process.md
│   └── decisions/
├── apps/
│   ├── launcher/
│   │   ├── CMakeLists.txt
│   │   └── src/
│   ├── runtime/
│   │   ├── CMakeLists.txt
│   │   └── src/
│   └── shader_worker/
│       ├── CMakeLists.txt
│       └── src/
├── libraries/
│   ├── common/           # Result types, logging, config, IPC protocol
│   ├── pe_loader/        # Checked PE parser and loader
│   ├── xbox_abi/         # Console ABI structures and calling conventions
│   ├── hle/              # High-level emulation/system API shims
│   ├── guest_memory/     # Guest VA, pages, aliasing, mappings, budgets
│   ├── graphics/         # D3D12.X compatibility and host D3D12 backend
│   ├── rdna2/            # ISA decoder, disassembler, semantic validation
│   ├── shader_ir/        # Intermediate representation and optimizer
│   ├── hlsl_emitter/     # IR -> HLSL
│   ├── shader_cache/     # Content-addressed shader and PSO cache
│   ├── storage/          # Filesystem overlay and DirectStorage mapping
│   ├── input/
│   └── audio/
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── gpu/
│   ├── fuzz/
│   ├── fixtures/         # Synthetic/non-copyrighted fixtures only
│   └── performance/
├── tools/
│   ├── inspect/          # Read-only executable/package inspection
│   ├── shader_dump/      # User-invoked diagnostic tool
│   └── benchmark/
├── scripts/
└── .github/
    └── workflows/
```

Do not commit game executables, game assets, private shader dumps, console keys, proprietary SDK material, or personal crash dumps. Keep test fixtures synthetic or independently created unless distribution rights are clear.

---

## 12. Milestones and exit criteria

The work should be milestone-driven. A milestone is complete only when its tests and acceptance criteria pass; reaching a menu is not equivalent to full compatibility.

### M0 — Legal, technical, and target discovery

**Tasks**
- Confirm the exact Xbox generation, title version, executable format, package state, and legally authorized input files.
- Build a feature inventory from imports, API calls, shader containers, graphics command identifiers, memory allocations, and storage usage.
- Establish the supported Windows version and minimum GPU feature set.
- Review licenses and provenance of every library and reference implementation.

**Exit criteria**
- Written target manifest and legal-use boundary.
- No dependency on DRM circumvention or proprietary files that cannot be redistributed/used lawfully.
- Initial API/feature matrix with unknowns clearly marked.

### M1 — Build system and application shell

**Tasks**
- Create the C++20 CMake workspace and CI builds.
- Build a Qt launcher shell with Home, Library, Settings, Shader Preparation, Logs, and Diagnostics pages.
- Add structured logging, configuration versioning, local IPC, and process supervision.
- Add a read-only game-folder scanner and metadata validator.

**Exit criteria**
- Clean build on a supported Windows 11 development machine.
- Launcher can add/remove a path, validate a synthetic test title, save profiles, and start/stop a dummy runtime process.
- No game folder modification during scanning.

### M2 — PE loader and minimal native runtime

**Tasks**
- Implement checked x64 PE parsing, relocation, import resolution, TLS, and exception metadata handling.
- Add an import router with explicit HLE/Windows/stub/unsupported outcomes.
- Implement basic logging and API-call telemetry.
- Create synthetic PE tests and malformed-input fuzz targets.

**Exit criteria**
- Synthetic PE corpus passes all loader tests.
- Fuzzing finds no unresolved crashes in the release parser path.
- A small lawful test executable can be loaded and call a test compatibility DLL.

### M3 — Guest memory and host memory manager

**Tasks**
- Implement guest address space, page IDs, mapping tables, alias tracking, and protection rules.
- Add D3D12 adapter discovery and UMA/discrete detection.
- Implement host backing classes and lazy allocation.
- Add local/non-local GPU budget monitoring and eviction/streaming policy.
- Add mapping fences and queue synchronization.

**Exit criteria**
- CPU/GPU visibility, aliasing, page remapping, and synchronization tests pass.
- No eager allocation of the complete guest memory range.
- Memory accounting explains every large host allocation and reports duplication ratio.
- Stress tests survive changing GPU budgets and out-of-memory errors without corrupting guest mappings.

### M4 — Graphics compatibility and shader translator

**Tasks**
- Implement resource/heap/descriptor abstractions and a minimal graphics command path.
- Implement RDNA2 container parsing and instruction decoding.
- Define and validate a shader IR.
- Emit HLSL and integrate DXC.
- Add versioned shader cache and parallel compilation.
- Add Wave64/Wave32 and partial-wave semantic tests.

**Exit criteria**
- Reference shader test suite passes on at least one AMD and one non-AMD GPU.
- Cache invalidation works when translator, DXC, or driver identity changes.
- Unsupported instructions fail clearly; they are not silently replaced with incorrect results.

### M5 — Pre-launch shader and PSO manager

**Tasks**
- Implement scan → decode → translate → DXC → PSO stages.
- Add bounded worker pool, pause/cancel, progress reporting, and compile logs.
- Add “fast launch” and “full preparation” modes.
- Add background on-demand compile in the runtime.

**Exit criteria**
- A supported test title can be prepared before launch.
- The launcher accurately distinguishes compiled shaders from fully buildable PSOs.
- Cold and warm cache timings are recorded and repeatable.
- A failed/cancelled compile never corrupts existing cache entries.

### M6 — Xbox system APIs and title integration

**Tasks**
- Implement the Xbox APIs observed in the target title, beginning with kernel, memory, filesystem, storage, runtime, input, and audio dependencies.
- Implement D3D12.X command queues/lists, resource barriers, fences, descriptors, and presentation.
- Add per-title compatibility configuration and missing-call reports.

**Exit criteria**
- The target reaches a defined smoke-test checkpoint (for example, loader, splash screen, menu, and one controlled gameplay sequence) without unsupported calls silently succeeding.
- Filesystem writes remain inside the overlay.
- Input, audio, frame pacing, and device-reset handling pass their tests.

### M7 — Launcher polish and user-facing diagnostics

**Tasks**
- Finish settings/profile pages, GPU selection, graphics options, log viewer, cache controls, and error explanations.
- Add a preflight report and one-click local diagnostic export with a preview/redaction step.
- Add graceful runtime shutdown, crash detection, restart, and cache repair.

**Exit criteria**
- A non-developer can select the title, prepare it, launch it, change supported settings, and understand common errors without using command-line tools.
- The launcher never claims a setting or feature works when the runtime marks it unsupported.

### M8 — Security review, performance tuning, and release candidate

**Tasks**
- Run parser fuzzing, static analysis, sanitizer builds, and dependency audits.
- Audit path handling, overlay boundaries, IPC, crash reporting, network behavior, and release workflows.
- Test AMD, NVIDIA, and Intel adapters where available, plus UMA hardware.
- Profile CPU cost, GPU time, memory overhead, and shader/PSO startup time.
- Generate SBOM, sign artifacts, verify hashes, and publish reproducible build instructions.

**Exit criteria**
- No known critical/high security issue remains unresolved.
- Security policy and private reporting channel are public.
- Release candidate passes the compatibility, memory, shader, and security test suites.
- Known limitations are published honestly.

---

## 13. Test strategy and success metrics

### Functional

- Loader import resolution and exception/unwind correctness.
- API-call compatibility coverage by title.
- Filesystem overlay containment and persistence.
- Input, audio, storage, rendering, command-list, and presentation tests.
- Save/config preservation across clean launch, crash, and restart.

### Graphics

- Instruction-level shader tests.
- Pixel/compute output comparisons against known references.
- Texture swizzle, mip-tail, array, cube, 3D, and depth-stencil layout tests.
- Resource barrier, descriptor lifetime, queue fence, and pipeline state tests.
- Test both warm and cold pipeline caches.

### Memory

- Guest-to-host allocation accounting.
- Alias correctness and page-map lifetime.
- UMA and discrete adapter paths.
- GPU budget reduction, eviction, device removal, and allocation failure.
- Peak commit, working set, local/non-local budget, and transfer-byte tracking.

### Security

- Fuzz corpus for every parser.
- Path traversal, symlink/junction/reparse-point and overlay escape tests.
- Crafted IPC messages and malformed settings.
- No automatic network requests during normal local launch.
- Crash-dump content review and opt-in test.
- Clean build from a tagged commit and release hash verification.

### Product metrics

Track per title and per hardware configuration:

- Launch success rate and failure reason.
- Time to first frame and time to interactive menu.
- Cold-cache and warm-cache shader preparation duration.
- p50/p95/p99 frame time and sustained frame rate.
- Peak process commit, peak system RAM, local/non-local GPU use.
- Shader compile failures, unsupported API counts, device-removal events.
- Runtime crash rate and reproducibility of reported failures.

Do not publish a single FPS number without the title build, hardware, driver, resolution, settings, cache state, and test duration.

---

## 14. Main risks and mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Incomplete Xbox API behavior | Game stops at startup/menu or later mission | API telemetry, feature matrix, deterministic tests, implement by observed call path. |
| Incorrect guest memory aliasing/coherency | Corruption, visual defects, crashes | Explicit page/alias model, fences, differential tests, memory accounting. |
| Host VRAM budget exceeded | Stutter, device removal, system commit growth | Lazy mapping, DXGI budget monitoring, streaming/eviction, limited workers. |
| Shader translation mismatch | Black screen, wrong rendering, GPU hangs | Instruction-level tests, semantic IR, differential GPU tests, explicit unsupported status. |
| First-run compilation time | Long startup and stutter | Pre-launch scan, bounded parallel precompile, persistent cache, on-demand compilation. |
| Driver differences | AMD/NVIDIA/Intel behavior diverges | Multi-vendor tests, capability detection, conservative fallbacks. |
| Malformed game/shader input | Parser crash or memory corruption | Fuzzing, strict bounds, process isolation for workers, no-admin runtime. |
| Compromised dependency or CI action | Supply-chain compromise | Pinned dependencies/actions, least-privilege workflows, signed builds, SBOM, protected branches. |
| Online/DRM/anti-cheat dependency | Incompatibility or legal/security risk | Do not bypass; mark unsupported unless there is a lawful, documented path. |
| Scope becomes too large | No working release | Vertical slices, one known test title first, milestones with hard exit criteria. |

---

## 15. First practical implementation slice

The first working deliverable should not attempt to boot GTA VI immediately. Build a small end-to-end vertical slice using synthetic fixtures and a legally usable test title:

1. Launcher opens and adds a game folder.
2. Read-only scanner validates a test executable and reports imports.
3. Runtime process launches with a minimal PE loader.
4. Guest memory module allocates and maps test pages.
5. A synthetic RDNA2-inspired shader test passes through the translator/HLSL/DXC cache pipeline.
6. Launcher shows shader preparation progress and cache results.
7. Runtime renders a simple test frame through D3D12.
8. Memory and shader metrics are recorded in a session report.
9. Fuzz and sanitizer CI jobs run for all parsers.

Once this slice works, add actual Xbox structures and APIs incrementally. This avoids debugging a loader, a memory model, a shader translator, a graphics backend, and a title-specific issue simultaneously.

---

## 16. Recommended immediate backlog

Priority order:

- [ ] Write a target manifest for the exact Xbox Series game build and lawful input format.
- [ ] Create CMake C++20 workspace, CI, and `SECURITY.md` before adding complex parsers.
- [ ] Create separate launcher and runtime executables with versioned local IPC.
- [ ] Build read-only PE inspection and synthetic PE test fixtures.
- [ ] Define the guest memory address/page/alias model on paper and in unit tests before writing GPU allocation code.
- [ ] Implement host adapter detection, UMA/discrete paths, DXGI budget polling, and memory telemetry.
- [ ] Implement a minimal shader parser → IR → HLSL → DXC → disk-cache pipeline.
- [ ] Add shader-preparation UI with bounded concurrency and cancellation.
- [ ] Add a D3D12 test renderer and GPU integration tests.
- [ ] Add Xbox API shims only from a measured target import/call inventory.
- [ ] Add fuzzers for PE/shader/texture/archive parsers and path-overlay tests.
- [ ] Run multi-vendor GPU validation before claiming broad compatibility.

---

## 17. Research references

The following sources were consulted on 26 November 2026. Repository pages can change; re-check the commit and documentation before implementing against them.

1. [XCL — public repository and architecture](https://github.com/xcl-dev/xcl)
2. [XCL README — supported features, memory, graphics, shader compiler, and launch flow](https://github.com/xcl-dev/xcl/blob/main/README.md)
3. [XCL HANDOFF — implementation notes and measurements](https://github.com/xcl-dev/xcl/blob/main/HANDOFF.md)
4. [XCL SECURITY.md — threat assumptions and disclosure policy](https://github.com/xcl-dev/xcl/blob/main/SECURITY.md)
5. [Xbox Series X technical specifications](https://www.xbox.com/en-us/consoles/xbox-series-x)
6. [Microsoft Learn — Direct3D 12 memory management](https://learn.microsoft.com/en-us/windows/win32/direct3d12/memory-management)
7. [Microsoft Learn — D3D12 residency](https://learn.microsoft.com/en-us/windows/win32/direct3d12/residency)
8. [Microsoft Learn — DXGI_QUERY_VIDEO_MEMORY_INFO](https://learn.microsoft.com/en-us/windows/win32/api/dxgi1_4/ns-dxgi1_4-dxgi_query_video_memory_info)
9. [Microsoft DirectX Specs — D3D12 Resource Heaps](https://microsoft.github.io/DirectX-Specs/d3d/ResourceHeaps.html)
10. [Microsoft DirectX Shader Compiler (DXC)](https://github.com/microsoft/DirectXShaderCompiler)
11. [Qt 6 — Getting started with CMake and Widgets](https://doc.qt.io/qt-6/cmake-get-started.html)
12. [GitHub Docs — Secure use of GitHub Actions](https://docs.github.com/en/actions/reference/security/secure-use)
13. [LLVM — libFuzzer](https://llvm.org/docs/LibFuzzer.html)
14. [Microsoft Learn — MSVC AddressSanitizer](https://learn.microsoft.com/en-us/cpp/build/reference/fsanitize?view=msvc-160)
15. [Microsoft Learn — Windows process mitigation policies](https://learn.microsoft.com/en-us/windows/win32/api/winnt/ne-winnt-process_mitigation_policy)

---

## Final recommendation

Build the launcher in C++ with Qt 6 and build the Windows runtime as a modular C++20 project using Win32 and Direct3D 12. Keep the launcher separate from the game process. Treat the Xbox's unified memory as a guest semantic model, not as a physical layout that can be copied directly to every PC. Make shader precompilation a staged, cacheable preparation step, with runtime compilation for resources that are only discovered during execution. Make security and reproducible releases part of the first milestone.

Most importantly, do not sell this as an easy conversion of an Xbox executable into a Windows executable. The CPU side may execute natively, but the compatibility layer still has to implement the Xbox environment the title expects. A reliable first-frame/menu milestone, measured memory behavior, correct rendering, and controlled gameplay tests are the evidence that the port is progressing—not the existence of a launcher or a successful executable load.
