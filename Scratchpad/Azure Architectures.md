# Azure Architecture proposal

[TOC]

## Terminology

Azure has a lot of terminology that is specific to Azure. Since we're going to use quite a few of them, here we explain them by topic.

### Compute environments

**WebApp:** A web app (also known as [*App Service*](https://learn.microsoft.com/en-us/azure/app-service/overview) or *Web App Service*), is basically
a small server that runs a web application. It only supports certain languages (Python, Java, .NET, C#, and PHP), but Microsoft is responsible for keeping the OS and frameworks up-to-date.
- Run continuously
- Pay per hour

<details>
    <summary>WebApp Pricing</summary>

    Azure uses **ACU**, "Active Compute Units", as a measure for "performance".

    [WebApp pricing](https://azure.microsoft.com/en-us/pricing/details/app-service/linux/).
    We only highlight plans relevant to our use case.
    
      | Plan            | Price per month | RAM   | CPUs | Storage | Performance |
      | --------------- | --------------- | ----- | ---- | ------- | ----------- |
      | Basic B2        | $25             | 3.5GB | 2    | 30GB    | 100 ACU     |
      | Premium v3 P0V3 | $61             | 4GB   | 1    | 250GB   | 195 ACU     |
      | Premium v3 P1V3 | $123            | 8GB   | 2    | 250GB   | 195 ACU     |

    It is possible to run multiple app services on the same "App Service Plan",
    so then multiple web-apps share the same computer.
</details>

**Functions:** Functions (also known as [*Azure Functions*](https://learn.microsoft.com/en-us/azure/azure-functions/overview) or
*serverless compute*) is different from WebApps in that instead of running a
server continuously, it only spawns a new instance when a request is made. As a
result:
- It's a lot cheaper (since no idle time needs to be paid for)
- It can scale automatically (since you just need to add a new instance for every request).
- It has a cold start time (since the server needs to be initialized before it can handle
  requests).

<details>
  <summary>Different Azure Functions plans</summary>

  There are multiple Azure function plans. The ones we'd like to highlight are:
  1. Consumption Plan
  2. Flex Consumption plan
  3. Premium Plan

  Check [Azure
  docs](https://learn.microsoft.com/en-us/azure/azure-functions/functions-scale#service-limits)
  for a more detailed explanation, but in a nutshell:
  - Consumption plan does not allow for any always-ready instance, has the least computing power,
  but is the cheapest of the three.
  - Flex consumption is basically consumption but better: it allows for always-ready instances,
  has more computing power, and has more max ram. (4GB instead of 1.5GB). However, it's more expensive.

</details>

<details>
    <summary> Azure Functions pricing</summary>
    For [Functions](https://azure.microsoft.com/en-us/pricing/details/functions/),
    it's a bit more difficult to calculate, as it uses two metrics:
    - €0.000025 GB-s (seconds compute time for every GB of memory used)
    - €0.371 per million function executions

    The first 100 000 GB-s and 250 000 executions are free.

    Example: suppose we look at our production server, in the last 30 days](https://portal.azure.com/#@agxeed.com/blade/Microsoft_Azure_MonitoringMetrics/Metrics.ReactView/Referer/MetricsExplorer/ResourceId/%2Fsubscriptions%2Fd71a458e-0b74-41b5-8179-d0fe92d2c546%2FresourceGroups%2Fagx-we-p-rg-aav%2Fproviders%2FMicrosoft.Web%2Fsites%2Fagx-we-p-azfun-field-1-aav/TimeContext/%7B%22relative%22%3A%7B%22duration%22%3A2592000000%7D%2C%22showUTCTime%22%3Afalse%2C%22grain%22%3A1%7D/ChartDefinition/{"isCompressed":true,"value":"eJzVk0lPwzAQhf9K5HNMSNTS0iurBOXCdkAcJvYktZTakZd0U%2F87dkIQBQoSByRyssbv%2BZt5o2xIk7EZaGvI5GlD5mi1YN1Zo1FOM5yiBQ4WyGRDBCcTkhiXG6ZFbYWSJuGjFAbDMdLDfDSggzQf0nE6Oqb8sMDjjGdsODhK%2BscutHK1SaBc0gXSmuqSAjRJrVUjOGqTTAXTyqjCHjxinhhh8Z0a1oWTtBBYcZoGI9nGRMIcfVfnTrLQ0NkSmQuHE%2BWkJTGBstRYQijdrWovTTuPqYEF4%2FyNuOiJ3tUl8SCMg0qsW3cIgAtTV7C62WVGb9Cop%2FYDn%2B4YvptkG%2F%2F%2FzO%2BlsObPM%2B%2Bpv8j8OSZW2Crobt082rfQCCSP9ghaelQoHX1DesVcCcnbMJqPQ7Y%2FYZfVMCYVlij5pySE8SWRh3atdhiTWvns2sssJjO%2FzUvVoD4B7SkFVAa76jXkWIU4TGf0G4Rl%2B9ju%2B8svIUHadZZ54%2BoHTbr13%2FP2BTlPiW0%3D"})(https://portal.azure.com/#@agxeed.com/resource/subscriptions/26047d6f-4e24-41ec-ba57-cad424fa1e04/resourceGroups/agx-we-d-rg-aav/providers/Microsoft.Web/sites/agx-we-d-azfun-field-0-aav/metrics) (snapshot on April 11, 2025):
    - 43.07 billion execution units
    - 359 170 executions

    Execution Units are given in MB-milliseconds. To convert to GB-seconds, divide by 1 024 000.
    
       Therefore, we roughly use last month on production:
       - 42 060 GB-s
       - 359 170 executions
    
       After subtracting the free 100 000 GB-s and 250 000 executions, we roughly
       have to pay €0.04 for production per month without any always ready instance.
       (If we do not take into account the free units, we would get a monthly fee of around €1.20 a month.)
    
    
       The always ready pricing is as follows:
       - €0.0000038/GB-s baseline (depends on RAM Provisioned)
       - €0.0000149/GB-s Execution Time (depends on RAM used)
       - €0.370439 per million executions
    
       Per always ready instance of 4GB, this would result in a baseline price of:
       - $\texteuro 0.0000038 * (60*60*24*30) \text{sec/month} * 4 \text{GB} = \texteuro 39.40$ a month
    
       (The price for execution time and executions are less than €1, so negligible.)

</details>

### Storage

#### Storage Accounts

There are many storage options available in Azure within a **Storage account**:
- [Blob storage:](https://learn.microsoft.com/en-us/azure/storage/blobs/storage-blobs-overview)
  General purpose file store
  
- [Queue Storage:](https://learn.microsoft.com/en-us/azure/storage/queues/storage-queues-introduction) 
  Storing messages in a queue up-to 64Kb per message. Often used in [Web-Queue-Worker architecture](https://learn.microsoft.com/en-us/azure/architecture/guide/architecture-styles/web-queue-worker)
- [Table Storage:](https://learn.microsoft.com/en-us/azure/storage/tables/table-storage-overview) 
  Storing structured data (key value pairs). It's actually really similar to (albeit [slower than](https://learn.microsoft.com/en-us/azure/cosmos-db/table/support?toc=https%3A%2F%2Flearn.microsoft.com%2Fen-us%2Fazure%2Fstorage%2Ftables%2Ftoc.json&bc=https%3A%2F%2Flearn.microsoft.com%2Fen-us%2Fazure%2Fbread%2Ftoc.json)) a NoSQL database / CosmosDB, see our database section.

- [Azure Files:](https://learn.microsoft.com/en-us/azure/storage/files/storage-files-introduction)
  File storage for mounting, similar to how we mount `Z:` in AgXeed.

- [Azure Data Lake Storage:](https://learn.microsoft.com/en-us/azure/storage/blobs/data-lake-storage-introduction)
  Mainly used for large-scale data-analytics, out-of-scope.

All of these save files onto a hard drive.

For us, blob story is important to store the routing binaries.

There's a variety of prices for blob storage

<details>
    <summary> Blob storage pricing</summary>
    For [Blob Storage](https://azure.microsoft.com/en-gb/pricing/details/storage/blobs/#pricing), there are multiple price tiers with different trade offs.
    
    The relevant ones for us:
    
    | Storage tier | Price per GB | Write (per 10 000) | Read (per 10 000) | Read (per GB) |
    | ------------ | ------------ | ------------------ | ----------------- | ------------- |
    | Hot          | €0.0204      | €0.0602            | €0.0047           | Free          |
    | Cool¹        | €0.00927     | €0.1204            | €0.0121           | €0.0093       |
    ¹There exists a penalty for deleting files. If you delete it before 30 days, you will get charged for 30 days.
    
    
    For a cost estimation, suppose we have 3000 routes a month, 70% of which succeed and result a protobuf binary of an average 3MB. So, in total, this would result in:
    - 9GB, 2100 writes, 2100 reads
    - Hot: €0.1836 for storage, and  $0.012642 + 0.000987 + 0.0 \approx \texteuro 0.01$ for monthly read/write fee.
    - Cold: €0.08343 for storage, and $0.025284 + 0.002541 + 0.0837 \approx \texteuro 0.109$ for monthly read/write fee.
    
    Since we keep storage of previous months (until say a few years), cold storage would remain cheaper for us, although the price is kinda negligible.
</details>



#### Databases

A database is a service that is made to store data with a higher access frequency than the database storage options. In Azure, these are stored on SSDs while the other storage options are stored on hard drive, and other optimizations are made to improve latency for reading/writing.

In general, there are two types of databases:
- **SQL Databases:** these store data in tables with rows and columns. You can relate certain data to other data. Also known as relational databases.
- **NoSQL databases**: these store data in documents, which can be retrieved by their unique ID and contain basically key-value pairs like JSON. 

The most popular SQL database is **PostgreSQL** and most popular No-SQL database is MongoDB.
There are also Azure specific alternatives: Azure SQL Database (SQL) and **CosmosDB** (No-SQL).

Since we're not really interested in relational databases, but instead just want to store the state of a run inside a database, we're looking at NoSQL databases. MongoDB does not really exist in Azure, so instead we'll look at CosmosDB.


<details>
    <summary>Database pricing</summary>

     Pricing is a little difficult to compare, because they use different units.
    
    The [PostgresSQL](https://azure.microsoft.com/en-us/pricing/details/postgresql/server/) database costs us:
    - €24.609 a month for 1vCore and 2GiB RAM
    -  €0.102 per GB per month

    [CosmosDB](https://azure.microsoft.com/en-us/pricing/details/cosmos-db/autoscale-provisioned/) costs:
    - 

    For [Blob Storage](https://azure.microsoft.com/en-gb/pricing/details/storage/blobs/#pricing), there are multiple price tiers with different trade offs.
    
    The relehttps://azure.microsoft.com/en-us/pricing/details/postgresql/server/vant ones for us:
    
    | Storage tier | Price per GB | Write (per 10 000) | Read (per 10 000) | Read (per GB) |
    | ------------ | ------------ | ------------------ | ----------------- | ------------- |
    | Hot          | €0.0204      | €0.0602            | €0.0047           | Free          |
    | Cool¹        | €0.00927     | €0.1204            | €0.0121           | €0.0093       |
    ¹There exists a penalty for deleting files. If you delete it before 30 days, you will get charged for 30 days.
    
    
    For a cost estimation, suppose we have 3000 routes a month, 70% of which succeed and result a protobuf binary of an average 3MB. So, in total, this would result in:
    - 9GB, 2100 writes, 2100 reads
    - Hot: €0.1836 for storage, and  $0.012642 + 0.000987 + 0.0 \approx \texteuro 0.01$ for monthly read/write fee.
    - Cold: €0.08343 for storage, and $0.025284 + 0.002541 + 0.0837 \approx \texteuro 0.109$ for monthly read/write fee.
    
    Since we keep storage of previous months (until say a few years), cold storage would remain cheaper for us, although the price is kinda negligible.
</details>











### Existing architecture

The existing architecture

### Proposed new environments




###







## High-level overview

The motion planning team is responsible for the software of the motion planning system.
In the simplest sense, the architecture looks like:

<div class="figure">
  <img src="Images/architecture-simple.png" alt="image" width="500">
  <div class="caption">
    <i>
    A high-level overview of the Azure architecture for the motion planning team.
    </i>
  </div>
</div>
</br>

As input, we get the data for a route from the portal, and as output we send the route to the
portal.

### API

When the portal sends a request to the motion planning system, the request is received by the API.
This API is inside the `AgxRoute.API` project and is the entry point of our project.

<div class="figure">
  <img src="Images/architecture-api.png" alt="image" width="500">
  <div class="caption">
    <i>
    The AgxRoute.API project is the entry point of our project.
    </i>
  </div>
</div>
</br>

The AgxRoute.API project creates a [REST API](https://blog.postman.com/rest-api-examples/) as
interface for the motion planning system. Our API uses a serverless architecture running in
[Azure Functions](https://learn.microsoft.com/en-us/azure/azure-functions/functions-overview?pivots=programming-language-csharp).
This means each call to each endpoint creates a new compute instance that handles the request.
The project creates multiple endpoints that can be polled for data such as:

- `GET /api/openapi/v3.json` returns JSON of the openapi spec.
- `POST /api/v1/recipe/extract-id` returns the task id of the provided protobuf file.

For more information on the API development, see the
[WebApp API documentation](@ref Prototype/AgxRoute.WebApp/Documentation/AspNetCoreApi.md)
and the
[Azure Functions API documentation](@ref Prototype/AgxRoute.Functions/Documentation/AzFunctionsAPI.md).

## Motion-planning software

All of the motion planning software runs as a library within the API. This library is loaded by the
Azure Functions compute instance for each new request.

## Authentication

### Access Key Authentication

Azure functions can authenticate endpoints with _Access Keys_. This is a simple solution that limits
access to the functions; requests must include an access key in the request. For more information
about
[Working with access keys](https://learn.microsoft.com/en-us/azure/azure-functions/function-keys-how-to?tabs=azure-portal).

In this model we will need a secure method of distributing the access key. This may be able to be
done using Azure's
[Key Vault](https://learn.microsoft.com/en-us/azure/app-service/app-service-key-vault-references?toc=/azure/azure-functions/toc.json)

### API Management

API Management is a service offered by Azure that allows a greater degree of control and security
over an API.

It provides features such as:

- API Gateway: Acts as a front door for your APIs, handling all incoming API requests and routing
  them to the appropriate backend services.
- Security: Offers various security mechanisms including OAuth 2.0, JWT validation, IP filtering,
  and rate limiting to protect your APIs from unauthorized access and abuse.
- Analytics: Provides detailed analytics and insights into API usage, performance, and errors,
  helping you to monitor and optimize your APIs.
- Developer Portal: A customizable portal where developers can discover, learn about, and test your
  APIs. It also provides documentation and code samples to help developers get started quickly.
- Policy Management: Allows you to define and enforce policies on your APIs, such as transformation,
  validation, and caching, without changing the backend services.

<div class="figure">
  <img src="Images/architecture-api-management.png" alt="image" width="700">
  <div class="caption">
    <i>
    The Azure architecture with the additional API Management (APIM) component.
    </i>
  </div>
</div>
</br>

This may be implemented in the future together with portal team.
