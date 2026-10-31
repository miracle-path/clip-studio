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
        {"who": "boxer", "x": 0.36, "to": 0.36, "y": 0.76, "s": 1.05, "act": "idle"},
        {"who": "zombie", "x": 0.58, "to": 0.58, "y": 0.76, "s": 1.05, "act": "idle"}
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

## What should be visible

- Boxer initially stands facing the zombie.
- Boxer raises guard at 22% of the shot.
- Boxer performs one punch; the arm extends toward the authored contact time (58%), then retracts by the recovery event.
- Zombie begins its hit reaction at 58%, facing the boxer.
- Boxer returns to idle/recovery at 76%.
- The punch should not repeat continuously just because time advances.

## Known limitations of this first increment

- This version drives actor action state and punch timing; it does not yet calculate physical collision or automatically move the target's body away from the impact point.
- `notice` and `turn` currently map to idle plus facing target; independent eye gaze and head tracking are not implemented.
- `approach` selects walk but requires a movement destination in the actor data; event-authored path movement is not yet implemented.
- Only the supported actor rig actions are usable. Unsupported event action strings should not be used in this smoke test.
- Camera Director vocabulary (shot sizes, angle simulation, pan/track/tilt) has not yet been implemented.
- No browser/runtime test has been performed by the maintainer. User testing is required before marking this fixture PASS.

## Report back

Please report whether the punch occurs once, whether the zombie reacts at the right time, and whether the facing direction looks correct. A short screen recording is more useful than a screenshot for timing problems.
