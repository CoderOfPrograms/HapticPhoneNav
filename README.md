# TextHeartbeat Pocket Navigator

Vue + Vite app that uses `web-haptics` and phone motion sensors to provide directional haptic guidance while the phone is in your pocket.

## Features

- Detects device heading and pocket up/down axis from motion sensors.
- Continuous forward pulse while on course.
- Distinct left and right turn buzz patterns.
- Wrong-way alert with a heavier long buzz.
- Auto wrong-way detection when heading drifts too far from calibrated forward direction.
- Audio debug mode (uses `web-haptics` debug audio so patterns can be tested without phone vibration).
- Route script playback mode for timed step-by-step haptic cues.
- Live route import from public directional APIs:
	- Nominatim (geocoding)
	- OSRM (walking directions)

## Run

```bash
npm install
npm run dev -- --host
```

Open the dev URL from your phone on the same Wi-Fi.

## How To Use On Phone

1. Tap `Enable Motion Sensors` (required on iOS).
2. Stand facing your forward direction and place the phone in your pocket.
3. Tap `Calibrate Forward + Pocket Up/Down`.
4. Use `Start Forward Pulse`, `Cue Left Turn`, `Cue Right Turn`, or `Trigger Wrong-Way Alert`.
5. Tap `Stop All Buzzing` to silence all patterns.

## Route Script + API Mode

1. Use `Load Demo Script` for a built-in timed route.
2. Or enter `Origin` and `Destination`, then tap `Load Real Route (OSM + OSRM API)`.
3. Tap `Start Script Playback` to run haptic guidance through route steps.
4. Turn on `Audio Debug Mode` to hear haptic patterns on devices without vibration.

## Notes

- Haptics require mobile browser vibration support.
- Compass heading quality varies by browser and environment.
- Turn completion auto-detects when heading is close to the new turn target.
- Public APIs can rate-limit or fail occasionally; use the demo script as fallback.
- Google Maps Directions can be added later with your API key if needed.
