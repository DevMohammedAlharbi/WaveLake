<div align="center">

# 🎧 WaveLake

### A music streaming lakehouse on Azure and Databricks

Streaming data (users, tracks and listening events) moves from **Azure SQL Database** into **ADLS Gen2** with change-aware incremental loads, then through **Databricks** into a **Star Schema** with SCD Type 2 history and built-in data quality rules.

![Azure Data Factory](https://img.shields.io/badge/Azure_Data_Factory-0078D4?style=for-the-badge&logo=microsoftazure&logoColor=white)
![Databricks](https://img.shields.io/badge/Databricks-FF3621?style=for-the-badge&logo=databricks&logoColor=white)
![PySpark](https://img.shields.io/badge/PySpark-E25A1C?style=for-the-badge&logo=apachespark&logoColor=white)
![Delta Lake](https://img.shields.io/badge/Delta_Lake-00ADD4?style=for-the-badge&logo=delta&logoColor=white)
![Unity Catalog](https://img.shields.io/badge/Unity_Catalog-1B3139?style=for-the-badge&logo=databricks&logoColor=white)
![Jinja](https://img.shields.io/badge/Jinja-B41717?style=for-the-badge&logo=jinja&logoColor=white)

</div>

---

## 📌 Overview

WaveLake models the data behind a music streaming app: who listens, what they listen to, and when. The goal was to build the full path from an operational database to analytics tables, and to make every step incremental, reusable and safe to rerun.

## 🏗️ Architecture

```mermaid
flowchart LR
    SQL[(🗄️ Azure SQL DB<br/>users, tracks, streams)] -->|ADF incremental CDC| BR[(🥉 Bronze<br/>ADLS Gen2)]
    BR -->|Autoloader + PySpark| SV[(🥈 Silver<br/>Unity Catalog)]
    SV -->|Lakeflow Declarative Pipelines| GD[(🥇 Gold<br/>Star Schema, SCD 2)]
    ADF[Azure Data Factory] -. alerts .-> LA[📧 Logic Apps]
    DAB[Databricks Asset Bundle] -. deploys .-> SV
    DAB -. deploys .-> GD
```

## ✨ Highlights

**Ingestion with Azure Data Factory**
- A single parameterized pipeline loops over every source table with `ForEach`, so new tables need configuration, not new pipelines.
- Change-aware loads: a `Lookup` reads the last CDC timestamp, `Copy` pulls only newer rows, `Get Metadata` checks the output, and `If Condition` either updates the CDC marker or deletes empty files.
- A `Web` activity calls a **Logic App** to send alerts.

**Transformation with Databricks**
- **Silver:** PySpark with **Autoloader** for incremental file ingestion, governed by **Unity Catalog**, with shared helper functions in `utils/`.
- **Jinja templates** generate reusable SQL instead of repeating the same joins.
- **Gold:** **Lakeflow Declarative Pipelines** build `DimUser`, `DimTrack`, `DimDate` and `FactStream`. Expectations drop invalid rows (for example, null keys), and `auto CDC` flows keep **SCD Type 2** history.
- Packaged as a **Databricks Asset Bundle** with separate `dev` and `prod` targets, so the gold pipeline is deployed as code.

## ⭐ Gold data model

```mermaid
erDiagram
    FactStream }o--|| DimUser : user_id
    FactStream }o--|| DimTrack : track_id
    FactStream {
        int stream_id
        int user_id
        int track_id
        int listen_duration
    }
    DimUser {
        int user_id
        string user_name
        string subscription_type
        timestamp __START_AT
        timestamp __END_AT
    }
    DimTrack {
        int track_id
        string track_name
        string artist_name
    }
```

`DimDate` completes the model as a calendar dimension. `DimUser` and `DimTrack` keep full history (SCD Type 2), and `FactStream` is upserted on `stream_id`.

## 📁 Repository structure

```
.
├── pipeline/              # ADF: incremental_ingestion, incremental_loop
├── dataset/               # parameterized SQL, Parquet and JSON datasets
├── linkedService/         # Azure SQL, ADLS Gen2
└── mybundle_spotify/      # Databricks Asset Bundle
    ├── src/silver/        # silver dimensions (PySpark + Autoloader)
    ├── src/gold/dlt/      # Lakeflow pipelines: dims, fact, SCD 2
    ├── Jinja/             # Jinja-templated SQL
    ├── utils/             # shared transformations
    ├── resources/         # job + pipeline definitions
    └── databricks.yml     # bundle targets (dev / prod)
```

## 🚀 Deploy it yourself

1. Create Azure SQL Database, ADLS Gen2 (`bronze`, `silver`, `gold`), Azure Data Factory and a Databricks workspace with Unity Catalog.
2. Connect ADF to this repository through **Git configuration** and update the linked services with your own resources (keep secrets in **Azure Key Vault**).
3. Run `incremental_loop` to land the source tables in bronze.
4. Deploy the Databricks side with the bundle:

   ```bash
   cd mybundle_spotify
   databricks bundle validate
   databricks bundle deploy -t dev
   databricks bundle run wavelake_gold -t dev
   ```

## 🧠 What I learned

- Building incremental ingestion that survives reruns and empty loads.
- Splitting responsibilities: ADF moves data, Databricks transforms it.
- Declaring data quality and SCD Type 2 directly in Lakeflow pipelines.
- Packaging Databricks work as code with Asset Bundles.

---

<div align="center">

**Mohammed Alharbi** · Data Engineer

[Portfolio](https://devmohammedalharbi.github.io) · [LinkedIn](https://linkedin.com/in/mohammed-alharbi-1606a7388) · [GitHub](https://github.com/DevMohammedAlharbi)

</div>
