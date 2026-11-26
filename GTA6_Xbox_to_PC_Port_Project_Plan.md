# GTA VI Xbox Series X|S → Windows PC Compatibility Project

> **Document type:** Agent execution brief / technical project plan  
> **Prepared:** 2026-11-24  
> **Primary target:** Windows 10/11 x64  
> **Initial strategy:** Audit and extend the most advanced maintainable existing implementation; do not rewrite the entire port from scratch.  
> **Preferred starting baseline:** [Marvisak/VIctory](https://github.com/Marvisak/VIctory), subject to the mandatory repository and toolchain audit in Phase 0.  
> **Reference implementations:** [ProjectLeonidaL/leonida (Leonida Legacy)](https://github.com/ProjectLeonidaL/leonida), [sammwyy/xvalkyrie](https://github.com/sammwyy/xvalkyrie), and [pixl27/ProjectReCity](https://github.com/pixl27/ProjectReCity).  
> **Project status:** Research and engineering effort. This document does not represent a working port or promise that GTA VI is currently playable on PC.

---

## 0. Instructions to the AI agent

You are the engineering agent assigned to investigate, build, test, and incrementally improve a local Windows compatibility implementation for the Xbox Series X|S version of Grand Theft Auto VI.

Your immediate assignment is **not** to claim that you can port the whole game in one step. Your assignment is to establish a reproducible development environment, audit the current public implementations, choose one codebase as the foundation, run its tests, reproduce the earliest supported milestones when legally available input files exist, and then fix the highest-priority verified blocker with a small tested change.

### Mandatory first actions

1. Inspect the current working directory, operating system, CPU, GPU, RAM, available disk space, installed SDKs, compiler toolchains, and Git state.
2. Determine whether the current workspace already contains a repository or user modifications. **Never overwrite or reset an existing workspace.**
3. Read this entire document before editing source code.
4. Re-check the current upstream repositories and their latest commit hashes. The repository status in this document is a dated snapshot, not a permanent truth.
5. Compare the available codebases using the criteria in Section 5. Prefer a small, evidence-based decision over combining multiple large codebases.
6. Create a baseline report containing repository URL, commit SHA, branch, license, dependencies, build commands, test results, GPU capabilities, and known blockers.
7. Run tests that do not require game files before attempting any game-specific run.
8. Start implementation only after the baseline builds or after a specific, reproducible build blocker has been isolated.

### What counts as progress

Valid progress is a code change accompanied by a test, a reproducible diagnostic, a documented blocker, or a verified improvement. A successful compilation alone is not evidence that the game works. A shader translating to an intermediate representation is not proof that it renders correctly. A menu appearing is not proof that gameplay works.

### Do not do these things

- Do not invent progress, hardware compatibility, test results, performance figures, API coverage, or game access.
- Do not assume an executable, key, package, or decrypted file exists in the workspace.
- Do not search random folders, Downloads directories, other drives, or cloud accounts for game content without explicit configuration from the user.
- Do not acquire, distribute, upload, or commit proprietary game binaries, keys, firmware, extracted game assets, generated game shaders, save files, memory dumps, or other copyrighted game content.
- Do not attempt to decrypt protected game packages, defeat DRM, bypass console security, extract keys, exploit a console, or circumvent access controls. This project must consume only files the user is legally entitled to use and has independently made lawfully readable.
- Do not add online functionality, Rockstar account integration, matchmaking, anti-cheat bypasses, or access to official online services. The initial runtime must remain offline.
- Do not copy code from another repository until its license, attribution requirements, provenance, and compatibility with the selected base have been reviewed.
- Do not run untrusted binaries, installers, scripts, or build outputs from unofficial sources without inspection.
- Do not disable security software or request administrator privileges unless a documented, narrowly scoped requirement makes it unavoidable. Prefer a normal user process.
- Do not publish a fork, create releases, upload artifacts, or open public issues/PRs without the user's explicit authorization.
- Do not replace, remove, reformat, or mass-edit files unrelated to the task.

### Expected outputs from the first working session

- `docs/BASELINE_AUDIT.md` — evidence-based review of candidate repositories and chosen base.
- `docs/ENVIRONMENT_REPORT.md` — machine/toolchain capability report, without personally identifying paths where unnecessary.
- `docs/BUILD_AND_TEST_LOG.md` — exact commands, exit codes, and test outcomes.
- A clean local branch or worktree for project changes.
- One small, completed engineering task, or a precise blocker report if no safe/valid first task can be completed.

---

## 1. Project purpose and definition of success

### 1.1 Purpose

Develop a local Windows compatibility runtime that can execute the Xbox Series X|S version of GTA VI using the user's own legally usable local game files. The runtime should provide the platform behavior expected by the title while translating console graphics behavior to a supported Windows graphics API.

The Xbox Series CPU and desktop PC CPUs both use the x86-64 instruction-set family. This makes native execution of much of the title's CPU code a practical starting point. It does **not** make an Xbox executable automatically compatible with Windows: executable loading, imports, thread-local storage, memory behavior, system APIs, graphics APIs, resource layouts, GPU synchronization, and platform services still need to be handled.

### 1.2 Main objective

Reach the following milestones in order:

1. Reproducible build and passing tests without game data.
2. Correct parsing and inspection of supported, already-readable local package files.
3. Stand-in executable tests proving loader and HLE behavior.
4. Repeatable execution of the title through the first-boot/menu stages, when a lawful supported local executable is available.
5. Successful world load with meaningful scene rendering.
6. Stable, controllable gameplay in an offline local session.
7. Improve compatibility, image correctness, frame pacing, performance, and hardware coverage.

### 1.3 Success is not

- Merely opening the game's executable in a disassembler.
- Exporting some models or textures.
- Translating all shader binaries without validating their behavior on a GPU.
- Showing a title card or a menu while the world is black.
- Reaching a game loading screen and then crashing.
- Claiming compatibility based on a single screenshot or unrepeatable video.
- Building a general Xbox emulator before proving it is needed.

### 1.4 Non-goals for the first release

- Xbox firmware emulation or booting the Xbox operating system.
- CPU emulation unless a verified instruction or execution-state incompatibility requires it.
- Xbox Live, Rockstar Online, multiplayer, cloud saves, achievements, or network account services.
- Anti-cheat interaction or bypass.
- A general solution for every Xbox Series title.
- A complete cross-platform runtime for Linux/macOS before Windows is stable.
- A custom game engine or reimplementation of GTA VI's gameplay systems.
- Supporting every Xbox package version immediately. Support one verified version first; expand only after regression tests exist.

---

## 2. Verified project landscape at the time this document was prepared

The following is a **snapshot of public repository claims checked on 2026-11-24**. Re-check the current README, code, issues, releases, CI, and commit history before relying on any item.

| Project | Publicly described architecture | Reported state in the checked material | Intended role in this project |
|---|---|---|---|
| [VIctory](https://github.com/Marvisak/VIctory) | Rust workspace; maps and runs the title's x86-64 code in-process; implements HLE/platform services; uses host Windows D3D12 and translates shaders at first use | README dated 2026-11-23 reports first-boot/menu stages and world loading; world scene is still mostly dark/incorrect and very slow; warns that a real GPU may hang/reset its driver | **Preferred initial base** because its reported execution path is furthest along. Verify the exact commit and reproduce the result. |
| [Leonida Legacy](https://github.com/ProjectLeonidaL/leonida) | Python tooling plus native runtime; shader/asset tooling; RDNA2 shader translation to SPIR-V; Vulkan-backed Xbox D3D12 compatibility work | README says the game does not yet boot; shader and asset tooling are more advanced than the title boot path; Xbox D3D12-to-Vulkan remains incomplete | Reference for package/version checks, RPF9/texture/audio tooling, RDNA2 shader analysis, Vulkan GPU harness, and documented graphics gaps. |
| [XValkyrie](https://github.com/sammwyy/xvalkyrie) | C++20 Xbox One/Series compatibility layer; title code runs natively in a Windows process; console imports implemented over host APIs | Experimental/general compatibility project; not a GTA VI-specific ready-made port | Research/reference for a broader HLE/compatibility-layer architecture. Do not assume its API coverage is sufficient for this title. |
| [Project ReCity](https://github.com/pixl27/ProjectReCity) | Leonida-derived static recompilation and tooling | Public repository describes itself as derived from Leonida; inspect its exact delta and provenance | Useful for comparing later modifications and GPU-test work; do not duplicate it if it contributes no verified advantage over Leonida Legacy. |

### 2.1 Important correction about the architecture

The preferred direction is **not** to write a new CPU emulator by default. The reported VIctory architecture maps the title's PE image and runs its x86-64 code unchanged inside a Windows process, while implementing the Xbox platform layer underneath it. The graphics and other console-specific data still require translation or adaptation.

Therefore:

- Keep the original CPU instructions native where possible.
- Implement the loader and ABI correctly.
- Resolve imported Xbox/platform APIs through explicit HLE modules.
- Translate console GPU programs to the host-supported shader representation.
- Reproduce Xbox GPU/resource semantics on the host API.
- Introduce selective instruction translation only when a reproducible incompatibility demonstrates that it is required.

### 2.2 Primary repository choice

Start by auditing VIctory. It currently has the strongest reported end-to-end execution milestone among the checked candidates. This is a **default choice**, not an unconditional mandate. If the code is no longer maintained, cannot be built, has incompatible licensing, has regressed, or is materially behind another candidate at the time of the audit, document the evidence and recommend the better base before proceeding.

Do not merge VIctory, Leonida Legacy, XValkyrie, and ReCity wholesale. Their architectures, data models, interfaces, and licenses may differ. Port only narrow, well-understood components when there is a demonstrated advantage and a license/provenance review.

---

## 3. Legal, privacy, security, and data-handling requirements

This project must be designed as a local compatibility tool, not a distribution mechanism for game content.

### 3.1 Required input model

- The user supplies a local game directory that they are legally entitled to use.
- The tool accepts only supported, already-readable files.
- The tool must not contain or provide methods to decrypt protected content or bypass platform access controls.
- If the required executable or data is absent or unreadable, report the missing input clearly and continue with synthetic tests, GPU tests, loader fixtures, and other work that does not require proprietary game files.
- The agent must not ask the user to upload the game executable, keys, full package, extracted assets, or memory dumps to a chat or repository.

### 3.2 Repository hygiene

Never commit or upload:

- Game executable or DLL files.
- Game archives, package files, or proprietary data files.
- Extracted models, textures, audio, animations, maps, or cinematics.
- Generated shader binaries or per-game shader caches.
- Save data, profile data, user configuration, or crash memory dumps.
- Screenshots or recordings that reproduce proprietary game content unless the user explicitly confirms that sharing is lawful and appropriate.

Allowed repository content should be limited to original source code, tests, generic synthetic fixtures, non-proprietary documentation, build scripts, and test reports that do not expose private or protected data.

Add `.gitignore` patterns for all local game-data, cache, build-output, capture, dump, save, and log directories. Add a pre-commit asset-policy check if the selected repository provides one; preserve and understand it rather than disabling it.

### 3.3 Privacy

- Never perform an unrestricted whole-drive scan.
- Require an explicit game path or use the selected base's existing opt-in path workflow after auditing it.
- Do not upload telemetry, asset hashes, crash reports, shader data, or system information.
- Redact usernames and personal directory names from shared logs.
- Keep game data, generated data, caches, and saves outside the source repository.
- Do not log raw memory contents by default.

### 3.4 Offline operation

The first working runtime is offline-only. Platform APIs that expect an online account or service should return a documented, deterministic offline result sufficient for local startup, where appropriate. Do not fake success for operations that the game requires to be genuinely satisfied. Log any unavailable service and its effect on compatibility.

### 3.5 Third-party code and licensing

Before copying or adapting any component:

1. Identify its original repository and exact commit.
2. Read its license and third-party notices.
3. Check whether source or binary redistribution is permitted.
4. Preserve copyright notices and attribution.
5. Record the source and modifications in `THIRD_PARTY_NOTICES.md` or the existing equivalent.
6. Keep independent components separated when license compatibility is uncertain.
7. Do not assume that public GitHub availability means unrestricted reuse.

The checked VIctory and Leonida Legacy repositories describe GPL-3.0-family licensing. XValkyrie also lists GPL-3.0. Verify the current exact license files and all bundled dependencies.

---

## 4. Technical strategy

### 4.1 High-level execution model

```text
                    USER-SELECTED LOCAL GAME DIRECTORY
                                  |
                     supported-file/version validation
                                  |
             +--------------------+---------------------+
             |                                          |
       game executable                            game data files
             |                                          |
       PE image loader                      archive / texture / audio readers
             |                                          |
       import resolution                                |
             |                                          |
       Xbox HLE runtime                       local streaming / decode cache
             |                                          |
             +----------------------+-------------------+
                                    |
                              title x86-64 code
                                    |
                       Xbox D3D12.X / platform calls
                                    |
                   graphics compatibility implementation
                                    |
                    resource, descriptor and queue model
                                    |
                         host graphics backend
                                    |
                    D3D12 initially; Vulkan only if
                    baseline evidence supports it
                                    |
                           host GPU + display
```

Supporting services: input, windowing, audio, local save/profile, configuration, logging, crash reporting, shader cache, file streaming, and tests.

### 4.2 CPU strategy

**Default: native x86-64 execution.**

The CPU path should focus on:

- Validating the PE32+ image and section layout.
- Mapping sections with correct permissions and alignment.
- Applying relocations correctly.
- Resolving imports and delay imports.
- Supporting title DLL dependencies and load order.
- Correct entry-point and initialization behavior.
- Thread-local storage (TLS) and static TLS behavior.
- Exception/unwind metadata and stack unwinding.
- Calling conventions and ABI details.
- Thread creation, scheduling assumptions, synchronization, atomics, and wait primitives.
- Virtual memory semantics and page protections.
- Correct process/thread lifetime and shutdown behavior.

Do not add a general-purpose CPU emulator until a concrete instruction-level requirement has been demonstrated. If one is needed, isolate it behind a narrow interface and add regression tests for every handled case.

### 4.3 Runtime/HLE strategy

Implement the Xbox-facing interfaces the title actually imports or dynamically queries. Avoid implementing an imaginary complete Xbox API surface.

Use an import census and runtime traces to establish the required surface. Organize each implementation by module/library and API family. Every API should have:

- Exact observed import name and ordinal, if applicable.
- Expected calling convention and parameter/return layout.
- Known structure layouts and alignment requirements.
- Host implementation or deterministic offline behavior.
- Error/status mapping.
- Thread-safety requirements.
- Logging and trace policy.
- Unit tests and a stand-in executable test where practical.
- A compatibility status: implemented, partial, stubbed, unsupported, or unverified.

An unimplemented import must never silently appear to work. In diagnostic mode, trap or fail loudly. Any optional fallback mode must log the import and be clearly marked as unsafe for correctness.

### 4.4 Graphics strategy

Use the selected base's existing renderer first. VIctory currently describes a host D3D12 backend and DXIL shader translation. Leonida Legacy describes an Xbox D3D12-to-Vulkan backend and SPIR-V translation. These are different implementation paths.

**Do not create two full renderers at once.** Pick one backend for the first functional milestone, preserve a clean interface boundary, and use measurements to justify any second backend.

The graphics compatibility layer must eventually cover:

- Device and adapter selection.
- Command queues and queue submission.
- Command allocators and command lists.
- Fences and GPU/CPU synchronization.
- Resource creation, placement, aliasing, residency, and lifetime.
- Resource states and barriers.
- Descriptor heaps and descriptor lifetime rules.
- Root signatures and binding-layout translation.
- Graphics and compute pipeline state objects.
- Vertex/index buffers and input assembly.
- Render targets, depth/stencil targets, resolves, and copies.
- Samplers and texture views.
- Draw, indexed draw, instanced draw, indirect draw, and dispatch operations.
- Mesh shader and amplification/task shader paths where required.
- Geometry/tessellation or merged-stage behavior if observed.
- Stream output if observed.
- Query heaps, timestamps, predication, and occlusion queries.
- Swapchain/presentation and frame synchronization.
- Texture tiling, swizzling, metadata planes, and format conversion.
- GPU memory addressing and synchronization semantics.
- HDR, scaling, and final presentation only after basic correctness.

Never substitute a fake successful no-op for a graphics method that affects output. Unsupported features must be reported and should fail in diagnostic builds.

### 4.5 Shader strategy

The console's precompiled RDNA2 GPU programs are not ordinary PC shaders. The project needs a translation layer with a documented semantic contract.

Possible pipeline paths, depending on the chosen base:

- RDNA2 machine code → an internal IR/SSA representation → DXIL for host Direct3D 12.
- RDNA2 machine code → an internal IR/SSA representation → SPIR-V for Vulkan.

Use LLVM AMDGPU facilities as a reference/tooling source where the selected project already does so. Use DXC for DXIL generation/validation and SPIRV-Tools for SPIR-V assembly/validation when appropriate.

Do not assume that a shader that assembles or validates is semantically correct. Required work includes:

1. Parse and identify the console shader container.
2. Validate container lengths, offsets, alignment, version, and metadata.
3. Decode native program metadata and stage information.
4. Disassemble instructions using the correct target assumptions.
5. Build a typed internal representation.
6. Preserve lane masks and execution semantics.
7. Track scalar/vector registers and partial-register behavior.
8. Track descriptors, resource roots, memory accesses, and address spaces.
9. Translate control flow, branches, loops, barriers, atomics, LDS/shared memory, and wave operations.
10. Translate vertex fetch, system values, barycentrics, exports, texture sampling, and image operations.
11. Preserve numeric precision and floating-point corner cases.
12. Validate generated intermediate code.
13. Compile and execute micro-tests on actual GPU hardware.
14. Cache by shader content hash, compiler version, backend version, and relevant GPU feature set.
15. Record unsupported opcodes and translation diagnostics; never silently drop them.

#### Wave-size correctness

A major risk is wave64 behavior. Some GPU vendors/drivers primarily expose 32-lane subgroups. Wave64 instructions cannot simply be treated as wave32 instructions. Implement and test explicit semantics for lane shuffles, ballots, reductions, masks, cross-half operations, barriers, and wave-uniform branches. Any emulation must be validated against reference results.

Maintain tests for both native 64-lane execution and 32-lane emulation paths. Mark shader features as exact, emulated-exact, approximate, or unsupported. Approximate behavior must be visible in reports and must not be called fully compatible.

### 4.6 Asset and streaming strategy

Use the selected project's existing file readers and archive decoders where possible. The checked Leonida Legacy work describes tools for RPF9 archives, texture pools, model export, audio banks, and shader carving. Audit exact supported versions and limitations before depending on them.

Required stages:

- Detect a user-configured game root.
- Validate supported package identity using safe metadata and hashes where appropriate.
- Produce a local manifest (name, size, supported version identifier, and SHA-256 where useful).
- Validate archive headers and bounds before reading.
- Avoid trusting offsets or sizes from untrusted data.
- Decode only supported known formats.
- Make archive reads asynchronous where needed, with bounded worker pools.
- Avoid unbounded memory allocation from archive metadata.
- Implement cancellation and clear error reporting.
- Use streaming and caching rather than eagerly duplicating all large assets in RAM.
- Keep extracted/cached assets outside the repository.
- Make interrupted builds resumable and ensure partial outputs are not treated as valid.

Do not write a new archive format decoder if an already-tested compatible decoder exists and its license is suitable.

### 4.7 Audio

Audit audio bank formats and the selected runtime's codec support. The checked repositories mention Opus and XMA2 content, but do not assume every stream has been validated against actual title playback.

Tasks:

- Identify audio-bank/index structure from documented parsers.
- Preserve sample rate, channel layout, loop points, and timing metadata.
- Implement or reuse permitted Opus/XMA2 decoding.
- Compare decoded output against known test vectors where available.
- Route audio to Windows through the selected backend (WASAPI/XAudio2 or existing project abstraction).
- Test streaming, latency, device changes, voice lifetime, reverb/effects where required, and clean shutdown.
- Provide a capture path for debugging that saves only to a user-selected local directory.

### 4.8 Input and windowing

Minimum initial input target:

- XInput-compatible gamepads.
- Keyboard/mouse fallback if already supported by the base.
- Correct analog stick ranges and dead zones.
- Button/trigger mapping.
- Focus and window activation handling.
- Device disconnect/reconnect handling.
- Rumble only after basic input correctness.

Do not add an extensive custom GUI before the title can run. Keep CLI startup, configuration, logs, and diagnostics reliable first.

### 4.9 Local platform services

Implement only the offline subset needed for startup and local play:

- Local user/profile identity expected by the title.
- Offline license/entitlement behavior only where legitimately applicable and documented by the base project; do not forge or bypass protected entitlements.
- Local package metadata required for the runtime.
- Persistent local save directory.
- Text-entry dialog where required.
- Local storage paths mapped to a dedicated user-data directory.
- Asynchronous task/queue semantics needed by the game.
- Time and locale services.
- Local logging and crash reporting.

For any service where a correct offline implementation is not understood, report it as unsupported rather than manufacturing arbitrary success values.

### 4.10 Network policy

Network is disabled by default. The title should not connect to external game services. Any API requiring remote authentication, online license validation, multiplayer, matchmaking, telemetry, or cloud operations should return a documented offline state. Do not redirect the game to unofficial servers.

---

## 5. Repository selection and integration decision

### 5.1 Comparison checklist

For each candidate, record:

- [ ] Exact Git URL and repository owner.
- [ ] Latest default-branch commit SHA and commit date.
- [ ] License and third-party notices.
- [ ] Last meaningful source change, not merely README edits.
- [ ] Build success on the current machine.
- [ ] Unit/integration test success.
- [ ] Supported OS/compiler/toolchain versions.
- [ ] Current executable loader state.
- [ ] Import resolution and HLE coverage.
- [ ] CPU/TLS/unwind support.
- [ ] Graphics backend and tested GPUs.
- [ ] Shader instruction coverage and validation status.
- [ ] Asset/archive format coverage.
- [ ] Audio/input/save support.
- [ ] Crash/diagnostic tooling.
- [ ] Open blockers and maintainer activity.
- [ ] Whether the exact code can legally be reused in this project.

### 5.2 Selection rule

Choose one primary codebase based on **reproducible end-to-end progress**, not stars, forks, headlines, Discord claims, or screenshots.

Default selection order:

1. VIctory if the current commit builds and its loader/HLE/runtime/graphics path is demonstrably the best route to the next gameplay milestone.
2. Another candidate only if the audit shows a clear technical advantage that outweighs migration cost.
3. If no candidate is reproducibly buildable, create a small research harness and fix the build/environment issue before starting a rewrite.

### 5.3 Integration rules

- Treat the selected repository as the source of truth.
- Keep other projects as read-only references until a component-level migration is approved.
- Compare data models and API semantics before porting any module.
- Preserve upstream commit history and attribution when forking.
- Maintain an upstream remote and record the fork base commit.
- Do not rename every module or restructure the workspace during the first sprint.
- Avoid introducing a new language into an existing codebase without a measurable need.
- Since VIctory is a Rust workspace, keep Rust as the default implementation language if VIctory is selected. Use C/C++ only when required by a stable native ABI, platform SDK, dependency, or measured performance requirement. Do not rewrite Rust modules in C++ merely because C++ is common in game development.
- If Leonida Legacy is selected, preserve its actual split between Python tools and native runtime rather than forcing it into VIctory's Rust architecture.

---

## 6. Development environment and capability audit

### 6.1 Minimum environment audit

Record the following without exposing unnecessary personal data:

| Item | What to record | Why |
|---|---|---|
| OS | Windows edition/build, x64 | Loader/runtime compatibility |
| CPU | Vendor/model, logical cores, AVX2 availability | Build/runtime requirements |
| RAM | Installed and available RAM; commit limit | Large build and shader operations |
| GPU | Vendor/model, VRAM | Shader and resource behavior |
| Driver | Exact graphics driver version | Reproducibility and GPU bugs |
| Graphics API | D3D12 feature level, Shader Model, mesh shader support; Vulkan version/extensions if relevant | Backend eligibility |
| Storage | Free space on source/work/cache volumes | Avoid partial builds and disk exhaustion |
| Toolchains | Rust, MSVC, Windows SDK, Python, .NET, LLVM, Vulkan SDK versions as required by chosen base | Reproducible builds |
| Git | Version and working tree state | Source integrity |

### 6.2 Expected tools for the default VIctory path

Use the pinned versions in the repository, not arbitrary latest versions.

- Git for source control.
- Rust toolchain pinned by `rust-toolchain.toml`.
- Cargo for workspace builds and tests.
- Visual Studio Build Tools / Windows SDK only where required by the selected native dependencies.
- Windows 10/11 x64.
- CPU with AVX2, as stated by the checked VIctory README.
- Direct3D 12-capable GPU with Shader Model 6.6; the checked VIctory README states mesh-shader support is needed for the world.
- Sufficient Windows commit/pagefile capacity. The checked VIctory README mentions several GiB of available commit and error 1455 when commit is exhausted.

Do not assume that a GPU is compatible simply because it supports DirectX 12. Query the actual runtime feature levels and required shader/mesh capabilities.

### 6.3 Reference-path tools

If researching Leonida Legacy's Vulkan path, verify its current documented dependencies. The checked README lists Windows 10/11 x64, Python 3.11+, .NET 8 SDK, Vulkan SDK, LLVM 21+ / `llvm-mc`, Visual Studio 2022 C++ desktop development tools, and substantial free work space (the checked README states approximately 350 GB for its workflow). Treat the space figure as a repository-specific current estimate, not a universal minimum.

Potential reference tools:

- LLVM AMDGPU disassembler/toolchain.
- DirectX Shader Compiler (`dxc`, `dxv`).
- SPIRV-Tools (`spirv-as`, `spirv-val`, `spirv-dis`).
- Ghidra or another PE/RTTI inspection tool for lawful static analysis.
- PIX / D3D12 debug layer / DRED where appropriate and available.

### 6.4 PC readiness script

If the selected repository already provides a readiness tool, run it and review what it collects. Do not duplicate it without reason. Otherwise, create a read-only `tools/pc_readiness` utility that:

1. Detects OS and architecture.
2. Queries CPU feature support.
3. Queries memory and disk capacity.
4. Queries D3D12 adapter and required feature support.
5. Queries Vulkan capabilities only if the Vulkan backend is being evaluated.
6. Checks required tools and exact versions.
7. Checks that configured source/data/work/cache paths exist and are writable where necessary.
8. Prints pass/warn/fail with remediation guidance.
9. Writes no game data and performs no unrestricted filesystem scan.

---

## 7. Architecture and module responsibilities

Follow the selected repository's existing architecture. The structure below is a conceptual responsibility map, not an instruction to create duplicate modules if equivalent modules already exist.

### 7.1 Executable/PE layer

Responsibilities:

- Read PE32+ headers safely.
- Validate machine type and image characteristics.
- Parse sections, data directories, imports, delay imports, exports, relocations, TLS, exception/unwind data, and required metadata.
- Map image sections with correct alignment and protection.
- Apply relocations using checked arithmetic.
- Bind imports to HLE exports, approved host exports, or explicitly loaded title-supplied modules.
- Maintain module load order and pseudo-handles only where the platform API contract requires them.
- Track all allocated ranges and release them on unload or failure.
- Report import resolution with DLL, symbol/ordinal, resolved provider, and confidence/status.

Tests:

- Synthetic PE fixture with sections and relocations.
- Missing import fixture.
- Duplicate import fixture.
- Malformed offsets and truncated files.
- TLS fixture.
- DLL load fixture.
- Exception/unwind fixture where possible.
- Stand-in executable with simple startup, file, thread, and shutdown operations.

### 7.2 HLE module manager

Responsibilities:

- Maintain a registry of supported Xbox libraries and exports.
- Resolve exact import names and ordinals.
- Support runtime lookup APIs such as LoadLibrary/GetProcAddress equivalents when required.
- Keep host pass-through policy explicit and allowlisted.
- Distinguish title-local DLLs, host DLLs, console-only libraries, and unsupported imports.
- Avoid broad host-DLL pass-through that could accidentally resolve an incompatible symbol.
- Emit structured logs for unresolved APIs.

Suggested logical API domains (actual library names must come from import census):

- Kernel and process/thread primitives.
- Memory and virtual-memory management.
- File and storage operations.
- Runtime/package/user identity.
- Async task/queue APIs.
- Graphics and D3D12.X interfaces.
- Input/controller APIs.
- Audio APIs.
- Text input/dialog APIs.
- Time/locale utilities.

### 7.3 GPU abstraction

The current backend is the source of truth. A useful internal conceptual split is:

- `Device`: adapter/device creation and feature query.
- `Queue`: queue type, submit, fence wait/signal.
- `Allocator`: command storage management.
- `CommandList`: recording and reset lifecycle.
- `Resource`: buffers, textures, placed/reserved resources and state.
- `DescriptorHeap`: descriptor allocation, writes, copies, lifetime.
- `RootSignature`: binding layout and root parameter mapping.
- `Pipeline`: graphics/compute PSO construction and cache.
- `QueryHeap`: timestamps, occlusion, predication.
- `Swapchain`: presentation, resize, and back-buffer lifecycle.
- `ShaderCompiler`: console code → host representation.

Every object must have defined ownership, thread safety, lifetime, error behavior, and debug naming. If a COM-compatible interface is required, follow the ABI precisely; do not approximate vtable layout or reference counting.

### 7.4 Shader translator

Split the translator into independently testable stages:

1. Container identification and extraction.
2. Native instruction stream decoding.
3. Metadata decoding.
4. IR construction.
5. CFG reconstruction.
6. Register and lane execution model.
7. Resource/descriptor analysis.
8. Semantic lowering.
9. Backend emission.
10. Validation.
11. GPU micro-test execution.
12. Cache insertion and retrieval.

For every unsupported opcode/feature, emit a structured diagnostic containing shader hash, stage, opcode/feature, relevant metadata, and translation stage. Avoid dumping copyrighted shader data into public logs.

### 7.5 File/asset pipeline

- Keep source paths, work output, and cache output separate.
- Use a version manifest and exact supported-version list.
- Validate every length and offset before allocation or seek.
- Use bounded parallelism and bounded queues.
- Keep deterministic output ordering where feasible.
- Write outputs to temporary paths and atomically rename after validation.
- Record tool version and source hash in local cache metadata.
- Resume interrupted jobs only when source hash and tool version match.
- Provide a clean-cache command that cannot delete the user's original game directory.

### 7.6 Crash and diagnostic subsystem

Every crash report should include:

- Project version and commit SHA.
- OS build and driver version.
- CPU/GPU identifiers needed for diagnosis.
- Runtime backend and selected adapter.
- Game/package version identifier, not the game binary itself.
- Faulting module plus RVA when known.
- Thread identifier and stack trace.
- Last runtime/HLE event.
- Last GPU submission and DRED/device-removed details if available.
- Shader/pipeline hash identifiers, not raw copyrighted shader contents.
- Whether the run was normal, diagnostic, software-rendered, or GPU-based-validation mode.

Do not capture arbitrary process memory by default. Any opt-in memory capture must be local, explicit, bounded, and clearly labelled as potentially containing proprietary/user data.

---

## 8. Diagnostic and performance strategy

### 8.1 Safe bring-up sequence

1. Run parser and unit tests without game data.
2. Run PE/HLE stand-in tests.
3. Run GPU tests with synthetic triangle/texture/compute workloads.
4. Run the game only if the user has a supported, readable local copy and the current project prerequisites pass.
5. Start with diagnostic logging and conservative settings.
6. Prefer a software adapter or safe diagnostic fallback if the upstream project documents one.
7. Do not repeatedly trigger a known GPU hang on a production machine.
8. Record the first reproducible failing stage and stop changing unrelated components.

### 8.2 D3D12 diagnostics

If using the VIctory codebase, inspect its existing diagnostic variables before adding new ones. The checked README documents:

- `VICTORY_LOG` for component log levels.
- `VICTORY_STACKS` for timed thread stack logging.
- `VICTORY_WATCH` for watching a title memory address.
- `VICTORY_KEYS` for timed input injection.
- `VICTORY_CRASH_SAVE` for explicit bounded memory capture.
- `VICTORY_CAPTURE` / `VICTORY_CAPTURE_FRAMES` for frame capture.
- `VICTORY_DUMP_TARGETS` for render-target dumps during capture.
- `VICTORY_D3D12_DEBUG` for the D3D12 debug layer.
- `VICTORY_D3D12_GBV` for GPU-based validation.
- `VICTORY_D3D12_DRED` for device-removed diagnostics.
- `VICTORY_SKIP_PIPELINES` for temporarily omitting specified pipelines.
- `VICTORY_GPU_TIMING` for GPU timing diagnostics.
- `VICTORY_D3D12_WARP=1` for the software adapter, which is slow but avoids a physical GPU driver reset.
- `VICTORY_SHADER_LOOP_LIMIT` for a diagnostic shader loop limit.

These switches are specific to the checked VIctory build. Confirm that they still exist in the current commit and read their implementation before using them. Do not set undocumented variables by guesswork.

### 8.3 GPU failure investigation

For a black/incorrect world or device removal:

1. Record the exact scene transition and time to failure.
2. Record GPU/driver/OS/backend and diagnostic mode.
3. Capture logs and DRED output.
4. Determine whether the CPU submitted commands and whether the GPU executed them.
5. Count submitted, skipped, unsupported, and failed draws/dispatches.
6. Group failures by pipeline/shader hash and stage.
7. Check resource state transitions and barriers.
8. Check descriptor validity and descriptor lifetime.
9. Check resource sizes, offsets, alignments, and format/tiling conversion.
10. Check root signature/descriptor mapping and shader resource binding.
11. Check mesh pipeline and indirect argument interpretation.
12. Check shader instruction coverage, especially wave operations, atomics, export semantics, and vertex fetch.
13. Construct a minimal synthetic reproducer for the first confirmed defect.
14. Fix one defect and add a regression test before moving on.

Do not assume the first black frame is caused by one missing shader. It may also result from resource states, synchronization, swapchain/present, texture layout, descriptor lifetime, query behavior, or a driver/device fault.

### 8.4 Performance measurement

Measure separately:

- Cold start time.
- Warm start time.
- Shader translation time.
- Pipeline creation time.
- Game data decode/stream time.
- CPU frame time.
- GPU frame time.
- Present/frame pacing time.
- Peak committed memory and working set.
- GPU memory allocation and residency.
- Cache hit rate.
- Draw/dispatch count and skipped pipeline count.

Do not optimize before obtaining a correct baseline. First target correctness and stability; then remove pathological frame-time spikes and improve cache behavior.

---

## 9. Testing strategy and quality gates

### 9.1 Test layers

| Layer | Test type | Must run in CI? | Requires game data? |
|---|---|---:|---:|
| Parsers | Unit/property/fuzz tests for bounds and malformed input | Yes | No |
| PE loader | Synthetic PE fixtures and stand-in executable | Yes | No |
| HLE | Unit tests for signatures, status mapping, threading, and offline behavior | Yes | No |
| Formats | Synthetic archives/textures/audio test vectors | Yes | No |
| Shader IR | Instruction-level and CFG tests | Yes | No |
| Shader compiler | Translation, validator, and golden tests | Yes where toolchain available | No proprietary corpus required |
| GPU harness | Triangle, texture, buffer, compute, synchronization tests | Hardware CI if available | No |
| Runtime integration | Local user-provided game package | No public CI | Yes |
| Regression | Reproduce known boot/world failure | Local hardware lab | Usually yes |

### 9.2 Required test practices

- Every bug fix must add a regression test at the lowest practical layer.
- Every parser must reject truncated, oversized, overlapping, and out-of-range data without panic or unbounded allocation.
- Every unsafe block must document its invariants.
- FFI boundaries must not unwind Rust panics or C++ exceptions across ABI boundaries.
- Concurrent code must be exercised with stress tests where practical.
- Cache output must be deterministic for the same input/toolchain/feature set.
- Tests must distinguish “not supported” from “passed”.
- GPU tests must report adapter and driver identifiers.
- Long-running tests must have timeout and cleanup behavior.
- Failures must retain enough diagnostic information to reproduce them.

### 9.3 Continuous integration

Create/retain CI jobs for:

1. Formatting and linting.
2. Unit tests.
3. Parser fuzz/sanitizer tests where practical.
4. Stand-in executable tests.
5. Shader IR/translator tests that do not need game content.
6. License and third-party notice checks.
7. Asset-policy checks that fail if game files or generated proprietary content are added.
8. Build on the supported Windows toolchain.
9. Optional GPU harness on known hardware runners.

Public CI must not require the user to upload game files and must not contain proprietary assets.

---

## 10. Detailed implementation roadmap

Work in small tasks. Do not skip a gate because the next phase looks more exciting. The checkboxes are intended to become the project issue list.

### Phase 0 — Workspace, audit, and baseline (P0)

**Goal:** Know exactly what code is being built, what machine is available, and which reported capabilities can be reproduced.

- [ ] Inspect current directory and list existing files.
- [ ] Check whether current directory is already a Git repository.
- [ ] Record current branch, remotes, commit SHA, and `git status`.
- [ ] Preserve all user modifications; do not run `git reset --hard`, `git clean -xfd`, or destructive cleanup.
- [ ] Read this plan and current agent instructions in the repository.
- [ ] Fetch/read the latest VIctory README and architecture documentation.
- [ ] Fetch/read the latest Leonida Legacy README, design documentation, D3D12/Vulkan documentation, and contribution guide.
- [ ] Inspect XValkyrie and ReCity only for relevant reference areas.
- [ ] Record exact upstream commit SHAs and dates.
- [ ] Compare licenses and third-party dependencies.
- [ ] Record current OS, CPU/AVX2, RAM, disk, GPU, driver, D3D12 feature level, Shader Model, mesh-shader support, and Vulkan support as relevant.
- [ ] Verify installed Rust toolchain and pinned toolchain compatibility.
- [ ] Verify C++/MSVC/Windows SDK only if required by selected codebase.
- [ ] Verify Python/.NET/LLVM/Vulkan tools only if required by selected reference/build path.
- [ ] Clone the selected upstream into a separate directory only if the workspace is empty or a safe separate path is available.
- [ ] Keep the upstream checkout untouched; create a local branch/worktree for modifications.
- [ ] Run the upstream build command.
- [ ] Run all upstream tests that do not require game data.
- [ ] Capture failures exactly, including command, output, exit code, and environment.
- [ ] Reproduce any claimed milestone only if authorized local input files exist.
- [ ] Write `docs/BASELINE_AUDIT.md` and `docs/ENVIRONMENT_REPORT.md`.
- [ ] Make a clear baseline decision: VIctory, another project, or stop at an environment blocker.

**Exit gate:** A clean baseline report and a reproducible build/test state. No game-data access is required to pass this phase.

### Phase 1 — Developer workflow and repository safety (P0)

**Goal:** Make it difficult to corrupt the workspace or accidentally publish proprietary content.

- [ ] Confirm `.gitignore` covers build output, cache, data, saves, captures, dumps, logs, and temporary files.
- [ ] Add or validate asset-policy/pre-commit checks.
- [ ] Confirm source and user data are separated.
- [ ] Add a documented local configuration example with placeholder paths only.
- [ ] Ensure no real game path is committed.
- [ ] Ensure logging redacts usernames and personal paths where possible.
- [ ] Document safe build, test, run, clean, and diagnostics commands.
- [ ] Add an issue/bug report template that explicitly forbids attaching game data, extracted assets, or proprietary shader binaries.
- [ ] Add/update third-party notices.
- [ ] Add a reproducible build report format.

**Exit gate:** Repository safety checks pass and a new developer can build/tests without game content.

### Phase 2 — Baseline loader and import census (P0/P1)

**Goal:** Verify the loader's correctness with synthetic and stand-in executables; understand the APIs required by the title.

- [ ] Read the PE parser and loader implementation end to end.
- [ ] Map out the loader's phases and ownership model.
- [ ] Trace section mapping and page protection behavior.
- [ ] Trace relocation handling and overflow checks.
- [ ] Trace normal and delay import resolution.
- [ ] Trace title DLL load/unload behavior.
- [ ] Trace TLS initialization and cleanup.
- [ ] Trace exception/unwind support.
- [ ] Trace process/thread startup and shutdown.
- [ ] Verify loader diagnostics include module and RVA.
- [ ] Run existing stand-in HLE fixture tests.
- [ ] Add tests for a missing library and missing export.
- [ ] Add tests for malformed PE data and bad offsets.
- [ ] Add tests for relocation and TLS behavior.
- [ ] Verify unsupported imports trap loudly in diagnostic mode.
- [ ] Produce an import census using lawful local input only when available.
- [ ] Group imports by library and API subsystem.
- [ ] Prioritize the smallest missing APIs blocking the next observed startup milestone.
- [ ] Create an HLE coverage table with status and tests.

**Exit gate:** Stand-in executable passes through load, imports, startup, file/thread operations, and clean exit; title import requirements are documented where known.

### Phase 3 — HLE runtime correctness (P1)

**Goal:** Implement the smallest accurate set of platform services needed for title startup and offline use.

- [ ] Review import signatures against trustworthy headers/specifications or observed ABI data.
- [ ] Define ABI-safe types and structure layouts.
- [ ] Validate pointer alignment and size handling.
- [ ] Implement memory APIs with correct page granularity and protection transitions.
- [ ] Implement file APIs and virtual path mapping.
- [ ] Implement thread creation, wait, signal, and synchronization behavior.
- [ ] Implement async queue/task semantics needed by the title.
- [ ] Implement package/user/profile services required for offline startup.
- [ ] Implement local save persistence with atomic writes and corruption recovery.
- [ ] Implement text input only if required by the observed flow.
- [ ] Define deterministic offline return values for known optional online services.
- [ ] Return explicit unsupported status for unimplemented critical services.
- [ ] Add multithreaded stress tests.
- [ ] Add restart/persistence tests.
- [ ] Add crash/cleanup tests for failed initialization.
- [ ] Document each API and its observed use.

**Exit gate:** Runtime services are deterministic and tested; no unsupported critical operation silently succeeds.

### Phase 4 — GPU backend and minimal rendering (P1)

**Goal:** Prove the graphics backend independently of GTA VI.

- [ ] Map the current backend abstraction and ownership model.
- [ ] Verify adapter feature detection before device creation.
- [ ] Enable debug layer in diagnostic builds.
- [ ] Implement/verify device creation and feature reporting.
- [ ] Implement/verify a basic graphics queue.
- [ ] Implement/verify command allocator lifecycle.
- [ ] Implement/verify command list lifecycle.
- [ ] Implement/verify fence signal/wait.
- [ ] Implement/verify buffer creation and copy.
- [ ] Implement/verify texture creation and copy.
- [ ] Implement/verify RTV/DSV/SRV/UAV/sampler descriptors as applicable.
- [ ] Implement/verify basic root signature and graphics PSO.
- [ ] Implement/verify vertex/index binding and draw.
- [ ] Implement/verify compute PSO and dispatch.
- [ ] Implement/verify resource barriers and state tracking.
- [ ] Implement/verify swapchain and present.
- [ ] Implement a synthetic triangle test.
- [ ] Implement a textured triangle test.
- [ ] Implement a compute-to-buffer/readback test.
- [ ] Implement multi-queue/fence ordering tests if required.
- [ ] Verify device loss and recovery reporting.
- [ ] Verify clean shutdown and resource destruction.
- [ ] Add GPU tests to CI where a stable runner exists.

**Exit gate:** Synthetic GPU tests pass with validation enabled on at least one documented physical GPU and, where available, a software adapter.

### Phase 5 — Shader translation correctness (P1)

**Goal:** Identify and implement missing shader semantics without guessing.

- [ ] Read the translator design and internal IR definitions.
- [ ] Create an opcode/feature coverage matrix.
- [ ] Separate “parses”, “translates”, “validates”, and “executes correctly” statuses.
- [ ] Confirm stage classification and metadata extraction.
- [ ] Verify scalar/vector register semantics.
- [ ] Verify EXEC/lane-mask handling.
- [ ] Verify branch and loop reconstruction.
- [ ] Verify partial-register operations.
- [ ] Verify wave operations and wave-size assumptions.
- [ ] Verify LDS/shared memory, barriers, atomics, and memory ordering.
- [ ] Verify descriptors, root parameters, bindless access, and address conversion.
- [ ] Verify vertex fetch and input layout.
- [ ] Verify stage system values and shader exports.
- [ ] Verify texture/sample/image operations and format semantics.
- [ ] Verify floating-point precision, denormals, NaN, signed zero, and contraction behavior where relevant.
- [ ] Add instruction-level golden tests.
- [ ] Add CFG and resource-analysis tests.
- [ ] Validate generated DXIL/SPIR-V with the correct validator.
- [ ] Execute micro-shaders on actual hardware.
- [ ] Add targeted wave64 tests on a 64-lane capable device if available.
- [ ] Add 32-lane wave64-emulation tests if that path is claimed.
- [ ] Mark unsupported or approximate features in generated reports.
- [ ] Add cache invalidation on translator/compiler/backend version changes.

**Exit gate:** All supported shader features have tests; unsupported features are reported and do not silently become dropped draws.

### Phase 6 — World-scene rendering blocker (P1)

**Goal:** Turn the current black/incomplete world scene into correct, visible rendering.

This phase begins only after the selected base has a reproducible path to the scene-loading stage. If that milestone is unavailable in the current commit or local input, keep working on synthetic equivalents and document the blocker.

- [ ] Reproduce the last known failing scene transition.
- [ ] Capture CPU logs, GPU logs, DRED/device removal data, and frame captures using documented local tools.
- [ ] Record all skipped/failed draw and dispatch calls.
- [ ] Group missing output by pipeline and shader stage.
- [ ] Identify whether mesh pipelines, vertex/pixel shaders, compute dispatches, indirect draws, or resource bindings are the first failure.
- [ ] Verify mesh PSO creation and draw argument layout.
- [ ] Verify indirect command decoding and clamping behavior.
- [ ] Verify descriptor heap lifetime across asynchronous submission.
- [ ] Verify root signature mapping and resource binding.
- [ ] Verify texture tiling/detiling and resource format metadata.
- [ ] Verify depth/stencil and render-target metadata planes.
- [ ] Verify resource transitions, queue synchronization, and fences.
- [ ] Verify shader exports and render-target writes.
- [ ] Verify query/predication behavior if it changes rendering decisions.
- [ ] Use pipeline-skip tooling only as a diagnostic isolation technique, not as a final fix.
- [ ] Minimize each defect to a synthetic test.
- [ ] Fix the highest-impact confirmed defect.
- [ ] Add a regression test and compare captures before/after.
- [ ] Repeat until the scene is visibly rendered with no device hang.

**Exit gate:** A repeatable world scene with geometry, materials, lighting, and camera output—not merely a title card—renders without device removal on the documented test GPU.

### Phase 7 — Asset streaming and full game data path (P1/P2)

**Goal:** Make the world load reliably and efficiently from the user's local data directory.

- [ ] Audit archive reader coverage and supported version identity.
- [ ] Verify all reads use bounds checks and safe integer arithmetic.
- [ ] Verify compressed streams and decompression limits.
- [ ] Verify texture dictionary/pool handling and skipped formats.
- [ ] Verify model layout decoding and unresolved layout inventory.
- [ ] Verify audio-bank indexes and stream metadata.
- [ ] Add bounded asynchronous I/O.
- [ ] Add priority scheduling for visible/nearby assets if required by observed load behavior.
- [ ] Avoid full-game extraction when on-demand streaming is possible.
- [ ] Add cache size management and safe eviction.
- [ ] Add interrupted-build recovery.
- [ ] Verify file handles and memory mappings are released.
- [ ] Profile loading stalls and memory copies.
- [ ] Add a progress report for asset-loading tasks.

**Exit gate:** The title can complete a repeatable world load without unbounded memory growth, data corruption, or unrecoverable streaming stalls.

### Phase 8 — Audio, input, saves, and local session (P2)

- [ ] Verify controller mapping with at least one XInput gamepad.
- [ ] Verify keyboard fallback if supported by the base.
- [ ] Verify analog sticks, triggers, and dead zones.
- [ ] Add disconnect/reconnect handling.
- [ ] Validate audio decoder output using test vectors.
- [ ] Validate in-game music, sound effects, and voice streams where accessible.
- [ ] Validate stream timing and synchronization.
- [ ] Test device changes and audio endpoint selection.
- [ ] Verify save creation, update, restart, and recovery.
- [ ] Verify profile/settings persistence.
- [ ] Verify local user/session behavior after application restart.
- [ ] Keep all network-dependent functionality offline and explicit.

**Exit gate:** A local offline session can be controlled, heard, saved, and resumed reliably.

### Phase 9 — Stability, compatibility, and performance (P2)

- [ ] Run repeated cold starts and warm starts.
- [ ] Run a sustained session test.
- [ ] Run repeated scene transitions.
- [ ] Run save/load loops.
- [ ] Measure CPU, GPU, memory, streaming, and compilation costs separately.
- [ ] Fix severe frame-time outliers before chasing peak average FPS.
- [ ] Improve shader/pipeline cache hit rate.
- [ ] Improve asset streaming and avoid redundant copies.
- [ ] Test multiple GPU vendors where hardware is available.
- [ ] Test at least one system without native 64-lane subgroups if emulation is claimed.
- [ ] Document known hardware exclusions and workarounds.
- [ ] Verify no driver reset or device removal during the defined test period.
- [ ] Create a known-issues matrix.

**Exit gate:** A defined, repeatable stability test passes on each claimed hardware configuration.

### Phase 10 — Packaging and release readiness (P3)

- [ ] Create a clean Windows release build.
- [ ] Verify no proprietary game data is present in the release.
- [ ] Verify all third-party license notices are included.
- [ ] Include setup and troubleshooting instructions.
- [ ] Include supported game-version requirements without sharing game files.
- [ ] Include PC capability check output.
- [ ] Include safe crash-report collection instructions.
- [ ] Include data/cache location and cleanup instructions.
- [ ] Sign binaries only if the user authorizes and has appropriate certificate access.
- [ ] Publish only after explicit user authorization and final legal/security review.

**Exit gate:** A reproducible, self-contained compatibility tool that contains no game content and clearly documents its limitations.

---

## 11. Priority backlog

### P0 — Must be done before feature work

- Repository audit and exact commit pinning.
- Build/test reproduction.
- Environment and GPU capability detection.
- License and third-party review.
- Game-data exclusion and asset-policy enforcement.
- No-game-data CI.
- Crash/diagnostic baseline.

### P1 — Critical path to a rendered world

- Loader correctness and import census.
- HLE implementation of required startup APIs.
- D3D12/Xbox graphics object and ABI correctness.
- Descriptor/resource/barrier/fence semantics.
- Shader instruction and wave64 correctness.
- Mesh shader/indirect draw support.
- Texture layout/metadata correctness.
- Render-target and presentation path.
- Reproducible synthetic test cases for each graphics defect.

### P2 — Critical path to a usable offline session

- Asset streaming and memory behavior.
- Audio playback and synchronization.
- Controller/input correctness.
- Save/profile persistence.
- Stability across longer sessions.
- Cross-vendor compatibility.
- Shader and pipeline caching/performance.

### P3 — Quality improvements

- Improved UI/configuration experience.
- HDR and advanced display options.
- Additional graphics backends.
- Broader Xbox title compatibility.
- More extensive hardware coverage.
- Installer and release automation.

---

## 12. First sprint: exact execution checklist

The agent should work through these steps in order. Keep changes small and stop at a failed gate rather than silently skipping it.

### Sprint task 1 — Inspect the workspace

- [ ] Print current directory.
- [ ] List top-level files and directories.
- [ ] Detect Git repository and working-tree status.
- [ ] Detect existing user project files and preserve them.
- [ ] Check whether this plan is already inside an existing repository.
- [ ] Do not create or clone over an existing directory.

### Sprint task 2 — Inspect machine capabilities

- [ ] Detect Windows version/build and x64 architecture.
- [ ] Detect CPU model and AVX2.
- [ ] Detect available RAM and commit limit.
- [ ] Detect free disk space on workspace and intended cache volume.
- [ ] Detect GPU and driver version.
- [ ] Query D3D12 feature level, shader model, and mesh-shader support.
- [ ] Query Vulkan version/extensions only if considering Vulkan.
- [ ] Detect Rust/Cargo version and whether repository-pinned toolchain is installed.
- [ ] Detect MSVC/Windows SDK/Python/.NET/LLVM only if required.
- [ ] Write `docs/ENVIRONMENT_REPORT.md`.

### Sprint task 3 — Re-evaluate the repositories

- [ ] Read VIctory README and `docs/architecture.md` at the latest commit.
- [ ] Inspect the `Cargo.toml` workspace and crate layout.
- [ ] Inspect CI workflows and test fixture strategy.
- [ ] Read current license and third-party files.
- [ ] Read Leonida Legacy README, `AGENTS.md`, `CONTRIBUTING.md`, and relevant docs.
- [ ] Inspect its current shader/asset/runtime division.
- [ ] Check the current XValkyrie architecture and license.
- [ ] Compare ReCity's changes to its parent baseline; do not assume it is independent.
- [ ] Make a comparison table and choose one baseline.
- [ ] Write `docs/BASELINE_AUDIT.md` with commit SHAs and evidence.

### Sprint task 4 — Establish baseline build

If VIctory is selected and the pinned Rust toolchain is available, start with the commands documented by its current README. At the time checked, the repository documents commands in this family:

```powershell
# From the selected VIctory repository root
rustup show
cargo xtask install-hooks
Copy-Item victory.example.toml victory.toml
cargo test --workspace
cargo run -p victory -- config
cargo build --release -p victory
```

Notes:

- Review `victory.example.toml` before copying or editing it.
- Keep `victory.toml` local and uncommitted if it contains machine-specific paths.
- `cargo xtask install-hooks` modifies the local Git hook configuration; inspect the command and proceed only if appropriate for the workspace.
- Do not run the game command without a supported, user-supplied, lawfully readable local game directory.
- Record exact failures; do not patch around a failing test just to obtain a green build.

### Sprint task 5 — Build the no-game-data test baseline

- [ ] Run all workspace tests.
- [ ] Run formatting/linting commands documented by the repository.
- [ ] Run the synthetic HLE/loader fixture.
- [ ] Run the GPU harness or synthetic renderer tests if they do not require game data.
- [ ] Save a concise results summary in `docs/BUILD_AND_TEST_LOG.md`.
- [ ] Identify the first failing test or missing prerequisite.
- [ ] Fix only the smallest verified cause.
- [ ] Add a regression test.
- [ ] Re-run the relevant test and full impacted test suite.

### Sprint task 6 — Create the engineering map

- [ ] List workspace crates/modules and their responsibilities.
- [ ] Draw the runtime call path from CLI → loader → HLE → GPU → host API.
- [ ] Draw the data path from local package → file readers → cache → runtime.
- [ ] Identify all existing logs, diagnostics, and configuration switches.
- [ ] Identify all existing tests and missing test layers.
- [ ] Identify external dependencies and their licenses.
- [ ] Write `docs/CODEBASE_MAP.md`.

### Sprint task 7 — Pick one small engineering issue

Choose only after the previous tasks. The task should meet all of these requirements:

- It is backed by an observed defect or missing test.
- It is contained within one subsystem where possible.
- It can be validated without uploading game data.
- It has a clear pass/fail acceptance criterion.
- It does not require bypassing protection or modifying proprietary game content.

Examples of acceptable first engineering tasks:

- Fix a bounds-check bug in a PE/archive parser using synthetic fixtures.
- Add a missing HLE stand-in test.
- Improve a diagnostic message to include module and RVA.
- Add a GPU resource-lifetime test using synthetic buffers.
- Add a shader IR regression test for a documented unsupported operation.
- Fix a cache invalidation bug using generated synthetic shader inputs.

Do not choose a speculative full GPU backend rewrite as the first task.

### Sprint task 8 — Finish with a truthful report

At the end of the first sprint, report:

1. Which repository was selected and why.
2. Exact commit SHA and license.
3. Environment summary and missing prerequisites.
4. Exact build/test commands and outcomes.
5. Files changed and why.
6. Tests added and results.
7. Known blockers.
8. Next single highest-priority task.
9. Whether any game-data-dependent milestone was attempted. If not, say so plainly.

---

## 13. Engineering standards

### 13.1 General

- Follow existing repository formatting, naming, error-handling, and module conventions.
- Prefer small, reviewable diffs.
- Do not mass-format unrelated files.
- Use typed errors and include context at subsystem boundaries.
- Avoid broad `unwrap`/`expect` or unchecked casts in runtime paths.
- Check all sizes, offsets, counts, and integer conversions.
- Bound allocations and worker queues.
- Make long operations cancellable where practical.
- Use structured logs with stable event names.
- Document assumptions and unknowns next to the implementation.

### 13.2 Rust-specific rules (if VIctory is selected)

- Respect the pinned toolchain and workspace lint configuration.
- Keep `unsafe` blocks minimal and document invariants.
- Do not allow Rust panics to cross C/COM/OS ABI boundaries.
- Do not hold locks while invoking unknown host callbacks unless required and documented.
- Use RAII for handles and GPU objects.
- Make ownership and lifetime of mapped game memory explicit.
- Avoid data races in shared runtime state.
- Do not assume a pointer remains valid after a title callback or module unload.
- Keep FFI layout types explicit (`repr(C)` or the repository's equivalent) where required.

### 13.3 C/C++ rules (if used)

- Follow the repository's C++ standard and compiler settings.
- Use RAII and explicit ownership.
- Avoid exception propagation across ABI boundaries.
- Use static analysis and sanitizers where supported.
- Avoid reinterpret casts without a documented ABI justification.
- Match COM vtable order, calling convention, reference counting, and HRESULT behavior exactly where applicable.
- Add tests for alignment and structure packing.

### 13.4 Python tooling rules

- Pin tool versions or record them in the build report.
- Validate input paths and file sizes before processing.
- Use bounded memory and explicit encoding.
- Avoid network requests and broad filesystem scans unless explicitly needed and approved.
- Make tools resumable and deterministic where practical.
- Keep generated outputs outside source directories.

### 13.5 Documentation rules

Every subsystem documentation page should state:

- Purpose and scope.
- Input/output contract.
- Supported behavior.
- Unsupported behavior.
- Current implementation status.
- Test coverage.
- Known bugs.
- Data/provenance restrictions.
- Related source files.

---

## 14. Proposed documentation set

Use the repository's existing docs layout when possible. Avoid duplicate docs with the same purpose.

```text
docs/
  PROJECT_PLAN.md                 # this plan
  BASELINE_AUDIT.md               # repository choice and evidence
  ENVIRONMENT_REPORT.md           # machine/toolchain capability report
  BUILD_AND_TEST_LOG.md           # reproducible build/test record
  CODEBASE_MAP.md                 # crates/modules and data-flow maps
  ARCHITECTURE.md                 # project-specific architecture
  LOADER.md                       # PE loading, relocations, imports, TLS, unwind
  HLE_API_COVERAGE.md             # library/export implementation matrix
  GRAPHICS_BACKEND.md             # device/resources/queues/descriptors/pipelines
  SHADER_TRANSLATION.md           # IR and shader semantic coverage
  ASSET_PIPELINE.md               # version checking, archives, textures, streaming
  AUDIO.md                        # codecs, stream handling, playback
  INPUT.md                        # gamepad/keyboard mapping
  DIAGNOSTICS.md                  # logs, captures, DRED, crash reports
  TESTING.md                      # all test layers and how to run them
  HARDWARE_COMPATIBILITY.md       # verified GPU/driver/feature matrix
  KNOWN_ISSUES.md                 # reproducible outstanding issues
  THIRD_PARTY_NOTICES.md          # reused components and license notices
```

---

## 15. Definition of done

A task is done only when all applicable items below are true:

- [ ] The change addresses a specific documented requirement or defect.
- [ ] The implementation follows the selected architecture.
- [ ] Error handling is explicit and safe.
- [ ] No protected game content has been added to the source tree.
- [ ] Relevant tests have been added or updated.
- [ ] Relevant tests pass locally.
- [ ] The full impacted test suite passes.
- [ ] Diagnostics are sufficient to troubleshoot failure.
- [ ] Documentation and status tables are updated.
- [ ] Third-party notices are updated if code/dependencies changed.
- [ ] No unrelated files were modified.
- [ ] The final report distinguishes verified results from assumptions.

A milestone is done only when it can be reproduced by another developer using the documented toolchain, test inputs, commands, and expected result.

---

## 16. Open technical questions to resolve through evidence

Do not guess answers. Add evidence and a decision to the relevant architecture document.

1. Which exact Xbox package/build version is supported by the current chosen base?
2. Which required game files are already readable and legitimately available to the user?
3. Which executable and DLLs are required by the title's startup path?
4. Does the current loader correctly handle the title's PE metadata, TLS, unwind information, and dynamic imports?
5. Which Xbox libraries and exports are actually used during startup, menu, loading, and world rendering?
6. Which HLE APIs can safely return offline status, and which are hard startup dependencies?
7. Does the current renderer use host D3D12 only, or is a Vulkan backend available and tested?
8. Which GPU feature set is required by the currently failing scene?
9. Which shader stages and instruction classes are responsible for dropped/incorrect world draws?
10. Are wave64 semantics preserved on the target GPU, or emulated with verified exactness?
11. Are all texture formats, tiling modes, metadata planes, and sampler modes used by the title understood?
12. Are mesh shaders, indirect draws, stream output, geometry/tessellation behavior, queries, and presentation fully implemented where required?
13. Is the current black/incomplete world caused by shader gaps, resource binding, texture layouts, state transitions, synchronization, or a combination?
14. Which asset layouts and compressed formats remain unsupported?
15. Which audio codecs and effects are still unverified?
16. What is the minimum viable GPU feature set for a reliable first world render?
17. Which performance bottleneck dominates after correctness is achieved: translation, CPU execution, asset streaming, memory copying, or GPU execution?
18. Which features are truly required for the first offline playable session, and which can remain unsupported until later?

---

## 17. Source references checked for this document

These links are starting points for verification, not substitutes for reading the current source code. The agent must re-check current content, commit IDs, and licenses at execution time.

### Project repositories

- [VIctory repository](https://github.com/Marvisak/VIctory)
- [VIctory architecture documentation](https://github.com/Marvisak/VIctory/blob/main/docs/architecture.md)
- [Leonida Legacy repository](https://github.com/ProjectLeonidaL/leonida)
- [Leonida Legacy contributing guide](https://github.com/ProjectLeonidaL/leonida/blob/main/CONTRIBUTING.md)
- [Leonida Legacy agent instructions](https://github.com/ProjectLeonidaL/leonida/blob/main/AGENTS.md)
- [Project ReCity](https://github.com/pixl27/ProjectReCity)
- [XValkyrie](https://github.com/sammwyy/xvalkyrie)

### Graphics, shaders, and compiler references

- [Microsoft Direct3D 12 programming guide](https://learn.microsoft.com/windows/win32/direct3d12/directx-12-programming-guide)
- [D3D12 Debug Layer and GPU-based validation](https://learn.microsoft.com/windows/win32/direct3d12/using-d3d12-debug-layer-gpu-based-validation)
- [D3D12 diagnostics and DRED overview](https://learn.microsoft.com/windows/win32/direct3d12/understanding-the-d3d12-debug-layer)
- [Microsoft DirectX Shader Compiler (DXC)](https://github.com/microsoft/DirectXShaderCompiler)
- [Khronos SPIR-V registry/specification](https://registry.khronos.org/SPIR-V/)
- [LLVM AMDGPU backend user guide](https://llvm.org/docs/AMDGPUUsage.html)

### Notes on source status

The checked VIctory README reported, as of 2026-11-23, first-boot screens, intro video, start/profile/save flow, and world loading, while the first world scene was still mostly dark/incorrect and could hang/reset a physical GPU. It also described an AVX2 CPU, D3D12 Shader Model 6.6, and mesh-shader-capable GPU requirement for the world path.

The checked Leonida Legacy README reported, as of 2026-11-24, supported package detection and broad shader/asset tooling, but no full game boot yet; the Xbox D3D12-to-Vulkan implementation remained incomplete. Its documented build path listed Python 3.11+, .NET 8, Vulkan SDK, LLVM 21+, Visual Studio 2022 C++ tools, and approximately 350 GB free work space for that workflow.

These are repository-maintainer claims. Reproduce them before treating them as established facts.

---

## 18. Final instruction to the agent

Start with **Phase 0 and the First Sprint checklist**. Do not attempt to implement every subsystem immediately. Use VIctory as the default starting point only if the latest code still builds and provides the best reproducible path toward a rendered world. Use Leonida Legacy and XValkyrie as technical references, not as permission to merge code without a license and architecture review.

The immediate engineering objective is:

> **Establish a clean, tested, legally bounded development environment; reproduce the selected project's current baseline; isolate the next verified blocker; and fix that blocker with the smallest testable change.**

Be exact. Be incremental. Preserve the user's workspace. Never represent an unverified milestone as completed.
