# Pose pipeline: temporal handling

This describes what the code in `src/lib/poseLandmarks.js` and `src/lib/poseRecognition.js` does today. It records the
current behaviour only; nothing here changes it. The pipeline recognises visual body postures. It does not decide
whether a prayer is religiously valid.

```
Camera -> MediaPipe worker -> summarizePoseResult -> createPoseLandmarkProcessor (EMA)
       -> createPrayerPoseRecognizer (stable-frame confirmation) -> UI
```

The two mechanisms below act on different things and are independent. Neither replaces the other, and no other temporal
mechanism exists in this integration.

## 1. EMA smoothing (signal level) - `createPoseLandmarkProcessor`

- Smooths landmark coordinates (x, y, z) and joint-angle values to reduce jitter.
- Exponential moving average: `new = old + smoothingFactor * (raw - old)`. Default `smoothingFactor` is 0.55 (weight of the
  current frame; 1 disables smoothing). The Salah screen creates it with `minConfidence: 0.5, smoothingFactor: 0.55`.
- Only reliable joints are smoothed; an unreliable joint drops its history and is never invented.
- History is cleared (`reset`) when: tracking status is `no-pose`, `tracking-lost`, `multiple`, `unknown` or `invalid`;
  the track id changes; the timestamp does not increase; or the gap between frames exceeds `maxGapMs` (default 1000 ms).
- It never decides a posture.

## 2. Stable-frame confirmation (decision level) - `createPrayerPoseRecognizer`

- Decides when a posture is accepted. A candidate state becomes the confirmed (`committed`) state after
  `stableFrames` consecutive frames (default 3); a different candidate restarts the count.
- A frame with no usable evidence classifies as `UNKNOWN`; the previously confirmed state is kept but not extended
  (`stable: false`), and `UNKNOWN` is never forced into a posture.
- History is cleared (`reset`) on tracking loss (`no-pose`, `tracking-lost`, `multiple`, `unknown`, `invalid`), on a
  non-increasing timestamp, or when the gap between frames exceeds `historyFrames` (default 1800; it is compared with
  timestamps in milliseconds, despite the name). After a reset the output is `UNKNOWN` until a fresh candidate is confirmed.
- `classifyPrayerPose` reads the confirmed state as short-term context (for example RUKU -> ITIDAL). This is context for
  visually equivalent postures, not a sequence validator.

## Not in this integration

No prayer-sequence recognition, no contextual resolver and no transition validation. Those are separate, later work.
