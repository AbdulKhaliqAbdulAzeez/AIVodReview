# Sprint 05 — Logic Engine

**Status:** Upcoming
**Depends on:** Sprint 04 complete
**Goal:** Implement `pipeline/analyze.py` with the V1 rule set. The system can read a processed state DataFrame and produce a list of timestamped, categorized `MistakeEvent` objects.

---

## Sprint Goal

> "Given a real processed VOD state CSV, `run_analysis()` returns at least one correctly identified `MistakeEvent`, and zero false positives on known-clean segments of the VOD."

---

## Background

The logic engine is where the system goes from "data pipeline" to "coaching tool." The rules in V1 are intentionally conservative — it's better to miss a real mistake than to flood the player with false positives. Thresholds are all driven from `config.yaml` so they can be tuned without touching code.

Each rule must be:
1. **Independent** — a single Python function that can be enabled/disabled without touching others.
2. **Documented** — a docstring explaining exactly what the rule detects and why.
3. **Threshold-driven** — no hardcoded numbers inside rule functions.

---

## Tasks

### 1. Define `MistakeEvent` and rule framework

At the top of `analyze.py`, define the `MistakeEvent` dataclass and a type alias for rule functions:

```python
from dataclasses import dataclass
from typing import List, Callable
import pandas as pd

@dataclass
class MistakeEvent:
    timestamp_ms: int
    category: str
    severity: str        # "low" | "medium" | "high"
    description: str

RuleFunction = Callable[[pd.DataFrame, dict], List[MistakeEvent]]
```

Define a registry list that `run_analysis()` iterates over:

```python
RULES: List[RuleFunction] = [
    rule_wasted_ultimate,
    rule_critical_health_engaged,
    rule_ult_into_lost_fight,
    rule_tunnel_vision,
]
```

`run_analysis()` is then simply:

```python
def run_analysis(df: pd.DataFrame, config: dict) -> List[MistakeEvent]:
    events = []
    for rule in RULES:
        events.extend(rule(df, config))
    return sorted(events, key=lambda e: e.timestamp_ms)
```

This makes adding, removing, or reordering rules trivial.

---

### 2. Implement Rule 1 — Wasted Ultimate

**Function:** `rule_wasted_ultimate(df, config)`

**Detection logic:**
1. Find all rows where `ult_activated == True`.
2. For each such row, check if `team_alive <= config["rules"]["wasted_ult_team_alive_max"]` (default: 2).
3. If so, emit a `MistakeEvent`.

```python
def rule_wasted_ultimate(df: pd.DataFrame, config: dict) -> List[MistakeEvent]:
    """
    Flags any ultimate used when the player's team has 2 or fewer members alive.
    In Overwatch, using an ultimate when a fight is already lost wastes a
    resource that could be decisive in the next fight.
    """
    threshold = config["rules"]["wasted_ult_team_alive_max"]
    events = []
    ult_frames = df[(df["ult_activated"] == True) & (df["team_alive"] <= threshold)]
    for _, row in ult_frames.iterrows():
        events.append(MistakeEvent(
            timestamp_ms=int(row["timestamp_ms"]),
            category="Ultimate Economy",
            severity="high",
            description=f"Ultimate used while your team had only {int(row['team_alive'])} player(s) alive. "
                        f"The fight is likely already lost — save it for the next one."
        ))
    return events
```

**Acceptance criteria:** Correctly flags a manually identified "wasted ult" moment in a test VOD.

---

### 3. Implement Rule 2 — Critical Health While Engaged

**Function:** `rule_critical_health_engaged(df, config)`

**Detection logic:**
1. Mark rows where `health_pct < config["rules"]["critical_health_pct"]` (default: 0.20).
2. Find contiguous runs of critical-health rows using a groupby on a change-detection column.
3. For each run lasting more than `config["rules"]["critical_health_duration_ms"]` milliseconds (default: 3000ms), flag it.
4. Use the timestamp of the first frame in the run as the event timestamp.

```python
def rule_critical_health_engaged(df: pd.DataFrame, config: dict) -> List[MistakeEvent]:
    """
    Flags periods where the player remains at critical health (< 20%) for more
    than 3 seconds without dying or recovering. This indicates the player is
    not prioritizing self-preservation or is too deep in a fight.
    """
```

**Tip for contiguous run detection:**
```python
df["is_critical"] = df["health_pct"] < threshold
df["run_id"] = (df["is_critical"] != df["is_critical"].shift()).cumsum()
critical_runs = df[df["is_critical"]].groupby("run_id")
```

**Acceptance criteria:**
- Flags a segment where the player is visibly at low health for 3+ seconds.
- Does NOT fire if the health dips critical for only 1–2 frames (transient noise).

---

### 4. Implement Rule 3 — Ultimate Into Lost Fight

**Function:** `rule_ult_into_lost_fight(df, config)`

**Detection logic:**
1. Find all rows where `ult_activated == True`.
2. For each, look backwards in the DataFrame over a 5-second window.
3. Count the number of rows in that window where `kill_event_str` contains an allied death.
4. If 3 or more allied deaths occurred in the preceding 5 seconds, flag it.

**Data limitation:** Kill events in the MVP don't reliably encode team affiliation (Tesseract OCR doesn't parse icon colors). For the MVP, use a simpler proxy: if `team_alive` at the ult activation frame is ≤ 2 AND `time_since_last_kill_ms` < 5000ms (a kill happened very recently), flag it. Document this as an approximation.

**Acceptance criteria:** Correctly flags at least one instance across a full VOD without false-positive triggering in winning fights.

---

### 5. Implement Rule 4 — Tunnel Vision

**Function:** `rule_tunnel_vision(df, config)`

**Detection logic:**
1. Mark rows where `health_pct < 0.25`.
2. Within those rows, find windows where `time_since_last_kill_ms > config["rules"]["tunnel_vision_stall_ms"]` (default: 8000ms).
3. Flag the start of such a window if it lasts more than 5 consecutive seconds.

**Logic rationale:** If you have been at low health, no kills are being exchanged in either direction, and you are not dying or recovering, you are likely stuck in a bad fight that should have been disengaged from already.

**Acceptance criteria:** Flags at least one "stalled engagement while low health" scenario in a manually reviewed VOD segment.

---

### 6. Add a minimum confidence guard to all rules

Before emitting any event, each rule should check a minimum confidence score based on the quality of the input data:

```python
def _data_quality_ok(rows: pd.DataFrame) -> bool:
    """
    Returns False if more than 30% of the rows in the window have null health_pct.
    This guards against firing rules on frames where the UI was not visible.
    """
    null_ratio = rows["health_pct"].isna().mean()
    return null_ratio < 0.30
```

Call this inside each rule before appending a `MistakeEvent`. If data quality is insufficient for a window, skip the event rather than potentially misfiring.

---

### 7. Write unit tests for `analyze.py`

**`tests/test_analyze.py`:**

Build synthetic DataFrames that perfectly trigger (or should NOT trigger) each rule:

- **`test_rule1_fires()`** — Build a DataFrame with `ult_activated=True` and `team_alive=1`. Assert one `MistakeEvent` returned with `category="Ultimate Economy"`.
- **`test_rule1_no_fire()`** — `ult_activated=True` and `team_alive=5`. Assert empty list returned.
- **`test_rule2_fires()`** — Build 50 consecutive rows with `health_pct=0.10` (>3 seconds at 10 FPS). Assert one event returned.
- **`test_rule2_no_fire()`** — Only 2 rows with `health_pct=0.10`. Assert empty list.
- **`test_rule4_fires()`** — 100 rows with `health_pct=0.20` and `time_since_last_kill_ms=10000`. Assert one event returned.
- **`test_run_analysis_sorted()`** — Build a DataFrame that triggers multiple rules. Assert the returned list is sorted by `timestamp_ms` ascending.

**Acceptance criteria:** All 6 tests pass.

---

### 8. Manual validation on a real VOD segment

Select a 5-minute segment of a real Overwatch VOD that you have manually reviewed and know the mistakes in. Run the full pipeline on it and compare the output to your manual review:

| Expected Event | Detected? | Notes |
|---|---|---|
| (fill in during sprint) | | |

Document false positives and false negatives. Use findings to tune thresholds in `config.yaml`.

---

## Definition of Done

- [ ] `MistakeEvent` dataclass and rule registry defined
- [ ] All 4 rules implemented
- [ ] Data quality guard implemented in all rules
- [ ] All 6 unit tests pass
- [ ] `run_analysis()` returns sorted events list
- [ ] Manual validation table completed for a 5-minute VOD segment
- [ ] Threshold calibration pass done — config.yaml values updated to reflect tuned values
