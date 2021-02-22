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
[Chapter 6: Setting up Event-Triggered Pipelines with Cloud Functions](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_06_event_triggers.md) <br>
[Chapter 7: Parallel Processing with Dataproc and Spark](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_07_parallel_processing.md) <br>
[Chapter 8: Streaming Data with Pub/Sub](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_08_streaming.md) <br>
[Chapter 9: Managing Credentials with Google Secret Manager](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_09_secrets.md) <br>
[Chapter 10: Infrastructure as Code with Terraform](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_10_infrastructure_as_code.md) <br>
[Chapter 11: Deployment Pipelines with Cloud Build](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_11_deployment_pipelines.md) <br>
**Chapter 12: Monitoring and Alerting** <br>
[Chapter 13: Up and Running - Building a Complete Data Engineering Infrastructure](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_13_up_and_running.md) <br>
[Appendix A: Example Code Repository](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/appendix_a_example_code/README.md)


---

# [Chapter 12](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_12_monitoring.md): Monitoring and Alerting

We've spent 11 chapters in this book going over how to set up your infrastructure. But that doesn't do us much good if everything breaks and you don't fix it. There will be problems, whether with your code, or a GCP outage, or a malformed file. It's our responsibility as data engineers to quickly identify when there's a problem and give ourselves as much information as we can to fix the issue.

Conveniently, GCP provides dashboard for each of its services, so we can easily monitor the health of our infrastructure and make adjustments as needed. While the monitoring dashboards are automatically set up for you and can be easily customized through the console, you'll be responsible for setting up your own alerts. Consequently, this Chapter is going to be focused on setting Alerts, but you can check out GCP's [Monitoring Service](https://console.cloud.google.com/monitoring) for setting up monitoring dashboards.

In this chapter we're going to set up alerting on a Composer (Airflow) instance, and on a deployment pipeline. We'll be using Airflow's built-in alerting functionality for failed Tasks, and rolling our own alerting functionality for the Cloud Build deployment pipeline. There's lots of places you can send alerts, such as BigQuery, Pub/Sub, or GCS, but in this chapter we'll be sending our alerts to a Slack channel.

While we're only covering alerting for two pieces of infrastructure in this chapter, you should set up alerting for all of your critical infrastructure in a production environment. In [Chapter 13](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_13_up_and_running.md) we'll be setting up a complete infrastructure, including alerting for all our critical systems, so check that out if you want to see how alerting is done on other tools, such as DataProc.

Specifically what we're going to do in this chapter is:
1. Instantiate a Composer Environment.
2. Create a Slack workspace.
3. Create a deployment pipeline.
4. Create DAGs with custom alerting.
5. Create alerting on our deployment pipeline.
6. Deploy a DAG and see how the alerts work.

Many of these steps were just covered in the last chapter (Chapter 11), so I'll be brief on the details. Please refer to [Chapter 11: Deployment Pipelines with Cloud Build](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_11_deployment_pipelines.md) if you want a more detailed explanation of how to set up a deployment pipeline, and Chapters 2 and 5 for more details on setting up a Composer Environment.

## Starting a Composer Environment
This command should be pretty familiar to you by now. We do this first because it'll take awhile for GCP to set up our instance.
``` bash
> gcloud composer environments create my-environment \
    --location us-central1 \
    --zone us-central1-f \
    --machine-type n1-standard-1 \
    --image-version composer-1.12.2-airflow-1.10.10 \
    --python-version 3 \
    --node-count 3 \
    --service-account composer-dev@de-book-dev.iam.gserviceaccount.com 
```
In a production setting we would create this Composer Environment using Terraform, discussed in [Chapter 10: Infrastructure as Code with Terraform](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_10_infrastructure_as_code.md).

## Create a Slack Workspace
We need to send the alerts somewhere that will catch the eye of the data engineering team, so Slack is a good choice. Slack has a pretty intuitive setup process, and creating a workspace is free. So go [here](https://slack.com/create) to create your workspace. 

Once you've got your new slack workspace set up go [here](https://api.slack.com/messaging/webhooks) to set up incoming webhooks, which will allow you to post messages to Slack through an HTTP POST request.

The URL you've just generated should be kept secret (so that random strangers can't post messages to your Slack channels). Consequently, we're going to add it to Google Secret Manager, discussed in more detail in [Chapter 9: Managing Credentials with Google Secret Manager](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_09_secrets.md).
``` Bash
> echo -n "https://hooks.slack.com/services/QWERTY/ASDFG/123456" | gcloud secrets create slack_webhook --data-file=-
```
Replace the above string with the incoming webhook URL indicated by Slack for your account.

## Create a Deployment Pipeline
Just like last chapter, we'll need to create a GitHub repository and link it to our GCP account. Rather than just repeating that whole section in this chapter, you should go [read that section from [Chapter 11: Deployment Pipelines with Cloud Build](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_11_deployment_pipelines.md)](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_11_deployment_pipelines.md#create-a-github-repo-and-connect-it-to-gcp).

Now that we have our GitHub Repo connected to GCP, we need to clone the repo to our local machine:
``` Bash
> git clone https://github.com/Nunie123/gcp_alerting_test.git
Cloning into 'gcp_alerting_test'...
warning: You appear to have cloned an empty repository.
> cd gcp_alerting_test/
```

Let's build our file structure and (empty) files:
``` Bash
> touch cloudbuild.yaml
> touch requirements.txt
> mkdir dags
> cd dags
> mkdir python_scripts
> mkdir tests
> touch my_dag.py
> touch python_scripts/my_script.py
> touch tests/test_python_scripts.py
```

Before we can fill in our `cloudbuild.yaml` file we'll need the bucket name Composer will use to look for our DAGs:
``` Bash
> gcloud composer environments describe my-environment \
    --location us-central1 \
    --format="get(config.dagGcsPrefix)"
gs://us-central1-my-environment-9567c0a7-bucket/dags
```
Note that you may need to wait for your Composer Environment to finish building before the above command will work.

Now we can write our `cloudbuild.yaml` file:
``` YAML
# cloudbuild.yaml
steps:
- name: 'docker.io/library/python:3.7'
  id: Test
  entrypoint: /bin/sh
  args: [-c, 'python -m unittest dags/tests/test_python_scripts.py']
- name: gcr.io/google.com/cloudsdktool/cloud-sdk
  id: Deploy
  entrypoint: bash
  args: [ '-c', 'gsutil -m rsync -d -r ./dags gs://${_COMPOSER_BUCKET}/dags']
substitutions:
    _COMPOSER_BUCKET: us-central1-my-environment-9567c0a7-bucket
```
Finally, let's deploy our Cloud Build Trigger to GCP:
``` Bash
> gcloud beta builds triggers create github \
    --repo-name=gcp_alerting_test \
    --repo-owner=Nunie123 \
    --branch-pattern="master" \
    --build-config=cloudbuild.yaml
```

Everything we did in this section was described in more detail in the previous chapter, so check out [Chapter 11: Deployment Pipelines with Cloud Build](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_11_deployment_pipelines.md) if you want to know a bit more about what we just did.

## Create DAGs with Custom Alerting
We're going to need to use the `requests` and `google-cloud-secret-manager` libraries to send messages to our Slack workspace. Let's add these packages to our Composer Environment. First, let's fill in the requirements.txt file in the top level of our repo:
``` text
requests
google-cloud-secret-manager
```

Now let's update our Composer Environment:
``` Bash
> gcloud composer environments update my-environment \
    --update-pypi-packages-from-file requirements.txt \
    --location us-central1
```
While you're waiting for this to complete we can carry on filling out our files. We just need this to complete before we push our code to our repo.

Now that we've taken care of our Python dependencies, let's make a DAG that will run a custom alert whenever a task fails. We do this with the `on_error_callback` option, which can be set at the DAG or Task level and indicates a function that will be executed when a task fails. This function can do anything, but we're going to make the callback function send a message to Slack.

``` Python
# my_dag.py
import datetime
import textwrap

import requests
from google.cloud import secretmanager
from airflow import DAG
from airflow.operators.python_operator import PythonOperator

from python_scripts import my_script

def get_secret(project, secret_name, version):
    client = secretmanager.SecretManagerServiceClient()
    secret_path = client.secret_version_path(project, secret_name, version)
    secret = client.access_secret_version(secret_path)
    return secret.payload.data.decode("UTF-8")

def send_error_to_slack(context):
    webhook_url = get_secret('de-book-dev', 'slack_webhook', 'latest')
    message = textwrap.dedent(f"""\
            :red_circle: Task Failed.
            *Dag*: {context.get('task_instance').dag_id}
            *Task*: <{context.get('task_instance').log_url}|*{context.get('task_instance').task_id}*>""")
    message_json = dict(text=message)
    requests.post(webhook_url, json=message_json)

default_args = {
    'owner': 'DE Book',
    'depends_on_past': False,
    'email': [''],
    'email_on_failure': False,
    'email_on_retry': False,
    'retries': 1,
    'retry_delay': datetime.timedelta(seconds=30),
    'start_date': datetime.datetime(2020, 10, 17),
    'on_error_callback': send_error_to_slack,
}

dag = DAG(
    'my_dag',
    schedule_interval="0 0 * * *",   # run every day at midnight UTC
    max_active_runs=1,
    catchup=False,
    default_args=default_args
)

t_run_my_script = PythonOperator(
    task_id="run_my_script",
    python_callable=my_script.fail_sometimes,
    on_failure_callback=send_error_to_slack,
    dag=dag
)
```

Let's create our Python function for our Task. We'll make it work for now, and will edit it in the Testing Our Alerts section to see what happens when an Exception is raised.
``` Python
# my_script.py
def fail_sometimes():
    # div_by_zero = 1/0
    return 'No Exception!'
```

Finally, let's create our test. Like above, we'll make the test pass for now, but will edit it to fail in the Testing Our Alerts section.
``` Python
# test_python_scripts.py
import unittest

class TestScripts(unittest.TestCase):

    def test_script(self):
        self.assertTrue(1==1)
        # self.assertTrue(1==0)

if __name__ == '__main__':
    unittest.main()
```

In this section we created alerts directly in Airflow. In the next section we'll use GCP-specific tools.

## Add Alerts to the Deployment Pipeline
To get deployment notifications to Slack we are going to rely on Cloud Build, Pub/Sub, and Cloud Functions. The process looks like this:
1. We push code changes to our GitHub repository.
2. Cloud Build executes our tests and deploys our code to GCP.
3. Cloud Build then publishes the outcome of its actions (e.g. success or failure of the deployment) to a Pub/Sub Topic.
4. This triggers a Cloud Function which sends our alerts to Slack.

The only magic in this process is Cloud Build automatically publishing to a Pub/Sub topic. We have to build all the other steps, but we've already covered all these tools previously in this book.

We already set up Cloud Build, above, so let's create our Pub/Sub Topic. I cover Pub/Sub topics in more detail in [Chapter 8: Streaming Data with Pub/Sub](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_08_streaming.md).
``` Bash
> gcloud pubsub topics create cloud-builds
```
The name of the Topic must be `cloud-builds` so that Cloud Build knows where to publish.

Now for the last piece, our Cloud Function. I covered Cloud Functions in more detail in [Chapter 6](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_06_event_triggers.md). 

At the top level of the repository we created for our Airflow DAGs, lets create our function files:
``` Bash
> mkdir functions
> mkdir functions/send-to-slack
> touch functions/send-to-slack/main.py
> touch functions/send-to-slack/requirements.txt
```

Now we are basically re-implementing the alerting function we used for our Airflow Task, above. We'll start with our requirements file, as we'll need Secrets Manager to get the Slack endpoint, and we'll need the `requests` library to send our HTTP Post request.

requirements.txt:
``` Text
requests
google-cloud-secret-manager
```

main.py:
``` Python
def send_to_slack(event, context):
    import requests
    from google.cloud import secretmanager

    client = secretmanager.SecretManagerServiceClient()
    name = "projects/204024561480/secrets/slack_webhook/versions/latest"
    response = client.access_secret_version(request={"name": name})
    webhook_url = response.payload.data.decode("UTF-8")

    build_status = event.get('attributes').get('status')
    build_id = event.get('attributes').get('buildId')
    if build_status == 'SUCCESS':
        slack_text = f':large_green_circle: Cloud Build Success for {build_id}'
    elif build_status == 'FAILURE'::
        slack_text = f':red_circle: ATTENTION! Cloud Build {build_id} did not succeed. Status is {build_status}.'
    slack_message = dict(text=slack_text)
    requests.post(webhook_url, json=slack_message)
```
To get the fully specified name of your secret (`projects/204024561480/secrets/slack_webhook/versions/latest` above) execute:
``` Bash
> gcloud secrets versions describe latest --secret=slack_webhook
createTime: '2021-02-01T01:07:29.214017Z'
name: projects/204024561480/secrets/slack_webhook/versions/1
replicationStatus:
  automatic: {}
state: ENABLED
```

Now let's deploy our function. In a production environment we would just add another step to our `cloudbuild.yaml` file we created above, but we want to see our alerts on our first push to our repo, so we'll deploy them manually right now. Navigate to the `send-to-slack` folder and execute:
``` Bash
> gcloud functions deploy send_to_slack \
    --runtime python37 \
    --trigger-topic cloud-builds \
    --service-account composer-dev@de-book-dev.iam.gserviceaccount.com 
```
I used my `composer-dev@de-book-dev.iam.gserviceaccount.com` service account I created back in Chapter 1. You can find yours by executing:
``` bash
> gcloud iam service-accounts list
DISPLAY NAME                            EMAIL                                               DISABLED
composer-dev                            composer-dev@de-book-dev.iam.gserviceaccount.com    False
Compute Engine default service account  204024561480-compute@developer.gserviceaccount.com  False
```

Because our Cloud Function will be accessing Secret Manager we'll need to make sure out service account can also access secrets.
``` bash
> gcloud projects add-iam-policy-binding 'de-book-dev' \
    --member='serviceAccount:composer-dev@de-book-dev.iam.gserviceaccount.com' \
    --role='roles/secretmanager.admin'
```

That's it. Deployment messages will now be sent to our Slack workspace. In the next section we'll see it in action.


## Testing our Alerts
Now we have two alerts set up. One will send an alert if a Task fails. The other will send an alert whenever there's a deployment. So let's start by deploying our code and seeing a Slack alert notifying us whether the deploy was successful. As we talked about in [Chapter 11: Deployment Pipelines with Cloud Build](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_11_deployment_pipelines.md), we kick off our deploy process by pushing our code to GitHub:
``` Bash
> git status
> git add --all
> git commit -m 'Initial commit of alerting test code'
> git push
```

It'll take a moment for your deployment process to finish. You can monitor your deployment process from your repo in GitHub, or from the GCP Cloud Build console. But pretty soon you'll be getting a message letting you know your deployment succeeded.

Let's update our unit test so that our deployment fails:
``` Python
# test_python_scripts.py
import unittest

class TestScripts(unittest.TestCase):

    def test_script(self):
        self.assertTrue(1==1)
        self.assertTrue(1==0) # This will cause an exception

if __name__ == '__main__':
    unittest.main()
```

Now when we deploy our code we will see a failure notification in Slack:
``` Bash
> git status
> git add --all
> git commit -m 'Making unit test fail'
> git push
```

We are now alerted when our deployments succeed or fail. Let's test our Airflow alerting for when a Task fails.

First we'll fix our unit test:
``` Python
# test_python_scripts.py
import unittest

class TestScripts(unittest.TestCase):

    def test_script(self):
        self.assertTrue(1==1)
        # self.assertTrue(1==0)

if __name__ == '__main__':
    unittest.main()
```

Now lets update the script our DAG runs so that it raises an exception:
``` Python
# my_script.py
def fail_sometimes():
    div_by_zero = 1/0   # This will raise an exception
    return 'No Exception!'
```

Let's deploy our changes:
``` Bash
> git status
> git add --all
> git commit -m 'Making Airflow Task fail'
> git push
```

Now let's visit our Airflow Web UI to trigger our DAG. We can find the web address by executing:
``` Bash
> gcloud composer environments describe my-environment \
    --location us-central1 \
    --format="get(config.airflowUri)"
https://ic1434f8836d84236p-tp.appspot.com
```

Let's trigger the DAG (check out [Chapter 5](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_05_dags.md) for more details on interacting with the Airflow UI). Our task will retry once. When it fails on it's second try it will send us an alert in Slack.

## Wrapping Up
We did a lot in this chapter. We set up a Composer Environment, a DAG, a Cloud Function, a Cloud Build Trigger, a Secret, and a Pub/Sub Topic. Hopefully you feel pretty good about how your knowledge has grown over this book such that you're able to tie multiple GCP services together to build your infrastructure. 

This is the last chapter in this book where we will be introducing new tools. While there is a lot more we could have talked about on GCP, you should now have a good understanding of the major components of a typical data engineering stack, and how to stand those components up using GCP.

The next chapter, [Chapter 13](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_13_up_and_running.md), will go over how to get a complete Data Engineering infrastructure up and running.

## Cleaning Up
We used a lot of different GCP services in this chapter, so let's start taking them down.

We'll begin with finding the bucket for our Composer Environment and deleting it:
``` Bash
> gcloud composer environments describe my-environment \
    --location us-central1 \
    --format="get(config.dagGcsPrefix)"
gs://us-central1-my-environment-9567c0a7-bucket/dags
> gsutil rm -r gs://us-central1-my-environment-9567c0a7-bucket
```

Now we can delete the Composer Environment:
``` bash
> gcloud composer environments delete my-environment --location us-central1
```

That's going to take awhile, so let's go to a new terminal session and delete our Cloud Function:
``` bash
> gcloud functions delete send_to_slack
```

We can delete our Pub/Sub Topic:
``` Bash
> gcloud pubsub topics delete cloud-builds
```

And our Secret:
``` Bash
> gcloud secrets delete slack_webhook
```

Finally, let's delete our Cloud Build Trigger:
``` Bash
> gcloud beta builds triggers list
---
createTime: '2021-01-23T14:15:19.273898053Z'
filename: cloudbuild.yaml
github:
  name: cloud-build-test
  owner: Nunie123
  push:
    branch: master
id: 53042473-7323-471d-817d-a8e59939d84a
name: trigger
> gcloud beta builds triggers delete trigger
```

In [Chapter 11: Deployment Pipelines with Cloud Build](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_11_deployment_pipelines.md) I showed you how to delete your GitHub repository. See the instructions there if you want to delete the repo we made in this chapter.

---

Next Chapter: [Chapter 13: Up and Running - Building a Complete Data Engineering Infrastructure](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_13_up_and_running.md)