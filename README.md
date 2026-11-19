# Citi Bike Big Data Pipeline (2024–2025)

An end-to-end data pipeline for **24 months of NYC Citi Bike trip data** (about 17.6 GB of compressed archives). It downloads the data, cleans it, loads it into MongoDB and HDFS, runs statistical analysis with **Hadoop MapReduce**, stores the results in **MongoDB Atlas** and presents them in a **Streamlit** dashboard.

> Team project (Big Data, Cracow University of Technology) by **Karolina Kokoszka** and **Aleksandra Frączek**.

![Python](https://img.shields.io/badge/Python-3.11-blue) ![Hadoop](https://img.shields.io/badge/Hadoop-3.2-yellow) ![MongoDB](https://img.shields.io/badge/MongoDB-7%20%2B%20Atlas-47A248) ![Docker](https://img.shields.io/badge/Docker-Compose-2496ED) ![Streamlit](https://img.shields.io/badge/Streamlit-dashboard-FF4B4B)

<!-- Add a dashboard screenshot here: ![Dashboard](docs/dashboard.png) -->

## Architecture

```
citibikenyc.com/system-data ──► scrape & build manifest ──► download & unzip ──► clean & normalise
                                                                                      │
                     ┌────────────────────────────────────────────────────────────────┤
                     ▼                                                                ▼
         raw ZIP archives ──► HDFS /tripdata/raw                       local MongoDB (idempotent)
                     │
                     ▼
     prepare_mapreduce_input: ZIP from HDFS ──► clean TSV ──► HDFS /tripdata/processed_tmp
                                                                    │
                                                         Hadoop Streaming (mapper/reducer)
                                                                    │
                                                                    ▼
                                                   MongoDB Atlas ──► Streamlit dashboard
```

## Highlights

- **Web scraping**: the list of monthly archives is scraped from the official Citi Bike *System Data* page and saved as a manifest, together with a scraping report.
- **HDFS as the data lake**: the raw ZIP archives are stored in HDFS, and the cleaned TSV input for MapReduce is generated from them into a separate working directory.
- **Idempotent import**: an `imported_files` collection tracks each archive, and records use `_id = ride_id::source_file`, so re-running the import never creates duplicates.
- **One-pass MapReduce**: the mapper computes 6 statistics per ride (rides per month, day of week, hour × customer type, bike type and start station, plus trip duration per customer type). It emits count, sum and sum of squares, so the reducer can compute the mean and standard deviation.
- **Verified results**: the MapReduce output for January 2025 was compared with an independent local pandas analysis. All **2,215 groups matched, with 0 discrepancies**.
- **Data quality**: for January 2025, 2,124,475 input rows became 2,124,186 after cleaning (289 invalid rows rejected).
- **Containerised**: MongoDB, HDFS NameNode and DataNode run in Docker. The Hadoop images are extended with Python 3 for Hadoop Streaming.
- **Unit tests** with `pytest`.

## Sample insights (January 2025)

- **About 70% of rides** were on electric bikes.
- Members ride shorter trips (**9.2 min** on average) than casual riders (**13.5 min**).

## Tech stack

Python 3.11 (pandas, requests, BeautifulSoup, pymongo) · Docker Compose · Hadoop 3.2 / HDFS / Hadoop Streaming · MongoDB 7 · MongoDB Atlas · Streamlit · pytest

## Project structure

```
├── bigdata_project/   # shared logic: config, download, transform, MongoDB and HDFS operations
├── scripts/           # pipeline steps (scrape, stage, load, ZIPs to HDFS, MapReduce input, MapReduce, Atlas)
├── hadoop/            # mapper.py, reducer.py
├── app/dashboard.py   # Streamlit dashboard
├── docker/            # Hadoop + Python image
└── tests/             # unit tests
```

## How to run

```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
pip install -r requirements.txt
Copy-Item .env.example .env        # set HDFS_USER=root (and the Atlas URI for the last step)

docker compose up -d --build

# Quick test on a single month
python scripts/scrape_tripdata.py
python scripts/download_and_stage.py        --year 2024 --month 1 --limit 1
python scripts/load_local_mongo.py          --year 2024 --month 1 --limit 1
python scripts/push_to_hdfs.py              --year 2024 --month 1 --limit 1
python scripts/prepare_mapreduce_input.py   --year 2024 --month 1 --limit 1
python scripts/run_mapreduce.py             --subdir year=2024/month=01
python scripts/load_results_atlas.py

streamlit run app/dashboard.py
python -m pytest
```

Run the scripts without `--year/--month/--limit` to process the full 2024–2025 range.

## Data source

[Citi Bike System Data](https://citibikenyc.com/system-data): official monthly trip archives.
