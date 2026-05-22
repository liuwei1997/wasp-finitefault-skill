# Common Pitfalls in WASP Finite-Fault Inversions

## 1. Only 1 station used in inversion

**Symptom**: `waveforms_body.txt` lists only 1 station, `tele_waves.json` has
only 1–2 entries, despite having 466 SAC files.

**Root cause — incomplete PZ extraction**:
`Teleseismic_Data.zip` bundles SAC files and PZ response files together.
Default `unzip` may only extract a few PZ files.

**Fix**:
```bash
cd examples/data
unzip -l Teleseismic_Data.zip | grep SAC_PZ | wc -l   # expect 466
ls SAC_PZ* | wc -l                                      # check extracted count
unzip -o Teleseismic_Data.zip "SAC_PZ*"                 # fix missing
cp SAC_PZ* ../tutorial_work/<event_id>/ffm.N/data/      # copy to workdir
```

Then delete `P/`, `SH/`, `logs/` from the workdir and re-run.

**Root cause — PZ location code mismatch**:
`__remove_response_body` (`data_processing.py:267-272`) matches PZ files by
location code. If the SAC file has `loc_code="00"` but PZ filenames use `_00`,
the match may fail.

**Diagnose**:
```python
from obspy.io.sac import SACTrace
h = SACTrace.read("problematic.sac")
print(h.kstnm, h.kcmpnm, h.khole)
# Check if matching PZ file exists under expected name
```

**Root cause — SNR threshold too strict**:
`data_management.py:725`: `min_snr = 5.0` for P waves. If PZ files are correct
but few stations pass, lower the threshold:
```python
min_snr = 2.0  # was 5.0
```

But always verify PZ files first — this is rarely the actual bottleneck.

## 2. Python import errors

### cartopy / pygmt not installed

**Symptom**: `ModuleNotFoundError: No module named 'cartopy'` or `'pygmt'`

**Fix**: These are optional — wrap imports in lazy loaders:

```python
# In wasp/plot_utils.py, wasp/plot_graphic.py, wasp/plot_graphic_NEIC.py:
try:
    import cartopy.crs as ccrs
    import cartopy.feature as cfeature
except ImportError:
    ccrs = None
    cfeature = None

# In wasp/plot_graphic_NEIC.py, PlotMap class (~line 1380):
try:
    import pygmt
except ImportError:
    pygmt = None
    # Add graceful degradation to PlotMap
```

Each use of `cartopy` or `pygmt` must be guarded with `if ccrs is not None:`.

Files affected: `plot_utils.py`, `plot_graphic.py`,
`plot_graphic_NEIC.py`, `src/test/*_test.py`.

### numpy.trapz removed in NumPy 2.0

**Symptom**: `AttributeError: module 'numpy' has no attribute 'trapz'`

**Fix**: Replace `np.trapz` → `np.trapezoid`:

```python
# In plot_graphic.py and plot_graphic_NEIC.py:
# np.trapz(y, x)  →  np.trapezoid(y, x)
```

This affects 2 files.

## 3. Surface-wave GF bank missing or parameter mismatch

**Symptom**: Running `-t surf` fails with file-not-found error referencing a
binary GF bank file, OR Fortran crashes with an obscure error during GF computation.

**Root cause — GF bank file absent**: The surface-wave Green's Function bank is
~875 MB of pre-computed binary data. It is NOT distributed with the repository.

**Root cause — parameter mismatch**: The first 3 lines of
`fortran_code/gfs_nm/long/low.in` must match the internal header of the
`fd_bank` binary. If they don't match (e.g. the repo ships with default values
that differ from the binary), the Fortran run will crash during GF computation.

**Fix**:
```bash
# 1. Verify low.in matches fd_bank
head -3 fortran_code/gfs_nm/long/low.in
# Correct values (from FiniteFault_Ori):
#   9.0 21.6 5.0

# 2. If they differ, backup and fix
cp fortran_code/gfs_nm/long/low.in fortran_code/gfs_nm/long/low.in.bak
# Edit line 1-3 to: 9.0 21.6 5.0
```

**Options for missing GF bank**:
1. Use body-wave only (`-t body`) — works without GF bank
2. Compute GF bank with `gf_surf_tel` (requires the full Fortran surf code)
3. Request from USGS/NEIC

## 4. wasp_prep command not found

**Symptom**: `wasp_prep: command not found`

**Fix**: `wasp_prep` is a compiled Fortran binary. Options:
- Compile from `src/fortran/wasp_prep.f` with `gfortran`
- Use the Python CLI instead: `wasp model run <dir> auto_model -g <cmt> -t body`
  (the Python wrapper may handle data copying internally)
- Manually copy data files to the working directory

## 5. Surface-wave inversion: only 1 station

**Symptom**: Surface-wave (or joint) inversion produces only 1 station, despite
body-wave inversion using 43 stations with the same dataset.

**Root cause — PZ files not in search path**: WASP does NOT automatically
extract PZ response files from ZIP archives. During body-wave processing,
`__remove_response_body` extracts and copies PZ files to the data directory.
But surface-wave processing expects PZ files to already exist in the same
directory as the SAC data.

**Fix — use `--data-dir` to point to pre-populated PZ directory**:
```bash
# Step 1: Run body-wave first (this populates PZ files in the data dir)
wasp model run ffm_body auto_model -g <cmt> -t body

# Step 2: Point surf/joint to the same data dir
wasp model run ffm_joint auto_model \
  --data-type body --data-type surf \
  -g <cmt> --data-dir ffm_body/data
```

> **Surface-wave data preprocessing constraints:**
> - Epicentral distance window: 31°–89° (same as body wave)
> - Start time constraint: `starttime < origin + 20min` (strict)
> - Both constraints are hard filters; stations outside these windows are
>   silently dropped before PZ matching even begins.

## 6. Fortran compilation errors

**Symptom**: `gfortran: error: unrecognized command line option '-fopenmp'`

**Fix**: Install `gfortran` with OpenMP support:
```bash
# Ubuntu/Debian
sudo apt install gfortran libgomp1
# macOS
brew install gcc  # provides gfortran with OpenMP
```

**Symptom**: `undefined reference to ...` during linking

**Fix**: Ensure all `.o` files are linked and order matters:
```bash
gfortran -fopenmp gf_bank_tel.o lib_ffm.o -o gf_bank_tel
```

## 7. PlotMap missing pygmt

**Symptom**: `NameError: name 'pygmt' is not defined` in `plot_graphic_NEIC.py`

**Fix** (~line 1380, `PlotMap` class):
```python
from importlib import import_module
if import_module('pygmt', package=None):
    import pygmt
else:
    print("INFO - pygmt not installed, skipping map plot")
    return  # graceful exit
```

## 8. config.ini paths

**Symptom**: WASP can't find data files.

**Fix**: Edit `config.ini`:
```ini
[PATHS]
default_data_dir = /absolute/path/to/examples/data

[LITHO1.0]
path = /absolute/path/to/examples/data/LITHO1.0.nc
```

Use absolute paths if relative paths don't resolve correctly.

## 9. Fortran character(100) path overflow — "Cannot open file …/20"

**Symptom**: Fortran binaries crash with:
```
At line XX of file retrieve_gf.f95 (unit = 1)
Fortran runtime error: Cannot open file '.../ffm_X/20': No such file or directory
```
The path is truncated at exactly 100 characters (the `20` is the start of
`20150916225432/...`).

**Root cause**: Multiple Fortran source files declare
`character(len=100)` for path variables. When the working directory exceeds
100 characters, paths are silently truncated. This affects:

| File | Line(s) | Variable(s) |
|------|---------|-------------|
| `src_dc_f95/green_bank_fk_openmp.f95` | 23 | `directory`, `gf_file`, `vel_model`, `gf_bank` |
| `src_dc_f95/retrieve_gf.f95` | 27-28 | `gf_file`, `vel_model`, `gf_bank` |
| `src_dc_f95/vel_model_data.f95` | 22 | `vel_model` |
| `src_dc_f95/gf_static.f95` | 16 | `input` |
| `bin_str_f95/get_strong_motion.f95` | 17 | `gf_file`, `gf_bank`, `vel_model`, etc. |
| `bin_str_f95/retrieve_gf.f95` | 25-26, 47 | `gf_file`, `vel_model`, `gf_bank` |
| `bin_str_f95/vel_model_data.f95` | 22 | `vel_model` |
| `bin_str_f95/store_gf.f95` | 84 | `filter_file`, `wave_file`, etc. |

**Fix**: Replace all `character(len=100)` with `character(len=500)` and
recompile:
```bash
cd fortran_code/src_dc_f95
sed -i 's/character(len=100)/character(len=500)/g' *.f95
make clean && make green_bank_openmp && make gf_static
cd ../bin_str_f95
sed -i 's/character(len=100)/character(len=500)/g' *.f95
make get_strong_motion
```

> **Note**: The bundled notebook will not hit this bug because its working
> directory path (~75 chars) fits within 100 characters.

## Diagnostic commands summary

```bash
# Count PZ files in workdir
ls SAC_PZ* | wc -l

# Count stations in inversion
grep "^name:" ffm.N/waveforms_body.txt | sort -u | wc -l

# Count BHZ files processed
ls ffm.N/data/P/*_BHZ.sac | wc -l

# Check what channels a station has
python3 -c "
from obspy.io.sac import SACTrace
import glob
for f in sorted(glob.glob('*BH*sac')):
    h = SACTrace.read(f)
    if h.kstnm == 'CCD': print(f'{f}: {h.kcmpnm} khole={h.khole}')
"
```
