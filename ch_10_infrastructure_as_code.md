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
[Chapter 9: Managing Credentials with Google Secret Manager](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_9_secrets.md) <br>
**Chapter 10: Infrastructure as Code with Terraform** <br>
[Chapter 11: Deployment Pipelines with Cloud Build](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_11_deployment_pipelines.md) <br>
[Chapter 12: Monitoring and Alerting](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_12_monitoring.md) <br>
Chapter 13: Start to Finish - Building a Complete Data Engineering Infrastructure <br>
Appendix A: Example Code Repository


---

# Chapter 10: Infrastructure as Code with Terraform

Much of a Data Engineer's responsibility is to manage their tools. So far in this book we've discussed Composer, GCS, BigQuery, Cloud Functions, Dataproc, Pub/Sub, and Secret Manager. We've been managing these resources through command line scripts, which is useful for learning, but not a good way to handle your production environment. We want to define these resources in text files and have those files in source control. This is called Infrastructure as Code.

Terraform is a tool that allows you to define your infrastructure, and the various permissions to access that infrastructure, in a series of configuration files. Terraform is not owned or managed by Google, but it does work well with GCP.

Terraform is not a GCP service, so we'll be running Terraform on our local machine.

## Installing Terraform
If you're on a Mac, Terraform can easily be installed through [Homebrew](https://brew.sh/):
``` Bash
> brew tap hashicorp/tap
> brew install hashicorp/tap/terraform
```

Terraform also works on Linux and Windows. More installation methods are available [here](https://learn.hashicorp.com/tutorials/terraform/install-cli).

If you want to enable auto-complete for Terraform on your bash or zsh shell, then execute the following command then restart your terminal:
``` Bash
> terraform -install-autocomplete
```

That's it. We now have Terraform on our local machine. Next we'll create some configuration files to define our infrastructure.
## Creating Your Configuration Files
One of the main benefits of using Terraform, or Infrastructure as Code more generally, is the ability to define our infrastructure in a way that allows us to put it into source control. In this section we'll be writing the configuration files that will go into our source control (e.g. git).

While Terraform allows you to get sophisticated in your setup, it also allows you to get up and running with minimal code. Let's create our `main.tf` file:

``` JS
// This "terraform block" tells Terraform it will need to look in the google registry when identifying which resources to deploy
terraform {
    required_providers {
        google = {
        source = "hashicorp/google"
        version = "3.5.0"
        }
    }
}

// This "provider block" configures your google account.
provider "google" {
    credentials = file("../keys/de-book-dev-secret-key.json") # provide the file path to your GCP secret key file
    project = "de-book-dev"
    region  = "us-central1"
    zone    = "us-central1-c"
}

// This "resource block" configures the specific infrastructure you wish to deploy
resource "google_storage_bucket" "de-book-terraform-test" {
    name = "de-book-terraform-test-1234567654321"   # This is the name of the bucket to be created
    location = "US"
    force_destroy = false   # This setting prevents us from deleting a bucket if there are files within
    storage_class = "STANDARD"
}
```

This code allows us to create a bucket called "de-book-terraform-test-1234567654321". While the above code only creates a single resource (a GCS bucket), we can add as many resource blocks as we need to this file to define all the infrastructure we need. We'll show this in the **Updating your Infrastructure** section, below.

While our infrastructure configuration files, like the one above, should go into source control, Terraform will also generate some configuration files for internal use that are not valuable to add to source control. For those we can create a `.gitignore` file:
``` text
.tfstate
.terraform
.tfplan
```

## Deploying Your Infrastructure to GCP
Now that we've defined our infrastructure, let's deploy it. In the same directory as our `main.tf` file, execute the following:

``` Bash
> terraform apply

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # google_storage_bucket.de-book-terraform-test will be created
  + resource "google_storage_bucket" "de-book-terraform-test" {
      + bucket_policy_only = (known after apply)
      + force_destroy      = false
      + id                 = (known after apply)
      + location           = "US"
      + name               = "de-book-terraform-test-1234567654321"
      + project            = (known after apply)
      + self_link          = (known after apply)
      + storage_class      = "STANDARD"
      + url                = (known after apply)
    }

Plan: 1 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value:
```

Before allowing us to deploy our infrastructure, Terraform prints an execution plan with all of the actions it will take. This is especially useful when multiple engineers are working on the same infrastructure (as is often the case). If someone else has modified your infrastructure, but that change is not present in your files, you will see it in this execution plan. So be sure to actually read this plan, otherwise you could end up accidentally rolling back infrastructure your colleagues deployed. And no one wants to be that guy.

But in this case, we see just the one change we were expecting: a new bucket is being added. So let's type "yes" and deploy our bucket:
``` Bash
  Enter a value: yes

google_storage_bucket.de-book-terraform-test: Creating...
google_storage_bucket.de-book-terraform-test: Creation complete after 1s [id=de-book-terraform-test-1234567654321]

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

Let's confirm that our bucket was created:
``` Bash
> gsutil ls

gs://de-book-terraform-test-1234567654321/
```

Obviously we'll be using a lot more infrastructure in production. I provide an example Terraform file with more resources listed in Appendix A. Documentation for the full list of infrastructure you can deploy is [here](https://registry.terraform.io/providers/hashicorp/google/latest/docs).
## Updating your Infrastructure
We can expect our infrastructure needs to continue to change. Fortunately implementing that change is as simple as updating our Terraform file. Let's create a Pub/Sub Topic, a subscription for that Topic, and set `force_destroy` to `true` for our existing bucket. Our `main.tf` file now looks like:
``` JS
terraform {
    required_providers {
        google = {
        source = "hashicorp/google"
        version = "3.5.0"
        }
    }
}

provider "google" {
    credentials = file("../keys/de-book-dev-2d2abed79f9f.json")
    project = "de-book-dev"
    region  = "us-central1"
    zone    = "us-central1-c"
}

resource "google_storage_bucket" "de-book-terraform-test" {
    name = "de-book-terraform-test-1234567654321" 
    location = "US"
    force_destroy = true   # Updated this to true
    storage_class = "STANDARD"
}

// Added the resources below
resource "google_pubsub_topic" "terraform-test" {
  name = "terraform-test-topic"
}

resource "google_pubsub_subscription" "example" {
  name  = "example-subscription"
  topic = google_pubsub_topic.terraform-test.name
  ack_deadline_seconds = 20
}
```

Now we can apply our changes (typing "yes" when prompted):
``` Bash
> terraform apply
google_storage_bucket.de-book-terraform-test: Refreshing state... [id=de-book-terraform-test-1234567654321]

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create
  ~ update in-place

Terraform will perform the following actions:

  # google_pubsub_subscription.example will be created
  + resource "google_pubsub_subscription" "example" {
      + ack_deadline_seconds       = 20
      + id                         = (known after apply)
      + message_retention_duration = "604800s"
      + name                       = "example-subscription"
      + path                       = (known after apply)
      + project                    = (known after apply)
      + topic                      = "terraform-test-topic"

      + expiration_policy {
          + ttl = (known after apply)
        }
    }

  # google_pubsub_topic.terraform-test will be created
  + resource "google_pubsub_topic" "terraform-test" {
      + id      = (known after apply)
      + name    = "terraform-test-topic"
      + project = (known after apply)

      + message_storage_policy {
          + allowed_persistence_regions = (known after apply)
        }
    }

  # google_storage_bucket.de-book-terraform-test will be updated in-place
  ~ resource "google_storage_bucket" "de-book-terraform-test" {
        bucket_policy_only = false
      ~ force_destroy      = false -> true
        id                 = "de-book-terraform-test-1234567654321"
        labels             = {}
        location           = "US"
        name               = "de-book-terraform-test-1234567654321"
        project            = "de-book-dev"
        requester_pays     = false
        self_link          = "https://www.googleapis.com/storage/v1/b/de-book-terraform-test-1234567654321"
        storage_class      = "STANDARD"
        url                = "gs://de-book-terraform-test-1234567654321"
    }

Plan: 2 to add, 1 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

google_pubsub_topic.terraform-test: Creating...
google_storage_bucket.de-book-terraform-test: Modifying... [id=de-book-terraform-test-1234567654321]
google_storage_bucket.de-book-terraform-test: Modifications complete after 0s [id=de-book-terraform-test-1234567654321]
google_pubsub_topic.terraform-test: Creation complete after 1s [id=projects/de-book-dev/topics/terraform-test-topic]
google_pubsub_subscription.example: Creating...
google_pubsub_subscription.example: Creation complete after 2s [id=projects/de-book-dev/subscriptions/example-subscription]

Apply complete! Resources: 2 added, 1 changed, 0 destroyed.
```

We see that Terraform added two resources and changed one resource, as we expected.

While it is still possible to provision GCP infrastructure through other means, such as the GCP Console or command line, in practice our Terraform file or files should represent our entire infrastructure on GCP.

We can remove a resource by simply deleting the resource block from our Terraform file. Let's delete out bucket:
``` JS
terraform {
    required_providers {
        google = {
        source = "hashicorp/google"
        version = "3.5.0"
        }
    }
}

provider "google" {
    credentials = file("../keys/de-book-dev-2d2abed79f9f.json")
    project = "de-book-dev"
    region  = "us-central1"
    zone    = "us-central1-c"
}

// Our GCS resource used to be here

resource "google_pubsub_topic" "terraform-test" {
  name = "terraform-test-topic"
}

resource "google_pubsub_subscription" "example" {
  name  = "example-subscription"
  topic = google_pubsub_topic.terraform-test.name
  ack_deadline_seconds = 20
}
```
Now let's apply our changes:
``` Bash
> terraform apply
google_pubsub_topic.terraform-test: Refreshing state... [id=projects/de-book-dev/topics/terraform-test-topic]
google_storage_bucket.de-book-terraform-test: Refreshing state... [id=de-book-terraform-test-1234567654321]
google_pubsub_subscription.example: Refreshing state... [id=projects/de-book-dev/subscriptions/example-subscription]

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  - destroy

Terraform will perform the following actions:

  # google_storage_bucket.de-book-terraform-test will be destroyed
  - resource "google_storage_bucket" "de-book-terraform-test" {
      - bucket_policy_only = false -> null
      - force_destroy      = true -> null
      - id                 = "de-book-terraform-test-1234567654321" -> null
      - labels             = {} -> null
      - location           = "US" -> null
      - name               = "de-book-terraform-test-1234567654321" -> null
      - project            = "de-book-dev" -> null
      - requester_pays     = false -> null
      - self_link          = "https://www.googleapis.com/storage/v1/b/de-book-terraform-test-1234567654321" -> null
      - storage_class      = "STANDARD" -> null
      - url                = "gs://de-book-terraform-test-1234567654321" -> null
    }

Plan: 0 to add, 0 to change, 1 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value: yes

google_storage_bucket.de-book-terraform-test: Destroying... [id=de-book-terraform-test-1234567654321]
google_storage_bucket.de-book-terraform-test: Destruction complete after 1s

Apply complete! Resources: 0 added, 0 changed, 1 destroyed.
```

## Cleaning Up
We used Terraform to create three resources, then destroy one of them. So we should have two resources left:
``` Bash
> terraform state list
google_pubsub_subscription.example
google_pubsub_topic.terraform-test
```

We could delete these Pub/Sub resources with the command line, as we did in Chapter 8. However, we can also tell Terraform to destroy all of the resources it manages:
``` Bash
> terraform destroy
google_pubsub_topic.terraform-test: Refreshing state... [id=projects/de-book-dev/topics/terraform-test-topic]
google_pubsub_subscription.example: Refreshing state... [id=projects/de-book-dev/subscriptions/example-subscription]

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  - destroy

Terraform will perform the following actions:

  # google_pubsub_subscription.example will be destroyed
  - resource "google_pubsub_subscription" "example" {
      - ack_deadline_seconds       = 20 -> null
      - id                         = "projects/de-book-dev/subscriptions/example-subscription" -> null
      - labels                     = {} -> null
      - message_retention_duration = "604800s" -> null
      - name                       = "example-subscription" -> null
      - path                       = "projects/de-book-dev/subscriptions/example-subscription" -> null
      - project                    = "de-book-dev" -> null
      - retain_acked_messages      = false -> null
      - topic                      = "projects/de-book-dev/topics/terraform-test-topic" -> null

      - expiration_policy {
          - ttl = "2678400s" -> null
        }
    }

  # google_pubsub_topic.terraform-test will be destroyed
  - resource "google_pubsub_topic" "terraform-test" {
      - id      = "projects/de-book-dev/topics/terraform-test-topic" -> null
      - labels  = {} -> null
      - name    = "terraform-test-topic" -> null
      - project = "de-book-dev" -> null
    }

Plan: 0 to add, 0 to change, 2 to destroy.

Do you really want to destroy all resources?
  Terraform will destroy all your managed infrastructure, as shown above.
  There is no undo. Only 'yes' will be accepted to confirm.

  Enter a value: yes

google_pubsub_subscription.example: Destroying... [id=projects/de-book-dev/subscriptions/example-subscription]
google_pubsub_subscription.example: Destruction complete after 1s
google_pubsub_topic.terraform-test: Destroying... [id=projects/de-book-dev/topics/terraform-test-topic]
google_pubsub_topic.terraform-test: Destruction complete after 1s

Destroy complete! Resources: 2 destroyed.
```

---

Next Chapter: [Chapter 11: Deployment Pipelines with Cloud Build](https://github.com/Nunie123/data_engineering_on_gcp_book/blob/master/ch_11_deployment_pipelines.md)