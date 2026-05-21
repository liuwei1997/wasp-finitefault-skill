# wasp-finitefault — WASP Finite-Fault Inversion Skill

USGS/NEIC WASP finite-fault inversion workflow — set up the codebase, compile
Fortran, install Python dependencies, prepare data (SAC, PZ instrument
responses, Green's Function banks), and run body-wave or surface-wave
auto_model inversions.

## What this covers

- **Setup**: Fortran compilation, Python dependencies, environment config
- **Data preparation**: SAC waveforms, SACPZ instrument responses, CMT files,
  LITHO1.0, Green's Function banks
- **Running inversions**: `auto_model` for body waves and surface waves
- **Common pitfalls**: PZ file extraction, cartopy/pygmt imports, NumPy 2.0
  compatibility, GF bank missing, SNR thresholds

## Files

```
wasp-finitefault/
├── SKILL.md                  # Decision tree — agent-facing routing hub
└── reference/
    ├── setup.md              # Code compilation & Python deps
    ├── data-prep.md          # Data sources & preparation
    ├── running.md            # Running inversions & diagnosis
    └── pitfalls.md           # 7 common failure modes & fixes
```

## Target audience

LingTai agents working with the USGS/NEIC `neic-finitefault` finite-fault
inversion code (WASP — Wavelet And Simulated-annealing Processor).

Not intended for: general seismology, other finite-fault codes (CPS, mudpy,
Pyrocko-GF), or waveform processing unrelated to WASP.

## Example use

An agent presented with "run the 2015 Illapel finite-fault inversion" would:
1. Load `SKILL.md` → decision tree sends them to `reference/setup.md`
2. Follow Fortran compilation steps
3. `reference/data-prep.md` → check PZ files, extract ZIP properly
4. `reference/running.md` → run `wasp model run ... auto_model -t body`
5. `reference/pitfalls.md` → if only 1 station, fix PZ extraction
