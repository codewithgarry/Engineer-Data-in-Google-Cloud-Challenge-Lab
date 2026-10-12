# Engineer Data in Google Cloud: Challenge Lab (GSP327)

![Google Cloud](https://img.shields.io/badge/Google_Cloud-4285F4?logo=googlecloud&logoColor=white)
![Lab](https://img.shields.io/badge/Lab-GSP327-34A853)
![BigQuery ML](https://img.shields.io/badge/BigQuery_ML-669DF6?logo=googlebigquery&logoColor=white)

Reference solution for the Google Cloud challenge lab **Engineer Data in Google Cloud** (GSP327).

## 📖 Overview

You build an end-to-end **taxi fare prediction** pipeline entirely in BigQuery:

1. Clean and sample raw NYC taxi trips into a training table.
2. Train a linear regression model with feature engineering inside a `TRANSFORM` clause.
3. Run batch predictions and save them to a table.

## 🧰 Services used

BigQuery · BigQuery ML · Geography functions (`ST_Distance`) · Dataprep (optional in the lab)

## 🚀 Solution

### Task 1: Clean the training data

Keep realistic trips only: positive distance, a minimum fare, at least one passenger and coordinates within the New York area. `RAND() < 0.001` takes a small random sample.

```sql
CREATE OR REPLACE TABLE taxirides.taxi_training_data AS
SELECT
  (tolls_amount + fare_amount) AS fare_amount,
  pickup_datetime,
  pickup_longitude  AS pickuplon,
  pickup_latitude   AS pickuplat,
  dropoff_longitude AS dropofflon,
  dropoff_latitude  AS dropofflat,
  passenger_count   AS passengers
FROM taxirides.historical_taxi_rides_raw
WHERE RAND() < 0.001
  AND trip_distance > 0
  AND fare_amount >= 2.5
  AND pickup_longitude  BETWEEN -78 AND -70
  AND dropoff_longitude BETWEEN -78 AND -70
  AND pickup_latitude   BETWEEN 37 AND 45
  AND dropoff_latitude  BETWEEN 37 AND 45
  AND passenger_count > 0;
```

### Task 2: Train the model

The `TRANSFORM` clause stores feature engineering with the model, so prediction inputs need only raw columns.

```sql
CREATE OR REPLACE MODEL taxirides.fare_model
TRANSFORM (
  * EXCEPT (pickup_datetime),
  ST_Distance(ST_GeogPoint(pickuplon, pickuplat),
              ST_GeogPoint(dropofflon, dropofflat))   AS euclidean,
  CAST(EXTRACT(DAYOFWEEK FROM pickup_datetime) AS STRING) AS dayofweek,
  CAST(EXTRACT(HOUR      FROM pickup_datetime) AS STRING) AS hourofday
)
OPTIONS (input_label_cols = ['fare_amount'], model_type = 'linear_reg') AS
SELECT * FROM taxirides.taxi_training_data;
```

Check that the RMSE is below the lab's target:

```sql
SELECT SQRT(mean_squared_error) AS rmse
FROM ML.EVALUATE(MODEL taxirides.fare_model);
```

### Task 3: Batch prediction

```sql
CREATE OR REPLACE TABLE taxirides.`2015_fare_amount_predictions` AS
SELECT *
FROM ML.PREDICT(
  MODEL taxirides.fare_model,
  (SELECT * FROM taxirides.report_prediction_data)
);
```

---

## ⚠️ Notes

- Resource names, zones, regions and ports are generated per lab session. Always use the values shown in **your** lab instructions.
- These steps were written when the lab was live. Google revises labs over time, so the current version may differ.

## 📌 Disclaimer

This repository is a study reference. Attempt the lab on your own first, then compare approaches. It is not affiliated with or endorsed by Google.

## 👤 Author

**Girish Sharma** · Cloud & DevOps Engineer · [GitHub](https://github.com/codewithgarry) · [LinkedIn](https://www.linkedin.com/in/codewithgarry/)
