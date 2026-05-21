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

## 3. Surface-wave GF bank missing

**Symptom**: Running `-t surf` fails with file-not-found error referencing a
binary GF bank file.

**Root cause**: The surface-wave Green's Function bank is ~875 MB of
pre-computed binary data. It is NOT distributed with the repository.

**Options**:
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

## 5. Fortran compilation errors

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

## 6. PlotMap missing pygmt

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

## 7. config.ini paths

**Symptom**: WASP can't find data files.

**Fix**: Edit `config.ini`:
```ini
[PATHS]
default_data_dir = /absolute/path/to/examples/data

[LITHO1.0]
path = /absolute/path/to/examples/data/LITHO1.0.nc
```

Use absolute paths if relative paths don't resolve correctly.

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
