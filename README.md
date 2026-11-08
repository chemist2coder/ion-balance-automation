# Ion Chromatography Data Processing Automation (Ion Balance Automation)

*Русская версия — [README.ru.md](README.ru.md).*

## About the project
This repository contains scripts for processing ion chromatography data and calculating the ion balance of natural waters of any type for 9 anions and 5 cations (the ion list can easily be extended or shortened inside the script). A preliminary export of the instrument data in xlsx format is required (anonymised example data are provided).

**The problem:**
For 14 ion types, chromatography is run at standard dilutions of 2/5/10/25/50/100/500× and signal gains of 0.1 and 0.01. The maximum number of chromatograms per sample can reach 16 variants (including the mandatory measurement of two native samples for each gain). Even with partial data, the total number of concentration combinations in meq/L for building an ion balance can reach **50–100 combinations per sample**. Done manually, the ion balance for dozens of samples (accounting for dilutions and conductivity-detector overloads) takes days and often suffers from arithmetic errors by laboratory analysts.

**The solution:**
An ETL pipeline in Python:
- `01_convert_data.py` — cleaning of "dirty" Excel files and flattening them (transposing) into a tidy table.
- `02_balance_calc.py` — conversion of concentrations from mg/L to meq/L for the ion balance. Iteration over dilution combinations (~25,000 variants in total across all samples) to find the ideal balance for each sample. When cations are missing, the balance automatically computes Na "by difference" according to the standard hydrochemistry scheme, always marking the value as calculated.
- `03_summary_report.py` — generation of the final Excel summary report, convenient for transferring results into the laboratory sample register via Excel (VLOOKUP). Additionally, the script produces a separate list of errors caused by low sample pH (humic acids, unaccounted silicates, missing bicarbonates, etc.).

## Tech stack
- Python (Pandas, NumPy, Itertools)
- Jupyter Lab
- Openpyxl

## How to run (the data-processing pipeline)
Before running, install the required libraries: `conda env create -f environment.yml`. The processing is deliberately split into 3 sequential stages for code transparency and for keeping constants and dependencies in separate files. Run them strictly in order, because the output of each step is the input of the next:

**Step 1. `01_convert_data.ipynb` (extraction and cleaning)**
- **Input:** the raw Excel file (`raw_data.xlsx`) with the matrix structure of the chromatograph output.
- **Action:** extracts the data, removes dashes, handles different dilutions and gains.
- **Output:** `clean_data.xlsx` (a transposed table).

**Step 2. `02_balance_calc.ipynb` (the mathematical calculation)**
- **Input:** `clean_data.xlsx`.
- **Action:** iterates over combinations, searches for ion-balance convergence, computes sodium by difference, and detects anomalies.
- **Output:** two files: `Final_Balance_*.xlsx` (successful samples) and `Errors_and_Recommendations_*.xlsx` (problematic samples). The `*` is replaced by a timestamp in the "YYYY-MV-DD_H-M-S" format.

**Step 3. `03_summary_report.ipynb` (report generation)**
- **Input:** the script automatically finds the most recent `Final_Balance_*.xlsx` in the folder, so there is no need to edit file paths by hand.
- **Action:** builds a pivot summary table and fills empty cells with the detection limits (`< 0.10`), according to RD and PNDF (Russian methodological standards).
- **Output:** the final report `Summary_Ion_Balance_*.xlsx`.

## Sample data
The `sample_data/` folder contains an anonymised `raw_data.xlsx` — a real export from an ion chromatograph in the native format the scripts work with.
* **Format:** matrix structure (ions in rows, samples in columns).
* **Data:** ion concentrations (mg/L) plus gain and dilution coefficients.
* **Purpose:** this file can be used to test `01_convert_data.ipynb`, which transposes the matrix into a flat table suitable for the balance calculation.

## Configuration and adaptation

The scripts were written for the specifics of natural-water analysis in the author's laboratory, but are easy to adapt to other tasks:

1. **The `exclusions` list:** the code defines an `exclusions` variable that by default excludes the ions `PO4`, `NO2`, `CO3`, `Br`, `NH4`. This is because in the water types studied by the author these components are either absent or below the instrument detection limit, according to 5-year statistics. If you analyse wastewater or heavily polluted water containing ions from this list, simply remove the unneeded ions from `exclusions`, or add new ions that should be excluded from the ion balance.
2. **Detection limits:** in `03_summary_report.ipynb` you can configure the `limits` dictionary with the lower calibration-curve thresholds for each ion (e.g., `< 0.01` for fluoride).
3. **Equivalent weights:** the `weights` reference dictionary at the top of the script lets you easily add or change ions if your laboratory's method list expands.
