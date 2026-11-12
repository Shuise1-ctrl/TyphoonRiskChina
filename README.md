# TyphoonRiskChina

[中文说明](README.zh-CN.md)

TyphoonRiskChina is a browser-based local framework for forecast-driven typhoon wind-field downscaling and direct economic-loss assessment in coastal China. It links numerical forecast fields and a matching storm-track workbook to hourly 10-m wind fields at approximately 3 km grid spacing, event-maximum wind hazard, residential-building exposure, vulnerability functions, and spatially aggregated loss estimates. The Vue interface and FastAPI calculation service run on the same computer; scientific inputs and generated results are not sent to a remote calculation server.

> This repository is designed to be cloned and run locally. It is not configured for GitHub Pages: the current asset paths and history-based routes are intended for the local root URL, and Pages cannot run the Python/TensorFlow calculation API.

## What is included

| Component | Included in Git | Works after basic setup | Additional requirements |
|---|---:|---:|---|
| Compiled web interface | Yes | Yes | Internet only for the Windy embed and OpenStreetMap tiles |
| Vue 3 source code | Yes | Yes | Node.js is required only to rebuild the frontend |
| Local FastAPI service | Yes | Yes | CPython 3.10 x64 and the basic requirements |
| Historical CMA-track interface | Yes | Yes | The packaged local `cma-data.js`; map tiles use OpenStreetMap |
| Lightweight reviewer CSV workflow | Yes | API only | Basic Python environment |
| FNL wind-field downscaling | Source only | No | Full Python environment, H5 model, scalers, and 28 FNL files |
| Five-province loss assessment | Source only | No | Full environment plus exposure/grid Parquet files |

## Manuscript terminology

| Term used in the manuscript | Meaning in TyphoonRiskChina |
|---|---|
| Forecast-driven typhoon wind-field downscaling | Converts numerical forecast fields into higher-resolution wind fields along the storm track. |
| Physics-guided decoupled U-Net | The downscaling model used to generate high-resolution wind hazard. |
| Hourly 10-m wind speed and event-maximum wind speed | Hourly downscaled wind fields are summarized on a fixed grid to obtain the event maximum used by the loss workflow. |
| Hazard–exposure–vulnerability (HEV) framework | Combines wind hazard, residential-building exposure, and vulnerability functions to estimate losses. |
| Whole-building vulnerability function and mean loss ratio | The adopted curve maps 10-min mean wind speed to a whole-building mean loss ratio; the loss module aligns wind input to the curve's metric. |
| Modeled direct economic loss | A scenario-based estimate, not an observation of actual damage or a claim of independent validation. |

The interface labels, output names, and NOUL example below follow these terms.

Large scientific assets and generated jobs are intentionally excluded. A normal GitHub repository cannot carry the roughly 15 GB FNL cases or the historical calculation outputs, and the upstream redistribution rights of the model and exposure data must be checked separately.

## Repository layout

```text
TyphoonRiskChina/
├─ frontend_source/       Vue 3 + Vite source
├─ web/                   compiled frontend used by the local server
├─ backend/               FastAPI wrapper and wind/loss stage runners
├─ engine/                lightweight reviewer-demo engine
├─ algorithm_full/        TyphoonRiskChina 1.1.0 scientific source
├─ runtime/               ignored placeholders for external scientific assets
├─ example_folder/        small reviewer example
├─ data/                  generated jobs and local registry at runtime
├─ tests/                 backend and integration tests
├─ docs/                  architecture and validation notes
├─ setup_windows.bat      basic local installation
├─ setup_full_windows.bat full scientific dependency installation
└─ start_windows.bat      starts the integrated web/API service on port 5173
```

## System requirements

- Windows 10 or 11 x64.
- 64-bit CPython 3.10.x. The full pipeline is pinned to Python 3.10 because it uses `tensorflow-cpu==2.10.1`.
- Internet access during dependency installation.
- Approximately 2 GB free space for the full Python environment, plus space for external FNL/exposure data and generated results.
- Node.js 18/20/22 or newer only if you intend to modify and rebuild the frontend. Node is not required for ordinary use of the compiled web application.

## Quick start on Windows

### 1. Obtain the repository

```powershell
git clone https://github.com/Shuise1-ctrl/TyphoonRiskChina.git
cd TyphoonRiskChina
```

You can also download GitHub's source ZIP and extract it to a normal writable folder.

### 2. Install the basic web/API environment

Double-click:

```text
setup_windows.bat
```

This creates `.venv` and installs FastAPI, Uvicorn, upload validation, YAML, and XLSX support. It is sufficient to open the application, browse the historical archive, call the lightweight reviewer API, and inspect service readiness.

### 3. Install the full scientific environment

To run wind-field downscaling or loss assessment, also double-click:

```text
setup_full_windows.bat
```

This creates `.venv-full` and installs the pinned Windows CPU stack. The environment is large; installation may take several minutes. Do not replace `algorithm_full/requirements-windows.txt` with the generic TensorFlow or NetCDF requirement sets.

### 4. Add the external scientific assets

Place the data under `runtime/` using the layout documented in [runtime/README.md](runtime/README.md). The essential paths are:

```text
runtime/downscaling/models/direct_unet_model.h5
runtime/downscaling/scalers/scaler_x.save
runtime/downscaling/scalers/scaler_y.save
runtime/cases/<CASE>/FNL/006.nc ... 168.nc
runtime/exposure/00000 沿海省份风险暴露/*.parquet
runtime/exposure/00000 沿海省份网格/*.parquet
```

Assets may be stored outside the repository. Copy `config.local.json.example` to `config.local.json`, edit the paths, and do not commit that local file. This JSON file is read by `start_windows.bat`; a manual Uvicorn or Vite development launch must instead set the corresponding `TYPHOON_*` environment variables shown in `.env.example`.

### 5. Start the application

Double-click:

```text
start_windows.bat
```

The browser opens at [http://127.0.0.1:5173/](http://127.0.0.1:5173/). Keep the console window open; closing it stops the local service.

To inspect readiness while the service is running, double-click `verify_windows.bat` or open:

- health status: `http://127.0.0.1:5173/api/health`
- API documentation: `http://127.0.0.1:5173/docs`

Top-level `status: ok` means the web/API service is alive. Full scientific readiness must also show `engines.wind.local.ready: true` and `engines.loss.ready: true`.

## Using the web interface

### Historical typhoon database

The archive loads the packaged CMA best-track dataset, supports year/name filtering, point-by-point animation, and hover inspection. The boundary files and track archive are local. OpenStreetMap basemap tiles require an Internet connection.

### Wind-field workflow

1. Select the folder named exactly `FNL`.
2. Select the matching `.xlsx` track workbook.
3. Wait for both inputs to be verified.
4. Generate the wind field.
5. Inspect the interactive maximum-wind map and download the result package.

The browser sends only the FNL file manifest (relative names, sizes, and modification times). The large NetCDF bodies are read in place by the local backend from a configured/approved folder on the same computer. The selected folder must therefore be the same physical folder that the backend can read under `runtime/cases` or the local approved-source registry.

The FNL folder must contain exactly these 28 top-level files and no additional `.nc` files:

```text
006.nc, 012.nc, 018.nc, ... , 162.nc, 168.nc
```

Each file must expose the FuXi array variable `__xarray_dataarray_variable__` with the dimensions `time`, `step`, `level`, `lat`, and `lon`. After case-insensitive label normalization, the `level` coordinate must contain `u10`, `v10`, `t2m`, `msl`, and `r850`.

The track input must be one unencrypted `.xlsx` file under 16 MiB. Use `time`, `LAT`, and `LONG` columns, with at least two distinct valid times. Latitude must be within -90 to 90 and longitude within -180 to 360. Track times must use the same Beijing-time convention expected by the downscaling case.

### Loss workflow

After a wind job completes, open the Loss module and calculate loss directly from that wind job. The backend samples the fixed-grid maximum wind against the five coastal-province exposure/grid datasets, applies the vulnerability engine and the default 17.3 m/s reporting threshold, and generates grid/city/province summaries.

The current UI intentionally uses the just-completed wind job. Advanced API endpoints also support a compatible fixed-grid NetCDF or a `wind_input_for_loss.zip`, but repository-generated products are recommended because case identity and track matching are validated.

### Step-by-step reproduction of NOUL (ID2612)

The GitHub package includes the NOUL storm-track workbook named track.xlsx, but excludes the large FNL, model, scaler, and exposure files.

1. **Install the environment.** Use 64-bit CPython 3.10. Run setup_windows.bat, then setup_full_windows.bat, and wait for setup to finish. If setup_full_windows.bat displays “Press any key to continue,” installation exited with an error; read the console message above it.
2. **Prepare the case.** Create runtime/cases/NOUL_ID2612/ with config.yml, track.xlsx, and an FNL subfolder. The supplied track workbook must match NOUL (paper event ID 202612; software case ID2612) and contain time, LAT, and LONG columns. Place exactly the original 28 NetCDF files, 006.nc through 168.nc at 6-hour intervals, directly in FNL; do not add other NetCDF files.
3. **Add the scientific assets.** Place direct_unet_model.h5 under runtime/downscaling/models/, scaler_x.save and scaler_y.save under runtime/downscaling/scalers/, and the manuscript-matched five-province exposure and grid Parquet data under runtime/exposure/. These assets are not in the GitHub package.
4. **Configure external storage if needed.** Copy config.local.json.example to config.local.json and set actual asset paths. If the case is not under the configured cases_root, copy data/approved_wind_sources.example.json to data/approved_wind_sources.json and register NOUL_ID2612. Set case_dir to the parent folder containing FNL, and select that same physical FNL folder in the browser; the service checks file names, sizes, and modification times.
5. **Start and calculate.** Run verify_windows.bat and confirm engines.wind.local.ready and engines.loss.ready are both true. Double-click start_windows.bat and keep its console open. Open the local site, select the FNL folder itself and the same track.xlsx, wait for both inputs to show Verified, click Generate wind field, continue to loss calculation, and click Calculate loss.
6. Compare the result with the manuscript: landfall near Huizhou, Guangdong; city maximum wind speed about 47.01 m/s in Shanwei; total modeled loss about RMB 74.93 million (about RMB 74.49 million in Guangdong and RMB 0.44 million in Fujian).

On the Windows 11 laptop reported in the manuscript (Intel Core i7-13700H, 32 GB RAM, NVIDIA RTX A1000 Laptop GPU, Python 3.10.22), wind-field calculation took about 27 minutes and loss calculation about 2 minutes. The Windows scientific environment uses CPU TensorFlow; actual runtime varies by hardware and input data. This is an illustrative modeled result, not an independently validated observation of actual loss.


### Map rendering versus scientific data

For browser performance, the interactive payload may aggregate or downsample scientific grid cells, and the renderer interpolates colours between legend stops. Cells without a source in the payload remain transparent and are identified as having no data source during inspection. This display handling does not change the source NetCDF and is not used in statistics or loss calculations.

## Current interface and result updates

- The home page presents the TyphoonRiskChina name and includes Windy source attribution.
- Wind results show landfall details in UTC+8, distinguish land and sea, and provide a fixed-grid wind map with rankings for the five monitored coastal provinces.
- Wind and loss maps use the same storm coverage mask. The separate storm-track composite image is not shown in the results.
- Loss results separate provincial totals from city rankings, omit zero-loss locations, show currency units, and display the referenced vulnerability curve with interactive values.
- Wind-field and loss outputs can each be downloaded as a ZIP package.

## Scientific input layout

The full standard layout is:

```text
runtime/
├─ cases/
│  └─ Bavi_ID2609/
│     ├─ config.yml
│     ├─ track.xlsx
│     └─ FNL/006.nc ... 168.nc
├─ downscaling/
│  ├─ models/direct_unet_model.h5
│  └─ scalers/{scaler_x.save,scaler_y.save}
└─ exposure/
   ├─ 00000 沿海省份风险暴露/
   │  └─ {fujian,guangdong,hainan,jiangsu,zhejiang}.parquet
   └─ 00000 沿海省份网格/
      └─ one <province>_*.parquet file per province
```

For a case outside `runtime/cases`, copy `data/approved_wind_sources.example.json` to the ignored file `data/approved_wind_sources.json` and set an absolute `case_dir`. The backend never accepts an arbitrary filesystem path from an HTTP request.

## Processing chain

```text
28 × 6-hour FNL + track workbook
  -> local manifest and track validation
  -> CPU downscaling with H5 model/scalers
  -> 6-hour to 1-hour interpolation
  -> moving-window field on a fixed latitude/longitude grid
  -> temporal maximum wind NetCDF
  -> exposure/grid sampling for five coastal provinces
  -> vulnerability and threshold screening
  -> grid, city, and province loss summaries
  -> interactive JSON, PNG, CSV, NetCDF, PDF, and ZIP outputs
```

## Outputs

Wind-field jobs can generate:

- `fixed_grid_max_wind.nc`
- `fixed_grid_max_wind.png`
- `hourly_summary.csv`
- `wind_manifest.json`, `stage_manifest.json`, and calculation logs
- `interactive_wind_map.json`
- `wind_input_for_loss.zip`

Loss jobs can generate:

- `grid_wind.csv` and `grid_loss.csv`
- maximum-wind and loss-distribution PNG maps
- `typhoon_loss_assessment.pdf`
- `result_manifest.json`, stage metadata, logs, and interactive JSON
- a summary ZIP; very large grid CSV files remain available as individual downloads

All generated jobs are stored under `data/jobs/<job_id>/` and are ignored by Git.

## Lightweight reviewer API

The legacy lightweight workflow is still available through the API, although the current UI does not expose it. Its track CSV uses `time,lat,lon`; its exposure CSV uses `grid_id,province,city,wind_speed_ms,exposure_value`. See `example_folder/` and the automatic API documentation at `/docs`.

## Frontend development

```powershell
cd frontend_source
npm ci
npm run dev
```

Development mode starts Vite on port 5173 and a local API on port 8788. Stop any existing 5173/8788 process first. To build production assets:

Vite uses the repository-relative `runtime/` layout by default. `config.local.json` is a production-launcher setting and is not read by Vite; set the `TYPHOON_*` variables from `.env.example` in the same PowerShell session when developing against assets stored elsewhere.

```powershell
npm run build
```

Copy the clean `frontend_source/dist/` output to `web/` before preparing a release. Do not commit `node_modules`, `.npm-cache`, or `dist`.

## Tests

Basic tests:

```powershell
.\.venv\Scripts\python.exe -m pip install -r requirements-dev.txt
.\.venv\Scripts\python.exe -m pytest -q tests/test_web_wrapper.py tests/test_combined_server.py
```

Tests that import NumPy/Xarray or execute scientific stages must use `.venv-full`. Real-data regression tests are skipped unless `TYPHOON_TEST_VALIDATION_ROOT` points to a local validation dataset.

## Troubleshooting

- **Python/TensorFlow installation fails:** run `py -0p`; install CPython 3.10 x64. “No matching distribution” usually means the wrong Python version or architecture.
- **Port 5173 is occupied:** inspect it with `Get-NetTCPConnection -LocalPort 5173 -State Listen`. Close the known old Vite/Uvicorn window or launch `scripts/start_windows.ps1 -Port 5174`; verify that port with `scripts/verify_windows.ps1 -Port 5174`.
- **Page opens but wind/loss is unavailable:** inspect `/api/health`. Common missing items are `.venv-full`, the H5/scalers, exposure Parquet files, or a configured FNL case.
- **FNL verification fails:** Select the FNL folder itself. It must contain the exact 28 files and match the backend-readable source by name, byte size, and modification time. If it is outside the configured cases_root, register the parent case folder in data/approved_wind_sources.json; case_dir points to the folder containing FNL, not to FNL itself.
- **Map background is blank:** Windy and OpenStreetMap require Internet access and may be blocked by a firewall, ad blocker, or content policy. This does not imply the local calculation service failed.
- **A job fails:** inspect `data/jobs/<job_id>/job.json`, `calculation.log`, and `stage_manifest.json`; also check disk space and case/track identity.

## Privacy, network access, and safety

- The API binds to `127.0.0.1` and is intended for one local computer.
- The backend contains no outbound HTTP calculation client.
- The Windy iframe and OpenStreetMap tile layer are external network resources.
- Package installation connects to Python/npm registries.
- Local job folders may contain sensitive input metadata and should be managed according to the user's data policy.

The outputs are research scenarios, not operational warnings, observed loss, insurance pricing, or safety-critical advice.

## Licensing and data provenance

Repository code is provided under the included MIT license where explicitly applicable. External FNL data, model/scaler files, exposure/grid Parquet data, CMA best tracks, boundary sources, Windy, and OpenStreetMap retain their own terms. Confirm redistribution permission before publishing any scientific asset. See [DATA_CARD.md](DATA_CARD.md), [MODEL_CARD.md](MODEL_CARD.md), and [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md).

For citation metadata, see [CITATION.cff](CITATION.cff).
