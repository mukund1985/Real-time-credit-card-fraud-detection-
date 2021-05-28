
# Real time credit card fraud detection
This pattern demonstrates GCP capabilities to predict whether a credit card transaction is fradulent or not in real time. This uses a streaming dataflow pipeline to
* consume incoming transaction details from Cloud Pub/Sub
* does data preprocessing (by calling Firestore for data enrichment using transaction history)
* invokes multiple ML models deployed on AI Platform
* stores the prediction results to BigQuery and 
* sends notification to another Pub/Sub topic when a transaction is predicted fraudlent for downstream consumptions

Refer the below architecture diagram
## Architecture

![Alt text](diagram.png?raw=true)

## About the dataset
Below are the details of BigQuery tables & models used for this pattern which are publicly available (The underlying source dataset is from Kaggle)

Project ID: qp-fraud-detection

Dataset name: cc_data

Tables:
* train_raw - Data used for ML model training
* test_raw - Data used for ML model evaluation
* simulation_data - Data to be used for realtime inferences
* train_simple - BQ view providing features for training simple model
* train_w_aggregates - BQ view providing features for training model with aggregates
* test_simple - BQ view providing test data for evaluating simple model
* test_w_aggregates - BQ view providing test data for evaluating model with aggregates
* demographics - Data comprising customer demographics like name, gender, address

| Data       | Window start time     | Window end time |
| ------------- |-------------| --------------|
| Training      | 2019-01-01 00:00:18 UTC          | 2020-06-21 12:13:37 UTC            |
| Testing      | 2020-06-21 12:14:25 UTC          | 2020-12-20 01:44:05 UTC            |
| Inference      | 2020-12-20 01:42:27 UTC          | 2020-12-31 23:59:34 UTC            |

## Machine Learning Model
For this pattern, we opted for XGBoost model which worked really well while still retaining some level of [model explainability](https://cloud.google.com/bigquery-ml/docs/reference/standard-sql/bigqueryml-syntax-importance). We initially used the boosted tree classifier in BigQuery ML by using standard SQL to train the model and arrive at the probability score for each transaction. Due to the imbalanced nature of the dataset, we used F1 score and AUC to evaluate the performance of the model. After the initial evaluation, to boost the performance of the model we derived additional features from the dataset focusing on the frequency of the transactions and the average transaction amount over a period of time. As part of the solution, we have used predictions from both the models as part of the pipeline. The model using the standard features gives you relatively faster results, the other model uses the features derived from looking at historical data to make the predictions.

## About the models
In this example, we have two models
1. Model with standard features: This uses the features which are present in the dataset and doesn't rely on any feature generation techniques
2. Model with aggregate fetures: Along with the provided features, this uses feature generation techniques to compute transaction frequency, average spend etc for a given credit card

    * trans_freq_24 - Number of transactions in the last 24 hours

    * trans_diff - Time difference between current transaction and last transaction in seconds
    
    * avg_spend_pw - Average transaction amount in the past 1 week
    
    * avg_spend_pm - Average transaction amount in the past 1 month

| Feature       | Derived     | Used in Model1 | Used in Model2 |
| ------------- |-------------| --------------:| --------------:|
| category      | No          | Yes            | Yes            |
| amt           | No          | Yes            | Yes            |
| gender        | No          | No            | No            |
| state         | No          | Yes            | Yes            |
| job           | No          | Yes            | Yes            |
| unix_time     | No          | Yes            | Yes            |
| city_pop      | No          | Yes            | Yes            |
| merchant      | No          | Yes            | Yes            |
| day           | Yes         | Yes            | Yes            |
| age           | Yes         | Yes            | Yes            |
| distance      | Yes         | Yes            | Yes            |
| trans_freq_24 | Yes         | No             | Yes            |
| trans_diff    | Yes         | No             | Yes            |
| avg_spend_pw  | Yes         | No             | Yes            |
| avg_spend_pm  | Yes         | No             | Yes            |
 

## Before you begin
* Select or create a Google Cloud project and make sure billing is enabled, required APIs (Compute Engine, Pub/Sub, BigQuery, Firestore, Dataflow, AI Platform etc) are enabled. Use the latest gcloud sdk version to avoid any errors.
* Ensure you have the following IAM permissions
  * roles/pubsub.editor
  * roles/storage.admin
  * roles/bigquery.dataEditor
  * roles/bigquery.jobUser
  * roles/ml.developer
  * roles/datastore.user
  * roles/dataflow.developer
  * roles/compute.viewer
* Replace the below placeholders and set these environment variables on your system
```
export PROJECT_ID='VALUE_HERE'
export DATASET='VALUE_HERE'
export OUTPUT_BQ_TABLE='VALUE_HERE'
export MODEL_NAME_WITHOUT_AGG='simplemodel'
export MODEL_NAME_WITH_AGG='model_w_agg'
export BUCKET_NAME='VALUE_HERE'
export AI_MODEL_NAME='VALUE_HERE'
export VERSION_NAME_WITHOUT_AGG='VALUE_HERE'
export VERSION_NAME_WITH_AGG='VALUE_HERE'
export FIRESTORE_COL_NAME='VALUE_HERE'
export PUBSUB_TOPIC_NAME='VALUE_HERE'
export PUBSUB_SUBSCRIPTION_NAME='VALUE_HERE'
export PUBSUB_NOTIFICATION_TOPIC='VALUE_HERE'
```
* Create a Pub/Sub topic and subscription to contain input transaction details
```
gcloud pubsub topics create $PUBSUB_TOPIC_NAME
gcloud pubsub subscriptions create $PUBSUB_SUBSCRIPTION_NAME --topic=$PUBSUB_TOPIC_NAME
```
* Create a GCS bucket to export the model to and to serve staging directories for the dataflow pipeline
```
gsutil mb gs://$BUCKET_NAME
```
* Create a Pub/Sub topic which acts a fraud notification channel
```
gcloud pubsub topics create $PUBSUB_NOTIFICATION_TOPIC
```
* Create the output BQ dataset and table to hold transactions, predictions and confidence scores
```
bq --location=US mk --dataset $PROJECT_ID:$DATASET
bq mk --table $DATASET.$OUTPUT_BQ_TABLE utilities/output_schema.json
```

## BigQuery View definitions
The queries used for creating the publicly available views are provided below for reference.
### Training Data for simple model
```
CREATE VIEW `qp-fraud-detection.cc_data.train_simple`  AS (
SELECT 
      EXTRACT (dayofweek FROM trans_date_trans_time) AS day,
      DATE_DIFF(EXTRACT(DATE FROM trans_date_trans_time),dob, YEAR) AS age,
      ST_DISTANCE(ST_GEOGPOINT(long,lat), ST_GEOGPOINT(merch_long, merch_lat)) AS distance,
      category,
      amt,
      state,
      job,
      unix_time,
      city_pop,
      merchant,
      is_fraud
  FROM (
    SELECT * EXCEPT(cc_num)
    FROM `qp-fraud-detection.cc_data.train_raw`  t1
    LEFT JOIN `qp-fraud-detection.cc_data.demographics` t2 ON t1.cc_num =t2.cc_num))
```
### Training Data for model with aggregates
```
CREATE VIEW `qp-fraud-detection.cc_data.train_w_aggregates`  AS (
SELECT  
    EXTRACT (dayofweek FROM trans_date_trans_time) AS day,
    DATE_DIFF(EXTRACT(DATE FROM trans_date_trans_time),dob, YEAR) AS age,
    ST_DISTANCE(ST_GEOGPOINT(long,lat), ST_GEOGPOINT(merch_long, merch_lat)) AS distance,
    TIMESTAMP_DIFF(trans_date_trans_time, last_txn_date , MINUTE) AS trans_diff, 
    AVG(amt) OVER(
                PARTITION BY cc_num
                ORDER BY unix_time
                -- 1 week is 604800 seconds
                RANGE BETWEEN 604800 PRECEDING AND 1 PRECEDING) AS avg_spend_pw,
    AVG(amt) OVER(
                PARTITION BY cc_num
                ORDER BY unix_time
                -- 1 month(30 days) is 2592000 seconds
                RANGE BETWEEN 2592000 PRECEDING AND 1 PRECEDING) AS avg_spend_pm,
    COUNT(*) OVER(
                PARTITION BY cc_num
                ORDER BY unix_time
                -- 1 day is 86400 seconds
                RANGE BETWEEN 86400 PRECEDING AND 1 PRECEDING ) AS trans_freq_24,
    category,
    amt,
    state,
    job,
    unix_time,
    city_pop,
    merchant,
    is_fraud
  FROM (
          SELECT t1.*,t2.* EXCEPT(cc_num),
              LAG(trans_date_trans_time) OVER (PARTITION BY t1.cc_num ORDER BY trans_date_trans_time ASC) AS last_txn_date,
          FROM  `qp-fraud-detection.cc_data.train_raw`  t1
          LEFT JOIN  `qp-fraud-detection.cc_data.demographics`  t2 ON t1.cc_num =t2.cc_num))
```
### Testing Data for simple model
```
CREATE VIEW `qp-fraud-detection.cc_data.test_simple` AS (
SELECT 
      EXTRACT (dayofweek FROM trans_date_trans_time) AS day,
      DATE_DIFF(EXTRACT(DATE FROM trans_date_trans_time),dob, YEAR) AS age,
      ST_DISTANCE(ST_GEOGPOINT(long,lat), ST_GEOGPOINT(merch_long, merch_lat)) AS distance,
      category,
      amt,
      state,
      job,
      unix_time,
      city_pop,
      merchant,
      is_fraud
  FROM (
    SELECT * EXCEPT(cc_num)
    FROM `qp-fraud-detection.cc_data.test_raw` t1
    LEFT JOIN `qp-fraud-detection.cc_data.demographics` t2 ON t1.cc_num =t2.cc_num))
```
### Testing Data for model with aggregates
```
CREATE VIEW `qp-fraud-detection.cc_data.test_w_aggregates`  AS (
WITH t1 as (
SELECT *, 'train' AS split FROM  `qp-fraud-detection.cc_data.train_raw` 
UNION ALL 
SELECT *, 'test' AS split FROM  `qp-fraud-detection.cc_data.test_raw`
),
v2 AS (
  SELECT t1.*,t2.* EXCEPT(cc_num),
              LAG(trans_date_trans_time) OVER (PARTITION BY t1.cc_num ORDER BY trans_date_trans_time ASC) AS last_txn_date,
            FROM t1
            LEFT JOIN  `qp-fraud-detection.cc_data.demographics` t2 ON t1.cc_num = t2.cc_num
),
v3 AS (
  SELECT
        EXTRACT (dayofweek FROM trans_date_trans_time) as day,
        DATE_DIFF(EXTRACT(DATE FROM trans_date_trans_time),dob, YEAR) AS age,
        ST_DISTANCE(ST_GEOGPOINT(long,lat), ST_GEOGPOINT(merch_long, merch_lat)) as distance,
        TIMESTAMP_DIFF(trans_date_trans_time, last_txn_date , MINUTE) AS trans_diff, 
        Avg(amt) OVER(
                    PARTITION BY cc_num
                    ORDER BY unix_time
                    RANGE BETWEEN 604800 PRECEDING AND 1 PRECEDING) AS avg_spend_pw,
        Avg(amt) OVER(
                    PARTITION BY cc_num
                    ORDER BY unix_time
                    RANGE BETWEEN 2592000 PRECEDING AND 1 PRECEDING) AS avg_spend_pm,
        count(*) OVER(
                    PARTITION BY cc_num
                    ORDER BY unix_time
                    RANGE BETWEEN 86400 PRECEDING AND 1 PRECEDING) AS trans_freq_24,
        category,
        amt,
        state,
        job,
        unix_time,
        city_pop,
        merchant,
        is_fraud,
        split
    FROM v2
)
SELECT * EXCEPT(split) FROM v3 WHERE split='test')
```
## ML Model Training
Do substitution for variables PROJECT_ID, DATASET and MODEL_NAME_WITHOUT_AGG
### Model 1 (without aggregates)
```
CREATE OR REPLACE MODEL 
  `[PROJECT_ID].[DATASET].[MODEL_NAME_WITHOUT_AGG]`
OPTIONS (
  model_type ='BOOSTED_TREE_CLASSIFIER',
  NUM_PARALLEL_TREE =8,
  MAX_ITERATIONS = 50,
  input_label_cols=["is_fraud"]
) AS
SELECT 
  *
FROM 
  `qp-fraud-detection.cc_data.train_simple`
```


### Model 2 (with aggregates)
```
CREATE OR REPLACE MODEL 
  `[PROJECT_ID].[DATASET].[MODEL_NAME_WITH_AGG]`
OPTIONS (
  model_type ='BOOSTED_TREE_CLASSIFIER',
  NUM_PARALLEL_TREE =8,
  MAX_ITERATIONS = 50,
  input_label_cols=["is_fraud"]
) AS
SELECT
  *
FROM 
  `qp-fraud-detection.cc_data.train_w_aggregates`
```
         

## Model Evaluation
Test both the models using ML.EVALUATE
```
SELECT 
  "simplemodel" AS model_name,
  *
FROM
  ML.EVALUATE(
    MODEL `[PROJECT_ID].[DATASET].[MODEL_NAME_WITHOUT_AGG]`,
    (SELECT * FROM `qp-fraud-detection.cc_data.test_simple`))
UNION ALL
SELECT
  "model_w_aggregates" AS model_name,
  *
FROM
  ML.EVALUATE(
    MODEL `[PROJECT_ID].[DATASET].[MODEL_NAME_WITH_AGG]`,
    (SELECT * FROM  `qp-fraud-detection.cc_data.test_w_aggregates`))
```

## Exporting model to AI Platform
BQML is good for quickly developing ML models using simple SQL statements and testing them out. For production implementations, it's ideal to export the model to AI platform and host it on AI Platform for scalability and customizing the deployment environment.
### Exporting model to GCS
1. Model 1 (without aggregates)
```
bq extract --destination_format ML_XGBOOST_BOOSTER -m $PROJECT_ID:$DATASET.$MODEL_NAME_WITHOUT_AGG gs://$BUCKET_NAME/$MODEL_NAME_WITHOUT_AGG
```
2. Model 2 (with aggregates)
```
bq extract --destination_format ML_XGBOOST_BOOSTER -m  $PROJECT_ID:$DATASET.$MODEL_NAME_WITH_AGG gs://$BUCKET_NAME/$MODEL_NAME_WITH_AGG
```
            

### Creating Model in AI Platform
1. Create a model in AI platform to hold both our versions (without aggregates and with aggregates)
```
gcloud ai-platform models create $AI_MODEL_NAME --region global
```
2. Create 1 model version for model without aggregates
```
gcloud beta ai-platform versions create $VERSION_NAME_WITHOUT_AGG \
--model=$AI_MODEL_NAME \
--origin=gs://$BUCKET_NAME/$MODEL_NAME_WITHOUT_AGG \
--package-uris=gs://$BUCKET_NAME/$MODEL_NAME_WITHOUT_AGG/xgboost_predictor-0.1.tar.gz \
--prediction-class=predictor.Predictor \
--runtime-version=1.15 \
--region global
```
3. Create 1 model version for model with aggregates
```
gcloud beta ai-platform versions create $VERSION_NAME_WITH_AGG \
--model=$AI_MODEL_NAME \
--origin=gs://$BUCKET_NAME/$MODEL_NAME_WITH_AGG \
--package-uris=gs://$BUCKET_NAME/$MODEL_NAME_WITH_AGG/xgboost_predictor-0.1.tar.gz \
--prediction-class=predictor.Predictor \
--runtime-version=1.15 \
--region global
```
### Testing Online Predictions
To test the deployed models, send online prediction requests using the sample inputs [input_wo_aggregates.json](sample_inputs/input_wo_aggregates.json) and [input_w_aggregates.json](sample_inputs/input_w_aggregates.json)
1. Model 1 (WITHOUT AGGREGATES)
```
gcloud ai-platform predict --model $AI_MODEL_NAME \
--version $VERSION_NAME_WITHOUT_AGG \
--region global \
--json-instances sample_inputs/input_wo_aggregates.json
```

2. Model 2 (WITH AGGREGATES)
```
gcloud ai-platform predict --model $AI_MODEL_NAME \
--version $VERSION_NAME_WITH_AGG \
--region global \
--json-instances sample_inputs/input_w_aggregates.json
```
### Load transaction history into Firestore
The past transaction history (from train_raw and test_raw BQ tables) needs to be loaded into Firestore so the dataflow pipeline can do lookup while the simulated transactions (simulation_data BQ table) is used for real time inference. Lauch the script [load_to_firestore.py](utilities/load_to_firestore.py) to fetch the train and test transactions from BigQuery and to load these documents into Firestore. Ensure you have google-cloud-bigquery and google-cloud-firestore python modules installed.
```
python3 utilities/load_to_firestore.py $PROJECT_ID $FIRESTORE_COL_NAME
```
### Launch the Dataflow pipeline
Run the below command to launch a streaming dataflow pipeline for ML inference
```
python3 inference_pipeline.py \
--project $PROJECT_ID \
--firestore-project $PROJECT_ID \
--subscription-name $PUBSUB_SUBSCRIPTION_NAME \
--firestore-collection $FIRESTORE_COL_NAME \
--dataset-id $DATASET \
--table-name $OUTPUT_BQ_TABLE \
--model-name $AI_MODEL_NAME \
--model-with-aggregates $VERSION_NAME_WITH_AGG \
--model-without-aggregates $VERSION_NAME_WITHOUT_AGG \
--fraud-notification-topic $PUBSUB_NOTIFICATION_TOPIC \
--staging_location gs://$BUCKET_NAME/dataflow/staging \
--temp_location gs://$BUCKET_NAME/dataflow/tmp/ \
--region us-east1 \
--streaming \
--runner DataflowRunner \
--job_name predict-fraudulent-transactions \
--requirements_file requirements.txt
```

### Simulate real time transactions
Launch the python script [bq_to_pubsub.py](utilities/bq_to_pubsub.py) which reads from BigQuery table simulation_data and ingests records into input pubsub topic to simulate real time traffic on the Dataflow pipeline

```
python3 utilities/bq_to_pubsub.py $PROJECT_ID $PUBSUB_TOPIC_NAME
```

For production grade pipelines, one need to consider AI Platform quotas such as [number of Online Predictions per minute](https://cloud.google.com/ai-platform/prediction/docs/quotas#online_prediction_requests)
  
### Monitoring Dashboard
We have created a sample dashboard to get started on monitoring few of the important metrics, refer to [dashboard template](utilities/dashboard_template.json). Replace your project ID or number in the template and make a [REST API call](https://cloud.google.com/monitoring/api/ref_v3/rest/v1/projects.dashboards/create) to create the dashboard in your project.
  

### Clean Up
To avoid incurring charges to your Google Cloud account, ensure you terminate the resources used in this pattern




 







