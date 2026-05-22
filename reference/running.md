# Running WASP Inversions

## Quick start: body-wave auto_model

```bash
# From the project root
wasp model run examples/tutorial_work auto_model \
  -g examples/data/20003k7a_cmt_CMT \
  -t body
```

Paths must be absolute or relative to the working directory. The `-g` flag
expects the full path to the CMT file; `examples/tutorial_work` must exist.

## What auto_model does

The `auto_model -t body` pipeline (`wasp_admin/model.py`):

1. **`wasp_prep`** — copies SAC and PZ files from `data_dir` to the working
   directory under `ffm.N/data/`
   - Creates `ffm.0/` on first run, increments to `ffm.1/`, `ffm.2/`, etc.
   - New runs create NEW ffm directories; old results are preserved

2. **`select_process_tele_body`** — pre-selects and processes body waves
   - Pre-selection (`__pre_select_tele`): distance 31°–89°, time window <20min,
     location code must be `00` or empty
   - Arrival picking (`__picker`): TauP predictions for P, S arrivals → sets
     SAC header fields `t1`–`t7`
   - Instrument response removal (`__remove_response_body`): matches PZ files,
     deconvolves, writes to `P/` and `SH/`
   - Rotation (`__rotation`): BH1+BH2 → radial + transverse
   - Station selection (`select_tele_stations`): SNR-based per-azimuth-bin
     selection

3. **Finite-fault inversion** — Chen-Ji's wavelet kinematic modeling
   - Simulated annealing over fault slip parameters
   - Two nodal planes (NP1, NP2) modeled independently
   - Output: slip distribution, moment rate, waveform fits

## Command-line options

| Flag | Purpose |
|------|---------|
| `-g <path>` | Path to CMT file |
| `-t body` | Body-wave only inversion |
| `-t surf` | Surface-wave only (requires GF bank) |
| `-t body -t surf` | Joint body + surface wave inversion |

## Surface-wave & joint inversion

### Prerequisites

Surface-wave inversion requires a Green's Function bank (`fd_bank`), which is
~875 MB of pre-computed binary data distributed separately from the repo.

The GF bank parameters in `fortran_code/gfs_nm/long/low.in` (first 3 lines)
**must match** the `fd_bank` binary. If they don't match, the Fortran code
will crash with an obscure error. Always verify:

```bash
# Check low.in parameters
head -3 fortran_code/gfs_nm/long/low.in
# These must match the fd_bank binary's internal header.
# Default WASP values (from FiniteFault_Ori):
#   9.0 21.6 5.0
```

### Running surface-wave only

```bash
wasp model run examples/tutorial_work/20150916225432/ffm_surf auto_model \
  -g examples/data/20003k7a_cmt_CMT \
  -t surf \
  --data-dir examples/tutorial_work/20150916225432/ffm_body/data
```

### Running joint inversion (body + surface wave)

```bash
wasp model run examples/tutorial_work/20150916225432/ffm_joint auto_model \
  --data-type body --data-type surf \
  -g examples/data/20003k7a_cmt_CMT \
  --data-dir examples/tutorial_work/20150916225432/ffm_body/data
```

> **`--data-dir` tip:** WASP does NOT auto-extract PZ files from ZIP archives.
> PZ files must exist alongside SAC data in the directory pointed to by
> `--data-dir`. Always run body-wave first (to populate PZ files), then point
> surface-wave/joint runs to the same `data/` directory.

### Expected joint inversion output

The console prints TWO independent inversion blocks (one per nodal plane).
Key metrics to watch:

- `Amount of data values: N` — body+surf combined; expect 12,000+ for global events
- `Total Mag: MwX.XX` — reported for each NP separately
- Waveform fit plots include both `body BHZ/SH` and `surf BHZ/SH`

## Adding local data

After a successful `auto_model` (joint body+surf), you can incrementally add
local data types using `manual_model_add_data`. The workflow follows
`WASP_Tutorial.ipynb` sections 4.1–4.3:

### 4.1 Strong motion

```bash
# Extract strong motion data to the data/ directory
unzip StrongMotion_Accelerometer_Data.zip -d data/

# Copy NP1 to a new working copy
cp -r NP1 NP1.1

# Add strong motion data
wasp model run NP1.1 manual_model_add_data -d data/ -t strong
```

### 4.2 High-rate GNSS (cGPS)

```bash
cp -r NP1.1 NP1.2
unzip HighRateGNSS_Data.zip -d data/
wasp model run NP1.2 manual_model_add_data -d data/ -t cgps
```

### 4.3 Static GPS offsets

```bash
cp -r NP1.2 NP1.3
cp gps_data NP1.3/
wasp manage fill-dicts NP1.3 -t gps
wasp manage update-inputs NP1.3 -t gps
wasp process greens NP1.3 -t gps

# Rerun full inversion with all data types
wasp model run NP1.3 manual_model -t body -t surf -t strong -t cgps -t gps
```

> **Note**: Static GPS uses `manual_model` (not `manual_model_add_data`)
> and requires `fill-dicts` + `update-inputs` + `process greens` beforehand.

### GPS static output

The Fortran binary writes GPS predictions to `static_synthetics.txt` in the
working directory. Format: `index name lat lon U N E` (all in cm).

## Interpreting results

Success produces:
```
ffm.N/
├── waveforms_body.txt     # Processed waveforms for FFI Fortran code
├── tele_waves.json        # Station metadata (azimuth, distance, etc.)
├── channels_body.txt      # Channel list for Fortran
├── NP1/                   # Nodal Plane 1 results
│   └── plots/
│       ├── crust_body_wave_vel_model.png
│       ├── MomentRate.png
│       ├── P_body_waves.png
│       ├── SlipDist_plane0.png
│       └── SlipTime_plane0.png
├── NP2/                   # Nodal Plane 2 results
└── data/
    ├── P/                 # Processed P-wave (BHZ) files
    │   ├── II_SHEL_BHZ.sac
    │   └── final_II_SHEL_BHZ.sac
    └── SH/                # Processed SH-wave files
```

Key metrics in console output:
- `Total Mag: MwX.XX` — moment magnitude
- `Max Slip of Solution: XXX cm` — peak slip
- `Amount of data values: N` — total data points used
- `averaged misfit error` — waveform misfit

## Diagnosing station count

```bash
# How many stations in inversion?
grep "^name:" ffm.1/waveforms_body.txt | sort -u | wc -l

# How many BHZ files processed?
ls ffm.1/data/P/*_BHZ.sac | wc -l

# Station list from tele_waves.json
python3 -c "
import json
with open('ffm.1/tele_waves.json') as f:
    data = json.load(f)
print(f'Stations: {len(set(d[\"name\"] for d in data))}')
for d in data:
    print(f'  {d[\"name\"]}  az={d[\"azimuth\"]:.1f}  dist={d[\"distance\"]:.1f}')
"
```
