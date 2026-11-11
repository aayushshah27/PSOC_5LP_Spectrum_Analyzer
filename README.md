# PSOC_5LP_Spectrum_Analyzer
# PSoC-5LP 128-point fixed-point spectrum analyzer

This package contains the firmware sources, a Windows-friendly Python PC application,
configurable input LPF, numerical tests, and hardware setup/measurement instructions.
Target: CY8CKIT-059, CY8C5888LTI-LP097, PSoC Creator 4.4 / ARM GCC.

## Read this first

**This is an implementation/source package, not a generated PSoC Creator workspace.**
There is no `.cywrk`, `.cyprj`, `.cysch`, generated peripheral source, or flashable HEX.
Create the schematic once using [docs/CREATOR_SETUP.md](docs/CREATOR_SETUP.md), then add
these source files and build in Creator. Creator and the physical board were not
available during preparation. Peripheral integration, actual sampling rate, analog
accuracy, and achievable performance still require testing on your board. Do not
submit this package as a measured, completed hardware demonstration.

The portable C DSP was compiled and tested on a host computer. See
[tests/results/summary.json](tests/results/summary.json) for exact scope and results.
It is not a substitute for the required signal-generator demonstration.

## Start here

1. Extract the ZIP to a short Windows path, e.g. `C:\PSoC\SpectrumAnalyzer`.
2. Install Python 3.10 or later if needed. To try the app first, open a terminal in `pc` and run:
   ```text
   python -m venv .venv
   .venv\Scripts\activate
   python -m pip install -r requirements.txt
   python app.py --demo
   ```
   Demo mode is labeled **SIMULATION / PC FFT**. Its hardware performance score is
   deliberately unavailable. Normal hardware mode never computes an FFT on the PC.
3. Assemble the Creator schematic from `docs/CREATOR_SETUP.md`.
4. Add the four `.c` files from `firmware` and all five headers, including `tables.h`.
   Replace Creator's empty `main.c` with the supplied `main.c`; do not add two mains.
5. Build and program. Start at the default **32,000 samples/s**, LPF bypass, flat-top.
6. Connect a biased sine wave as described in `docs/HARDWARE_AND_DEMO.md`.
7. Run `pc/run_windows.bat`, choose the KitProg COM port, and connect.
8. Validate amplitudes and frequencies first. Then increase Fs and run the benchmark.

On Linux/macOS, use `source .venv/bin/activate` instead of the Windows activation
command. Linux may require the system's `python3-tk` package and serial-port access.

## Files

| Path | Purpose |
|---|---|
| `firmware/main.c` | Command handling, spectrum loop, on-chip measurements |
| `firmware/board.c` | ADC/DMA acquisition, UART or optional USB CDC transport |
| `firmware/config.h` | Clocks, rate limits, transport selection |
| `firmware/dsp.c`, `dsp.h`, `tables.h` | Original fixed-point FFT, windows, integer magnitude, LPF |
| `firmware/wire.c`, `wire.h` | Binary serialization and CRC32 |
| `pc/app.py` | Live graph, peak readout, settings and CSV export |
| `pc/benchmark.py` | Counter-based sustained throughput measurements |
| `docs/CREATOR_SETUP.md` | Exact schematic/component and pin recipe |
| `docs/HARDWARE_AND_DEMO.md` | Wiring, calibration, accuracy and LPF demonstrations |
| `docs/DESIGN_AND_PROTOCOL.md` | Arithmetic, buffer ownership, framing and commands |
| `docs/MEASUREMENT_SHEET.csv` | Blank hardware results table |
| `tests/validate.py` | Native compilation of actual C DSP and numerical tests |
| `tests/syntax_check.py` | C syntax check using explicitly fake peripheral declarations |
| `tests/results/` | Reproducible host results; no fabricated board measurements |

## What is implemented

- Hardware-clocked 12-bit SAR acquisition, 2-byte-per-request DMA, four cyclic buffers.
- Short block-completion ISR; FFT and all magnitude processing run in main context.
- 128-point radix-2 FFT using integers only. No CMSIS/library-version dependency.
- 65 single-sided amplitude values in ADC-count Q16, including Nyquist; DC removed.
- Flat-top default, plus Hann and rectangular windows; coherent-gain correction.
- Stateful one-pole digital LPF before windowing/FFT; cutoff configured from the PC.
- CRC32-protected binary snapshots, parser resynchronization, nonblocking TX.
- On-chip valid-spectrum counter, elapsed time, DSP timing, gap/clipping/fault reporting.
- GUI and CSV export; CLI sampling-rate sweep and performance report.

## Accuracy domain and limitations

Use flat-top for isolated off-bin tone amplitude measurements. Our host validation
covers single tones between bins 6 and 58, 40–1700 ADC counts peak, and separated
pairs with at least 12 bins between tones. The included seeded tests passed 700
tone measurements, with maximum amplitude error about 0.28% and frequency error
about 0.15 bin. Those numbers describe **numerical tests**, not ADC/board accuracy.
All-bin comparisons against NumPy, sampled square/triangle harmonics, LPF gain,
settling, clipping, and packet corruption are also checked.

Flat-top broadens peaks: nearby components cannot always be resolved or measured
independently. DC and Nyquist neighborhoods need separate treatment; the GUI can
plot them but the generic off-bin amplitude guarantee does not cover them. Hann
and rectangular windows are available for resolution/coherent harmonic tests;
their tallest off-bin bin need not meet ±10%. There is no universal separation-free
multi-tone accuracy guarantee for a 128-point FFT.

The LPF is first order: configurable cutoff and visible attenuation, with a gentle
roll-off. It does not provide a sharp stopband. Settling frames are flagged and
excluded from valid-spectrum counts. Changing settings restarts acquisition and
measurement. A gap resets the LPF and restarts settling; uninterrupted LPF behavior
requires zero gaps.

ADC calibration, reference, generator loading, clock error, analog bandwidth and
aliasing must be checked in hardware. Use V peak in both reference and displayed
amplitudes; convert Vpp/2 or Vrms*sqrt(2) for sine waves. Digital filtering cannot
undo aliasing that already happened at the ADC.

## Performance: what is counted

The PSoC increments `frame` only after completing the FFT and all 65 magnitudes.
It separately increments `valid` when that frame is not settling, clipped, or
faulted. The GUI displays the conservative valid-spectra rate since the current
epoch began. The CLI uses differences of those **on-chip** counters and elapsed
times, not PC packet arrival timing. A faulted display must not be used as a score.

For contiguous non-overlapping acquisition, the upper bound is Fs/128 spectra/s.
The ADC's advertised 1 MS/s is not a claim that this full analyzer supports 1 MS/s.
The configured 500 kS/s request ceiling is an experiment limit, not a result.

The PC receives up to **20 spectrum snapshots/s**, while the PSoC can generate
many more. Snapshot selection is separate from skipped acquisition blocks. UART
at 115200, 8-N-1 carries ~7,360 payload bytes/s at 20 snapshots/s, below its nominal
11,520 bytes/s limit. `tx_dropped` counts snapshots not queued because the previous
one is still sending. `skipped` counts ADC blocks omitted from FFT processing.
Disclose snapshot selection during evaluation; ask the instructor if every
computed spectrum must instead be transferred. If required, qualify an end-to-end
rate using the USB path or a reduced generation rate and suitable transport.

Benchmark, with the GUI disconnected from the port:

```text
python benchmark.py --port COM5 --rates 32000 64000 128000 256000 --seconds 15 --output benchmark.csv
```

A continuity PASS is not an amplitude-accuracy PASS. Demonstrate both at the same
operating point. Multiply the sampling frequency and spectrum rate measured at
that operating point; do not multiply maxima from separate settings.

## Troubleshooting

| Observation | Check |
|---|---|
| No COM port | KitProg drivers/cable; use USB-A KitProg connector for default UART path |
| No frames | UART TX=P12[7], RX=P12[6], 115200 baud, correct names, programmed target |
| Fault 1 | DMA ISR cadence outside tolerance; check clocks, IRQ priority, lost DMA events |
| Fault 2 | Critical block copy exceeds 1/4 block duration; lower Fs |
| Fault 4 | DMA ring spans a 64-KiB addressing boundary; inspect linker map and relocate array |
| Fault 8 | DMA setup/allocation failed; check generated DMA config/macros and available TDs |
| Fault 16 | Raw ADC format/range differs from expected 12-bit setup; inspect generated GetResult16 |
| Fault 32 | Missing DMA blocks; check `eoc -> drq`, `nrq -> isr_dma`, trigger clock, DWT and ADC |
| Large amplitude error | Flat-top selection, ADC full scale, Vpeak vs Vpp/RMS, generator Hi-Z setting, clipping |
| Multiple bins around one tone | Window main lobe; check peak readout rather than treating every bin as a distinct tone |
| Nonzero skipped blocks | FFT/main loop cannot keep up at that Fs; lower Fs and reset measurement |
| ADC works only at low rate | Reference bypass, ADC clock/conversion budget, input source impedance |
| Crash after adding files | Do not add `tests/stubs/project.h`; Creator must generate its real project.h |

Fault codes are a bit mask; several may be combined. Reset after correcting the
cause. DMA IRQ cadence and critical-copy limits are defensive checks, not proof
that no hardware conversion was lost; scope the sample and EOC probe pins too.

## Teammate and submission handoff

After the project builds and passes the hardware checks, save it, then use
Creator's project/workspace archive feature (or Clean and zip the complete
workspace). Include `.cywrk`, `.cyprj`, `.cysch`, `.cydwr`, all custom `.c/.h` files,
this PC folder, README and your real measurement CSV/screenshots. Keep relative
paths. Have your teammate extract to a different directory and **Rebuild** before
submission. This source ZIP alone is not that final Creator workspace archive.
