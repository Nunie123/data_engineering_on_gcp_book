# Up and Running: Data Engineering on the Google Cloud Platform
The completely free E-Book for setting up and running a Data Engineering stack on Google Cloud Platform.

NOTE: This book is currently incomplete. If you find errors or would like to fill in the gaps, read the [Contributions section](https://github.com/Nunie123/data_engineering_on_gcp_book#user-content-contributions).

## Table of Contents
[Preface](https://github.com/Nunie123/data_engineering_on_gcp_book) <br>
**Chapter 1: Setting up a GCP Account** <br>
[Chapter 2: Setting up Batch Processing Orchestration with Composer and Airflow](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_2_orchestration.md) <br>
[Chapter 3: Building a Data Lake with Google Cloud Storage (GCS)](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_3_data_lake.md) <br>
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

# Chapter 1: Setting up a GCP Account

GCP usually allows you to perform the same task many different ways. They offer a console through their website, CLI tools, and libraries for a variety of languages. When possible, we'll stick to using CLI tools and Python libraries, as this allows us to put our commands for instantiating and configuring our infrastructure into version control.

However, there are going to be some things that must be done through GCP's website, and setting up an account is one of those things.

In this chapter I'll also show you how to install the command line tools, so we can avoid using the console in the future.

## Signing up for GCP
Head to [cloud.google.com](https://cloud.google.com/) and click "Get Started for Free". <br>
![GCP home page](images/gcp_home_page.png)

Next you'll link your GCP account to an existing Google account. GCP may suggest one by default. If you do not have a Google account you must create one before proceeding.

GCP will then request personal information, including billing information. Unfortunately, you must provide billing information before you can create an account. However, new accounts receive a $300 credit towards any charges, and GCP will notify you before it starts charging you for cloud services.

## Creating a Project

In GCP, a Project acts as a namespace under which services are organized, permissions are set, and billing is tracked. For example, if you wanted to copy a file in a GCS bucket you would need to provide the Project associated with that bucket, otherwise GCP will be unable to find the bucket. You can have resources from different Projects talk to each other without much difficulty, the Projects generally just act as namespaces to help you organize your infrastructure. For example, we'll be creating a Project for our development infrastructure, and a separate Project for our production infrastructure.

On the GCP Console you'll always see your current project listed at the top of the screen. If you just created a new account then GCP has created the "My First Project" Project for you. Click on the Project name, then in the window that appears select "Create New Project". 

![GCP Dashboard](images/gcp_dashboard.png)

On this next page you'll provide your project name, which can be anything (I'm using `de-book-dev`). This Project will be for our development environment. Also note that while your Project name can be anything, your Project ID must be unique across all of GCP, so GCP will auto-generate it based on the name you provide, but you can choose to pick your own. Click the "Create" button.

![GCP Create Project Page](images/gcp_create_project.png)

Once we have our GCP command line tools set up (discussed below) we will be able to create a project from the terminal using the command:
``` bash
> gcloud projects create de-book-dev
```

## Installing the GCP Command Line Tools

GCP offers several command line utilities, but fortunately they are all easily installed by installing and configuring Google Cloud SDK. Google Cloud SDK can be installed though your favorite package manager such as [Homebrew](https://formulae.brew.sh/cask/google-cloud-sdk) or [Snap](https://snapcraft.io/install/google-cloud-sdk/debian), or though curl (`curl https://sdk.cloud.google.com | bash`). You can find more installation options [here](https://cloud.google.com/sdk/docs/install).

Once installed, run:
``` bash
> gcloud init
```
Following the prompts you will create a profile and link it to the Google account you used to log in to GCP. Your browser will open and authenticate using the credentials you set for your GCP account. At the prompt, select the Project you wish to work on (for me that's `de-book-dev`). You can change this profile or create a new profile by running `gcloud init` again.

You now have access to the `gcloud`, `bq`, and `gsutil` command line tools. `gsutil` is used for managing Google Cloud Storage, `bq` is used for managing BigQuery, and `gcloud` is a general purpose utility. All three will be used within this book.

### Setting up a Service Account
We'll be using a Service Account to authenticate with GCP. You can set up a Service Account through the Console by selecting the hamburger menu in the upper-left, then going to "IAM & Admin" > "Service Accounts". From this page select "CREATE SERVICE ACCOUNT" at the top.

![GCP Service Accounts Page](images/gcp_service_accounts_page.png)

Now you'll be prompted for the name of your service account and an optional description. Since this account is going to be for accessing Composer in my Dev environment I named my account "composer-dev". Clicking to the next page prompts us to select a role from a drop-down menu. Select "Cloud Composer" > "Composer Administrator", and then "Continue". We can ignore the last page and select "Done". You will be brought back to the main Service Accounts page, showing all your accounts, including the one you just created. On the account you just created select the three dots on the far right, and then select "Create key". On the menu that pops up select "JSON" and then "CREATE". You will be prompted to download a JSON file. Keep this file safe, as this is your key for accessing Composer.

![GCP create key menu](images/gcp_create_key.png)

You can also set up a service account through the `gsutil` utility:
``` bash
> gcloud iam service-accounts create 'composer-dev'
```
In order to set up a role for this account you'll need it's emails, which can found with:
``` bash
> gcloud iam service-accounts list
DISPLAY NAME                            EMAIL                                               DISABLED
composer-dev                            composer-dev@de-book-dev.iam.gserviceaccount.com    False
```

Finally, we download the JSON file that is our key:
``` bash
> gcloud iam service-accounts keys create ./keys/my_secret_composer_key.json \
    --iam-account='composer-dev@de-book-dev.iam.gserviceaccount.com'
```

Now that we have our key saved locally we can define a variable in our shell so our command line tools can find our key:
``` bash
> export GOOGLE_APPLICATION_CREDENTIALS="/path/to/keys/de-book-dev.json"
```
This variable will only last as long as your terminal session. Every time you open a new terminal session and plan to use the GCP command line tools you should start by running the above command.

---

Next Chapter: [Chapter 2: Batch Processing Orchestration with Composer and Airflow](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_d_orchestration.md) <br>