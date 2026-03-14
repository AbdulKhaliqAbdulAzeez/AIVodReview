# AI VOD Review Tool — Project Framework & Build Plan

## Overview

An automated Overwatch VOD analysis pipeline that ingests raw gameplay footage, extracts structured game-state data using computer vision, and generates timestamped coaching feedback by applying game-logic rules to that data.

The system is architected as a four-stage ETL pipeline:

```
MP4 Video → [Perception] → [Tracking & State] → [Logic Engine] → [Report]
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| Language | Python 3.11+ |
| Computer Vision | Ultralytics YOLOv11 |
| GPU Acceleration | CUDA (NVIDIA) |
| Video Decoding | OpenCV (`cv2`) |
| Object Tracking | ByteTrack (built into Ultralytics) |
| Data Processing | Pandas |
| Storage | SQLite (local dev) → PostgreSQL (production) |
| Output/Reporting | JSON + Markdown/HTML report template |
| Annotation | Label Studio or Roboflow |

---

## Project Directory Structure

```
AIVodReviewTool/
├── data/
│   ├── raw_videos/          # Input MP4 VOD files
│   ├── frames/              # Sampled frames (temp, cleared after processing)
│   ├── annotations/         # YOLO-format label files for training
│   └── processed/           # Output CSVs / SQLite databases per VOD
│
├── models/
│   ├── hero_detector/       # Trained weights: enemy/ally hero detection
│   ├── ui_reader/           # Trained weights: health, ult charge, cooldowns
│   └── killfeed_reader/     # Trained weights: kill feed OCR/classification
│
├── pipeline/
│   ├── ingest.py            # Video → sampled frames
│   ├── perceive.py          # Frames → YOLO detections
│   ├── track.py             # Detections → tracked time-series DataFrame
│   ├── analyze.py           # DataFrame → flagged mistake events
│   └── report.py            # Flagged events → human-readable report
│
├── training/
│   ├── collect_frames.py    # Script to extract frames for annotation
│   ├── train_hero.py        # Training script for hero detector model
│   ├── train_ui.py          # Training script for UI reader model
│   └── train_killfeed.py    # Training script for kill feed model
│
├── tests/
│   └── ...
│
├── config.yaml              # Global config: FPS sample rate, confidence thresholds, paths
├── main.py                  # Entry point: orchestrates the full pipeline
├── requirements.txt
└── planning.md
```

---

## Stage 1 — Ingestion (`pipeline/ingest.py`)

**Goal:** Decode the input video and emit a stream of sampled frames for downstream processing.

- Accept an MP4 file path as input.
- Use OpenCV to open the video stream and read its native FPS and resolution.
- Sample at a configurable rate (default: **10 FPS**). At 60 FPS native video, this means processing every 6th frame — sufficient to capture all meaningful UI state changes while reducing compute load by ~83%.
- Write sampled frames to a temp directory (or pass them in-memory via a generator to avoid disk I/O).
- Emit metadata alongside each frame: `{ frame_index, timestamp_ms, source_file }`.

**Key decisions:**
- Generator-based frame emission to keep memory usage flat regardless of VOD length (no loading the full video into RAM).
- Configurable sample rate in `config.yaml` to allow tuning without code changes.

---

## Stage 2 — Perception (`pipeline/perceive.py`)

**Goal:** Run YOLO inference on each frame and return structured detection data.

Three separate YOLO models run against each frame:

### Model A — Hero Detector
- **Detects:** Enemy heroes (red team), allied heroes (blue team), and the player character.
- **Classes:** One class per hero per team (e.g., `enemy_genji`, `ally_lucio`), plus a generic `enemy` and `ally` fallback class for early training stages.
- **Output:** Bounding box `(x1, y1, x2, y2)`, confidence, class label, track ID (assigned by ByteTrack).

### Model B — UI Reader
- **Detects:** Specific UI regions — player health bar (value + state: normal/critical), ultimate charge percentage, ability cooldown indicators (ready vs. on cooldown).
- **Approach:** Crop to known UI regions (bottom-left for health/ults) before inference to reduce noise from gameplay elements. Use OCR or a classifier head for reading numeric values like health percentage.
- **Output:** Structured dict per frame: `{ health_pct, ult_pct, ability_1_ready, ability_2_ready, ability_3_ready }`.

### Model C — Kill Feed Reader
- **Detects:** Each line of the kill feed (top-right corner). Classifies the eliminating hero, eliminated hero, and team affiliation.
- **Approach:** Crop the top-right region. Run an object detector to find individual kill feed rows, then classify each row's content.
- **Output:** List of kill events per frame: `[{ killer_team, killer_hero, ability_used, victim_team, victim_hero, timestamp_ms }]`.

**Execution:**
- All three models run sequentially per frame on GPU via CUDA.
- Raw outputs per frame aggregated into a single `FrameDetection` dataclass and passed to the tracking stage.

---

## Stage 3 — Tracking & State (`pipeline/track.py`)

**Goal:** Convert per-frame detections into a continuous, time-ordered game-state dataset.

- **ByteTrack** (integrated into Ultralytics) assigns persistent `track_id` values to detected objects across frames, solving the identity problem (same enemy across 100 frames = same ID).
- All per-frame detections are merged into a Pandas DataFrame. Each row represents one time-step (one sampled frame), with columns for every tracked entity and UI state value.
- **Derived metrics computed in this stage:**
  - `crosshair_to_nearest_enemy_px`: Euclidean pixel distance from screen center to the nearest enemy bounding box center.
  - `team_alive_count` / `enemy_alive_count`: Derived from kill feed event log. Starts at 5v5, decremented on each kill event.
  - `time_since_last_kill_ms`: Rolling time delta since the last kill feed event.
  - `player_in_los`: Boolean estimated from whether any enemy bounding box is visible on screen.
- Final output: a single Pandas DataFrame (serialized to the `data/processed/` directory as a `.csv` and loaded into a SQLite database for querying).

**DataFrame schema (key columns):**

| Column | Type | Description |
|---|---|---|
| `timestamp_ms` | int | Timestamp of the sampled frame |
| `player_health_pct` | float | Player health as 0.0–1.0 |
| `player_ult_pct` | float | Ultimate charge as 0.0–1.0 |
| `ability_1_ready` | bool | Ability 1 off cooldown |
| `ability_2_ready` | bool | Ability 2 off cooldown |
| `crosshair_to_enemy_px` | float | Pixel distance from screen center to nearest enemy |
| `team_alive` | int | Number of allies alive (0–5) |
| `enemy_alive` | int | Number of enemies alive (0–5) |
| `ult_activated` | bool | Ultimate was activated this frame |
| `kill_event` | str | Kill feed entry parsed this frame (nullable) |

---

## Stage 4 — Logic Engine (`pipeline/analyze.py`)

**Goal:** Query the structured dataset with game-logic rules and produce a list of timestamped mistake events.

Each rule is a Python function that takes the full DataFrame as input and returns a list of `MistakeEvent` objects:

```python
@dataclass
class MistakeEvent:
    timestamp_ms: int
    category: str        # e.g., "Ultimate Economy"
    severity: str        # "low" | "medium" | "high"
    description: str     # Human-readable feedback string
```

### Initial Rule Set (V1)

**Rule 1 — Wasted Ultimate**
- Trigger: `ult_activated == True` AND `team_alive <= 2` (player's team is down 3+)
- Logic: When your team has already lost 3 players, the fight is effectively over. Using ultimate at this moment wastes a resource that could shift the next fight.
- Feedback: `"Ultimate used while your team was down 3+ players. Consider saving it for the next fight."`

**Rule 2 — Critical Health / Not Seeking Healing**
- Trigger: `player_health_pct < 0.20` sustained for > 3 seconds AND `crosshair_to_enemy_px < 300` (player is still fighting, not retreating)
- Feedback: `"You remained engaged at critical health (<20%) for over 3 seconds. Disengage and seek healing."`

**Rule 3 — Missed Engagement (Aim Consistency)**
- Trigger: Over any 5-second window where an enemy is on screen, `crosshair_to_enemy_px` mean > 150px
- Feedback: `"Crosshair tracking was consistently off during this engagement. Review aim technique."`

**Rule 4 — Ultimate Into Lost Fight (Economy)**
- Trigger: `ult_activated == True` AND in the preceding 5 seconds `kill_event` contains 3+ allied deaths
- Feedback: `"Ultimate activated immediately after losing 3+ teammates. The fight was already lost."`

**Rule 5 — Tunnel Vision (Ignoring Team Fight)**
- Trigger: `player_health_pct < 0.25` AND `time_since_last_kill_ms > 8000` (no kills being traded) AND `ult_activated == False`
- Feedback: `"Extended low-health engagement with no kills traded. Reassess fight or reposition."`

Rules are modular — each lives in its own function, making it easy to add, remove, or tune thresholds without touching unrelated logic.

---

## Stage 5 — Report (`pipeline/report.py`)

**Goal:** Transform the list of `MistakeEvent` objects into a human-readable coaching report.

- Group events by category (e.g., all "Ultimate Economy" mistakes together).
- For each event, output the timestamp formatted as `MM:SS` so the player can seek to it in their VOD.
- Produce a summary section: total mistake count per category, most frequent mistake type.
- Output formats:
  - **JSON** — machine-readable, for potential future web UI integration.
  - **Markdown** — clean, human-readable report the player can read immediately.

**Example output:**

```
## VOD Review — [source_file.mp4]

### Ultimate Economy (2 events)
- [04:32] Ultimate used while your team was down 3+ players.
- [11:07] Ultimate activated immediately after losing 3+ teammates.

### Survivability (1 event)
- [07:15] Remained engaged at critical health (<20%) for over 3 seconds.

### Summary
Most frequent mistake: Ultimate Economy
Total flagged events: 3
```

---

## Model Training Plan

### Data Collection
- Record or source Overwatch gameplay footage across multiple heroes and maps.
- Use `training/collect_frames.py` to extract frames at 5 FPS from raw recordings.
- Target **500–1000 annotated images per class** for the hero detector as a baseline.

### Annotation
- Use **Roboflow** or **Label Studio** for bounding box annotation.
- Export annotations in **YOLO format** (`.txt` files alongside images).
- For the UI reader, crop images to the UI region before annotation to reduce label noise.

### Training
- Fine-tune **YOLOv11n** (nano) or **YOLOv11s** (small) as the base model — these are fast enough for real-time inference and require smaller datasets to fine-tune.
- Train on a GPU using the Ultralytics CLI: `yolo detect train data=hero.yaml model=yolo11s.pt epochs=100 imgsz=1280`
- Use `imgsz=1280` to preserve the fine UI detail that matters for reading health bars and kill feed text.
- Validate on a held-out set (80/10/10 train/val/test split).

### Iteration Order
1. **Kill feed reader first** — provides the richest signal (team alive counts, fight outcomes) and is a bounded, consistent UI region that is easier to annotate.
2. **UI reader second** — health and ultimate charge are static regions; straightforward to crop and classify.
3. **Hero detector last** — most complex due to hero variety, visual occlusion, and 3D perspectives. Warrants the largest dataset.

---

## Development Milestones

| Milestone | Deliverable |
|---|---|
| M1 | Project scaffolding, `config.yaml`, `ingest.py` with frame sampling working end-to-end |
| M2 | Kill feed annotated dataset + trained Model C (`killfeed_reader`) |
| M3 | UI reader annotated dataset + trained Model B (`ui_reader`) |
| M4 | `perceive.py` and `track.py` complete; full DataFrame output from a test VOD |
| M5 | `analyze.py` with V1 rule set; first end-to-end mistake detection run |
| M6 | `report.py`; first complete human-readable VOD review output |
| M7 | Hero detector dataset + trained Model A; integrated into pipeline |
| M8 | Tuning, threshold calibration, and validation against manually reviewed VODs |
