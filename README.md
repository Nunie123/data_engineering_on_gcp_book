# Data Engineering in the Cloud: Start to Finish
The completely free E-Book for setting up and running a Data Engineering stack on Google Cloud Platform.

NOTE: This book is currently incomplete. If you find errors or would like to fill in the gaps, read the **Contributions** section below.

## Table of Contents
**Preface** <br>
Chapter 1: Setting up a GCP Account
Chapter 2: Batch Processing Orchestration with Composer and Airflow
Chapter 3: Building a Data Lake with Google Cloud Storage (GCS)
Chapter 4: Building a Data Warehouse with BigQuery
Chapter 5: Setting up Event-Triggered Pipelines with Cloud Functions
Chapter 6: Parallel Processing with DataProc and Spark
Chapter 7: Streaming Data with Pub/Sub
Chapter 8: Managing Credentials with Google Secret Manager
Chapter 9: Creating a Local Development Environment
Chapter 10: Infrastructure as Code with Terraform
Chapter 11: Continuous Integration with Jenkins
Chapter 12: Monitoring and Alerting
Appendix A: Example Code Repository


---

# Preface
This is a book designed to teach you how to set up and maintain a production-ready data engineering stack on [Google Cloud Platform](https://cloud.google.com/)(GCP). In each chapter I will discuss an important component of Data Engineering infrastructure. I will give some background on what the component is for and why it's important, followed by how to implement that component on GCP. I'll conclude each chapter by referencing similar services offered by other cloud providers.

By the end of this book you will know how to set up a complete tech stack for a Data Engineering team using GCP. 

Be warned that this book is opinionated. I've chosen a stack that has worked well for me and that I believe will work well for many data engineering teams, but it's entirely possible the infrastructure I describe in this book will not be a great fit for your team. If you think there's a better way than what I've laid out here, I'd love to hear about it. Please refer to the **Contributions** section, below.

## Who This Book Is For
This book is for people with coding familiarity that are interested in setting up professional data pipelines and data warehouses using Google Cloud Platform. I expect the readers to include:
* Data Engineers looking to learn more abut GCP.
* Junior Data Engineers looking to learn best practices for building and working with data engineering infrastructure.
* Software Engineers, DevOps Engineers, Data Scientists, Data Analysts, or anyone else that is tasked with performing Data Engineering functions to help them with their other work.

This book assumes your familiarity with SQL and Python (if you're not familiar with Python, you should be able to muddle through with general programming experience). If you do not have experience with these languages (particularly SQL) it is recommended you learn these languages and then return to this book.

This book covers a lot of ground. Many of the subjects we'll cover in just part of a chapter will have entire books written about them. I will provide references for further research. Just know that while this book is comprehensive in the sense that it provides all the information you need to get a stack up and running, there is still plenty of information a Data Engineer needs to know that I've omitted from this book.

Finally, there are a vast array of Data Engineering tools that are in use. I cover many popular tools for Data Engineering, but many more have been left out of this book due to brevity and my lack of experience with them. If you feel I left off something important, please read the **Contributions** section below.

## How to Read This Book
This book is divided into chapters discussing major Data Engineering concepts and functions. Most chapters is then divided into three parts: an overview of the topic, implementation examples, and references to other articles and tools. 

If you're looking to use this book as a guid to set up your Data Engineering infrastructure from scratch, I recommend you read this book front-to-back. Each chapter describes a necessary component of the tech stack, and they are ordered such that the infrastructure described in one chapter builds of the previously described infrastructure.

Likely, many people will find their way to this book trying to solve a specific problem (e.g. how to set up alerting on GCP's Composer/Airflow service). For these people I've tried to make each chapter as self-contained as possible. When I use infrastructure created in a previous chapter I'll always provide a link to the previous chapter where it's explained.

The best way to learn is by doing, which is why each chapter provides code samples. I encourage you to build this infrastructure with me, as you read through the chapters. Included with this book in **Appendix A** I've provided an example of what your infrastructure as code will look like.

## Contributions

You may have noticed: this book is hosted on GitHub. This results in three great things:
1. The book is hosted online and freely available.
2. You can make pull requests.
3. You can create issues.

If you think the book is wrong, missing information, or otherwise needs to be edited, there are two options:
1. **Make a pull request** (the preferred option). If you think something needs to be changed, fork this repo, make the change yourself, then send me a pull request. I'll review it, discuss it with you, if needed, then add it in. Easy peasy. If you're not very familiar with GitHub, instructions for doing this are [here](https://gist.github.com/Chaser324/ce0505fbed06b947d962). If your PR is merged it will be considered a donation of your work to this project. You agree to grant a [Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/) license for your work. You will be added the the **Contributors** section on this page once your PR is merged.
2. **Make an issue**. Go to the [Issues tab](https://github.com/Nunie123/data_engineering_on_gcp_book/issues) for this repo on GitHub, click to create a new issue, then tell me what you think is wrong, preferably including references to specific files and line numbers.

I look forward to working with you all.

## Contributors
**Ed Nunes**. Ed lives in Chicago and works as a Data Engineer for [Zoro](https://www.zoro.com). Feel free to reach out to him on [LinkedIn](https://www.linkedin.com/in/ed-nunes-b0409b14/).


## License
This book is licensed under the [Attribution-NonCommercial-NoDerivatives 4.0 International (CC BY-NC-ND 4.0)](https://creativecommons.org/licenses/by-nc-nd/4.0/) license.

---

Next Chapter: Chapter 1: Setting up a GCP Account