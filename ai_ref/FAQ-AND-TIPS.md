---
title: LTspice FAQ and Tips
description: Updates, license, Linux, simulation speed, efficiency calculations, and SMPS Bode plots — frequently asked questions and practical tips.
version: "24+"
---

[← AI Reference](README.md)

# LTspice FAQ and Tips

Frequently asked questions, simulation speed, efficiency calculations, SMPS Bode plots, platform notes, and resources.

---

## Table of Contents

1. [Program Updates](#program-updates)
2. [License and Distribution](#license-and-distribution)
3. [Running Under Linux](#running-under-linux)
4. [Settings Overview](#settings-overview)
5. [Speeding Up Simulations](#speeding-up-simulations)
6. [Efficiency Calculation](#efficiency-calculation)
7. [SMPS Bode Plots (FRA)](#smps-bode-plots-fra)
8. [Additional Resources](#additional-resources)

---

## Program Updates

### Updating LTspice

Three methods:
1. Download latest from https://www.analog.com/ltspice
2. **Help > Check for LTspice updates**
3. Windows Start Menu shortcut

### Updating Components and Models

**Tools > Update components** downloads the latest component libraries and models.

### Change Logs

- **Help > Show LTspice Change Log** — program changes
- **Help > Show Model Change Log** — model/component updates

### Important Warnings

- Older versions cannot be recovered after update
- All library databases are overwritten automatically
- **Do not edit files in `%LOCALAPPDATA%\LTspice`** — they may be overwritten on update
- Store custom files in `Documents\LTspice\` (user library directory)

---

## License and Distribution

- **Classification**: EAR99 (no export restrictions)
- **License**: Non-exclusive, non-transferable for internal use and circuit simulation
- **Free**: No cost, no license limits
- **Restrictions**:
  - No modification, reverse engineering, or decompilation
  - Not licensed for semiconductor manufacturers for product design/promotion (requires special permission from ADI)
  - Not suitable for high-risk applications (nuclear, aircraft, life support, weapons, autonomous driving)
- **Jurisdiction**: Massachusetts, USA

### Third-Party Components

LTspice includes code from:
- Berkeley SPICE (University of California)
- NXP MEXTRAM model
- Hiroshima University HiSIM
- ZLIB compression
- SQLiteC++ database

---

## Running Under Linux

Analog Devices does not provide an official Linux version. LTspice runs under WINE.

### Installation

```bash
# Install WINE
# (from http://www.winehq.com)

# Download LTspice
wget https://LTspice.analog.com/download/latest/LTspice64.msi

# Install
wine msiexec /i LTspice64.msi
```

### Running

Launch from desktop icon or:
```bash
wine ltspice.exe
```

### Known Issues

- Font scaling less smooth than on Windows
- PWL editor display issues — fix with native Windows DLLs:
  ```bash
  wine -dll commctrl,comctl32=n ltspice.exe
  ```

---

## Settings Overview

Access via gear icon or **Tools > Settings**. Configuration sections:

| Tab | Purpose |
|-----|---------|
| Compression | Waveform data compression tolerances |
| Save Defaults | Default trace saving behavior |
| SPICE | General-purpose simulation settings |
| Schematic | Drafting options (grid, snap, pen, undo) |
| Netlist | Netlist generation options |
| Search Paths | Symbol and library paths |
| Waveforms | Waveform viewer appearance |
| Operation | Application behavior (marching waveforms, auto-delete, etc.) |
| Hacks | Internal development settings (deprecated) |
| Internet | Update behavior |

---

## Speeding Up Simulations

Pointers to the measures documented elsewhere in this reference, grouped by what each one costs
you. Nothing here is a new setting — follow the links for syntax and caveats.

### Reduce the work the simulator does

These change how much gets simulated, not how accurately.

- **Skip the startup transient with `loadstate`.** Save the settled state once, then start every
  later run from it. For iterative work — tuning a compensation network, sweeping a component
  value, repeating an FRA sweep — this is usually the largest single saving available: the startup
  transient is often most of the run, and it is otherwise re-simulated from scratch every time.
  Component and parameter values may differ from those the state was saved with, and the run will
  usually still converge — see
  [Changing the Circuit Between Save and Load](SIMULATION-COMMANDS-REFERENCE.md#changing-the-circuit-between-save-and-load).
  See also [State Management](SIMULATION-COMMANDS-REFERENCE.md#state-management), and
  [Step 1](#step-1-verify-basic-operation) for priming an `.fra` sweep this way.
- **Stop the run when the circuit settles**, using the `steady` modifier rather than guessing a
  `Tstop` and overshooting it. See
  [Steady-State Detection](TROUBLESHOOTING-GUIDE.md#steady-state-detection).
- **Prefer LTspice's native devices and SMPS macromodels** over generic SPICE/PSpice equivalents —
  the native models exist largely for simulation speed. See
  [Speed Considerations](DEVICE-MODELS-GUIDE.md#speed-considerations).

### Trade accuracy for speed — validate the setup first

Each of these can change the answer. Establish a result you trust, then apply them and confirm the
answer did not move.

- **FRA sweeps**: reduce `tsettle` and `tavgmin` where the circuit responds quickly, use `fcoarse`
  to coarsen the expensive low-frequency end, and only then consider raising `nmax`. See
  [Step 8: Speed Up](#step-8-speed-up-optional) and
  [`nmax`](CIRCUIT-ELEMENTS-REFERENCE.md#--frequency-response-analyzer), which puts the circuit's
  own distortion on the frequencies being measured.
- **Relaxed `reltol`** gets past a difficult startup quickly. The sound way to use it is to relax
  tolerance for the startup only, then continue at tight tolerance from a saved state — see
  [State Save/Load for Partial Relaxation](TROUBLESHOOTING-GUIDE.md#state-saveload-for-partial-relaxation).
- **`method=gear`** cures trap ringing but needs more timesteps for the same accuracy, and can make
  an unstable circuit appear stable. See
  [Integration Methods](TROUBLESHOOTING-GUIDE.md#integration-methods).
- **Waveform compression** is lossy. See
  [Waveform Compression](TROUBLESHOOTING-GUIDE.md#waveform-compression).

### Reduce disk and memory rather than run time

Listed so they are not mistaken for run-time savings — the circuit is still simulated in full, only
less of the result is kept.

- **`.save`** limits which traces are written
  ([.SAVE](SIMULATION-COMMANDS-REFERENCE.md#save--limit-saved-data)); unchecking the subcircuit save
  defaults does the same for internal node voltages and device currents.
- **`Tstart`** discards data before a given time, but still simulates that time.
- For both, see [Reducing Data Size](TROUBLESHOOTING-GUIDE.md#reducing-data-size) and
  [Memory and Performance](TROUBLESHOOTING-GUIDE.md#memory-and-performance).

Turning marching waveforms off is *not* worth doing for performance — its effect on both run time
and memory is negligible.

### Speeds viewing, not simulating

- **Fast Access conversion** reorganizes a finished `.raw` file so the waveform viewer can plot from
  it quickly — roughly a factor equal to the number of saved traces. It does not make the
  simulation any faster, and the conversion itself can take longer than the original run. See
  [Fast Access File Format](WAVEFORM-VIEWER-GUIDE.md#fast-access-file-format).

---

## Efficiency Calculation

### Quick Method

Add `steady` keyword to `.tran` command:
```spice
.tran 10m steady
```

### Requirements

- Exactly **one voltage source** (identifies as input)
- Exactly **one current source** or **Rload resistor** (identifies as output)

### How It Works

1. Simulation runs until steady state is detected (via switching regulator macromodel)
2. Energy stored in reactances noted at clock edges
3. Efficiency = output power / input power, adjusted for reactance energy changes
4. Report displayed as comment block on schematic

### Viewing Results

**View > Efficiency Report** after simulation completes.

### Manual Adjustment

If automatic detection is too aggressive or not critical enough:
```spice
.options sstol=0.0001      ; tighter steady-state tolerance
.options ststdelay=100u    ; wait before starting detection
```

### Checking Individual Losses

After efficiency report, check individual device power dissipation by Alt+clicking on component bodies in the schematic.

### Detailed Efficiency Analysis

For more precise efficiency estimates including switching and parasitic losses, use **LTpowerCAD**: https://www.analog.com/en/design-center/ltpowercad.html

---

## SMPS Bode Plots (FRA)

Frequency Response Analysis for measuring loop gain and phase margin of switched-mode power supplies.

### Concept

Uses Middlebrook's method (voltage injection) to measure the loop transfer function of a feedback system in the time domain. A sinusoidal stimulus is injected at the feedback loop break point, and Fourier analysis extracts gain and phase at each frequency.

### Example Circuits

**File > Open Examples > Educational\FRA\**

### Procedure

#### Step 1: Verify Basic Operation
Run a standard `.tran` simulation to confirm the SMPS starts up and reaches steady state.

Save the settled state while you are here. Steps 3-8 re-run the circuit many times, and every run
otherwise repeats the whole startup transient:

```spice
.tran 2m savestate ; verify startup, save the state at the end of the simulation
```

Then start the FRA runs from it:

```spice
.fra loadstate
```

The FRA device's `delay` is absolute simulation time measured from t=0, not time elapsed since the
run began, so a state saved at or after `delay` has already satisfied it: the state above is saved
at t=2m, so an FRA device with `delay=1m` starts injecting immediately when that state loads. Leave
`delay` at the value the un-primed circuit genuinely needs to settle — it costs no run time once the
state is loaded past it. (Use `savestatetime=<time>` if you need the state captured at a particular
time rather than at the end of the run.) State files are interchangeable between `.tran` and `.fra`; see
[State Management](SIMULATION-COMMANDS-REFERENCE.md#state-management).

#### Step 2: Insert FRA Component
Break the feedback loop and insert the FRA device (prefix `@`) at the injection point. Valid placement must satisfy two criteria:

1. **The device must completely interrupt the feedback.** Every feedback path has to pass through it. If any path bypasses the device — a feedforward capacitor around the divider, a second sense connection, an auxiliary loop — part of the loop stays closed and what you measure is not the loop gain.
2. **The device must point from lower impedance to higher impedance.** Connect the **OUT** terminal to the low-impedance side of the break and the **IN** terminal to the high-impedance side — typically OUT toward the converter output and IN toward the feedback divider or error-amplifier input. If the source impedance at the break is not small compared with the load impedance, the injection loads the loop and the measured response is corrupted.

#### Step 3: Initial Exploratory FRA
Start with 2-3 frequencies to verify the setup works — use `flist` on the FRA device to apply just those frequencies individually:
```spice
.fra
```

Key FRA device parameters (see [CIRCUIT-ELEMENTS-REFERENCE.md](CIRCUIT-ELEMENTS-REFERENCE.md#--frequency-response-analyzer) for the full list):
- **fstart**: Starting frequency (typically 100Hz-1kHz for SMPS)
- **fend**: Ending frequency (typically 100kHz-1MHz)
- **tsettle**: Settling time at each frequency before analysis begins. Start at `2/fcross`, where fcross is the expected 0dB crossover. Defaults to `10/fend`.
- **tavgmin**: Minimum analysis time per frequency. For an SMPS, start at `100/fsw`, where fsw is the switching frequency.

#### Step 4: Check for Nonlinearity
Inspect FRA transient waveforms — stimulus should be small enough to not disturb the operating point significantly.

#### Step 5: Initial Bode Plot
Analyze the gain/phase plot. Identify the 0dB crossover frequency (f0dB).

#### Step 6: Adjust Stimulus Amplitude
Use the frequency-dependent amplitude parameters (`pp0`, `pp1`, `f0`, `f1` — the recommended method) to inject a larger stimulus at low frequencies and a smaller one at high frequencies. Where loop gain is high, the loop suppresses the injected perturbation, so a larger stimulus is needed to get a measurable response; near and above crossover the loop no longer attenuates it, so the same amplitude would disturb the operating point and distort the result.

```spice
* 2mV up to 1kHz, tapering to 1mV above 2kHz
@1 A B fstart=1k fend=500k pp0=2m pp1=1m f0=1k f1=2k
```

#### Step 7: Add More Frequencies
Use 2-3 points per octave (`oct=2` or `oct=3`) for a smooth Bode plot.

#### Step 8: Speed Up (Optional)
Reduce `tsettle` and `tavgmin` where the circuit responds quickly, and use `fcoarse` (set to 2-10× `fstart`) to coarsen the sweep at low frequencies, where each point is most expensive.

Only at this point — with the setup already validated by the steps above — consider raising `nmax` above its default of 1 to inject harmonics alongside the fundamental. It trades accuracy for speed, because the circuit's own harmonic distortion then falls on the measured frequencies and cannot be separated from the real response. See [`nmax`](CIRCUIT-ELEMENTS-REFERENCE.md#--frequency-response-analyzer) before using it.

### Reference

R.D. Middlebrook, "Measurement of Loop Gain in Feedback Systems," International Journal of Electronics, 1975.

---

## Additional Resources

### Official

| Resource | URL |
|----------|-----|
| LTspice downloads & training | https://www.analog.com/ltspice |
| EngineerZone support forum | https://ez.analog.com/design-tools-and-calculators/ltspice/ |
| LTpowerCAD efficiency tool | https://www.analog.com/en/design-center/ltpowercad.html |

### Community

| Resource | URL |
|----------|-----|
| Independent users' group | https://groups.io/g/LTspice |
| Simon Bramble tutorials | http://www.simonbramble.co.uk |

The groups.io Files section contains tutorials, additional component libraries, and user-contributed example circuits.

---

*See also: [TROUBLESHOOTING-GUIDE.md](TROUBLESHOOTING-GUIDE.md) for convergence solutions, [SIMULATION-COMMANDS-REFERENCE.md](SIMULATION-COMMANDS-REFERENCE.md) for .FRA command details*
---

*Documentation source: [github.com/analogdevicesinc/ltspice-reference](https://github.com/analogdevicesinc/ltspice-reference)*
