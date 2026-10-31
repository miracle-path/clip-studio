# Phase 02 — Action Event Smoke Test

## Purpose

Verify the first incremental Action Event implementation on top of the Phase 01 renderer. This is a test fixture, not a claim that the animation is production-ready.

## How to test

1. Open the `phase-02-action-cinema` version of `Clip_Maker_Enhanced.html` in Chrome or Edge.
2. Copy the JSON below into the JSON field.
3. In Duration, choose **ใช้ความยาวจาก JSON**.
4. Press Play and watch the entire 8-second shot.
5. Repeat playback and seek is not yet available in the UI; do not treat this smoke test as seek/export parity verification.

## Fixture

```json
{
  "duration": 8,
  "cast": {
    "boxer": {"c": "human", "col": "#d8322f"},
    "zombie": {"c": "zombie", "col": "#78936c"}
  },
  "scenes": [
    {
      "theme": "ring",
      "sky": ["#171a2e", "#4b3545"],
      "ground": "#29232b",
      "actors": [
        {"who": "boxer", "x": 0.40, "to": 0.40, "y": 0.76, "s": 1.05, "act": "idle"},
        {"who": "zombie", "x": 0.52, "to": 0.52, "y": 0.76, "s": 1.05, "act": "idle"}
      ],
      "events": [
        {"at": 0.08, "type": "notice", "actor": "boxer", "target": "zombie"},
        {"at": 0.22, "type": "guard", "actor": "boxer", "target": "zombie"},
        {"at": 0.42, "type": "attack", "actor": "boxer", "target": "zombie", "action": "punch", "contact": 0.58},
        {"at": 0.58, "type": "react", "actor": "zombie", "source": "boxer", "action": "hit"},
        {"at": 0.76, "type": "recover", "actor": "boxer"}
      ],
      "text": "นักมวยตั้งการ์ดก่อนสวนหมัดเข้าเป้าหมาย",
      "highlight": "สวนหมัด",
      "cam": {
        "from": {"x": 0.5, "y": 0.62, "z": 1},
        "to": {"x": 0.5, "y": 0.62, "z": 1}
      }
    }
  ]
}
```


### Staging correction — commit pending

The first fixture placed the actor centers at x=0.36 and x=0.58 (0.22 canvas-width apart). The renderer does not yet implement automatic attack approach or collision resolution, so this staging distance made the punch visibly miss. The smoke-test fixture is now tightened to x=0.40 and x=0.52 (0.12 canvas-width apart) to place the target within the current procedural arm's approximate reach. This is a fixture correction, not a claim that automatic approach/collision has been implemented. A future increment must add authored approach movement and contact alignment for arbitrary starting distances.

## What should be visible

- Boxer and zombie both face each other from the first frame (unless an explicit actor `face` overrides the inferred event pairing).
- Boxer raises guard at 22% of the shot.
- One `attack` event produces exactly one visible punch: the lead arm winds up, extends once toward the authored contact time (58%), then retracts. The rear arm remains in guard; it must not alternate with the lead arm or read as extra punches.
- Multiple punches must be authored as multiple `attack` events, each with its own contact time and a corresponding target `react` event.
- At 58%, the zombie should snap backward immediately as the punch reaches contact, with a brief whole-body displacement plus a stronger torso recoil; it must not react before 58%.
- The initial knockback should be sharp and brief, then settle smoothly rather than looking like a delayed wobble.
- Boxer returns to idle/recovery at 76%.
- The punch should not repeat continuously just because time advances.

## Latest source update — commit `451a5ae`

- Event-linked opponents now infer their initial facing direction from the first `target` / `source` relationship, so the zombie should face the boxer before the reaction event begins. An explicit actor-level `face` remains authoritative.
- The `hit` pose now uses a stronger torso recoil and body rotation that decays over the reaction event, instead of the previous subtle lean and time-based shake.
- Source-level checks confirm the zombie has no `hit` state at k=0.579 and enters `hit` at k=0.58. JavaScript inline-script syntax check passed. Browser playback and visual timing still require user verification.

## Known limitations of this first increment

- This version drives actor action state and punch timing; it does not yet calculate physical collision or automatically move the target's body away from the impact point.
- `notice` and `turn` currently map to idle plus facing target; independent eye gaze and head tracking are not implemented.
- `approach` selects walk but requires a movement destination in the actor data; event-authored path movement is not yet implemented.
- Only the supported actor rig actions are usable. Unsupported event action strings should not be used in this smoke test.
- Camera Director vocabulary (shot sizes, angle simulation, pan/track/tilt) has not yet been implemented.
- No browser/runtime test has been performed by the maintainer. User testing is required before marking this fixture PASS.

## Report back

Please report whether the punch occurs once, whether the zombie reacts at the right time, and whether the facing direction looks correct. A short screen recording is more useful than a screenshot for timing problems.


## Latest impact-response update — commit `51cc30a`

- The human/zombie hit pose now has a stronger torso lean and rotation, with a short displacement away from the attacker during the first 18% of the reaction event.
- The displacement is applied to the human/zombie render path before drawing the rig, so it affects the zombie in this fixture (not only other actor types).
- The hit event still starts at the authored event time (58%); this change strengthens the visible response at that instant rather than moving the event earlier.
- Static source inspection only. Browser playback and perceived sync still require user verification.


## Contact-sync and recovery refinement — commit `6acd0d2`

- Punch extension now holds back during most of its travel and accelerates through the final 18% before the authored contact point. This is intended to make the visible hand/body contact occur at the same instant as the target's `react` event, rather than making the fist appear to arrive early.
- `react` and `hit` events now use a shorter default event duration of 8% of the scene (unless an explicit `end` is authored). In the 8-second fixture this is about 0.64 seconds, instead of the previous 2.24-second fallback.
- The initial knockback still begins at the authored `react.at` time (58%); the event time itself has not been moved earlier.
- Static source inspection only. Browser playback is still required to verify perceived contact sync and recovery quality.


## Single-punch pose correction — latest increment

- Static inspection found the previous `punch` pose drove both arm angles in opposite phases (`q`), making one attack event look like repeated/alternating punches.
- The lead arm now performs one wind-up → extension → retraction arc driven by the event contact curve; the rear arm stays in a fixed guard pose.
- The event schema is unchanged. One `attack` event means one punch; author additional `attack` / `react` pairs for combinations.
- Actors using the legacy `act: "punch"` without an authored event retain their time-based looping motion; the single-punch event arc applies when an event phase is present.
- Source-only change. Browser playback has not been run by the maintainer; verify that the lead fist visibly contacts at 58% and that only one punch is perceived.
