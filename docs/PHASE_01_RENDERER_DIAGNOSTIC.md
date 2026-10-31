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

### R01 — Nested shot schema is not consumed (confirmed primary root cause)
- The supplied JSON stores every event under `scenes[].shots[]`; the current normalizer only resolves the top-level `j.scenes` array (lines 375–378) and iterates those outer scene objects (line 381). It does not flatten or compile nested `shots`.
- `build()` then compiles each outer scene as one timeline item (lines 411–415). Since the outer objects have no `actors`, `props`, or `text` fields, all shot-level actions, props, and text are omitted from playback.
- The outer scene backgrounds still render, which can make the output look like static environments with missing characters/actions.
- The user's shot-level `dur` is not read. The existing duration alias only checks `sc.duration` (line 385), and timeline weights use `s.w||1` (line 412). The supplied 24 shots therefore do not become 24 timeline items.
- Confirmed consequence: shot-level camera, actors, props, transitions, and text are not compiled into the current timeline. This directly explains why the story does not play as authored and is the first issue to fix.
- Proposed fix: add a deliberate scene/shot compiler. Each nested shot becomes a timeline item; inherit scene-level theme/sky/ground/decor/lights/atmosphere into its shots; map `dur` to a validated duration weight; map `text.t` and `text.h` to the renderer's text/highlight fields; preserve shot-level `cam`, `tr`, `actors`, `props`, and `gfx`. Do not mutate the original authoring JSON; compile to an internal representation.

### R02 — Theme renderer registry is already populated for this fixture
- Correction to the earlier preliminary note: the initial registry is declared at line 324, and additional renderers are registered at line 437: `map, photo, lake, shophouse, yamo, ring, arena`.
- Therefore `yamo`, `shophouse`, and `ring` in the supplied JSON are registered. Theme registration is **not** the root cause for this fixture.

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

## 6. Exact supplied JSON fixture — confirmed mapping gaps

The user supplied a seven-scene JSON with 24 nested shots and approximately 44.5 seconds of declared shot durations.

| Authoring JSON field | Current runtime expectation | Result |
|---|---|---|
| `scenes[].shots[]` | top-level `scenes[]` compiled directly | Nested shots ignored |
| `shots[].dur` | `scene.duration` / `scene.w` | Shot duration ignored |
| `shots[].text.t` and `text.h` | compiled `sc.tx` / `sc.hl` | No subtitle for these shots |
| `shots[].actors` | `scene.actors` | Characters/actions ignored |
| `shots[].props` | `scene.props` | Shot props ignored |
| root `crowd` | `scene.crowd` | Root crowd ignored |
| `shots[].gfx` | `scene.gfx` compiled to `sc.gx` | Shot graphics ignored |

Supported items in this fixture:
- Scene themes `yamo`, `shophouse`, `ring`, `road`, and `custom` are registered.
- Actions including `run`, `jump`, `guard`, `punch`, `hit`, `kick`, `ko`, `sit`, `cheer`, `dance`, `wave`, `point`, `hold`, `cast`, and `channel` have branches in the current actor renderer.
- Props `spark`, `aura`, `beam`, and `songthaew` have renderer branches.
- Thus these actor/action/prop types are not the primary failure in this fixture; they are currently nested one level too deep for the runtime compiler to see.

Duration note: the supplied 24 shot durations sum to approximately 44.5 seconds. Current build logic clamps total duration to at least 60 seconds (line 412), independently of the nested-shot issue. This conflicts with the locked decision that 60 seconds is a default, not a hard minimum.

## 7. Revised immediate implementation order

1. Implement and unit-test the nested shot compiler first, preserving scene-level visual context and shot-level actions.
2. Map the text object contract and graphics object contract explicitly.
3. Move root-level crowd data into compiled shots/scenes according to a documented inheritance rule.
4. Fix duration semantics so a valid explicit duration is honored and the default is 60 seconds only when no valid duration is supplied.
5. Then run the supplied JSON as a regression fixture and verify each of its 24 shots is represented in the compiled timeline.
6. Only after timeline compilation is correct, tune character posing, camera motion, visual effects, and subtitle layout.


## 8. Implementation update — nested-shot compiler

Source changes were committed to `phase-01-core`:
- Commit `10201036f8b280405fdd6c541cecb4ecdd5a3253`: flatten nested `scenes[].shots[]`, preserve parent scene context, map `dur` to timeline weight, convert `text.t/text.h` into renderer text/highlight fields, apply root crowd data, infer total duration from valid shot durations, and remove the hard 60-second clamp in favor of UI duration → JSON duration → 60-second default.
- Commit `916b1ad3314d816484d9b493fdde6f3804781d3f`: ensure inherited object-form text is normalized too.
- Latest source blob SHA at time of verification: `fd85bc52481e625872f2a13d33193883fbbebd7b`.
- Latest source size/shape before the final one-line condition correction: 104,184 bytes / 905 lines. The final correction changes only the condition text; re-fetch is required for the exact final byte/line count.

**Verification status:** source was re-fetched after commits and the modified normalization/build sections were inspected. This is static source verification only. No browser playback, pixel comparison, console test, or export test has been run. Do not mark the user's four failed baseline tests as passed yet.

**Known remaining work:** add robust validation for malformed scenes/weights/actors; verify actual playback using this exact fixture; inspect the character/action timing, camera follow, subtitle placement and FX only after confirming all 24 shots reach the timeline.

## 9. Safety and scope

- Do not modify `main`.
- Do not replace the application or rewrite the renderer.
- Source code has now been modified incrementally on `phase-01-core`; see the implementation update above.
- No browser/runtime execution is claimed.
- Work only on `phase-01-core`; make incremental commits and re-fetch/verify each changed file.
