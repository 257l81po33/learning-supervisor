# AMB82-Mini + AI Edge Gallery Smart Reading & Writing Eye Guardian

## Project proposal

本專案設計一套以 **AMB82-Mini 為邊緣感知與警報控制器、Android 手機上的 Google AI Edge Gallery 作為高階 AI 與互動介面** 的閱讀／寫作護眼裝置。裝置持續或定期檢查使用者的閱讀距離、紙面照度、光源方向，以及視線是否落在紙面閱讀區域；當任何已啟用的條件持續超出使用者設定範圍時，裝置會發出可設定音量與時長的聲音警報，並透過 Wi-Fi 將事件送到手機。

本提案把感測、規則判斷與安全警報放在 AMB82-Mini／本地韌體端，手機負責設定、紀錄、通知與自然語言說明。**警報不依賴雲端或大型語言模型才能觸發**，以避免網路或 AI 延遲影響基本功能。

> **重要工程假設：** AMB82-Mini 的相機與內建 AI 能力可作為視覺感知入口，但不能直接假定一般物件偵測模型就能精確量出眼睛至紙面的公分數、計算光源入射角，或可靠判斷視線落點。距離需以校正後的視覺幾何／深度方案或額外距離感測器驗證；照度建議加裝校正過的環境光感測器；光源角度可透過多點光感測器或經校正的影像估計；視線追蹤需要能辨識眼部的相機視角、足夠解析度、適當照明與個人校正。以上皆為待實測的工程功能，不應在尚未驗證前宣稱達到醫療級精度。

## System architecture

```text
                    SMART READING & WRITING EYE GUARDIAN
 ┌────────────────────────────────────────────────────────────┐
 │                         AMB82-Mini                         │
 │                                                            │
 │  Camera ──> face / eye / page ROI ──> gaze-on-page estimate│
 │      └────> calibrated geometry ─────> distance estimate   │
 │                                                            │
 │  Ambient-light sensor ───────────────> illuminance (lux)   │
 │  Multi-point light sensors / camera ─> light direction     │
 │  Optional ToF distance sensor ───────> distance cross-check│
 │                                                            │
 │  Local rule engine + calibration + debounce / hysteresis   │
 │                     │                                      │
 │          Reading Guardian Device API                       │
 │           HTTP / WebSocket JSON                            │
 │                     │                                      │
 │       ┌─────────────┼──────────────┐                        │
 │       ▼             ▼              ▼                        │
 │    Buzzer         Status LED    Button / mute              │
 │    alarm          indicator     / calibration              │
 └─────────────────────┬──────────────────────────────────────┘
                       │ Wi-Fi (LAN preferred)
                       │
              Android phone / tablet
 ┌─────────────────────▼──────────────────────────────────────┐
 │                 Google AI Edge Gallery                     │
 │                                                            │
 │    on-device model / agent (where supported)               │
 │                    │                                       │
 │       Reading Guardian Skill / MCP client integration     │
 │                    │                                       │
 │       phone-side bridge / MCP server (if required)         │
 │                    │                                       │
 │    ┌───────────────┼────────────────┐                      │
 │    ▼               ▼                ▼                      │
 │  status()       settings()       history()                 │
 │  distance()     light()          acknowledge_alarm()       │
 │  gaze()         alerts()         calibration_help()        │
 │                                                            │
 │  Dashboard · Threshold settings · Notifications · History  │
 └────────────────────────────────────────────────────────────┘
```

## Why use a small hardware API?

MCP is most useful at the agent/tool integration layer. AMB82-Mini should remain responsible for deterministic embedded functions: camera capture, sensor sampling, local estimation, threshold checks, buzzer control, LEDs, and connection status. A compact HTTP/JSON API is easier to debug and reduces coupling between the firmware and the phone-side agent.

The device should expose semantic functions such as:

```text
guardian.status()
guardian.distance()
guardian.light()
guardian.light_direction()
guardian.gaze()
guardian.settings()
guardian.set_thresholds(...)
guardian.acknowledge_alarm()
guardian.history()
guardian.calibration_status()
```

Do **not** expose unrestricted low-level GPIO writes, raw PWM duty cycles, or arbitrary firmware commands to the model. Settings should be validated against hard-coded safe ranges in firmware. A model may explain a warning or help the user configure the device, but it must not override the local safety rules or silence alarms indefinitely.

## Two-level AI and sensing design

```text
       Fast / deterministic                  Cognitive / interaction
       AMB82-Mini                            Android phone

       camera capture                        conversation
       face / eye / page region              dashboard & settings
       distance estimation                   history summaries
       ambient light sampling                agent skills / MCP
       light direction estimate              natural-language advice
       threshold checks                      phone notifications
       buzzer / LED / mute                    user preferences
       offline alarm operation               optional AI explanation
```

### Measurement strategy

| Requirement | Proposed implementation | Engineering notes |
|---|---|---|
| Reading distance | Camera-based calibrated geometry and/or an I2C ToF distance sensor aimed at a suitable reference point | Define whether the measured value is eye-to-page or head-to-page. A sensor aimed at the page measures its own sensor-to-page distance, not automatically eye-to-page distance. Validate across posture, paper angle, and head movement. |
| Brightness | Calibrated ambient-light sensor placed near the reading plane; optionally compare with camera exposure as a secondary signal | Use lux values and user-configurable minimum/maximum thresholds. Camera auto-exposure is not a reliable lux meter without calibration. |
| Light-source direction | Two or more light sensors positioned with known orientations, or calibrated camera-based shadow/highlight analysis | A single light sensor measures light level, not direction. Distinguish source direction from glare and hand-shadow risk; the latter may be more practical to detect. |
| Gaze on paper | Camera-based eye/iris landmarks plus a calibrated page polygon / region of interest (ROI) | Requires eye visibility, camera placement, calibration and sufficient image quality. Glasses, reflections, low light, and head pose can reduce accuracy. If gaze confidence is low, report “unable to determine” instead of assuming the user is not reading. |
| Alarm | Buzzer or speaker driven by a supported output circuit; status LED | Local alarm remains available without phone connection. Provide a physical mute/acknowledge button and an alarm cooldown to avoid continuous nuisance noise. |
| Phone link | Wi-Fi HTTP API plus WebSocket event stream; optional BLE for provisioning if implemented | Keep the device and phone on a trusted local network. Actual background notifications depend on the phone-side app, operating-system permissions, and the chosen integration. |

## First-release functional requirements

1. **Distance monitoring:** allow the user to set a minimum comfortable reading distance. Trigger a warning only after a configurable persistence period, not from a single noisy measurement.
2. **Illuminance monitoring:** measure and display ambient light in lux; warn when below or above configured thresholds.
3. **Light direction / shadow monitoring:** estimate whether the light comes from the configured preferred direction or whether a strong shadow/glare condition is present. The UI must show confidence or “not calibrated” when this cannot be reliably measured.
4. **Gaze-on-page monitoring:** determine whether the user's estimated gaze falls inside a calibrated paper/reading region. Support “unknown” when eyes or paper are not visible.
5. **Local sound alarm:** any enabled, valid metric that stays outside its configured range for the configured duration can trigger a buzzer pattern. Different patterns may identify distance, brightness, light direction, or gaze warnings.
6. **Phone connection:** show current measurements, device online/offline state, active warning, threshold configuration, and recent events.
7. **Event notifications:** send an event to the phone as soon as a warning begins or clears. Background push-style notifications require a compatible phone-side service/app; the firmware HTTP server alone cannot guarantee Android notifications while an app is closed.
8. **Privacy by default:** process frames locally where possible; do not save or upload raw video by default. Expose a camera-active indicator and a physical power switch.

## Example events

Distance warning:

```json
{
  "event": "threshold_warning",
  "metric": "reading_distance_cm",
  "value": 27.4,
  "threshold": {
    "min": 35.0,
    "max": null
  },
  "unit": "cm",
  "severity": "warning",
  "confidence": 0.91,
  "duration_ms": 1800,
  "timestamp_ms": 1720000000
}
```

Brightness warning:

```json
{
  "event": "threshold_warning",
  "metric": "illuminance_lux",
  "value": 145,
  "threshold": {
    "min": 300,
    "max": 900
  },
  "unit": "lux",
  "severity": "warning",
  "confidence": 0.98,
  "duration_ms": 2500,
  "timestamp_ms": 1720000000
}
```

Gaze status:

```json
{
  "event": "gaze_status",
  "gaze_on_page": false,
  "state": "outside_reading_region",
  "confidence": 0.82,
  "calibrated": true,
  "timestamp_ms": 1720000000
}
```

Unknown / low-confidence state:

```json
{
  "event": "measurement_unavailable",
  "metric": "gaze_on_page",
  "reason": "eyes_not_visible",
  "action": "no_gaze_alarm",
  "timestamp_ms": 1720000000
}
```

These are **illustrative schemas**, not measured output from verified hardware. Timestamps should use a defined clock format in the final implementation (for example, Unix milliseconds after NTP sync, plus uptime for offline operation).

## Recommended first six tools

| Tool | Device / phone role | Agent purpose |
|---|---|---|
| `status()` | Read uptime, Wi-Fi, sensor availability, alarm state | Check whether the guardian is working |
| `distance()` | Return calibrated distance and confidence | Explain whether reading distance is within the user's threshold |
| `light()` | Return illuminance in lux and sensor validity | Explain brightness warnings |
| `light_direction()` | Return estimated direction / shadow or glare state and confidence | Help user adjust lamp position |
| `gaze()` | Return gaze-on-page state, confidence and calibration status | Report whether gaze appears to be on the reading region |
| `settings()` / `set_thresholds()` | Read or update bounded thresholds | Let the user configure warning preferences |

Additional tools may include `history()`, `acknowledge_alarm()`, and `calibration_status()`. Every setter must validate ranges locally and return a structured error if a value is invalid.

## Example interaction

```text
User: "為什麼剛剛一直叫？"

Phone agent -> status()
Device -> {
  "alarm_active": true,
  "active_metrics": ["reading_distance_cm"],
  "reading_distance_cm": 28.1,
  "minimum_distance_cm": 35.0
}

Phone agent -> formulates a response:
"目前偵測距離約 28 公分，低於你設定的 35 公分。
請把紙張稍微移遠，或調整閱讀姿勢。"

Device -> local buzzer remains governed by firmware rules
Phone -> displays warning and records event
```

Another example:

```text
Device -> light sensor reports 160 lux
Local rule engine -> value remains below configured minimum for 2.5 s
Device -> buzzer pattern + LED warning
Device -> emits threshold_warning over WebSocket
Phone -> displays notification and updates dashboard
```

## Firmware / API design

The firmware should provide a compact, documented interface. Endpoint names below are a proposed API contract; implement only after confirming the exact sensor modules and the AMB82-Mini SDK APIs.

```text
GET  /status
GET  /measurements
GET  /distance
GET  /light
GET  /light-direction
GET  /gaze
GET  /settings
POST /settings
POST /alarm/acknowledge
GET  /calibration
POST /calibration/start
GET  /history
WS   /events
```

Example `/measurements` response:

```json
{
  "device": "AMB82-Mini",
  "online": true,
  "alarm_active": false,
  "measurements": {
    "reading_distance_cm": {
      "value": 42.3,
      "valid": true,
      "confidence": 0.89
    },
    "illuminance_lux": {
      "value": 420,
      "valid": true,
      "confidence": 0.98
    },
    "light_direction": {
      "value": "left_front",
      "valid": false,
      "reason": "not_calibrated"
    },
    "gaze_on_page": {
      "value": true,
      "valid": true,
      "confidence": 0.84
    }
  }
}
```

Example settings request:

```json
{
  "distance_min_cm": 35,
  "illuminance_min_lux": 300,
  "illuminance_max_lux": 900,
  "warning_persistence_ms": 2000,
  "alarm_cooldown_ms": 5000,
  "enabled": {
    "distance": true,
    "illuminance": true,
    "light_direction": false,
    "gaze_on_page": false
  }
}
```

The values above are **example defaults for software testing only**, not medical recommendations or universal eye-health standards. Let the user adjust them, document the calibration method, and avoid presenting this prototype as a medical device or as a guarantee against myopia or eye strain.

### Firmware responsibilities

- Connect to Wi-Fi and expose the device API.
- Sample sensors at defined rates and attach validity/confidence to each result.
- Run local threshold logic with persistence, debounce, and hysteresis to prevent noisy readings from causing repeated alarms.
- Trigger the buzzer locally even when the phone is offline.
- Provide a physical mute/acknowledge input without allowing indefinite silent operation by accident.
- Keep a small, bounded ring buffer of event metadata; do not store images by default.
- Implement authentication or a pairing token for control endpoints; avoid exposing the device API to the public internet.
- Return explicit errors for unsupported sensors, missing calibration, stale measurements, and invalid settings.
- Use watchdogs and safe startup states. On sensor failure, show “sensor unavailable” instead of fabricating a measurement.

## AI Edge Gallery Skill / phone integration

The phone-side agent should have a deliberately small tool vocabulary. The model may read device measurements, explain warnings, summarize history, and help the user configure settings within allowed bounds. It should not directly control arbitrary hardware pins or bypass firmware limits.

Suggested skill description:

```text
You are the Reading Guardian assistant.
Use guardian.status() and guardian.measurements() to retrieve current device data.
Explain each measurement with its unit, validity, confidence, and configured threshold.
Do not claim a measurement is accurate if calibration is missing or confidence is low.
Do not diagnose eye disease or promise prevention of myopia.
You may suggest practical changes such as moving the paper farther away,
adjusting the desk lamp, or improving room lighting.
Never disable alarms or change thresholds without explicit user confirmation.
The local firmware, not the language model, controls the physical alarm.
```

### Integration options

1. **Direct MCP integration:** if the installed Google AI Edge Gallery Android version supports registering an MCP server over Streamable HTTP, run a compatible MCP server on a trusted host that forwards only approved semantic tools to the AMB82 HTTP API.
2. **Phone app / JavaScript bridge:** create a small companion dashboard or local bridge that calls the AMB82 API. Do not assume a standalone `SKILL.md` or `scripts/index.html` will automatically execute inside AI Edge Gallery; confirm the extension mechanism supported by the installed version.
3. **Simple first prototype:** use the phone browser to open the device dashboard and test `/status`, `/measurements`, and `/events` before adding an LLM or MCP.

A practical network detail: `guardian.local` / mDNS may not work on every router or Android configuration. The first version should display the device IP address and offer a simple reconnect procedure.

## Version 2: event-driven monitoring

After sensor measurements, local alarms, and phone control are stable, add a WebSocket event channel:

```text
ws://<device-ip>/events
```

Possible events:

```json
{"event":"device_online"}
{"event":"threshold_warning","metric":"reading_distance_cm","value":28.1}
{"event":"threshold_cleared","metric":"reading_distance_cm","value":39.2}
{"event":"threshold_warning","metric":"illuminance_lux","value":160}
{"event":"gaze_status","gaze_on_page":false,"confidence":0.84}
{"event":"sensor_unavailable","sensor":"ambient_light"}
{"event":"alarm_acknowledged"}
```

Use event-driven updates rather than continuously polling high-rate measurements. The phone should reconnect with backoff, request a fresh status after reconnecting, and show a clear offline state if the device is unreachable. For Android notifications when the app is not active, implement and test a compatible companion app/notification service; a WebSocket connection alone does not guarantee delivery while the operating system suspends the app.

## Safety, privacy, and limitations

- **Not a medical device:** this project is a posture / environment reminder prototype, not a diagnostic tool or a guarantee of preventing myopia, eye strain, or other eye conditions.
- **Distance accuracy:** camera-based distance estimates require calibration and can be affected by head pose, lens geometry, occlusion, and paper position. An external ToF sensor can cross-check distance but may measure sensor-to-paper rather than eye-to-paper distance.
- **Light level:** a light sensor needs suitable placement and calibration. Its reading may differ from illumination across the whole page.
- **Light direction:** a single lux sensor cannot identify the light-source angle. Use multiple sensors or a validated vision method; otherwise mark this feature as experimental.
- **Gaze tracking:** eyes may be hidden by glasses, reflections, head movement, hair, or poor lighting. Low confidence must produce an “unknown” state, not a false accusation that the user is distracted.
- **Alarm usability:** provide configurable volume, short alarm patterns, cooldown, physical acknowledgement, and an option to temporarily pause monitoring. Avoid an excessively loud or continuous alarm.
- **Hardware safety:** verify voltage levels, current draw, pin assignments, and I2C addresses before wiring. Do not power a buzzer or high-current peripheral directly from a GPIO unless the electrical specifications explicitly permit it; use a suitable driver circuit.
- **Privacy:** process camera frames locally whenever practical; do not upload or save raw images/audio by default. Provide a visible camera-active indicator and physical power control.
- **Network security:** do not expose unauthenticated control endpoints to the internet. Prefer a trusted local network and a pairing token; do not include Wi-Fi passwords or secrets in source control.
- **Fail-safe behavior:** if a sensor is disconnected or calibration expires, report the fault and avoid presenting stale values as current.
- **Prototype status:** complete bench testing and user testing before relying on the device for daily use.

## Suggested implementation order

1. Confirm the exact AMB82-Mini board revision, camera module, Arduino core version, and available peripherals.
2. Bring up Wi-Fi and `GET /status`; display IP, uptime, and connection state.
3. Connect a calibrated ambient-light sensor over a supported interface and implement `GET /light`.
4. Add local threshold logic, LED indication, buzzer driver, acknowledgement button, and cooldown.
5. Add the distance sensor or camera-based distance estimator; record its calibration procedure and error across realistic postures.
6. Add a multi-sensor or vision-based light-direction / shadow estimator and validate it against known lamp positions.
7. Add camera-based face/eye/page-region detection; implement calibration and confidence-aware `GET /gaze`.
8. Implement `GET /measurements`, `GET /settings`, validated `POST /settings`, and a small event history.
9. Add WebSocket `/events`; test reconnect, packet loss, and phone-offline behavior.
10. Build the phone dashboard and notifications; verify Android background behavior on the target phone.
11. Integrate Google AI Edge Gallery through a supported MCP/agent mechanism only after the device API is reliable.
12. Run repeatable tests for distance error, lux error, direction classification, gaze accuracy, alarm latency, and false alarms. Document measured results in the repository.

## Acceptance tests

| Test | Pass condition |
|---|---|
| Wi-Fi startup | Device obtains an IP and `/status` returns valid JSON |
| Sensor disconnect | Device marks the affected measurement invalid and does not reuse stale data as current |
| Distance threshold | A sustained out-of-range test triggers local alarm within the configured persistence window plus measured processing latency |
| Light threshold | Calibrated sensor readings below/above the test thresholds trigger the expected event |
| Light direction | Each supported lamp placement is tested against labelled reference positions; unvalidated positions return unknown |
| Gaze region | Test with multiple paper positions and head poses; report accuracy and unknown rate rather than assuming perfect tracking |
| Phone disconnect | Local alarm continues; events are buffered within a bounded limit and status reflects phone offline |
| Phone reconnect | Dashboard refreshes current state and receives new events without treating old events as live |
| Invalid settings | Firmware rejects out-of-range values with a structured error |
| Privacy | No raw camera frames are persisted or transmitted by default |

## Repository layout

```text
reading-guardian/
├── README.md
├── proposal.md
├── firmware/
│   └── ReadingGuardian.ino
├── phone/
│   ├── SKILL.md
│   ├── mcp-server/             # optional, only if required by the selected integration
│   └── dashboard/
│       └── index.html
├── docs/
│   ├── HARDWARE.md
│   ├── CALIBRATION.md
│   ├── API.md
│   ├── TEST_PLAN.md
│   └── PRIVACY.md
└── .gitignore
```

## References

- AMB82-Mini / Ameba Arduino documentation: https://ameba-doc-arduino-sdk.readthedocs-hosted.com/en/latest/ameba_pro2/amb82-mini/index.html
- AMB82-Mini getting started and peripheral pin reference: https://www.amebaiot.com/en/amebapro2-amb82-mini-arduino-getting-started/
- Google AI Edge Gallery MCP integration announcement (May 19, 2026): https://developers.googleblog.com/a-smarter-google-ai-edge-gallery-mcp-integration-notifications-and-session-continuity/

---

**Project status:** Proposal / hardware validation required.

**Core principle:** the AMB82-Mini performs sensing and deterministic local alarms; the phone provides configuration, history, notifications, and optional AI assistance. Each sensing feature must be calibrated and validated before being treated as reliable.
