# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

**eo_pipe** is the LSST Camera team's electro-optical (EO) testing analysis package for sensor characterization. It analyzes outputs from the LSST calibration pipelines (`cp_pipe`) and implements custom analysis pipelines using LSST stack code. The algorithms are based on LSST CCD testing specifications (LCA-128 and LCA-10103).

## Repository Structure

```
eo_pipe/
├── python/lsst/eo/pipe/          # Main Python package with analysis tasks
│   ├── *Task.py                  # ~30+ pipeline tasks (ReadNoise, PTC, Defects, etc.)
│   ├── plotting/                 # Focal plane and visualization utilities
│   ├── *_utils.py                # Utility modules (isr, report, selection, dsref, ingest)
│   └── eoPipelines.py            # Pipeline orchestration and bps submission
├── pipelines/                    # YAML pipeline definitions (~26 pipelines)
├── bps/                          # Batch processing submission configs for bps
│   ├── bps_*.yaml                # Individual pipeline bps configs
│   └── cp_pipe/                  # cp_pipe related configs
├── tests/                        # Unit tests
├── data/                         # Configuration files and templates
│   ├── eo_pipelines_config.yaml  # Pipeline run type configurations
│   ├── eo_html_report.yaml       # Report template
│   └── style.css                 # Report styling
├── doc/                          # Documentation (Jupyter notebooks)
└── SConstruct                    # LSST SCons build configuration
```

## Build & Installation

The package uses **LSST's SCons build system** (not setuptools). Installation is managed through the LSST Stack environment.

```bash
# Build the package in-place
scons

# Build with verbose output
scons -v

# Clean build artifacts
scons -c
```

The package requires the LSST stack environment with dependencies including:
- `lsst.pipe.base` - LSST pipeline framework
- `lsst.daf.butler` - Data Butler for data access
- `lsst.obs.lsst` - LSST camera definitions
- `lsst.ip.isr` - Instrument signature removal
- `lsst.cp.pipe` - LSST calibration pipelines
- matplotlib, pandas, numpy

## Running Tests

```bash
# Run all tests
python -m pytest tests/

# Run a specific test
python -m pytest tests/test_defects.py::test_tabulate_defects

# Run with verbose output
python -m pytest tests/ -v

# Run the test directly (if it has a main block)
python tests/test_defects.py
```

Currently there is one main test file: `tests/test_defects.py` which tests the defect tabulation logic for both ITL and e2V CCDs.

## Architecture: Core Concepts

### 1. Pipeline Task Structure

The codebase follows the **LSST Pipe Base Task pattern**. Each analysis pipeline consists of:

- **Task Class**: Inherits from `lsst.pipe.base.PipelineTask` (or `Task`), implements the analysis algorithm
- **TaskConfig Class**: Inherits from `lsst.pipe.base.PipelineTaskConfig`, defines configurable parameters
- **TaskConnections**: Defines inputs (Butler datasets) and outputs (analysis results or plots)
- **Run Methods**: 
  - `run()` or `runQuantum()` - implements the analysis logic
  - Helper functions (module-level) for extracting data (`get_amp_data()`, `get_plot_locations()`)

### 2. Key Task Categories

**Analysis Tasks** (produce quantitative results):
- `ReadNoiseTask` - Read noise from bias frames
- `PtcPlotsTask` - Photon transfer curve analysis
- `DefectsTask` (bright/dark) - Pixel defect detection
- `DarkCurrentTask`, `EperTask`, `LinearityPlotsTask`, `BfAnalysisTask`

**Diagnostic Tasks** (detect hardware issues):
- `BiasStabilityTask` - Bias level stability
- `FlatGainStabilityTask` - Gain stability
- `BiasShiftsTask` - Bias shifts in darks
- `DivisaderoTearingTask` - Tearing artifacts

**Correlation Tasks** (characterize amp interactions):
- `RaftAmpCorrelationsTask` - Correlation between amps
- `CtiVsFluxTask` - CTI vs flux relationship

**Mosaic/Reporting Tasks**:
- `RaftMosaicTask`, `RaftCalibMosaicTask` - CCD-level mosaics
- `FocalPlaneMosaicTask` - Focal plane mosaics
- Report generation tasks produce focal plane visualizations

### 3. Data Flow

1. **Input**: Raw camera data ingested into Butler collections
2. **Processing**: Tasks orchestrated via YAML pipelines, executed using `bps submit`
3. **Outputs**: 
   - **Datasets** (stored in Butler): DataFrames, Defects objects, camera calibrations
   - **Plots**: Matplotlib figures (focal plane mosaics, histograms)
4. **Reporting**: `report_utils.generate_report()` aggregates plots and data into HTML reports

### 4. Data Retrieval Pattern

Module-level functions follow this convention:

```python
def get_amp_data(repo, collections, camera=None):
    """Return amp-level data as dict[field_name][det_name][amp_name] = value"""
    butler = daf_butler.Butler(repo, collections=collections)
    # Query datasets and extract per-amplifier statistics
    return amp_data

def get_plot_locations(repo, collections):
    """Return dict mapping plot dataset types to file locations"""
    return get_plot_locations_by_dstype(repo, collections, dstype_names)
```

This pattern allows `report_utils.py` to aggregate results from all tasks into a unified report.

### 5. Pipeline Orchestration

**`eoPipelines.py`** manages batch submission:
- `PipelinesBase` - Base class for pipeline submission
- `EoPipelines` - Submits eo_pipe analysis pipelines via `bps submit`
- `CpPipelines` - Sequentially submits cp_pipe calibration pipelines with status tracking

**Run Types** (defined in `data/eo_pipelines_config.yaml`):
- `b_protocol` - Full characterization (~14 pipelines)
- `ptc` - PTC and related analysis
- `bias_sequence`, `dark_sequence` - Calibration-focused runs
- `flat_gain_stability`, `persistence`, etc. - Targeted analysis

## Key Utilities

- **`isr_utils.py`**: Minimal ISR (Instrument Signature Removal) - overscan correction, bias/dark subtraction
- **`report_utils.py`**: HTML report generation, aggregating results from all tasks
- **`selection_utils.py`**: Dataset selection and filtering
- **`dsref_utils.py`**: Butler dataset reference utilities
- **`ingest_utils.py`**: Data ingestion helpers
- **`plotting/`**: Focal plane visualization (focal_plane_plotting.py, divisadero_plotting.py)

## Environment & Configuration

### Required Environment Variables

Set before running pipelines:
```bash
export BUTLER_CONFIG=<path to butler config>
export INSTRUMENT_NAME=LSSTCam|LSST-TS8|LATISS
export INSTRUMENT_CLASS=lsst.obs.lsst.LsstCam (etc.)
export EO_PIPE_DIR=<eo_pipe installation directory>
export EO_PIPE_INCOLLECTION=<input butler collection>
export DATASET_SELECTION=<query string for raw data selection>
export DATASET_LABEL=<label for outputs>
export WEEKLY=<weekly run identifier>
```

### BPS (Batch Processing Service)

Pipelines are submitted via the `bps` command using YAML configurations in `bps/`:
```bash
bps submit bps/bps_eoReadNoise.yaml
bps report --id <submit_dir>
```

BPS configs include HTCondor settings and Parsl execution settings for distributed processing.

## Common Development Patterns

### Adding a New Analysis Task

1. Create `python/lsst/eo/pipe/myTask.py` with:
   - `MyTaskConnections` class (define inputs/outputs)
   - `MyTaskConfig` class (configuration parameters)
   - `MyTask` class (implements `run()` or `runQuantum()`)
   - Module-level `get_amp_data()` and `get_plot_locations()` functions

2. Add to `python/lsst/eo/pipe/__init__.py`: `from .myTask import *`

3. Create YAML pipeline in `pipelines/eoMyAnalysis.yaml`

4. Create BPS submission config in `bps/bps_eoMyAnalysis.yaml`

5. Add to pipeline run type in `data/eo_pipelines_config.yaml`

### Key Import Conventions

```python
import lsst.pipe.base as pipeBase
from lsst.pipe.base import connectionTypes as cT
import lsst.pex.config as pexConfig
import lsst.daf.butler as daf_butler
from lsst.obs.lsst import LsstCam  # or LsstTS8, Latiss
```

### Focal Plane Plotting Pattern

Tasks generating focal plane visualizations use:
```python
from .plotting import plot_focal_plane, hist_amp_data, append_acq_run

# Create per-amp data dict: {det_name: {amp_name: value, ...}, ...}
plot_focal_plane(ax, amp_data, camera=camera, title=title, z_range=None, use_log10=False)
hist_amp_data(amp_data, xlabel, hist_range=None, use_log10=False, title=title)
```

## Tests

The single test file is `tests/test_defects.py`, which tests defect tabulation logic against synthetic images for both ITL (detector 27) and e2V (detector 94) CCD types using `lsst.obs.lsst.LsstCam`.
