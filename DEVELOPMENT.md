# Development Standards & Engineering Practices

This document defines the engineering standards for the AI VOD Review Tool. Every contributor (including future-you) should follow these practices from day one. Consistency here is what separates a project that can grow from one that collapses under its own technical debt.

---

## Table of Contents

1. [Philosophy](#1-philosophy)
2. [Git Workflow](#2-git-workflow)
3. [Test-Driven Development (Red / Green / Refactor)](#3-test-driven-development-red--green--refactor)
4. [Test Types & Coverage Requirements](#4-test-types--coverage-requirements)
5. [Code Quality Standards](#5-code-quality-standards)
6. [Model Development Lifecycle](#6-model-development-lifecycle)
7. [Experiment Tracking](#7-experiment-tracking)
8. [Codebase Auditing](#8-codebase-auditing)
9. [Dependency Management & Security](#9-dependency-management--security)
10. [Continuous Integration (CI)](#10-continuous-integration-ci)
11. [Data Versioning](#11-data-versioning)
12. [Documentation Standards](#12-documentation-standards)
13. [Release & Versioning](#13-release--versioning)

---

## 1. Philosophy

**Build a small thing that actually works before building a big thing that might.**

This project uses YOLO models, classical CV, OCR, and a custom rule engine. Each of those components is a potential failure point. The practices in this document exist to:

- Catch regressions the moment they are introduced, not three sprints later.
- Make it safe to refactor — you should be able to change the internals of any module without fear, because tests will catch anything that breaks.
- Keep the audit trail clean — every change, experiment, and decision should be traceable.
- Protect the model training process — training data and model weights are assets. Treat them like code.

**The guiding rule:** If you can't test it, you don't understand it well enough to ship it.

---

## 2. Git Workflow

### 2.1 Branching Strategy (GitHub Flow)

This project uses GitHub Flow — a lightweight, trunk-based model appropriate for a small team or solo developer. There are two permanent branch types:

```
main          ← Always deployable. Protected. Only merged into via PR.
feature/*     ← All work happens here. Short-lived. One branch per sprint task.
```

**Do not use GitFlow** (the `develop` + `release` branch model) for this project. It adds ceremony without benefit at this scale.

#### Branch naming convention:

| Type | Pattern | Example |
|---|---|---|
| Feature | `feature/<sprint>-<short-description>` | `feature/s01-project-scaffolding` |
| Bugfix | `fix/<short-description>` | `fix/health-pct-null-on-respawn` |
| Experiment | `experiment/<description>` | `experiment/hsv-threshold-tuning` |
| Chore | `chore/<description>` | `chore/update-dependencies` |

#### Rules:
- `main` is **always in a working state**. Never push broken code to `main`.
- Feature branches are **short-lived** — merge within 1–2 days of opening, or rebase daily to avoid divergence.
- Never commit directly to `main`. Always open a Pull Request (even when working solo — PRs create a review checkpoint and a searchable history of decisions).

---

### 2.2 Commit Message Convention

Follow [Conventional Commits](https://www.conventionalcommits.org/). Every commit message must be structured as:

```
<type>(<scope>): <short summary>

[optional body]

[optional footer]
```

**Types:**

| Type | When to use |
|---|---|
| `feat` | A new feature or capability |
| `fix` | A bug fix |
| `test` | Adding or updating tests |
| `refactor` | Code change that neither fixes a bug nor adds a feature |
| `data` | Adding/updating training data or annotations |
| `model` | Training runs, weight updates, hyperparameter changes |
| `docs` | Documentation changes |
| `chore` | Dependency updates, config changes, build tooling |
| `ci` | Changes to CI/CD pipeline |

**Scopes (for this project):**

`ingest`, `perceive`, `track`, `analyze`, `report`, `training`, `tests`, `config`, `pipeline`

**Examples:**

```
feat(ingest): implement sample_video generator with configurable FPS

fix(perceive): handle null return from pytesseract on black frames

test(analyze): add red test for rule_wasted_ultimate with team_alive=5

model(killfeed_reader): train v1 on 800 annotated frames, val mAP50=0.82

data(killfeed): add 200 annotated frames from Junkertown map

chore: bump ultralytics to 8.3.2, pin opencv-python-headless to 4.10.0.84
```

**Rules:**
- Summary line is **50 characters or fewer**, imperative mood ("add", not "added" or "adds").
- Body explains **why**, not what. The diff shows what changed.
- Never use vague messages: `fix bug`, `update`, `wip`, `asdf`.

---

### 2.3 Pull Request Process

Every feature branch merges via PR. Even for solo work, writing a PR description forces you to articulate what changed and why.

**PR description template** (save as `.github/pull_request_template.md`):

```markdown
## Summary
<!-- What does this PR do? One paragraph max. -->

## Changes
<!-- Bullet list of specific changes. -->
- 

## Testing
<!-- How was this tested? What tests were added or updated? -->
- [ ] New unit tests added
- [ ] Existing tests still pass (`pytest tests/`)
- [ ] Manually tested on a real VOD frame / segment

## Definition of Done Checklist
<!-- Copy from the relevant sprint task -->
- [ ] 
- [ ] 

## Notes
<!-- Anything reviewers should know. Known limitations, follow-up work, etc. -->
```

**PR rules:**
- All CI checks must pass before merging.
- Squash-merge feature branches into `main` to keep the `main` history clean and linear.
- Delete the feature branch after merging.

---

### 2.4 Tagging and Milestones

Tag `main` at the completion of each sprint milestone:

```bash
git tag -a v0.1.0 -m "Sprint 01 complete: project scaffolding"
git tag -a v0.2.0 -m "Sprint 02 complete: ingestion layer"
# ...
git tag -a v1.0.0 -m "Sprint 06 complete: MVP"
```

Use semantic versioning (`MAJOR.MINOR.PATCH`):
- `MAJOR` — breaking change to the pipeline interface or output format.
- `MINOR` — new sprint milestone, new feature, new rule added.
- `PATCH` — bug fix, threshold tuning, minor improvement.

Model weights follow their own versioning in `models/<name>/versions/` (see Section 6).

---

## 3. Test-Driven Development (Red / Green / Refactor)

TDD is the single most effective practice for this codebase. The rule engine and perception layer are both fundamentally **logic against data** — exactly where TDD provides the most value.

### 3.1 The TDD Cycle

**Red → Green → Refactor. Always in that order.**

```
┌─────────────────────────────────────────────────────────┐
│                                                         │
│   1. RED    Write a test that FAILS for the right       │
│             reason. The feature doesn't exist yet.      │
│             The test should be specific and minimal.    │
│                                                         │
│   2. GREEN  Write the minimum code needed to make       │
│             the test pass. No more. Don't design,       │
│             just make it green.                         │
│                                                         │
│   3. REFACTOR  Now clean up. Remove duplication,        │
│             improve naming, restructure if needed.      │
│             Tests must stay green throughout.           │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 3.2 What a Red Test Looks Like

A red test is written **before any implementation exists**. It fails, and it fails for the **right reason** (not an import error or syntax error — those are not red tests, they are broken tests).

**Example — Writing a red test for Rule 1 (Wasted Ultimate):**

Step 1: Write the test first, in `tests/test_analyze.py`:

```python
def test_rule_wasted_ultimate_fires_when_team_has_one_alive(sample_config):
    """RED: rule_wasted_ultimate should return one event when ult used with 1 teammate alive."""
    df = make_synthetic_df([
        {"timestamp_ms": 60000, "ult_activated": True, "team_alive": 1,
         "health_pct": 0.8, "ult_pct": 0.0, "time_since_last_kill_ms": 2000}
    ])
    events = rule_wasted_ultimate(df, sample_config)
    assert len(events) == 1
    assert events[0].category == "Ultimate Economy"
    assert events[0].severity == "high"
    assert events[0].timestamp_ms == 60000
```

Step 2: Run it. It fails with `ImportError` or `NotImplementedError`. That is the red state.

Step 3: Implement `rule_wasted_ultimate()` until the test passes. That is the green state.

Step 4: Refactor if needed. Run the test again to confirm still green.

### 3.3 Red Test Naming Convention

Red tests must be named with the prefix `test_` and describe **exactly what behavior they assert**. Never name a test `test_rule1()` — name it `test_rule_wasted_ultimate_fires_when_team_has_one_alive()`.

The test name is the specification. A failing test name should read like a bug report.

### 3.4 Green Test Rules

A green test is any test that currently passes. Key rules:

- **Never commit a failing test to `main`.** A failing test on `main` is a broken build.
- **Never skip a test to make CI pass** (`@pytest.mark.skip`). If a test needs to be skipped, open a tracking issue and document why.
- **A green test suite is a contract.** If you change implementation and a green test goes red, you either introduced a regression (fix the code) or the behavior intentionally changed (update the test and document why in the commit).

---

## 4. Test Types & Coverage Requirements

### 4.1 Unit Tests

**Location:** `tests/test_<module>.py`
**Scope:** One function, one behavior, no I/O, no real files.

Unit tests must be:
- **Fast** — no test should take more than 100ms. If it does, mock the slow dependency.
- **Isolated** — no test should depend on the output of another test. Use `conftest.py` fixtures.
- **Deterministic** — same input always produces same output. No randomness unless seeded.

Use `conftest.py` for shared fixtures (sample config, synthetic frames, synthetic DataFrames):

```python
# tests/conftest.py
import pytest
import yaml
import numpy as np
import pandas as pd

@pytest.fixture
def sample_config():
    with open("config.yaml") as f:
        return yaml.safe_load(f)

@pytest.fixture
def blank_frame_1080p():
    """A black 1920x1080 BGR frame."""
    return np.zeros((1080, 1920, 3), dtype=np.uint8)

def make_synthetic_df(rows: list) -> pd.DataFrame:
    """Helper to build a minimal state DataFrame for logic engine tests."""
    defaults = {
        "timestamp_ms": 0, "health_pct": 1.0, "ult_pct": 0.0,
        "ult_activated": False, "kill_event_str": None,
        "team_alive": 5, "enemy_alive": 5, "time_since_last_kill_ms": 99999
    }
    return pd.DataFrame([{**defaults, **row} for row in rows])
```

### 4.2 Integration Tests

**Location:** `tests/integration/`
**Scope:** Two or more pipeline stages together. Uses real files (small fixtures).

Integration tests verify that the output of Stage N is correctly consumed by Stage N+1. They are slower than unit tests and are run separately:

```bash
pytest tests/integration/ -v --tb=short
```

Example integration tests:
- `test_ingest_to_perceive`: Run `sample_video()` on a 10-second synthetic MP4, pass each frame to `perceive_frame()`, assert no exceptions and all `FrameDetection` objects are non-null.
- `test_perceive_to_track`: Generate 100 synthetic `FrameDetection` objects, pass to `build_state_dataframe()`, assert correct schema and row count.
- `test_track_to_analyze`: Load a known processed `.csv`, run `run_analysis()`, assert expected events for a segment with known mistakes.

### 4.3 Regression Tests

**Location:** `tests/regression/`
**Scope:** End-to-end on fixed test VOD segments. The "golden file" pattern.

When you manually verify that a segment produces correct output, save that output as a `.json` golden file. The regression test re-runs the pipeline on that segment and diffs the output against the golden file.

```
tests/regression/
├── fixtures/
│   ├── segment_wasted_ult/
│   │   ├── frames/          ← extracted frames (or tiny MP4)
│   │   └── expected.json    ← expected MistakeEvent list
│   └── segment_critical_health/
│       ├── frames/
│       └── expected.json
└── test_regression.py
```

Regression tests catch silent regressions — cases where a code change causes a previously correct detection to disappear or a new false positive to appear. **Run regression tests before every merge to `main`.**

### 4.4 Model Performance Tests

**Location:** `tests/model/`
**Scope:** YOLO model evaluation on held-out validation sets.

Each trained model has a minimum mAP50 (mean Average Precision at 50% IoU) threshold. If a new training run produces a model that falls below this threshold, the test fails:

```python
# tests/model/test_killfeed_model.py
def test_killfeed_model_meets_map50_threshold():
    """Model must achieve mAP50 >= 0.80 on the validation set."""
    results = run_yolo_validation("models/killfeed_reader/weights/best.pt",
                                  data="data/annotations/killfeed.yaml")
    assert results.box.map50 >= 0.80, (
        f"Kill feed model mAP50 {results.box.map50:.3f} is below the 0.80 threshold. "
        f"Do not deploy this model version."
    )
```

**Minimum thresholds (to be updated as models mature):**

| Model | Minimum mAP50 | Current Best |
|---|---|---|
| Kill feed reader | 0.80 | — |
| UI reader | 0.85 | — |
| Hero detector | 0.75 | — |

### 4.5 Coverage Requirements

Measure test coverage with `pytest-cov`:

```bash
pytest tests/ --cov=pipeline --cov-report=term-missing
```

**Minimum coverage requirements:**

| Module | Required Coverage |
|---|---|
| `pipeline/ingest.py` | ≥ 90% |
| `pipeline/perceive.py` | ≥ 80% |
| `pipeline/track.py` | ≥ 90% |
| `pipeline/analyze.py` | ≥ 95% |
| `pipeline/report.py` | ≥ 85% |

The logic engine (`analyze.py`) has the highest requirement because every rule branch must be explicitly tested — both the "fires" path and the "does not fire" path.

Add to `requirements.txt` (dev):
```
pytest>=8.0.0
pytest-cov>=5.0.0
```

---

## 5. Code Quality Standards

### 5.1 Formatting — Black

Use `black` (the uncompromising Python formatter). No configuration, no debates.

```bash
black pipeline/ training/ tests/
```

Black is run automatically in CI. Any PR that doesn't pass `black --check` is blocked.

### 5.2 Linting — Ruff

Use `ruff` for fast linting. It replaces flake8, isort, and several other tools:

```bash
ruff check pipeline/ training/ tests/
```

**`.ruff.toml` config:**
```toml
line-length = 100
select = ["E", "F", "W", "I", "N", "UP", "B", "C4"]
ignore = ["E501"]   # black handles line length
```

### 5.3 Type Checking — Mypy

All pipeline code must have type annotations. Run mypy in strict mode:

```bash
mypy pipeline/ --strict --ignore-missing-imports
```

Type annotations serve as inline documentation and catch entire classes of bugs (passing a `str` where a `Path` is expected, returning `None` where a `float` is required, etc.).

### 5.4 Pre-commit Hooks

Install pre-commit to run quality checks automatically before every commit. This prevents style issues from ever reaching the remote:

```bash
pip install pre-commit
pre-commit install
```

**`.pre-commit-config.yaml`:**
```yaml
repos:
  - repo: https://github.com/psf/black
    rev: 24.10.0
    hooks:
      - id: black

  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.8.0
    hooks:
      - id: ruff
        args: [--fix]

  - repo: https://github.com/pre-commit/mirrors-mypy
    rev: v1.13.0
    hooks:
      - id: mypy
        additional_dependencies: [pandas-stubs, types-PyYAML, types-Pillow]

  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v5.0.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-json
      - id: check-merge-conflict
      - id: debug-statements     # catches leftover breakpoint() calls
```

**Add to dev requirements:**
```
pre-commit>=4.0.0
black>=24.10.0
ruff>=0.8.0
mypy>=1.13.0
pandas-stubs
types-PyYAML
types-Pillow
```

---

## 6. Model Development Lifecycle

This section defines the process for going from raw video to a deployed, versioned YOLO model. Every model must go through all stages — skipping stages leads to models that work in training but fail on real footage.

### 6.1 Stage 1 — Data Collection

**Who:** Developer
**Tool:** `training/collect_frames.py`

- Extract frames from raw VODs at 5 FPS (higher than inference rate to increase annotation density).
- Aim for **diversity** across: maps, heroes, lighting conditions, HUD states, resolutions.
- Store raw extracted frames in `data/frames/<model_name>/raw/`.
- Target counts (per model):

| Model | Target Unique Images |
|---|---|
| Kill feed reader | 1,000+ |
| UI reader | 800+ |
| Hero detector | 2,000+ (across all hero classes) |

**Log every collection run:**
```
data/frames/killfeed_reader/raw/
├── collection_log.csv   ← source VOD, timestamp_ms, extracted frame filename
```

This log is critical. If you discover annotation errors later, you need to know which VOD and timestamp they came from.

### 6.2 Stage 2 — Annotation

**Who:** Developer
**Tool:** Roboflow (preferred) or Label Studio

**Rules:**
- Annotate in sessions of no more than 90 minutes — annotation quality degrades rapidly with fatigue.
- Every annotation session should be reviewed by at least one "second pass" before export.
- Export in **YOLO format** (`images/` and `labels/` directories with `.txt` files).
- Store exports in `data/annotations/<model_name>/v<N>/` where `N` is the annotation version.
- Document any ambiguous classes or edge cases in `data/annotations/<model_name>/annotation_guide.md`.

**Inter-annotator agreement:** If more than one person annotates, measure overlap. Disagreement rate > 15% on the same images indicates the class definitions are ambiguous — resolve before continuing.

### 6.3 Stage 3 — Dataset Versioning

Before training, create a `dataset.yaml` (YOLO format) and commit it to the repo:

```yaml
# data/annotations/killfeed_reader/v1/dataset.yaml
path: data/annotations/killfeed_reader/v1
train: images/train
val:   images/val
test:  images/test
nc: 6
names: ["ally_kill", "enemy_kill", "ally_death", "enemy_death", "self_kill", "assist"]
```

Split ratio: **70% train / 15% val / 15% test**.

The test set is held out entirely. It is **never** used during training or hyperparameter tuning. It is only used once — for the final evaluation before deployment.

### 6.4 Stage 4 — Training

**Script:** `training/train_<model_name>.py`

Each training script must:
1. Log the Git commit hash of the code at training time.
2. Log the dataset version used.
3. Save all training arguments to a `train_config.yaml` in the run directory.
4. Be reproducible — seed all random generators.

**Standard training command (via Ultralytics CLI):**
```bash
yolo detect train \
  data=data/annotations/killfeed_reader/v1/dataset.yaml \
  model=yolo11s.pt \
  epochs=100 \
  imgsz=1280 \
  batch=16 \
  lr0=0.01 \
  seed=42 \
  project=models/killfeed_reader \
  name=run_v1
```

**Never train on the test set. Never tune hyperparameters using test set metrics.**

### 6.5 Stage 5 — Evaluation

After training completes:

1. Run validation on the val set (automatic, done by Ultralytics during training).
2. Run evaluation on the **test set** once: `yolo detect val model=best.pt data=dataset.yaml split=test`
3. Record the following metrics in the run log:

| Metric | Description |
|---|---|
| mAP50 | Primary metric. Must meet threshold in Section 4.4. |
| mAP50-95 | Stricter metric. Track for trend over model versions. |
| Precision | Fraction of detections that were correct. |
| Recall | Fraction of real objects that were detected. |
| Inference speed (ms/frame) | Must stay below 50ms on your GPU for 10 FPS processing. |

4. Save the confusion matrix and example prediction images from the val set to the run directory.

### 6.6 Stage 6 — Model Versioning

Deployed model weights are versioned under:

```
models/killfeed_reader/
├── versions/
│   ├── v1/
│   │   ├── best.pt
│   │   ├── train_config.yaml
│   │   ├── eval_results.json
│   │   └── git_commit.txt       ← hash of code at training time
│   └── v2/
│       └── ...
└── weights/
    └── best.pt                  ← symlink to current production version
```

`models/<name>/weights/best.pt` is a symlink pointing to the currently deployed version. Switching versions is:

```bash
ln -sf models/killfeed_reader/versions/v2/best.pt models/killfeed_reader/weights/best.pt
```

**Never overwrite a previous version's weights.** Storage is cheap. Losing the ability to roll back is expensive.

### 6.7 Stage 7 — Model Integration Testing

Before updating the production symlink, run the model performance regression test (Section 4.4) and the end-to-end regression tests (Section 4.3). Both must pass.

```bash
pytest tests/model/ tests/regression/ -v
```

If either fails, do not update the symlink. Investigate and retrain.

---

## 7. Experiment Tracking

Every training run, threshold-tuning session, and architecture change is an experiment. Experiments that aren't tracked are experiments that cannot be learned from.

### 7.1 Experiment Log

Maintain a running `EXPERIMENTS.md` log in the repo root. Each entry follows this template:

```markdown
## EXP-007 — 2026-04-15
**Hypothesis:** Increasing imgsz from 1280 to 1600 will improve kill feed OCR accuracy on small text.
**Dataset:** killfeed_reader/v2 (1200 annotated images)
**Config changes:** imgsz=1600, batch=8 (reduced due to VRAM), epochs=100
**Result:** mAP50 improved from 0.82 → 0.87. Inference time increased from 18ms → 31ms/frame.
**Decision:** Deploy. The accuracy gain is worth the inference cost at our 10 FPS sample rate.
**Commit:** abc1234
**Weights:** models/killfeed_reader/versions/v3/best.pt
```

### 7.2 Ultralytics Run Directories

Ultralytics automatically saves training artifacts to the `project/name` directory specified in the training command:

```
models/killfeed_reader/run_v1/
├── weights/
│   ├── best.pt
│   └── last.pt
├── results.csv     ← per-epoch metrics
├── confusion_matrix.png
├── val_batch0_pred.jpg
└── args.yaml       ← full training config
```

Do not delete these directories. They are the permanent record of what each run produced.

### 7.3 Hyperparameter Tuning

Use Ultralytics' built-in tuning when you need to search for better hyperparameters:

```bash
yolo detect tune data=dataset.yaml model=yolo11s.pt epochs=30 iterations=100
```

This runs 100 training experiments with varied hyperparameters and reports the best configuration. Log the result in `EXPERIMENTS.md` with the experiment ID.

Never tune on the test set. Use the val set for all tuning decisions.

---

## 8. Codebase Auditing

Regular audits prevent the accumulation of technical debt. Schedule these as recurring tasks — not "when there's time" (there is never time).

### 8.1 Sprint Retrospective Audit (After Every Sprint)

At the end of each sprint, before moving it to `completed/`, do a quick pass:

**Code quality:**
- [ ] Run `black --check pipeline/ tests/` — 0 violations?
- [ ] Run `ruff check pipeline/ tests/` — 0 warnings?
- [ ] Run `mypy pipeline/ --strict` — 0 errors?
- [ ] Run `pytest tests/ --cov=pipeline` — coverage requirements met?

**Architecture:**
- [ ] Did any module grow beyond its intended responsibility? (Signs: one file > 300 lines, a function > 40 lines, or a function doing two distinct things.)
- [ ] Are there any new magic numbers that should be moved to `config.yaml`?
- [ ] Did any technical debt get introduced that needs a follow-up issue?

**Data:**
- [ ] Were any new annotation batches added this sprint? Are they committed and versioned?
- [ ] Are model weight directories clean (no orphaned runs)?

### 8.2 Dependency Audit (Monthly)

Run the following to check for security vulnerabilities in dependencies:

```bash
pip install pip-audit
pip-audit
```

`pip-audit` queries the Python Packaging Advisory Database (PyPA) for known CVEs in every package in your environment. Any HIGH or CRITICAL severity finding must be addressed before the next sprint starts.

Also check for significantly outdated packages:

```bash
pip list --outdated
```

Don't blindly update everything — evaluate each update:
1. Does the new version have breaking changes? (Check changelog.)
2. Do tests still pass after updating?
3. Commit the update in its own `chore` commit with the package name and version change in the message.

### 8.3 Dead Code Audit (Per Sprint, During Refactor Phase)

After each sprint's features are implemented and tested, scan for dead code:

```bash
pip install vulture
vulture pipeline/ training/ --min-confidence 80
```

`vulture` finds unused functions, variables, and imports. Review each finding:
- If it's truly unused, delete it. Dead code is a lie — it implies something is built when it isn't.
- If it's a planned-but-not-yet-used interface, add a `# noqa: vulture` comment with a note explaining why it exists.

### 8.4 Performance Audit (Per Sprint)

After each sprint that touches the pipeline, profile the full run on a standard 10-minute test VOD:

```bash
python -m cProfile -o profile.out main.py --input data/raw_videos/benchmark_vod.mp4
python -m pstats profile.out
```

Record the following in the sprint retrospective:

| Metric | Target | Actual |
|---|---|---|
| Total wall-clock time (10 min VOD) | < 5 min | |
| Peak RAM usage | < 4 GB | |
| GPU utilization during inference | > 80% | |
| Frames processed per second | > 30 | |

If any metric regresses significantly from the previous sprint, investigate before closing the sprint.

### 8.5 Data Audit (Before Each Training Run)

Before training any YOLO model, audit the annotation dataset:

1. **Class balance check:** Plot the count per class. If any class has < 50% of the mean class count, the model may underperform on that class. Collect more samples.
2. **Duplicate check:** Images extracted from the same footage 5 seconds apart may be near-identical. Run a perceptual hash deduplication:
   ```bash
   pip install imagehash
   python training/dedup_frames.py --dir data/annotations/killfeed_reader/v1/images/train/
   ```
3. **Label sanity check:** Randomly sample 50 annotated images and visually inspect the bounding boxes. Label errors are common and have an outsized impact on model quality.
4. **Resolution check:** Ensure all images in the dataset are the same resolution as what the model will be trained on (1280px longest edge, after Ultralytics resizing).

---

## 9. Dependency Management & Security

### 9.1 Separate Runtime and Dev Dependencies

`requirements.txt` — runtime only (what's needed to run `main.py`):
```
ultralytics>=8.3.0
opencv-python-headless>=4.10.0
pandas>=2.2.0
pytesseract>=0.3.13
Pillow>=10.4.0
pyyaml>=6.0.2
```

`requirements-dev.txt` — development tools only:
```
-r requirements.txt
pytest>=8.0.0
pytest-cov>=5.0.0
black>=24.10.0
ruff>=0.8.0
mypy>=1.13.0
pre-commit>=4.0.0
vulture>=2.13
pip-audit>=2.7.0
pandas-stubs
types-PyYAML
types-Pillow
```

Install dev deps with: `pip install -r requirements-dev.txt`

### 9.2 Pin All Versions

Every version in both requirements files must be pinned or minimum-pinned:
- `>=` for minimum version (allows compatible updates, tested periodically).
- `==` for exact version (use when a breaking change is known in newer versions).

Never use unpinned dependencies (`ultralytics` without a version constraint). Unpinned dependencies break reproducibility.

### 9.3 No Secrets in Code

- Never commit API keys, credentials, or access tokens to the repository.
- Use environment variables for any credentials: `os.environ.get("ROBOFLOW_API_KEY")`.
- Add a `.env.example` file documenting what environment variables are needed (with no real values).
- Ensure `.gitignore` includes `.env`:

```gitignore
.env
*.env
data/raw_videos/
data/frames/
data/processed/
models/*/weights/*.pt
models/*/versions/
__pycache__/
.venv/
*.pyc
```

---

## 10. Continuous Integration (CI)

Set up a CI pipeline using GitHub Actions. The CI runs on every push and every PR.

**`.github/workflows/ci.yml`:**

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  quality:
    name: Code Quality
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
          cache: "pip"
      - run: pip install -r requirements-dev.txt
      - name: Format check (black)
        run: black --check pipeline/ training/ tests/
      - name: Lint (ruff)
        run: ruff check pipeline/ training/ tests/
      - name: Type check (mypy)
        run: mypy pipeline/ --strict --ignore-missing-imports

  test:
    name: Unit Tests
    runs-on: ubuntu-latest
    needs: quality
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
          cache: "pip"
      - run: sudo apt-get install -y tesseract-ocr
      - run: pip install -r requirements-dev.txt
      - name: Run unit tests with coverage
        run: pytest tests/ -v --cov=pipeline --cov-report=term-missing --cov-fail-under=80
      - name: Upload coverage report
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: htmlcov/

  security:
    name: Dependency Security Audit
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.11"
      - run: pip install pip-audit
      - run: pip-audit -r requirements.txt
```

**CI rules:**
- All three jobs (`quality`, `test`, `security`) must pass before any PR can merge into `main`.
- CI failures are not optional. Do not merge around a failing CI check.
- If a flaky test causes intermittent CI failures, fix the test — don't re-run until it passes by chance.

---

## 11. Data Versioning

Raw videos, annotation datasets, and model weights are too large for standard Git. Manage them with DVC (Data Version Control).

### 11.1 DVC Setup

```bash
pip install dvc
dvc init
dvc remote add -d local_storage /path/to/external/drive/dvc_storage
```

Add large directories to DVC tracking:
```bash
dvc add data/raw_videos/
dvc add data/annotations/
dvc add models/
```

DVC creates small `.dvc` pointer files that ARE committed to Git. The actual large files live in the DVC remote storage.

```bash
git add data/raw_videos.dvc .dvcignore
git commit -m "data: track raw_videos with DVC"
dvc push
```

### 11.2 Reproducing the Data State

Anyone checking out the repo can restore the exact data state for any Git commit:

```bash
git checkout v0.4.0       # e.g., the commit when Model C v1 was trained
dvc pull                  # restores the exact annotation dataset and weights at that point
```

### 11.3 What Goes in Git vs DVC

| Artifact | Git | DVC |
|---|---|---|
| Source code | ✅ | ❌ |
| `config.yaml`, `*.yaml` configs | ✅ | ❌ |
| `dataset.yaml` (YOLO configs) | ✅ | ❌ |
| `collection_log.csv` | ✅ | ❌ |
| `eval_results.json` | ✅ | ❌ |
| `train_config.yaml` | ✅ | ❌ |
| Raw videos (`.mp4`) | ❌ | ✅ |
| Annotated images | ❌ | ✅ |
| Model weights (`.pt`) | ❌ | ✅ |
| Processed CSVs/DBs | ❌ | ✅ |

---

## 12. Documentation Standards

### 12.1 What Needs Docstrings

Every public function in the `pipeline/` and `training/` directories needs a docstring. A docstring answers three questions:

1. What does this function do?
2. What are the inputs and their types?
3. What is returned, and what exceptions can be raised?

```python
def perceive_frame(
    frame: np.ndarray,
    timestamp_ms: int,
    config: dict,
    previous_detection: Optional[FrameDetection] = None,
) -> FrameDetection:
    """
    Run all perception logic against a single video frame.

    Args:
        frame: BGR image array of shape (H, W, 3), as returned by OpenCV.
        timestamp_ms: Timestamp of this frame within the video, in milliseconds.
        config: Loaded config.yaml dictionary.
        previous_detection: The FrameDetection from the immediately preceding frame,
            used to detect ultimate activation (requires before/after comparison).

    Returns:
        A FrameDetection dataclass populated with health_pct, ult_pct,
        ult_activated flag, and any kill events parsed from the kill feed.

    Raises:
        ValueError: If frame dimensions do not match config input_resolution.
    """
```

### 12.2 What Does NOT Need Comments

Do not write comments that restate what the code obviously does:

```python
# BAD — restates the code
# Set health to None if below zero
health = None if raw_value < 0 else raw_value

# GOOD — explains why
# Tesseract occasionally returns negative values for occluded UI regions.
# Treat these as unreadable rather than erroneously low health.
health = None if raw_value < 0 else raw_value
```

Comments explain intent and non-obvious decisions. The code explains what is happening.

### 12.3 `EXPERIMENTS.md`

Maintain this file at the repo root. Every experiment goes here before it goes anywhere else. See Section 7.1 for the template.

### 12.4 Architecture Decision Records (ADRs)

For significant technical decisions (choosing ByteTrack over SORT, using SQLite instead of a flat CSV, choosing pytesseract over a custom OCR model), write a brief ADR in `docs/decisions/`:

```
docs/decisions/
├── adr-001-classical-cv-before-yolo.md
├── adr-002-sqlite-for-state-storage.md
└── adr-003-generator-based-ingestion.md
```

ADR template:
```markdown
# ADR-001: Use Classical CV for Perception MVP Before YOLO

**Date:** 2026-03-13
**Status:** Accepted

## Context
Training YOLO models requires weeks of data collection and annotation. We want to validate the full pipeline end-to-end first.

## Decision
Use OpenCV HSV thresholding for health/ult reading and Tesseract OCR for kill feed parsing in the MVP.

## Consequences
- **Positive:** Pipeline can be tested end-to-end within Sprint 03 without any model training.
- **Negative:** Lower accuracy than a trained model. Health reading will be confused by shield HP. OCR will misread stylized font.
- **Mitigation:** Replace with YOLO models in post-MVP sprints (07+).
```

---

## 13. Release & Versioning

### 13.1 Version Number Policy

Version numbers follow Semantic Versioning (`MAJOR.MINOR.PATCH`):

```
v0.x.x  — Pre-MVP sprints (Sprints 01–06)
v1.0.0  — MVP complete (Sprint 06 done, full pipeline working on a real VOD)
v1.1.0  — Kill feed YOLO model integrated (Sprint 07)
v1.2.0  — UI reader YOLO model integrated (Sprint 08)
v2.0.0  — Hero detector integrated; hero-specific rules enabled
```

### 13.2 Release Checklist

Before tagging any release on `main`:

- [ ] All CI checks green on the release commit
- [ ] `pytest tests/ --cov=pipeline --cov-fail-under=80` passes
- [ ] `pip-audit -r requirements.txt` shows no HIGH or CRITICAL vulnerabilities
- [ ] `EXPERIMENTS.md` updated with any model changes in this release
- [ ] `config.yaml` reviewed — all thresholds tuned for the release version
- [ ] End-to-end test on benchmark VOD completed and processing time recorded
- [ ] Git tag created with annotated message:
  ```bash
  git tag -a v1.0.0 -m "MVP: full pipeline working end-to-end on 1080p Overwatch footage"
  git push origin v1.0.0
  ```

### 13.3 Hotfix Process

If a critical bug is found on `main` after a release:

1. Branch from `main`: `git checkout -b fix/critical-null-health-on-respawn`
2. Fix, add a regression test that would have caught the bug, verify tests pass.
3. Open a PR directly to `main`.
4. After merge, tag a patch release: `v1.0.1`.

---

*Last updated: 2026-03-13*
*This document is a living standard — update it when practices change, not after the fact.*
