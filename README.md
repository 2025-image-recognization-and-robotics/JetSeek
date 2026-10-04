# JetSeek

**Object-searching JetBot, driven by a PC.** Pick a target such as `cup`, `bottle` or `cell phone` and the JetBot wanders the room looking for it. When YOLO spots the target, the robot turns to face it and drives up until it is close.

This repository is the **PC-side server**. The JetBot only streams camera frames and runs motor commands. Detection, search behavior and control all run on the PC.

## How it works

```
 ┌──────────── JetBot ────────────┐            ┌──────────────────────── PC (this repo) ────────────────────────┐
 │                                │  JPEG      │                                                                │
 │  camera ── stream client ──────┼──────────► │  ImageServer ──► EventBus ──► YoloInference ──┐                │
 │                                │  :8080     │                                               │ target found?  │
 │                                │            │  RandomWalkDaemon (scan + relocate) ──────────┤                │
 │  motor server ◄────────────────┼─────────── │  Controller ◄── drive/set_velocity ◄── Commander (10 Hz)       │
 │                                │  JSON      │                                                                │
 └────────────────────────────────┘  :8081     │  Tk GUI ──► choose target class                                │
                                               └────────────────────────────────────────────────────────────────┘
```

1. **Search.** `RandomWalkDaemon` repeats a two-phase loop. It scans by turning 360° in six 60° steps. Then it relocates: it turns a random 40–120° (alternating left and right) and drives forward for 1–2 s.
2. **Detect.** `YoloInference` runs on every incoming frame and keeps only boxes that match the selected target class. If there are several, it uses the largest one.
3. **Approach.** If the target's center is more than 200 px off the image center, the robot turns toward it. Once centered, it drives forward until the box covers half of the frame, then stops.
4. **Arbitrate.** Ten times a second, `Commander` sends the YOLO command if a target is visible and the random-walk command if not.

## Project structure

```
src/
├── app/
│   ├── main.py                 # Entry point: wires up and runs every service
│   └── gui.py                  # Tkinter window for choosing the target class
├── communication/
│   ├── image_receiver/
│   │   └── server.py           # TCP server that receives length-prefixed JPEG frames
│   └── jetbot_api/
│       └── controller.py       # Persistent TCP client that sends motor commands to the JetBot
├── perception/
│   └── yolo_inference.py       # YOLOv8 detection, target filtering and approach control
├── random_walk/
│   └── random_walk.py          # Search behavior (scan + relocate), plus a turn calibration mode
├── task_manager/
│   └── motor_controller.py     # Commander: picks YOLO or random walk and publishes velocity
└── core/
    ├── config.py               # AppConfig, loaded from environment variables / .env
    ├── events.py               # Small async pub/sub event bus
    └── logging.py              # Logging setup
run_test.py                     # Debug runner: image server + YOLO + OpenCV preview window
```

## Getting started

### Requirements

- Python 3.10+
- A JetBot on the same network that runs:
  - a camera client that streams JPEG frames to this server, and
  - a motor server that listens on port `8081` for JSON commands

### Install

```bash
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

The YOLOv8 weights (`yolov8s.pt`) are downloaded automatically the first time you run it.

### Run

```bash
python -m src.app.main
```

1. The server starts listening for frames on `0.0.0.0:8080`.
2. Start the camera stream on the JetBot.
3. In the **Select YOLO Target** window, choose a class. The robot starts searching for it.
4. Press `Ctrl+C` to stop.

Supported targets: `handbag`, `remote`, `bottle`, `cup`, `laptop`, `mouse`, `cell phone`, `wallet`, `scissors`, `book`, `person`. To change the list, edit `target_classes_list` in [src/app/main.py](src/app/main.py).

## Configuration

Set these as environment variables or in a `.env` file at the project root:

| Variable      | Default   | Description                                 |
| ------------- | --------- | ------------------------------------------- |
| `APP_HOST`    | `0.0.0.0` | Address the image server binds to           |
| `APP_PORT`    | `8080`    | Port the image server listens on            |
| `YOLO_DEVICE` | `cpu`     | Inference device, e.g. `cpu`, `cuda:0`, `mps` |
| `IMG_WIDTH`   | `640`     | Width of the camera frames                  |
| `IMG_HEIGHT`  | `480`     | Height of the camera frames                 |

> **JetBot address:** the robot's IP and port are currently hard-coded in [src/communication/jetbot_api/controller.py](src/communication/jetbot_api/controller.py) (`172.20.10.9:8081`). Change them to match your robot.

## Protocols

**Camera frames (JetBot → PC, TCP `:8080`):** each frame is sent as a 4-byte big-endian length followed by that many bytes of JPEG data.

**Motor commands (PC → JetBot, TCP `:8081`):** one JSON object per line. Values are clamped to `[-1.0, 1.0]`.

```json
{"left": 0.15, "right": 0.15}
```

## Tuning

| What                       | Where                                   | Parameter                                        |
| -------------------------- | --------------------------------------- | ------------------------------------------------ |
| Detection confidence       | `src/app/main.py`                       | `conf_threshold` (0.5)                           |
| Centering tolerance        | `src/perception/yolo_inference.py`      | `center_deadzone` (200 px)                       |
| Approach / turn speed      | `src/perception/yolo_inference.py`      | `_calculate_velocity`                            |
| Search speeds and timing   | `src/random_walk/random_walk.py`        | `forward_speed`, `turn_speed`, `min/max_move_time` |
| Turn calibration           | `src/random_walk/random_walk.py`        | `seconds_per_degree` (0.0105)                    |

To recalibrate turning on a new floor surface, set `is_calibration_mode = True` in `RandomWalkDaemon`. The robot then repeats 90° left, 90° right and 180° turns. Adjust `seconds_per_degree` until the angles are accurate.

## Debugging

```bash
python run_test.py
```

This starts only the image server and YOLO, with an OpenCV window for previewing detections. The robot does not move in this mode.
