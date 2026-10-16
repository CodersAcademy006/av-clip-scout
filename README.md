# AV Clip Scout

A metadata-driven clip browser for [`nvidia/PhysicalAI-Autonomous-Vehicles`](https://huggingface.co/datasets/nvidia/PhysicalAI-Autonomous-Vehicles) — filter its 306,152 clips by country, weather, road type, time-of-day, and sensor coverage, see an estimated download size, and export a manifest, *before* touching any of the dataset's 133TB.

**Status: design phase.** Spec is written and approved, code isn't yet. See [`docs/superpowers/specs/2026-10-16-av-clip-scout-design.md`](docs/superpowers/specs/2026-10-16-av-clip-scout-design.md) for the full design.

## Why

NVIDIA's PhysicalAI-Autonomous-Vehicles dataset is 133TB across 306,152 twenty-second clips from 25 countries and 2500+ cities — multi-camera on every clip, LiDAR on 298,326, radar on 160,761. The Hugging Face dataset viewer is disabled for it, and the sensor data ships in per-sensor chunk zips of ~100 clips each.

NVIDIA's own dataset card says the clip-level metadata is meant for researchers to pick a subset of chunks before downloading, to save "significant storage space and bandwidth" — but ships no tool to actually browse that metadata. Just three parquet files you're expected to load into pandas yourself. At 3.4M downloads of the parent dataset, that's a gap a lot of people hit, not just whoever's reading this.

AV Clip Scout is a small tool that fills that one gap: filter, see the estimated size, export a manifest, hand that to NVIDIA's own [`physical_ai_av`](https://github.com/NVlabs/physical_ai_av) devkit to actually pull data.

## What it will do (Phase 1)

- Filter clips by country, weather, road type, time-of-day, traffic density, and per-sensor presence (camera/LiDAR/radar).
- Print an estimated download size (per sensor + total) for the filtered result before you export anything, so you see the cost before committing.
- Export a manifest (clip UUIDs + chunk file references) for use with NVLabs' `physical_ai_av` devkit.
- CLI and a local Streamlit dashboard, same filtering core under both.

## What it explicitly won't do

- **No dataset redistribution.** This repo contains, and will ever contain, zero NVIDIA dataset files. NVIDIA's license (clause 4.6) forbids redistributing the Dataset "in whole or in part." You accept NVIDIA's license and download the three small metadata files yourself; this tool only reads files already on your disk.
- **No devkit wrapping.** Manifest out, `physical_ai_av` in — stays decoupled, no dependency on devkit internals.
- **No bundled synthetic/demo data.** Running it requires the real metadata files.
- **No video/sensor-content analysis** (yet). See [Future direction](docs/superpowers/specs/2026-10-16-av-clip-scout-design.md#future-direction-not-scoped-here) in the spec — a possible later extension toward auto-tagging driving events from decoded clips, generalizable to arbitrary dashcam footage. Not started, not promised.

## Getting the data

1. Accept the license on the [dataset page](https://huggingface.co/datasets/nvidia/PhysicalAI-Autonomous-Vehicles).
2. Download just these three files (tens of MB total, not the 133TB):
   - `clip_index.parquet`
   - `metadata/feature_presence.parquet`
   - `metadata/data_collection.parquet`
3. Point AV Clip Scout at the directory containing them (once the CLI exists).

## License

Code in this repository: MIT (see [`LICENSE`](LICENSE)). No NVIDIA dataset files, images, or derived data are included or distributed here.
