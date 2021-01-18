# Up and Running: Data Engineering on the Google Cloud Platform
The completely free E-Book for setting up and running a Data Engineering stack on Google Cloud Platform.

NOTE: This book is currently incomplete. If you find errors or would like to fill in the gaps, read the [Contributions section](https://github.com/Nunie123/data_engineering_on_gcp_book#user-content-contributions).

## Table of Contents
[Preface](https://github.com/Nunie123/data_engineering_on_gcp_book) <br>
[Chapter 1: Setting up a GCP Account](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_1_gcp_account.md) <br>
[Chapter 2: Setting up Batch Processing Orchestration with Composer and Airflow](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_2_orchestration.md) <br>
[Chapter 3: Building a Data Lake with Google Cloud Storage (GCS)](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_3_data_lake.md) <br>
**Chapter 4: Building a Data Warehouse with BigQuery** <br>
[Chapter 5: Setting up DAGs in Composer and Airflow](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_5_dags.md) <br>
[Chapter 6: Setting up Event-Triggered Pipelines with Cloud Functions](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_6_event_triggers.md) <br>
[Chapter 7: Parallel Processing with Dataproc and Spark](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_7_parallel_processing.md) <br>
[Chapter 8: Streaming Data with Pub/Sub](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_8_streaming.md) <br>
[Chapter 9: Managing Credentials with Google Secret Manager](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_9_secrets.md) <br>
[Chapter 10: Infrastructure as Code with Terraform](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_10_infrastructure_as_code.md) <br>
[Chapter 11: Continuous Integration with Cloud Build](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_11_continuous_integration.md) <br>
Chapter 12: Monitoring and Alerting <br>
Appendix A: Example Code Repository


---

# Chapter 4: Building a Data Warehouse with BigQuery
BigQuery is GCP's fully managed Data Warehouse service. There are no steps to take to get it running, as long as you've got a GCP account you can just start using it. 

While GCP's Data Engineering infrastructure services are often quite similar to corresponding AWS services, BigQuery is actually a significant departure from AWS's Data Warehouse service: Redshift. 

It's features and quirks include:
1. BigQuery is fully-managed, which means that there are no options for setting compute power, storage size, or any other parameters you might be used to setting for a cloud-based database. BigQuery will auto-scale under-the-hood, so you never have to worry about whether you need to increase CPU or storage. Consequently, the billing is quite different. Rather than paying for up-time, like you will with Composer, you are billed for BigQuery based on how much data you are querying, and how much data you are storing. This can be a little dangerous, as it's not hard to accidentally run a lot of expensive queries and run up your bill.
2. BigQuery behaves like a relational database in most respects, but does allow nested fields. For example, you can define a "users" table that has a "phone_numbers" field. That field could have multiple repeated sub-fields such as "phone_number" and "phone_type", allowing you to store multiple phone numbers with a single "user" record.
3. BigQuery lacks primary keys and indexing, but does allow partitioning and clustering. Partitioning is useful because full table scans can cost a lot of money on large tables. Filtering by partition allows you to scan less data, reducing the cost of the query while improving performance. Partitions can still be somewhat limiting, however, as they can only be designated on a single column per table and the column must be a date, datetime, timestamp, or integer. Clustering can be used more similarly to how indexing is used on a RDBMS, improving performance by specifying a column or columns that are used frequently in queries.
4. BigQuery can be managed by its Data Definition Language (DDL) and Data Manipulation Language (DML), but can also use be managed using the `bq` command line tool and dedicated libraries for a variety of programming languages.
5. BigQuery tables are created inside "Datasets", which are just name-spaces for groups of tables. All Datasets are associated with a particular Project (the same way Composer Environments and GCS Buckets must be associated with a Project).

## BigQuery Query Editor
BigQuery works well with GCS and Composer, allowing you to load and transform your data in your Warehouse using Python and the `bq` command line tool. However, you're still probably going to want a SQL client to debug, prototype, and explore the data in your Warehouse. Fortunately, the GCP Console includes a [BigQuery SQL client](https://console.cloud.google.com/bigquery) for you to use. If you want to use your own SQL client, you can connect via JDBC and ODBC drivers, as described [here](https://cloud.google.com/bigquery/providers/simba-drivers).

While you can do SQL scripting inside the Query Editor, you're going to want to use the `bq` command line tool and BigQuery Python library, described below.

## Using the `bq` Command Line Tool
In Chapter 1 I discussed installing the GCP command line tools. You'll need them for this section.

We'll also need to give our service account permission to access BigQuery:
``` bash
> gcloud projects add-iam-policy-binding 'de-book-dev' \
   --member='serviceAccount:composer-dev@de-book-dev.iam.gserviceaccount.com' \
   --role='roles/bigquery.admin'
[...]
> export GOOGLE_APPLICATION_CREDENTIALS="/path/to/keys/de-book-dev.json"
```

### Creating Tables from the Command Line
All tables must have a Dataset inside which they can be created. So we start by creating a Dataset, then verifying it was created:
``` bash
> bq mk --dataset my_dataset
> bq ls -d
```
Now let's create a table in our new Dataset:
``` bash
> bq mk --table my_dataset.inventory product_name:STRING,product_count:INT64,price:FLOAT64
> bq ls --format=pretty my_dataset
+-----------+-------+--------+-------------------+------------------+
|  tableId  | Type  | Labels | Time Partitioning | Clustered Fields |
+-----------+-------+--------+-------------------+------------------+
| inventory | TABLE |        |                   |                  |
+-----------+-------+--------+-------------------+------------------+
> bq show --schema my_dataset.inventory
[{"name":"product_name","type":"STRING"},{"name":"product_count","type":"INTEGER"},{"name":"price","type":"FLOAT"}]
```
In the above code we created the table by providing the schema in-line with the command. Defining the schema in-line is usually impractical, so instead we'll create a file with the schema defined as JSON:

``` bash
> echo '
[
    {
        "description": "The name of the item being sold.",
        "mode": "REQUIRED",
        "name": "product_name",
        "type": "STRING"
    },
    {
        "description": "The count of all items in inventory.",
        "mode": "NULLABLE",
        "name": "product_count",
        "type": "INT64"
    },
    {
        "description": "The price the item is sold for, in USD.",
        "mode": "NULLABLE",
        "name": "price",
        "type": "FLOAT64"
    }
]' > inventory_schema.json
```
And now we'll create a table based on the schema file:
``` bash
> bq mk --table my_dataset.inventory_2 inventory_schema.json
> bq ls --format=pretty my_dataset
+-------------+-------+--------+-------------------+------------------+
|   tableId   | Type  | Labels | Time Partitioning | Clustered Fields |
+-------------+-------+--------+-------------------+------------------+
| inventory   | TABLE |        |                   |                  |
| inventory_2 | TABLE |        |                   |                  |
+-------------+-------+--------+-------------------+------------------+
bq show --schema my_dataset.inventory_2
[{"name":"product_name","type":"STRING","mode":"REQUIRED","description":"The name of the item being sold."},{"name":"product_count","type":"INTEGER","mode":"NULLABLE","description":"The count of all items in inventory."},{"name":"price","type":"FLOAT","mode":"NULLABLE","description":"The price the item is sold for, in USD."}]
```
More information on defining the schema is available [here](https://cloud.google.com/bigquery/docs/schemas).

You can also define a new table by making based on a query from an existing table.
``` bash
> bq query --destination_table my_dataset.inventory_3 \
> --use_legacy_sql=false \
> 'select * from my_dataset.inventory'
```
Finally, you can create a new table as part of the `bq load` command. I'll talk more about loading data in the next section.
``` bash
> bq load \
    --source_format=NEWLINE_DELIMITED_JSON \
>   my_dataset.my_table \
>   gs://path/to/blob/in/bucket/file.json \
>   my_schema.json
```

### Loading Data from the Command Line
The major pieces of information you'll need to know for your load operation are:
* The destination project, dataset, and table
* The source file location
* The source file format ([Avro](https://cloud.google.com/bigquery/docs/loading-data-cloud-storage-avro), [Parquet](https://cloud.google.com/bigquery/docs/loading-data-cloud-storage-parquet), [ORC](https://cloud.google.com/bigquery/docs/loading-data-cloud-storage-orc), [CSV](https://cloud.google.com/bigquery/docs/loading-data-cloud-storage-csv), [JSON](https://cloud.google.com/bigquery/docs/loading-data-cloud-storage-json))
* The schema of the destination table
* Whether you are replacing or appending to the destination table (`--replace` and `--noreplace` flags)
* How you are choosing to partition the table

Let's start by creating our raw data file to be loaded into BigQuery:
``` bash
> echo '
[
    {
        "product_id": 1,
        "product_name": "hammer",
        "product_tags": [
            {
                "tag_id": 1,
                "tag_name": "sale"
            },
            {
                "tag_id": 2,
                "tag_name": "tool"
            }
        ],
        "created_on": "2020-10-19"
    },
    {
        "product_id": 2,
        "product_name": "pen",
        "product_tags": [
            {
                "tag_id": 1,
                "tag_name": "sale"
            },
            {
                "tag_id": 3,
                "tag_name": "stationary"
            }
        ],
        "created_on": "2020-10-19"
    },
    {
        "product_id": 3,
        "product_name": "drill",
        "product_tags": [
            {
                "tag_id": 2,
                "tag_name": "tool"
            }
        ],
        "created_on": "2020-10-19"
    }
]' > products.json
```
While BigQuery can ingest JSON files, they must be newline delimited JSON (also called JSON Lines) files. This means that each record is separated by a newline character. Many source systems will export to newline delimited JSON, but if you're stuck with a JSON blob like we have above you'll have to convert it yourself. Let's convert the above file using the [jq](https://stedolan.github.io/jq/download/) command line tool:
``` bash
> cat products.json | jq -c '.[]' > products.jsonl
```
Note that I used the ".jsonl" extension for the destination file. I could have stuck with the ".json" extension, as BigQuery doesn't care what the extension is.

Now that we have our data we can define our schema. As mentioned in above, the load command will create a table if one does not exist, so providing the schema here will define your table. The `--autodetect` flag can be used to have BigQuery infer your schema from the data, but that should generally be avoided outside of development and prototyping. Let's create a schema definition file:
``` bash
> echo '
[
    {
        "name": "product_id",
        "type": "INT64",
        "mode": "REQUIRED",
        "description": "The unique ID for the product."
    },{
        "name": "product_name",
        "type": "STRING",
        "mode": "NULLABLE",
        "description": "The name of the product."
    },{
        "name": "product_tags",
        "type": "RECORD",
        "mode": "REPEATED",
        "description": "All tags associated with the product.",
        "fields": [{
            "name": "tag_id",
            "type": "INT64",
            "mode": "NULLABLE",
            "description": "The unique ID for the tag."
        }, {
            "name": "tag_name",
            "type": "STRING",
            "mode": "NULLABLE",
            "description": "The name of the tag."
        }]
    },{
        "name": "created_on",
        "type": "DATE",
        "mode": "REQUIRED",
        "description": "The date the product was added to inventory."
    }
]' > products_schema.json
```
You'll notice that this schema defines a nested field, called a "REPEATED" field in BigQuery. Fields can be nested up to 15 layers deep. The schema above includes the "product_tags" field, a field of "REPEATED" "RECORDS" which is the equivalent of an array or objects in JSON. BigQuery also supports structures like an array of integers, or an array of dates, but it does not support an array of arrays.

Now that we have our schema file, let's load our data.
``` bash
> bq load \
    --source_format=NEWLINE_DELIMITED_JSON \
    --replace \
    --time_partitioning_type=DAY \
    --time_partitioning_field created_on \
>   my_dataset.products \
>   products.jsonl \
>   products_schema.json
Upload complete.
Waiting on bqjob_r3f8263b30008fbdf_00000175442a455c_1 ... (1s) Current status: DONE  
> bq head --table my_dataset.products
+------------+--------------+---------------------------------------------------------------------------+------------+
| product_id | product_name |                               product_tags                                | created_on |
+------------+--------------+---------------------------------------------------------------------------+------------+
|          1 | hammer       |       [{"tag_id":"1","tag_name":"sale"},{"tag_id":"2","tag_name":"tool"}] | 2020-10-19 |
|          2 | pen          | [{"tag_id":"1","tag_name":"sale"},{"tag_id":"3","tag_name":"stationary"}] | 2020-10-19 |
|          3 | drill        |                                        [{"tag_id":"2","tag_name":"tool"}] | 2020-10-19 |
+------------+--------------+---------------------------------------------------------------------------+------------+
```
I specified that the table is partitioned on the `created_on` field, so right now this. This means that each distinct date in that field will be treated as a distinct partition, improving performance when filtering on that field. 

Partitions are always optional, but are useful when you know you will be filtering on a particular field (e.g. querying for all products created in the last seven days). If I included `--time_partitioning_type=DAY` but did not provide a field to partition on, BigQuery would have automatically assigned a partition date of the date the record was ingested into BigQuery. We can filter on this automatically generated partition date by using BigQuery's [pseudo-columns](https://cloud.google.com/bigquery/docs/querying-partitioned-tables#limiting_partitions_queried_using_pseudo_columns): `_PARTITIONDATE`, and `_PARTITIONTIME`.

## Using the `google.cloud.bigquery` Python Library
We can also interact with BigQuery in Python. In Chapter 2 we used the `google.cloud.storage` library, and now for BigQuery we'll need to install the `google.cloud.bigquery` library:
``` bash
> pip install --upgrade google-cloud-bigquery
```

### Creating Tables from Python

``` python
from google.cloud import bigquery

def create_dataset(dataset_name: str, project_name: str) -> None:
    dataset_id = f'{project_name}.{dataset_name}'
    dataset_obj = bigquery.Dataset(dataset_id)
    client = bigquery.Client()
    dataset = client.create_dataset(dataset_obj)

def create_table(table_name: str, dataset_name: str, project_name: str, schema: list) -> None:
    table_id = f'{project_name}.{dataset_name}.{table_name}'
    table_obj = bigquery.Table(table_id, schema)
    client = bigquery.Client()
    table = client.create_table(table_obj)

project_name = 'de-book-dev'
dataset_name = 'my_other_dataset'
table_name = 'user_purchases'
schema = [
    bigquery.SchemaField('user_name', 'STRING', mode='REQUIRED'),
    bigquery.SchemaField('items_purchased', 'INT64', mode='REQUIRED'),
    bigquery.SchemaField('dollars_spent', 'FLOAT64', mode='REQUIRED')
]

create_dataset(dataset_name, project_name)
create_table(table_name, dataset_name, project_name, schema)
```

We can also create a table from a query:
``` python
from google.cloud import bigquery

def create_table_from_query(table_name: str, dataset_name: str, project_name: str, raw_sql: str) -> None:
    table_id = f'{project_name}.{dataset_name}.{table_name}'
    job_config = bigquery.QueryJobConfig(destination=table_id)
    client = bigquery.Client()
    query_job = client.query(raw_sql, job_config=job_config)
    query_job.result()

project_name = 'de-book-dev'
dataset_name = 'my_other_dataset'
table_name = 'user_purchases_copy'
raw_sql = 'select user_name, items_purchased, dollars_spent from my_other_dataset.user_purchases'
create_table_from_query(table_name, dataset_name, project_name, raw_sql)
```
You can find the documentation for the Python API [here](https://googleapis.dev/python/bigquery/latest/index.html).

### Loading Data from Python

Now let's load our data using the "products.json" file we created above:
``` python
import json
from google.cloud import bigquery

def load_json_data(table_id: str, source_file_location: str, schema: list) -> None:
    client = bigquery.Client()
    job_config = bigquery.LoadJobConfig(
        schema=schema,
        source_format=bigquery.SourceFormat.NEWLINE_DELIMITED_JSON,
    )
    with open(source_file_location, 'r') as f:
        data = json.load(f)
        load_job = client.load_table_from_json(
            data,
            table_id,
            job_config=job_config,
        )
        load_job.result()

table_id = 'de-book-dev.my_other_dataset.products'
source_file_location = 'products.json'
schema = [
    bigquery.SchemaField('product_id', 'INT64', mode='REQUIRED'),
    bigquery.SchemaField('product_name', 'STRING', mode='NULLABLE'),
    bigquery.SchemaField('product_tags'
                        , 'RECORD'
                        , mode='REPEATED'
                        , fields=[
                            bigquery.SchemaField('tag_id', 'INT64', mode='REQUIRED'),
                            bigquery.SchemaField('tag_name', 'STRING', mode='NULLABLE'),
                        ]),
    bigquery.SchemaField('created_on', 'DATE', mode='REQUIRED')
]
load_json_data(table_id, source_file_location, schema)

```
Just like above, we're loading data with a nested field. Unlike the `bq load` command, if you wish to use the BigQuery's Python API to load data into a partitioned table you must create the table with the specified partition first, then load the data.

## Cleaning Up
Because BigQuery bills based on data processed and storage quantity, rather than uptime, leaving our test data in there from this chapter will incur almost no costs. Nonetheless, it's good to clean up so we can have a blank start when we start creating and loading tables as part of our DAGs in Chapter 5.

Let's start by listing all our datasets:
``` bash
> bq ls -d
my_dataset        
my_other_dataset  
```

Now let's delete them. We use the `-r` flag to indicate we also want the associated tables deleted:
``` bash
> bq rm -d -r my_dataset
> bq rm -d -r my_other_dataset
```

---

Next Chapter: [Chapter 5: Setting up DAGs in Composer and Airflow](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_5_dags.md)