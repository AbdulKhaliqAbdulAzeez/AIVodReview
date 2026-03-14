# Sprint 03 — Perception Layer (Classical CV)

**Status:** Upcoming
**Depends on:** Sprint 02 complete
**Goal:** Implement `pipeline/perceive.py` using OpenCV color thresholding (health/ult) and Tesseract OCR (kill feed) — no YOLO model training required. By end of sprint the system can extract meaningful data from raw frames.

---

## Sprint Goal

> "Given a single Overwatch frame, `perceive_frame()` returns a `FrameDetection` object with a plausible `health_pct`, `ult_pct`, and any kill events visible in the frame."

---

## Background

This sprint replaces trained YOLO models with classical computer vision techniques for the MVP. This is intentional: it allows us to validate the full pipeline end-to-end without spending weeks on dataset collection and training. The trade-off is lower accuracy — thresholds and crop coordinates will need tuning against real footage.

All crop coordinates in `config.yaml` are defined for **1920×1080**. If a different resolution is used, they must be recalibrated.

---

## Tasks

### 1. Implement health bar reading via HSV color thresholding

**Location:** `pipeline/perceive.py` → `_read_health(frame, config)`

**Logic:**
1. Crop the frame to the `health_bar` region from config: `frame[y_start:y_end, x_start:x_end]`.
2. Convert the crop to HSV colorspace.
3. Apply three masks using the green, yellow, and red HSV thresholds from config.
4. Count the number of non-zero pixels in each mask.
5. Determine the filled portion of the bar by comparing the green/yellow/red pixel count to the total bar width × height.
6. Map to a `health_pct` float between 0.0 and 1.0.
7. Determine state: `>0.50` = healthy, `0.20–0.50` = low, `< 0.20` = critical.

**Known challenges:**
- Overwatch's health bar includes a shield/armor layer that appears yellow regardless of health percentage. Shield HP will inflate the yellow pixel count. Document this as a known limitation in a code comment — a later YOLO-based model will fix it properly.
- Particle effects and on-screen explosions can bleed into the health bar crop region momentarily. Average the reading across 3 consecutive frames (handled in `track.py`) to smooth this.

**Acceptance criteria:**
- On a manually selected frame where health is clearly full, returns `health_pct >= 0.95`.
- On a clearly near-death frame, returns `health_pct <= 0.15`.
- Returns `None` if the crop region is entirely black (e.g., respawn screen or end-of-match).

---

### 2. Implement ultimate charge reading via HSV thresholding

**Location:** `pipeline/perceive.py` → `_read_ult(frame, config)`

**Logic:**
1. Crop to the `ult_charge` region from config.
2. Convert to HSV. Apply the `ult_blue` mask to isolate the blue ult charge bar.
3. Measure the width of the filled blue region as a fraction of the total crop width.
4. Return as `ult_pct` (0.0 = empty, 1.0 = full).
5. Detect ult activation: if `ult_pct` drops sharply from `>= 0.90` to `<= 0.10` between two consecutive frames, set `ult_activated = True` on the frame where the drop is first detected.

**Note on ult activation detection:**
- A single frame cannot detect activation — it requires state from the previous frame. `perceive_frame()` should accept an optional `previous_detection: Optional[FrameDetection]` argument. If `previous_detection` is provided and `ult_activated` logic applies, set the flag. This is simpler than doing it in `track.py`.

**Acceptance criteria:**
- Empty ult bar returns `ult_pct` close to 0.0.
- Full ult bar returns `ult_pct` close to 1.0.
- A frame where the ult was just used (bar goes from full to empty) sets `ult_activated = True`.

---

### 3. Implement kill feed reading via Tesseract OCR

**Location:** `pipeline/perceive.py` → `_read_killfeed(frame, config)`

**Logic:**
1. Crop the frame to the `killfeed` region from config.
2. Pre-process the crop for OCR:
   - Convert to grayscale.
   - Apply a slight Gaussian blur to denoise.
   - Threshold to binarise (Otsu's method works well for the kill feed's high-contrast text).
3. Pass the processed crop to `pytesseract.image_to_string()` with `config='--psm 6'` (assume uniform block of text).
4. Split the raw OCR output into lines.
5. For each line, run it through `_parse_killfeed_line()`:
   - Match against a dictionary of all Overwatch hero names (`HERO_NAMES` constant defined in the file).
   - A valid kill event line must contain exactly two hero names (killer and victim).
   - Determine team affiliation from line color (extract the dominant color of the icon region to the left of each line — red = enemy, blue = ally).
   - Return a `KillEvent` or `None` if the line doesn't match the pattern.

**`HERO_NAMES` constant (top of file):**
```python
HERO_NAMES = {
    "ana", "ashe", "baptiste", "bastion", "brigitte", "cassidy",
    "dva", "doomfist", "echo", "genji", "hanzo", "illari",
    "junker queen", "junkrat", "kiriko", "lifeweaver", "lucio",
    "mauga", "mccree", "mei", "mercy", "moira", "orisa",
    "pharah", "ramattra", "reaper", "reinhardt", "roadhog",
    "sigma", "sojourn", "soldier 76", "sombra", "symmetra",
    "torbjorn", "tracer", "venture", "widowmaker", "winston",
    "wrecking ball", "zarya", "zenyatta"
}
```

**Known challenges:**
- Tesseract will frequently misread hero icons or ability icons in the kill feed as random characters. Filtering by `HERO_NAMES` will discard most garbage lines.
- Hero names with numbers (e.g., "D.Va", "Soldier: 76") will be OCR'd inconsistently. Add normalized variants to the lookup set.
- Kill feed entries scroll up and disappear. The same kill event may appear across multiple consecutive frames. De-duplication is handled in `track.py`, not here.

**Acceptance criteria:**
- On a frame with a clear single kill feed entry (`Tracer eliminated Genji`), returns a `KillEvent` with correct hero names.
- On a frame with no kill feed entries, returns an empty list.
- Does not crash on unrecognized OCR output — all failures are caught and return `None` for that line.

---

### 4. Wire everything into `perceive_frame()`

```python
def perceive_frame(
    frame: np.ndarray,
    timestamp_ms: int,
    config: dict,
    previous_detection: Optional[FrameDetection] = None
) -> FrameDetection:
    health_pct = _read_health(frame, config)
    ult_pct, ult_activated = _read_ult(frame, config, previous_detection)
    kill_events = _read_killfeed(frame, config)

    return FrameDetection(
        timestamp_ms=timestamp_ms,
        health_pct=health_pct,
        ult_pct=ult_pct,
        ult_activated=ult_activated,
        kill_events=kill_events,
    )
```

Update `main.py` to pass `previous_detection` from the last iteration:

```python
previous = None
detections = []
for _, ts, frame in frames:
    det = perceive.perceive_frame(frame, ts, config, previous_detection=previous)
    detections.append(det)
    previous = det
```

**Acceptance criteria:** `perceive_frame()` returns a fully populated `FrameDetection` without errors on a real Overwatch frame.

---

### 5. Create a calibration script

**`training/calibrate_crops.py`** — a developer utility, not part of the main pipeline.

This script takes a single frame (or a path to a video) and displays the cropped UI regions overlaid on the frame using `cv2.imshow()`. This makes it fast to visually verify that the crop coordinates in `config.yaml` are correct for a given recording setup.

```bash
python training/calibrate_crops.py --frame path/to/screenshot.png
```

**Output:** OpenCV window showing the frame with crop rectangles drawn in different colors (green = health, blue = ult, yellow = killfeed). Press any key to close.

**Acceptance criteria:** Running the script on a real 1920×1080 Overwatch screenshot shows all three crop zones in the correct positions.

---

### 6. Write unit tests for `perceive.py`

**`tests/test_perceive.py`:**

Use a set of saved Overwatch screenshots as test fixtures (committed to `tests/fixtures/` directory):
- `full_health.png` — player at 100% HP
- `critical_health.png` — player at <20% HP
- `full_ult.png` — ult bar at 100%
- `empty_ult.png` — ult bar at 0%
- `killfeed_single.png` — one clearly visible kill event
- `no_killfeed.png` — no kill events visible
- `respawn_screen.png` — death screen (all UI hidden)

Tests:
- `test_health_full()` — asserts `health_pct >= 0.90`
- `test_health_critical()` — asserts `health_pct <= 0.20`
- `test_ult_full()` — asserts `ult_pct >= 0.90`
- `test_ult_empty()` — asserts `ult_pct <= 0.10`
- `test_killfeed_parsed()` — asserts at least one `KillEvent` returned
- `test_killfeed_empty()` — asserts empty list returned
- `test_respawn_returns_none()` — asserts `health_pct is None`

**Acceptance criteria:** All 7 tests pass. Fixtures committed to repo.

---

## Calibration Notes (fill in during sprint)

Document the actual crop coordinates verified against your footage here:

| Region | Config Key | Expected Position | Verified? |
|---|---|---|---|
| Health bar | `ui_crops.health_bar` | Bottom-left, ~0–300px wide | |
| Ult charge | `ui_crops.ult_charge` | Bottom-center-left | |
| Kill feed | `ui_crops.killfeed` | Top-right corner | |

---

## Definition of Done

- [ ] `_read_health()` returns plausible values on real frames
- [ ] `_read_ult()` returns plausible values and detects activation correctly
- [ ] `_read_killfeed()` parses at least one valid kill event from real footage
- [ ] `perceive_frame()` wired correctly with `previous_detection` threading
- [ ] `calibrate_crops.py` script works and shows correct crop zones
- [ ] All 7 unit tests pass with committed fixture images
- [ ] All crop coordinates verified against your actual recording resolution
