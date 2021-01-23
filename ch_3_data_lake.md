# Up and Running: Data Engineering on the Google Cloud Platform
The completely free E-Book for setting up and running a Data Engineering stack on Google Cloud Platform.

NOTE: This book is currently incomplete. If you find errors or would like to fill in the gaps, read the [Contributions section](https://github.com/Nunie123/data_engineering_on_gcp_book#user-content-contributions).

## Table of Contents
[Preface](https://github.com/Nunie123/data_engineering_on_gcp_book) <br>
[Chapter 1: Setting up a GCP Account](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_1_gcp_account.md) <br>
[Chapter 2: Setting up Batch Processing Orchestration with Composer and Airflow](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_2_orchestration.md) <br>
**Chapter 3: Building a Data Lake with Google Cloud Storage (GCS)** <br>
[Chapter 4: Building a Data Warehouse with BigQuery](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_4_data_warehouse.md) <br>
[Chapter 5: Setting up DAGs in Composer and Airflow](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_5_dags.md) <br>
[Chapter 6: Setting up Event-Triggered Pipelines with Cloud Functions](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_6_event_triggers.md) <br>
[Chapter 7: Parallel Processing with Dataproc and Spark](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_7_parallel_processing.md) <br>
[Chapter 8: Streaming Data with Pub/Sub](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_8_streaming.md) <br>
[Chapter 9: Managing Credentials with Google Secret Manager](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_9_secrets.md) <br>
[Chapter 10: Infrastructure as Code with Terraform](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_10_infrastructure_as_code.md) <br>
[Chapter 11: Deployment Pipelines with Cloud Build](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_11_deployment_pipelines.md) <br>
[Chapter 12: Monitoring and Alerting](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_12_monitoring.md) <br>
Chapter 13: Start to Finish - Building a Complete Data Engineering Infrastructure <br>
Appendix A: Example Code Repository


---

# Chapter 3: Building a Data Lake with Google Cloud Storage (GCS)
GCS provides cheap and easily accessible storage for any type of file. It is quite similar to AWS Simple Storage Solution (S3), and even has dedicated commands to be interoperable with S3.

## What's a Data Lake?
A Data Lake is a schemaless data repository used for storing source files in their native format. If you're completely unfamiliar with the concept, you can think of it as a folder where all the data you use is saved together.

While a Data Warehouse imposes a schema on the data in the form of tables and relationships between tables (see Chapter 4 for more information on Data Warehousing), a Data Lake imposes no schema on the data it stores. Some benefits are:
1. It's easy to ingest source data if the entire ingestion process is just to save the data unedited from the source system.
2. It ensures data is not lost as a result of data transformations. The original data is always accessible for downstream processing.

Some drawbacks are:
1. It's hard to query. The schemaless structure can make it difficult to understand how data relates to each other.
2. It can be messy. Source system data may benefit from cleaning (de-duplication field value normalization, etc.)before it is used downstream.

Technologies allowing users query Data Lakes directly continues to improve. While there may be some scenarios where it makes sense to use a Data Lake without a Data Warehouse, such as organizations that are focused purely on Machine Learning and are not using Data Engineers to support Analytics, it is more common for an organization to use a Data Lake and Data Warehouse together.

While Data Lakes are generally considered to be schemaless, there is usually the ability to add some structure. Below we'll discuss how to use multiple GCS buckets and bucket sub-folders to organize our Data Lake.

## Using GCS as a Data Lake
GCS works well as a Data Lake because it is cheap, easy to access, and integrates well with other GCP services, particularly BigQuery. 

Every file saved in GCS is called an Object (also referred to as a "Blob"). Every Object is contained within a Bucket. Technically, every Object is saved at the top level of the Bucket. There is no hierarchical structure like would be used in a Windows or MacOS file system. However, GCS allows the Objects to be named as if they are in sub-directories, so for practical purposes we can save files in sub-directories inside a Bucket.

You can manage GCS through the Console, the `gsutil` command line tool, through various code libraries, or through a REST API. I'll discuss `gsutil` and the Python library, since those integrate easiest with Airflow (I'll demonstrate using GCS with Airflow/Composer in Chapter 5).

### Using the `gsutil` command line tool
In Chapter 1 I discussed installing the GCP command line tools. You'll need them for this section.

Creating a bucket is quite easy:
``` bash
> gsutil mb gs://de-book-dev
```
The name of a Bucket must be unique across all GCP accounts, so you may find that the Bucket name you want to use is not available.
``` bash
> gsutil mb gs://de-book-dev
Creating gs://de-book-bucket/...
ServiceException: 409 Bucket de-book-dev already exists.
```
We can see all our Buckets by running:
``` bash
> gsutil ls
```
Now let's create some files and then copy them into our Bucket:
``` bash
> mkdir files
> echo "This is text" > ./files/a_text_file.txt
> echo '{"this_is_json: true}' > ./files/a_json_file.json
> gsutil cp ./files/a_text_file.txt gs://de-book-dev
[...]
> gsutil cp -r ./files gs://de-book-dev
[...]
> gsutil cp ./files/*.json gs://de-book-dev/json_files/
[...]
```
The three `cp` commands above demonstrate:
1. Copying a single file into a Bucket.
2. Copying an entire directory into a Bucket.
3. Copying all files with a ".json" extension into a Bucket's sub-directory.

Let's take a look at the Bucket to make sure the Objects made it in:
``` bash
> gsutil ls gs://de-book-dev
gs://de-book-dev/a_text_file.txt
gs://de-book-dev/json_files/
gs://de-book-dev/files/

> gsutil ls gs://de-book-dev/files
gs://de-book-dev/files/a_json_file.json
gs://de-book-dev/files/a_text_file.txt

> gsutil ls gs://de-book-dev/json_files
gs://de-book-dev/json_files

> gsutil ls gs://de-book-dev/*
[...]
```

We can just as easily use the `cp` command to download the files from GCS by switching the source and destination arguments in the examples above. The `cp` command can also be used to copy files between GCS buckets.

The [`gsutil rm`](https://cloud.google.com/storage/docs/gsutil/commands/rm) and [`gsutil mv`](https://cloud.google.com/storage/docs/gsutil/commands/mv) commands, among [others](https://cloud.google.com/storage/docs/gsutil/commands/help), are also available for use.

One command we should discuss that isn't based on a UNIX command like `cp` is the [`gsutil rsync`](https://cloud.google.com/storage/docs/gsutil/commands/rsync) command. Using `rsync` on a local folder will ensure that any file in that folder is also added to the designated Bucket, if it doesn't yet exist there:
``` bash
> gsutil rsync ./files gs://de-book-dev/synced_files/
```


Another nice feature of GCS is that you can use the `gsutil cp` and `gsutil rsync` commands directly with AWS S3:
``` bash
> gsutil cp s3://my-aws-bucket/some_file.txt gs://my-gcs-bucket
```
For this to work you'll need to make a Boto configuration file to manage your AWS credentials. Details are [here](https://cloud.google.com/storage/docs/boto-gsutil).

### Using the `google.cloud.storage` Python Library
To access this library will need have the Python `google-cloud-package library` [installed and configured](https://cloud.google.com/storage/docs/reference/libraries#client-libraries-install-python).

Inside your Python virtual environment run:
``` bash
> pip install --upgrade google-cloud-storage
```
Now we need to configure our credentials. In Chapter 2 we set up a Service Account and generated a secret key. You'll need that service account and key file to continue. You can see all your service accounts with:
``` bash
> gcloud iam service-accounts list
DISPLAY NAME                            EMAIL                                               DISABLED
composer-dev                            composer-dev@de-book-dev.iam.gserviceaccount.com    False
Compute Engine default service account  204024561480-compute@developer.gserviceaccount.com  False
```

We now need to give our composer-dev service account permission to manage GCS:
``` bash
> gcloud projects add-iam-policy-binding 'de-book-dev' \
    --member='serviceAccount:composer-dev@de-book-dev.iam.gserviceaccount.com' \
    --role='roles/storage.admin'
```
Finally, we need to set the `GOOGLE_APPLICATION_CREDENTIALS` variable in our shell, referencing our secret key file that we saved in Chapter 2:
``` bash
> export GOOGLE_APPLICATION_CREDENTIALS="/path/to/keys/de-book-dev.json"
```
If you like, you can save this variable definition in your .bash_profile file, so that it will be set by default every time your terminal loads.

Now we'll create some handy functions using the [create_bucket()](https://googleapis.dev/python/storage/latest/client.html#google.cloud.storage.client.Client.create_bucket) method from the Client object, and the [upload_from_filename()](https://googleapis.dev/python/storage/latest/blobs.html#google.cloud.storage.blob.Blob.upload_from_filename) method from the Blob object.


``` python
from google.cloud import storage

def create_bucket(bucket_name: str) -> None:
    """
    This function creates a bucket with the provided bucket name. The project and location are set to default.
    """
    client = storage.Client()
    client.create_bucket(bucket_name)

def upload_file_to_bucket(source_filepath: str, bucket_name: str, blob_name: str) -> None:
    """
    This function uploads a file to a bucket. The blob_name will be the name of the file in GCS.
    """
    client = storage.Client()
    bucket = client.bucket(bucket_name)
    blob = bucket.blob(blob_name)
    blob.upload_from_filename(source_filepath)

create_bucket('de-book-test-bucket')
upload_file_to_bucket('./files/a_text_file.txt', 'de-book-test-bucket', 'my_renamed_file.txt')
```
Documentation for the Python storage library is [here](https://googleapis.dev/python/storage/latest/client.html).

### Cleaning Up
While the [prices](https://cloud.google.com/storage/pricing) for storage in GCS are pretty cheap, it's still worth cleaning up our Buckets. We can see all our Buckets with this command:
``` bash
> gsutil ls
gs://de-book-dev/
gs://de-book-test-bucket/
```
If you didn't clean up after Chapter 2, you should see some Buckets related to the Composer Environment we set up, in addition to the Buckets we created this chapter.

To delete a Bucket and everything inside run:
``` bash
> gsutil rm -r gs://de-book-test-bucket
```

To delete all the objects in a Bucket, but leave the Bucket itself, run:
``` bash
> gsutil rm gs://de-book-dev/**
```

---

Next Chapter: [Chapter 4: Building a Data Warehouse with BigQuery](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_4_data_warehouse.md)