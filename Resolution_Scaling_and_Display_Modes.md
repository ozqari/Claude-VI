# Resolution Scaling and Display Modes — Xbox Series X|S to Windows

**Document type:** Engineering design and implementation plan  
**Target:** A Windows compatibility layer / native PC runtime for Xbox Series X|S titles  
**Primary example:** Render at 1280×720 (720p) while displaying on a 1920×1080 (1080p) or 2560×1440 monitor  
**Recommended implementation language:** C++ for a new native Windows runtime and launcher; preserve the existing implementation language when integrating into an established project such as XCL  
**Status:** Proposed design; not a claim that every feature is already implemented

---

## 1. Executive recommendation

The project should support user-selectable resolutions, but it must not treat “resolution” as one setting. It should expose three separate concepts:

1. **Render resolution** — the dimensions at which the game renders its 3D scene and intermediate render targets. Example: 1280×720.
2. **Presentation resolution** — the dimensions of the final image presented by the Windows runtime. Example: 1920×1080.
3. **Window / desktop resolution** — the size of the game window or the active Windows display mode. Example: a borderless window covering a 1920×1080 desktop.

For the common performance use case, use **1280×720 render resolution → upscale to the monitor's native resolution**. On a 1080p monitor this means 720p rendering followed by a 720p-to-1080p upscale. The Windows desktop can remain at 1080p; the launcher should not need to change the monitor's global display mode.

A simple smaller window is a separate option: a 1280×720 game window on a larger desktop. This changes the window size, but it does not necessarily reduce the game's internal rendering workload unless the game's render targets are also reduced.

**Important:** resizing the game window or changing the swap-chain size alone does not guarantee lower GPU load. To obtain the performance benefit of 720p, the compatibility runtime must make the game render its scene into smaller resolution-dependent resources, or use a game-supported render-scale setting / resolution API. The runtime must not blindly rewrite every texture or viewport size.

## 2. Current XCL baseline and what this plan adds

The public XCL README describes an XCL window and frame presentation through `PresentX`; it also documents a working shader translation/cache pipeline and an FSR 4 console-DLL path tested with a 1280×720 input frame upscaled to 1920×1080 on an RX 9070. It documents game selection when `xcl.exe` is opened from Explorer, but the public README does not describe a general-purpose launcher option that forces every game's 3D render resolution to an arbitrary value.

Therefore, this plan should be treated as a resolution-control design on top of the compatibility layer, not as evidence that an arbitrary game can already be forced to render at 720p. A game-specific FSR path is also not the same thing as universal render-resolution control.

Useful current upstream references:

- XCL repository and current README: <https://github.com/xcl-dev/xcl>
- XCL README's documented 1280×720 → 1920×1080 FSR 4 test, `PresentX`, and shader/pipeline behavior: <https://github.com/xcl-dev/xcl#readme>
- Microsoft Direct3D 12 swap-chain guidance: <https://learn.microsoft.com/en-us/windows/win32/direct3d12/swap-chains>

Re-check upstream documentation and source before implementation because compatibility-layer capabilities change rapidly.

## 3. User-facing resolution modes

The launcher should provide a clear “Display & Graphics” page with independent controls.

### 3.1 Display mode

- **Windowed:** A resizable window. Its client area can be 1280×720, 1600×900, 1920×1080, etc.
- **Borderless fullscreen:** A window covers the selected monitor at its desktop resolution. This is the recommended default for modern Direct3D 12.
- **Fullscreen / display-mode switch:** Optional compatibility mode, exposed only if the runtime and driver path can validate it. Do not assume exclusive fullscreen behaves like older Direct3D versions.

### 3.2 Render resolution

Offer presets and custom values, subject to the game's support and runtime capability:

| Preset | Render size | Typical purpose |
|---|---:|---|
| 720p | 1280×720 | Large performance reduction; useful on weaker GPUs |
| 900p | 1600×900 | Middle ground |
| 1080p | 1920×1080 | Standard full-HD rendering |
| 1440p | 2560×1440 | Higher detail on QHD displays |
| 4K | 3840×2160 | High-resolution rendering |
| Native / Auto | Game or monitor dependent | Do not force a fixed render size |
| Custom | User-defined | Advanced use; validated and clamped by runtime |

The launcher must label this control **Render Resolution**, not merely “Resolution”, so users understand that it can differ from the monitor output.

### 3.3 Output / presentation resolution

Recommended options:

- **Native monitor resolution** (default)
- **Match window size**
- **Custom output resolution** (advanced)

For example, a user can choose 1280×720 rendering and 1920×1080 output. The game draws fewer pixels; a final scaling pass creates the image that fills the 1080p output.

### 3.4 Upscaling and aspect ratio

Provide:

- Off / direct scaling
- Linear or Lanczos-style scaling for a basic fallback
- An integrated upscaler when supported by the title and runtime (for example, an existing FSR implementation)
- Preserve aspect ratio (recommended default)
- Stretch to fill (may distort the image)
- Center / integer scaling (useful for pixel-art titles, usually not for a 3D game)
- Letterbox / pillarbox when aspect ratios differ

Do not promise DLSS, XeSS, or a specific FSR version unless the necessary implementation, GPU support, game inputs, and licensing/distribution requirements have been verified.

## 4. Understand the two common 720p use cases

### Use case A — 720p game window

Example: a 1920×1080 desktop with a 1280×720 window.

- Create or resize the host window to a 1280×720 client area.
- Match the presentation surface to the window client area, or render into an intermediate 1280×720 surface and present it.
- Preserve the game's internal render dimensions where possible.
- This mode does not require changing the Windows desktop resolution.

This is useful for playing in a smaller window, but it only reduces 3D rendering cost if the game's own render targets also become 1280×720.

### Use case B — 720p rendering on a 1080p fullscreen display

Example: a 1920×1080 borderless desktop, with the game scene rendered at 1280×720.

1. Keep the borderless game window and final presentation surface at 1920×1080.
2. Arrange for the game scene's supported resolution-dependent render targets to use 1280×720.
3. Render the scene and any required intermediate passes at the lower resolution.
4. Run the selected upscaling or scaling pass into a 1920×1080 output surface.
5. Render UI at presentation resolution where the title's pipeline makes that possible; otherwise preserve the title's own UI behavior and test readability.
6. Present the 1920×1080 result through the Windows swap chain.

This is the recommended way to get the performance benefit of 720p while filling a 1080p monitor.

### Use case C — 720p output mode

The user may specifically want the final presented image to be 1280×720. The runtime can create a 1280×720 presentation surface or a 1280×720 window. If the user selects fullscreen, behavior depends on the Windows presentation path and supported output modes. Do not silently change the entire Windows desktop mode. Keep display-mode switching optional, validated, and reversible.

## 5. Architecture

Keep resolution responsibilities separated. The launcher sends a validated configuration to the runtime; the runtime owns game-facing display compatibility and actual GPU resource behavior.

```text
┌──────────────────────────────────────────────────────┐
│ Windows launcher                                     │
│ Display mode / monitor / render size / output size   │
│ Upscaler / aspect ratio / UI scale / safe fallback   │
└──────────────────────────┬───────────────────────────┘
                           │ validated launch config
                           ▼
┌──────────────────────────────────────────────────────┐
│ Runtime configuration + display manager              │
│ Monitor enumeration / virtual console mode / resize  │
└──────────────────────────┬───────────────────────────┘
                           │ game-facing compatibility
                           ▼
┌──────────────────────────────────────────────────────┐
│ Xbox graphics compatibility layer                    │
│ D3D12.X resources / command lists / fences / barriers │
│ Track render targets and resolution-dependent usage  │
└──────────────────────────┬───────────────────────────┘
                           │ scene output
                           ▼
┌──────────────────────────────────────────────────────┐
│ Scaling / upscaling stage                            │
│ 1280×720 → 1920×1080 (example)                       │
│ Preserve aspect ratio and color-space correctness    │
└──────────────────────────┬───────────────────────────┘
                           │ final output
                           ▼
┌──────────────────────────────────────────────────────┐
│ Windows presentation                                 │
│ DXGI swap chain / borderless or windowed             │
└──────────────────────────────────────────────────────┘
```

### 5.1 Launcher responsibilities

- Enumerate connected monitors and their supported modes.
- Read each monitor's desktop dimensions and refresh rate.
- Offer separate render, output, and window dimensions.
- Validate that dimensions are positive, within device limits, and supported by the chosen mode.
- Store settings per game profile, not just globally.
- Write a versioned configuration file and pass its path or contents to the runtime.
- Provide a safe startup fallback (borderless, native output, automatic render resolution).
- Offer a confirmation timeout when changing a risky display mode; revert automatically if the user does not confirm.

### 5.2 Runtime responsibilities

- Create the Windows window and DXGI swap chain.
- Maintain a game-facing “virtual display mode” that is consistent with what the compatibility APIs report.
- Track the game's actual render-target dimensions and their usages.
- Apply title-aware render-resolution control only where it is safe and supported.
- Run the scaling/upscaling pass before final presentation.
- Handle resize, Alt-Tab, monitor changes, device removal, and lost/recreated GPU resources.
- Log the requested resolution, effective render resolution, output resolution, and reason whenever a requested value cannot be applied.

## 6. Critical technical rule: do not only resize the swap chain

A game's frame may contain many render targets: scene color, depth, normals, motion vectors, reflection buffers, shadow maps, post-processing buffers, temporal history, UI layers, and temporary compute outputs.

Changing only the final swap-chain buffer from 1920×1080 to 1280×720 can change the presentation size, but the game may still render its 3D scene at 1920×1080. It can therefore produce a smaller output without gaining the expected GPU performance improvement.

Similarly, changing only `RSSetViewports` or scissor rectangles is not a complete render-resolution implementation. In Direct3D 12, viewport and scissor state determine rasterization bounds; they do not reallocate the texture resources bound as render targets. A mismatch can cause clipping, unused areas, incorrect post-processing, or rendering errors.

The implementation must identify the correct point in the game's rendering pipeline where resolution is chosen and ensure that all dependent resources and passes remain consistent.

### 6.1 Preferred order of implementation

1. **Use the game's own setting or supported GDK display/resolution API** when it exists and can be mapped correctly.
2. **Use a title-specific compatibility profile** when the game's resolution selection is known and can be safely adapted.
3. **Add a generic render-graph/resource policy only after instrumentation** can identify resolution-dependent resources and their relationships.
4. If a safe render-resolution override is not supported, expose only window/output resizing and label it honestly. Do not claim that it reduces the 3D render workload.

### 6.2 Resource tracking

For every graphics resource, track at least:

- Resource identity and lifetime.
- Width, height, depth, array size, mip count, format, and sample count.
- Heap type, placement, alignment, and memory visibility properties.
- Resource flags and usage (render target, depth/stencil, UAV, shader resource, copy source/destination, etc.).
- Whether it is the final scene output, an intermediate pass, temporal history, UI, or unknown.
- The game-facing console address and the host D3D12 resource mapping.

Use this information to decide which resources are candidates for render scaling. Never scale arbitrary textures merely because their dimensions happen to match the old output resolution; shadow maps, cube maps, texture atlases, LUTs, and UI atlases may intentionally use independent dimensions.

## 7. DXGI / Direct3D 12 implementation notes

### 7.1 Preferred presentation path

Use a modern flip-model DXGI swap chain for the Windows window. Keep the normal path windowed or borderless and let Windows handle presentation optimizations. The swap-chain back buffer should generally match the final presentation size, not necessarily the game's internal render size.

When the window or presentation dimensions change:

1. Stop submitting work that references the old back buffers.
2. Wait for GPU fences so no in-flight command list still uses those buffers.
3. Release all references to old swap-chain buffers and their views.
4. Call `ResizeBuffers` with the new dimensions and compatible format/flags.
5. Reacquire the buffers and recreate RTV descriptors and any size-dependent presentation resources.
6. Rebuild only resources that actually depend on the changed output size.
7. Resume frame submission.

Microsoft's Direct3D 12 guidance specifically notes that `ResizeBuffers` is required after window/fullscreen transitions and that swap-chain buffer references must be handled correctly. See the references at the end of this document.

### 7.2 Fullscreen behavior

Do not design around the assumption that Direct3D 12 uses classic exclusive fullscreen. Microsoft's D3D12 swap-chain documentation describes fullscreen optimizations and requires buffer resizing around transitions. Borderless fullscreen should be the default because it avoids changing the user's global desktop mode and is generally easier to recover from.

If an optional mode-switch path is implemented, enumerate supported monitor modes, validate the selected width/height/refresh rate, provide a timed confirmation, and restore the previous state on failure or timeout. Never hard-code a refresh rate that may not be supported by the selected monitor.

### 7.3 Viewport and scissor state

When rendering into a resource of a particular size, set viewport and scissor values consistently with the intended rendering rectangle. For a 1280×720 render target, a full-target viewport and scissor rectangle would cover 1280×720. The runtime should not overwrite game command-list state blindly; it should translate or adapt the game's state only when its resource remapping strategy requires it.

Microsoft references:

- `RSSetViewports`: <https://learn.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12graphicscommandlist-rssetviewports>
- `RSSetScissorRects`: <https://learn.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12graphicscommandlist-rssetscissorrects>

## 8. Upscaling pipeline

### 8.1 Simple spatial scaling fallback

Implement a simple final-pass scaler first. It should:

- Read the final scene image at render resolution.
- Write to the presentation-resolution target.
- Preserve the intended aspect ratio.
- Handle the source and destination formats and color spaces explicitly.
- Avoid reading and writing the same resource in an invalid state.
- Use the appropriate D3D12 resource barriers and synchronization.

This is a fallback for compatibility and testing, not necessarily the best image-quality option.

### 8.2 Temporal or game-integrated upscalers

Temporal upscalers generally need more than a color image: they may require motion vectors, depth, jitter information, exposure, and temporal history. The runtime must not inject a temporal upscaler as though it were a simple resize operation.

Use the game's own integrated upscaler when available and supported. XCL's public README currently documents an FSR 4 console-DLL integration test with a 1280×720 input and 1920×1080 output. This is a useful proof that one real game-provided upscaling path can work, but it does not prove that the runtime can force arbitrary titles to use arbitrary render resolutions.

For an independent integration, use official vendor documentation and follow the full resource/input contract. AMD's FidelityFX documentation distinguishes render-resolution inputs from presentation-resolution outputs and lists temporal inputs such as current/previous-frame resources for temporal upscaling.

References:

- AMD FidelityFX Super Resolution overview: <https://gpuopen.com/fidelityfx-superresolution/>
- AMD FSR upscaler integration documentation: <https://gpuopen.com/manuals/fidelityfx_sdk/techniques/super-resolution-upscaler/>

### 8.3 UI composition

Preferred order, when the game's rendering architecture allows it:

1. Render 3D scene at the selected render resolution.
2. Upscale the 3D scene to presentation resolution.
3. Composite UI at presentation resolution.
4. Present the final frame.

Some titles render UI into the same target as the 3D scene or use game-specific post-processing. Do not assume UI can always be separated. For those titles, preserve original behavior and test HUD size, subtitles, menus, reticles, and safe areas at every supported scale.

## 9. Configuration format

Use separate values so that a launcher setting cannot be misinterpreted by the runtime.

Example `display.json`:

```json
{
  "schema_version": 1,
  "display": {
    "mode": "borderless",
    "monitor": "primary",
    "window_width": "auto",
    "window_height": "auto",
    "output_width": "native",
    "output_height": "native",
    "refresh_rate": "auto",
    "vsync": true
  },
  "rendering": {
    "render_resolution_mode": "custom",
    "render_width": 1280,
    "render_height": 720,
    "resolution_override": "game_api_or_title_profile_only",
    "upscaler": "auto",
    "scaling_filter": "linear",
    "aspect_policy": "preserve",
    "ui_scale": 1.0
  },
  "recovery": {
    "confirm_display_change": true,
    "revert_after_seconds": 15,
    "fallback_mode": "borderless_native"
  }
}
```

The strings `native` and `auto` are symbolic values, not numeric dimensions. Validate and resolve them before creating graphics resources. The example is a target configuration schema; implementation names may differ.

### 9.1 Validation

- Reject zero, negative, malformed, or unreasonably large dimensions.
- Apply documented device and runtime limits.
- Ensure render and output aspect ratios are handled intentionally.
- Validate refresh-rate numerator/denominator rather than assuming an integer such as 60 Hz.
- Record both requested and effective settings in the run report.
- If a game or GPU path rejects the override, fall back to a safe mode and explain why.

## 10. Shader and pipeline cache behavior

Resolution changes should not automatically trigger recompilation of every translated shader. Shader translation and resolution-dependent render-resource configuration are different concerns.

- Keep the translated shader cache keyed by shader identity and translator/toolchain version.
- Keep pipeline-state cache entries keyed by their actual pipeline-defining state, including shader hashes, formats, sample count, root signature, and other relevant PSO fields.
- Store resolution-dependent constants and render-resource metadata separately where possible.
- If a specific upscaler generates size-specific pipelines or resources, include the relevant render/output dimensions and upscaler version/settings in that component's cache key.
- Reuse compatible driver pipeline blobs when the driver and adapter identity match; invalidate them when the toolchain, driver, or relevant pipeline state changes.
- Do not precompile every possible width/height combination. Precompile shader programs and known PSOs; allocate size-dependent resources at runtime.

XCL already documents a shader translation cache and pipeline precompilation. The new resolution system should integrate with that cache rather than duplicating it.

## 11. Dynamic resolution (later milestone)

Dynamic resolution should not be the first implementation. First make fixed render sizes correct and stable.

Later, add an optional controller that adjusts the render target within a user-defined range based on measured GPU frame time. Requirements:

- Use GPU timestamps or reliable frame timing, not CPU frame time alone.
- Set a target frame time based on the selected frame cap (for example, 16.67 ms for 60 FPS).
- Use a moving average and hysteresis to prevent rapid resolution oscillation.
- Change resolution in small increments and impose a cooldown between changes.
- Keep the presentation resolution constant.
- Reallocate or recycle size-dependent resources safely after GPU fences complete.
- Reset temporal upscaler history when required after a discontinuous size change.
- Expose min/max render scale and an enable/disable switch in the launcher.

## 12. Testing matrix

Do not consider the feature complete after a single 720p screenshot. Test the display and rendering paths independently.

| Test | Render size | Output size | Expected result |
|---|---:|---:|---|
| A. 720p window | 1280×720 | 1280×720 | Correctly sized window; no clipping |
| B. 720p to 1080p | 1280×720 | 1920×1080 | Fullscreen 1080p output; 16:9 preserved |
| C. 720p to 1440p | 1280×720 | 2560×1440 | Correct upscale, no stretched aspect ratio |
| D. Native 1080p | 1920×1080 | 1920×1080 | Baseline image and performance |
| E. 900p to 1080p | 1600×900 | 1920×1080 | Correct scaling and UI behavior |
| F. Ultrawide output | 1280×720 | 2560×1080 | Letterbox/pillarbox policy applied correctly |
| G. Window resize | Fixed render target | Variable window | No stale back-buffer references or crash |
| H. Alt-Tab / focus loss | Fixed | Fixed | Stable resume, no black screen |
| I. Monitor change | Fixed | New monitor output | Re-enumeration and safe swap-chain recreation |
| J. Unsupported mode | Requested invalid size | Safe fallback | Clear error and restoration |
| K. Repeated launch | 1280×720 | 1080p | Shader and pipeline cache reused correctly |
| L. Dynamic-resolution stress | Variable | Fixed | No oscillation, corruption, or resource lifetime errors |

For every test, capture:

- Requested render and output dimensions.
- Actual scene color/depth target dimensions.
- Swap-chain buffer dimensions and format.
- GPU frame time and CPU frame time.
- GPU memory and system committed memory.
- Number and size of copies between resources/heaps.
- Shader compile and PSO creation counts.
- Frame captures before and after scaling.
- Error/device-removed logs.

Compare the 720p render path with native resolution to verify that the actual 3D render targets became smaller and GPU work fell. Do not use FPS alone as proof that the resolution override worked.

## 13. Development milestones

### Milestone 1 — Display settings and window management

- Add a Display & Graphics page to the launcher.
- Enumerate monitors and display modes.
- Support windowed and borderless modes.
- Add independent window and output dimensions.
- Implement safe swap-chain resizing and fallback behavior.

**Exit criteria:** The user can select a 1280×720 window and return to the previous mode without crashes or a black screen.

### Milestone 2 — Separate render and output resolution

- Add independent render and presentation dimensions to the runtime configuration.
- Add a 1280×720 intermediate scene-output test path.
- Add a final scaling pass to a 1920×1080 swap-chain buffer.
- Verify aspect ratio, formats, and synchronization.

**Exit criteria:** A controlled test scene renders at 1280×720 and is presented at 1920×1080, without changing the Windows desktop mode.

### Milestone 3 — Instrument the real game's render targets

- Log all texture/resource creation and dimensions.
- Identify scene color, depth, motion vectors, temporal history, UI, and post-process targets.
- Identify which resolution decisions are controlled by the title and which are exposed by supported console interfaces.
- Build title-specific profiles only from verified data.

**Exit criteria:** The team can identify the game's true internal render resolution and its resolution-dependent resource graph.

### Milestone 4 — Safe render-resolution override

- Prefer the game's own supported resolution setting/API.
- Implement a per-title override only where its full render-resource chain is understood.
- Keep unrelated textures and independent targets unchanged.
- Add fallback to native/game-selected resolution if validation fails.

**Exit criteria:** Selecting 1280×720 actually changes the intended scene render targets to 1280×720 and does not break depth, post-processing, motion vectors, or UI.

### Milestone 5 — Upscaler integration

- Use the game's integrated upscaler if compatible.
- Otherwise integrate a supported spatial scaler first, followed by a temporal upscaler only when required inputs are available.
- Implement correct history resets, jitter, motion-vector, depth, exposure, color-space, and resource-state handling as required by the chosen upscaler.

**Exit criteria:** 720p rendering produces a stable 1080p output with acceptable image quality and no temporal artifacts beyond the chosen algorithm's known limitations.

### Milestone 6 — Performance and dynamic resolution

- Benchmark 720p, 900p, 1080p, and native modes.
- Measure GPU time, memory pressure, and transfer overhead.
- Add optional dynamic resolution only after fixed modes pass the full test matrix.

**Exit criteria:** Results are repeatable, measurable, and do not cause resource churn or frame-time spikes during mode changes.

## 14. Recommended launcher layout

**Display**

- Display monitor: Primary / Monitor 2 / Monitor 3
- Window mode: Windowed / Borderless / Optional display-mode switch
- Window size: Match render / Custom
- Output resolution: Native / 720p / 900p / 1080p / 1440p / 4K / Custom
- Refresh rate: Auto / Supported values
- VSync: On / Off

**Rendering**

- Render resolution: Game default / 720p / 900p / 1080p / 1440p / Custom
- Upscaler: Auto / Off / Supported implementations
- Upscaler quality: Quality / Balanced / Performance / Custom (only if the selected upscaler supports these modes)
- Aspect ratio: Preserve / Stretch / Letterbox
- UI scale: 75% / 100% / 125% / 150% / Game default
- Dynamic resolution: Off / On (later milestone)

Display effective values below the controls, for example:

```text
Requested render resolution: 1280×720
Effective render resolution: 1280×720
Presentation resolution:      1920×1080
Window mode:                  Borderless
Upscaling:                    FSR (title-integrated path)
```

If the runtime cannot safely apply the render override, show the actual value instead of pretending the requested setting took effect.

## 15. Risks and mitigations

| Risk | Mitigation |
|---|---|
| Only the swap chain changes; game still renders at native resolution | Instrument scene render targets and verify real internal dimensions |
| Arbitrary resource resizing breaks the game | Use title-aware/resource-aware policies; never resize all textures by dimension alone |
| UI becomes blurry or misaligned | Preserve or separately composite UI where supported; test menus/HUD/subtitles |
| Temporal upscaler produces artifacts | Supply correct motion vectors, depth, jitter, exposure, and history; otherwise use spatial scaling |
| Resize while GPU work is in flight | Use fences and release back-buffer references before `ResizeBuffers` |
| Unsupported refresh rate or mode causes black screen | Enumerate modes, use safe defaults, and implement timed revert |
| Shader/PSO cache grows for every resolution | Separate shader identity from runtime dimensions; cache only state that actually depends on size |
| Resource memory spikes during resizing | Reuse pools where safe; wait for fences; release obsolete size-dependent allocations |
| Game reports a display size inconsistent with actual output | Maintain a coherent virtual display state and validate game-facing API results |

## 16. Definition of done

The feature is complete only when all of the following are true:

- The user can select 720p, 900p, 1080p, 1440p, and a validated custom resolution where supported.
- Render resolution, output resolution, and window size are distinct settings.
- 720p can be used as an actual internal scene resolution—not merely a smaller window or swap chain—on at least one validated title path.
- 720p can be upscaled to a higher-resolution output without changing the Windows desktop mode.
- Windowed and borderless modes survive Alt-Tab, resize, and monitor changes.
- Aspect ratio, UI readability, and color-space behavior are verified.
- Shader/pipeline caches remain correct and do not rebuild unnecessarily for simple window-size changes.
- Requested versus effective settings are logged and visible to the user.
- Unsupported per-title overrides fail safely and revert to game/default resolution.
- Automated tests cover resource lifetime, swap-chain resize, invalid settings, and repeated mode changes.

## 17. Sources and further reading

The following are primary project or vendor documentation references. Re-check them when implementation begins because upstream documentation and runtime capabilities can change.

1. **XCL — Xbox Series X|S to Windows compatibility layer:** <https://github.com/xcl-dev/xcl>  
   Current repository README documents the XCL window, `PresentX`, shader/pipeline cache and precompilation, and the FSR 4 console-DLL test from 1280×720 to 1920×1080.
2. **Microsoft — Direct3D 12 Swap Chains:** <https://learn.microsoft.com/en-us/windows/win32/direct3d12/swap-chains>  
   Swap-chain behavior, buffer lifetime, fullscreen optimizations, and the need to resize buffers after relevant transitions.
3. **Microsoft — IDXGISwapChain::SetFullscreenState:** <https://learn.microsoft.com/en-us/windows/win32/api/dxgi/nf-dxgi-idxgiswapchain-setfullscreenstate>  
   Fullscreen state and Microsoft recommendations for windowed swap-chain creation and safe mode selection.
4. **Microsoft — DXGI Best Practices:** <https://learn.microsoft.com/en-us/windows/win32/direct3darticles/dxgi-best-practices>  
   Window resizing, mode transitions, `ResizeBuffers`, and refresh-rate considerations.
5. **Microsoft — ID3D12GraphicsCommandList::RSSetViewports:** <https://learn.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12graphicscommandlist-rssetviewports>  
   D3D12 viewport state.
6. **Microsoft — ID3D12GraphicsCommandList::RSSetScissorRects:** <https://learn.microsoft.com/en-us/windows/win32/api/d3d12/nf-d3d12-id3d12graphicscommandlist-rssetscissorrects>  
   D3D12 scissor state.
7. **AMD GPUOpen — FidelityFX Super Resolution:** <https://gpuopen.com/fidelityfx-superresolution/>  
   Spatial upscaling overview.
8. **AMD GPUOpen — FSR upscaler integration:** <https://gpuopen.com/manuals/fidelityfx_sdk/techniques/super-resolution-upscaler/>  
   Render-resolution versus presentation-resolution inputs and temporal-resource requirements.

---

## Final design decision

Implement **720p render resolution with native-resolution borderless output** as the primary performance feature. Keep the Windows desktop resolution unchanged. Start with a safe final-pass scaler and display/window management, then add true internal render-resolution control only through a verified game setting, supported display API, or title-aware resource policy. Treat arbitrary global resizing of game render targets as unsafe and out of scope until the resource graph is understood.
