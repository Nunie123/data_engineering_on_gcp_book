# Up and Running: Data Engineering on the Google Cloud Platform
The completely free E-Book for setting up and running a Data Engineering stack on Google Cloud Platform.

NOTE: This book is currently incomplete. If you find errors or would like to fill in the gaps, read the [Contributions section](https://github.com/Nunie123/data_engineering_on_gcp_book#user-content-contributions).

## Table of Contents
[Preface](https://github.com/Nunie123/data_engineering_on_gcp_book) <br>
[Chapter 1: Setting up a GCP Account](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_1_gcp_account.md) <br>
[Chapter 2: Setting up Batch Processing Orchestration with Composer and Airflow](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_2_orchestration.md) <br>
[Chapter 3: Building a Data Lake with Google Cloud Storage (GCS)](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_3_data_lake.md) <br>
[Chapter 4: Building a Data Warehouse with BigQuery](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_4_data_warehouse.md) <br>
[Chapter 5: Setting up DAGs in Composer and Airflow](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_5_dags.md) <br>
[Chapter 6: Setting up Event-Triggered Pipelines with Cloud Functions](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_6_event_triggers.md) <br>
**Chapter 7: Parallel Processing with Dataproc and Spark** <br>
[Chapter 8: Streaming Data with Pub/Sub](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_8_streaming.md) <br>
[Chapter 9: Managing Credentials with Google Secret Manager](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_9_secrets.md) <br>
Chapter 10: Creating a Local Development Environment <br>
Chapter 11: Infrastructure as Code with Terraform <br>
Chapter 12: Continuous Integration with Jenkins <br>
Chapter 13: Monitoring and Alerting <br>
Appendix A: Example Code Repository


---

# Chapter 7: Parallel Processing with Dataproc and Spark
Dataproc is GCP's fully managed service for running Apache Spark. Spark is an open-source program that has a wide array of capabilities, such as machine learning and data streaming, but I'm going to show you how we can use Spark perform transformations on very large data files. Spark's Python API (called PySpark), will look familiar to you if you have used Python's Pandas library. However, the key difference between processing data in Pandas vs. Spark is that Pandas works entirely in memory on a single machine, whereas Spark is designed to work across multiple machines and can manage the data in memory and on disk.

Spark is operating on a Cluster of machines, and each machine is working in coordination to process a transformation job. This behavior works by default and allows Spark to perform massive parallel processing. That means we can quickly perform complex transformations on extremely large amounts of data.

A Dataproc Cluster behaves similarly to a Composer Environment, in that we issue commands to create the Cluster and GCP charges us for the amount of time the Cluster is running. Like Composer, Dataproc does not automatically auto-scale (though unlike Composer, you can set an autoscaling policy as described [here](https://cloud.google.com/dataproc/docs/concepts/configuring-clusters/autoscaling)). You set the number and type of machines being used when you instantiate your Cluster.

In this chapter we'll look at a file that won't load into BigQuery, and we'll use PySpark to transform the file to allow us to load it. We'll need a bucket to store the file we'll be transforming, so let's make that now:
``` bash
> gsutil mb gs://de-book-dataproc
```

## Why do we need Dataproc?
So far in this book I've been demonstrating an ELT (Extract Load Transform) approach to building a Data Pipeline, as distinguished from ETL. In an ELT approach we get our raw data into our Data Lake and Data Warehouse as quickly as possible, and then we perform our transformations.

However, there are good reasons to perform transformations before loading your data into your Warehouse. Perhaps your team is more comfortable working in Python, and it's more convenient to do your processing in Python before you load in your data. 

Even if you are taking an ELT approach to your Pipelines, there may be instances where you will be unable to load the data into BigQuery without first doing some transformations. For example, while BigQuery can ingest nested data such as an array of integers or an array of objects (in BQ the objects are called "Structs"), BigQuery cannot ingest an array of arrays. Additionally, you may run into other problems with source files, such as them being malformed, that prevents BigQuery from loading the files.

In Chapter 5 we did some pre-processing before we loaded our JSON data into BigQuery: we converted the JSON file from standard JSON format to Newline-Delimited JSON. We were able to do this using the compute power from our Composer Environment cluster because the amount of data we were dealing with was small. If we're dealing with large amounts of data that needs pre-processing, then we need to spin up additional resources so that we don't overwhelm our Composer Environment.

Creating a Dataproc Cluster is an excellent way to bring more compute power to bear on your data before it is loaded into BigQuery. 

## Creating a Dataproc Cluster
Before we can start working with Dataproc we need to enable it on GCP:
``` bash
> gcloud services enable dataproc.googleapis.com
```

There are [a lot of options](https://cloud.google.com/sdk/gcloud/reference/dataproc/clusters/create) you can set for creating a Dataproc Cluster. Here is a typical command:
``` bash
> gcloud dataproc clusters create my-cluster \
    --region=us-central1 \
    --num-workers=2 \
    --worker-machine-type=n2-standard-2 \
    --image-version=1.5-debian10
```
You can see I've specified the number of workers and the type of machine, which controls how much processing power the Cluster has. The image name defines the environment Spark will be running in. You can see more details about the images you can use [here](https://cloud.google.com/dataproc/docs/concepts/versioning/dataproc-versions).

We could have also included the `--metadata` and `--initialization-actions` options to install Python packages into our environment:
``` bash
    --metadata='PIP_PACKAGES=requests==2.24.0' \
    --initialization-actions=gs://de-book-dataproc/pip-install.sh
```
Google provides the script to install the requirements in an open GCS location, but recommends you copy the file locally to ensure updates they make to the file don't break your code. We can do that here:
``` bash
> gsutil cp gs://goog-dataproc-initialization-actions-us-central1/python/pip-install.sh gs://de-book-dataproc
```
We won't need to install any Python packages for our job, below.

We now have a Dataproc Cluster running Spark. Let's submit a job and have our Cluster do some work.

## Submitting a PySpark Job
Let's suppose we have a data source that provides a JSON file where one of its fields is a list of lists, which is a structure BigQuery doesn't support. We can use Dataproc to transform the data and then save the file back to GCS to be loaded into BigQuery.

First, let's create our source file and save it to our bucket:
``` bash
> echo '{"one_dimension": [1,2,3], "two_dimensions": [["a","b","c"],[1,2,3]]}
{"one_dimension": [3,4,5], "two_dimensions": [["d","e","f"],[3,4,5]]}' > raw_file.json
> gsutil cp raw_file.json gs://de-book-dataproc
```
Now, let's build our PySpark file that will perform our transformation:
``` python
#!/usr/bin/env python
from pyspark.sql import SparkSession
from pyspark.sql.functions import flatten

def main():
    spark = SparkSession.builder.appName("FileCleaner").getOrCreate()
    df = spark.read.json("gs://de-book-dataproc/raw_file.json")
    df2 = df.withColumn('two_dimensions', flatten(df.two_dimensions))
    df2.write.json("gs://de-book-dataproc/cleaned_files/", mode='overwrite')

if __name__ == "__main__":
    main()
```
Here we're doing a simple transformation of flattening our two-dimensional array into a one-dimensional array. Spark has a lot of versatility in transforming data, with [whole books](https://www.amazon.com/Spark-Definitive-Guide-Processing-Simple/dp/1491912219/) being written about it.

Now that we have our PySpark file let's move it up to GCS:
``` bash
> gsutil cp flattener.py gs://de-book-dataproc
```
Finally, we are ready to submit our dataproc job:
``` bash
> gcloud dataproc jobs submit pyspark \
    gs://de-book-dataproc/flattener.py \
    --cluster=my-cluster \
    --region=us-central1
```
With our job complete we can see our transformed file:
``` bash
> gsutil ls gs://de-book-dataproc/cleaned_files
gs://de-book-dataproc/cleaned_files/
gs://de-book-dataproc/cleaned_files/_SUCCESS
gs://de-book-dataproc/cleaned_files/part-00000-a07721fc-76a2-4235-912c-b88df333e4d4-c000.json
> gsutil cat gs://de-book-dataproc/cleaned_files/part-00000-a07721fc-76a2-4235-912c-b88df333e4d4-c000.json
{"one_dimension":[1,2,3],"two_dimensions":["a","b","c","1","2","3"]}
{"one_dimension":[3,4,5],"two_dimensions":["d","e","f","3","4","5"]}
```
Unfortunately, Spark does not allow you to name your output files, so if you need to distinguish which files were the output of a particular job you should group your files by timestamped folders (e.g. `gs://de-book-dataproc/cleaned_files_20201031/`).

## Deleting a PySpark Cluster
If you have periodic need for data processing with Spark, then you should delete your Clusters once you are done using them. It only takes a minute or two to build a new Cluster, and you don't want to be paying for a Cluster you're not using.
``` bash
> gcloud dataproc clusters delete my-cluster --region=us-central1
```
However, if you are continuously using your Dataproc Cluster (e.g. to process data from a pipeline that has batches every 10 minutes), then it may make sense to leave your Cluster up all the time.

## Dataproc and Composer
We've talked about how to use Dataproc, but we haven't really discussed how to integrate it into our Data Pipelines. The answer is simple enough, we can create tasks in Airflow (discussed in Chapter 5) to create the Cluster, submit the job, then delete the Cluster:
``` python
t_create_cluster = BashOperator(
    task_id='create_cluster',
    bash_command="""
        gcloud dataproc clusters create my-cluster \\
            --region=us-central1 \\
            --num-workers=2 \\
            --worker-machine-type=n2-standard-2 \\
            --image-version=1.5-debian10
    """,
    dag=dag
)

t_submit_job = BashOperator(
    task_id='submit_job',
    bash_command="""
        gcloud dataproc jobs submit pyspark \\
            gs://de-book-dataproc/flattener.py \\
            --cluster=my-cluster \\
            --region=us-central1
    """,
    dag=dag
)
t_submit_job.set_upstream(t_create_cluster)

t_delete_cluster = BashOperator(
    task_id='delete_cluster',
    bash_command='gcloud dataproc clusters delete my-cluster --region=us-central1',
    dag=dag
)
t_delete_cluster.set_upstream(t_submit_job)
```
When managing a Dataproc Cluster from your Composer Environment it is important to have good alerting (discussed in Chapter 13). It is possible for Airflow to mark a task as failed (e.g. because it has timed out), without the Dataproc job ending or Cluster deleting itself.

## Cleaning Up
We've already deleted our Dataproc Cluster above, so all that's left is for us to delete the bucket we created and the buckets created by Dataproc:
``` bash
> gsutil ls
gs://dataproc-staging-us-central1-204024561480-kkb48scf/
gs://dataproc-temp-us-central1-204024561480-3i3d24k7/
gs://de-book-dataproc/

> gsutil rm -r gs://dataproc-staging-us-central1-204024561480-kkb48scf/
> gsutil rm -r gs://dataproc-temp-us-central1-204024561480-3i3d24k7/
> gsutil rm -r gs://de-book-dataproc/
```

---

Next Chapter: [Chapter 8: Streaming Data with Pub/Sub](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_8_streaming.md)