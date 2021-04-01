---
title: "handoff: Connecting Cloud Data at Scale without Managing a Cluster"
date: 2021-03-20 13:47:40
tags:
---

![data flow](/images/keep-flowing-handoff.gif)

This article describes a way to develop and deploy many data pipelines tasks without managing a cluster.

## Business context

You can skip this section if you are only interested in the technical part. Or you can read this section only to grasp the business implication of this technical article. Before diving into the tech of it, I wanted to provide a brief business context so you understand why we implement it in a certain way.

We are a technology-leveraged service company. One of the services is custom ETL/ELT for many clients. ETL stands for Extracting, Transforming, and Loading of data. Recently, ELT workflow is becoming popular; the data is extracted and loaded on the data warehouse (DWH) first before the transformation is executed by the massively parallel modern DWH engine.

More and more companies, even when their core value is not in tech, started to leverage the cloud applications and their data in order to make their operation lean and effective. For example, D2C (direct to consumer) companies produce niche house-hold products and ship directly to their customers instead of relying on a traditional retail distribution channel. In doing so, The D2C uses the marketing automation platform like Marketo or Pardot, runs ads on Facebook and Google platform, and manage the supply chain with Flexport: All of them produce data about their customers. ETL process extracts data from each cloud services and load on one DWH. By combining the data and analyzing them as a whole, the quality of the customer experience is continuously improved.

More than ever, the need for establishing data pipelines to do ETL is growing. There are companies that provide data connectors for major cloud applications like Salesforce. But the development of the data connectors is much slower than the growth of the cloud applications in the Software as a Service (SaaS) market.  Meanwhile, not every company is interested in developing an internal data engineering team.

![long tail: no cloud apps left behind](/images/long-tail.png)

That is why [fully-managed ETL services](https://handoff.cloud) like ours are becoming the new option: With a fraction of the cost of hiring a data engineer, the non-tech companies get the tailor-made data pipeline service.

## Technical challenges and Initial Solution

If you are the director of data engineer for your company's internal operation, it makes sense to set up an Apache Airflow or subscribe to a managed service by Astronomer or Google Cloud Platform.

Instead, we are a lean team helping others to solve the data integrations of the cloud applications in the long-tail. Simultaneously serving the data pipelines for many different clients change the technical requirements by a lot. The most important part is to provide separate CPU, memory, and storage capacities for each client. We wanted to achieve so with maximum flexibility: When we started, we only had a few data pipelines to manage, but we were expecting the number to rapidly grow.

###  AWS Lambda vs. Fargate

As you might have guessed, we went for a serverless approach, and our first prototype used Amazon Web Services (AWS) Lambda. It was quick to implement. The pay-as-you-go pricing was very attractive instead of idling the virtual machine and deal with the auto-scaling reliably at peak-time. But we soon realized that Lambda has a 15 minutes limit on the execution and it is not a viable platform to run a long-running process.

So we switched to Fargate, another pay-as-you-go serverless platform by AWS. It has no time limit and the life-cycle management of the computing resource and the scaling of it is fully managed by the platform.

Compared to Lambda, Fargate's deployment process is much more complex: While Lambda could be deployed after writing a single file script in the programming language of your choice, Fargate deployment involves creating a Docker image, pushing the image to Elastic Container Registry, creating resources like Virtual Private Cloud and Security Groups, defining the Fargate task and managing the Policy.

## Birth of handoff: A Serverless Data Pipeline Orchestration Framework

The intricated Fargate deployment gave us an idea of implementing a command-line interface (CLI) that we named "handoff". handoff simplifies the development and deployment steps for AWS Fargate. We are also thinking to provide a multi-cloud capability in the future: handoff unifies the commands to deploy to AWS, Microsoft Azure, Google Cloud Platform, and so on.

handoff describes the data pipeline logic in a YAML file. The first thing you would describe is the installation of the modules required by the pipeline:

`project.yml:`

```
version: 0.3
description: Fetch foreign exchange rates

installs:
- venv: extractor
  command: pip install tap-exchangeratesapi
- venv: loader
  command: pip install target-bigquery-partition
```

As you can see, handoff is created primarily for Python. Multiple virtual environments can be automatically generated if you want to install modules in separate environments. In the local testing, the installation is as easy as running:

```
handoff workspace install --project <project_dir> --workspace <workspace_dir>
```

The body of the pipeline logic is groups of tasks in which each command can be linked as Unix pipeline:

```
tasks:
- name: fetch_exchange_rates
  description: Fetch exchange rates and load on BigQuery
  pipeline:
  - command: tap-exchangeratesapi
    args: --config files/tap-config.json
    venv: extractor
  - command: "target-bigquery --config files/target_config.json"
    venv: loader
```

This is particularly useful in deploying memory-efficient tasks such as the ELT task by singer.io framework.

Running locally is as easy as:

```
handoff run local -p <project_dir> -w <workspace_dir>
```

### Doing more than a simple data replication task

handoff also implements simple control flow such as for-each and fork. And of course, you can define a task beyond simple ETL. Look how we automate the task of fetching stock market data, make a GIF animation, and post on Twitter:

<blockquote class="twitter-tweet"><p lang="en" dir="ltr">A fun demo: handoff&#39;s <a href="https://twitter.com/hashtag/datapipeline?src=hash&amp;ref_src=twsrc%5Etfw">#datapipeline</a> can automatically harvest data and post data-driven content to Twitter <a href="https://twitter.com/hashtag/etl?src=hash&amp;ref_src=twsrc%5Etfw">#etl</a> <a href="https://twitter.com/hashtag/analytics?src=hash&amp;ref_src=twsrc%5Etfw">#analytics</a>. <a href="https://t.co/0ITR771mh9">pic.twitter.com/0ITR771mh9</a></p>&mdash; handoff (@handoffcloud) <a href="https://twitter.com/handoffcloud/status/1377477316448636935?ref_src=twsrc%5Etfw">April 1, 2021</a></blockquote> <script async src="https://platform.twitter.com/widgets.js" charset="utf-8"></script>

## Deployment features

We would estimate that authoring a pipeline logic is just 20% of the whole ETL development. More work has to be done to manage:

- Run-time configuration
- Secret keys
- Scheduling
- Logging
- Monitoring
- Containerization

and the list goes on. handoff's project.yml helps to keep things organized and it is intuitive to understand:

```
vars:
- key: gcp_project_id
  value: "handoff-demo"
- key: dataset_id
  value: exchangerate

envs:
- key: GOOGLE_APPLICATION_CREDENTIALS
  value: files/google_client_secret.json

deploy:
  cloud_provider: aws
  cloud_platform: fargate
  resource_group: handoff-etl
  container_image: tap-rest-api-target-bigquery
  task: usgs-earthquakes

schedules:
- target_id: 1
  description: Run everyday at 00:00:00Z
  envs:
  - key: __VARS
    value: 'start_datetime=$(date -Iseconds -d "00:00 yesterday") end_datetime=$(date -Iseconds -d "00:00 today")'
  cron: '0 0 * * ? *'
```

The containerization and the deployment commands are also simple and intuitive:

```
# Push project configuration, supporting files, and secrets to AWS
handoff project push -p <project_dir>

# Build a Docker image and push to ECR
handoff container build -p <project_dir>
handoff container push -p <project_dir>

# Create resource group, bucket, task...
handoff cloud bucket create -p <project_dir>
handoff cloud resources create -p <project_dir>
handoff cloud task create -p <project_dir>

# Schedule a task
handoff cloud schedule create -p <project_dir>
```

Without handoff, you would need to author Cloud Formation file, boto3 commands to handle file transfer to S3, encrypt and store the secrets, configure the CloudWatch for logging and metrics, etc to securely deploy the task and monitor the execution.

For a full tutorial, please visit [https://dev.handoff.cloud](https://dev.handoff.cloud)

We made handoff [a public project on GitHub](https://github.com/anelendata/handoff) so every engineer having a similar challenge can use it. We also welcome collaborators to improve this new framework.

![handoff](/images/this_is_handoff.png)

## What's next?

As we mentioned earlier, we are thinking to support deployment to other cloud platforms such as Microsoft Azure and Google Cloud Platform. Unifying the deployment command for multi-cloud would be very useful in quickly switching the platform.

We also prototyped a web UI built on top of the CLI:

![handoff ui](/images/handoff-ui.gif)

The web UI could be running as a stand-alone application on your local machine as it connects to the cloud resources in the backend, keeping the serverless philosophy. It can also be hosted on a server for enterprise use cases.

## Conclusion

AWS Fargate let us create a cost-effective and scalable solution for ETL deployment, but it hasn't been a popular approach due to the complex deployment process. handoff, a serverless data pipeline orchestration framework simplifies the process. It also manages facilitates the various aspects of the orchestration such as security and monitoring. We made handoff an open-source project so that the data engineering community can benefit from the framework and collaborate for futher innovation.
