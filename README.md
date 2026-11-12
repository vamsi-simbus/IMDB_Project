# 🎬 IMDB Data Engineering Project (Azure Databricks)

An end-to-end data engineering project that ingests the public **IMDB datasets** from **Azure Data Lake Storage Gen2 (ADLS Gen2)** into **Azure Databricks**, processes them using the **Medallion Architecture** (Bronze → Silver → Gold) with **PySpark** and **Spark SQL**, stores them as **Delta tables** in **Unity Catalog**, and deploys the project using **Databricks Asset Bundles (DABs)** with **CI/CD through GitHub Actions**.

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Business Use Cases](#business-use-cases)
3. [Architecture](#architecture)
4. [Tech Stack and Concepts Used](#tech-stack-and-concepts-used)
5. [Repository Structure](#repository-structure)
6. [Source Datasets](#source-datasets)
7. [Bronze Layer](#bronze-layer)
8. [Silver Layer](#silver-layer)
9. [Gold Layer](#gold-layer)
10. [Unity Catalog Objects](#unity-catalog-objects)
11. [Databricks Asset Bundle (DAB)](#databricks-asset-bundle-dab)
12. [Databricks Job (Workflow)](#databricks-job-workflow)
13. [Delta Live Tables (DLT) Pipeline](#delta-live-tables-dlt-pipeline)
14. [Python Wheel Package](#python-wheel-package)
15. [Testing](#testing)
16. [CI/CD with GitHub Actions](#cicd-with-github-actions)
17. [Getting Started](#getting-started)
18. [Security Notes](#security-notes)
19. [Future Enhancements](#future-enhancements)
20. [Glossary](#glossary)

---

## Project Overview

IMDB publishes large datasets about movies, TV shows, people (actors, directors), crews, episodes and ratings in **TSV (tab-separated values)** format. This project builds a data pipeline that:

- **Ingests** raw IMDB TSV files from ADLS Gen2 into Bronze Delta tables.
- **Migrates** tables from the legacy **Hive Metastore** to **Unity Catalog** using **CLONE**.
- **Fixes data types** (dates, integers, arrays) in the Bronze layer.
- **Cleans, filters and joins** data in the Silver layer for two business use cases.
- **Aggregates** data in the Gold layer into analytics-ready tables.
- **Packages and deploys** the project using **Databricks Asset Bundles**, a **Python wheel**, and **GitHub Actions**.

---

## Business Use Cases

### Use Case 1 – Genre-wise Movie Ratings by Year
Analyze how different movie genres perform over time (from the year 2000 onwards) using **average rating** and **total votes** per genre per year.

### Use Case 2 – Popular Actors and Directors
Identify popular **actors** and **directors** (from movies released in 2000 or later) based on:
- Number of movies
- Average rating of their movies
- Total votes received

Also track **how many movies each person worked on per year**.

---

## Architecture

```
┌───────────────────────────┐
│  ADLS Gen2                │
│  Container: internal-project
│  Folder: IMDB_DataSets/   │
│  (7 TSV files)            │
└─────────────┬─────────────┘
              │  abfss:// + OAuth (Service Principal)
              ▼
┌────────────────────────────────────── Azure Databricks ───────────────────────────────────────┐
│                                                                                                │
│  🥉 BRONZE  (raw → Delta, data type fixes)                                                     │
│     Hive Metastore (bronze_imdb)  ──CLONE──►  Unity Catalog                                     │
│     az_adb_simbus_training.imdb_project_schema.bronze_*                                         │
│                          │                                                                     │
│                          ▼                                                                     │
│  🥈 SILVER  (clean, filter, join)                                                               │
│     silver_usecase1_table           silver_popular_actor_directors                             │
│                          │                                                                     │
│                          ▼                                                                     │
│  🥇 GOLD    (aggregate, business-ready)                                                         │
│     gold_layer_usecase1   golden_popular_actors   golden_popular_directors   gold_movies_per_year│
│                                                                                                │
└────────────────────────────────────────────────────────────────────────────────────────────────┘
              ▲
              │  databricks bundle validate / deploy
┌─────────────┴─────────────┐
│ GitHub Actions (CI/CD)    │
│ Databricks Asset Bundles  │
└───────────────────────────┘
```

---

## Tech Stack and Concepts Used

| Category | Technology / Concept | Where It Is Used |
|----------|----------------------|------------------|
| Cloud | **Microsoft Azure** | Hosting storage and Databricks |
| Storage | **Azure Data Lake Storage Gen2 (ADLS Gen2)** | Stores raw IMDB TSV files |
| Storage Access | **ABFSS protocol** (`abfss://`) | Reading files from ADLS Gen2 |
| Authentication | **OAuth 2.0 with Azure Service Principal** (Client ID, Client Secret, Tenant endpoint) via `ClientCredsTokenProvider` | Secure Databricks access to ADLS |
| Compute | **Azure Databricks**, **Apache Spark** | Data processing |
| Languages | **PySpark (Python)**, **Spark SQL** | Transformations and table creation |
| File Format | **TSV** (read with CSV reader and `\t` delimiter) | Source data |
| Table Format | **Delta Lake** | All Bronze, Silver and Gold tables |
| Metadata / Governance | **Hive Metastore** (legacy) and **Unity Catalog** (catalog → schema → table) | Table storage and migration |
| Migration | **Delta CLONE** (`CREATE TABLE ... CLONE`) | Moving tables from Hive Metastore to Unity Catalog |
| Cluster Access Modes | **Standard (Shared)** for Unity Catalog, **No Isolation** for direct ADLS access, **Single User** for job clusters | Cluster configuration |
| Data Architecture | **Medallion Architecture** (Bronze, Silver, Gold) | Layered data processing |
| Transformations | Type casting, `split`, `year`, `dropna`, `filter`, `dropDuplicates`, column renaming | Data cleaning |
| Joins | **Inner joins** on `tconst` and `nconst` | Combining titles, ratings, people |
| Aggregations | `groupBy`, `avg`, `sum`, `count`, `round`, `first` | Gold metrics |
| Write Options | `mode("append")`, `mode("overwrite")`, `overwriteSchema`, `mergeSchema` | Schema evolution and table refresh |
| Deployment | **Databricks Asset Bundles (DABs)** – `databricks.yml` | Infrastructure as code |
| Orchestration | **Databricks Jobs (Workflows)** – notebook, pipeline and Python wheel tasks | Scheduled pipeline runs |
| Pipelines | **Delta Live Tables (DLT)** | Declarative pipeline (template) |
| Packaging | **Python Wheel (.whl)**, `setuptools`, `setup.py`, entry points | Reusable Python code |
| Testing | **pytest**, **Databricks Connect** | Unit testing |
| CI/CD | **GitHub Actions**, **Databricks CLI** (`databricks/setup-cli`) | Automated validation and deployment |
| Secrets | **GitHub Secrets and Variables** (`TOKEN`, `HOST`) | Secure CI/CD authentication |
| Version Control | **Git**, **GitHub** (branching) | Source code management |
| IDE | **Visual Studio Code** with Databricks extension | Local development |
| Notifications | **Email notifications on job failure** | Monitoring |

---

## Repository Structure

```
IMDB_Project/
├── .github/
│   └── workflows/
│       └── deploy_to_dev.yaml                 # GitHub Actions CI/CD workflow
├── .vscode/                                   # VS Code settings and extensions
├── fixtures/                                  # Test fixtures (placeholder)
├── resources/
│   ├── IMDB_Project.job.yml                   # Databricks Job definition
│   └── IMDB_Project.pipeline.yml              # DLT pipeline definition
├── scratch/                                   # Personal exploratory notebooks (git-ignored)
├── src/
│   ├── Bronze_Notebook/
│   │   └── Ingest_Raw_IMDB_Datasets.ipynb     # ADLS → Bronze ingestion + UC migration + type fixes
│   ├── Silver_Notebook/
│   │   ├── Silver_Layer_UseCase1_IMDB_Project.ipynb   # Use Case 1 silver table
│   │   └── Silver_Popular_Actors_Directors.ipynb      # Use Case 2 silver table
│   ├── Gold_Notebook/
│   │   ├── Gold layer IMDB usercase 1.ipynb           # Use Case 1 gold table
│   │   └── gold_popular_actors_directors.ipynb        # Use Case 2 gold tables
│   ├── IMDB_Project/
│   │   ├── __init__.py
│   │   └── main.py                            # Python package entry point (wheel)
│   ├── dlt_pipeline.ipynb                     # DLT pipeline notebook (template)
│   └── notebook.ipynb                         # Default job notebook (template)
├── tests/
│   └── main_test.py                           # pytest unit test
├── databricks.yml                             # Databricks Asset Bundle config
├── setup.py                                   # Wheel build configuration
├── pytest.ini                                 # pytest configuration
├── requirements-dev.txt                       # Local development dependencies
└── README.md
```

---

## Source Datasets

Seven IMDB TSV files are stored in ADLS Gen2 at:
```
abfss://internal-project@<storage-account>.dfs.core.windows.net/IMDB_DataSets/
```

| File | Description | Key Columns |
|------|-------------|-------------|
| `name.basics.tsv` | People (actors, directors, writers) | `nconst`, `primaryName`, `birthYear`, `deathYear`, `knownForTitles` |
| `title.akas.tsv` | Alternative/regional titles | `titleId`, `title`, `region`, `language` |
| `title.basics.tsv` | Basic title information | `tconst`, `titleType`, `primaryTitle`, `startYear`, `endYear`, `runtimeMinutes`, `genres` |
| `title.crew.tsv` | Directors and writers per title | `tconst`, `directors`, `writers` |
| `title.episode.tsv` | TV episode details | `tconst`, `parentTconst`, `seasonNumber`, `episodeNumber` |
| `title.principals.tsv` | Main cast and crew per title | `tconst`, `nconst`, `category` |
| `title.ratings.tsv` | Ratings and votes | `tconst`, `averageRating`, `numVotes` |

**Keys used for joins:**
- `tconst` – unique title (movie/show) ID
- `nconst` – unique person ID

---

## Bronze Layer

**Notebook:** `src/Bronze_Notebook/Ingest_Raw_IMDB_Datasets.ipynb`

### Step 1 – Configure ADLS Gen2 Access
Spark is configured to access ADLS Gen2 using **OAuth 2.0 with a Service Principal**:
- `fs.azure.account.auth.type` = `OAuth`
- `fs.azure.account.oauth.provider.type` = `ClientCredsTokenProvider`
- Client ID, Client Secret and Tenant OAuth token endpoint

### Step 2 – Create Unity Catalog Schema
```sql
USE CATALOG az_adb_simbus_training;
CREATE SCHEMA IF NOT EXISTS az_adb_simbus_training.imdb_project_schema;
```
The cluster access mode was switched to **Standard (Shared)** to enable Unity Catalog, and to **No Isolation** when connecting directly to ADLS Gen2.

### Step 3 – Ingest TSV Files
- Listed files using `dbutils.fs.ls()`.
- Read each TSV file with the CSV reader using `header = true`, `inferSchema = true` and `delimiter = "\t"`.
- Wrote each DataFrame as a **Delta table** (append mode) into the Hive Metastore schema `bronze_imdb`.

### Step 4 – Migrate to Unity Catalog
Used **Delta CLONE** to copy tables from Hive Metastore into Unity Catalog:
```sql
CREATE TABLE az_adb_simbus_training.imdb_project_schema.bronze_name_basics
CLONE hive_metastore.bronze_imdb.bronze_name_basics;
```
After migration, the old Hive Metastore tables were dropped with `DROP TABLE IF EXISTS`.

### Step 5 – Fix Data Types
Columns were converted to correct types and saved with `overwriteSchema = true`:

| Table | Column | Change |
|-------|--------|--------|
| `bronze_name_basics` | `birthYear`, `deathYear` | String → Date |
| `bronze_name_basics` | `knownForTitles` | String → Array of Strings (`split` on `,`) |
| `bronze_title_basics` | `startYear`, `endYear` | String → Date |
| `bronze_title_basics` | `runtimeMinutes` | String → Integer |
| `bronze_title_episode` | `seasonNumber`, `episodeNumber` | String → Integer |

### Bronze Tables Created
`bronze_name_basics`, `bronze_title_akas`, `bronze_title_basics`, `bronze_title_crew`, `bronze_title_episode`, `bronze_title_principals`, `bronze_title_ratings`

---

## Silver Layer

### Use Case 1 – `silver_usecase1_table`
**Notebook:** `src/Silver_Notebook/Silver_Layer_UseCase1_IMDB_Project.ipynb`

1. Read `bronze_title_ratings` and removed null rows (`dropna(how='any')`).
2. Read `bronze_title_basics`, selected `tconst`, `startYear`, `genres`, `titleType`, removed nulls, extracted the **year** from `startYear` and renamed it to `Year`.
3. **Inner joined** ratings and basics on `tconst`.
4. Filtered only **movies** (`titleType = 'movie'`) and selected `Year`, `genres`, `averageRating`, `numVotes`, `titleType`.
5. Kept movies from **2000 onwards** and extracted the **first genre** from the comma-separated `genres` column.
6. Saved as `silver_usecase1_table` (overwrite, `mergeSchema = true`).

### Use Case 2 – `silver_popular_actor_directors`
**Notebook:** `src/Silver_Notebook/Silver_Popular_Actors_Directors.ipynb`

1. From `bronze_title_basics`, selected **movies** with `tconst`, `primaryTitle` and release year.
2. From `bronze_title_principals`, selected only **actors** and **directors**.
3. **Inner joined** movies with principals on `tconst`.
4. From `bronze_name_basics`, selected `nconst` and `primaryName`.
5. From `bronze_title_ratings`, selected `averageRating` and `numVotes`, and joined on `tconst`.
6. Joined with names on `nconst` and kept movies from **2000 onwards**.
7. Renamed columns to `person_name`, `movie_title`, `start_Year`, `role`, `average_rating`, `num_votes` and removed duplicates.
8. Saved as `silver_popular_actor_directors` (Delta, overwrite).

---

## Gold Layer

### Use Case 1 – `gold_layer_usecase1`
**Notebook:** `src/Gold_Notebook/Gold layer IMDB usercase 1.ipynb`

- Removed rows where `genres` is null or `\N` (IMDB's null marker).
- Grouped by **genre** and **year**:
  - `avg_rating` – average rating, rounded to 2 decimals
  - `total_votes` – sum of votes
- Saved as `gold_layer_usecase1`.

### Use Case 2 – Popular Actors and Directors
**Notebook:** `src/Gold_Notebook/gold_popular_actors_directors.ipynb`

| Gold Table | Logic |
|------------|-------|
| `golden_popular_actors` | Actors grouped by name with `movie_count`, `average_rating` (2 decimals), `total_votes`, `role`; only people with **more than 10 movies** |
| `golden_popular_directors` | Same logic for directors |
| `gold_movies_per_year` | Number of movies per person, per year and role (`movies_in_year`) |

---

## Unity Catalog Objects

**Catalog:** `az_adb_simbus_training` | **Schema:** `imdb_project_schema`

| Layer | Table |
|-------|-------|
| Bronze | `bronze_name_basics`, `bronze_title_akas`, `bronze_title_basics`, `bronze_title_crew`, `bronze_title_episode`, `bronze_title_principals`, `bronze_title_ratings` |
| Silver | `silver_usecase1_table`, `silver_popular_actor_directors` |
| Gold | `gold_layer_usecase1`, `golden_popular_actors`, `golden_popular_directors`, `gold_movies_per_year` |

---

## Databricks Asset Bundle (DAB)

**File:** `databricks.yml`

Databricks Asset Bundles let you define jobs, pipelines and settings as code and deploy them to different environments.

| Setting | Description |
|---------|-------------|
| `bundle.name` | `IMDB_Project` |
| `include` | Loads all resource files from `resources/*.yml` |
| **`dev` target** | `mode: development` (default). Resources are prefixed with `[dev <username>]` and schedules are paused |
| **`prod` target** | `mode: production`. Deploys to a fixed `root_path` with `CAN_MANAGE` permission for the owner |
| `workspace.host` | Azure Databricks workspace URL |

---

## Databricks Job (Workflow)

**File:** `resources/IMDB_Project.job.yml`

| Setting | Value |
|---------|-------|
| Job name | `IMDB_Project_job` |
| Trigger | **Periodic** – every 1 day |
| Notifications | **Email on failure** |
| Cluster | **Job cluster**, Spark `15.4.x-scala2.12`, node `Standard_D3_v2`, **autoscale 1–4 workers**, access mode `SINGLE_USER` |

**Tasks:**
```
notebook_task  ──►  refresh_pipeline  ──►  main_task
(Notebook)          (DLT Pipeline)         (Python Wheel)
```

| Task | Type | Description |
|------|------|-------------|
| `notebook_task` | Notebook task | Runs `src/notebook.ipynb` |
| `refresh_pipeline` | Pipeline task | Refreshes the DLT pipeline |
| `main_task` | Python wheel task | Runs the `main` entry point from the `IMDB_Project` wheel |

> The job currently runs the default template notebook, pipeline and wheel. The Bronze, Silver and Gold notebooks can be added as tasks to automate the full Medallion pipeline (see [Future Enhancements](#future-enhancements)).

---

## Delta Live Tables (DLT) Pipeline

**Files:** `resources/IMDB_Project.pipeline.yml`, `src/dlt_pipeline.ipynb`

| Setting | Value |
|---------|-------|
| Pipeline name | `IMDB_Project_pipeline` |
| Catalog | `az_adb_simbus_training` |
| Schema | `IMDB_Project_<target>` (e.g. `IMDB_Project_dev`) |
| Source | `src/dlt_pipeline.ipynb` |

The notebook uses `@dlt.view` and `@dlt.table` decorators to define a **declarative pipeline**. It currently contains the template example (sample taxi data with a filter), which can be replaced with IMDB Bronze → Silver → Gold logic.

---

## Python Wheel Package

**Files:** `setup.py`, `src/IMDB_Project/`

- The project code in `src/IMDB_Project` is packaged as a **Python wheel (.whl)** using `setuptools`.
- The version uses a **UTC timestamp** as a local version identifier so new builds are always picked up by clusters.
- The entry point `main=IMDB_Project.main:main` is used by the job's **Python wheel task**.
- `main.py` creates a Spark session using **Databricks Connect** (`DatabricksSession`), falling back to a normal `SparkSession`.

Build locally:
```bash
pip install setuptools wheel
python setup.py bdist_wheel
```

---

## Testing

**Files:** `tests/main_test.py`, `pytest.ini`

- Tests use **pytest**, with `testpaths = tests` and `pythonpath = src`.
- `main_test.py` checks that the sample data function returns more than 5 rows.
- Tests can run locally against Databricks using **Databricks Connect**.

```bash
pip install -r requirements-dev.txt
pytest
```

**Development dependencies (`requirements-dev.txt`):** `databricks-dlt`, `pytest`, `setuptools`, `wheel`, and optionally `databricks-connect` (matching the cluster runtime 15.4).

---

## CI/CD with GitHub Actions

**File:** `.github/workflows/deploy_to_dev.yaml`

| Setting | Value |
|---------|-------|
| Workflow name | `deploy_to_dev` |
| Trigger | `workflow_dispatch` (manual run from GitHub Actions tab) |
| Concurrency | 1 (one deployment at a time) |
| Runner | `ubuntu-latest` |
| Environment | `dev` |

**Steps:**
1. **Checkout** the repository (`actions/checkout@v3`).
2. **Install Databricks CLI** (`databricks/setup-cli@main`).
3. **Validate** the bundle: `databricks bundle validate`.
4. **Deploy** the bundle: `databricks bundle deploy` to the `dev` target.

**Authentication:**
- `DATABRICKS_HOST` – from GitHub **variable** `HOST`
- `DATABRICKS_TOKEN` – from GitHub **secret** `TOKEN`

To set these up: GitHub repo → **Settings → Secrets and variables → Actions**, then add the `HOST` variable and `TOKEN` secret under the `dev` environment.

---

## Getting Started

### Prerequisites
- Azure subscription with **ADLS Gen2** and **Azure Databricks** (Unity Catalog enabled)
- Azure **Service Principal** with access to the storage account
- **Databricks CLI** installed
- Python 3.10+ (for local development)

### 1. Clone the repository
```bash
git clone https://github.com/vamsi-simbus/IMDB_Project.git
cd IMDB_Project
```

### 2. Authenticate the Databricks CLI
```bash
databricks configure
```

### 3. Upload IMDB datasets
Download the IMDB TSV files from [IMDB Non-Commercial Datasets](https://developer.imdb.com/non-commercial-datasets/) and upload them to the `IMDB_DataSets/` folder in your ADLS Gen2 container.

### 4. Run the notebooks in order
1. `Bronze_Notebook/Ingest_Raw_IMDB_Datasets`
2. `Silver_Notebook/Silver_Layer_UseCase1_IMDB_Project`
3. `Silver_Notebook/Silver_Popular_Actors_Directors`
4. `Gold_Notebook/Gold layer IMDB usercase 1`
5. `Gold_Notebook/gold_popular_actors_directors`

### 5. Deploy with Databricks Asset Bundles
```bash
databricks bundle validate
databricks bundle deploy --target dev     # development
databricks bundle deploy --target prod    # production
databricks bundle run IMDB_Project_job
```

### 6. Validate results
```sql
SELECT * FROM az_adb_simbus_training.imdb_project_schema.gold_layer_usecase1 ORDER BY Year, avg_rating DESC;
SELECT * FROM az_adb_simbus_training.imdb_project_schema.golden_popular_actors ORDER BY total_votes DESC LIMIT 10;
SELECT * FROM az_adb_simbus_training.imdb_project_schema.golden_popular_directors ORDER BY average_rating DESC LIMIT 10;
```

---

## Security Notes

- **Never store credentials in notebooks or code.** Use **Databricks Secret Scopes** (or Azure Key Vault-backed scopes) with `dbutils.secrets.get()` for the Service Principal secret.
- If a client secret has ever been committed to the repository, **rotate it in Azure immediately**, since Git history keeps old versions.
- Prefer **Unity Catalog external locations and storage credentials** over cluster-level Spark configs for ADLS access.
- Keep the Databricks token only in **GitHub Secrets**, never in the YAML file.

---

## Future Enhancements

- Add Bronze, Silver and Gold notebooks as tasks in the Databricks Job for full automation.
- Replace the template DLT pipeline with IMDB logic and add **DLT expectations** for data quality.
- Move secrets to Databricks Secret Scopes / Azure Key Vault.
- Use **Unity Catalog external locations** instead of No Isolation clusters.
- Add **Auto Loader** for incremental ingestion of new files.
- Add unit tests for Silver and Gold transformations.
- Add a CI/CD workflow for the `prod` target with approvals.
- Build Power BI / Databricks SQL dashboards on Gold tables.

---

## Glossary

| Term | Meaning |
|------|---------|
| **ADLS Gen2** | Azure Data Lake Storage Gen2, scalable cloud storage for big data |
| **ABFSS** | Azure Blob File System Secure – the protocol used to read ADLS Gen2 files |
| **Service Principal** | An Azure identity used by applications to access resources securely |
| **OAuth 2.0** | Standard authentication protocol used to get access tokens |
| **Medallion Architecture** | Layered design: Bronze (raw), Silver (clean), Gold (aggregated) |
| **Delta Lake** | Storage format with ACID transactions, schema enforcement and time travel |
| **Hive Metastore** | Legacy Databricks metadata store for tables |
| **Unity Catalog** | Databricks governance layer with catalog → schema → table structure |
| **CLONE** | Delta command to copy a table's data and metadata to a new location |
| **overwriteSchema / mergeSchema** | Options to replace or evolve a table's schema when writing |
| **tconst / nconst** | IMDB unique IDs for titles and people |
| **Databricks Asset Bundles (DABs)** | Infrastructure-as-code framework to deploy Databricks jobs and pipelines |
| **Job Cluster** | Temporary cluster created for a job run and terminated afterwards |
| **Autoscaling** | Automatically adding or removing workers based on load |
| **Delta Live Tables (DLT)** | Declarative framework for building reliable data pipelines |
| **Python Wheel** | A packaged Python library (.whl) that can be installed on clusters |
| **Databricks Connect** | Library to run Spark code from a local IDE on a Databricks cluster |
| **pytest** | Python testing framework |
| **GitHub Actions** | GitHub's CI/CD automation service |
| **workflow_dispatch** | Manual trigger for a GitHub Actions workflow |
