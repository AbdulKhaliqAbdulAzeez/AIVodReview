# Sprint 01 — Project Scaffolding & Environment Setup

**Status:** Active
**Goal:** Stand up a fully working project skeleton. By the end of this sprint every file and folder in the planned directory structure exists, the environment installs cleanly, and `main.py` can be invoked without errors (even if it doesn't do anything yet).

---

## Sprint Goal

> "A developer can clone this repo, run one command to install dependencies, and run `python main.py --input sample.mp4` without any import errors or missing file errors."

---

## Tasks

### 1. Create the full directory structure

Create all directories defined in `planning.md`. Add `.gitkeep` files to empty directories so they are tracked by Git.

```
data/raw_videos/
data/frames/
data/annotations/
data/processed/
models/hero_detector/
models/ui_reader/
models/killfeed_reader/
pipeline/
training/
tests/
sprints/active/
sprints/upcoming/
sprints/completed/
```

**Acceptance criteria:** `find . -type d` shows all directories above.

---

### 2. Create `requirements.txt`

Pin versions to ensure reproducibility. The MVP dependencies are:

```
ultralytics>=8.3.0
opencv-python-headless>=4.10.0
pandas>=2.2.0
pytesseract>=0.3.13
Pillow>=10.4.0
pyyaml>=6.0.2
```

**Notes:**
- Use `opencv-python-headless` (not `opencv-python`) — headless avoids pulling in GUI libraries that fail in server/CI environments.
- `Pillow` is required by pytesseract for image handling.
- `ultralytics` is included now even though the MVP doesn't train models — it will be used for ByteTrack integration in future sprints and provides useful utilities.

**Acceptance criteria:** `pip install -r requirements.txt` completes without error in a clean virtual environment.

---

### 3. Verify system dependency — Tesseract

The pipeline's kill feed reader depends on the Tesseract OCR binary being installed at the system level (not via pip).

Install command (Ubuntu/Debian):
```bash
sudo apt install tesseract-ocr
```

Verify:
```bash
tesseract --version
```

Document the required version in a `SETUP.md` note (min: Tesseract 5.x).

**Acceptance criteria:** `pytesseract.get_tesseract_version()` returns a version without raising an error.

---

### 4. Create `config.yaml`

This file is the single source of truth for all tunable parameters. Nothing should be hardcoded in the pipeline scripts.

```yaml
# --- Video Ingestion ---
sample_fps: 10               # Frames per second to sample from the input video
input_resolution:
  width: 1920
  height: 1080

# --- Paths ---
paths:
  raw_videos: data/raw_videos/
  frames: data/frames/
  processed: data/processed/
  models:
    hero_detector: models/hero_detector/weights/best.pt
    ui_reader: models/ui_reader/weights/best.pt
    killfeed_reader: models/killfeed_reader/weights/best.pt

# --- Perception: UI Crop Regions (pixel coords for 1920x1080) ---
# Format: [x_start, y_start, x_end, y_end]
ui_crops:
  health_bar:    [10, 960, 310, 1080]
  ult_charge:    [310, 960, 510, 1080]
  killfeed:      [1500, 0, 1920, 400]

# --- Perception: Color Thresholds (HSV) ---
# Used for classical CV health/ult reading before YOLO models are trained
color_thresholds:
  health_green:
    lower: [40, 100, 100]
    upper: [80, 255, 255]
  health_yellow:
    lower: [20, 100, 100]
    upper: [40, 255, 255]
  health_red:
    lower: [0, 150, 100]
    upper: [10, 255, 255]
  ult_blue:
    lower: [100, 150, 100]
    upper: [130, 255, 255]

# --- Logic Engine Thresholds ---
rules:
  critical_health_pct: 0.20         # Below this = critical
  critical_health_duration_ms: 3000 # Must be critical for this long to flag
  wasted_ult_team_alive_max: 2      # Flag ult use if team has <= this many alive
  aim_consistency_window_ms: 5000   # Window for aim analysis
  aim_consistency_px_threshold: 150 # Mean distance above this = flag
  tunnel_vision_stall_ms: 8000      # No kills for this long while critical = flag

# --- Output ---
output:
  report_format: markdown            # "markdown" | "json" | "both"
  report_dir: data/processed/
```

**Acceptance criteria:** `import yaml; yaml.safe_load(open('config.yaml'))` succeeds and all expected keys are present.

---

### 5. Create pipeline module stubs

Create each pipeline file as a documented stub with the correct function signatures but `pass` or `raise NotImplementedError` bodies. This lets the entire pipeline be imported and wired together in `main.py` without any stage needing to be fully implemented.

**`pipeline/__init__.py`** — empty

**`pipeline/ingest.py`** stub:
```python
from typing import Generator, Tuple
import numpy as np

def sample_video(video_path: str, sample_fps: int) -> Generator[Tuple[int, int, np.ndarray], None, None]:
    """
    Yields (frame_index, timestamp_ms, frame_array) for each sampled frame.
    Raises FileNotFoundError if video_path does not exist.
    """
    raise NotImplementedError
```

**`pipeline/perceive.py`** stub:
```python
from dataclasses import dataclass, field
from typing import List, Optional
import numpy as np

@dataclass
class KillEvent:
    killer_team: str
    killer_hero: str
    victim_team: str
    victim_hero: str

@dataclass
class FrameDetection:
    timestamp_ms: int
    health_pct: Optional[float]
    ult_pct: Optional[float]
    ult_activated: bool
    kill_events: List[KillEvent] = field(default_factory=list)

def perceive_frame(frame: np.ndarray, timestamp_ms: int, config: dict) -> FrameDetection:
    """
    Runs all perception logic against a single frame.
    Returns a FrameDetection summary.
    """
    raise NotImplementedError
```

**`pipeline/track.py`** stub:
```python
import pandas as pd
from typing import List
from pipeline.perceive import FrameDetection

def build_state_dataframe(detections: List[FrameDetection]) -> pd.DataFrame:
    """
    Converts a list of FrameDetection objects into a time-series DataFrame
    with derived metrics (team_alive, crosshair_to_enemy_px, etc).
    """
    raise NotImplementedError
```

**`pipeline/analyze.py`** stub:
```python
from dataclasses import dataclass
from typing import List
import pandas as pd

@dataclass
class MistakeEvent:
    timestamp_ms: int
    category: str
    severity: str   # "low" | "medium" | "high"
    description: str

def run_analysis(df: pd.DataFrame, config: dict) -> List[MistakeEvent]:
    """
    Applies all logic rules to the state DataFrame.
    Returns a list of MistakeEvent objects.
    """
    raise NotImplementedError
```

**`pipeline/report.py`** stub:
```python
from typing import List
from pipeline.analyze import MistakeEvent

def generate_report(events: List[MistakeEvent], source_file: str, config: dict) -> str:
    """
    Formats a list of MistakeEvents into a human-readable report string.
    Writes output to the configured report_dir.
    Returns the path of the generated report file.
    """
    raise NotImplementedError
```

**Acceptance criteria:** `from pipeline import ingest, perceive, track, analyze, report` runs without import errors.

---

### 6. Create `main.py`

Wire the pipeline together end-to-end. At this stage it will fail gracefully at the first `NotImplementedError`, but the wiring and CLI argument parsing must be complete.

```python
import argparse
import yaml
from pipeline import ingest, perceive, track, analyze, report

def load_config(path: str = "config.yaml") -> dict:
    with open(path, "r") as f:
        return yaml.safe_load(f)

def main():
    parser = argparse.ArgumentParser(description="AI VOD Review Tool")
    parser.add_argument("--input", required=True, help="Path to input MP4 file")
    parser.add_argument("--config", default="config.yaml", help="Path to config file")
    args = parser.parse_args()

    config = load_config(args.config)
    sample_fps = config["sample_fps"]

    print(f"[1/5] Ingesting: {args.input} @ {sample_fps} FPS")
    frames = ingest.sample_video(args.input, sample_fps)

    print("[2/5] Running perception...")
    detections = [perceive.perceive_frame(frame, ts, config) for _, ts, frame in frames]

    print("[3/5] Building state...")
    df = track.build_state_dataframe(detections)

    print("[4/5] Analyzing for mistakes...")
    events = analyze.run_analysis(df, config)

    print("[5/5] Generating report...")
    output_path = report.generate_report(events, args.input, config)
    print(f"Report written to: {output_path}")

if __name__ == "__main__":
    main()
```

**Acceptance criteria:** `python main.py --help` prints the usage message without errors.

---

### 7. Create `tests/` structure

```
tests/
├── __init__.py
├── test_ingest.py       # stub
├── test_perceive.py     # stub
├── test_track.py        # stub
├── test_analyze.py      # stub
└── conftest.py          # shared fixtures (e.g., sample frame array, sample config)
```

Each test file should have at least one placeholder test that passes:
```python
def test_placeholder():
    assert True
```

**Acceptance criteria:** `python -m pytest tests/` passes with 0 failures.

---

## Definition of Done

- [ ] All directories exist with `.gitkeep` where needed
- [ ] `pip install -r requirements.txt` succeeds in a clean venv
- [ ] `tesseract --version` returns 5.x
- [ ] `config.yaml` loads without error and all keys are present
- [ ] `python main.py --help` prints usage without import errors
- [ ] `python -m pytest tests/` passes with 0 failures
- [ ] All pipeline stub files importable without error
