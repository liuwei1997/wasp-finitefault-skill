# Setup: WASP Code & Environment

## Clone the repository

```bash
git clone https://github.com/usgs/neic-finitefault.git
cd neic-finitefault
```

## Fortran compilation

WASP requires four Fortran binaries for Green's Function computation:

```bash
cd src/fortran
gfortran -fopenmp -O3 -ffixed-line-length-none -c gf_bank_tel.f
gfortran -fopenmp -O3 -ffixed-line-length-none -c lib_ffm.f
gfortran -fopenmp gf_bank_tel.o lib_ffm.o -o gf_bank_tel
gfortran -fopenmp -O3 -ffixed-line-length-none -c fsp_c.f
gfortran -fopenmp fsp_c.o lib_ffm.o -o fsp_c
gfortran -fopenmp -O3 -ffixed-line-length-none -c wasp_prep.f
gfortran -fopenmp wasp_prep.o lib_ffm.o -o wasp_prep
gfortran -fopenmp -O3 -ffixed-line-length-none -c gf_surf_tel.f
gfortran -fopenmp gf_surf_tel.o lib_ffm.o -o gf_surf_tel
cd ../..
```

Move compiled binaries to a directory in `PATH` or keep them at the project
root.

## Python dependencies

```bash
pip install obspy numpy scipy matplotlib
pip install cartopy   # optional — for map plots
pip install pygmt     # optional — for map plots (heavy dependency)
pip install pytest    # optional — for tests
```

The minimum required: `obspy`, `numpy`, `scipy`, `matplotlib`.

## Verify installation

```bash
# Test Fortran binaries
./gf_bank_tel  # should print "USAGE: gf_bank_tel ..."

# Verify Python imports
python -c "from wasp import data_management, data_processing, fault_plane"
```

## Project structure after setup

```
neic-finitefault/
├── src/
│   ├── wasp/           # Python library
│   │   ├── data_management.py
│   │   ├── data_processing.py
│   │   ├── fault_plane.py
│   │   └── wasp_admin/model.py  # CLI entry point
│   └── fortran/        # Fortran sources (already compiled)
├── examples/
│   ├── data/           # Downloaded data (SAC, PZ, CMT)
│   └── tutorial_work/  # Working directories
├── gf_bank_tel         # Compiled binary
├── fsp_c               # Compiled binary
├── wasp_prep           # Compiled binary (optional in newer versions)
└── config.ini          # WASP configuration
```

## config.ini

The `config.ini` file tells WASP where to find data directories:

```ini
[PATHS]
default_data_dir = examples/data

[LITHO1.0]
path = examples/data/LITHO1.0.nc
```

Key sections:
- `[PATHS].default_data_dir` — where CMT files and downloaded data live
- `[LITHO1.0].path` — path to the LITHO1.0 velocity model (NetCDF)
