# AGENTS.md — SeriesBridge Project Instructions

These instructions apply to the entire repository unless a more specific `AGENTS.md` exists in a subdirectory. They are written for coding agents and human contributors working on SeriesBridge.

## 1. Project identity and current status

SeriesBridge is a research-driven Windows compatibility project intended to investigate running supported, lawfully obtained Xbox Series X|S title files on Windows. Its proposed implementation uses:

- **C++20** for the launcher and compatibility runtime.
- **Qt 6 Widgets** for the desktop launcher.
- **Win32, DXGI, and Direct3D 12** for Windows process, display, memory, and graphics integration.
- **CMake + Ninja** for builds.
- **DXC** to compile generated HLSL into DXIL after a separate Xbox shader decoder/translator has produced valid HLSL.

The project is at the planning / early-architecture stage unless the checked-out source proves otherwise. Never describe planned, stubbed, mocked, or untested functionality as implemented. Do not claim that GTA VI, any other specific title, or any class of Xbox executable works without repeatable evidence.

Before changing code, inspect the current repository state. Do not assume that every directory, target, tool, feature, or document described below already exists.

## 2. Required reading before substantial changes

Read the relevant project documents before implementing or changing an architectural subsystem:

- `README.md` — project goals, boundaries, architecture, stack, and roadmap.
- `Xbox_Series_to_Windows_Development_Plan.md` — full development strategy, XCL baseline, memory model, shader pipeline, security, milestones, and acceptance criteria.
- `Resolution_Scaling_and_Display_Modes.md` — display/render/output resolution separation, scaling, DXGI resizing, and resolution test matrix.
- `SECURITY.md`, if present — vulnerability reporting and security policy.
- Any subsystem design document under `docs/` that applies to the task.

If these documents conflict, do not silently choose one. Prefer the safer and more conservative behavior, identify the conflict, and update the relevant documentation as part of the proposed change.

## 3. Non-negotiable legal and safety boundaries

- Do not implement or provide instructions for bypassing DRM, licensing, entitlement checks, package encryption, console security, anti-cheat, or online authentication.
- Do not extract, embed, publish, or redistribute proprietary firmware, private keys, SDK components, game executables, game assets, or copyrighted game data.
- Do not add automatic downloads of game files or protected console content.
- Assume users are responsible for supplying files they are legally entitled to use through an authorized process.
- Do not disable security controls or require administrator privileges merely to make the application work.
- Do not claim the runtime is a sandbox. Running native game code with the user's token gives it that user's permissions unless a separately designed and tested isolation boundary is actually in place.
- Keep title-specific compatibility work separate from mechanisms that defeat platform protections. If a task crosses these boundaries, stop and explain the limitation instead of implementing it.

## 4. Architecture boundaries

Keep the following responsibilities separated. Avoid shortcuts that entangle launcher UI code with runtime internals.

### 4.1 Launcher — `apps/launcher/`

The launcher is a normal Windows desktop application. It owns:

- Game-library entries and user-selected paths.
- Hardware and display capability reporting.
- Per-title profiles and validated user settings.
- Window mode, display resolution, render resolution, output resolution, scaling, and VSync preferences.
- Shader/pipeline preparation queue controls, progress, cancellation, and cache management UI.
- Runtime start/stop requests, logs, compatibility status, and diagnostic export.

The launcher must not directly manipulate guest memory, issue guest GPU commands, or perform executable loading. Those belong to the runtime.

### 4.2 Compatibility runtime — `runtime/`

The runtime owns:

- Read-only inspection and supported executable loading.
- Import routing and Xbox-facing service compatibility.
- Guest virtual-address and page semantics.
- Graphics resource, descriptor, queue, fence, barrier, and presentation translation.
- Shader decode/translation orchestration and runtime compilation.
- Input, audio, storage, and other compatibility services as they are justified by verified title requirements.

Keep platform-specific code behind narrow interfaces where practical. Do not let Qt types leak into runtime libraries.

### 4.3 Shader worker — isolated tool/process where practical

Shader parsing and compilation should be designed as a bounded worker task or separate process. It must have:

- Explicit input and output paths.
- Resource and concurrency limits.
- Cancellation support.
- Structured error reporting.
- No unnecessary network access.
- No permission to modify arbitrary user files.

A separate process reduces accidental coupling but is not, by itself, a complete security sandbox.

### 4.4 Local IPC

Use a versioned local IPC protocol between launcher and runtime/worker. On Windows, named pipes are an acceptable starting point.

- Apply restrictive pipe ACLs to the intended user/session.
- Validate message size, version, enum values, paths, and all variable-length fields.
- Handle disconnects, partial messages, timeouts, stale processes, and malformed requests.
- Do not trust the launcher or any IPC client merely because it is local.
- Never transmit credentials, keys, or protected secrets through logs or diagnostic messages.

## 5. C++ engineering standards

- Target **C++20** unless a documented toolchain constraint requires otherwise.
- Use RAII for handles, COM interfaces, memory, locks, files, and GPU resources.
- Prefer `std::unique_ptr`, `std::shared_ptr` only when shared ownership is real, `std::span`, `std::array`, `std::vector`, `std::string_view`, and explicit value types where appropriate.
- Prefer explicit ownership and lifetime rules over raw owning pointers.
- Do not use `new`/`delete` directly where a standard or project RAII wrapper can express ownership.
- Do not use undefined behavior, unchecked pointer arithmetic, unchecked integer conversions, or unvalidated reinterpretation of untrusted byte buffers.
- Check arithmetic for overflow before computing buffer sizes, offsets, alignment, counts, or address ranges.
- Keep exceptions inside a well-defined C++ boundary. Do not allow exceptions to cross C ABI, Win32 callback, COM, or IPC boundaries.
- Return structured errors (`expected`-style result, status type, or the existing project convention). Do not silently swallow failures.
- Use `enum class`, fixed-width integer types for binary formats, and explicit serialization. Do not serialize raw C++ structs as a stable file or IPC format.
- Avoid global mutable state. Pass dependencies explicitly or use narrow service objects.
- Make threading, cancellation, and shutdown behavior explicit. Avoid detached threads for long-lived work.
- Use `const` by default where it improves correctness. Do not overuse templates or abstractions without a demonstrated benefit.
- Follow the existing formatter and naming convention. If none exists, use consistent `PascalCase` for types and `snake_case` for functions/variables, and document the selected convention before broad reformatting.
- Keep warnings enabled. Do not suppress a warning globally to hide a local problem.

## 6. Memory-model requirements

The Xbox guest memory model and the PC host memory model are not interchangeable. **Do not equate Xbox unified memory with a single PC RAM/VRAM allocation.** Preserve guest-observable semantics while choosing host allocations appropriate to the detected adapter.

The design must keep these concepts separate:

1. Guest virtual address and guest page state.
2. Host CPU virtual mapping and backing allocation.
3. D3D12 resource, heap, descriptor, and GPU virtual address.
4. Residency and adapter-local/non-local memory budget.
5. CPU/GPU ownership, visibility, coherency, and synchronization state.

Implementation rules:

- Never assume the guest's physical memory layout can be copied literally onto a PC GPU.
- Never assume all PC systems have dedicated VRAM; account for UMA/integrated GPUs.
- Preserve guest address identity and aliases where the title can observe them.
- Track page commitment, aliasing, backing storage, lifetime, access permissions, and dirty/visibility state explicitly.
- Use lazy commitment and streaming when correctness permits; do not eagerly allocate an entire guest address range without measurements.
- Respect D3D12 fence and resource-state requirements. Do not use a resource on the GPU before its required data and transitions are complete.
- Use DXGI memory-budget information as advisory runtime data, not as a fixed hardware truth.
- Track bytes copied, duplicate backing, residency, evictions, fence waits, and allocation failures.
- Do not use `memcpy` as a substitute for a correct coherency or synchronization model.
- Provide deterministic tests for aliases, overlapping mappings, page boundaries, read/write visibility, decommit, resource lifetime, and device-removal handling.

A memory optimization is acceptable only if it preserves tested guest-visible behavior. Performance claims must include hardware, driver, title/build, workload, and measurement method.

## 7. Graphics, shader, and pipeline requirements

Treat shader translation as a pipeline with explicit stages:

`input validation → decode → intermediate representation → translation → HLSL validation → DXC compilation → pipeline-state creation → cache`

- DXC compiles HLSL; it is not an Xbox RDNA2 shader decoder. Do not present DXC alone as a complete shader-recompilation solution.
- Keep decoder, translator, HLSL generator, DXC invocation, and D3D12 PSO creation independently testable.
- Validate all offsets, sizes, instruction counts, and resource references before using them.
- Never execute generated or supplied code as part of a parser test.
- Cache entries must be versioned and keyed by all relevant inputs: source content hash, translator version, compiler version/flags, target profile, and relevant adapter/driver identifiers for host-specific compiled artifacts.
- Write cache entries atomically. Detect corruption and safely discard/rebuild invalid entries.
- Use bounded worker concurrency and memory limits. Support cancellation without leaving partially written cache files.
- Record cache hit/miss, decode/translation/compile/PSO timings, and failure reasons.
- Precompile only resources that can be discovered and translated reliably before launch. Keep an on-demand path for shaders and PSOs discovered at runtime.
- Do not claim that every shader can be compiled before game launch.
- Do not silently substitute an incorrect shader when translation fails; report a structured compatibility error or use a documented safe fallback.

## 8. Resolution and presentation requirements

Keep these dimensions distinct in code, settings, and UI:

- **Window size:** dimensions of the application window/client area.
- **Display/output resolution:** dimensions submitted for presentation to the monitor.
- **Internal render resolution:** dimensions used by the game's scene/render targets.
- **Upscaling mode:** spatial or temporal method used to convert render resolution to output resolution.

For example, 1280×720 rendering presented at 1920×1080 is not the same as setting the monitor to 1280×720.

Implementation rules:

- Do not change the Windows desktop resolution as a side effect of selecting an in-game render resolution.
- Do not assume resizing the DXGI swap chain changes the game's internal render targets.
- Track resolution-dependent resources and recreate/resize them at safe points with correct synchronization.
- Update viewport, scissor, render-target dimensions, depth buffers, temporal history, motion vectors, and upscaler inputs consistently.
- Preserve aspect ratio unless the user explicitly selects a stretch mode.
- Keep UI composition and UI scaling separate from 3D scene upscaling where the title's rendering path allows it.
- Validate supported modes against the display and adapter capabilities.
- Handle windowed, borderless, and exclusive fullscreen deliberately; avoid forcing exclusive fullscreen when borderless is more reliable.
- Treat dynamic resolution as a later feature. It requires measured GPU timing and safe render-resource resizing, not just changing a numeric setting.
- Test at least 1280×720, 1600×900, 1920×1080, and the native display mode where supported. Include aspect-ratio and multi-monitor cases.

## 9. Launcher UX and settings

- Keep the launcher responsive while scanning files, detecting hardware, compiling shaders, or launching the runtime. Use background workers and marshal UI updates safely to the UI thread.
- Provide clear status states: not started, scanning, ready, preparing, compiling, launching, running, stopping, failed, and cancelled.
- Show progress only when it can be measured. Do not invent an estimated remaining time when no reliable estimate exists.
- Provide a detailed log view and a concise user-facing error message.
- Validate settings both in the UI and again in the runtime. The runtime is authoritative.
- Store user settings separately from binaries, source files, game files, and shader cache.
- Use a versioned configuration schema and migration tests. Unknown settings should not crash startup.
- Keep advanced settings visibly separate from safe defaults. Explain options that can cause instability.
- Support a reset-to-default action without deleting game files or unrelated user data.
- Do not imply Rockstar Games or Microsoft endorsement. Use project branding and clear unofficial-project wording.

## 10. Security requirements

Treat title files, executable images, shader binaries, textures, archives, configuration files, IPC messages, and cache files as untrusted inputs.

- Validate file sizes, offsets, counts, alignment, encoding, and integer arithmetic before reading or allocating.
- Canonicalize filesystem paths and enforce root containment. Test symlinks, junctions, reparse points, alternate path forms, and traversal attempts.
- Do not load DLLs from the current working directory or an uncontrolled search path. Use explicit, trusted dependency paths and safe Windows DLL search behavior.
- Do not require administrator privileges for normal operation.
- Avoid shell command construction from user-controlled paths or values. Prefer direct process APIs with explicit argument handling.
- Do not execute scripts or plugin code from game directories.
- Apply restrictive ACLs to IPC endpoints and writable runtime directories.
- Keep logs free of credentials, tokens, private keys, and unnecessary personal paths. Let users review diagnostics before sharing them.
- Disable telemetry and network access by default unless a documented feature requires it and the user opts in.
- Fuzz PE, shader, archive, texture, configuration, IPC, and cache parsers with malformed and boundary inputs.
- Add regression tests for every confirmed security bug.
- Do not claim a sandbox exists because the launcher, runtime, or shader worker is a separate process. Any sandbox claim needs an explicit Windows isolation design and security review.

### Supply-chain and release controls

- Pin third-party dependencies and review dependency changes.
- Commit lockfiles where applicable.
- Pin GitHub Actions by immutable commit SHA where practical; use least-privilege workflow permissions.
- Do not run untrusted pull-request code with repository secrets.
- Separate build, test, signing, and release permissions.
- Generate an SBOM and publish hashes for release artifacts.
- Sign release artifacts where signing infrastructure is available. Explain the signing identity and verification method.
- Never treat a successful build as proof that the binary corresponds to the reviewed source. Prefer reproducible builds and document any non-reproducible steps.
- Do not execute downloaded release binaries as part of routine source review.

## 11. Testing and quality gates

Every behavior change should include tests at the narrowest useful level. Do not add tests that merely repeat the implementation without checking expected behavior.

### Required test categories

- **Unit:** configuration validation, address arithmetic, page state, resource tracking, cache keys, shader IR validation, and IPC serialization.
- **Integration:** launcher/runtime startup, process exit, cancellation, IPC disconnect, cache invalidation, and settings migration.
- **Graphics:** shader correctness, resource state transitions, descriptor handling, resize behavior, device removal, and presentation modes.
- **Memory:** aliases, boundary pages, CPU/GPU visibility, residency, budget pressure, and allocation failure.
- **Security:** parser fuzzing, path traversal, reparse-point attacks, malformed IPC, corrupted cache entries, DLL search behavior, and permission checks.
- **Performance:** cold/warm shader preparation, time to first frame, p50/p95/p99 frame time, memory usage, transfer volume, and synchronization stalls.

### Definition of done for a code change

A change is not complete until:

1. The implementation matches the documented responsibility boundaries.
2. Relevant tests are added or updated and pass.
3. Errors and unsupported cases are reported explicitly.
4. No secrets, game files, generated binaries, or personal data are added to the repository.
5. Relevant documentation and configuration examples are updated.
6. Build and test commands, limitations, and any unverified behavior are stated in the final summary.

Do not claim a build or test passed unless it was actually run and the result was observed.

## 12. Build and dependency workflow

First inspect the repository for the actual build system and toolchain files. If the project is still documentation-only, do not invent successful build commands or create large scaffolding without a clear task.

For a CMake-based implementation, the intended Windows developer workflow is:

```powershell
cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Debug
cmake --build build --config Debug
ctest --test-dir build --output-on-failure
```

Use the commands only after the corresponding CMake targets and presets exist. If the project uses `CMakePresets.json`, prefer documented presets. Keep Debug and Release builds distinct.

Before adding a dependency:

1. Explain why the standard library, Windows SDK, or an existing dependency is insufficient.
2. Check license, maintenance status, known vulnerabilities, platform support, and transitive dependencies.
3. Pin a known version and update the lock/manifest.
4. Add a minimal integration test.
5. Document any bundled runtime DLLs and their provenance.

## 13. Git and repository hygiene

- Make focused changes. Avoid unrelated formatting or mass renaming.
- Inspect `git status` before editing and before committing; do not overwrite user changes.
- Do not reset, clean, force-push, or discard user modifications unless explicitly requested.
- Do not commit build directories, IDE state, shader caches, crash dumps, game files, compiled shaders, or generated binaries unless the repository explicitly requires a small test fixture.
- Keep test fixtures synthetic and redistributable. Never include copyrighted title assets.
- Use descriptive commit messages if asked to commit.
- Add or update `.gitignore` when introducing new generated paths.

## 14. Documentation expectations

Update documentation when changing:

- Public settings, configuration schema, IPC messages, or command-line arguments.
- Memory semantics, host allocation strategy, synchronization, or residency behavior.
- Shader input formats, translation stages, compiler flags, or cache keys.
- Resolution, presentation, upscaling, fullscreen, or dynamic-resolution behavior.
- Security boundaries, dependencies, build/release workflow, or data collection.
- Compatibility claims, supported hardware, or known limitations.

Use precise terms. Distinguish **implemented**, **experimental**, **planned**, **unsupported**, and **not tested**. Include evidence for compatibility and performance claims.

## 15. Agent workflow for each task

1. **Inspect:** Read the relevant files, repository status, existing code, and tests. Do not assume the plan documents describe completed code.
2. **Scope:** Identify the smallest subsystem boundary that can satisfy the request. Note security and compatibility implications.
3. **Plan:** For non-trivial work, state a short implementation plan and the files likely to change.
4. **Implement:** Follow the existing patterns, preserve public interfaces unless a change is required, and avoid unrelated refactoring.
5. **Verify:** Run relevant formatting, build, unit, integration, and security tests that are available. Report commands and actual outcomes.
6. **Document:** Update design docs and examples where behavior or project status changed.
7. **Summarize:** List what changed, what was verified, what remains unsupported, and any risks. Be explicit when a test could not be run.

When blocked by missing hardware, proprietary formats, unavailable SDKs, or missing test files, implement safe synthetic tests and clearly mark hardware/title validation as not performed. Never fabricate test results.

## 16. Priority order when trade-offs arise

Use this priority order:

1. Legal compliance and user/system security.
2. Correctness of guest-visible behavior and data integrity.
3. Stability and recoverable failure handling.
4. Reproducibility and testability.
5. Performance measured on representative hardware.
6. Launcher convenience and visual polish.

Do not trade correctness or security for a higher FPS number, a shorter demo, or a more impressive compatibility claim.
