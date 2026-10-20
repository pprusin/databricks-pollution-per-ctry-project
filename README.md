# CO₂ per capita: a medallion pipeline on Databricks

A university project: a data pipeline in the medallion architecture (bronze → silver → gold) built on Databricks with Python/PySpark and Delta tables in Unity Catalog. It combines two data sources and computes CO₂ emissions per capita for countries and regions of the world.

## Data sources

| Source | Format | Contents |
|---|---|---|
| [World Bank API](https://api.worldbank.org/v2/country?format=json&per_page=400) | JSON (nested) | countries and aggregates: ISO3 code (`id`), region (`region.value`) |
| [Our World in Data: CO₂](https://github.com/owid/co2-data) | CSV | one row per country and year: `co2` (million tonnes), `population` (people), `iso_code` |

Join key: the country's ISO3 code (`World Bank id` = `iso_code`).

## Architecture

```
World Bank API (JSON) ─► bronze.countries ──┐
                                             ├─► silver.countries_x_pollution ─┬─► gold.co2_tonnes_per_capita_by_country
OWID CSV ─────────────► bronze.pollution ───┘                                  └─► gold.co2_tonnes_per_capita_by_region
```

All tables live in the `pollution_per_country_pipeline_project` catalog, in the `bronze`, `silver` and `gold` schemas.

### Bronze: raw data

Data lands in Delta tables unchanged. Writes use `createOrReplace()`, so they are idempotent: re-running a notebook does not duplicate rows.

| Notebook | Table | Description |
|---|---|---|
| `01_bronze_countries` | `bronze.countries` | API response (2nd element of the list; the 1st is metadata), 296 rows |
| `02_bronze_pollution` | `bronze.pollution` | the full OWID CSV |

### Silver: cleaned and joined data

Notebook `03_silver`, table `silver.countries_x_pollution` (22,386 rows, one per country and year).

- **countries:** de-duplicate on `id`, drop aggregate rows (`region.value = 'Aggregates'`, e.g. "Africa"), flatten the nested region into a `region` column.
- **pollution:** drop rows with a missing `iso_code`, `year`, `co2` or `population` (this removes non-country rows such as "World"), drop `co2 = 0`, de-duplicate on (`iso_code`, `year`).
- **join:** `left join` on `iso_code`, so real territories missing from the World Bank list (e.g. TWN, WLF, NIU) are not lost.
- **missing region:** null values are replaced with `Unclassified Region - {country}`.

The `co2 = 0` filter matters for gold: the population of such rows would inflate the denominator of the regional indicator.

### Gold: analytical tables

Notebook `04_gold`. In both tables the indicator is `avg_co2_per_capita`, in **tonnes of CO₂ per person**. The raw `co2` is in million tonnes, hence the `* 1,000,000` conversion.

| Table | Grain | Calculation |
|---|---|---|
| `gold.co2_tonnes_per_capita_by_country` | country + year | `co2 / population * 1e6` |
| `gold.co2_tonnes_per_capita_by_region` | region + year | `sum(co2) / sum(population) * 1e6` |

The regional indicator is a ratio of sums, **not** an average of country ratios. Averaging ratios would weigh a small country the same as a large one and distort the result.

`Unclassified Region - {country}` groups stay in the regional table as single-country groups. Consumers can filter them out.

## Repository layout

```
notebooks/
  01_bronze_countries.ipynb
  02_bronze_pollution.ipynb
  03_silver.ipynb
  04_gold.ipynb
```

## Running it

1. In Databricks, create the catalog `pollution_per_country_pipeline_project` with the schemas `bronze`, `silver` and `gold` (the project was built on Databricks Free Edition with serverless compute).
2. Import the notebooks from `notebooks/` into your workspace.
3. Run them in order: `01` → `02` → `03` → `04`.

The sources are fetched over HTTP, so there are no files to upload and no API keys to configure.

## Stack

Python, PySpark, Delta Lake, Databricks (Unity Catalog), `requests`, `pandas`.

## Possible improvements

- The bronze CSV is read with `pandas` (types are guessed, and there is no raw copy of the file). Alternative: store the file in a Volume and read it with `spark.read.csv`.
- An `is_unclassified` column in silver instead of filtering on the `Unclassified Region` string.
- Orchestrate the notebooks with a Databricks Job.
