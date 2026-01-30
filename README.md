# Sionna–SUMO 3D Ray‑Tracing V2X Simulation

> **Code availability (for paper reuse):** operational code + scenario assets for reproducing the SUMO↔Sionna RT trace-generation pipeline are available in this repository.

This repository provides an **operational research pipeline** for generating **3D ray‑tracing link metrics** (e.g., RSSI / path-loss style traces, depending on the post‑processing implemented in the scripts) by coupling:

- **SUMO** — microscopic traffic simulator (mobility + traffic flows), and  
- **Sionna RT** — 3D ray tracing for wireless propagation (scene + terrain meshes).

The repository is structured for reproducibility: scenarios are stored under `scenarios/`, and Python dependencies are pinned in `requirements.txt`.

---

## Table of Contents

- [What this repo is for](#what-this-repo-is-for)
- [Repository layout](#repository-layout)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Scenarios](#scenarios)
- [How to run](#how-to-run)
- [Outputs](#outputs)
- [Post-processing and plots](#post-processing-and-plots)
- [Reproducibility checklist](#reproducibility-checklist)
- [Troubleshooting](#troubleshooting)
- [Citation](#citation)

---

## What this repo is for

The intended usage is:

1. Select / prepare a scenario (SUMO configuration + 3D scene meshes).
2. Run the main orchestration script that:
   - starts SUMO,
   - queries vehicle states via TraCI,
   - runs Sionna RT ray tracing for the link(s) of interest,
   - logs metrics over time,
   - optionally renders frames.
3. Use the included plotting helpers to produce publication-ready figures.

**Important note:** This repository focuses on *ray-tracing propagation + mobility coupling*. Higher‑layer PHY/MAC or decoding is not the target here unless explicitly implemented in the scripts.

---

## Repository layout

At the repository root you will find (names as committed):

- `scenarios/` — scenario folders (each scenario should be self-contained).
- `signal_propagation_3d_RT.py` — **main** script: orchestrates SUMO + Sionna RT and produces trace logs.
- `requirements.txt` — pinned Python dependencies for reproducibility.
- `test.ipynb` — exploratory notebook.
- `blender_auto.py` / `blender_projection.py` — helper scripts for Blender-based scenario preparation and geometry projection.
- `path_loss_graphs.py` / `graphs_handler.py` — plotting utilities for trace logs.
- `frames2video.py` — helper to stitch rendered frames to video (if you render frames).
- `Protected` — repository artifact file (as committed).

---

## Prerequisites

### System prerequisites

- **Python** 3.x (recommended: match the Python version you used in experiments).
- **SUMO** installed and accessible from terminal as `sumo`.
- (Recommended) **NVIDIA GPU** with CUDA drivers compatible with your installed TensorFlow/Sionna RT stack.

### Optional prerequisites (scenario building)

- **Blender** (version depends on your pipeline; ensure compatibility with your add-ons).
- The Blender add-on you use to import map/OSM data (e.g., Blosm), if your scenario pipeline relies on it.

---

## Installation

### 1) Create a virtual environment

```bash
python -m venv .venv
# Linux/macOS
source .venv/bin/activate
# Windows (PowerShell)
# .\.venv\Scripts\Activate.ps1
```

### 2) Install pinned dependencies

```bash
pip install --upgrade pip
pip install -r requirements.txt
```

**Why pinned deps matter:** Sionna RT depends on a specific stack (TensorFlow + Mitsuba + Dr.Jit). Using pinned versions from `requirements.txt` helps avoid “works on my machine” issues.

### 3) Verify SUMO works

Check that SUMO is installed and visible:

```bash
sumo --version
```

If `sumo` is not found, install SUMO and ensure it is on your `PATH`.

---

## Scenarios

Each scenario should live under:

```
scenarios/<scenario_name>/
```

A scenario is expected to include **at least**:

- `sumo_dir/` containing your SUMO config (commonly `osm.sumocfg`)
- `scenario.xml` — Sionna RT scene description (geometry + materials)
- `terrain.xml` — terrain mesh / geometry description used by the scene (if your pipeline separates it)

> If your repo uses different naming per scenario, keep the scenario folder self-contained and update the main script to match.

### Scenario checklist (recommended)

- SUMO:
  - road network and routes defined
  - simulation duration / step length defined
  - stable and reproducible route generation (if you use randomness, fix seeds where possible)
- Sionna RT:
  - all referenced meshes exist in the scenario folder
  - coordinate system is consistent with SUMO positions (same units, same origin convention)
  - material settings are defined and documented (if you tune materials, record values)

---

## How to run

### 1) Run the main pipeline

```bash
python signal_propagation_3d_RT.py
```

The script typically:
- launches SUMO with the selected scenario config,
- connects via TraCI,
- loads the Sionna RT scene,
- iterates time steps and logs link metrics.

**Configuration:** If you need to change scenario selection, SUMO port, ray-tracing depth, rendering flags, logging output paths, etc., open `signal_propagation_3d_RT.py` and adjust the constants/parameters documented in its header or main section.

### 2) Headless / server execution

If you run on a remote server:
- use headless SUMO (no GUI) → `sumo` (not `sumo-gui`)
- ensure rendering options do not require a display (or disable rendering)

### 3) CPU-only execution (if no GPU)

Sionna RT / Dr.Jit can run in CPU mode via an LLVM backend, but it is **much slower**.  
If you plan to report CPU-vs-GPU timings in your paper, run the same scenario with GPU enabled and disabled and record:
- wall-clock per simulation step
- total run time
- hardware + software versions

---

## Outputs

Depending on script configuration, outputs are typically written under the scenario directory, e.g.:

- `output_data/` — trace data (CSV / NPY / NPZ / etc., depending on script)
- `render_frames/` — rendered frames (if enabled)

Because exact filenames depend on your implementation, treat the scenario folder as the canonical storage location:
- input assets live under `scenarios/<scenario>/...`
- outputs go to `scenarios/<scenario>/(output_data|render_frames)/...`

---

## Post-processing and plots

Two plotting helpers exist in this repository:

- `path_loss_graphs.py` — plot path-loss / RSSI style traces
- `graphs_handler.py` — orchestration / utilities for plotting

Typical flow:
1. run `signal_propagation_3d_RT.py` to generate traces
2. run plotting scripts to produce figures
3. optionally convert frames to video with `frames2video.py`

> If your plotting scripts expect a specific folder structure or filenames, document it in comments at the top of those scripts and keep naming stable.

---

## Reproducibility checklist

For a reviewer-friendly reproducibility statement, record the following in your paper / README:

### Software versions
- OS (e.g., Ubuntu 22.04)
- Python version
- SUMO version
- CUDA version + GPU driver (if GPU is used)
- pinned Python package versions (use `requirements.txt`)

### Determinism
- fix random seeds used by:
  - mobility generation (SUMO route generation, if applicable)
  - ray tracing (PathSolver / sampling)
- keep scenario assets identical between runs
- note that JIT compilation can affect the *first* iteration timing (warm up before benchmarking)

### Hardware
- GPU model + VRAM
- CPU model
- RAM

### Trace generation protocol (recommendation)
- Run one “warm-up” pass (for JIT compilation) before measuring performance.
- Then run the measured pass with identical configuration.

---

## Troubleshooting

### SUMO is not found
- Install SUMO and ensure the `sumo` executable is on `PATH`.
- Verify with `sumo --version`.

### TraCI connection errors
Common causes:
- SUMO did not start
- wrong remote port
- firewall / container networking issues

Fix:
- confirm the SUMO process is running
- ensure the port in your script matches the SUMO launch arguments

### Missing GPU / CUDA issues
- ensure `nvidia-smi` works
- ensure TensorFlow sees the GPU
- validate that your CUDA driver/toolkit matches your pinned TF stack

### Scene loading errors
- missing mesh files referenced by `scenario.xml` / `terrain.xml`
- wrong relative paths
- inconsistent coordinate system

Fix:
- keep all meshes inside the scenario folder
- use relative paths in XML when possible
- validate that vehicle coordinates match scene units

---

## Citation

If you use this repository in academic work, cite the associated paper and link to this repository.

```bibtex
@article{yourkey2026sionna_sumo,
  title   = {Title of the IEEE Access paper},
  author  = {Author1 and Author2 and ...},
  journal = {IEEE Access},
  year    = {2026},
  note    = {Code available at https://github.com/Chartist-1/Sionna_SUMO_simu}
}
```

---

## License

See the repository for a `LICENSE` file. If no license file is present, treat the code as “all rights reserved” until a license is added.
