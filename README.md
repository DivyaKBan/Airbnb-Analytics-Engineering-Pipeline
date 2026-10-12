# Airbnb Analytics Engineering Pipeline

A data engineering portfolio project that transforms Airbnb bookings, host details, and property listings into structured datasets for analysis. The workflow combines **AWS S3**, **Snowflake**, and **dbt** to organize raw inputs, clean and standardize records, and publish models for analytics.

## Project at a glance

- **Data domain:** Airbnb listings, bookings, and hosts
- **Warehouse:** Snowflake
- **Transformations:** dbt Core and SQL/Jinja
- **Cloud storage:** AWS S3 (for the CSV ingestion path)
- **Python:** 3.12+
- **Version control:** Git

The project follows a **Bronze → Silver → Gold** medallion pattern. It also includes examples of incremental models, reusable macros, dbt tests, documentation, and snapshots for tracking changes over time.

## Pipeline design

```text
CSV files
   |
   v
AWS S3 (ingestion / storage)
   |
   v
Snowflake staging tables
   |
   v
Bronze models  --> minimally transformed source records
   |
   v
Silver models  --> cleaned, standardized, validated records
   |
   v
Gold models    --> reporting-friendly and analytics-ready datasets
```

The CSV-to-S3 step depends on the ingestion process configured for your environment. The SQL models described below start from the staging tables in Snowflake.

## Data layers

### Bronze: source-aligned data

The Bronze models bring staging data into the dbt transformation workflow with limited changes.

| Model | Purpose |
|---|---|
| `bronze_bookings` | Booking-level records |
| `bronze_hosts` | Host attributes |
| `bronze_listings` | Property and listing details |

### Silver: standardized data

The Silver layer applies cleaning and business-friendly transformations to make the source records easier to use downstream.

| Model | Purpose |
|---|---|
| `silver_bookings` | Validated and standardized booking records |
| `silver_hosts` | Host information enriched with quality-related metrics |
| `silver_listings` | Standardized listing attributes, including price categories |

### Gold: analytics models

Gold models combine the cleaned data into structures intended for reporting and analysis.

- **`fact`** — a fact-oriented model for dimensional analysis.
- **`obt` (One Big Table)** — a denormalized dataset that combines booking, listing, and host information.
- **Ephemeral models** — intermediate transformations that dbt inlines into downstream SQL rather than creating standalone warehouse tables.

### Historical tracking with snapshots

dbt snapshots can preserve versions of records as source attributes change. The project includes snapshot configurations for:

- `dim_bookings`
- `dim_hosts`
- `dim_listings`

These are intended to support historical and point-in-time analysis using Slowly Changing Dimension Type 2 (SCD Type 2) patterns. Snapshot behavior depends on the configuration and source fields defined in the YAML files.

## Repository layout

```text
AWS_DBT_Snowflake/
├── README.md
├── pyproject.toml
├── main.py
├── SourceData/
│   ├── bookings.csv
│   ├── hosts.csv
│   └── listings.csv
├── DDL/
│   ├── ddl.sql
│   └── resources.sql
└── aws_dbt_snowflake_project/
    ├── dbt_project.yml
    ├── ExampleProfiles.yml
    ├── models/
    │   ├── sources/
    │   │   └── sources.yml
    │   ├── bronze/
    │   │   ├── bronze_bookings.sql
    │   │   ├── bronze_hosts.sql
    │   │   └── bronze_listings.sql
    │   ├── silver/
    │   │   ├── silver_bookings.sql
    │   │   ├── silver_hosts.sql
    │   │   └── silver_listings.sql
    │   └── gold/
    │       ├── fact.sql
    │       ├── obt.sql
    │       └── ephemeral/
    │           ├── bookings.sql
    │           ├── hosts.sql
    │           └── listings.sql
    ├── macros/
    │   ├── generate_schema_name.sql
    │   ├── multiply.sql
    │   ├── tag.sql
    │   └── trimmer.sql
    ├── analyses/
    │   ├── explore.sql
    │   ├── if_else.sql
    │   └── loop.sql
    ├── snapshots/
    │   ├── dim_bookings.yml
    │   ├── dim_hosts.yml
    │   └── dim_listings.yml
    ├── tests/
    │   └── source_tests.sql
    └── seeds/
```

The tree above describes the intended layout; adjust paths and filenames if your local repository differs.

## Requirements

Before running the project, make sure you have:

1. A Snowflake account with permission to create or use the required database, schemas, tables, and warehouse.
2. Python 3.12 or a compatible version supported by the installed dbt packages.
3. `dbt-core` and the `dbt-snowflake` adapter.
4. AWS access if you are using S3 for source-file storage.
5. The source CSV files and the SQL scripts referenced in this README.


Replace `<repository-url>` with your Git repository URL.

### 1. Create and activate a virtual environment

**Windows PowerShell:**

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
```

If PowerShell blocks activation scripts, you can run the environment's Python directly instead:

```powershell
.\.venv\Scripts\python.exe --version
```

**Linux/macOS:**

```bash
python3 -m venv .venv
source .venv/bin/activate
```

### 2. Install dbt and the Snowflake adapter

Using `uv`:

```bash
uv pip install dbt-core dbt-snowflake
```

Or using pip:

```bash
python -m pip install dbt-core dbt-snowflake
```

Confirm the installation:

```bash
dbt --version
```



### 3. Load the source CSV files

Load the files from `SourceData/` into the staging tables expected by the dbt sources:

| File | Expected staging table |
|---|---|
| `bookings.csv` | `AIRBNB.STAGING.BOOKINGS` |
| `hosts.csv` | `AIRBNB.STAGING.HOSTS` |
| `listings.csv` | `AIRBNB.STAGING.LISTINGS` |

The exact loading method (for example, Snowflake stages and `COPY INTO`, or an AWS S3 integration) depends on the setup you choose. Make sure the actual table and column names match `models/sources/sources.yml`.

## Running the project

Run dbt commands from the directory containing `dbt_project.yml`, unless you pass the project directory explicitly.

```bash
cd aws_dbt_snowflake_project
```


### Check the connection and configuration

```bash
dbt debug
```

### Fetch package dependencies

```bash
dbt deps
```

### Execute models

```bash
dbt run
```

Run a particular layer:

```bash
dbt run --select bronze
dbt run --select silver
dbt run --select gold
```

The selection syntax assumes the model paths or configuration are organized so that the layer names can be selected as shown. If your project uses different selectors, update these commands accordingly.

### Run data tests and snapshots

```bash
dbt test
dbt snapshot
```

### Build models and run associated tests

```bash
dbt build
```

### Generate and view documentation

```bash
dbt docs generate
dbt docs serve
```

The documentation site includes model descriptions, test metadata, and lineage information available in the dbt project.

## Transformation patterns demonstrated

### Incremental models

Incremental materialization can limit processing to newly added or changed records after the first build. The exact filter should use a reliable timestamp or change-tracking column from your source.

Illustrative pattern:

```sql
{{ config(materialized='incremental') }}

select *
from {{ source('staging', 'bookings') }}

{% if is_incremental() %}
where created_at > (
    select coalesce(max(created_at), '1900-01-01'::timestamp)
    from {{ this }}
)
{% endif %}
```

Adapt the source name, table name, and timestamp column to your actual project. If existing records can be updated, a timestamp filter alone may not be sufficient; consider a suitable `unique_key` and incremental strategy.

### Reusable macros

Macros keep repeated SQL logic in one place. The repository includes examples such as:

- `tag()` — assigns price bands such as low, medium, or high.
- `multiply()` — reusable multiplication logic.
- `trimmer()` — string cleanup utilities.
- `generate_schema_name()` — controls how dbt derives target schema names.

Example usage, assuming the macro accepts a SQL expression:

```sql
{{ tag('CAST(price_per_night AS INT)') }} AS price_per_night_tag
```

Check the macro definitions for the exact arguments and business rules implemented in this repository.

### Jinja-based SQL generation

Jinja lets models generate repetitive SQL from configuration lists or loops. This can help keep a denormalized model such as `obt` easier to maintain as joins are added or changed.

### Schema separation by layer

The custom schema-generation macro is intended to organize outputs by layer, for example:

- Bronze models → `AIRBNB.BRONZE`
- Silver models → `AIRBNB.SILVER`
- Gold models → `AIRBNB.GOLD`

The final schema names depend on `generate_schema_name.sql`, dbt configuration, and the target profile.

## Data quality and lineage

The repository includes a SQL test file and dbt's built-in testing capabilities. A useful validation suite can cover:

- Required values (`not_null`)
- Unique identifiers (`unique`)
- Valid relationships between models (`relationships`)
- Accepted values and business rules
- Source data assumptions

dbt builds a dependency graph from model and source references. This graph helps trace upstream inputs and understand which downstream models may be affected by a change.

## Operational and development practices

- **Credentials:** Keep secrets outside version control and use least-privilege Snowflake roles.
- **Code review:** Review SQL and model changes before merging.
- **Formatting:** Use a SQL formatter such as `sqlfmt` if configured for the repository.
- **Incremental processing:** Validate timestamp logic and use full refreshes only when appropriate.
- **Intermediate models:** Use ephemeral materialization when an intermediate result does not need its own warehouse table.
- **Warehouse performance:** Choose warehouse size and clustering only after checking query patterns and data volume.
- **Reproducibility:** Keep dependency versions and setup instructions consistent with the project configuration.

## Troubleshooting

| Symptom | Checks to perform |
|---|---|
| dbt cannot connect to Snowflake | Run `dbt debug`; check account identifier, credentials, role, warehouse, database, schema, and network access. |
| `dbt_project.yml` not found | Change into the directory containing the file, or pass the correct project directory. |
| Model compilation fails | Review Jinja syntax, `ref()`/`source()` names, YAML configuration, and model dependencies. |
| A source or relation is missing | Verify that staging tables exist and that source definitions match their database, schema, and table names. |
| Incremental results are incomplete | Check the timestamp filter, update behavior, `unique_key`, and whether a full refresh is needed. |
| PowerShell blocks virtual environment activation | Use `.\.venv\Scripts\Activate.ps1` from PowerShell, or invoke `.\.venv\Scripts\python.exe` directly. |
| `dbt` launcher fails | Confirm the active interpreter and reinstall dbt packages inside the intended environment if needed. |

For an incremental model that needs a complete rebuild, and only after confirming that a rebuild is appropriate, run:

```bash
dbt run --full-refresh
```

## Possible next steps

- Add automated CI checks for SQL compilation and dbt tests.
- Build a dashboard on top of the Gold models in Power BI or Tableau.
- Add monitoring and alerts for failed loads or quality checks.
- Expand business metrics for bookings, host performance, listing pricing, and trends.
- Review whether sensitive fields need masking or restricted access.




