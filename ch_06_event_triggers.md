# Up and Running: Data Engineering on the Google Cloud Platform
The completely free E-Book for setting up and running a Data Engineering stack on Google Cloud Platform.

NOTE: This book is currently incomplete. If you find errors or would like to fill in the gaps, read the [Contributions section](https://github.com/Nunie123/data_engineering_on_gcp_book#user-content-contributions).

## Table of Contents
[Preface](https://github.com/Nunie123/data_engineering_on_gcp_book) <br>
[Chapter 1: Setting up a GCP Account](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_01_gcp_account.md) <br>
[Chapter 2: Setting up Batch Processing Orchestration with Composer and Airflow](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_02_orchestration.md) <br>
[Chapter 3: Building a Data Lake with Google Cloud Storage (GCS)](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_03_data_lake.md) <br>
[Chapter 4: Building a Data Warehouse with BigQuery](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_04_data_warehouse.md) <br>
[Chapter 5: Setting up DAGs in Composer and Airflow](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_05_dags.md) <br>
**Chapter 6: Setting up Event-Triggered Pipelines with Cloud Functions** <br>
[Chapter 7: Parallel Processing with Dataproc and Spark](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_07_parallel_processing.md) <br>
[Chapter 8: Streaming Data with Pub/Sub](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_08_streaming.md) <br>
[Chapter 9: Managing Credentials with Google Secret Manager](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_09_secrets.md) <br>
[Chapter 10: Infrastructure as Code with Terraform](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_10_infrastructure_as_code.md) <br>
[Chapter 11: Deployment Pipelines with Cloud Build](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_11_deployment_pipelines.md) <br>
[Chapter 12: Monitoring and Alerting](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_12_monitoring.md) <br>
[Chapter 13: Up and Running - Building a Complete Data Engineering Infrastructure](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_13_up_and_running.md) <br>
[Appendix A: Example Code Repository](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/appendix_a_example_code/README.md)


---

# [Chapter 6](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_06_event_triggers.md): Setting up Event-Triggered Pipelines with Cloud Functions

In [Chapter 5](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_05_dags.md) I demonstrated how to set up a Data Pipeline that ran on a schedule (or by manually triggering through the Web UI). Airflow has a lot of flexibility for when and how often a DAG should run, but sometimes you don't want your DAG to run on a schedule. Suppose your organization's Data Science team periodically generates a CSV file with valuable information, and you want to ingest that file into your Data Warehouse as soon as it is available. You can't ingest on a schedule, because you don't know when the file will be uploaded. What we can do instead is set up a Cloud Function to listen for a new file to be uploaded, and once a new file is detected it can kick off the Data Pipeline for ingesting that file.

## Overview of Cloud Functions
GCP's Cloud Functions is a "serverless" code execution service. It allows predefined code to be executed when triggered by an event, such as GCS events, HTTP events, and Pub/Sub events. We'll focus on GCS events in this chapter, as responding to a new file being uploaded to a GCS Bucket is a common task for Data Engineers, but Cloud Functions has much more functionality than what I'll cover. Additionally, I'll talk about Pub/Sub in [Chapter 8: Streaming Data with Pub/Sub](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_08_streaming.md) as I discuss streaming Data Pipelines.

## Setting Up an Event-Triggered Pipeline

In this chapter we'll be setting up a process where a file is uploaded to a GCS Bucket, which triggers a function that loads that file into a BigQuery table.

### Prep Work
We need to do a few things to make sure our Cloud Function works, once we set it up.

Let's start by creating a Bucket that will trigger the Cloud Function:
``` bash
> gsutil mb gs://de-book-trigger-function
```

We need somewhere for the data to go, so let's create a BigQuery Dataset and table:
``` bash
> bq mk --dataset food
> bq mk --table food.food_ranks food_name:STRING,ranking:INT64
```

Now let's enable the Cloud Function API on GCP. In [Chapter 2](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_02_orchestration.md) I showed how to enable the Composer API through the console, but we can also enable services through the command line:
``` bash
> gcloud services enable cloudfunctions.googleapis.com
```

### Building and Deploying a Cloud Function
Now let's create a function that will be executed once a new file is loaded into the GCS Bucket. It's important to note that the file must be called `main.py`. To differentiate this function file from others GCP recommends you use subdirectories such as `/functions/my_function_name/main.py`.
``` python
def load_to_bq(event, context):
    import os
    from google.cloud import bigquery

    table_id = 'de-book-dev.food.food_ranks'
    uri = os.path.join('gs://', event['bucket'], event['name'])
    schema = [
        bigquery.SchemaField('food_name', 'STRING', mode='NULLABLE'),
        bigquery.SchemaField('ranking', 'INT64', mode='NULLABLE')
    ]
    job_config = bigquery.LoadJobConfig(
        schema=schema,
        source_format=bigquery.SourceFormat.CSV,
        skip_leading_rows=1
    )
    client = bigquery.Client()
    load_job = client.load_table_from_uri(
            uri,
            table_id,
            job_config=job_config
        )
    load_job.result()
```
In the above function we used the library `google.cloud.bigquery`, which means we'll need to install it into our environment. Fortunately, we can simply create a `requirements.txt` file in the same directory as our `main.py` file:
``` text
google-cloud-bigquery==2.2.0
```

Now let's deploy the function. When we run the below command we should be in the same directory as the `main.py` file. It may take a few minutes for the function to deploy.
``` bash
> gcloud functions deploy load_to_bq \
    --runtime python37 \
    --trigger-resource gs://de-book-trigger-function \
    --trigger-event google.storage.object.finalize
```

### Testing the Cloud Function

We've got our Cloud Function up and running, so let's test it out. First let's create a file to upload that will trigger the function:
``` bash
> echo 'food_name, ranking
pizza, 1
cheeseburgers, 2
hot dogs, 3
nachos, 4
tacos, 5' > yum.csv
```
Now we can upload the file to our Bucket and trigger our function:
``` bash
> gsutil cp yum.csv gs://de-book-trigger-function
```
If all went as planned we should now see our data in our table:
``` bash
> bq query 'select * from food.food_ranks'
+---------------+---------+
|   food_name   | ranking |
+---------------+---------+
| pizza         |       1 |
| cheeseburgers |       2 |
| hot dogs      |       3 |
| nachos        |       4 |
| tacos         |       5 |
+---------------+---------+
```

### Combining Cloud Functions with Composer
One architecture design that can work well is having all of your batch pipelines managed by Composer. In the above Cloud Function we loaded the data directly into BigQuery using the `google.cloud.bigquery` Python library. However, another option would be to have the Cloud Function trigger a DAG, which will then perform the loading and transformation Tasks. You can read more about setting up a Cloud Function to trigger a DAG [here](https://cloud.google.com/composer/docs/how-to/using/triggering-with-gcf).

### Cleaning Up
Cloud Function [pricing](https://cloud.google.com/functions/pricing) is done by use, so our work this chapter is not breaking the bank. Nonetheless, it's good to take down what you're not using. Note that the Cloud Functions service created some storage buckets for it to use, in addition to the Bucket we created, that should also be removed.
``` bash
> gcloud functions delete load_to_bq 

> bq rm --dataset -r food

> gsutil ls
gs://de-book-trigger-function/
gs://gcf-sources-204024561480-us-central1/
gs://us.artifacts.de-book-dev.appspot.com/

> gsutil rm -r gs://de-book-trigger-function
> gsutil rm -r gs://us.artifacts.de-book-dev.appspot.com/
```

A note about `gs://gcf-sources-204024561480-us-central1`: This bucket is used as the location where Cloud Functions are deployed in GCP. It takes the format "gcf-sources-<PROJECT_NUMBER>-<REGION>. If you delete this bucket, you may find that future deployments of Cloud Functions fail. You can manually create a bucket with this name to resolve the issue.

---

Next Chapter: [Chapter 7: Parallel Processing with DataProc and Spark](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_07_parallel_processing.md)