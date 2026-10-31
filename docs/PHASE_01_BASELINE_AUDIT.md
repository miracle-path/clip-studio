# Phase 01 — Baseline Audit

- Repository: `miracle-path/clip-studio`
- Branch: `phase-01-core`
- Baseline source: `Clip_Maker_Enhanced.html`
- Baseline source blob SHA: `d02793fb568414fc8c545e557ac7fc4b124b86c6`
- Source size at baseline: 102,974 bytes / 882 lines
- Audit type: user-reported browser baseline + static source inspection
- Browser/runtime execution by the architect: NOT performed
- Source code modifications: NONE

## 1. User-reported baseline result

Passed: 3/10
- Page opens normally.
- Sample JSON loads.
- Timeline can change scenes.

Failed / not working as reported:
- Characters and scenes do not display completely.
- Play does not animate.
- Camera and effects do not work.
- Text and subtitles do not display correctly.
- Pause and Stop do not work.
- Video export does not work.
- Console is not clean (user marked the no-red-errors check as failed).

Important test correction:
- The current UI contains Play, Stop, and Export buttons. It has no Pause button and no pause handler. Therefore Pause is currently an absent feature, not a verified broken control. Stop must be retested after the Play/runtime error is identified.

## 2. Confirmed static findings from source

1. `normalizeSceneData()` mutates scene objects and applies defaults before `build()` resolves `cont` inheritance. This can block intended inheritance of theme, sky, ground, and atmosphere fields.
2. `inferTheme()` returns `custom` even when no theme can be inferred. Since this value is truthy, later `if (!s.theme ...)` inheritance branches cannot resolve the missing theme.
3. Empty arrays such as `sky: []` are truthy in JavaScript and are retained by `sc.sky = sc.sky || ...`; they are not treated as missing/invalid.
4. Scene normalization runs outside the JSON.parse try/catch. A malformed root, null scene, or invalid nested structure can throw without a controlled build error.
5. Timeline weights are summed with `s.w || 1`. A numeric zero is converted to 1 (so zero does NOT cause division by zero); a truthy malformed value such as a nonnumeric string can poison the sum and produce invalid scene durations.
6. The duration calculation enforces a hard 60-second minimum using `Math.max(60, ...)`, despite the project decision that 60 seconds is a default rather than a hard minimum.
7. The playback loop calls `frame(time)` without an exception boundary. A renderer exception can prevent the next animation frame from being scheduled.
8. There is no Pause button or pause function in the current HTML/JS.
9. Export creates `MediaRecorder`, starts it, and downloads a Blob without a surrounding error boundary or a recorder `onerror` handler. The selected MIME type may be an empty string when none of the tested formats is supported; this path needs browser verification and a deliberate fallback.
10. Export creates an object URL but does not revoke it after download.

## 3. Findings not yet confirmed

- Exact first Console exception and its source line.
- Whether the first runtime exception occurs during initial `frame(0.5)`, Build, Play, or a particular renderer.
- Whether canvas rendering is failing globally or only specific scene/actor/theme paths.
- Whether the user's export failure is caused by MIME support, MediaRecorder construction/start, browser security, or an earlier render failure.
- Whether subtitle/font appearance is a rendering failure or a data/visibility issue.

These must not be guessed. Obtain the first red Console error and reproduce the smallest failing path before patching.

## 4. Phase 01 implementation order

### P0 — Find and contain the first runtime failure
- Capture the first Console error, stack, and line number.
- Add controlled error reporting around normalization/build and per-frame rendering.
- Keep the last valid frame visible when rendering fails.
- Do not silently skip a failed scene.

### P1 — Repair input pipeline
- Validate root JSON type and scene array.
- Normalize aliases without injecting defaults.
- Validate/skip malformed scene entries with warnings.
- Resolve inheritance only when `cont: true`.
- Apply defaults after inheritance.
- Validate actors, props, crowd, sky, colors, and finite positive weights.
- Preserve v1 `char`, `sign`, and `badge` behavior.

### P2 — Repair timeline and playback lifecycle
- Apply duration precedence: valid UI duration > valid JSON duration > 60-second default; no hard minimum.
- Guard empty/invalid timelines.
- Make Play/Stop state transitions reliable.
- Add Pause only as an explicit UI feature if approved; do not pretend it already exists.

### P3 — Repair export lifecycle
- Check `captureStream`, MediaRecorder availability, and supported MIME types.
- Try supported codecs in order; if none are supported, report a clear error.
- Catch constructor/start/stop errors and handle recorder error/stop/data events.
- Stop media tracks and revoke the download object URL after use.

### P4 — Regression fixtures
- Valid v1 legacy sample.
- Valid v2 scenes/cast sample.
- Empty root, wrong root type, empty scenes.
- Null scene and malformed actor/props/crowd.
- Invalid/zero/string/negative weights.
- Missing/empty sky and partial atmosphere.
- Inheritance with and without `cont: true`.
- Invalid colors and missing assets.
- Playback boundary and export capability failures.

## 5. Acceptance criteria

- Valid baseline sample renders a visible first frame and plays through all scenes.
- Build failures show actionable messages and do not crash the page.
- Invalid scenes are reported and handled according to the validation policy.
- Inheritance works only when `cont: true`.
- 30-second and other positive custom durations are allowed; 60 seconds remains the fallback default.
- Playback errors preserve the last valid frame and display the scene/time context.
- Stop works during playback and recording.
- Export either creates a valid playable file or reports the exact unsupported/failure reason without leaving the UI stuck.
- No unhandled Console errors in the agreed regression fixture set.

## 6. Immediate user evidence required

Before changing renderer logic, provide a screenshot or copy/paste of the FIRST red Console error:
1. Open the app in Chrome/Edge.
2. Press F12 → Console.
3. Clear the Console.
4. Reload the page.
5. Copy the first red error and its clickable source line/stack.
6. If reload has no error, click Play once and copy the first red error.

Do not send only the last error if multiple errors appear. The first exception is the priority because later failures may be cascading symptoms.
