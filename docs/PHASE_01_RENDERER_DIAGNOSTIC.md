# Phase 01 — Renderer Diagnostic & Source Change Plan

**Branch:** `phase-01-core`  
**Source reviewed:** `Clip_Maker_Enhanced.html`  
**Source commit at review:** `d02793fb568414fc8c545e557ac7fc4b124b86c6`  
**Source SHA:** `d02793fb568414fc8c545e557ac7fc4b124b86c6`  
**Review type:** Static source inspection through GitHub. No browser/runtime test was performed.

## 1. Current baseline

The corrected user-reported baseline is **6/10 passed**:
- PASS: page opens, sample JSON loads, timeline changes scenes, Pause/Stop (as user reported), export, Console has no red errors.
- FAIL: complete character/scene display, Play animation, camera/effects, text/subtitles.

Important discrepancy retained from the audit: the checked-in HTML has Play, Stop and Export controls, but no Pause control/handler was found in the exact source inspected. The user's Pause/Stop pass result is recorded as observed, but it is not attributed to this source until the tested file/version is confirmed.

## 2. Confirmed source findings

### R01 — Theme renderer registry is incomplete
- `THEMES` at line 324 registers only `road, room, city, nature, space, custom`.
- Dedicated renderer functions exist for `ring` (line 626) and `lake` (line 423), but neither is registered.
- `frame()` at line 715 dispatches with `(THEMES[sc.theme]||custom)`, so a scene with `theme:"ring"` or `theme:"lake"` silently renders using `custom`, not its dedicated renderer.
- This is a confirmed scene fidelity defect for those themes. It may explain incomplete/mismatched environments when those themes are used; it does not by itself explain all four failed baseline items.
- Proposed fix: register supported renderers explicitly, after verifying all supported theme names and their function signatures. Also add a theme-registry test.

### R02 — Camera follow reads actor positions from the previous frame
- `frame()` computes camera follow from `POSG[c.follow]` before it calls `actors()`.
- `actors()` refreshes `POSG` at line 266.
- Therefore follow camera uses the previous rendered frame's actor position (one-frame lag); on the first frame it may have no position to follow.
- Proposed fix: Phase 02 architecture work should compute actor frame state before camera transform, or otherwise resolve follow target positions from the current scene/time without mutating render order. Do not patch by simply moving draw calls across camera transforms.

### R03 — DOF filter is applied only to background and back props
- `frame()` sets `g.filter='blur(...)'` before theme rendering and back props, then resets it to `none` before drawing actors.
- The current behavior does not provide a general depth-of-field treatment across scene layers. Confirm intended visual contract before changing; likely belongs with renderer/camera improvements rather than a blind filter move.

### R04 — Normalization and build errors are not handled as one guarded operation
- JSON.parse is inside a try/catch, but `normalizeSceneData(j)` and subsequent build processing occur outside that catch.
- Invalid structure such as a non-array `scenes`, malformed `actors`, or non-object root may throw instead of producing a field-specific validation message.
- Proposed fix: Phase 01 validation pipeline: parse → root validation → alias normalization → inheritance → defaults → validation → timeline compilation. Catch and report errors with scene/field context.

### R05 — Duration is hard-clamped to 60 seconds
- UI says “ให้ AI กำหนด (ไม่ต่ำกว่า 60 วินาที)” at line 62.
- `build()` uses `Math.max(60,...)` at line 412.
- This conflicts with the locked Blueprint decision: 60 seconds is the default, not a hard minimum.
- Proposed fix: accept a valid positive UI duration, else a valid positive JSON duration, else default to 60; add a custom-duration input if the current selector cannot express shorter clips.

### R06 — Subtitle drawing path exists, but requires targeted data-flow verification
- `subtitle()` exists at line 361.
- `frame()` draws it only when `oSub` is checked and `sc.tx` is truthy (line 719).
- `build()` carries text/highlight and assigns `tx/hl/ts` to each scene (line 414).
- Static inspection confirms the path exists; it does not establish why the user's subtitles appear incorrectly. Next inspect actual tested JSON fields, text carry-forward semantics, and timing/segment layout before changing it.

### R07 — Character renderer path exists, but actor data must be validated
- `normalizeSceneData()` maps each actor's type/action/position at lines 389–393.
- `actors()` resolves cast data and dispatches SVG, critters, cats/dogs, humans/zombies, kangaroos, robots and beasts at lines 264–315.
- Static source alone does not prove why the user's characters are missing or incomplete. Need reproduce using the exact JSON the user tested and compare actor `c/who/x/y/act/keys` against renderer support. Do not redesign the character engine before that.
- Note: actor action timing mixes scene progress `k` for position/keyframes with clip-global `t` for cyclic motion. Verify whether this is intentional for each action before changing it.

## 3. Prioritized implementation plan

### Phase 01A — Input and scene integrity
1. Add safe root/scenes/actors/props/crowd validation.
2. Normalize aliases without inserting defaults prematurely.
3. Resolve `cont:true` inheritance field-by-field, then apply defaults.
4. Validate positive finite scene weights and duration.
5. Convert thrown build errors into actionable Thai messages.

### Phase 01B — Renderer dispatch and visual data
1. Build an explicit registry of every supported theme and validate unknown themes.
2. Register existing dedicated `ring` and `lake` renderers where their contracts match.
3. Verify map/photo/custom renderer dispatch separately; do not register by guess.
4. Validate actor species/action support and SVG asset readiness; report unsupported/missing actors visibly.
5. Add subtitle fixture cases for Thai wrapping, highlight ranges, scene changes, and empty text.

### Phase 02 — Animation and camera architecture
1. Separate time-derived actor state from drawing.
2. Resolve camera follow using current-frame actor state.
3. Define action timing semantics (scene-local vs clip-global) and test walk/run/punch/jump/keyframes.
4. Define camera/DOF layer behavior and regression tests.
5. Consider deterministic procedural randomness for stable frames/export.

## 4. Required reproduction input before patching visual failures

The four reported failures are broad symptoms. To avoid guessing, use the exact JSON currently pasted in the app:
- Copy all contents of section 2 (JSON).
- Paste it into the next chat message or attach it as a `.json` file.
- Do not modify it before sending.

With that exact fixture, inspect the actual scene themes, actor IDs/types/actions, camera keys, text/highlight fields, and timeline weights. Then make the smallest source patch and add a matching regression fixture.

## 5. Safety and scope

- Do not modify `main`.
- Do not replace the application or rewrite the renderer.
- No source code has been modified as part of this diagnostic.
- No browser/runtime execution is claimed.
- Work only on `phase-01-core`; make incremental commits and re-fetch/verify each changed file.
