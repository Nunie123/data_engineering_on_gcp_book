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
**Chapter 8: Streaming Data with Pub/Sub** <br>
[Chapter 9: Managing Credentials with Google Secret Manager](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_9_secrets.md) <br>
[Chapter 10: Infrastructure as Code with Terraform](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_10_infrastructure_as_code.md) <br>
[Chapter 11: Deployment Pipelines with Cloud Build](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_11_deployment_pipelines.md) <br>
[Chapter 12: Monitoring and Alerting](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_12_monitoring.md) <br>
Chapter 13: Start to Finish - Building a Complete Data Engineering Infrastructure <br>
Appendix A: Example Code Repository


---

# Chapter 8: Streaming Data with Pub/Sub
In previous chapters we've gone over how to set up batch-processing Data Pipelines, where a large number of records are added to our Warehouse at one time. Streaming, by contrast, focuses on adding small amounts of data very quickly, so that our Warehouse is updated constantly from the source system with minimal lag.

To accomplish this we will be using GCP's Pub/Sub service. Pub/Sub is a messaging service that uses a "publication" and "subscription" architecture. For a particular Data Pipeline we will create a "Topic" in Pub/Sub. A Publisher service will publish data from the source system to the Topic, and then a Subscriber service will receive the data from the Topic and process it, such as inserting that data into BigQuery.

One of the big concerns about streaming data is that new data will be generated faster than it can be ingested (this problem also exists in traditional batch processing, but is more likely to be a concern with streaming pipelines). Pub/Sub overcomes this potential issue by separating the service that gets the data from the source and the service that inserts the data into our warehouse. Because Pub/Sub persists the data until we are able to consume it, we have less concerns about the data being lost in transit. And once the data is published, Pub/Sub ensures the data will not be lost even if a failure occurs while the subscriber is pulling the data.

In this chapter we will create a Topic, deploy a Publisher service to generate our source data, and deploy a Subscriber service that pulls the source data.

## Creating a Pub/Sub Topic
Generally, we'll have a single Topic per pipeline. A single Topic can have multiple Publishers and multiple Subscribers, so it offers lots of flexibility in where the data is coming from and where it is sent to.

Creating a Topic is simple. Unlike Composer or Dataproc, there are [few configuration options](https://cloud.google.com/sdk/gcloud/reference/pubsub/topics/create), and you'll most likely be setting up your Topic by simply executing:
``` bash
> gcloud pubsub topics create my_topic
```
In Chapter 11 we'll discuss how to manage your Topics in Terraform, along with your other cloud infrastructure.

Now that we have our Topic, we're ready to create our Publisher and Subscriber.

## Creating a Publisher
The Publisher will need to have authorization to publish to your Topic. This means that the Publisher must either be controlled by your team, or you must work with another team to grant them access. Sometimes it'll be convenient to grant credentials to another team, as they will be working with a service that can publish directly. 

However, your source data may come from a system that is unable or unwilling to publish directly to your Topic. In such an instance you will have to set up your own service to gather the data, then publish to your Topic. An example would be gathering data in small chunks (micro-batches) through a JDBC connection and publishing that data to a Pub/Sub Topic.

For our purposes, we are going to upload data to a GCS bucket (covered in Chapter 3) and use a Cloud Function (covered in Chapter 6) to publish the data to a Pub/Sub Topic. Instead of running in a Cloud Function, you may decide to run the Publisher on Google Kubernetes Engine (GKE) or Cloud Run, which are also good options.

First let's create a bucket that will trigger the Cloud Function publisher:
``` Bash
> gsutil mb gs://de-book-publisher
```

Now let's create our Cloud Function to publish our data. As discussed in Chapter 6, this file must be saved as `main.py` before being uploaded to GCP.
``` Python
def publish_data(event, context):
    from google.cloud import storage
    from google.cloud import pubsub_v1

    # get the message to publish
    bucket_name = event['bucket']
    file_path = event['name']
    storage_client = storage.Client()
    bucket = storage_client.get_bucket(bucket_name)
    blob = bucket.blob(file_path)
    message = blob.download_as_text()

    # publish the message
    project_id = 'de-book-dev'
    topic_id = 'my_topic'
    publisher = pubsub_v1.PublisherClient()
    topic_path = publisher.topic_path(project_id, topic_id)
    response = publisher.publish(topic_path, message.encode('utf-8'))
    print(response.result())

```

Our Cloud Function requires two libraries, so we've got to let GCP know we need these in our environment. As discussed in Chapter 6, we do this by adding a `requirements.txt` file in the same folder as our Cloud Function:
``` text
google-cloud-storage=1.35.0
google-cloud-pubsub=2.2.0
```

Now let's deploy this function and tell it to trigger whenever a new file is uploaded to our bucket. Run the following from the same folder where the `main.py` file you just created is saved.

``` Bash
> gcloud functions deploy publish_data \
    --runtime python37 \
    --trigger-resource gs://de-book-publisher \
    --trigger-event google.storage.object.finalize
```

That's it. We now have a function that will publish messages to our topic. Now we just need a subscriber so we can read them.
## Creating a Subscriber
There are two types of subscriptions for a topic: "push" and "pull". The push subscription has Pub/Sub send the message to a designated HTTPS endpoint whenever it receives a message. A pull subscription requires a different service to poll the Topic to see if messages are available, and then pull them down.

Let's create a pull subscription (the default):
``` Bash
> gcloud pubsub subscriptions create my-subscriber --topic=my_topic
```

Now that our publisher and subscriber are set up let's upload a file to the bucket that will publish a message:
``` Bash
> echo "My first message" | gsutil cp - gs://de-book-publisher/first_message.txt
```

We've created a file in our bucket, which has triggered a Cloud Function that copied the contents of the file into a message that it published to a Pub/Sub Topic. Let's check to see if the message is there:
``` Bash
> gcloud pubsub subscriptions pull my-subscriber
┌──────────────────┬──────────────────┬──────────────┬────────────┬──────────────────┬──────────────┐
│       DATA       │    MESSAGE_ID    │ ORDERING_KEY │ ATTRIBUTES │ DELIVERY_ATTEMPT │ ACK_ID       │
├──────────────────┼──────────────────┼──────────────┼────────────┼──────────────────┼──────────────┤
│ My first message │ 1901834288157567 │              │            │                  │ ABC123       │
└──────────────────┴──────────────────┴──────────────┴────────────┴──────────────────┴──────────────┘
```

One of the ways that Pub/Sub allows your streaming service to be resilient is that it will not remove a message from a Topic until the message has been acknowledged. This helps ensure that a failure while your subscriber is pulling data doesn't result in data loss (though this does leave open the possibility of receiving the same message twice).

So this time let's pull the message down and acknowledge:
``` Bash
> gcloud pubsub subscriptions pull my-subscriber --auto-ack
┌──────────────────┬──────────────────┬──────────────┬────────────┬──────────────────┐
│       DATA       │    MESSAGE_ID    │ ORDERING_KEY │ ATTRIBUTES │ DELIVERY_ATTEMPT │
├──────────────────┼──────────────────┼──────────────┼────────────┼──────────────────┤
│ My first message │ 1901834288157567 │              │            │                  │
└──────────────────┴──────────────────┴──────────────┴────────────┴──────────────────┘
```

If we try to pull again we find there are no messages:
``` Bash
> gcloud pubsub subscriptions pull my-subscriber --auto-ack
Listed 0 items.
```

In practice you're more likely to pull messages with Python code like so:
``` Python
from google.cloud import pubsub_v1

project_id = 'de-book-dev'
subscription_id = 'my-subscriber'

subscriber = pubsub_v1.SubscriberClient()
subscription_path = subscriber.subscription_path(project_id, subscription_id)

while True:
    response = subscriber.pull(subscription_path)

    if response.received_messages:
        for item in response.received_messages:
            print(item.message) # this is where you would do something like insert the data into BigQuery
            subscriber.acknowledge(subscription_path, ack_ids=[item.ack_id])
```

## Cleaning Up
We created a GCS bucket, Cloud Function, and Pub/Sub Topic in this chapter, so let's delete them:
``` Bash
> gsutil rm -r gs://de-book-publisher

> gcloud functions delete publish_data

> gcloud pubsub topics delete my_topic
```

---

Next Chapter: [Chapter 9: Managing Credentials with Google Secret Manager](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_9_secrets.md)