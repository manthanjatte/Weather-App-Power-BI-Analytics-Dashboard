# Weather App Power BI — Analytics Dashboard

A production-ready Power BI report for exploring historical and near‑real‑time weather metrics (temperature, humidity, wind, precipitation) across locations, with clean data modeling, reusable DAX, parameterized what‑ifs, and performance‑tuned visuals.

---

## 🔍 Overview

This dashboard helps analysts and business users:
- Track daily/hourly weather trends for selected cities/regions.
- Compare KPIs vs. previous period / previous year.
- Analyze anomalies (heat waves, heavy rainfall, extreme humidity) using flags.
- Export curated tables for downstream reporting.

> **File:** `Weater APP POWER BI.pbix`  
> **Tech:** Power BI Desktop, Power Query (M), DAX, (optional) Power BI Service for scheduled refresh.

---

## ✨ Key Features

- **Multi-page report**: Summary, Trend Deep‑Dive, City Comparison, Anomalies, Data Dictionary.
- **Flexible filters**: Date range, granularity (Day/Hour), City, Country, Station.
- **What‑If parameters**: Temperature threshold, Rainfall threshold for anomaly detection.
- **Reusable measures**: Consistent KPIs with calculation groups-style patterns.
- **Performance**: Star schema, incremental-friendly date table, query folding preserved.
- **Export buttons** for CSV/XLSX from key tables (Power BI Service).

---

## 📁 Suggested Repo Structure

```
weather-powerbi/
├─ README.md
├─ Weater APP POWER BI.pbix
├─ assets/
│  ├─ screenshot-summary.png
│  ├─ screenshot-trends.png
│  └─ model-diagram.png
├─ data-samples/
│  ├─ weather_daily_sample.csv
│  └─ weather_hourly_sample.csv
└─ docs/
   ├─ data_dictionary.md
   └─ refresh_playbook.md
```

> Add screenshots to `/assets` and update their paths below once ready.

---

## 🗂️ Data Model (Star Schema)

**Fact tables**
- `Fact_WeatherDaily` — one row per location per day
- `Fact_WeatherHourly` — one row per location per hour (optional, can be large)

**Dimension tables**
- `Dim_Date` — marked as Date table
- `Dim_Location` — City, State, Country, Lat/Long, Elevation
- `Dim_WeatherCodes` — code → description (e.g., rain, snow, haze)

**Relationships**
- `Dim_Date[Date]` → `Fact_WeatherDaily[Date]` (1:*, single, filter → fact)
- `Dim_Date[DateTime]` → `Fact_WeatherHourly[DateTime]` (1:*, single)
- `Dim_Location[LocationKey]` → `Fact_*[LocationKey]` (1:*, single)

---

## 🔄 Data Ingestion (Power Query)

Typical steps (Power Query **M**):
1. **Source**: CSV/Excel/API → `Source` queries (keep staging separate from final).
2. **Promote headers / set types**.
3. **Normalize** field names: `snake_case` or `PascalCase`.
4. **Create keys**: `LocationKey` (hash of City|State|Country).
5. **Buffer** small dims; avoid buffering large facts.
6. **Parameterize** file paths / API keys via `Parameters` pane.
7. Ensure **Query Folding** stays on for database sources; avoid breaking steps.

**Parameters to create**
- `pDataPath` (text): folder path for CSVs
- `pDefaultCountry` (text)
- `pApiBaseUrl` (text, optional for live sources)
- `pRefreshStartDate` / `pRefreshEndDate` (date, optional windowing)

---

## 📅 Date Table (DAX)

Create a dedicated date table and mark it as Date:
```DAX
Date =
ADDCOLUMNS (
    CALENDAR ( DATE(2015,1,1), DATE(2035,12,31) ),
    "Year", YEAR ( [Date] ),
    "Month", FORMAT ( [Date], "MMM" ),
    "Month Number", MONTH ( [Date] ),
    "Quarter", "Q" & FORMAT ( [Date], "Q" ),
    "Week", WEEKNUM ( [Date], 2 ),
    "Day", DAY ( [Date] ),
    "IsWeekend", IF ( WEEKDAY([Date],2) > 5, TRUE(), FALSE() )
)
```
For hourly analysis, add a `DateTime` table or generate at query time.

---

## 📏 Core Measures (DAX)

> Adjust table/column names to match your model.

```DAX
-- Base
Total Temperature (°C) := SUM ( Fact_WeatherDaily[AvgTempC] )
Avg Temperature (°C)   := AVERAGE ( Fact_WeatherDaily[AvgTempC] )
Total Rainfall (mm)    := SUM ( Fact_WeatherDaily[RainMM] )
Avg Humidity (%)       := AVERAGE ( Fact_WeatherDaily[HumidityPct] )
Max Wind (km/h)        := MAX ( Fact_WeatherDaily[WindMaxKmh] )

-- Time intelligence
Avg Temp MTD := CALCULATE ( [Avg Temperature (°C)], DATESMTD ( Dim_Date[Date] ) )
Avg Temp YTD := CALCULATE ( [Avg Temperature (°C)], DATESYTD ( Dim_Date[Date] ) )
Rainfall MTD := CALCULATE ( [Total Rainfall (mm)], DATESMTD ( Dim_Date[Date] ) )

-- Previous period / year
Avg Temp Prev Period :=
VAR CurrPeriod = VALUES ( Dim_Date[Date] )
RETURN CALCULATE ( [Avg Temperature (°C)], DATEADD ( Dim_Date[Date], -COUNTROWS(CurrPeriod), DAY ) )

Avg Temp YoY % :=
DIVIDE ( [Avg Temperature (°C)] - CALCULATE ( [Avg Temperature (°C)], SAMEPERIODLASTYEAR ( Dim_Date[Date] ) ),
        CALCULATE ( [Avg Temperature (°C)], SAMEPERIODLASTYEAR ( Dim_Date[Date] ) ) )

-- Thresholds (What‑If parameters)
Temp Threshold (°C) := SELECTEDVALUE ( 'Temp Threshold'[Temp Threshold Value], 35 )
Rain Threshold (mm) := SELECTEDVALUE ( 'Rain Threshold'[Rain Threshold Value], 50 )

-- Flags
Heatwave Days :=
CALCULATE (
    DISTINCTCOUNT ( Fact_WeatherDaily[Date] ),
    Fact_WeatherDaily[MaxTempC] >= [Temp Threshold (°C)]
)

Heavy Rain Days :=
CALCULATE (
    DISTINCTCOUNT ( Fact_WeatherDaily[Date] ),
    Fact_WeatherDaily[RainMM] >= [Rain Threshold (mm)]
)

-- Anomaly score (simple z-score example; replace with your columns)
Temp Z-Score :=
VAR mu  = AVERAGEX ( ALL ( Dim_Date ), [Avg Temperature (°C)] )
VAR sig = STDEVX.P ( ALL ( Dim_Date ), [Avg Temperature (°C)] )
RETURN DIVIDE ( [Avg Temperature (°C)] - mu, sig )
```

---

## 📊 Report Pages (Suggested)

1. **Executive Summary**
   - Cards: Avg Temp, Total Rain, Avg Humidity, Max Wind
   - YoY/MTD trend line
   - KPI indicators (▲▼) using conditional formatting
   - Top 5 cities by rainfall

2. **Trend Deep‑Dive**
   - Line chart: Temp/Humidity vs Date (toggle Day/Hour via slicer)
   - Column chart: Rainfall by Date
   - Decomposition Tree (drivers: Location, Weather Code)

3. **City Comparison**
   - Matrix: City × KPIs
   - Scatter: Avg Temp vs Humidity (size = Rainfall)
   - Map: Filled map or Azure Maps (lat/long from `Dim_Location`)

4. **Anomalies**
   - Heatwave / Heavy Rain flags
   - What‑If slicers for thresholds
   - Table with export data

5. **Data Dictionary**
   - Definitions, units, data lineage
   - Refresh notes & caveats

---

## 🧭 Slicers & Parameters

- **Date Range** (Between)
- **Granularity**: Day / Hour (sync with page level filters)
- **City / Country / Station**
- **Thresholds**: Temperature / Rain (What‑If parameters)
- **Weather Code**

---

## ⚙️ Refresh & Deployment

- **Local**: Manual refresh in Desktop.
- **Service**: Publish to workspace → set **Gateway** (if on‑prem data) and **Scheduled Refresh**.
- If using API sources, store secrets in **Parameters** and configure in Service.
- Consider **incremental refresh** for hourly fact (partition by date).

---

## 🚀 Performance Checklist

- Use **star schema**; avoid snowflaking for large dimensions.
- Mark **Dim_Date** as Date; use it for all time intel.
- Disable **auto date/time** in file options.
- Prefer **Import** mode; consider **Dual** for small dims if using composite.
- Reduce cardinality: round/quantize continuous fields where possible.
- Limit visuals per page; avoid heavy maps on default landing page.
- Use measures over calculated columns when aggregation is needed.
- Keep M steps foldable; avoid row-by-row operations on large sources.

---

## 🧪 QA / Validation

- Reconcile daily totals across facts (hourly → daily rollup).
- Spot-check 5 random cities and 10 random dates.
- Compare against known public records (e.g., unusually high/low values).

---

## 📚 Data Dictionary (excerpt)

| Field | Table | Type | Description |
|---|---|---|---|
| Date | Dim_Date | Date | Calendar date |
| DateTime | Fact_WeatherHourly | DateTime | Timestamp at hour |
| LocationKey | Fact_* / Dim_Location | Text/Int | Surrogate key for location |
| AvgTempC | Fact_WeatherDaily | Decimal | Average daily temperature in °C |
| MaxTempC | Fact_WeatherDaily | Decimal | Maximum daily temperature in °C |
| RainMM | Fact_WeatherDaily | Decimal | Precipitation in millimeters |
| HumidityPct | Fact_WeatherDaily | Decimal | Average relative humidity (%) |
| WindMaxKmh | Fact_WeatherDaily | Decimal | Max wind speed km/h |

> Full dictionary in `docs/data_dictionary.md` (optional).

---

## 🖼️ Screenshots

- Summary: `assets/screenshot-summary.png`
- Trends: `assets/screenshot-trends.png`
- Model: `assets/model-diagram.png`

Add images and update paths in this README.

---

## ✅ How to Use

1. Open `Weater APP POWER BI.pbix` in **Power BI Desktop**.
2. Update **Parameters** (data paths, API keys).
3. Refresh preview; check model diagram.
4. Explore pages with slicers. Use **Reset to default** if needed.
5. Publish to **Power BI Service** for sharing and scheduled refresh.

---

## 🔒 Governance & Sharing

- Put report in a **workspace** with appropriate roles.
- Use **Apps** for distribution; manage **Audience** permissions.
- Enable **Sensitivity labels** if needed.
- Export data enabled only for curated tables.

---

## 📝 Versioning

- Follow Semantic Versioning for the PBIX: `vMAJOR.MINOR.PATCH`.
- Track changes in this README under **Changelog**.

### Changelog
- `v1.0.0` — Initial dashboard and README.

---

## 📜 License

Choose one (e.g., MIT). Update this section based on your preference.

---

## 🙌 Credits

Built by **Manthan Jatte** — Data Analyst & Power BI Developer.  
Skills used: SQL, Power Query (M), DAX, Power BI.

