# BENAX — AI-Powered Single-Speaker Face Recognition & Camera Tracking

[![Python](https://img.shields.io/badge/Python-3.11+-blue.svg)](https://www.python.org/)
[![OpenCV](https://img.shields.io/badge/OpenCV-Computer%20Vision-green.svg)](https://opencv.org/)
[![MQTT](https://img.shields.io/badge/MQTT-Mosquitto-orange.svg)](https://mosquitto.org/)
[![ESP8266](https://img.shields.io/badge/ESP8266-Servo%20Controller-red.svg)](https://www.espressif.com/)
[![License](https://img.shields.io/badge/License-Educational-lightgrey.svg)]()

Locks onto **one pre-enrolled speaker**, ignores every other face in frame, and rotates a camera horizontally to keep that speaker centered. A PC runs the AI pipeline and publishes motor commands over **MQTT/Wi-Fi**; an **ESP8266** drives the **servo**.

```text
USB Camera → [PC: detect → recognize speaker → track → command] → MQTT → ESP8266 → Servo → camera pans
```

## Documentation

The project includes Mermaid-based architecture and workflow documentation:

| File                                | Description                               |
| ----------------------------------- | ----------------------------------------- |
| `docs/01-system-overview.md`        | High-level system architecture            |
| `docs/02-runtime-pipeline.md`       | Runtime recognition and tracking pipeline |
| `docs/03-mqtt-communication.md`     | MQTT communication flow                   |
| `docs/04-speaker-enrollment.md`     | Enrollment process and profile generation |
| `docs/05-tracking-state-machine.md` | Tracking/searching state machine          |

These files can be previewed directly by GitHub, VS Code, Obsidian, GitLab, or any Markdown viewer with Mermaid support.

---

## 1. Repository Layout

```text
embedded/
├─ config.json
├─ enroll.py
├─ track.py
├─ dashboard.py
├─ servo_test.py
├─ requirements.txt
├─ src/
│  ├─ camera.py
│  ├─ config.py
│  ├─ recognizer.py
│  ├─ tracker.py
│  ├─ mqtt_client.py
│  └─ logger.py
├─ templates/
│  └─ dashboard.html
├─ firmware/
│  └─ esp8266_servo/
│     └─ esp8266_servo.ino
├─ tools/
│  ├─ setup_broker.ps1
│  ├─ download_model.py
│  └─ fetch_pack.py
├─ docs/
│  ├─ 01-system-overview.md
│  ├─ 02-runtime-pipeline.md
│  ├─ 03-mqtt-communication.md
│  ├─ 04-speaker-enrollment.md
│  └─ 05-tracking-state-machine.md
├─ models/
├─ data/
│  └─ speaker_profile.json
└─ logs/
```

---

## 2. One-Time Setup

### 2.1 Python Dependencies

Install all required packages:

```powershell
.\venv\Scripts\python.exe -m pip install -r requirements.txt
```

### Recognition Models

The system supports fully offline recognition using:

* `models/face_landmarker.task`
* `models/embedder_arcface.onnx`

No internet connection is required once these files are present.

The recognizer automatically prefers the offline pipeline and falls back to InsightFace if required.

Optional model pack download:

```powershell
python tools\fetch_pack.py buffalo_s
```

---

### 2.2 MQTT Broker (Mosquitto)

The MQTT broker runs locally on the PC.

Current configuration:

```text
Broker Host : 10.11.74.164
Broker Port : 1883
Command Topic : benax/camera/command
Status Topic  : benax/camera/status
```

Start Mosquitto:

```powershell
powershell -ExecutionPolicy Bypass -File tools\setup_broker.ps1
```

Verify:

```powershell
mosquitto_sub -h 10.11.74.164 -p 1883 -t "#" -v
```

---

### 2.3 ESP8266 Firmware

Configure:

```cpp
WIFI_SSID
WIFI_PASS

MQTT_HOST = "10.11.74.164"
MQTT_PORT = 1883
```

Install required Arduino components:

```powershell
arduino-cli config init

arduino-cli config add board_manager.additional_urls ^
http://arduino.esp8266.com/stable/package_esp8266com_index.json

arduino-cli core update-index

arduino-cli core install esp8266:esp8266

arduino-cli lib install "PubSubClient"
```

Compile:

```powershell
arduino-cli compile --fqbn esp8266:esp8266:nodemcuv2 firmware\esp8266_servo
```

Upload:

```powershell
arduino-cli upload -p COM5 --fqbn esp8266:esp8266:nodemcuv2 firmware\esp8266_servo
```

---

### 2.4 Hardware Bring-Up

Test the servo independently:

```powershell
python servo_test.py
```

Interactive mode:

```powershell
python servo_test.py --interactive
```

---

## 3. Running the System

### Terminal 1 — Start Broker

```powershell
powershell -ExecutionPolicy Bypass -File tools\setup_broker.ps1
```

---

### Step A — Enroll Speaker

Only the target speaker should appear in the frame.

```powershell
python enroll.py --name speaker --samples 20
```

This creates:

```text
data/speaker_profile.json
```

---

### Step B — Start Tracking

```powershell
python track.py
```

The system will:

1. Detect faces
2. Identify the enrolled speaker
3. Ignore all other faces
4. Generate tracking commands
5. Publish MQTT commands
6. Log decisions

Dry run:

```powershell
python track.py --no-mqtt
```

---

### Web Dashboard

Alternative interface:

```powershell
python dashboard.py
```

Open:

```text
http://localhost:5000
```

Additional examples:

```powershell
python dashboard.py --no-mqtt

python dashboard.py --port 8000

python dashboard.py --camera 1
```

---

## 4. MQTT Verification

Subscribe:

```powershell
mosquitto_sub -h 10.11.74.164 -p 1883 -t "benax/camera/#" -v
```

Publish:

```powershell
mosquitto_pub -h 10.11.74.164 -p 1883 -t "benax/camera/command" -m "RIGHT:5"

mosquitto_pub -h 10.11.74.164 -p 1883 -t "benax/camera/command" -m "SCAN"

mosquitto_pub -h 10.11.74.164 -p 1883 -t "benax/camera/command" -m "CENTER"
```

ESP8266 heartbeat:

```text
benax/camera/status
```

Example:

```json
{
  "angle": 90,
  "mode": "TRACKING"
}
```

---

## 5. Evidence Logging

Every decision is stored in:

```text
logs/session_<timestamp>.csv
logs/session_<timestamp>.jsonl
```

| Column        | Description             |
| ------------- | ----------------------- |
| timestamp_iso | ISO timestamp           |
| epoch         | Unix timestamp          |
| speaker_id    | Enrolled speaker        |
| recognized    | Recognition status      |
| confidence    | Similarity score        |
| num_faces     | Faces detected          |
| error_norm    | Smoothed tracking error |
| status        | Tracking status         |
| command       | MQTT command            |

---

## 6. Demonstration Scenarios

### Multiple Faces

Only the enrolled speaker is tracked.

Other detected faces are ignored.

---

### Occlusion and Recovery

Temporarily hide the speaker:

```text
LOCKED → STOPPED → SEARCHING
```

Reappear:

```text
SEARCHING → LOCKED
```

---

### Lateral Movement

Move left or right.

The system publishes:

```text
LEFT:n
RIGHT:n
CENTER
```

and re-centers the camera.

---

## 7. Configuration

Main settings are stored in:

```text
config.json
```

| Key                              | Purpose                 |
| -------------------------------- | ----------------------- |
| recognition.similarity_threshold | Recognition strictness  |
| recognition.detect_every         | Detection frequency     |
| tracking.deadband_frac           | Center dead-zone        |
| tracking.max_step_deg            | Maximum servo step      |
| tracking.min_step_deg            | Minimum servo step      |
| tracking.smoothing               | Tracking filter         |
| tracking.invert_direction        | Reverse servo direction |
| tracking.lost_grace_frames       | Delay before loss       |
| tracking.scan_after_frames       | Delay before search     |
| camera.flip_horizontal           | Mirror preview          |

---

## 8. Troubleshooting

### Camera Will Not Open

Try:

```powershell
python track.py --camera 1
```

---

### Speaker Never Locks

Re-enroll:

```powershell
python enroll.py --samples 20
```

Improve lighting and pose variation.

---

### Other People Are Accepted

Increase:

```json
recognition.similarity_threshold
```

---

### ESP8266 Not Connecting

Verify:

```text
MQTT_HOST = 10.11.74.164
MQTT_PORT = 1883
```

Check:

```powershell
mosquitto_sub -h 10.11.74.164 -p 1883 -t "#" -v
```

and monitor ESP8266 Serial at:

```text
115200 baud
```

---

### Camera Moves in Wrong Direction

Set:

```json
tracking.invert_direction = true
```

---

### InsightFace Missing

Download fallback assets:

```powershell
python tools\download_model.py
```