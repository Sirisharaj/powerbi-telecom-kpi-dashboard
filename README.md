# 📊 Telecom Simulator KPI Summary Dashboard (Power BI)

An enterprise-grade self-service Business Intelligence project built in **Power BI Desktop** to track RF simulator performance across builds, frequency bands, and technologies (5G NR, LTE, NB-IoT), eliminating ad-hoc engineering interruptions and standardizing core telecom KPI definitions.

## 1. Business Problem & Objective

* **The Problem:** In a cellular simulator lab, performance numbers (Throughput, BLER, Attach Success, Latency) were locked inside isolated per-run export logs. Engineering managers and support staff constantly interrupted QA engineers asking for build metrics (e.g., *"What was 5G DL throughput on build v4.2.0?"*), costing 10+ engineering hours weekly. Furthermore, teams calculated Attach Success inconsistently by averaging percentage columns directly.

* **The Objective:**

  1. Standardize core telecom KPI definitions using mathematically rigorous **DAX measures** (e.g., calculating weighted attach success instead of an unweighted average of percentages).

  2. Implement a **Star Schema** data model connecting simulator fact runs to a dynamic calendar dimension.

  3. Deliver an interactive **2-Page User Experience**: a high-level **Summary Dashboard** for decision makers and a granular **Diagnostic Detail Page** with drill-through navigation for test engineers.

## 2. Repository Structure

```
telecom-kpi-summary-dashboard/
├── README.md
├── data/
│   └── simulator_kpi_runs.csv               <- 752 raw simulator runs (4 builds, 16 days, 8 bands)
├── reports/
│   ├── Telecom_Simulator_KPI_Dashboard.pbix  <- Power BI Desktop source file with data model & visuals
│   └── Telecom_Simulator_KPI_Dashboard.pdf   <- Exported 2-page executive presentation PDF
└── docs/
    └── DAX_Data_Dictionary.md               <- Complete measure documentation & formula definitions

```

## 3. Data Model Architecture (Star Schema)

The dashboard is built on a clean dimensional model avoiding flat-table performance bottlenecks:

```
┌────────────────────────┐
│     Dim_Calendar       │
│  (DateKey, Date, Day)  │
└───────────┬────────────┘
            │ 1
            │
            │ *
┌───────────┴────────────────────────────────────────┐
│               simulator_kpi_runs                   │
│  (run_id, run_time, build, technology, band,       │
│   test_case, dl_throughput_mbps, ul_mbps,          │
│   bler_pct, attach_attempts, attach_successes,     │
│   latency_p50_ms, latency_p95_ms)                  │
└────────────────────────────────────────────────────┘

```

## 4. Production DAX Measure Library

All measures are centralized inside a dedicated `_Measures` table:

### 1. Total Simulator Runs

```
Total Runs = COUNTROWS(simulator_kpi_runs)

```

### 2. Average Downlink Throughput (Mbps)

```
Avg DL Throughput (Mbps) = AVERAGE(simulator_kpi_runs[dl_throughput_mbps])

```

### 3. Average Block Error Rate - BLER (%)

*Lower is better. Represents RF channel block error corruption.*

```
Avg BLER (%) = DIVIDE(AVERAGE(simulator_kpi_runs[bler_pct]), 100, 0)

```

### 4. Attach Success Rate (%) — The Weighted Average Solution

*Crucial Data Quality Fix: Averaging percentage columns directly produces mathematically invalid results. This measure calculates the true weighted ratio of total successes over attempts:*

```
Attach Success Rate (%) = 
DIVIDE(
    SUM(simulator_kpi_runs[attach_successes]),
    SUM(simulator_kpi_runs[attach_attempts]),
    0
)

```

### 5. Headline Latency p95 (ms)

```
Latency p95 (ms) = AVERAGE(simulator_kpi_runs[latency_p95_ms])

```

## 5. 2-Page Dashboard Layout & Features

### Page 1: Manager Summary Dashboard (Decision-Maker Persona)

* **Top Filter Slicers:** Build (`v4.1.8` to `v4.2.1`), Technology (`5G NR`, `LTE`, `NB-IoT`), Band (`n78`, `n258`, `B3`, etc.), Date Range Slider.

* **4 Headline KPI Cards:** `Avg DL Throughput`, `Attach Success Rate`, `Avg BLER`, and `Latency p95`.

* **Daily Trend Line Chart:** Dual-axis visualization tracking Throughput on primary Y-axis vs. Latency on secondary Y-axis over time.

* **Clustered Bar Chart:** Throughput benchmark by `test_case`, color-coded by `technology` legend.

### Page 2: Engineer Diagnostic & Drill-Down (Engineer Persona)

* **Execution Run Grid:** Tabular display showing `run_id`, `run_time`, `build`, `technology`, `band`, `test_case`, and all KPI fields.

* **Conditional Formatting:** High BLER runs ($> 5\%$) highlighted in soft red for instant RF channel triage.

* **Interactive Drill-Through:** Right-clicking any test case on Page 1 allows engineers to drill through directly into Page 2 pre-filtered for that specific test run history, with a `<- Back` button to return.

## 6. Analytical Finding: Uncovering the Build v4.2.1 mmWave Bug

Using the interactive slicers on build `v4.2.1` and band `n258` (5G NR mmWave), the dashboard uncovered:

* **BLER Spike:** Rose from an average of `2.4%` up to `6.5%`.

* **Throughput Drop:** Downlink throughput dropped by \~35% on `NR_MIMO_4x4_downlink_tput`.

* **Business Impact:** Enabled QA engineers to identify an RF attenuator calibration issue on simulator bench `SIM-BENCH-01` before the software build was promoted to customer release testing.


## 7. Tools & Technologies

* **Power BI Desktop:** DAX, Data Modeling, Star Schema, Drill-Through Navigation, Visual Analytics.

* **Power Query (M):** ETL, Data Type Enforcement, Timestamp Normalization.

* **Git & GitHub:** Version control, project documentation, markdown reporting.
