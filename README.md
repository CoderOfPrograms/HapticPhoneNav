# TextHeartbeat Pocket Navigator

Vue + Vite app that uses `web-haptics` and phone motion sensors to provide directional haptic guidance while the phone is in your pocket.

## Features

- Exactly four haptic signals: `straight`, `left`, `right`, `wrong-way`.
- Left and right patterns are high-contrast and directional.
- Straight signal can emit from either top or bottom anchor button.
- 3-second countdown pocket calibration to infer top/bottom orientation.
- GPS-based heading checks with broad tolerance (does not micro-correct left/right drift).
- Auto wrong-way detection only when heading is substantially off course.
- Demo playback sequence: straight -> left -> straight -> right -> wrong-way.
- Audio debug mode (uses `web-haptics` debug audio for non-vibration testing).

## Run

```bash
npm install
npm run dev -- --host
```

Open the dev URL from your phone on the same Wi-Fi.

## How To Use On Phone

1. Tap `Enable Motion + GPS`.
2. Put the phone in your pocket and tap `Start 3s Pocket Countdown`.
3. Let countdown complete to auto-select top or bottom straight emitter.
4. Use `Top Straight Emitter` or `Bottom Straight Emitter`, plus `Strong Left`, `Strong Right`, and `Wrong-Way Alert`.
5. Tap `Stop All Buzzing` to silence all patterns.

## Demo Script Mode

1. Tap `Start Demo Playback` for the built-in sequence.
2. Tap `Stop Demo Playback` to end early.
3. Tap `Reload Demo Sequence` to restore defaults.

## Notes

- Haptics require mobile browser vibration support.
- GPS heading quality varies by speed and environment.
- Slow movement can reduce heading availability; keep moving briefly if heading is missing.
- Turn completion auto-detects when course is close to expected turn heading.
