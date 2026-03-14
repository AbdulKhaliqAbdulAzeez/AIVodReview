# Sprint 02 — Ingestion Layer

**Status:** Upcoming
**Depends on:** Sprint 01 complete
**Goal:** Implement `pipeline/ingest.py` fully. A developer can point the tool at any 1080p Overwatch MP4 and get back a memory-efficient stream of timestamped frames at the configured sample rate.

---

## Sprint Goal

> "Running `python main.py --input myvod.mp4` progresses past Stage 1 and prints the total number of sampled frames before failing at Stage 2 (NotImplementedError)."

---

## Background

The ingestion layer is the entry point for all data. Its design decisions — sample rate, memory model, frame format — cascade through every downstream stage. Getting it right here saves significant pain later.

Key constraint: VODs can be 30–60 minutes long at 60 FPS. That's 108,000–216,000 raw frames. We must not load these into memory. The generator pattern chosen in Sprint 01's stub is the correct approach.

---

## Tasks

### 1. Implement `sample_video()` in `pipeline/ingest.py`

Full implementation using OpenCV:

**Logic:**
1. Open the video with `cv2.VideoCapture(video_path)`.
2. Read the native FPS from the capture object.
3. Compute the frame interval: `frame_interval = round(native_fps / sample_fps)`. For a 60 FPS video sampled at 10 FPS, this is every 6th frame.
4. Loop: read one frame at a time. If `frame_index % frame_interval == 0`, yield the frame.
5. Compute `timestamp_ms = (frame_index / native_fps) * 1000`.
6. Release the capture object when the video ends.

**Error handling:**
- Raise `FileNotFoundError` if the file doesn't exist before opening.
- Raise `ValueError` if `cv2.VideoCapture.isOpened()` returns False (corrupt or unsupported format).
- Raise `ValueError` if the native FPS read from the file is 0 (can happen with certain container formats).

**Frame format:**
- Yield frames as `np.ndarray` in BGR format (OpenCV default).
- Do NOT convert to RGB here — conversion happens in perceive.py where needed, avoiding an unnecessary copy per frame.

**Acceptance criteria:**
- Works on a real MP4 file.
- Memory usage stays flat for a 60-minute video (verify with `memory_profiler` or Activity Monitor — no growing list of frames in RAM).
- `frame_index` and `timestamp_ms` values are accurate (spot-check: frame 600 at 60 FPS should be at `timestamp_ms = 10000`).

---

### 2. Add metadata logging

After the video opens, log the following to stdout before yielding any frames:

```
[Ingest] Source: myvod.mp4
[Ingest] Native FPS: 60.0
[Ingest] Resolution: 1920x1080
[Ingest] Duration: 32m 14s
[Ingest] Sample rate: 10 FPS (every 6th frame)
[Ingest] Estimated frames to process: ~19,340
```

This gives the user an immediate sanity check that the video was read correctly.

**Acceptance criteria:** Metadata block prints before first frame is yielded.

---

### 3. Add a `--dry-run` flag to `main.py`

For debugging ingestion without running the full pipeline, add a `--dry-run` CLI flag. When set, `main.py` only runs ingestion, prints the metadata block, counts frames, and exits.

```bash
python main.py --input myvod.mp4 --dry-run
```

Output:
```
[Ingest] Source: myvod.mp4
[Ingest] Native FPS: 60.0
...
[Dry run] Total frames sampled: 19,340
[Dry run] Done. Exiting before perception stage.
```

**Acceptance criteria:** `--dry-run` exits cleanly after counting all frames without calling `perceive.perceive_frame()`.

---

### 4. Write unit tests for `ingest.py`

**`tests/test_ingest.py`** should cover:

- **Happy path:** Create a minimal synthetic MP4 with OpenCV's `VideoWriter` (e.g., 60 frames of a solid-color image at 30 FPS). Assert that sampling at 10 FPS yields exactly 2 frames (`frame 0` and `frame 18`), and that `timestamp_ms` values are approximately `0` and `600`.
- **FileNotFoundError:** Assert that `sample_video("nonexistent.mp4", 10)` raises `FileNotFoundError`.
- **ValueError on corrupt file:** Assert that passing a text file renamed as `.mp4` raises `ValueError`.
- **Timestamp accuracy:** For the synthetic video, assert `timestamp_ms` is within ±50ms of expected value for each sampled frame.

Use `conftest.py` to house the synthetic video fixture so other test files can reuse it.

**Acceptance criteria:** `python -m pytest tests/test_ingest.py -v` passes all 4 tests.

---

### 5. Add frame-count progress reporting

For long VODs, ingestion will run for several minutes. Add a lightweight progress indicator:

- Every 500 sampled frames, print a progress line:
  ```
  [Ingest] Progress: 500 / ~19,340 frames (2.6%)
  ```
- Use the estimated total frame count computed from duration and sample FPS.

**Acceptance criteria:** Progress lines appear roughly every 50 seconds of video processed.

---

## Definition of Done

- [ ] `sample_video()` implemented and handles all error cases
- [ ] Metadata block logs correctly from a real MP4
- [ ] `--dry-run` flag works and exits cleanly
- [ ] All 4 unit tests pass in `test_ingest.py`
- [ ] Progress reporting works on a VOD longer than 5 minutes
- [ ] Memory usage remains flat during ingestion of a 30-minute VOD
