# Data Preparation for WASP

## Data sources

WASP needs:
1. **CMT file** — moment tensor parameters (source: USGS, GCMT, or IRIS)
2. **SAC waveform files** — teleseismic body waves (3-component: BHZ, BHN, BHE
   or BHZ, BH1, BH2)
3. **SACPZ instrument response files** — pole-zero files for each
   station/channel
4. **LITHO1.0.nc** — 3D velocity model (download from IRIS EMC)
5. **Green's Function banks** (optional — needed only for surface waves)

## Downloading CMT from USGS

Get the CMT solution from the USGS event page. Example for 2015 Illapel:

```bash
curl "https://earthquake.usgs.gov/earthquakes/eventpage/us20003k7a/moment-tensor" \
  -o examples/data/20003k7a_cmt_CMT
```

The CMT file format is plain text with:
- Event datetime, lat, lon, depth
- Moment tensor components (Mrr, Mtt, Mpp, Mrt, Mrp, Mtp)
- Nodal plane information

## Downloading waveform data

WASP expects SAC files from ScienceBase (USGS data repository). Example:
`Teleseismic_Data.zip` containing ~466 SAC files for ~100 stations, plus
matching SACPZ instrument response files.

**Common pitfall**: `Teleseismic_Data.zip` contains 466 SACPZ files but only
a few get extracted by default `unzip`. Always verify:

```bash
# Count PZ files in archive vs extracted
unzip -l Teleseismic_Data.zip | grep SAC_PZ | wc -l    # should be 466
ls SAC_PZ* | wc -l                                       # count extracted

# If mismatch, extract all PZ files
unzip -o Teleseismic_Data.zip "SAC_PZ*"
```

## PZ File Matching

WASP matches PZ files to SAC files using station name, channel, and location
code (`data_processing.py:265-272`):

```python
pz_files0 = [resp for resp in response_files if name in resp]
pz_files0 = [resp for resp in pz_files0 if channel in resp]
if loc_code == "00":
    pz_files = [response for response in pz_files0 if "00" in response]
```

**Common failure**: If PZ files weren't extracted, only 1 station (the one
with any PZ files) passes. The symptom is `waveforms_body.txt` listing only
1 station.

PZ filenames follow the pattern: `SAC_PZs_<NET>_<STA>_<CHAN>_<LOC>`.

## LITHO1.0 velocity model

Download from IRIS EMC:

```bash
wget http://ds.iris.edu/files/products/emc/emc-files/LITHO1.0.nc
```

Place at `examples/data/LITHO1.0.nc` and point `config.ini` to it.

## Green's Function banks

### Body waves (gf_bank_tel)

The body-wave GF bank is computed on-the-fly by `gf_bank_tel` during the
inversion. No pre-computed file needed.

### Surface waves (gf_surf_tel)

Surface-wave inversions (`-t surf`) require a pre-computed GF bank ~875 MB.
This file is NOT included in the repository.

**Options**:
1. Compute it yourself using `gf_surf_tel` (requires the full
   `neic-finitefault` source with surf GF routines)
2. Request it from USGS/NEIC
3. Skip surface waves and use body-wave only (`-t body`)

## Data directory layout after preparation

```
examples/data/
├── 20003k7a_cmt_CMT           # CMT solution
├── LITHO1.0.nc                # Velocity model
├── Teleseismic_Data.zip       # Original data archive
├── *.sac                      # 466 SAC waveform files
└── SAC_PZs_*                  # 466 instrument response files
```
