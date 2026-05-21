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
