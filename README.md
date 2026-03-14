# AI VOD Review Tool

An automated Overwatch gameplay analysis pipeline that ingests raw VOD footage, extracts structured game-state data using computer vision, and produces timestamped coaching feedback — telling a player exactly what mistakes they made and when.

> **Project Status: Pre-Development — Sprint 01 Planning**
> No code has been written yet. The project is fully planned and ready to begin implementation.

---

## What It Does

Drop in an Overwatch MP4. Get back a report like this:

```
## VOD Review — gameplay.mp4

### 🔴 Ultimate Economy (2 events)
- [04:32] Ultimate used while your team had only 1 player(s) alive. Save it for the next fight.
- [11:07] Ultimate activated immediately after losing 3+ teammates. The fight was already lost.

### 🟡 Survivability (1 event)
- [07:15] Remained engaged at critical health (<20%) for over 3 seconds. Disengage and seek healing.

### Summary
Most common mistake: Ultimate Economy | Total flagged events: 3
```

---

## How It Works

The system is a four-stage data pipeline:

```
MP4 Video
    │
    ▼
[Stage 1 — Ingestion]
  OpenCV samples frames at 10 FPS
    │
    ▼
[Stage 2 — Perception]
  • Health/ult bar → OpenCV HSV color thresholding
  • Kill feed      → Tesseract OCR + hero name matching
    │
    ▼
[Stage 3 — Tracking & State]
  Per-frame detections → time-series Pandas DataFrame
  Derived metrics: team_alive, time_since_last_kill, crosshair distance
    │
    ▼
[Stage 4 — Logic Engine]
  Rule-based analysis flags mistakes (Wasted Ult, Tunnel Vision, etc.)
    │
    ▼
[Stage 5 — Report]
  Timestamped Markdown/JSON coaching report
```

The MVP uses classical computer vision (no trained models required) so the full pipeline can be validated end-to-end before the heavy investment of YOLO model training.

---

## Current Project State

| Area | Status |
|---|---|
| Architecture & pipeline design | ✅ Complete — see [planning.md](planning.md) |
| Sprint plan (6 sprints to MVP) | ✅ Complete — see [sprints/](sprints/) |
| Engineering & dev standards | ✅ Complete — see [DEVELOPMENT.md](DEVELOPMENT.md) |
| Code scaffolding | 🔲 Not started — Sprint 01 |
| Ingestion layer | 🔲 Not started — Sprint 02 |
| Perception layer (classical CV) | 🔲 Not started — Sprint 03 |
| Tracking & state layer | 🔲 Not started — Sprint 04 |
| Logic engine (rule set) | 🔲 Not started — Sprint 05 |
| Report generation | 🔲 Not started — Sprint 06 |
| YOLO model training | 🔲 Post-MVP |

---

## Sprint Roadmap

| Sprint | Focus | Status |
|---|---|---|
| [01 — Scaffolding](sprints/active/sprint-01-scaffolding.md) | Directory structure, config, stubs, pytest | 🟡 Active |
| [02 — Ingestion](sprints/upcoming/sprint-02-ingestion.md) | OpenCV frame sampler from MP4 | ⬜ Upcoming |
| [03 — Perception](sprints/upcoming/sprint-03-perception.md) | HSV health/ult reading + Tesseract kill feed | ⬜ Upcoming |
| [04 — Tracking & State](sprints/upcoming/sprint-04-tracking-state.md) | Time-series DataFrame, kill dedup, derived metrics | ⬜ Upcoming |
| [05 — Logic Engine](sprints/upcoming/sprint-05-logic-engine.md) | V1 rule set, mistake detection | ⬜ Upcoming |
| [06 — Report + MVP](sprints/upcoming/sprint-06-report-mvp.md) | Markdown/JSON output, end-to-end validation | ⬜ Upcoming |

---

## Planned Tech Stack

| Purpose | Technology |
|---|---|
| Language | Python 3.11+ |
| Computer Vision | Ultralytics YOLOv11 + OpenCV |
| GPU Acceleration | CUDA (NVIDIA) |
| Object Tracking | ByteTrack (via Ultralytics) |
| Kill Feed OCR | Tesseract 5.x |
| Data Processing | Pandas |
| Storage | SQLite (dev) → PostgreSQL (production) |
| Testing | pytest + pytest-cov |
| Code Quality | Black, Ruff, Mypy, pre-commit |

---

## Planned Project Structure

```
AIVodReviewTool/
├── data/
│   ├── raw_videos/          # Input MP4 VOD files
│   ├── frames/              # Temp sampled frames
│   ├── annotations/         # YOLO training label files
│   └── processed/           # Output CSVs / SQLite DBs per VOD
├── models/
│   ├── hero_detector/       # (Post-MVP) Trained YOLO weights
│   ├── ui_reader/           # (Post-MVP) Trained YOLO weights
│   └── killfeed_reader/     # (Post-MVP) Trained YOLO weights
├── pipeline/
│   ├── ingest.py            # Video → sampled frames
│   ├── perceive.py          # Frames → YOLO/CV detections
│   ├── track.py             # Detections → DataFrame
│   ├── analyze.py           # DataFrame → MistakeEvents
│   └── report.py            # MistakeEvents → report
├── training/                # Model training & annotation scripts
├── tests/                   # Unit, integration, regression tests
├── sprints/                 # Sprint plans (active / upcoming / completed)
├── config.yaml              # All tunable parameters
├── main.py                  # Pipeline entry point
├── DEVELOPMENT.md           # Engineering standards & practices
└── planning.md              # Full architecture design document
```

---

## Development Standards

This project follows strict engineering practices defined in [DEVELOPMENT.md](DEVELOPMENT.md):

- **Git:** GitHub Flow with Conventional Commits. All work via PRs. `main` is always deployable.
- **TDD:** Red → Green → Refactor. Tests are written before implementation.
- **Coverage:** Minimum 80% overall; 95% on the logic engine (`analyze.py`).
- **Quality gates:** Black, Ruff, Mypy, and pip-audit all run in CI on every PR.
- **Auditing:** Sprint retrospective audit after every sprint; monthly dependency security audit.
- **Model versioning:** Trained weights are versioned under `models/*/versions/` with a production symlink.
- **Data versioning:** Large files (videos, annotations, weights) managed with DVC.

---

## Post-MVP Roadmap

Once the MVP (Sprint 06) is complete:

1. **Sprint 07** — Replace Tesseract kill feed with trained YOLOv11 classifier (Model C)
2. **Sprint 08** — Replace HSV UI reading with trained YOLOv11 UI model (Model B)
3. **Sprint 09** — Hero detection model (Model A) + hero-specific rule sets
4. **Sprint 10** — Web UI with seekable VOD timestamps
5. **Sprint 11** — Multi-VOD batch processing + player improvement tracking over time
