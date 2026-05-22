---
name: wasp-finitefault
description: >
  USGS/NEIC WASP finite-fault inversion workflow — set up the codebase, compile
  Fortran, install Python dependencies, prepare data (SAC, PZ instrument
  responses, Green's Function banks), and run body-wave or surface-wave
  auto_model inversions. Covers the full pipeline from `git clone` to finished
  Mw/slip-distribution plots, including common pitfalls (cartopy/pygmt lazy
  imports, NumPy 2.0 `trapz`, incomplete PZ extraction from ZIP archives, missing
  GF banks). Do NOT use for generic seismology or waveform processing — this is
  specifically the NEIC `neic-finitefault` WASP toolchain.
version: 1.1.0
tags: [python, fortran, seismology, finite-fault, workflow, joint-inversion]
---

# WASP Finite-Fault Inversion

End-to-end workflow for running USGS/NEIC WASP (Wavelet And Simulated-annealing
Processor) finite-fault inversions. Covers code setup, data preparation, running
inversions, and common failure modes.

> **Reference-class skill.** No executable scripts — this is procedural
> knowledge. Read the relevant reference file for your current task.

## When this applies

- Setting up the `neic-finitefault` repository for the first time
- Running a body-wave (`-t body`), surface-wave (`-t surf`), or joint (`-t body -t surf`) auto_model inversion
- Debugging: "only 1 station was used," "PZ file not found," "GF bank missing," "low.in parameters don't match fd_bank"
- Fixing Python import errors (cartopy, pygmt, NumPy) in WASP plotting code

Do NOT use this for:
- General SAC/ObsPy waveform processing
- Writing new Fortran forward-modelling code
- Non-USGS finite-fault codes (e.g. CPS, mudpy, Pyrocko-GF)

## Decision tree

| I need to… | Read |
|---|---|
| Set up the code / compile / install deps | `reference/setup.md` |
| Prepare data (SAC, PZ files, CMT, GF banks) | `reference/data-prep.md` |
| Run body-wave inversion | `reference/running.md` |
| Run surface-wave or joint inversion | `reference/running.md#surface-wave--joint-inversion` |
| Fix errors / understand common pitfalls | `reference/pitfalls.md` |

## What to expect

A successful body-wave inversion produces:
- `ffm.N/` directory with `waveforms_body.txt`, `tele_waves.json`
- `NP1/` and `NP2/` subdirectories with slip-distribution plots
- Output: Mw, moment-rate function, slip distribution, waveform fits
- Typical runtime: 30–300 s depending on station count and iteration budget

## Constraints

- **Fortran compiler required** — `gfortran` with OpenMP (`-fopenmp`)
- **Python ≥3.9** — ObsPy, NumPy, SciPy, matplotlib; cartopy/pygmt optional
- **Green's Function banks are large** — surface-wave GF bank ~875 MB, not
  distributed with the repo
- **PZ files must be extracted carefully** — the ZIP archive often gets only
  partially extracted
- **SNR threshold is aggressive** — `data_management.py:725` uses `min_snr=5.0`
  for P waves; if few stations pass, check PZ files first before lowering this
