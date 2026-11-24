# Urban Mobility and Public Transportation Telemetry Analytics | Chetumal, Mexico

### Passenger Flow Analysis, GPS Data Quality, Peak-Hour Detection, and Urban Mobility Trends with Python

<div align="center">

<img src="./mobi.jpg" alt="Urban Mobility and Public Transportation Telemetry Analytics" width="650">

*Urban Mobility Analytics — Exploring transportation demand, GPS telemetry, passenger flows, and operational patterns through Data Engineering.*

</div>

**Project Type:** Applied Data Engineering and Urban Mobility Analytics  
**Field:** Data Engineering | Data Science | Transportation Analytics | Smart Mobility  
**Programming Language:** Python  
**Environment:** Google Colab | Jupyter Notebook  
**Study Location:** Chetumal, Quintana Roo, Mexico  
**Author:** Data Engineer Luis Barahona

---

## 1. Project Overview

This project explores urban mobility and public transportation activity in Chetumal, Quintana Roo, Mexico, using large-scale telemetry records.

The notebook processes more than one million observations collected from transportation monitoring devices.

The dataset includes timestamps, GPS coordinates, vehicle and device identifiers, and passenger boarding and alighting counters.

The main objective is to transform raw transportation telemetry into structured analytical information that can help identify operational patterns and variations in recorded public transportation activity.

The analysis includes:

- Integration of multiple CSV datasets.
- Data cleaning and type normalization.
- GPS data quality assessment.
- Telemetry reporting-gap detection.
- Passenger boarding and alighting analysis.
- Vehicle and device activity aggregation.
- Peak-hour identification.
- Daily and monthly movement analysis.
- Temporal visualizations and heatmaps.
- Structured CSV report generation.

The project demonstrates how Python and data engineering techniques can support exploratory studies of urban mobility.

---

## 2. Project Objectives

The main objectives are to:

1. Integrate historical transportation telemetry files.
2. Process more than one million records using Pandas.
3. Standardize timestamps and numerical variables.
4. Identify GPS coordinates that fail the notebook's validity check.
5. Detect interruptions in telemetry reporting.
6. Calculate passenger boardings and alightings.
7. Analyze movement activity by device.
8. Identify peak hours and high-activity days.
9. Examine monthly transportation activity.
10. Generate time series and comparative visualizations.
11. Export structured datasets for additional analysis.
12. Explore opportunities for future mobility monitoring systems.

---

## 3. Study Area

**Location:** Chetumal, Quintana Roo, Mexico

Chetumal is the capital city of Quintana Roo.

The study focuses on transportation telemetry associated with the city's public mobility environment.

The analysis uses historical records supplied as CSV files.

It does not include a complete digital representation of all transportation services or routes in the city.

---

## 4. Dataset Description

### Historical Coverage

**August 2025 – February 2026**

The analyzed files contain records from seven calendar months.

August 2025 and February 2026 have incomplete monthly coverage.

### Dataset Volume

| Indicator | Recorded Value |
|:---|---:|
| Source CSV files | 13 |
| Total observations | 1,058,772 |
| Original columns | 18 |
| Main analytical environment | Google Colab |
| Data format | CSV |

### Source File Naming

The notebook uses files organized by month and date range.

Example:

```text
chetumal_agosto_2025_01-15.csv
chetumal_agosto_2025_16-31.csv
chetumal_septiembre_2025_01-15.csv
chetumal_septiembre_2025_16-30.csv
...
chetumal_febrero_2026_01-15.csv
```

The saved execution reports 13 files in total.

### Main Telemetry Variables

| Variable | Description |
|:---|:---|
| ID_Trx | Telemetry transaction identifier |
| FechaHora | Timestamp of the observation |
| Unidad | Vehicle or unit identifier |
| Dispositivo | Monitoring device identifier |
| Empresa | Operator or company identifier |
| Ruta | Route identifier, when available |
| Latitud | GPS latitude |
| Longitud | GPS longitude |
| SubidasPuerta1 | Boardings at door 1 |
| SubidasPuerta2 | Boardings at door 2 |
| BajadasPuerta1 | Alightings at door 1 |
| BajadasPuerta2 | Alightings at door 2 |

Additional cumulative counter fields are also used in experimental occupancy calculations.

### Data Interpretation

The dataset represents recorded telemetry activity.

It should not automatically be interpreted as a complete census of public transportation demand.

Coverage depends on the monitoring devices, time periods, and vehicles represented in the source files.

---

## 5. Data Engineering Workflow

### Step 1 — CSV Integration

The notebook uses Python and Pandas to locate and combine multiple source files.

```python
import glob
import os
import pandas as pd

files = sorted(glob.glob("/content/*.csv"))

dfs = []

for file in files:
    data = pd.read_csv(
        file,
        low_memory=False
    )

    data["source_file"] = os.path.basename(file)
    dfs.append(data)

df = pd.concat(dfs, ignore_index=True)
```

The `source_file` column preserves information about the origin of each record.

### Step 2 — Data Type Normalization

Vehicle, device, operator, and route identifiers are converted into string-compatible data types.

This prevents identifiers from being treated as numerical measurements.

### Step 3 — Timestamp Processing

The timestamp field is converted into a Pandas datetime representation.

```python
df["FechaHora"] = pd.to_datetime(
    df["FechaHora"],
    errors="coerce"
)
```

Additional temporal features are derived:

- Date.
- Hour.
- Month.
- Year.
- Year-month period.

These features support time-based aggregations.

### Step 4 — Numerical Data Preparation

Passenger counter fields are converted and prepared for aggregation.

Missing counter values are treated as zero in the corresponding movement calculations.

This convention supports analysis but should be distinguished from confirmed zero-passenger events.

---

## 6. GPS Data Quality Analysis

GPS coordinates are evaluated using a simple validity rule.

### Notebook Rule

```python
df["gps_valido"] = ~(
    (df["Latitud"].fillna(0) == 0)
    &
    (df["Longitud"].fillna(0) == 0)
)
```

The procedure flags records where both coordinates are zero or are treated as zero.

### Recorded Result

**48,111 records were flagged as invalid GPS coordinates.**

This represents approximately 4.54% of the dataset.

### Important Limitation

The current rule is a preliminary quality check.

It does not fully verify geographic validity.

A stronger GPS validation procedure would also consider:

- Latitude and longitude bounds.
- Whether coordinates lie within the expected operating region.
- Sudden position jumps.
- Implausible movement speeds.
- Missing or partially invalid coordinate pairs.

---

## 7. Telemetry Reporting-Gap Analysis

The notebook analyzes the time difference between consecutive observations associated with each monitoring device.

### Procedure

1. Sort records by device and timestamp.
2. Calculate the elapsed time between consecutive records.
3. Identify intervals longer than ten minutes.
4. Create a Boolean reporting-gap indicator.

### Python Implementation

```python
df = df.sort_values(
    ["Dispositivo", "FechaHora"]
)

df["dt_seg"] = (
    df.groupby("Dispositivo")["FechaHora"]
    .diff()
    .dt.total_seconds()
)

df["gap_10min"] = df["dt_seg"] > 600
```

### Recorded Result

**21,312 intervals exceeded ten minutes.**

These gaps indicate interruptions or pauses in the sequence of observed records.

They do not, by themselves, prove a hardware failure or communication outage.

Some intervals may reflect normal operating patterns or gaps in data collection.

---

## 8. Passenger Movement Calculation

The notebook combines passenger counters from two doors.

### Total Boardings

```python
df["subidas_total"] = (
    df["SubidasPuerta1"].fillna(0)
    +
    df["SubidasPuerta2"].fillna(0)
)
```

### Total Alightings

```python
df["bajadas_total"] = (
    df["BajadasPuerta1"].fillna(0)
    +
    df["BajadasPuerta2"].fillna(0)
)
```

### Total Recorded Movements

```python
df["mov"] = (
    df["subidas_total"]
    +
    df["bajadas_total"]
)
```

### Interpretation

The movement indicator represents the sum of boarding and alighting events.

It is not equivalent to the number of unique passengers.

For example, the same passenger may contribute one boarding and one alighting event during a journey.

---

## 9. Device-Level Mobility Analysis

Records are grouped by monitoring device and date.

The notebook calculates:

- Total boardings.
- Total alightings.
- Total movements.
- Number of telemetry records.

### Example

```python
daily = (
    df.groupby(
        ["Dispositivo", "dia"],
        as_index=False
    )
    .agg(
        subidas=("subidas_total", "sum"),
        bajadas=("bajadas_total", "sum"),
        mov=("mov", "sum"),
        registros=("ID_Trx", "count")
    )
)
```

This procedure helps compare recorded activity across devices.

Device identifiers should be treated as operational identifiers and are not reproduced in this README.

---

## 10. Passenger Occupancy Exploration

The notebook contains an experimental calculation based on cumulative boarding and alighting counters.

### Calculation

```python
df["acc_subidas"] = (
    df["DatoSubidaPuerta1"]
    +
    df["DatoSubidaPuerta2"]
)

df["acc_bajadas"] = (
    df["DatoBajadaPuerta1"]
    +
    df["DatoBajadaPuerta2"]
)

df["ocup_suave"] = (
    df["acc_subidas"]
    -
    df["acc_bajadas"]
)
```

The method explores how differences between cumulative counters may approximate changes in passenger occupancy.

### Limitation

This calculation is not a validated onboard occupancy estimate.

Counter resets, missed events, synchronization differences, and negative calculated values must be investigated before operational use.

---

## 11. Peak-Hour Analysis

The notebook groups transportation movements by hour.

### Methodology

```python
peak_global = (
    by_hour.groupby(
        "hora",
        as_index=False
    )["mov"]
    .sum()
    .sort_values(
        "mov",
        ascending=False
    )
)
```

### Recorded Highest-Activity Hours

| Hour | Recorded Movements |
|:---|---:|
| 13:00 | 34,863 |
| 17:00 | 34,519 |
| 18:00 | 33,521 |
| 19:00 | 31,559 |
| 16:00 | 29,556 |

### Main Observation

The highest aggregate recorded activity occurred at 13:00.

The data also shows substantial activity during late-afternoon and early-evening hours.

These figures represent totals across the recorded period, not average daily passenger demand for each hour.

---

## 12. Peak Activity by Device and Day

The notebook also identifies:

- The highest recorded activity hour for each device.
- The five highest-activity device-hour observations.
- The three highest-activity hours for each day.

These analyses support exploratory comparison between devices and dates.

### Generated Reports

```text
horas_pico_global.csv
hora_pico_por_unidad.csv
top5_horas_pico_por_unidad.csv
top3_horas_pico_por_dia.csv
```

These files contain structured results produced by the notebook.

---

## 13. Daily Time Series Analysis

The notebook aggregates recorded activity by calendar date.

The resulting dataset contains:

- Date.
- Daily boardings.
- Daily alightings.
- Daily movements.
- Number of records.

### Python Example

```python
daily_global = (
    df_ts.groupby(
        "fecha",
        as_index=False
    )
    .agg(
        subidas=("subidas_total", "sum"),
        bajadas=("bajadas_total", "sum"),
        mov=("mov", "sum"),
        registros=("ID_Trx", "count")
    )
)
```

A line chart visualizes the daily movement series.

### Highest Recorded Day

**November 1, 2025**

| Indicator | Value |
|:---|---:|
| Boardings | 3,417 |
| Alightings | 3,489 |
| Total movements | 6,906 |
| Telemetry records | 7,980 |

This was the highest movement total in the notebook's saved daily ranking.

The result reflects recorded device activity rather than a complete count of citywide passengers.

---

## 14. Monthly Transportation Activity

The notebook groups the data by calendar month.

### Saved Monthly Results

| Month | Boardings | Alightings | Movements | Observed Days |
|:---|---:|---:|---:|---:|
| Aug 2025 | 22,251 | 22,678 | 44,929 | 25 |
| Sep 2025 | 32,228 | 33,530 | 65,758 | 30 |
| Oct 2025 | 51,924 | 55,265 | 107,189 | 31 |
| Nov 2025 | 43,332 | 44,100 | 87,432 | 30 |
| Dec 2025 | 40,136 | 42,175 | 82,311 | 31 |
| Jan 2026 | 37,925 | 38,690 | 76,615 | 31 |
| Feb 2026 | 13,631 | 13,485 | 27,116 | 10 |
| **Total** | **241,427** | **249,923** | **491,350** | **188** |

### Main Findings

October 2025 has the highest recorded monthly movement total.

However, monthly totals are sensitive to the number of observed days.

For example, February 2026 contains only ten observed days.

The notebook therefore calculates average daily movements within each month.

---

## 15. Average Daily Movement by Month

| Month | Average Daily Movements |
|:---|---:|
| Aug 2025 | 1,797.16 |
| Sep 2025 | 2,191.93 |
| Oct 2025 | **3,457.71** |
| Nov 2025 | 2,914.40 |
| Dec 2025 | 2,655.19 |
| Jan 2026 | 2,471.45 |
| Feb 2026 | 2,711.60 |

### Main Finding

October 2025 records the highest average daily movement.

August 2025 records the lowest average among the displayed months.

These comparisons do not control for changes in the number of active devices, operating coverage, or data completeness.

They should therefore be interpreted as recorded activity indicators rather than standardized ridership estimates.

---

## 16. Temporal Heatmap Analysis

The notebook creates heatmaps to visualize transportation movements by hour and month.

### Analysis Dimensions

- Hour of day: 00:00–23:00.
- Calendar month.
- Recorded movements.

### Heatmap Types

**Total movement heatmap**

Displays the sum of recorded activity by hour and month.

**Normalized heatmap**

Scales each month's hourly values relative to that month's maximum.

**Average daily movement heatmap**

Compares hourly patterns using average recorded movement across observed days.

### Benefits

These visualizations help identify:

- High-activity time windows.
- Differences between months.
- Changes in recorded hourly patterns.
- Periods with limited activity.

### Generated Files

```text
heatmap_hora_mes_mov.csv
heatmap_hora_mes_mov_normalizado.csv
```

These exports allow further analysis outside Google Colab.

---

## 17. Data Visualization

The notebook includes:

- Global hourly movement line charts.
- Daily movement time series.
- Monthly movement line charts.
- Monthly movement bar charts.
- Average daily movement comparisons.
- Top-ten highest-activity day charts.
- Highlighted peak-day and peak-month charts.
- Hour-by-month heatmaps.
- Normalized temporal heatmaps.
- Average hourly activity plots.

These visualizations help transform raw telemetry into interpretable operational summaries.

---

## 18. Technologies Used

| Technology | Purpose |
|:---|:---|
| Python | Programming language |
| Pandas | Large-scale tabular data processing |
| NumPy | Numerical operations |
| Matplotlib | Statistical and temporal visualization |
| CSV | Source data and report exports |
| Datetime processing | Time-based feature engineering |
| Folium | Installed as an optional mapping dependency |
| Scikit-learn | Installed as an optional analytical dependency |
| Google Colab | Cloud-based notebook execution |
| Jupyter Notebook | Interactive analytics environment |

The notebook primarily implements Pandas-based descriptive analysis.

Although additional libraries are installed, the saved experiments do not demonstrate a trained machine learning forecasting model or a completed interactive GPS mapping workflow.

---

## 19. How to Run

### Google Colab

[Open the Original Urban Mobility Notebook](https://colab.research.google.com/drive/1PE_eVkJCoeSYPsj9qQ64b0QOVhdDKbJk)

### Install Dependencies

```bash
pip install pandas numpy matplotlib folium scikit-learn
```

### Dataset Requirements

The notebook expects transportation telemetry CSV files to be available in:

```text
/content/
```

### Execution Instructions

1. Open the notebook in Google Colab.
2. Upload the required telemetry CSV files.
3. Confirm the source files and schema.
4. Execute the CSV integration cells.
5. Review the dataset dimensions.
6. Normalize identifiers and timestamps.
7. Calculate GPS validity indicators.
8. Detect reporting gaps.
9. Calculate boardings and alightings.
10. Generate device-level aggregations.
11. Identify peak hours.
12. Build daily and monthly time series.
13. Create the visualizations and heatmaps.
14. Export the summary CSV files.

### Important Note

The notebook loads all CSV files matching:

```python
glob.glob("/content/*.csv")
```

When rerunning the analysis, the working directory should contain only the intended source CSV files.

Otherwise, previously exported summary files may be unintentionally included in the data ingestion process.

---

## 20. Important Limitations

### Dataset Coverage

The analyzed records represent observed telemetry activity, not necessarily every public transportation operation in Chetumal.

### Incomplete Months

August 2025 and February 2026 have partial observation coverage.

### GPS Validation

The current GPS validity check is limited to coordinates where both values are zero or missing.

### Reporting Gaps

Intervals longer than ten minutes do not necessarily indicate device or network failures.

### Passenger Counts

Boardings plus alightings measure recorded movement events rather than unique passengers.

### Occupancy Estimation

The cumulative-counter occupancy calculation is exploratory and requires validation.

### Missing Route Information

The saved notebook indicates that the original route field is missing throughout the analyzed dataset.

This limits route-level demand interpretation.

### Source Data Quality

Duplicate records, device coverage changes, counter resets, and inconsistent telemetry may influence aggregations.

### No Predictive Model

The notebook develops historical descriptive and temporal analyses but does not train or validate a demand forecasting model.

### Data Sensitivity

Raw telemetry may contain operational identifiers, detailed timestamps, and precise GPS locations.

These data should not be published without confirming authorization, appropriate anonymization, and applicable data-sharing restrictions.

---

## 21. Future Improvements

Potential developments include:

- Implementing a reproducible data ingestion pipeline.
- Validating complete GPS coordinate ranges.
- Detecting abnormal position jumps.
- Improving telemetry gap classification.
- Adding explicit device coverage statistics.
- Validating boarding and alighting counters.
- Investigating duplicate observations.
- Recovering reliable route identifiers.
- Integrating official route geometry or GTFS data.
- Creating device-level performance indicators.
- Developing interactive geospatial dashboards.
- Implementing demand forecasting models.
- Comparing hourly and daily forecasting approaches.
- Evaluating model accuracy with chronological validation.
- Building automated reporting and monitoring workflows.

---

## 22. Potential Applications

The methods explored in this project can support research and prototyping in:

- Urban Mobility Analytics.
- Transportation Data Engineering.
- Passenger Flow Analysis.
- Public Transit Operations.
- Fleet Telemetry Monitoring.
- Smart City Data Platforms.
- Transportation Demand Exploration.
- Geographic Data Quality.
- Business Intelligence.
- Operational Reporting.

Any production use would require validated data, broader coverage, and appropriate institutional authorization.

---

## 23. Conclusion

This project demonstrates how Python and data engineering techniques can transform large-scale transportation telemetry into structured analytical information.

The notebook integrates 13 CSV files containing 1,058,772 observations from August 2025 through February 2026.

The recorded dataset includes 491,350 boarding and alighting movements.

Through timestamp processing, GPS quality checks, device-level aggregation, peak-hour detection, and temporal visualization, the analysis identifies patterns in recorded public transportation activity in Chetumal, Quintana Roo.

October 2025 had the highest recorded average daily movement, while November 1, 2025 was the highest-activity day in the saved results.

The project provides a practical foundation for future research in urban transportation analytics, telemetry processing, operational monitoring, and intelligent mobility systems.

---

## 24. Project Resources

### Google Colab Notebook

[Open the Urban Mobility and Public Transportation Telemetry Notebook](https://colab.research.google.com/drive/1PE_eVkJCoeSYPsj9qQ64b0QOVhdDKbJk)

### Pandas Documentation

[Pandas — Python Data Analysis Library](https://pandas.pydata.org/docs/)

### Matplotlib Documentation

[Matplotlib — Data Visualization](https://matplotlib.org/stable/)

### Folium Documentation

[Folium — Python Geospatial Visualization](https://python-visualization.github.io/folium/)

---

## Author

**Data Engineer Luis Barahona**

Mexican Data Engineer and 2026 graduate of Universidad del Caribe, interested in Data Engineering, Data Science, Artificial Intelligence, urban mobility, and technology applied to real-world challenges.

---

## Contact

| Platform | Contact |
|:---|:---|
| Email | [190300395@ucaribe.edu.mx](mailto:190300395@ucaribe.edu.mx) |
| GitHub | [github.com/LuisBarahonaMX](https://github.com/LuisBarahonaMX) |
| LinkedIn | [linkedin.com/in/luis-barahona-mx](https://www.linkedin.com/in/luis-barahona-mx/) |
| Instagram | [@luisbarahonamx](https://www.instagram.com/luisbarahonamx/) |
| X | [@luisbarahonaMX](https://x.com/luisbarahonaMX) |
| Website | [SIKO AI Platform](https://sikoai.web.app/) |
