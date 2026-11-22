# SkyPoints – Airline Loyalty Data Engineering Project

An airline loyalty data processing project built using Azure Databricks, Azure Data Lake Storage Gen2, PySpark, Spark SQL, Delta Lake, and Unity Catalog.

The project processes sample member-profile data and partner-airline redemption transactions through Bronze, Silver, and Gold layers, with separate validation and reject-handling notebooks.

## 1. Technology stack

- Azure Databricks
- Azure Data Lake Storage Gen2 (ADLS Gen2)
- Apache Spark / PySpark
- Spark SQL
- Delta Lake
- Unity Catalog
- GitHub

## 2. Project architecture

```text
Member profile files                 Partner redemption JSON
        |                                      |
        v                                      v
     ADLS Raw                                ADLS Raw
        |                                      |
        v                                      v
01 - Bronze ingestion                 04 - Redemption processing
        |
        v
   Bronze Delta
        |
        v
02 - Silver transformation
  - Standardise source columns
  - Parse and validate dates
  - Derive Age
  - Derive Stale_Member
        |
        v
   Silver Delta
        |
        v
03 - Gold member processing
  - Select latest member record
  - Split records by country
        |
        v
Country-specific Gold outputs

05 - Data quality validation
06 - Final validation
07 - Reject handling
```

## 3. Databricks notebooks

| Notebook | What it does |
|---|---|
| `01_Bronze_Ingestion` | Reads the sample Australia, India, and USA member workbooks, standardises their columns into a common structure, adds source and ingestion metadata, and writes Bronze Delta data. |
| `02_Silver_Transformation` | Standardises member data, parses date fields, records date-validation statuses, derives `Age` from DOB, derives `Stale_Member` based on days since last flight, and writes Silver Delta data. |
| `03_Gold_Member_Processing` | Selects the latest record using the implemented member-name-based ordering logic and writes country-specific Gold outputs for Australia, India, and USA. |
| `04_Gold_Redemption_Processing` | Reads the partner JSON feed, explodes the nested redemption array, creates one row per redemption transaction, and attempts a left join to member data using member ID. |
| `05_Data_Quality_Validation` | Checks mandatory member fields, invalid enrollment dates, DOB quality, duplicate-name observations, and redemption data quality. |
| `06_Final_Validation` | Checks record counts and selected output values across Bronze, Silver, Gold, and redemption data. |
| `07_Reject_Handling` | Identifies member records with missing or invalid mandatory values and prepares a rejected-record Delta output. |

The Databricks workspace exports are included as `.dbc` archives:
- `Incubyte_Data_Engineering.dbc` — contains the seven project notebooks.
- `08_DDL_Scrips.dbc` — contains the SQL DDL notebook.

Import these archives into a Databricks workspace to inspect the notebooks.

## 4. Data transformations implemented

### Member profile data

- Source columns from the sample country workbooks are mapped to a common member schema.
- Date fields are parsed and validation statuses are recorded.
- `Age` is derived from a valid DOB.
- `Stale_Member` is set based on whether the last flight was more than 90 days ago. When the last-flight date is unavailable or invalid, a valid stale calculation is not produced.
- The latest member record is selected using the current notebook ordering logic.
- Sample records are split into Australia, India, and USA outputs.

### Redemption data

- The nested `redemptions` array is exploded into individual transaction rows.
- Transaction ID, transaction date, partner, miles redeemed, status, member ID, and feed date are retained in the flattened output.
- A left join is attempted using the redemption member ID and member profile ID.

## 5. Sample run results

The sample execution produced the following observations:

| Output | Sample result |
|---|---:|
| Bronze member records | 9 |
| Silver member records | 9 |
| Australia Gold records | 3 |
| India Gold records | 3 |
| USA Gold records | 2 |
| Flattened redemption transactions | 2 |

Additional observations:
- One invalid enrollment date was identified in the sample member data.
- The two redemption transactions did not find matching member records in the sample workbook data because the member IDs did not match.
- All nine sample member records were marked stale in the recorded run because their last-flight dates were more than 90 days before the run date.

These results describe the sample execution only; they are not performance benchmarks.

## 6. SQL DDL notebook

The `08_DDL_Scrips.dbc` archive contains a Databricks SQL notebook with initial Delta table definitions for:

- Raw member landing table
- Staging member table
- Australia, India, and USA member target tables
- Redemption transaction table
- Rejected member records table
- Data-quality summary table

The DDL statements create managed Delta table structures. The existing processing notebooks write their outputs to ADLS Delta paths. The DDL tables have been created separately and are not yet wired into the existing notebook pipeline.

## 7. Data quality and reject handling

The validation notebooks currently cover:
- Missing mandatory member fields
- Invalid enrollment dates
- Missing or invalid DOB values
- Duplicate member-name observations
- Redemption transaction ID and field checks
- Redemption records without a matching member profile

The reject-handling notebook identifies records that fail the mandatory member-field/date checks and prepares a rejected-record output.

## 8. Current implementation notes

- The current member ingestion implementation uses sample Excel workbooks for Australia, India, and USA.
- The assessment's sample flat-file layout is pipe-delimited with `H` header and `D` detail records; that input format has not yet been implemented in the current ingestion notebook.
- Country-specific Gold outputs currently exist for Australia, India, and USA.
- The current latest-record selection partitions by `Member_Name`. Names can identify different people, so this is a limitation of the current sample implementation.
- The country movement scenario has not yet been fully tested to confirm removal or history handling in the previous country table.
- The rejected-record Delta output still needs execution verification.
- The DDL tables are separate from the ADLS Delta paths currently used by the processing notebooks.
- The project has been run against a small sample dataset; large-scale throughput has not been benchmarked.

## 9. Repository contents

```text
Incubyte-SkyPoints-Data-Engineering/
├── Incubyte_Data_Engineering.dbc
├── 08_DDL_Scrips.dbc
└── README.md
```

The repository currently contains the Databricks archive exports and this README. Individual `.py` or `.ipynb` notebook exports, separate `.sql` files, tests, and additional documentation can be added as they are prepared.
