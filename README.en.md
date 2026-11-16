# A.L.A. — a voice-driven guide robot for the FabLab

![Python](https://img.shields.io/badge/Python-3776AB?logo=python&logoColor=white)
![Raspberry Pi](https://img.shields.io/badge/Raspberry_Pi-A22846?logo=raspberrypi&logoColor=white)
![Amazon Transcribe · Polly](https://img.shields.io/badge/Amazon_Transcribe_%C2%B7_Polly-232F3E)
![openWakeWord](https://img.shields.io/badge/openWakeWord_%2B_Silero_VAD-555555)
![SmartThings API](https://img.shields.io/badge/SmartThings_REST_API-0b7285)
![PyQt](https://img.shields.io/badge/PyQt-41CD52?logo=qt&logoColor=white)
![UART](https://img.shields.io/badge/UART%2FSerial-444444)
![Private code · showcase](https://img.shields.io/badge/Private%20code-showcase-6e7781)

🇧🇷 [Versão em português](README.md)

## Context

A.L.A. (Assistente Laboratorial Autônomo, "autonomous lab assistant") is a guide robot built by a team at the
digital fabrication lab (FabLab) of Samsung Ocean Center Manaus. It welcomes visitors, leads a tour through the
lab's areas (mechanics, programming and 3D printing, electronics and IoT, projects), answers voice commands in
Portuguese and controls part of the room: lights, air conditioning and the laser cutter. The goal is for visits
to happen even when nobody from the team is free to guide them, and for the robot itself to showcase what the
lab builds.

## Highlights

- **Event-driven state machine**: voice, motion, tour and system each have their own states, and every module
  talks only through an **EventBus backed by a priority queue** (error, global stop and obstacle jump ahead of
  any other event).
- **Offline wake word** (openWakeWord + Silero VAD), with the same audio flowing straight into streaming
  transcription; the robot stops talking the moment someone calls it.
- **Listening and speaking in Portuguese**: Amazon Transcribe for recognition and Amazon Polly for speech, with
  generated audio cached and the tour's lines pre-recorded so they don't depend on the network.
- **Keyword-based intents**: stop, start, pause and resume the tour, go to a point, drive and turn, plus more
  than ten room-automation commands.
- **UART navigation**: motion commands sent to the robot's controller over serial, with arrival confirmation,
  travel timeout and an **automatic stop when an obstacle is detected** (ultrasonic sensor).
- **SmartThings automation**: "turn on the management area light", "switch off the computing area", "raise the
  temperature", "turn everything off". A device catalog maps the spoken area to the right devices.
- **Animated face on the display**, running in its own process, with expressions that follow the state
  (listening, thinking, speaking, alert, error).
- **Hardware-free test modes**: inject a recognized sentence, force a line of speech, run without TTS or test
  only the automation from the command line.

## Architecture

```mermaid
flowchart TB
    MIC["USB microphone"] --> AB["AudioBridge<br/>openWakeWord + Silero VAD"]
    AB -- "WAKE_DETECTED" --> BUS
    AB -- "text (streaming)" --> BUS
    AB -.->|"command audio"| STT["STTAdapter<br/>Amazon Transcribe"]
    STT -- "STT_TEXT_READY" --> BUS

    BUS[["EventBus<br/>priority queue"]] --> CTRL

    subgraph Core["Core"]
        CTRL["ALAController<br/>state machine"]
        CTX["RobotContext<br/>system, voice,<br/>motion and tour states"]
        IP["IntentParser"]
        VG["VisitGuide<br/>tour script (JSON)"]
        CTRL --- CTX
        CTRL --> IP
        CTRL --> VG
    end

    CTRL --> TTS["TTSAdapter<br/>Amazon Polly + local audio"]
    TTS --> SPK["USB speaker"]
    TTS -- "TTS_STARTED / FINISHED" --> BUS

    CTRL --> MOT["SerialUARTMotionDriver"]
    MOT <-->|"UART"| MCU["Motion controller<br/>+ ultrasonic sensor"]
    MOT -- "OBSTACLE / POINT_REACHED" --> BUS

    CTRL --> AUTO["SmartThingsAdapter"]
    AUTO --> REG["DeviceRegistry<br/>device catalog"]
    AUTO -- "REST" --> ST["SmartThings<br/>lights, air conditioning,<br/>laser cutter"]

    CTRL --> FACE["FaceProcess<br/>animated face (PyQt)"]
```

Every adapter implements an abstract interface (`STTGateway`, `TTSGateway`, `MotionDriver`,
`AutomationGateway`), so the controller never knows the concrete implementation and each piece has a fake
stand-in for testing.

## State machine

### Voice

```mermaid
stateDiagram-v2
    [*] --> LISTENING_WAKE: boot
    LISTENING_WAKE --> LISTENING_COMMAND: wake word detected
    LISTENING_COMMAND --> LISTENING_WAKE: timed out
    LISTENING_COMMAND --> PROCESSING_COMMAND: audio captured
    PROCESSING_COMMAND --> LISTENING_WAKE: text recognized or failure
    LISTENING_COMMAND --> LISTENING_WAKE: streamed text
    LISTENING_WAKE --> SPEAKING: speech started
    SPEAKING --> LISTENING_WAKE: speech finished
    SPEAKING --> LISTENING_COMMAND: wake word interrupts speech
```

### Tour

```mermaid
stateDiagram-v2
    [*] --> IDLE
    IDLE --> STARTING: start tour
    STARTING --> MOVING_TO_POINT
    MOVING_TO_POINT --> PRESENTING_POINT: reached the point
    PRESENTING_POINT --> MOVING_TO_POINT: speech done, next point
    PRESENTING_POINT --> COMPLETED: last point
    MOVING_TO_POINT --> PAUSED: pause, obstacle or timeout
    PRESENTING_POINT --> PAUSED: pause or speech interrupted
    PAUSED --> MOVING_TO_POINT: resume tour
    MOVING_TO_POINT --> IDLE: stop tour or error
    PRESENTING_POINT --> IDLE: stop tour or error
    PAUSED --> IDLE: stop tour
    COMPLETED --> [*]
```

The system state (`INITIALIZING`, `READY`, `ERROR`, `SHUTDOWN`) sits above everything else: an error event
stops motion before anything else happens.

### One automation command, end to end

```mermaid
sequenceDiagram
    actor V as Visitor
    participant A as AudioBridge
    participant B as EventBus
    participant C as ALAController
    participant I as IntentParser
    participant S as SmartThingsAdapter
    participant R as DeviceRegistry
    participant API as SmartThings API

    V->>A: wake word + turn off the computing light
    A->>B: WAKE_DETECTED
    B->>C: voice goes to LISTENING_COMMAND, face listening
    A->>B: STT_TEXT_READY (text)
    B->>C: event
    C->>I: parse(text)
    I-->>C: Intent LIGHT_OFF, target computacao
    C->>S: execute(light_off, area)
    S->>R: light devices in that area
    R-->>S: online devices
    S->>API: POST switch off command
    API-->>S: ACCEPTED
    S->>B: AUTOMATION_DONE
```

## Stack

| Layer | Technologies |
|---|---|
| Language and core | Python 3, `threading`, `queue.PriorityQueue`, dataclasses, ABCs |
| Voice | openWakeWord + Silero VAD (wake word), Amazon Transcribe (batch via S3 and streaming), Amazon Polly (pt-BR voice) |
| Motion | Raspberry Pi, UART/serial to the motion controller, ultrasonic sensor |
| Automation | SmartThings REST API (Z-Wave lights, air conditioning, the laser cutter's Zigbee switch); earlier Home Assistant adapter |
| Interface | Animated face in PyQt, running as a subprocess driven over `stdin` |
| Tooling | CLI with simulation modes, script that pre-generates the tour audio |

## Technical decisions and challenges

- **Everything goes through the EventBus.** With microphone, serial port, network and display running on
  different threads, no module calls the controller directly. The priority queue makes sure "stop" and
  "obstacle" are handled before a recognized sentence or a finished line of speech.
- **Ports and adapters.** Every external integration has an interface and a fake. That made it possible to
  start with placeholders, swap Home Assistant for SmartThings without touching the controller, and test the
  automation with the robot switched off.
- **A device catalog instead of IDs in code.** `DeviceRegistry` reads the SmartThings export, skips anything
  offline or not controllable (cameras, hubs, groups), classifies each device as light, air conditioning or
  switch, and maps spoken nicknames ("programming", "programming area") to the right area.
- **Air conditioning with no setpoint.** The lab's unit only accepts incremental commands (one degree up, more
  fan), so the voice intents became relative commands rather than "set it to 22 degrees".
- **Voice latency.** Batch transcription (upload, then poll) was too slow for conversation. The team moved to
  keeping capture open, detecting the wake word locally and sending that same audio straight to streaming
  transcription; the fixed tour lines became pre-recorded audio.
- **Operational safety.** An obstacle or "stop" triggers a global stop (motion, speech and capture) that keeps
  the position in the tour script, so the tour resumes where it left off; a critical error stops everything
  and resets the tour.

## My role

**This is a team project.** The repository has 19 commits; **4 are mine**. What I did, checked against the history:

- **The architecture's foundation (initial commit).** The first commit, which is mine, brought in the core:
  the priority-queue EventBus, the state machine enums, `ALAController`, the intent parser, abstract interfaces
  and placeholders, configuration and the UART motion driver.
- **The first real adapters.** STT with Amazon Transcribe, TTS with Amazon Polly and an automation adapter for
  Home Assistant (later replaced).
- **The tour script.** The first version of the file with the visit script.
- **SmartThings automation and per-area voice control.** `SmartThingsAdapter`, `DeviceRegistry` with the device
  catalog and area nicknames, the new voice intents (light, air conditioning, temperature, fan, laser, whole
  area), the command-line test mode and the project README.

The rest is the work of another team member, including the offline wake word with openWakeWord + Silero VAD and
streaming transcription, the PyQt animated face, wiring the tour script into the state machine, the local tour
audio and reorganizing the tour around four physical stops.

Outside this repository, I also worked on the lab's automation with Home Assistant and infrared control (Tuya).

---

The source code is private (it belongs to the FabLab organization). This page is only a showcase of the project.
Contact: [github.com/roger-mont](https://github.com/roger-mont)
