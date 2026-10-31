# Phase 02 — Block Pose Retest

## Goal

Retest the revised articulated guard geometry in the human/zombie renderer. This fixture checks a face-level high block followed by a rib-level body block. It is a visual smoke test, not a collision-physics test.

## How to run

1. Open [Clip_Maker_Enhanced.html](../Clip_Maker_Enhanced.html) from branch `phase-02-action-cinema` in Chrome or Edge.
2. Paste the JSON fixture below into the JSON input.
3. Set Duration to **ใช้ความยาวจาก JSON**.
4. Press Play and watch the full 8 seconds. Replay once to check consistency.

## Fixture JSON

```json
{
  "duration": 8,
  "cast": {
    "boxer": { "c": "human", "col": "#d8322f", "gloves": true },
    "opponent": { "c": "zombie", "col": "#78936c", "gloves": true }
  },
  "scenes": [
    {
      "theme": "ring",
      "sky": ["#171a2e", "#4b3545"],
      "ground": "#29232b",
      "actors": [
        { "who": "boxer", "x": 0.40, "to": 0.40, "y": 0.76, "s": 1.05, "act": "idle" },
        { "who": "opponent", "x": 0.52, "to": 0.52, "y": 0.76, "s": 1.05, "act": "idle" }
      ],
      "events": [
        { "at": 0.08, "end": 0.43, "type": "attack", "actor": "opponent", "target": "boxer", "action": "punch", "contact": 0.40 },
        { "at": 0.18, "end": 0.45, "type": "block", "actor": "boxer", "target": "opponent", "action": "blockHead" },
        { "at": 0.52, "end": 0.85, "type": "attack", "actor": "opponent", "target": "boxer", "action": "punch", "contact": 0.76 },
        { "at": 0.58, "end": 0.86, "type": "block", "actor": "boxer", "target": "opponent", "action": "blockBody" },
        { "at": 0.90, "type": "recover", "actor": "boxer" },
        { "at": 0.90, "type": "recover", "actor": "opponent" }
      ],
      "text": "ยกการ์ดรับหมัด แล้วปิดศอกป้องกันลำตัว",
      "highlight": "ป้องกันลำตัว",
      "cam": {
        "from": { "x": 0.5, "y": 0.62, "z": 1 },
        "to": { "x": 0.5, "y": 0.62, "z": 1 }
      }
    }
  ]
}
```

## Expected visual result

- At 18–45%, the boxer raises a compact high guard. Both arms begin at the shoulder line; elbows bend below the shoulder; fists sit beside the cheeks, not at the outer edges of the frame and not over the center of the face.
- At 58–86%, the near-side elbow drops toward the rib line while the far-side glove remains high. The near fist should sit in front of the body/rib area.
- Arms should read as connected shoulder → upper arm → elbow → forearm → fist segments, with a visible elbow joint and no detached `< >` silhouette.
- The boxer must remain in the block pose at the opponent's authored contact times (40% and 76%), rather than entering hit recoil.
- The punch-only and dodge + block fixtures in `PHASE_02_ACTION_EVENTS_TEST.md` should remain unchanged.

## Known limits

The current block matching is a deterministic event-timing and draw-order approximation. It does not solve arbitrary glove/forearm collisions or calculate damage. Browser playback and visual acceptance must be confirmed by the user; source edits alone are not a PASS.
