# Power-Plant-Analytics
Power Plant Energy Prediction Project (Databricks Lakehouse)
# ⚡ Power Plant Energy Output Prediction — Databricks Lakehouse Project

This project demonstrates an **end-to-end Lakehouse pipeline** built on **Databricks (Free Edition)** using  
**Unity Catalog, Volumes, Delta Lake, Auto Loader**, and **PySpark ML** to predict power plant energy output.

To keep the workflow compatible with the Free Edition, the project avoids MLflow Experiments and Public DBFS root paths.  
All storage uses **Unity Catalog Volumes**, and all ML models are saved manually instead of using MLflow.

---

# 📊 Project Overview

This project uses the public dataset:


The goal is to:

1. Build a **Medallion Architecture** (Bronze → Silver → Gold)
2. Use **Auto Loader** to ingest raw CSV files into Delta
3. Perform **data cleaning, casting, and feature engineering**
4. Compute **analytics KPIs** in the Gold layer
5. Train a **Random Forest Regression model** (PySpark ML)
6. Save model artifacts inside a **Unity Catalog Volume**
7. Store predictions in Delta tables
8. Automate the pipeline using **Databricks Workflows**
9. Visualize KPIs using Databricks SQL Dashboards

This pipeline simulates real industrial use cases such as **energy forecasting**, **efficiency analysis**, and **operational analytics**.

---

# 🧱 Architecture

           ┌────────────────────────────┐
           │   dbfs:/databricks-datasets│
           │       /power-plant/        │
           └──────────────┬─────────────┘
                          │ Auto Loader (cloudFiles)
                          ▼
┌────────────────────────────────────────────────────────┐
│ BRONZE LAYER │
│ Raw ingestion → Delta (schema inference + checkpoints) │
│ Volume storage: volume://powerplant.core.storage/ │
└────────────────────────────────────────────────────────┘
▼
┌────────────────────────────────────────────────────────┐
│ SILVER LAYER │
│ Data cleaning, casting, feature engineering │
│ SQL transformations in Unity Catalog │
└────────────────────────────────────────────────────────┘
▼
┌────────────────────────────────────────────────────────┐
│ GOLD LAYER │
│ Business KPIs, aggregations, temperature curves, │
│ humidity effects, correlation metrics │
└────────────────────────────────────────────────────────┘
▼
┌────────────────────────────────────────────────────────┐
│ ML MODEL (PySpark ML) │
│ Random Forest Regression → Predict energy output │
│ Model saved to UC Volume │
└────────────────────────────────────────────────────────┘
▼
┌────────────────────────────────────────────────────────┐
│ PREDICTIONS (Delta Table + Dashboard) │
└────────────────────────────────────────────────────────┘

📥 1. Bronze Layer — Auto Loader Ingestion

display(dbutils.fs.ls("dbfs:/databricks-datasets/power-plant/data/"))

%sql
CREATE CATALOG IF NOT EXISTS power_plant;
CREATE SCHEMA IF NOT EXISTS power_plant.core;
CREATE VOLUME IF NOT EXISTS power_plant.core.storage;

bronze_checkpoint = "volume://power_plant.core.storage/bronze/checkpoints"
bronze_schema = "volume://power_plant.core.storage/bronze/schemas"
Storage_for_ML_artifacts = "volume://power_plant.core.storage/ml/"


from pyspark.sql.functions import *

bronze_checkpoint = "/Volumes/power_plant/core/storage/bronze/checkpoints"
bronze_schema = "/Volumes/power_plant/core/storage/bronze/schemas"

source_path = "dbfs:/databricks-datasets/power-plant/data/"


df_stream = (spark.readStream.format("cloudFiles")
    .option("cloudFiles.format", "csv")  # Use CSV format
    .option("sep", "\t")  # Tab delimiter for TSV files
    .option("header", "true")
    .option("cloudFiles.inferColumnTypes", "true")
    .option("cloudFiles.schemaLocation", bronze_schema)
    .load(source_path)
)

(df_stream.writeStream.format("delta")
    .option("checkpointLocation", bronze_checkpoint)
    .trigger(once=True)
    .toTable("power_plant.core.bronze_powerplant")
)

🧼 2. Silver Layer — Clean & Feature Engineer

Notebook: 02_silver_transform.sql

USE CATALOG power_plant;
USE SCHEMA core;

CREATE OR REPLACE TABLE silver_powerplant AS
SELECT
  CAST(AT AS DOUBLE)  AS ambient_temp_c,
  CAST(V AS DOUBLE)   AS exhaust_vacuum_cm_hg,
  CAST(AP AS DOUBLE)  AS ambient_pressure_mbar,
  CAST(RH AS DOUBLE)  AS relative_humidity,
  CAST(PE AS DOUBLE)  AS net_output_mw,
  (AP / 1013.25)                 AS pressure_ratio,
  (RH / 100.0)                   AS humidity_fraction,
  ambient_temp_c * humidity_fraction AS temp_humidity_interaction
FROM bronze_powerplant

WHERE AT IS NOT NULL
  AND V IS NOT NULL
  AND AP IS NOT NULL
  AND RH IS NOT NULL
  AND PE IS NOT NULL;


Gold Layer:

USE CATALOG power_plant;
USE SCHEMA core;

CREATE OR REPLACE TABLE silver_powerplant AS
SELECT
  CAST(AT AS DOUBLE)  AS ambient_temp_c,
  CAST(V AS DOUBLE)   AS exhaust_vacuum_cm_hg,
  CAST(AP AS DOUBLE)  AS ambient_pressure_mbar,
  CAST(RH AS DOUBLE)  AS relative_humidity,
  CAST(PE AS DOUBLE)  AS net_output_mw,
  (AP / 1013.25)                 AS pressure_ratio,
  (RH / 100.0)                   AS humidity_fraction,
  ambient_temp_c * humidity_fraction AS temp_humidity_interaction
FROM bronze_powerplant

WHERE AT IS NOT NULL
  AND V IS NOT NULL
  AND AP IS NOT NULL
  AND RH IS NOT NULL
  AND PE IS NOT NULL;


  create or replace table gold_output_by_humidity as
select 
    case 
    when relative_humidity <30 THEN 'Dry (<30%)'
    when relative_humidity <60 Then 'Normal (30-60%)'
    ELSe 'humid( >60%)'
  END As humidity_band,
   count (*) as samples,
   avg(net_output_mw) as avg_output_mw
   from power_plant.core.silver_powerplant
  Group by
   case 
   when relative_humidity <30 then 'Dry (<30%)'
   when relative_humidity <60 Then 'Normal (30-60%)'
    ELSe 'humid( >60%)'
  END;

CREATE OR REPLACE TABLE gold_temp_vacuum_heatmap AS
SELECT
  ROUND(ambient_temp_c, 0)      AS temp_bucket,
  ROUND(exhaust_vacuum_cm_hg, 0) AS vacuum_bucket,
  AVG(net_output_mw)            AS avg_output_mw,
  COUNT(*)                      AS samples
FROM power_plant.core.silver_powerplant
GROUP BY
  ROUND(ambient_temp_c, 0),
  ROUND(exhaust_vacuum_cm_hg, 0);


  from pyspark.ml.feature import VectorAssembler
from pyspark.ml.regression import RandomForestRegressor
from pyspark.ml import Pipeline

df = spark.table("powerplant.core.silver_powerplant")

assembler = VectorAssembler(
    inputCols=["ambient_temp","vacuum","pressure","humidity",
               "pressure_ratio","humidity_fraction","temp_humidity_interaction"],
    outputCol="features"
)

rf = RandomForestRegressor(labelCol="power_output", featuresCol="features")
pipeline = Pipeline(stages=[assembler, rf])

model = pipeline.fit(df)

model.write().overwrite().save("volume://powerplant.core.storage/ml/rf_power_model")


from pyspark.ml import PipelineModel

model = PipelineModel.load("volume://powerplant.core.storage/ml/rf_power_model")

df = spark.table("powerplant.core.silver_powerplant")
pred = model.transform(df)

pred.write.format("delta").mode("overwrite").saveAsTable("powerplant.core.predictions")
