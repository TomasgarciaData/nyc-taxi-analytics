# nyc-taxi-analytics
# NYC Taxi Analytics Pipeline — Microsoft Fabric

End-to-end data engineering project built on Microsoft Fabric. Implements a medallion architecture (Bronze → Silver → Gold) processing millions of NYC yellow taxi trips with weather data integration via Open-Meteo API. Features automated pipelines, star schema dimensional modeling, and Power BI dashboards with Direct Lake connectivity.

## Architecture

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  NYC TLC API │────▶│              │────▶│              │────▶│              │
│  (Parquet)   │     │    BRONZE    │     │    SILVER    │     │     GOLD     │
│              │     │   Raw Data   │     │  Clean Data  │     │  Star Schema │
│  Open-Meteo  │────▶│              │────▶│              │────▶│              │
│  Weather API │     │              │     │              │     │              │
└──────────────┘     └──────────────┘     └──────────────┘     └──────┬───────┘
                                                                      │
                                                                ┌─────▼──────┐
                                                                │  Power BI  │
                                                                │ Dashboard  │
                                                                └────────────┘
```

## Data Sources

| Source | Description | Format |
|--------|-------------|--------|
| [NYC TLC Trip Records](https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page) | Yellow taxi trip data (Jan-Feb 2026). ~7M+ records with pickup/dropoff times, locations, fares, and payment info | Parquet |
| [NYC TLC Zone Lookup](https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page) | Reference table mapping 265 taxi zones to boroughs and service zones | CSV |
| [Open-Meteo Historical API](https://open-meteo.com/en/docs/historical-weather-api) | Daily weather data for NYC: temperature, precipitation, snowfall, wind speed. Free, no API key required | REST API (JSON) |

## Medallion Architecture

### Bronze (Raw)
- **Trips:** Parquet files ingested via Data Factory Copy Activity with parameterized URLs
- **Zones:** CSV lookup table loaded as-is
- **Weather:** JSON response from Open-Meteo API converted to DataFrame
- All data stored as Delta tables without transformations

### Silver (Clean)
- Renamed columns to snake_case
- Cast data types (timestamps, integers, doubles)
- Removed duplicates (0 found in trip data)
- Filtered invalid records:
  - Trips with zero distance AND zero/negative duration (cancelled trips)
  - Negative or zero trip duration
  - Negative or zero total amount
  - Zero passengers
- Filled nulls: `store_and_fwd_flag` nulls replaced with "N"
- Replaced `ratecode_id` 99 (unknown) with 1 (standard rate)
- Filtered outliers: distance > 620 miles, duration > 24 hours, total > $1000
- Created derived column: `trip_duration_min` from pickup/dropoff timestamps
- Weather data: renamed columns to readable format, cast date types
- **Result:** ~4.8M clean trip records from ~7.1M raw records

### Gold (Business)
Star schema dimensional model optimized for analytics:

| Table | Type | Description |
|-------|------|-------------|
| `gold_fact_trips` | Fact | Trip transactions with metrics and foreign keys |
| `gold_dim_zones` | Dimension | 265 NYC taxi zones with borough and service zone |
| `gold_dim_vendor` | Dimension | Taxi technology providers (Creative Mobile, Curb Mobility) |
| `gold_dim_payment_type` | Dimension | Payment methods (Credit card, Cash, Flex Fare, etc.) |
| `gold_dim_ratecode` | Dimension | Rate types (Standard, JFK, Newark, Negotiated, etc.) |
| `gold_dim_date` | Dimension | Calendar attributes (day, month, year, day of week) |
| `gold_dim_weather` | Dimension | Daily NYC weather (temperature, precipitation, snowfall, wind) |

## Data Model

```
  ┌────────────────┐    ┌────────────────┐    ┌────────────────┐
  │ gold_dim_vendor│    │ gold_dim_date  │    │gold_dim_weather│
  │────────────────│    │────────────────│    │────────────────│
  │ vendor_id (PK) │    │ date_key (PK)  │    │ date_key (PK)  │
  │ vendor_name    │    │ date           │    │ date           │
  └───────┬────────┘    │ day            │    │ temp_max       │
          │             │ month          │    │ temp_min       │
          │             │ year           │    │ precipitation  │
          │             │ day_of_week    │    │ snowfall       │
          │             │ day_name       │    │ wind_speed_max │
          │             │ month_name     │    └───────┬────────┘
          │             └───────┬────────┘            │
          │                     │                     │
          │         pickup_date_key / dropoff_date_key│
          │                     │                     │
  ┌───────▼─────────────────────▼─────────────────────▼────────┐
  │                    gold_fact_trips                          │
  │────────────────────────────────────────────────────────────│
  │ vendor_id (FK)           │ passenger_count                 │
  │ pickup_date_key (FK)     │ trip_distance                   │
  │ dropoff_date_key (FK)    │ trip_duration_min               │
  │ pu_location_id (FK)      │ fare_amount                     │
  │ do_location_id (FK)      │ tip_amount                      │
  │ payment_type (FK)        │ tolls_amount                    │
  │ ratecode_id (FK)         │ total_amount                    │
  │ tpep_pickup_datetime     │ extra                           │
  │ tpep_dropoff_datetime    │ mta_tax                         │
  │                          │ congestion_surcharge            │
  │                          │ airport_fee                     │
  └──────┬──────────────┬────┘                                 │
         │              │                                       
  ┌──────▼───────┐  ┌──▼─────────────┐  ┌─────────────────────┐
  │gold_dim_zones│  │gold_dim_payment│  │ gold_dim_ratecode   │
  │──────────────│  │────────────────│  │─────────────────────│
  │location_id(PK)│  │payment_type(PK)│  │ ratecode_id (PK)   │
  │ borough      │  │payment_desc    │  │ ratecode_description│
  │ zone         │  └────────────────┘  └─────────────────────┘
  │ service_zone │
  └──────────────┘
```

## Pipeline (Data Factory)

Automated ETL pipeline with parameterized ingestion:

```
┌─────────────┐     ┌─────────┐     ┌─────────┐
│ Copy Data   │────▶│ Bronze  │────▶│ Silver  │────┐
│ (Trips API) │     │ Trips   │     │ Trips   │    ├──▶ Gold
└─────────────┘     └─────────┘     └─────────┘    │
                    ┌─────────┐     ┌─────────┐    │
                    │ Bronze  │────▶│ Silver  │────┘
                    │ Weather │     │ Weather │
                    └─────────┘     └─────────┘
```

- **Copy Data Activity:** Parameterized with `year_month` variable to ingest any month from NYC TLC
- **Parallel execution:** Trip and weather pipelines run in parallel
- **Dependency management:** Gold layer waits for both Silver pipelines to complete
- **On Success triggers:** Each step only runs if the previous one succeeds

## Power BI Dashboard

Dashboard built with **Direct Lake** mode connecting to the Gold layer semantic model.

Visualizations include:
- Revenue trends over time
- Trips by borough and zone
- Payment method distribution
- Weather impact on trip volume
- KPI cards: total revenue, total trips, average distance

## Tech Stack

| Technology | Usage |
|-----------|-------|
| Microsoft Fabric | Cloud platform (Lakehouse, Notebooks, Pipelines) |
| PySpark | Data transformations across all medallion layers |
| Delta Lake | Storage format with ACID transactions and schema enforcement |
| Data Factory | Pipeline orchestration with parameterized ingestion |
| Open-Meteo API | External weather data integration (REST API) |
| Power BI | Dashboard and data visualization |
| Direct Lake | Semantic model connecting Power BI to Lakehouse |
| Python (requests) | API calls for weather data ingestion |

## Project Structure

```
├── notebooks/
│   ├── Bronze_Trips.ipynb        # Ingest raw trip data to Delta
│   ├── Bronze_Weather.ipynb      # Fetch weather data from Open-Meteo API
│   ├── Silver_Trips.ipynb        # Clean and validate trip data
│   ├── Silver_Weather.ipynb      # Clean weather data
│   └── Gold.ipynb                # Build star schema (facts + dimensions)
├── images/
│   └── dashboard.png             # Power BI dashboard screenshot
├── README.md
```

## Key Transformations

```python
# Bronze: Ingest from NYC TLC (parameterized by month)
df_trips = spark.read.parquet("Files/bronze/trips/")
df_trips.write.format("delta").mode("overwrite").saveAsTable("trips_bronze")

# Bronze: Fetch weather from Open-Meteo API
response = requests.get("https://archive-api.open-meteo.com/v1/archive", params={
    "latitude": 40.7128, "longitude": -74.0060,
    "start_date": "2026-01-01", "end_date": "2026-02-28",
    "daily": "temperature_2m_max,temperature_2m_min,precipitation_sum,snowfall_sum,wind_speed_10m_max"
})

# Silver: Clean and filter invalid records
df_trips = df_trips.filter(~((F.col("trip_distance") == 0) & (F.col("trip_duration_min") <= 0)))
df_trips = df_trips.filter((F.col("trip_duration_min") > 0) & (F.col("total_amount") > 0))
df_trips = df_trips.withColumn("trip_duration_min",
    (F.unix_timestamp("tpep_dropoff_datetime") - F.unix_timestamp("tpep_pickup_datetime")) / 60)

# Gold: Build fact table with date keys
df_trips = df_trips.withColumn("pickup_date_key", F.date_format("tpep_pickup_datetime", "yyyyMMdd").cast(IntegerType()))
fact_trips = df_trips.select("vendor_id", "pickup_date_key", "dropoff_date_key", ...)
```

## How to Reproduce

1. Activate a Microsoft Fabric trial at [fabric.microsoft.com](https://fabric.microsoft.com)
2. Create a workspace and Lakehouse
3. Run the pipeline or execute notebooks in order: Bronze → Silver → Gold
4. Trip data is automatically fetched from [NYC TLC](https://www.nyc.gov/site/tlc/about/tlc-trip-record-data.page)
5. Weather data is fetched from [Open-Meteo API](https://open-meteo.com) (free, no key needed)
6. Create a semantic model from Gold tables
7. Build Power BI dashboard with Direct Lake

## Future Improvements

- Incremental loads with Delta Lake merge (instead of full overwrite)
- Automated data quality checks
- Row-Level Security (RLS) in semantic model
- Additional months of data for trend analysis
- Pipeline failure alerts and monitoring

## Author

**Tomas** — Aspiring Data Engineer

- Building a portfolio in Microsoft Fabric
- Preparing for DP-600 certification
