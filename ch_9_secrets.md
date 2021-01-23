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
[Chapter 7: Parallel Processing with Dataproc and Spark](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_7_parallel_processing.md) <br>
[Chapter 8: Streaming Data with Pub/Sub](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_8_streaming.md) <br>
**Chapter 9: Managing Credentials with Google Secret Manager** <br>
[Chapter 10: Infrastructure as Code with Terraform](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_10_infrastructure_as_code.md) <br>
[Chapter 11: Deployment Pipelines with Cloud Build](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_11_deployment_pipelines.md) <br>
[Chapter 12: Monitoring and Alerting](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_12_monitoring.md) <br>
Chapter 13: Start to Finish - Building a Complete Data Engineering Infrastructure <br>
Appendix A: Example Code Repository


---

# Chapter 9: Managing Credentials with Google Secret Manager

There will likely be times where you are going to need to give your data pipelines access to credentials. For GCP resources we can manage access through permissions on our service accounts, but often your pipeline will need to access systems outside of GCP. By using Google Secret Manager we are able to securely store passwords and other secret information.

In this chapter I will show you how to create and access secrets using Google Secret Manager. Then we'll create an Airflow DAG that will simulate making an HTTP request to a secured web API, then saving the results to GCS.

## Creating a Secret
The first thing we need to do is enable the Google Secret Manager service:
``` bash
> gcloud services enable secretmanager.googleapis.com
```

Creating a secret is quite simple. Here I'm creating a secret called "source-api-password" that contains the value "abc123":
``` Bash
> echo -n "abc123" | gcloud secrets create source-api-password --data-file=-
```

We can also create a secret where the value is the contents of a file:
``` Bash
> gcloud secrets create source-api-password-2 --data-file=my-password.txt
```

In Chapter 10 we'll discuss how to manage secrets with Terraform. While it is good practice to create secrets with your Infrastructure as Code solution, the values of those secrets still need to be added manually to ensure they are not saved in your code repositories.
## Accessing a Secret
Accessing a secret is just as easy as creating it:
``` Bash
> gcloud secrets versions access latest --secret=source-api-password
abc123
```

The values in each secret have a particular "version", so you can update the value in a secret and still access older values for that same secret. In the above code we specify we want the value for the "latest" version, but we also could have specified an ID for a specific version.

While you will likely be setting secret values through the command line, you will most likely be accessing them within your code:
``` Python
from google.cloud import secretmanager

def get_secret(project, secret_name, version):
    client = secretmanager.SecretManagerServiceClient()
    secret_path = client.secret_version_path(project, secret_name, version)
    secret = client.access_secret_version(secret_path)
    return secret.payload.data.decode("UTF-8")

project = "de-book-dev"
secret_name = "source-api-password"
version = "latest"
plaintext_secret = get_secret(project, secret_name, version)
```

## Using Google Secret Manager in Airflow
Below is an example of how you might typically use Google Secret Manager to access secrets within your DAG for getting data from a web API. The web API in this example doesn't exist (so it won't work if you run it unmodified in your own Airflow instance), but the below code shows you a common type of DAG that would require accessing a secret.

``` Python
import requests
import json

from airflow import DAG
from airflow.operators.python_operator import PythonOperator
from airflow.operators.bash_operator import BashOperator

default_args = {
    'owner': 'DE Book',
    'depends_on_past': False,
    'email': [''],
    'email_on_failure': False,
    'email_on_retry': False,
    'retries': 3,
    'retry_delay': datetime.timedelta(seconds=30),
    'start_date': datetime.datetime(2020, 10, 17),
}

dag = DAG(
    'download_form_web_api',
    schedule_interval="0 * * * *",      # run every day at midnight UTC
    max_active_runs=1,
    catchup=False,
    default_args=default_args
)


def get_secret(project, secret_name, version='latest'):
    client = secretmanager.SecretManagerServiceClient()
    secret_path = client.secret_version_path(project, secret_name, version)
    secret = client.access_secret_version(secret_path)
    return secret.payload.data.decode('UTF-8')


def generate_credentials():
    username = 'my_username'
    password = get_secret('my_project', 'source-api-password')
    credentials = f'{username}:{password}'
    return credentials


def download_data_to_local():
    url = 'https://www.example.com/api/source-endpoint'
    credentials = generate_credentials()
    request_headers = {"Accept": "application/json",
                       "Content-Type": "application/json",
                       "Authorization": credentials}
    export_json = {
        "exportType": "Full_Data",
    }
    response = requests.post(url=url, json=export_json, headers=request_headers)
    data = response.json()
    # composer automatically maps "/home/airflow/gcs/data/" to a bucket so it can be treated as a local directory
    with open('/home/airflow/gcs/data/data.json', 'w') as f:
        json.dump(data, f)


t_download_data_to_local = PythonOperator(
    task_id='download_data_to_local',
    python_callable=get_data,
    op_kwargs={"sku_list": sku_list},
    dag=dag
)

t_copy_data_to_gcs = BashOperator(
    task_id='copy_data_to_gcs',
    bash_command='gsutil cp /home/airflow/gcs/data/data.json gs://my-bucket/web-api-files/'
    dag=dag
)
t_download_data_to_local.set_upstream(t_download_data_to_local)
```

## Cleaning Up
Google Secret Manager is quite cheap, with each secret version costing 6 cents per month, and an additional 3 cents for every 10,000 times you access your secret. Nonetheless, let's clean up what we're not using.

``` Bash
> gcloud secrets list
NAME                 CREATED              REPLICATION_POLICY  LOCATIONS
source-api-password  2021-01-13T04:07:31  automatic           -
> gcloud secrets delete source-api-password
```

---

Next Chapter: [Chapter 10: Infrastructure as Code with Terraform](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_10_infrastructure_as_code.md)