# Phase 02 — Action Choreography & Cinematic Camera Plan

## Goal

Move the renderer from repetitive time-driven poses to story-driven, coordinated performance. A shot must communicate **what a character wants, what they do, what they affect, and how others respond**. Camera behavior must support the dramatic beat rather than defaulting to zoom-only movement.

This phase builds on `phase-01-core`. It must not replace the renderer or break existing v1 JSON.

## Verified current-source findings

Source inspected on `phase-01-core`:
- `actors()` derives most procedural poses from the global scene time `t`; examples include punch/kick cycles driven by sine waves. This repeats regardless of story intent.
- Fight-facing currently resolves a left/right direction from `face` or the nearest actor. It does not provide a full gaze/torso/pose target system.
- `kf()` interpolates only `x`, `y`, `s`, and `rot`; the selected `act` is effectively stepped, not a complete pose interpolation.
- `frame()` interpolates camera `x/y/z` and supports `roll`, `shake`, and `follow`. It has no authored shot-size, camera-move, or angle vocabulary.
- The current Canvas 2D character rigs are procedural 2D drawings. They can support stronger staging, posing, framing, and parallax, but cannot provide true 3D orbit/perspective without new 3D assets or a 3D renderer.

These are source-inspection findings, not browser-test results.

## Locked implementation principles

1. Preserve current JSON fields and behavior as the legacy fallback.
2. New choreography is opt-in; old `act`, `x/to`, and `keys` continue to work.
3. Use normalized shot-local time (0–1) for authored action timing. Do not use global clip time to restart or repeat a one-off action.
4. Resolve actor identities and targets by `who`; never guess a target from arbitrary array order when an explicit target is available.
5. A physical interaction must coordinate both sides: initiator, target, contact moment, reaction, and recovery.
6. Every action has a start, a meaningful middle, and an end state. One-shot actions do not loop unless explicitly marked as looping.
7. Camera moves must have a clear subject and purpose; no automatic zoom on every shot.
8. Keep the Canvas 2D renderer. Simulate cinematic language with staging, framing, layered foreground/background, depth scaling, and camera translation. Clearly distinguish simulated 2D angles from true 3D viewpoints.
9. No simultaneous edits to this HTML file by multiple agents. Do not modify `main`.

## Proposed v2 shot data contract

### Shot-level camera

Legacy `cam.from/to` remains supported. Add optional descriptive fields:

```json
{
  "cam": {
    "shot": "wide",
    "angle": "eye",
    "move": "push_in",
    "subject": "hero_boxer",
    "focus": ["hero_boxer", "zombie1"],
    "from": {"x": 0.5, "y": 0.62, "z": 1},
    "to": {"x": 0.58, "y": 0.64, "z": 1.12},
    "duration": 0.8,
    "ease": "ease_in_out"
  }
}
```

Initial vocabulary:
- shot: `extreme_wide`, `wide`, `medium`, `close_up`, `extreme_close_up`, `over_shoulder`
- angle: `eye`, `low`, `high`, `dutch`, `profile`, `front`
- move: `static`, `push_in`, `pull_out`, `pan`, `tilt`, `track`, `handheld`, `whip_pan`
- ease: `linear`, `ease_in`, `ease_out`, `ease_in_out`

Unsupported/physically impossible 3D moves must be approximated deliberately or reported as unsupported; do not silently pretend a 2D canvas has a true 3D camera.

### Coordinated action events

Add optional shot-level `events` to express causality. Example:

```json
{
  "events": [
    {"at": 0.08, "type": "notice", "actor": "hero_boxer", "target": "zombie1"},
    {"at": 0.22, "type": "guard", "actor": "hero_boxer", "target": "zombie1"},
    {"at": 0.42, "type": "attack", "actor": "hero_boxer", "target": "zombie1", "action": "punch", "contact": 0.58},
    {"at": 0.58, "type": "react", "actor": "zombie1", "source": "hero_boxer", "action": "hit"},
    {"at": 0.76, "type": "recover", "actor": "hero_boxer"}
  ]
}
```

The event evaluator must produce deterministic actor states at any requested timestamp, including when the user seeks or exports. It must not depend on having played all earlier frames.

Initial event types: `notice`, `approach`, `turn`, `guard`, `attack`, `dodge`, `hit`, `fall`, `recover`, `pickup`, `use_prop`, `celebrate`.

Initial attack actions: `punch`, `kick`, `push`, `grab`, `throw`, `cast`. Only enable an action when the actor rig supports it; otherwise emit a clear build warning and use a safe fallback.

### Actor performance controls

Add optional per-actor `lookAt`, `face`, `intent`, and `motion` fields. `face` controls body orientation; `lookAt` controls gaze/head direction; `intent` is descriptive metadata and must not be rendered as an action by itself. Motion events control actual timed performance.

## Runtime architecture

1. **Compile:** validate and compile shot data into normalized timeline events.
2. **Evaluate:** `evaluateShot(shot, localTime)` computes all actor states from the shot start; pure/deterministic, no drawing.
3. **Resolve interactions:** resolve attack/contact/reaction pairs by explicit actor IDs and event times.
4. **Camera plan:** resolve shot size, angle approximation, subject framing, and move into a camera transform.
5. **Render:** draw environment, props, actors, interaction effects, atmosphere, subtitles, and overlays.
6. **Seek/export parity:** the same timestamp must produce the same actor pose, camera, and effects whether reached by playback, timeline seek, or export.

Keep the first implementation incremental: isolate evaluator helpers inside the existing inline script and route the current render loop through them only when new fields are present.

## Cinematic shot grammar

Use shot choice to communicate information:
- Establishing/location: wide or extreme wide, usually static or slow pan.
- Character intention: medium or over-shoulder.
- Discovery/reaction: close-up or extreme close-up.
- Threat/power: low angle; vulnerability/confusion: high angle.
- Action geography: wide/medium two-shot before impact.
- Contact: brief close-up/impact insert only when it clarifies the hit.
- Consequence: reaction close-up or wide reveal.
- Do not use the same shot size or move for more than two consecutive shots unless the story intentionally holds a composition.

Camera shake is reserved for impact, danger, or handheld urgency; it is not a substitute for camera movement.

## Action grammar and physical continuity

For a fight beat, the minimum readable chain is:
1. Establish relative positions and facing.
2. Anticipation / preparation.
3. Attack trajectory toward the named target.
4. Contact at a defined time and spatial relationship.
5. Target reaction in response to contact.
6. Follow-through / balance change.
7. Recovery or next intentional action.

The renderer must not make a character punch empty space when the target is elsewhere. If geometry cannot support contact, warn and avoid showing a false impact. Props must be attached to the actor or target they affect.

## Acceptance criteria

- Existing Phase 01 fixture still builds and plays without new required fields.
- Legacy scenes with only `act` render as before (or with a documented bug fix).
- A non-looping punch runs once per authored event; it does not repeat because wall-clock time advances.
- A two-character attack has a visible anticipation, contact, target reaction, and recovery.
- The target actor reacts at the authored contact time, not independently.
- Repeated playback and arbitrary seeks to the same timestamp produce identical poses/effects.
- At least six distinct shot sizes/angles/moves are visibly distinguishable within the supported 2D approximation.
- Camera can pan/track a moving subject without relying on zoom alone.
- No action/event with an unknown actor or target crashes rendering; build reports the shot and field.
- No browser/runtime PASS claim until Chrome/Edge testing is actually performed.

## Test fixtures to add

1. Boxer vs zombie: guard → feint → dodge → counterpunch → recoil → knockdown.
2. Character notices a moving vehicle, turns gaze/body, steps back, then runs.
3. Character picks up a prop, holds it, and uses it; prop follows the correct hand.
4. Two actors exchange turns without simultaneous repetitive attack loops.
5. Camera wide → over-shoulder → impact close-up → reaction close-up → aftermath wide.
6. Seek directly to contact/reaction/recovery timestamps and compare with normal playback.
7. Legacy Phase 01 JSON without `events` or new camera fields.

## Scope boundaries

Phase 02 is about deterministic action choreography, interaction causality, and cinematic 2D camera language. Full 3D character models, true camera orbit, skeletal IK, and advanced pose blending are separate later work unless the current 2D rig proves a specific small enhancement is feasible.

## Status

- Branch created: `phase-02-action-cinema` from `phase-01-core`.
- Initial Action Event evaluator added to the existing inline script; legacy scenes without `events` continue through the existing action path.
- Shot events are evaluated from normalized local shot time, so the selected event state is derived from the requested timestamp rather than prior playback history.
- Added contact-timed punch extension/retraction and target-facing for authored `attack` / `react` events.
- Corrected the duration selector label so value 0 clearly means use the JSON duration.
- Added `docs/PHASE_02_ACTION_EVENTS_TEST.md` with an 8-second boxer/zombie smoke-test fixture.
- Latest source commit at this update: `646c4affba362fbbc5ac164d0474926304d409a6` (subsequent docs commit does not alter source).
- Verification performed: source-level inspection of inserted helper, actor-renderer call site, punch phase calculation, and UI label. No browser playback, visual comparison, runtime console test, or export test has been run for this phase.
- Known remaining work: reaction-body displacement and collision/contact geometry; actor gaze/head orientation; event-driven approach paths; broader action state machine; camera shot grammar and camera moves; event validation/warnings; seek/export parity tests.

### Latest user smoke-test feedback and corrective patch

User confirmed the boxer raises guard and punches exactly once. User reported that the zombie reaction did not visually align with impact and that the zombie initially faced the wrong direction before turning toward the boxer.

Source correction committed as `451a5ae178976a7cfa73edd29589ce4354fb88c9` on `phase-02-action-cinema`:
- Infer initial actor facing from event `target` / `source` relationships before those events begin. Explicit actor-level `face` remains higher priority.
- Strengthen `hit` reaction with a clear torso recoil and body rotation that decay over the event phase.
- Static verification: inline JavaScript syntax passed; isolated event evaluator confirmed zombie has no `hit` state at k=0.579 and enters `hit` at k=0.58.
- Not verified: browser playback, visual hit/contact alignment, export parity. User must re-run the same 8-second fixture.


## Defensive-action increment — implementation added

Source commit: `451395ffa0ad9ec21b9d710c44799e61f1c870c2` (with the initial defensive-action implementation in the preceding commit).

- Added event type `dodge` as its own action instead of mapping it to a vertical jump.
- Added event type `block` as a distinct defensive pose; existing `guard` behavior remains unchanged.
- Dodge uses normalized event phase for a short backward displacement and lean-away/recovery arc, so the same timestamp evaluates deterministically.
- Block uses a compact forward forearm guard and restrained torso posture.
- Added a dedicated Dodge + Block test fixture to `docs/PHASE_02_ACTION_EVENTS_TEST.md`.
- Scope: currently implemented for the human/zombie procedural rig path. This is pose choreography, not collision detection or damage resolution.
- Verification status: source-level edits and fixture consistency checked. Browser playback and visual contact alignment have not been tested by the maintainer; user verification is required.
