---
title: Our recommended Platform Options for Datalake and Data Warehouse
tags:
---
One common approach is to have a data lake as the initial tier of the data store, then create processes to clean and organize the data and store it in a data warehouse.
In CDN logs case, it was convenient to have existing S3 buckets as the data lake for the CDN logs, then use Athena as the transformation process before storing it in BigQuery as the data warehouse.

Similarly, if we can stream the data from the production application to a data lake, that may be more convenient. Such data lake should be readily accessible by the cluster that takes care of the parallel data processing. For table data, I recommend Athena if the data is stored in S3 and BigQuery if the data is in Google Cloud Platform, namely GCP Cloud Storage.

When we created the data warehouse, I recommended BigQuery instead of AWS Athena or Redshift because of the performance and flexibility advantage. Athena is almost equivalent to a distributed SQL engine like Presto, and its interface and data management are not rich enough to be directly connected to analytics applications such as business intelligence tools. Redshfit is a parallelized version of an old version (8.x) of PostgreSQL and it's been quite costly to manage mainly because the computing nodes are deeply coupled with the storage.

While I am more accustomed to deploying web applications on AWS, I prefer GCP much more when it comes to analytics and machine learning infrastructure. I don't know about the future direction of Tiny Works, but if the company will have more opportunities to build data-driven intelligent systems, I continue recommending gathering the data in Google Cloud Platform.

As in the way I proposed in the ELT section in the Squid API doc, we could use Cloud Storage as a data lake from where BigQuery can parse into tables. We could also build more complex data pipelines using Google Dataflow when it becomes necessary. Finally, creating and deploying machine learning models would be much more productive with Google AI Platform: You could build and deploy an ML model without coding with AutoML Table by taking data from BigQuery. AWS also has a similar solution around SageMaker, but I feel Google's system is more modern and I will feel more productive.

I think today's singer.io based approach is fine when we pull data out of API servers. The structured data whose schema won't change often should be directly loaded on BigQuery via a singer.io process. But if we are concerned about the changing schema and unstructured (JSON) data, we may create a data lake first in Google Cloud Storage, and let BigQuery or other high-performance parallel system take care of the cleanup before loading on a data warehouse (BigQuery). That would be my suggestion.

When we create cloud applications, we can plan a multi-cloud deployment relatively easily, but when it comes to data storage, we need to marry to one platform because it is very costly to move the big data around. I think choosing a platform with a rich processing environment is as important as the storage performance itself. Right now, it seems to me GCP feels more productive environment than AWS although a lot of companies invested so much in AWS so they also do analytics and ML in AWS.


----

Hi Daigo,

I'm thinking about the future. We're likely to build other services in the future that need to expose data for analytics. How should we be designing these APIs? Is there a standard API format we should use, that's easy to consume? Should our services be pushing data somewhere, instead of an analytics process pulling?

I like how we established a reusable pattern for CloudFront / Athena data. Can we make better use of this? Is there another pattern in here that we can establish?

I'm just curious to know your thoughts on this.

Thanks,

Dylan
