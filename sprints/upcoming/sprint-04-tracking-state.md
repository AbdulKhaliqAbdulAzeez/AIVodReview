# Sprint 04 — Tracking & State Layer

**Status:** Upcoming
**Depends on:** Sprint 03 complete
**Goal:** Implement `pipeline/track.py` fully. Raw `FrameDetection` objects from the perception layer are transformed into a clean, enriched time-series DataFrame that the logic engine can query directly.

---

## Sprint Goal

> "After processing a VOD, a `.csv` file exists in `data/processed/` that contains one row per sampled frame, with no null values in critical columns (health, ult, team_alive), and derived metric columns correctly computed."

---

## Background

The DataFrame is the heart of the entire system. Every rule in the logic engine reads from it. Errors or noise in this stage (null values, duplicate kill event rows, bad derived metrics) will directly produce false positives or missed detections in the analysis stage.

The key challenge of this sprint is **kill event de-duplication**: the same kill appears in the kill feed across many consecutive frames as it fades out. The tracking layer must collapse these into a single event with one timestamp.

---

## Tasks

### 1. Implement `build_state_dataframe()` in `pipeline/track.py`

**Step 1 — Flatten detections to rows:**

Convert each `FrameDetection` into a dict and build the initial DataFrame:

```python
rows = []
for det in detections:
    rows.append({
        "timestamp_ms":  det.timestamp_ms,
        "health_pct":    det.health_pct,
        "ult_pct":       det.ult_pct,
        "ult_activated": det.ult_activated,
        "raw_kill_events": det.kill_events,  # temp column, removed later
    })
df = pd.DataFrame(rows)
```

**Step 2 — Smooth noisy sensor readings:**

Health and ult values can flicker due to particle effects over the UI. Apply a rolling median to smooth them:
- `health_pct`: rolling median over a 3-frame window.
- `ult_pct`: rolling median over a 3-frame window.
- Forward-fill any `None` values before smoothing (represent frames where UI was occluded as the last known good value).

```python
df["health_pct"] = df["health_pct"].ffill().rolling(window=3, min_periods=1).median()
df["ult_pct"]    = df["ult_pct"].ffill().rolling(window=3, min_periods=1).median()
```

**Step 3 — Kill event de-duplication:**

The same kill event will appear in `raw_kill_events` across multiple consecutive frames. De-duplicate by:
1. Exploding the `raw_kill_events` list column so each kill event gets its own temporary row.
2. Sorting by `timestamp_ms`.
3. For each pair of consecutive kill events with the same `(killer_hero, victim_hero)`, if the time gap is less than a threshold (e.g., 5000ms), keep only the first occurrence.
4. Drop the `raw_kill_events` column and add a clean `kill_event_str` column (nullable string) with the de-duplicated event at the correct timestamp (e.g., `"genji eliminated tracer"`).

**Step 4 — Derive `team_alive` and `enemy_alive`:**

Start at 5 for each team. Walk through the de-duplicated kill events chronologically and decrement the appropriate team count on each event:

```python
team_alive  = 5
enemy_alive = 5

for idx, row in df.iterrows():
    if pd.notna(row["kill_event_str"]):
        event = parse_kill_event(row["kill_event_str"])
        if event.victim_team == "ally":
            team_alive = max(0, team_alive - 1)
        else:
            enemy_alive = max(0, enemy_alive - 1)
    df.at[idx, "team_alive"]  = team_alive
    df.at[idx, "enemy_alive"] = enemy_alive
```

**Note on respawn:** Overwatch has respawning. A 5v5 fight that goes 4v4 will eventually reset back to 5v5 when players respawn. Tracking respawns is out of scope for MVP — document this as a known limitation. The counts will be wrong after the first death and reset only if the logic engine detects a sustained period where no enemies are on screen (proxy for "between fights").

**Step 5 — Derive `time_since_last_kill_ms`:**

- For each row, compute the time elapsed since the most recent `kill_event_str` was non-null.
- Use `cumsum` with a time-delta approach or a forward-fill on a "last kill timestamp" series.

**Step 6 — Finalize schema:**

Drop the `raw_kill_events` temp column. Assert that the final DataFrame has exactly these columns in order:

| Column | Type |
|---|---|
| `timestamp_ms` | int64 |
| `health_pct` | float64 |
| `ult_pct` | float64 |
| `ult_activated` | bool |
| `kill_event_str` | object (nullable) |
| `team_alive` | int64 |
| `enemy_alive` | int64 |
| `time_since_last_kill_ms` | float64 |

**Acceptance criteria:** DataFrame has correct dtypes, no nulls in non-nullable columns, and row count equals number of sampled frames.

---

### 2. Implement `save_state()` — persist the DataFrame

Write the DataFrame out to `data/processed/<video_stem>_state.csv` and also load it into a SQLite database at `data/processed/<video_stem>.db`.

```python
def save_state(df: pd.DataFrame, source_file: str, config: dict) -> str:
    """
    Saves the state DataFrame to CSV and SQLite.
    Returns the path of the CSV file.
    """
```

SQLite table name: `game_state`

The SQLite output allows ad-hoc SQL queries during development for rapid debugging of the logic engine, without needing to re-run the whole pipeline:

```bash
sqlite3 data/processed/myvod.db "SELECT * FROM game_state WHERE health_pct < 0.20 LIMIT 10;"
```

**Acceptance criteria:** Both `.csv` and `.db` files are written and queryable after processing a VOD.

---

### 3. Add a between-fight reset heuristic

As noted in Task 1, kill counts won't reset on respawn. Add a simple heuristic to detect the gap between team fights and reset the alive counts:

**Heuristic:** If `kill_event_str` has been null for more than 30 consecutive seconds AND `ult_activated` has been False, assume a new fight has started and reset both `team_alive` and `enemy_alive` to 5.

This won't be perfect but will prevent alive counts from drifting too far wrong across a long VOD. Document the threshold in `config.yaml`:

```yaml
rules:
  fight_reset_gap_ms: 30000
```

**Acceptance criteria:** In a test VOD with a clear gap between fights, alive counts reset to 5v5 at the correct timestamp.

---

### 4. Write unit tests for `track.py`

**`tests/test_track.py`:**

Build synthetic `FrameDetection` lists in test fixtures rather than using real video data.

- **`test_smoothing()`** — Create a list of detections with one spike value in the middle (e.g., `health_pct` of `[1.0, 1.0, 0.1, 1.0, 1.0]`). Assert the spike is smoothed out and doesn't appear in the final DataFrame.
- **`test_kill_deduplication()`** — Create 10 consecutive frames all containing the same `KillEvent`. Assert the final DataFrame has exactly one row where `kill_event_str` is non-null.
- **`test_team_alive_decrement()`** — Create detections where one ally kill event appears. Assert `team_alive` decrements from 5 to 4 at the correct timestamp.
- **`test_time_since_last_kill()`** — Create detections with a kill at t=1000ms. Assert that `time_since_last_kill_ms` at t=6000ms equals `5000`.
- **`test_fight_reset()`** — Create detections with a gap of >30s with no kills after one death. Assert alive counts reset to 5 after the gap.
- **`test_schema_complete()`** — Assert all 8 expected columns are present with correct dtypes.

**Acceptance criteria:** All 6 tests pass.

---

### 5. Update `main.py` to call `save_state()`

After `track.build_state_dataframe()`, call `track.save_state(df, args.input, config)` and log the output path.

**Acceptance criteria:** Running the pipeline end-to-end produces both `.csv` and `.db` output files.

---

## Definition of Done

- [ ] `build_state_dataframe()` implemented with all 6 steps
- [ ] Smoothing applied to health and ult readings
- [ ] Kill events de-duplicated correctly
- [ ] `team_alive` / `enemy_alive` derived and reset between fights
- [ ] `time_since_last_kill_ms` computed correctly
- [ ] CSV and SQLite output written by `save_state()`
- [ ] All 6 unit tests pass
- [ ] Manually spot-checked: open a processed `.db` file and verified values with SQLite queries
