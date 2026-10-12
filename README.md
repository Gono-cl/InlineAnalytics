# Automated Inline Analytics

A modular Python framework for **automated inline calibration and validated sample measurement in continuous-flow systems**.

The project was developed as a standalone prototype for inline UV–Vis analytics using:

- **HiTec Zang pumps** controlled through **OPC Expert HTTP**
- **Spectragryph** for continuous UV–Vis spectrum acquisition and file export
- Python for:
  - automated calibration
  - inline dilution
  - replicate acquisition
  - CV-based quality control
  - linear regression
  - calibration model storage
  - real-sample concentration calculation
  - persistent result logging

The current implementation uses methylene blue as a model system, but the architecture is designed to be reusable for other inline analytical applications.

---

## Workflow

![Automated Inline Analytics Workflow](assets/automated_inline_analytics_workflow.png)

The software supports two main workflows:

1. **Run calibration**
   - Calculate stock and water flow rates
   - Set pump flow rates automatically
   - Wait for the selected condition
   - Collect 5 replicate spectra
   - Calculate mean absorbance, standard deviation, and CV
   - If CV > 5%, acquire one additional spectrum
   - Keep the latest 5 values and recalculate CV
   - Repeat until the CV criterion is satisfied or the maximum number of extra measurements is reached
   - Fit a linear calibration model
   - Save the calibration curve, raw results, and calibration model

2. **Measure real sample**
   - Load the saved calibration model
   - Read the latest UV–Vis spectra exported by Spectragryph
   - Collect 5 replicate measurements
   - Apply the same CV-based validation strategy
   - Calculate concentration from the saved calibration
   - Check whether the result lies inside the calibration range
   - Append the result to `sample_results.csv`

---

## Project structure

```text
automated-inline-analytics/
│
├── main.py
├── README.md
├── requirements.txt
├── .gitignore
│
├── CalibrationExperiment/
│   ├── __init__.py
│   └── calibration_experiment.py
│
├── DataAnalysis/
│   ├── __init__.py
│   └── data_analysis.py
│
├── PumpController/
│   ├── __init__.py
│   └── set_flow.py
│
├── SampleMeasurement/
│   ├── __init__.py
│   └── sample_measurement.py
│
├── Spectrometer/
│   ├── __init__.py
│   └── spectragryph_reader.py
│
└── assets/
    └── automated_inline_analytics_workflow.png
```

---

## Main components

### `PumpController`

Handles pump communication through the OPC Expert HTTP interface.

Main functions:

```python
set_flow(pump, flow)
stop(pump)
stop_all()
```

A non-zero value written to the pump `W1` tag starts the pump. Writing `0` stops it.

The current implementation also handles the decimal-separator requirement of the OPC interface.

---

### `SpectragryphReader`

Reads continuously exported Spectragryph CSV files.

Main functions:

```python
get_latest_file()
wait_until_file_ready()
read_spectrum(file_path)
get_absorbance(file_path, wavelength)
get_latest_absorbance(wavelength)
get_band_average(file_path, center, half_width)
```

Spectragryph performs the acquisition independently. Python reads the most recently available spectrum when a measurement is required.

---

### `CalibrationExperiment`

Coordinates the calibration experiment.

Main functions:

```python
calculate_flows(target_concentration)
calculate_statistics(absorbance_values)
run_condition(target_concentration)
run_calibration(concentrations)
```

For each calibration standard, the software:

1. Calculates stock and water flow rates
2. Writes the pump setpoints
3. Waits for the new condition
4. Collects replicate UV–Vis measurements
5. Calculates mean, SD, and CV
6. Applies the rolling CV acceptance criterion
7. Stores the accepted result

---

### `DataAnalysis`

Processes calibration results.

Main functions:

```python
linear_regression()
save_csv()
plot_calibration()
save_calibration_model()
```

The linear calibration follows:

\[
A = mC + b
\]

where:

- \(A\) = absorbance
- \(C\) = concentration
- \(m\) = calibration slope
- \(b\) = intercept

The fitted model is saved to:

```text
calibration_model.json
```

Example:

```json
{
    "slope": 0.003794,
    "intercept": 0.009579,
    "r2": 0.9926,
    "range_min": 0.0,
    "range_max": 320.0,
    "wavelength": 663.857
}
```

---

### `SampleMeasurement`

Uses a saved calibration model to determine the concentration of a real sample in flow.

Main functions:

```python
load_calibration_model()
calculate_concentration(absorbance)
check_calibration_range(concentration)
calculate_statistics(absorbance_values)
measure_sample()
save_result(result)
```

Concentration is calculated from:

\[
C = \frac{A-b}{m}
\]

The sample measurement uses the same replicate/CV validation strategy as the calibration.

Results are appended to:

```text
sample_results.csv
```

---

## Quality-control logic

The default measurement validation uses:

```text
Initial measurements: 5
CV threshold: 5%
Maximum extra measurements: 10
```

If:

\[
CV \leq 5\%
\]

the measurement is accepted.

If:

\[
CV > 5\%
\]

the program:

1. Acquires one additional spectrum
2. Removes the oldest measurement
3. Keeps the latest 5 values
4. Recalculates mean, SD, and CV
5. Repeats until the criterion is satisfied or the maximum number of additional measurements is reached

This provides a simple rolling-window quality-control strategy for inline measurements.

---

## Installation

Python 3.10+ is recommended.

Install the required packages:

```bash
pip install requests numpy pandas scipy matplotlib
```

Or install from `requirements.txt`:

```bash
pip install -r requirements.txt
```

---

## Spectragryph configuration

Spectragryph should be configured to:

- acquire spectra continuously
- save each acquisition automatically
- export as CSV
- use automatic filename incrementing

Example filename pattern:

```text
my_acquired_spectra_001.csv
my_acquired_spectra_002.csv
my_acquired_spectra_003.csv
```

The Python reader searches for:

```text
my_acquired_spectra_*.csv
```

---

## Running the program

Start the application with:

```bash
python main.py
```

The current command-line menu is:

```text
AUTOMATED INLINE UV-VIS SYSTEM

1 - Run calibration
2 - Measure real sample
3 - Exit
```

### Option 1 — Run calibration

Runs the full automated calibration and generates:

```text
calibration_results.csv
calibration_curve.png
calibration_model.json
```

### Option 2 — Measure real sample

Loads an existing `calibration_model.json`, performs replicate-based sample validation, calculates the concentration, checks the calibration range, and appends the result to:

```text
sample_results.csv
```

---

## Configuration

The current prototype contains system-specific settings in `main.py`, including:

- OPC Expert host
- OPC area
- pump names
- Spectragryph export folder
- stock concentration
- total flow
- calibration concentrations
- target wavelength
- replicate settings
- CV threshold

These should be adapted to the target setup before use.

For a public or multi-system deployment, moving these values to a separate configuration file is recommended.

---

## Generated files

The software can generate:

```text
calibration_results.csv
calibration_curve.png
calibration_model.json
sample_results.csv
```

These files contain experimental results and are normally excluded from version control.

---

## Current status

The standalone prototype currently supports:

- automated inline calibration
- automatic pump-based dilution
- continuous Spectragryph file reading
- replicate measurements
- mean / SD / CV calculation
- rolling CV validation
- linear regression
- calibration model storage
- real-sample concentration calculation
- calibration-range checking
- persistent sample result logging

---

## Planned improvements

Potential future developments include:

- automated waiting-time calculation based on flow and system volume
- automatic sample dilution when concentration is above the calibration range
- improved steady-state detection
- calibration validity checks
- user-selectable calibration models
- richer metadata logging
- graphical user interface
- integration into larger autonomous flow-control software
- support for additional inline analytical techniques

---

## Notes

This repository is intended as a reusable research software framework for automated inline analytics in continuous-flow experiments.

Hardware communication, file paths, analytical wavelengths, calibration ranges, and instrument-specific settings must be adapted to the local experimental setup.

---

## License

A license has not yet been selected.

Before public release, add an appropriate open-source license depending on the intended use and institutional requirements.
