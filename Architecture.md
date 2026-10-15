# AMB82-Mini + Google AI Edge Gallery Smart Giraffe

## Project proposal

The recommended architecture is to make the **AMB82-Mini the giraffe's embodied hardware controller**, while **Google AI Edge Gallery on an Android device provides the higher-level agent/reasoning layer**. Rather than putting a full MCP implementation and a large language model on the microcontroller, the AMB82 exposes a small semantic JSON/HTTP API for perception and physical actions.

```text
                   SMART GIRAFFE
 ┌────────────────────────────────────────────┐
 │                AMB82-Mini                  │
 │                                            │
 │ Camera ──> local vision ──> perception     │
 │ Mic ─────> audio/event detection           │
 │                                            │
 │              Giraffe Agent API             │
 │        HTTP/WebSocket JSON interface       │
 │                    │                       │
 │       ┌────────────┼────────────┐          │
 │       ▼            ▼            ▼          │
 │    Speaker       Servos        LEDs         │
 │   / playback   head / ears   expression     │
 └────────────────────┬───────────────────────┘
                      │ Wi-Fi
                      │
              Android phone/tablet
 ┌────────────────────▼───────────────────────┐
 │          Google AI Edge Gallery            │
 │                                            │
 │        on-device model / agent             │
 │                 │                          │
 │         Giraffe Agent Skill                │
 │                 │                          │
 │        JS Skill / MCP ecosystem            │
 │                 │                          │
 │       ┌─────────┼─────────┐                │
 │       ▼         ▼         ▼                │
 │     look()    emotion()   move()            │
 │     status()  play()      etc.              │
 └────────────────────────────────────────────┘
```

## Why use a small hardware API?

MCP is most useful at the agent/tool integration layer. The AMB82 should remain responsible for deterministic embedded functions: camera inference, GPIO/PWM, audio playback, LEDs, sensor events, and motion safety. A compact HTTP/JSON API is easier to debug, uses less RAM/flash, and prevents the language model from directly commanding low-level PWM or GPIO values.

The agent should therefore request semantic actions such as:

```text
giraffe.look()
giraffe.nod()
giraffe.shake_head()
giraffe.wave()
giraffe.set_emotion("happy")
giraffe.play_sound("hello")
giraffe.status()
```

Do **not** expose low-level commands such as arbitrary servo angles, raw PWM duty cycles, or unrestricted GPIO writes to the agent.

## Two-level AI design

```text
       Fast / reactive                    Cognitive
       AMB82-Mini                         Android device

       local object detection             reasoning
       gesture/event detection            conversation
       camera capture                     agent skills
       LEDs / audio / servos              MCP integrations
       safety interlocks                  planning
```

The AMB82 can turn continuous sensor streams into compact events. For example:

```json
{
  "event": "object_detected",
  "objects": [
    {"class": "person", "confidence": 0.96},
    {"class": "book", "confidence": 0.88}
  ]
}
```

This event-driven approach avoids continuously sending camera frames to the phone.

## Recommended first six tools

| Tool | AMB82 role | Agent purpose |
|---|---|---|
| `look()` | Camera + local detector | Understand surroundings |
| `nod()` | Servo animation | Positive physical response |
| `shake()` | Servo animation | Negative physical response |
| `emotion()` | LED + bounded motion | Express state |
| `play()` | microSD/audio playback | Sounds or prerecorded speech |
| `status()` | Device telemetry | Check device state |

## Example interaction

```text
User: "What am I holding?"

Agent -> look()
AMB82 -> {"objects":[{"label":"book","confidence":0.88}]}
Agent -> formulates an answer
Agent -> emotion("happy")
Agent -> nod()
Phone/TTS or toy audio -> "It looks like a book."
```

## Firmware/API design

The reference firmware in `GiraffeAgent.ino` exposes endpoints such as:

```text
GET /status
GET /look
GET /nod
GET /shake
GET /wave
GET /emotion/happy
GET /emotion/curious
GET /emotion/thinking
GET /emotion/sleepy
GET /play/hello
```

The supplied implementation intentionally contains adapter functions for camera inference, servo control, LEDs, and audio. Replace those adapters with the APIs appropriate to the exact AMB82-Mini board package and peripherals used in the build.

## AI Edge Gallery Skill

The reference `SKILL.md` gives the model a deliberately small physical-action vocabulary. `scripts/index.html` acts as the bridge between the Skill and the AMB82 HTTP API.

The bridge validates actions before sending them to the toy. This is an important design property: model output is never translated into unrestricted hardware access.

## Version 2: events

After command/control is stable, add an event channel, for example:

```text
ws://giraffe.local/events
```

Possible events:

```json
{"event":"person_detected"}
{"event":"wave_detected"}
{"event":"button_pressed"}
{"event":"picked_up"}
{"event":"object_detected","object":"book"}
```

This allows the physical toy to initiate interactions without constant polling.

## Safety and privacy

- Enforce servo travel, speed, and timeout limits in firmware, not in prompts.
- Never expose arbitrary GPIO/PWM control to the language model.
- Keep moving linkages, batteries, and rigid parts inaccessible through the plush exterior.
- Use visible indicators for camera/microphone/network activity.
- Avoid continuous audio/video upload; perform local event detection where practical.
- Do not store images/audio by default.
- Provide a physical power switch and preferably hardware camera/microphone disable controls.
- Validate power supply current and peripheral voltage compatibility before assembly.
- Treat a modified plush as an engineering prototype unless it has undergone applicable toy/product safety testing.

## Suggested implementation order

1. Bring up Wi-Fi and `/status`.
2. Implement bounded LED expressions.
3. Implement one servo and `/nod` with hard travel limits.
4. Add camera capture/local detection and `/look`.
5. Add microSD/audio playback and `/play`.
6. Install the AI Edge Gallery Skill and test the JavaScript bridge.
7. Add event-driven perception only after command/control is reliable.
8. Add richer STT/TTS/MCP integrations at the Android agent layer.

## Repository layout

```text
giraffe-agent/
├── PROPOSAL.md
├── GiraffeAgent.ino
├── SKILL.md
└── scripts/
    └── index.html
```
