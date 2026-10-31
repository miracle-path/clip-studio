# Clip Maker — Phase 00 Blueprint
## Unified Product, Architecture & Visual Engine Blueprint

| Item | Value |
|---|---|
| Status | Phase 00 — Blueprint / design baseline |
| Date | 2026-10-31 |
| Repository | \`miracle-path/clip-studio\` |
| Primary application | \`Clip_Maker_Enhanced.html\` |
| Audited baseline branch | \`main\` |
| Audited baseline commit | \`d02793fb568414fc8c545e557ac7fc4b124b86c6\` |
| Baseline file size | 102,974 bytes |
| Baseline file length | 882 lines |
| Runtime target | Single HTML file; Vanilla JavaScript; Canvas 2D + SVG |
| Installation | None |
| Primary browsers | Current Chrome and Edge |
| Blueprint owner | ChatGPT (lead architect / integrator) |

> This document is the single planning source of truth for the next implementation phases. It combines the legacy-source audit, the DeepSeek Animation Core handoff, the Gemini Visual Engine specification, and the user's locked workflow. It is a design document, not a claim that the proposed modules have already been implemented or browser-tested.

---

## 1. Product Goal

Evolve the existing Clip Maker into a dependable procedural animation studio that turns an idea or compatible JSON sequence into a coherent vertical short film.

The finished application should provide:

- Believable, species-appropriate character motion with clear physical cause and effect.
- Cinematic shot composition, camera movement, continuity, and readable action.
- Richer environments with depth, layered set dressing, lighting, atmosphere, and controlled VFX.
- A consistent authoring contract that AI assistants can follow without inventing unsupported fields or impossible actions.
- Reliable preview, seeking, playback, and export.
- A usable single-file HTML workflow that requires no installation.

### Visual target and honest boundary

The target is **high-quality procedural 2D / 2.5D animation**, using Canvas 2D and SVG. Parallax, layered shading, articulated rigs, perspective cues, light falloff, particles, and camera language can create a cinematic impression.

This architecture does not promise true 3D geometry, physically correct volumetric rendering, or film-grade 3D character models. Those would require a different renderer and asset pipeline. WebGL may be evaluated later as an optional enhancement, but it is not a prerequisite for the core product.

---

## 2. Locked Project Constraints

These constraints are mandatory unless the user explicitly changes them.

1. **Preserve the existing application.** Do not replace the project with the Gemini dashboard or rewrite all painters.
2. **Single-file delivery.** The production app remains one HTML file with inline CSS and JavaScript. No package manager, build step, server, or installation.
3. **No external runtime dependency.** Vanilla JavaScript, Canvas 2D, SVG, and browser APIs are the foundation. Remote fonts must have a safe local fallback.
4. **Backward compatibility.** Existing v1 JSON and its aliases remain accepted while the internal runtime evolves.
5. **One source of truth.** Schema, aliases, defaults, action capabilities, limits, and prompt vocabulary must converge into a central contract rather than diverge between prompt text, normalizer, and painters.
6. **Incremental refactoring.** Keep working renderers and migrate them behind clear interfaces. No blind replacement and no mass deletion before regression coverage exists.
7. **Test before release.** Every phase must have explicit acceptance criteria and reproducible test fixtures.
8. **Manual JSON remains available.** AI generation is an assistant workflow, not a mandatory dependency.
9. **AI delegation.** ChatGPT leads architecture and integration; Claude, DeepSeek, and Gemini may provide bounded reviews or specialist proposals. Their code is treated as untrusted input until audited and tested.
10. **Handoff is mandatory.** At each substantial milestone, update a handoff document with exact repository state, completed work, tests, known issues, and next steps.

---

## 3. Decisions Already Locked

| Topic | Decision |
|---|---|
| Minimum duration | 60 seconds is the default preset, not a hard minimum. User-defined clips shorter than 60 seconds must be possible. |
| v2 data model | Internal runtime / compiled representation. Users and AI continue authoring v1-compatible JSON initially. |
| Export strategy | First make realtime MediaRecorder export robust. Design boundaries so frame-accurate export can be added later. |
| Photo / stylize system | Defer until the core engine is stable; it must not block animation, characters, or world rendering. |
| Browser support | Chrome and Edge are primary. Other browsers are best-effort with feature detection and graceful fallback. |
| AI integration | Keep manual JSON fallback. Host-specific \`claude.use('sample')\` must remain behind an adapter. |
| Rendering | Canvas 2D remains the primary renderer; SVG remains available for vector actors/assets. |
| Architecture style | Incremental namespaced modules inside the single script; do not rely on an external sidecar monkey-patching closed-over state. |

---

## 4. Baseline Audit — Existing Clip Maker

### 4.1 Verified repository baseline

The current \`main\` branch source was fetched directly from GitHub during this audit.

- File: \`Clip_Maker_Enhanced.html\`
- Commit: \`d02793fb568414fc8c545e557ac7fc4b124b86c6\`
- Size: 102,974 bytes
- Lines: 882
- Script form: classic inline \`<script>\`, not an ES module.

The older Architecture Review document reports 130,657 bytes / 881 lines in its header. That metadata does not match the current fetched file. The source commit and live file are the baseline authority for implementation.

### 4.2 Existing systems to preserve

The current application already contains substantial working procedural drawing code. Preserve these systems and improve them in place:

- Themes and environments: road, room, city, nature, space, lake, custom, map, ring, shophouse, and yamo.
- Character renderers: human / zombie, cat / dog / critter, fish, bird, blob, robot, beast, kangaroo, and SVG actor.
- Props and effects: rod, rocket, image, pin, route, dots, ring, shape, beam, aura, blast, songthaew, ball, and spark.
- Camera transform, zoom, roll, shake, and follow behavior.
- Lighting, shadows, atmosphere, subtitles, graphics overlays, crowd, geo layers, and map support.
- Existing AI idea-to-scenes workflow, prompt entry points, voice selection, and manual JSON editing.

These are valuable assets. The initial work should improve data flow and control around them, not redraw every renderer.

### 4.3 Confirmed high-priority defects

| ID | Finding | Impact | Planned phase |
|---|---|---|---|
| BASE-01 | \`normalizeSceneData()\` inserts theme / sky / ground / atmosphere defaults before \`build()\` inheritance. | A later scene can lose continuity and fall back to a new default environment. | Core Architecture |
| BASE-02 | \`build()\` and the render loop lack complete fault isolation. | Invalid input or a drawing exception can stop playback without a useful error. | Core Architecture |
| BASE-03 | \`frame(time)\` depends on mutable globals such as \`POSG\`, \`DIRS\`, \`ANCH\`, \`CAMI\`, \`LT\`, and \`GEO\`. | Seeking and repeated renders may not produce the same frame. | Core Architecture → Animation Core |
| BASE-04 | Timeline selection uses a linear \`findIndex\`; end-of-timeline and out-of-range handling is fragile. | Boundary frames can resolve to the wrong scene. | Animation Core |
| BASE-05 | Duration is forced by \`Math.max(60, ...)\`. | Short clips cannot be authored or previewed. | Core Architecture |
| BASE-06 | Keyframe interpolation supports only a small set of fields and uses a single hard-coded easing curve. | Limited timing control and abrupt action changes. | Animation Core |
| BASE-07 | Character action behavior is embedded in renderers and is mostly procedural sine motion rather than a unified motion state model. | Weak transitions, inconsistent action timing, and difficult species-specific upgrades. | Animation Core → Character Engine |
| BASE-08 | Camera follow reads actor positions from mutable state populated during actor drawing. | Follow can lag by a frame and is sensitive to render order. | Animation Core / Cinematic Engine |
| BASE-09 | \`Math.random()\` is used during rendering. | Re-rendering the same time can change visual output. | World / Cinematic / QA |
| BASE-10 | Asset loading is not fully awaited before playback/export. | SVG/image assets can be absent in early frames. | World / QA & Export |
| BASE-11 | \`subtitle()\` and \`crowdOf()\` perform avoidable work during frame rendering. | Unnecessary allocations and repeated text measurement. | Core Architecture / QA |
| BASE-12 | The prompt, aliases, and painter support lists are separate sources of truth. | AI may request unsupported actions or fields. | AI Director |
| BASE-13 | MediaRecorder export is realtime and has no captured speech audio. | Export depends on tab scheduling; TTS is not embedded in the video. | QA & Export |

### 4.4 Runtime-scope caution: globals and proposed sidecars

The source is a classic script, so top-level function declarations and top-level lexical declarations do not have identical exposure behavior.

- Functions such as \`frame\` and \`kf\` are declared as top-level function declarations.
- State such as \`scenes\`, \`total\`, \`DIRS\`, \`POSG\`, \`ANCH\`, \`GEO\`, and constants such as \`THEMES\` are declared with \`let\` / \`const\` and must not be assumed to exist as \`window.*\` properties.
- A prototype that reads \`window.scenes\` or \`window.DIRS\` is therefore not a safe integration strategy.
- Adding \`window.AnimCore\` alone does not create a reliable hook into lexical state or change the wall-clock behavior of the current \`loop()\`.
- Rendering a previous and current frame twice on the same canvas is not a safe crossfade strategy: the frame function clears/draws the canvas and mutates shared render state.
- Applying \`globalAlpha\` around one renderer does not blend two different poses; both poses must actually be evaluated and drawn.

**Architecture decision:** introduce explicit module interfaces and state ownership through controlled edits to the existing source. Do not make the production engine depend on speculative monkey-patching.

---

## 5. External Handoff Review

### 5.1 DeepSeek Animation Core proposal

**Useful concepts to adopt**

- Timeline boundary handling and a binary-search scene resolver.
- Per-keyframe easing and validation.
- A motion state model with action-local time.
- Optional physics rather than forcing physics onto every action.
- Camera follow based on the current frame's evaluated actor positions.
- Deterministic rendering and a debug view.
- A staged implementation and explicit acceptance tests.

**Items not to copy without correction**

- The sidecar's assumptions about \`window.scenes\`, \`window.total\`, and other lexical bindings.
- Fixed-step clock code that is not actually connected to the player loop.
- Crossfade code that renders two full frames into the same mutable canvas.
- Pose blending that changes alpha but does not evaluate and draw both poses.
- Any wrapper that depends on function replacement without proving the actual call path.
- Prototype details that are incomplete, inconsistent, or not covered by tests. For example, the described easing list includes Bezier while the shown prototype does not implement it.

### 5.2 Gemini Visual Engine proposal

**Useful visual direction**

- Species-specific anatomy and joint hierarchies.
- Layered character shading, rim light, and contact shadows.
- Environment depth layers and foreground framing.
- Stronger set dressing and environment-specific materials.
- Cinematic shot sizes, camera movement, and controlled impact shake.
- Secondary motion, particles, rain, dust, smoke, and splashes.
- A dedicated visual test bench and a visual acceptance matrix.

**Standalone dashboard assessment**

The supplied Gemini HTML is a reference prototype, not a production module ready to replace the current application.

Its actual behavior is narrower than its specification:

- The documented contract uses \`scenes[]\`, while the sample UI payload uses a singular \`scene\` object.
- Applying JSON only reads a few scene-level values (theme, camera preset, shake, VFX density); it does not render the supplied cast, actors, positions, lighting sources, environment elements, or VFX definitions from the full contract.
- Character and animation selectors update a small set of renderer flags; the supplied actor records do not drive a full actor timeline.
- The human, robot, quadruped, and monster drawings are illustrative primitive rigs, not the described joint hierarchy / analytical IK engine.
- Lighting is mostly background gradients and overlays; there is no unified light model applied consistently to the environment and character surfaces.
- Camera shake uses fresh random offsets each frame rather than a decaying impulse.
- The loop advances time by a fixed 0.016 seconds per rendered frame, so playback speed is tied to refresh rate and can drift.
- Particles continue updating while playback is paused.
- The code does not implement the claimed DOF distance buffer, motion blur accumulation, or full parallax layer model.
- The Google Fonts stylesheet is a network dependency, so the file is not strictly offline as written.

**Architecture decision:** adopt the visual requirements and selected algorithms, not the dashboard's code structure or production-readiness claims. Build each feature against the real Clip Maker data contract and existing renderer.

---

## 6. Target Architecture

### 6.1 Data flow

\`\`\`text
AI / Manual JSON (v1)
        |
        v
  Parse + Safe Input
        |
        v
  Normalize (aliases only; preserve authored intent)
        |
        v
  Validate (typed issues, limits, safe fallbacks)
        |
        v
  Compile (inheritance first, then defaults; timeline + immutable runtime data)
        |
        v
  Evaluate(project, time)  ---> FrameState (deterministic data)
        |
        v
  Renderer(RenderContext, FrameState)
        |
        v
  Existing Canvas 2D / SVG painters, upgraded incrementally
        |
        v
  Preview / Seek / Realtime Export
\`\`\`

### 6.2 Module boundaries

All modules remain inside the same HTML script. The \`CM\` namespace is the planned internal boundary; it is introduced incrementally.

| Module | Owns | Must not own |
|---|---|---|
| \`CM.util\` | Math, easing, seeded RNG, stable helpers | DOM, Canvas |
| \`CM.schema\` | Enums, aliases, field rules, limits, defaults, capability metadata | Rendering |
| \`CM.normalize\` | v1 aliases and structural normalization | Defaults, timeline state |
| \`CM.validate\` | Input validation and issue reporting | Canvas, playback |
| \`CM.compile\` | Inheritance, defaults, resolved timeline, immutable runtime project | DOM, Canvas |
| \`CM.evaluate\` | Time → actor poses, camera target, scene state, action-local progress | DOM, Canvas, mutable globals |
| \`CM.animation\` | Clock, timeline resolution, keyframe evaluation, motion state, optional physics | Painter implementation |
| \`CM.character\` | Species rigs, pose evaluation, anchors, renderer adapters | Timeline ownership |
| \`CM.world\` | Themes, parallax layers, decor, light/environment state | Player clock |
| \`CM.cinematic\` | Camera path, focus, shake, transitions, post-effects | Data validation |
| \`CM.renderer\` | Canvas state, render context, draw ordering, painter calls | UI controls |
| \`CM.player\` | Play, pause, stop, seek, requestAnimationFrame, error boundary | Painter internals |
| \`CM.ai\` | Prompt generation and host adapter | Rendering |
| \`CM.exporter\` | Asset readiness, MediaRecorder, output lifecycle | Scene authoring |
| \`CM.ui\` | DOM, panels, buttons, editor, messages | Animation math |
| \`CM.selftest\` | Deterministic unit / integration / capability tests | Production state mutation |

### 6.3 Runtime contract

- v1 remains the public authoring format during the migration.
- v2 is the normalized, validated, compiled internal representation.
- Compile is the only place where inheritance and defaults are resolved.
- Evaluation is a time-based calculation and must not draw to Canvas.
- Renderer consumes an explicit frame state and a render context.
- Existing painters are wrapped/adapted one at a time; their drawing internals are preserved until regression tests justify a change.
- New fields are optional during migration. Existing v1 fields keep their meaning unless a documented migration rule says otherwise.
- Unknown or invalid input produces a warning/error with a path; it must not silently mutate into a plausible but incorrect action.

---

## 7. Character & Animation Standards

### 7.1 Shared rig principles

- Use hierarchical local transforms: root → body/torso → head and limbs → joints / extremities.
- Keep rig dimensions in local character units; apply world position, depth scale, and facing at the root.
- Define named anchors (for example hand, mouth, center, foot) for props and interaction.
- Separate pose evaluation from drawing so the same pose can be inspected, blended, and tested.
- Use action-local progress and explicit transitions. Do not let a jump, punch, hit, or KO animation continue accidentally because it is driven only by scene-global time.
- Preserve species-specific proportions and locomotion. Never solve a kangaroo, fish, bird, quadruped, or robot by simply swapping a human head.

### 7.2 Action model

Initial action vocabulary remains compatible with the existing renderer:

\`idle, walk, run, jump, wave, dance, float, cheer, point, hold, pull, cast, channel, punch, guard, hit, ko, kick, sit, sleep, swim, fly\`

Action definitions must declare:

- Supported species / rig types.
- Required props or interaction targets, if any.
- Entry, active, and recovery phases where relevant.
- Root motion or in-place behavior.
- Required anchors.
- Whether the action may loop.
- What happens when the scene ends before the action finishes.

### 7.3 Physics policy

Physics is opt-in. Use deterministic, bounded simulation only for actions that benefit from it (jump arcs, falls, thrown objects, impacts). Ordinary idle, walk, swim, and fly may remain procedural until a physical model is justified.

Do not add collision or gravity to every actor by default. Keyframes remain authoritative where authored motion exists.

---

## 8. World, Lighting & Visual Standards

### 8.1 Depth and composition layers

Adopt a consistent logical order:

1. Far background: sky, distant silhouettes, distant skyline / mountain / forest.
2. Background: buildings, canopy, distant crowd, large environment structures.
3. Midground: primary stage, floor, ring, waterline, set dressing.
4. Actor and interaction layer: characters, shadows, anchors, held objects.
5. Foreground: near props, pillars, foliage, ropes, framing objects.
6. Atmospheric / screen layer: rain, dust, sparks, fog, vignette, subtitles, graphics.

The existing renderer has specific prop back/front passes. Keep those semantics while adding an explicit layer model; do not reorder all drawing in one change.

### 8.2 Environment standard

Each theme should define:

- Horizon / ground plane and perspective cues.
- Depth layers and parallax factors.
- Material language (wood, asphalt, water, stone, metal, foliage).
- Environment-specific set dressing.
- Light sources and ambient color.
- Optional weather / atmosphere.
- Safe actor staging zones and anchor / prop interaction rules.

### 8.3 Lighting standard

Start with a small, predictable light model:

- Ambient fill.
- One key light with direction and color.
- Optional rim / back light.
- Local glow sources for lamps, neon, fire, and magic.
- Contact shadow / grounding cues.

Do not claim physically correct ray tracing or volumetric lighting from Canvas gradients. The visual goal is consistent, art-directed lighting that reacts plausibly to scene direction.

### 8.4 VFX standard

VFX must have explicit type, coordinate space, layer, lifetime, seed, and quality budget. Prefer pooled or preallocated particle state over allocating new particles on every frame. VFX should degrade gracefully when performance is constrained.

---

## 9. Cinematic Camera Standards

Support shot intent and motion as explicit data, not only a zoom scalar.

Initial shot vocabulary:

- Wide / establishing.
- Medium / two-shot.
- Close-up / detail.
- Extreme close-up.
- Low angle / high angle.
- Dutch angle.

Initial motion vocabulary:

- Pan / tilt.
- Tracking / actor follow.
- Dolly / zoom.
- Impact shake with deterministic decay.
- Hold / settle.
- Optional focus / DOF treatment.

Camera evaluation must use actor positions from the **same evaluated frame**. Follow should support a dead zone and smoothing. Clamp zoom and focus values to valid bounds. A transition is a composition between two valid render states, not a second uncontrolled call to the full legacy frame renderer.

---

## 10. AI Director & Data Contract Standards

The AI Director must generate only supported v1 fields during the compatibility period.

It must reason through:

1. Idea, genre, emotion, and intended audience.
2. Characters, species, identity, and visual continuity.
3. Scene geography and staging.
4. Goal, obstacle, cause, action, reaction, and result.
5. Required props and whether they are visible before interaction.
6. Character state and position continuity.
7. Shot size, camera angle, movement, and edit rhythm.
8. Timing and subtitle / narration limits.
9. Schema validation and a final continuity check.

### Continuity rules

- A character cannot use an object that has not been introduced or placed within reach.
- An interaction must identify actor, target, contact point, and outcome.
- Props must persist or be explicitly handed off, dropped, removed, or reintroduced.
- Character species and identity cannot change between shots unless the story explicitly describes a transformation.
- Scene changes must preserve the world unless a new location is explicitly intended.
- Movement must have a reachable destination and respect the scene's staging.
- A shot must not require an unsupported action or unavailable prop.
- Action timing must allow the action to complete or explicitly show interruption.

### One source of truth

The schema registry must eventually generate or validate:

- Action lists and species compatibility.
- Theme and environment options.
- Prop and VFX types.
- Field names, aliases, ranges, and defaults.
- Prompt tool catalog and examples.
- Error and warning messages.

The prompt must not claim a renderer capability until that capability is implemented and tested.

---

## 11. Phased Delivery Roadmap

The user-defined order is authoritative. A phase may contain multiple small commits, but it must not skip its acceptance gate.

### Phase 00 — Blueprint (this document)

**Purpose:** agree on product goal, constraints, architecture boundaries, roadmap, and test standards.

**Exit gate:**
- This blueprint is committed.
- No production renderer code is changed in this phase.
- Known defects and prototype limitations are documented.
- Next phase has a bounded implementation scope.

### Phase 01 — Core Architecture

**Scope:**
- Establish \`CM\` namespace and module boundaries inside the single HTML file.
- Add capability detection and safe fallbacks for \`roundRect\`, \`ctx.filter\`, fonts, MediaRecorder, and related browser APIs.
- Add build / render / player error boundaries with clear Thai error messages.
- Separate authored data, normalized data, and compiled scene timing.
- Fix inheritance ordering: normalize aliases first, inherit authored values second, apply defaults last.
- Make 60 seconds the default duration while allowing a valid shorter user duration.
- Add baseline self-tests and deterministic fixtures.
- Keep legacy UI and painters running through adapters.

**Out of scope:** full skeleton rewrite, new physics, new environment system, WebGL, photo/stylize.

**Exit gate:**
- Invalid root data, null scenes, malformed arrays, and invalid colors do not silently kill the player.
- Theme / sky / ground / atmosphere continuity tests pass.
- Short duration can be built and played.
- Existing representative v1 fixtures still render.
- Main branch remains untouched until review.

### Phase 02 — Animation Core

**Scope:**
- Explicit timeline resolver with correct boundary semantics.
- Clock ownership for play / pause / seek / stop.
- Keyframe validation, sorting policy, interpolation, and per-key easing.
- Action-local time and motion state transitions.
- Same-frame actor position evaluation for camera follow.
- Optional deterministic physics for selected actions.
- Debug timeline / actor state overlay.

**Exit gate:**
- Seek-to-time equals continuous playback at the same time.
- Scene boundaries never jump to scene zero.
- Walk / run / jump / punch / hit / KO behave consistently at multiple frame rates.
- Legacy JSON without new fields retains its previous intended behavior.

### Phase 03 — Character Engine

**Scope:**
- Shared hierarchical transform and joint data model.
- Human rig first, then kangaroo, quadruped, bird, fish, robot, and other supported archetypes.
- Pose evaluation separated from drawing.
- Species-specific gait and action capability tables.
- Named anchors for props and contact.
- Controlled pose blending with actual outgoing and incoming pose evaluation.
- Secondary motion only after core poses are stable.

**Exit gate:**
- Species stress fixtures prove distinct anatomy and locomotion.
- Interaction anchors align with props.
- Action transitions do not snap or freeze.
- Existing renderers remain available as fallback until each replacement passes.

### Phase 04 — World & Environment Engine

**Scope:**
- Explicit environment layers and parallax.
- Improved environment geometry and set dressing.
- Ground plane / horizon / depth scaling consistency.
- Environment materials and light direction.
- Rain, fog, dust, sparks, water effects, and particle quality controls.
- Reuse existing themes and decor as the starting library.

**Exit gate:**
- Environment fixtures have clear depth separation.
- Actors remain grounded and scaled consistently.
- Lighting direction is coherent across environment, actors, and shadows.
- Particle load remains within the measured performance budget.

### Phase 05 — Cinematic Engine

**Scope:**
- Shot presets and shot intent.
- Camera path interpolation, tracking, dead zones, smoothing, and safe zoom.
- Deterministic impact shake.
- Transition composition.
- Optional DOF / motion treatment with capability-aware fallback.
- Cinematic test sequences.

**Exit gate:**
- Camera movement is continuous and intentional.
- Follow uses current-frame actor positions.
- No uncontrolled frame double-rendering.
- Visual tests cover wide, medium, close-up, low/high angle, and Dutch angle.

### Phase 06 — AI Director

**Scope:**
- Generate prompts and tool catalogs from the central schema.
- Add continuity reasoning and prop / target requirements.
- Enforce species/action compatibility.
- Add pre-render JSON validation and actionable repair messages.
- Keep manual JSON editing and host adapter fallback.
- Update examples to match actual renderer support.

**Exit gate:**
- AI outputs validate against the same schema the renderer uses.
- Test stories contain no missing-prop or impossible-action defects.
- Kangaroo boxing, fishing, multi-character interaction, and scene continuity fixtures pass.

### Phase 07 — QA & Export

**Scope:**
- Full regression suite and browser capability report.
- Asset readiness / decode barrier.
- Stable seeded randomness and repeated-frame tests.
- Performance profiling and budgets.
- Robust realtime MediaRecorder export.
- Visibility / tab-throttle warnings.
- Clear audio limitations and a separate future audio plan.
- Final documentation and handoff.

**Exit gate:**
- All phase acceptance tests pass in current Chrome and Edge.
- Export output duration is within the defined tolerance.
- No unhandled render exceptions across the fixture suite.
- Known limitations are visible to the user.

---

## 12. Required Test Fixture Suite

These fixtures are permanent regression assets. They should remain small, deterministic, and cover one primary behavior each.

| Fixture | Main coverage | Required observation |
|---|---|---|
| F01 — Basic human walk | v1 compatibility, depth scale, walk | Feet remain grounded; movement reaches destination. |
| F02 — Human run / stop | action-local time, state transition | Run accelerates and settles without snapping. |
| F03 — Jump and landing | optional physics / jump arc | Takeoff, apex, landing, and compression are visible. |
| F04 — Kangaroo professional boxing | species rig, guard / punch / hit / KO, gloves, ring | Kangaroo identity is retained; contact has cause and effect. |
| F05 — Fishing sequence | lake, rod, hand anchor, line / fish interaction | Rod exists before use; hand, line, and target align. |
| F06 — Two-character interaction | facing, anchors, follow | Both actors face and interact with the correct target. |
| F07 — Scene inheritance | theme, sky, ground, atmosphere, decor | Unspecified values inherit before defaults are applied. |
| F08 — Robot action | mechanical rig and action capability | Robot does not fall back to human anatomy. |
| F09 — Quadruped locomotion | four-leg gait and body weight | Diagonal gait and body movement are distinct from human walk. |
| F10 — Bird / fish motion | avian and aquatic rigs | Wing / spine motion follows species-specific rules. |
| F11 — Cinematic camera | shot presets, follow, shake, transitions | Smooth composition with no one-frame follow lag. |
| F12 — Invalid JSON / invalid painter input | validation and fault isolation | Useful error shown; app remains recoverable. |
| F13 — Asset readiness | SVG / image decode | First visible frame includes loaded assets. |
| F14 — Deterministic repeat | same project and same time | Frame state and seeded procedural effects repeat. |
| F15 — Short export | duration below 60 seconds | Playback and export respect the selected duration. |

---

## 13. Quality, Performance & Evidence Policy

- Capture a baseline before optimizing. Do not invent FPS claims.
- Profile on the same machine and browser version when comparing before / after.
- Record median and p95 frame time for representative fixtures.
- Use a quality budget for crowd count, particles, blur, and expensive gradients.
- Prefer caches and precomputation for text layout, crowd expansion, and compiled scene data.
- Use deterministic seeds for any procedural effect that appears in repeatable tests.
- Do not claim a feature is complete because a UI control or JSON field exists.
- Every completion report must label evidence as:
  - **Static audit** — inspected source; not executed.
  - **Automated test** — script/test executed with recorded result.
  - **Browser test** — opened and exercised in a real supported browser.
  - **Visual review** — inspected actual rendered output.
- If a test was not run, say so explicitly.

---

## 14. Git & Release Workflow

1. Keep \`main\` as the stable baseline.
2. Implement each phase on its own branch.
3. Make small commits with one coherent purpose.
4. Never update the same file concurrently through multiple write operations.
5. Before each write, fetch the current file and SHA; do not overwrite a newer version.
6. Run static checks and available automated tests before requesting browser validation.
7. Do not merge a phase until its acceptance gate is reviewed.
8. Keep the source file and this blueprint aligned; update the blueprint only when a decision or phase boundary actually changes.
9. At the end of each phase, update \`HANDOFF.md\` with:
   - repository / branch / commit SHA,
   - files changed,
   - implementation summary,
   - tests run and exact outcomes,
   - tests not run,
   - known issues,
   - next phase and exact starting instructions.

---

## 15. Immediate Next Step — Phase 01 Core Architecture

Phase 01 must begin with a fresh fetch of \`main\` and a new implementation branch based on the current stable commit.

Recommended bounded work order:

1. Add a lightweight baseline / self-test harness without changing painter behavior.
2. Add centralized error reporting and a player-loop fault boundary.
3. Separate normalize aliases from inheritance and defaults.
4. Correct timeline duration selection: default 60 seconds, but respect explicit shorter durations.
5. Add focused regression tests for inheritance, invalid input, and duration.
6. Verify representative existing scenes and compare screenshots in Chrome / Edge.
7. Produce a phase handoff before starting Animation Core.

Do not begin skeleton / IK, new visual effects, a full prompt rewrite, or a WebGL experiment during Phase 01. Those belong to later phases.

---

## 16. Phase 00 Completion Record

- [x] Existing production source fetched from GitHub.
- [x] Source baseline SHA, size, and line count recorded.
- [x] Existing systems to preserve documented.
- [x] DeepSeek Animation Core proposal reviewed for integration risks.
- [x] Gemini Visual Engine specification and standalone prototype reviewed.
- [x] Product constraints and prior decisions consolidated.
- [x] Phase order and acceptance gates defined.
- [ ] Browser screenshots / performance baseline captured (Phase 01; not performed in Phase 00).
- [ ] Production code changed (intentionally not performed in Phase 00).

**Phase 00 is complete when this blueprint is committed. No production code changes are included in this phase.**
