# AV Clip Scout — Design Spec

Date: 2026-10-16
Status: Approved for implementation (not yet built)

## Problem

`nvidia/PhysicalAI-Autonomous-Vehicles` on Hugging Face is 133TB across
306,152 clips (20s each, 25 countries, 2500+ cities). Multi-camera (7 views)
coverage on all clips, LiDAR on 298,326, radar on 160,761. The HF dataset
viewer is disabled (gated dataset), and the data itself is split into
per-sensor chunk zips of ~100 clips each.

NVIDIA's own README says clip-level metadata is "intended for researchers to
use to identify which subset of chunks they are interested in downloading,"
and that doing so saves "significant storage space and bandwidth." But no
browsing tool ships with the dataset — just three flat files the user is
expected to load into pandas by hand:

- `clip_index.parquet` (11MB, top-level)
- `metadata/feature_presence.parquet` (per-clip sensor availability)
- `metadata/data_collection.parquet` (country, month, time-of-day, etc.)

At 3.4M downloads of the parent dataset, this is a problem a lot of people
hit, not just us. AV Clip Scout is a small tool that fills that one gap.

## Goals (Phase 1 — this spec)

- Load and join the three metadata files locally.
- Filter clips by country, weather, road type, time-of-day, traffic density,
  and per-sensor presence (camera/LiDAR/radar).
- Export a manifest (JSON) of matching clip UUIDs + their chunk file
  references, for the user to feed into NVLabs' `physical_ai_av` devkit
  themselves.
- Before that manifest is written, print an estimated total download size
  (per sensor + total) for the filtered result, so the person sees the cost
  in GB/TB before they go run the devkit's actual pull. This is the one
  number that makes "133TB, 306K clips" tractable to reason about.
- CLI and a local Streamlit dashboard, both built on the same filtering core.

## Non-goals (Phase 1)

- Does not download, cache, or redistribute any NVIDIA dataset file. The
  license (clause 4.6) forbids redistributing the Dataset "in whole or in
  part" — this repo ships code only. Users accept NVIDIA's gate and download
  the 3 metadata files themselves; AV Clip Scout just reads files already on
  their disk.
- Does not call or wrap `physical_ai_av` to trigger downloads. Manifest out,
  devkit in, stays decoupled — no dependency on devkit API stability.
- Does not include a synthetic/demo data mode. Running the tool requires the
  real (small) metadata files.
- Does not build any dashcam/video-analysis feature (see Future Direction).

## Architecture

```
clip_index.parquet ─┐
feature_presence.parquet ─┼─▶ clip_index.py (load_index, filter_clips) ─▶ size_estimate.py (estimate_size)
data_collection.parquet ─┘                                    │                      │
                                                                │                      ▼
                                                                │          manifest.py (build_manifest) ─▶ manifest.json
                                                                ├─▶ cli.py (argparse)
                                                                └─▶ dashboard.py (Streamlit)
```

### `clip_index.py`
- `load_index(data_dir: Path) -> pd.DataFrame` — reads the 3 parquet files,
  joins on `clip_uuid` into one DataFrame.
- `filter_clips(df, country=None, weather=None, road_type=None,
  time_of_day=None, traffic=None, has_lidar=None, has_radar=None) ->
  pd.DataFrame` — all filters optional, AND-combined.

### `size_estimate.py`
- NVIDIA publishes total dataset size (133TB) and per-sensor clip coverage
  counts, but not a per-clip byte size column — so exact size isn't
  derivable from metadata alone. Approach: download one sample chunk per
  sensor type during implementation, measure its real zip size, divide by
  its clip count to get `SENSOR_AVG_BYTES[sensor] ≈ bytes/clip`. Hardcode
  that small constants table in this module.
  - `# ponytail: avg-bytes-per-clip constants measured from one sample chunk
    per sensor, not exact per-clip truth. Recalibrate if NVIDIA changes
    video/point-cloud encoding (watch the dataset's Version History table).`
- `estimate_size(filtered_df, sensors_wanted) -> dict` — returns
  `{sensor: estimated_gb, ...}` plus `total_gb`, using the constants above.
  Always labeled `~estimated` in CLI/dashboard output, never presented as
  exact.
- CLI `filter` and the dashboard both print this (count + per-sensor +
  total GB) before/alongside writing `manifest.json` — the actual
  terabyte-scale pull happens later in the devkit, not in this tool, so this
  is the one chance to see the cost before committing to that.

### `manifest.py`
- `build_manifest(filtered_df) -> list[dict]` — resolves each matching
  clip's chunk file reference per sensor, e.g.:

```json
[
  {
    "clip_uuid": "a1b2c3...",
    "country": "Germany",
    "weather": "rain",
    "chunks": {
      "camera_front_wide_120fov": "chunk_0042",
      "lidar_top_360fov": "chunk_0042",
      "radar_corner_front_left_srr_0": "chunk_0042"
    }
  }
]
```

### `cli.py` (stdlib `argparse`, no extra dependency)
```
avclipscout filter --country Germany --weather rain --has-lidar --out manifest.json
# Matched 4,231 clips.
# Estimated size: camera ~140GB, lidar ~35GB, radar 0GB (not requested). Total: ~175GB (~estimated).
# Wrote manifest.json

avclipscout stats --field country     # prints valid values + counts; also typo-help
```

### `dashboard.py` (Streamlit)
- Sidebar: multiselect/checkbox filters mirroring the CLI flags.
- Main: `st.bar_chart` for distribution of matches by country/weather,
  `st.dataframe` preview of first N matching clips, an estimated-size
  readout (per sensor + total, `~estimated`) shown above the download
  button for the same `manifest.json` the CLI produces.

## Data flow

1. User accepts NVIDIA's license on the HF dataset page.
2. User downloads only `clip_index.parquet` + the two `metadata/*.parquet`
   files (tens of MB, not the 133TB) into a local directory.
3. User points AV Clip Scout at that directory via CLI flag or dashboard
   input.
4. Filter → manifest.json.
5. User feeds that manifest into `physical_ai_av` (NVLabs devkit) themselves
   to pull only the matching sensor chunks. Out of scope here.

## Error handling

- Missing local metadata file → error names the exact missing file and the
  HF path to fetch it from, not "re-run the whole download."
- Unknown/misspelled filter value → empty result set + a pointer to
  `avclipscout stats --field <field>` to see valid values.

## Testing

- pytest, with a small hand-built parquet fixture (4-5 rows) matching the
  real schema — exercises `filter_clips`, `estimate_size`, and
  `build_manifest` without needing gated access. This is a test fixture
  only, not a shipped demo-mode feature (that was explicitly declined for
  the product itself). `estimate_size` tests use fixed constants, not real
  measured ones, so they stay deterministic.
- Dashboard gets a manual smoke run, not unit tests — it's thin glue over
  the already-tested filtering core.

## Dependencies

`pandas`, `pyarrow`, `streamlit`. CLI uses stdlib `argparse`. Nothing else.

## Future direction (not scoped here)

The filtering/manifest layer operates on *metadata*. A possible future
extension — auto-tagging "interesting" moments (hard braking, close
pedestrian proximity, adverse weather) from decoded clip content — would be
a separate module consuming *decoded sensor data*, not metadata. That's a
different input and a different concern, so no shared interface is being
built for it now. If it happens, it happens as its own spec against
downloaded clips, generalizable later to arbitrary dashcam footage (fleet /
insurance / personal use), not against this repo's metadata layer.

## License note

This repository's own code will be MIT licensed. It contains no NVIDIA
dataset files, images, or derived data of any kind.
