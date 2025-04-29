# Azure Architecture Proposal

[TOC]

### Abstract

In this investigation, we evaluate different Azure architectures for our routing calculations. We
first assess the current architecture, concluding that it cannot scale and will reach end-of-life in
2026. After evaluating several alternatives, we propose a new architecture that addresses these
issues. It runs 2–3× faster on complex tasks, matches performance on simple tasks, and is 10×
cheaper. Under higher load, it scales significantly better, performing up to 7× faster during three
concurrent requests. We conclude that the proposed architecture is a viable solution for our API.

## 1. Introduction

Currently, we use Azure to perform routing calculations in response to user
requests. As part of our ongoing efforts to improve our API, we decided to also
investigate our Azure architecture. During this investigation, we identified
several limitations in the current setup. To address these, we propose a new
architecture that offers greater efficiency, lower costs, and improved
future-proofing.

Since Azure contains a lot of specific services and features, we first start by explaining the
relevant Azure terminology in [Section 2](#2-azure-terminology). In [Section 3](#3-architecture), we
describe two architectural models for our routing computations, analyzing the current architecture
and introducing new candidate architectures.

We evaluate these candidates in the following sections: [Section 4](#4-cost-evaluation)
presents a cost analysis of the various Azure components, while
[Section 5](#5-performance-evaluation) assesses the performance of the candidate architectures.
We conclude the investigation in [Section 6](#6-conclusion), where we propose a new architecture
based on these evaluations.

If any unfamiliar terms arise, a [glossary](./DeveloperGuide.md#terms) is available for reference.

## 2. Azure Terminology

Azure has a lot of terminology that is specific to Azure. Since we're going to use quite a few of
them, here we explain them by topic.

### Compute environments

**WebApp:** A web app (also known as [*App Service*](https://learn.microsoft.com/en-us/azure/app-service/overview)
or *Web App Service*) is basically a small server that runs a web application. It only supports
certain languages (Python, Java, .NET, C#, and PHP), but Microsoft is responsible for keeping the OS
and frameworks up-to-date.

- Run continuously
- Pay per hour

**Functions:** Functions (also known as [*Azure Functions*](https://learn.microsoft.com/en-us/azure/azure-functions/overview) or
*serverless compute*) are different from WebApps in that instead of running a
server continuously, they only spawn a new instance when a request is made. As a
result:

- They're a lot cheaper (since no idle time needs to be paid for)
- They can scale automatically (since you just need to add a new instance for every request)
- They have a cold start time (since the server needs to be initialized before it can handle
  requests)

<details>
  <summary>Different Azure Functions plans</summary>

  There are multiple Azure function plans. The ones we'd like to highlight are:

  1. Consumption Plan
  2. Flex Consumption Plan
  3. Elastic Premium Plan

  Check [Azure
  docs](https://learn.microsoft.com/en-us/azure/azure-functions/functions-scale#service-limits)
  for a more detailed explanation, but in a nutshell:

  - Consumption plan does not allow for any always-ready instance, has the least computing power,
  but is the cheapest of the three.
  - Flex consumption is basically consumption but better: it allows for always-ready instances,
  has more computing power, and has more max RAM (4 GB instead of 1.5 GB). However, it's more expensive.
  - Elastic Premium gets really expensive really quickly.

  We'll mainly focus on the 'Flex Consumption' plan, because of its flexibility of creating
  always-ready instances and more RAM. Only if it becomes too expensive, we'll consider the
  'Consumption' plan.

</details>

### Storage

#### Storage Accounts

There are many storage options available in Azure within a **Storage account**:
- [Blob storage:](https://learn.microsoft.com/en-us/azure/storage/blobs/storage-blobs-overview)
  General purpose file store.

- [Queue Storage:](https://learn.microsoft.com/en-us/azure/storage/queues/storage-queues-introduction)
  Storing messages in a queue up to 64 KB per message. Often used in [Web-Queue-Worker architecture](https://learn.microsoft.com/en-us/azure/architecture/guide/architecture-styles/web-queue-worker).

- [Table Storage:](https://learn.microsoft.com/en-us/azure/storage/tables/table-storage-overview)
  Storing structured data (key value pairs). It's actually really similar to (albeit [slower than](https://learn.microsoft.com/en-us/azure/cosmos-db/table/support?toc=https%3A%2F%2Flearn.microsoft.com%2Fen-us%2Fazure%2Fstorage%2Ftables%2Ftoc.json&bc=https%3A%2F%2Flearn.microsoft.com%2Fen-us%2Fazure%2Fbread%2Ftoc.json)) a NoSQL database / CosmosDB, see our database section.

- [Azure Files:](https://learn.microsoft.com/en-us/azure/storage/files/storage-files-introduction)
  File storage for mounting, similar to how we mount `Z:` in AgXeed.

- [Azure Data Lake Storage:](https://learn.microsoft.com/en-us/azure/storage/blobs/data-lake-storage-introduction)
  Mainly used for large-scale data-analytics, out-of-scope.

All of these save files onto a hard drive. For us, **blob storage** is important to store the routing
binaries, while **queue storage** is useful for storing a queue of jobs (explained in the
[asynchronous job model section](#asynchronous-job-model)).

#### Databases

A database is a service designed to store data with higher access frequency than the database
storage options. In Azure, these are stored on SSDs (and contain other optimizations for better
latency for reading/writing).

In general, there are two types of databases:

- **SQL Databases:** These store data in tables with rows and columns. You can relate certain data
  to other data. Also known as relational databases.
- **NoSQL Databases:** These store data in documents, which can be retrieved by their unique ID and
  contain key-value pairs similar to JSON.

The most popular SQL database is **PostgreSQL** and the most popular NoSQL database is MongoDB. There
are also Azure-specific alternatives: Azure SQL Database (SQL) and **CosmosDB** (NoSQL).

Since we're not primarily interested in relational databases, but instead want to store the state
of a run inside a database, we're focusing on NoSQL databases. Our preferred choice is CosmosDB
as it integrates well with the .NET and Azure environment, unless there is a compelling reason (like
cost) to select another database.

## 2. Architecture

### Sketch of the situation

The motion planning team handles routing calculations. Other functionalities exist (like finding the
closest endpoint on the boundary), but motion planning calculations are our team's main
computational expense and the focus of our proposal.

#### Request-Response model

When you think of drafting the architecture of the routing calculations, an initial idea might look
like this:

<div class="figure">
  <img src="Images/AzureArchitectures_request_response.svg" alt="image" width="500">
  <div class="caption">
    <i>
        The simplest high-level architecture: Request-Response model.
    </i>
  </div>
</div>

Here, the Azure environment is just a simple function that returns the route given a set of task
parameters. This is known as the _Request-Response architecture_, and works well for small-scale
tasks.

However, in our scenario, this architecture has a couple of downsides:

1. It is not possible to see the status of the route.
2. Azure cuts off HTTP connections after 230 seconds.
   (See [footnote 1](https://learn.microsoft.com/en-us/azure/azure-functions/functions-scale#timeout))
3. From the portal side, having long-lived connections open blocks resources.

So, what alternatives exist for this architecture?

1. Jobs: Follow the asynchronous job pattern (explained below in more detail).
2. Web hooks: Let Portal open a public-facing webhook that our backend can call when routing is
   finished and to give status updates.
3. WebSockets: Using WebSockets to maintain a bi-directional connection.
4. gRPC, Server Streaming RPC: The client sends a single request, and the server sends back a
   sequence of responses.

From these options, we'll choose option 1, because it's the same architecture as we previously
used and requires no additional features from portal side. Reasons to reject alternatives:

- Option 2: Difficulties for authentication, requires more setup.
- Option 3: Not supported by OutSystems.
- Option 4: Not supported by OutSystems.

#### Asynchronous Job model

Instead of calculating the route immediately, we instead create a job that calculates the route. We
repeatedly poll the status of the job until it's finished, and once it's finished we can retrieve
the result.

In an architectural view, it looks like this:

<div class="figure">
  <img src="Images/AzureArchitectures_async_job.svg" alt="image" width="700">
  <div class="caption">
    <i>
        The main high-level architecture: Asynchronous Job model.
    </i>
  </div>
</div>

Thus, planning a route consists of three phases:

1. The portal team creates a routing job.
2. The portal team polls the status of the routing job.
3. Once the status returns either `completed` or `failed`, the portal team requests the result.

This has some benefits over the request-response model:

- No longer the 230s timeout problem
- All requests return immediately
- Can return the status of the job

The downsides are:

- More complex implementation and architecture
- Debugging and logging requires more attention

#### Relevant Terminology

For the next sections, we will discuss different architectures and their pros and cons.

To help understand the different architectures, we will use the following terminology:

- **Orchestrator**: A service that manages the job.
- **Worker**: A service that performs the actual work of a job.
- **Trigger**: An event that starts the worker.
  The trigger can be an HTTP request, a message in a queue and more.

See the following image for more details:

<div class="figure">
  <img src="Images/AzureArchitectures_terms_explained.svg" alt="image" width="700">
  <div class="caption">
    <i>
        Illustration of the terminology used in the next sections.
    </i>
  </div>
</div>

Moreover, we talk about two different ways of scaling:

- **Vertical scaling**: Scaling up the resources of a single instance.
- **Horizontal scaling**: Scaling out by adding more instances.

In practical terms, with horizontal scaling you can handle a lot more requests simultaneously, while
vertical scaling increases the computation speed of each request.

### Existing architecture

The existing architecture follows the Asynchronous Job model, where:

1. Uses Azure Functions for both Orchestrator and Worker.
2. The state is stored in memory.
3. The results are stored in memory.

<div class="figure">
  <img src="Images/AzureArchitectures_existing.svg" alt="image" width="700">
  <div class="caption">
        The existing architecture follows the Asynchronous Job model, but stores everything
        in-memory, resulting in a maximum of one Azure Functions worker.
  </div>
</div>

Because the state is stored in memory, there can be at most one active instance (as a second
instance would not have access to the state in the first instance).

It's actually really similar to using a WebApp, but instead using Azure Functions for it, making it
twice as expensive for production and still having the cold-start problem.

Pros:

- Relatively easy to implement
- Cheesy way to get free resources for dev/test

Cons:

- Not scalable at all.
- Similar to a WebApp, but twice as expensive for the same computation power.
- Requires the .NET in-memory model, which will be [deprecated on 10 November, 2026](https://azure.microsoft.com/en-us/updates?id=retirement-support-for-the-inprocess-model-for-net-apps-in-azure-functions-ends-10-november-2026)

### Proposed new architectures

#### Request-Response model Architecture

All the request-response models suffer from the following two major downsides:

1. No status checks
2. Maximum runtime of 230s timeout

However, we would still like to test these architectures for setting base benchmarks. Moreover, it
allows us to compare the raw compute performance of the different compute environments.

##### WebApp

This is the most similar to our existing architecture. For simplicity, we don't store the state in
memory but directly return it, although adding the status checks is not too difficult.

<div class="figure">
  <img src="Images/AzureArchitectures_webapp.svg" alt="image" width="500">
  <div class="caption">
        The simple Request-Response model, implemented with a WebApp.
        Mainly used for benchmarking.
  </div>
</div>

Pros:

- Simplest implementation
- Most similar to previous architecture, but cheaper.

Cons:

- Does not scale horizontally, unless we make it a lot more complex (Load balancer, K8s, ...)
- This implementation does not support status checks
- Cannot compute tasks >230s.

##### Azure Functions

Similar to the WebApp, but does scale horizontally as any new messages arrive.

<div class="figure">
  <img src="Images/AzureArchitectures_functions.svg" alt="image" width="500">
  <div class="caption">
        The simple Request-Response model, implemented with an Azure Functions.
        Mainly used for benchmarking, as this is the "best-case scenario".
  </div>
</div>

There's an alternative implementation without writing the state to the database, which we use to
compare how much overhead is added by connecting a database.

Pros:

- Simplest implementation
- Scales horizontally really well
- Stores the result

Cons:

- This implementation does not support status checks
- Cannot compute tasks >230s.

#### Asynchronous Job model Architecture

##### Options explained

Let's have a look at the asynchronous job model again.

<div class="figure">
  <img src="Images/AzureArchitectures_terms_explained.svg" alt="image" width="700">
  <div class="caption">
        The main high-level architecture: Asynchronous Job model.
  </div>
</div>

We can make several choices for each of the components. Let's discuss them separately.

**Orchestrator**

The Orchestrator is responsible for managing the workflow and coordinating the execution of tasks.
Since it's not computationally intensive, we can implement it as both a WebApp or Azure Functions.

Trade-off:

- WebApp: Might improve cold-start performance, but requires an additional project, is more
  expensive, might be resource bound on smaller plans.
- Azure Functions: Might have worse cold-start performance, but is more cost-effective and
  has better resources, and we only need a single AgxRoute.API project.

**Worker**

The worker needs to do the actual routing calculations, which can be computationally intensive.
This part should scale out horizontally, so only Azure Functions would be a suitable choice.

**Trigger**

The trigger is responsible for starting the Azure Functions workflow and cannot
be an HTTP trigger due to the 230s timeout. Possible triggers include:

1. Queue Storage Trigger
2. CosmosDB Trigger
3. Event Grid Trigger
4. Service Bus Trigger
5. Event Hub Trigger

To keep the scope manageable, we will focus on the first three options. The Event Grid trigger is
included in the test because it uses a different architecture (publish-subscribe) to the other two
(polling).

**State storage**

To store the state of the runs¸ we can either use a database or a cache.
Specifically:

1. Database: CosmosDB
2. Cache: Redis Cache

For the database, we only focussed on using CosmosDB because it's a NoSQL database that's fully
supported in Azure and C#, while also being the cheapest option, as seen in our
[Cost Analysis document](./Evaluation/Cost.md).

To keep the scope manageable, we did not test the Redis Cache as it is
[more expensive](https://azure.microsoft.com/en-us/pricing/details/cache/) than CosmosDB.
However, if we notice that the database is a limiting factor, it might be worth investigating
it in the future.

**Result Storage**

For storing the results, we only considered using a Blob Storage:

- Want to store binaries, so need to be able to store unstructured data.
- Want to store results for a longer period, resulting in significant storage space.

#### Microsoft recommended Architecture

There are multiple Microsoft recommendations for running our workflow:

1. The [Web-Queue-Worker architecture](https://learn.microsoft.com/en-us/azure/architecture/guide/architecture-styles/web-queue-worker),
  an implementation of the Asynchronous Job model.
2. [Durable Functions](https://learn.microsoft.com/en-us/azure/azure-functions/durable/durable-functions-overview#async-http),
  an API over Azure Functions used to manage long-running tasks.


Thus, following (1), Microsoft would recommend us:

- Orchestrator: WebApp
- Worker: Azure Functions
- Trigger: Queue Trigger
- State: Redis Cache / CosmosDB
- Results: Blob Storage

<div class="figure">
  <img src="Images/web-queue-worker-physical.png" alt="image" width="700">
  <div class="caption">
        The Web-Queue-Worker architecture recommended by Microsoft.
  </div>
</div>

The Durable Functions is an interesting alternative that encompasses both the Trigger, Worker, and
State. It requires rewriting our orchestrator code a little bit, as we need to call the correct
functions to store our state. This results in:

- Orchestrator: WebApp/Azure Functions
- Worker: Durable Functions
- Trigger: Durable Functions
- State: Durable Functions
- Results: Blob Storage

In Azure Functions, there even exists an always-ready version of the Durable Functions, which
we can test.

## 4. Cost-evaluation

### Method

We evaluated multiple configurations for each of the following components:

- **Compute environments:** Azure Functions and Azure Web Apps
- **Databases:** Azure CosmosDB (different versions) and PostgreSQL (different versions)
- **Storage:** Azure Blob Storage Hot Tier, Cool Tier and Cold Tier
- **Event triggers:** Queue Storage trigger, Event Grid trigger, CosmosDB trigger

For the evaluation, we'll consider three scenarios:
- Current production load
- 10x production load
- 100x production load

These figures have been determined together with product management and are based on expected
growth in the coming years. Only the costs for the production environments are calculated, as these
are the most significant costs for our team.

Where possible, we use data gathered from the production environment. This can either
come from Azure's monitoring tools or from our
[data-analysis scripts](https://bitbucket.org/agxeed/agx_routing_snippets/src/master/data-analysis-azure/).

Moreover, we use data from March/April, as this is the month that is a good average
representation of the computation load throughout the year according to product management.

All cost figures are based on current Azure pricing (as of evaluation date, April 11 2025) and
reflect North Europe region pricing unless otherwise specified.

### Results

#### Compute environments costs

To limit the scope of this comparison, we'll only consider flex-consumption and elastic premium.

The regular consumption plan is slightly cheaper than the flex-consumption plan,
but we would like to have the option of always-ready instances (and the
performance of flex-consumption is better).

##### Monthly usage estimate

For WebApps, we pay for the server per month, so we don't need to consider any
usage estimate for the costs. (In practice, the server will respond slower with a higher load.)

For functions, we have two measures of computations:
- GB-s: amount of computation we perform per second per GB RAM.
- Executions: number of calls to the functions being made in total

For this comparison, we're using the [data used in our production server of the last 30 days](da.gd/azurestats)
(snapshot on April 11, 2025):
- 43.07 billion execution units (MB-ms)
- 359 170 executions

Where $43.07\text{ billion MB-ms} / 1\ 024\ 000 = 42 060\text{ GB-s}$

Therefore:

| Load            | Computation (GB-s) | Function executions |
| --------------- | ------------------ | ------------------- |
| 1x prod. load   | 42 060             | 359 170             |
| 10x prod. load  | 420 600            | 3 591 700           |
| 100x prod. load | 4 206 000          | 35 917 000          |


##### Azure Web Apps

For a look at all the different plans, check the [WebApp pricing page](https://azure.microsoft.com/en-us/pricing/details/app-service/linux/)

Azure uses **ACU**, "Active Compute Units", as a measure for "performance". All Azure Functions run
on 210 ACU, for the web apps it depends per plan.

Here's a highlight of the plans that are important to us:

| Plan            | Price per month | RAM    | CPUs | Storage | Performance |
| --------------- | --------------- | ------ | ---- | ------- | ----------- |
| Basic B1        | €12.17          | 1.75GB | 1    | 30GB    | 100 ACU     |
| Basic B2        | €23.66          | 3.5GB  | 2    | 30GB    | 100 ACU     |
| Premium v3 P0V3 | €56.79          | 4GB    | 1    | 250GB   | 195 ACU     |
| Premium v3 P1V3 | €113.58         | 8GB    | 2    | 250GB   | 195 ACU     |

It's possible to run multiple app services on the same "App Service Plan", then multiple web-apps
share the same server.

##### Functions Flex-consumption pricing

The [flex-consumption pricing](https://azure.microsoft.com/en-us/pricing/details/functions/)
consists of two parts:
1. **On-demand** — A computer that spins up once the event is (has cold-start problem)
2. **Always-ready instance** -- An active computer that continuously runs on the background, but
  results in a fixed monthly costs.

| Meter                         | Free Grant (Per Month) | Pay as you go                    |
| ----------------------------- | ---------------------- | -------------------------------- |
| On Demand Execution Time      | 100,000 GB-s           | €0.000025/GB-s                   |
| On Demand Total Executions    | 250,000 executions     | €0.371 per million executions    |
| Always Ready Baseline         |                        | €0.0000038/GB-s                  |
| Always Ready Execution Time   |                        | €0.0000149/GB-s                  |
| Always Ready Total Executions |                        | €0.370439 per million executions |

Where GB-s is the execution time per GB RAM per second.

Per always ready instance, this results in a baseline price of:

- For 2GB instances: $\text{€}0.0000038 * (60*60*24*30) \text{ sec/month} * 2 \text{ GB} = \text{€} 19.70$ a month
- For 4GB instances: $\text{€}0.0000038 * (60*60*24*30) \text{ sec/month} * 4 \text{ GB} = \text{€} 39.40$ a month

##### Elastic Premium pricing

| Meter             | vCPU | RAM    | Price / hour | Price per month |
| ----------------- | ---- | ------ | ------------ | --------------- |
| Elastic Premium 1 | 1    | 3.5 GB | €0.211       | €135.86         |
| Elastic Premium 2 | 2    | 7 GB   | €0.423       | €271.72         |

Depending on how high we set the scale-out, the costs could be even higher. A
higher scale-out does mean a better throughput when multiple tasks are planned,
but any computing minutes need to be paid for. Since we need a minimum of 1
active instance, the price per month is the lower bound of the costs.

(As a side-note, since we have disabled scale-out in our current setup, it's
almost equivalent to using the P1V3 app-service plan, but then making it more
than twice as expensive.)

##### Comparison

| Service                                                            | Cost/month [1x prod. load] | Cost/month [10x prod. load] | Cost/month [100x prod. load] |
| ------------------------------------------------------------------ | -------------------------- | --------------------------- | ---------------------------- |
| Azure Functions (Flex Consumption)                                 | €0.04                      | €9.25                       | €115.14                      |
| Azure Functions (Flex Consumption) + 1 always ready instance (4GB) | €39.78                     | €46.53                      | €134.03                      |
| Azure Functions (Premium, current plan)                            | €271.72                    | €300 - €542                 | €500 - €3000                 |
| WebApp Basic B1                                                    | €12.17                     | N/A                         | N/A                          |
| WebApp Basic B2                                                    | €23.66                     | N/A                         | N/A                          |
| WebApp Premium v3 P0V3 (less than our current production plan)     | €56.79                     | €113.58                     | €340.74                      |
| WebApp Premium v3 P1V3 (similar to current production plan)        | €113.58                    | €227.16                     | €681.48                      |

The following assumptions have been made:
1. For the always-ready instance, half of the requests are handled by that instance.
2. Premium plans and WebApps follow the #nodes in a cluster to handle burst requests
  (very optimistic, likely need more):
  - For 10x production load, we require two concurrent servers.
  - For 100x production load, we require six concurrent servers.
3. We did not take into account any costs for load balancing or caching with the WebApps.
  It's difficult to get estimates for these (probably need to model with a poisson distribution to
  get a more accurate estimate).

<details>
   <summary>Calculation details</summary>

   Free grant contains 100 000 GB-s and 250 000 executions.

   Case: No always ready instance

   Calculation details (1x production load):
   - GB-s [on-demand]: $42 060 - 100 000 = 0$ GB-s [free grant exceeds our execution time]
   - executions [on-demand]: $359 170 / 2 - 250 000 = 109 170$ executions
   - **Total:** $109 170 * 0.371 / 1 000 000$ = €0.04

   Calculation details (10x production load):
   - GB-s [on-demand]: $420 600 - 100 000 = 320 600$ GB-s
   - executions [on-demand]: $3 591 700 - 250 000 = 3 341 700$ executions
   - **Total:** $320 600 * 0.000025 + 3 341 700 * 0.371 / 1 000 000$ = €9.25

   Calculation details (100x production load):
   - GB-s [on-demand]: $4 206 000 - 100 000 = 4 106 000$ GB-s
   - executions [on-demand]: $35 917 000 - 250 000 = 33 667 000$ executions
   - **Total:** $4 106 000 * 0.000025 + 33 667 000 * 0.371 / 1 000 000$ = €115.14

   Case: 1x always ready instance (4GB)

   Calculation details (1x production load):
   - GB-s [on-demand]: $42 060 / 2 - 100 000 = 0$ GB-s [free-grant]
   - executions [on-demand]: $359 170 - 250 000 = 0$ executions (on-demand) [free-grant]
   - GB-s [always-active]: $42 060 / 2 = 21 030$ GB-s (always-active)
   - executions [always-active]: $359 170 / 2 = 179 585$ executions (on-demand)
   - **Total (without baseline):** $21 030 * 0.0000149 + 179 585 * 0.370439 / 1 000 000 = \text{€}0.38$
   - **Total:** $39.40 + 0.04 = \text{€}39.78$

   Calculation details (10x production load):
   - GB-s on-demand: $420 600 / 2 - 100 000 = 110 300$ GB-s (on-demand)
   - executions on-demand: $3 591 700 / 2 - 250 000 = 1 545 850$ executions (on-demand)
   - GB-s always-active: $420 600 / 2 = 210 300$ GB-s (always-active)
   - executions always-active: $3 591 700 / 2 = 1 795 850$ executions (on-demand)
   - **Total (without baseline):** $110 300 * 0.000025 + 210 300 * 0.0000149 + (1 795 850 * 0.371 + 1 545 850 * 0.370439) / 1 000 000 = \text{€}7.13$
   - **Total:** $39.40 + 7.13 = \text{€}46.53$

   Calculation details (100x production load):
   - GB-s on-demand: $4 206 000 / 2 - 100 000 = 2 003 000$ GB-s (on-demand)
   - executions on-demand: $35 917 000 / 2 - 250 000 = 17 708 500$ executions (on-demand)
   - GB-s always-active: $4 206 000 / 2 = 2 103 000$ GB-s (always-active)
   - executions always-active: $35 917 000 / 2 = 17 958 500$ executions (on-demand)
   - **Total (without baseline):** $2 003 000 * 0.000025 + 2 103 000 * 0.0000149 + (17 708 500 * 0.371 + 17 958 500 * 0.370439) / 1 000 000 = \text{€}94.63$
   - **Total:** $39.40 + 94.63 = \text{€}134.03$

   (Although in the final scenario, perhaps we need more than 1x always ready instance.)

   All costs have been rounded to the nearest cent.

</details>

##### Conclusion

1. For development/testing:
  - Choosing Azure Functions (Flex Consumption) saves money over using webapps B1 and B2 and is more future-proof.
2. For production:
  - Choosing Azure Functions (Flex Consumption with an always active instance)
   saves a lot of money over the other production options.
  - The WebApp Premium v3 P1V3 might be a superior option over Elastic Premium 2 if scaling-out is
    not needed. (This does change the architecture of the source code, however.)
3. Future test: Consider testing 2GB flex consumption, this saves half the money of 4GB per
   always-ready instance.

#### Database Costs

##### Database monthly usage estimate

Using our production data, we get a rough estimate for the usage of our database:

| Load            | Requests Units (RU) | Storage (GB) | Consistent throughput (RU/s) |
| --------------- | ------------------- | ------------ | ---------------------------- |
| 1x prod. load   | 290 000             | 0.01         | 2                            |
| 10x prod. load  | 2 900 000           | 0.1          | 20                           |
| 100x prod. load | 29 000 000          | 1            | 200                          |

<details>
   <summary> Rough estimation of database costs</summary>

   For the Database costs, we'll make use of the production data:
   - January: 720 runs (75% success rate)
   - February: 2554 runs (69% success rate)
   - March: 7681 (64% success rate)
   - April [first 9 days]: 3059 (63% success rate) --> roughly equivalent to 10 000 runs a month

   It seems like there's a correlation between #runs and success rate.
   For these calculations, we'll assume an average run rate of 10 000 and a success rate of 65%.

   There are multiple scenarios we can consider. Here, we only consider the scenario where we store
   the status of each run, which would look like this:
   ```
   {
   "id": "789ea3f6-64d9-4ee8-a8b5-3e710fb66774",
   "jobId": "789ea3f6-64d9-4ee8-a8b5-3e710fb66774",
   "Status": "Completed",
   "TimeSpent": 11.0269823,
   "RouteId": "00000000-0000-0000-0000-000000000000",
   "ErrorMessage": null,
   "LastUpdated": "2025-04-11T14:18:32.6991156+00:00"
   }
   ```

   This file itself is 285 bytes, but for ease of calculation we'll use 1KB.

   If we consider, per run, that we perform the following operations:
   1. Create: 1 create operation of 1KB (small status document, JSON with around 8 fields)
   2. Patch: 2 updates operations of 1KB (could optimize in patch to lower costs if necessary)
   3. Get: 4 get operations (of 1KB, job id is partition key, 3 status updates on average per run)

   Database costs are expressed in Request Units (RUs). We're using [this official table](https://learn.microsoft.com/en-us/azure/cosmos-db/plan-manage-costs#estimating-serverless-costs)
   for estimates:
   4. Create: 5 RUs
   5. Update: 10 RUs
   6. Read: 1 RU

   Thus, we get a total of $5 + 2*10 + 4*1 = 29$ RUs per run.

   This results in a total, per month, of:
   - RUs: $10 000 * 29 = 290 000$ RUs
   - Storage: $10 000 * 1\text{ KB} = 10\text{ MB}$

   Which would translate to roughly: $290 000 RUs / (26 * 8 * 3600\text{ s}) ≈ 0.387 RU/s
   (Assuming a uniform distribution for requests for 9AM - 5PM, 6 days a week, for 30 days.)

   Since requests will never be uniformly distributed over the day, let's just consider 2 RU/s as a
   baseline to catch a peak moment for requests.

</details>

##### Database pricing

For CosmosDB pricing, it depends on the plan we choose:
- [Serverless](https://azure.microsoft.com/en-us/pricing/details/cosmos-db/serverless/)
  costs €0.263 per million requests
- [Autoscale provisioned](https://azure.microsoft.com/en-us/pricing/details/cosmos-db/autoscale-provisioned/)
  costs €8.113 for 100 RU/s per month. (minimum)
- [Standard provisioned](https://azure.microsoft.com/en-us/pricing/details/cosmos-db/standard-provisioned/)
  costs €21.634 for 400 RU/s per month. (minimum)

All of these cost €0.232 per GB per month for storage.

For PostgreSQL, we also have two options (considering [flexible server](https://azure.microsoft.com/en-us/pricing/details/postgresql/flexible-server/),
  since the standard server is getting deprecated.):
- Burstable: 1 vCore, 2GB RAM: €12.169 per month.
- Basic (provisioned): 2 vCores, 8GB RAM: €133.588 per month.

Storage costs: €0.118 per GB per month.

##### Results

| Configuration                       | Cost/month [1x prod. load] | Cost/month [10x prod. load] | Cost/month [100x prod. load] |
| ----------------------------------- | -------------------------- | --------------------------- | ---------------------------- |
| CosmosDB Serverless                 | €0.08                      | €0.76                       | €7.63                        |
| CosmosDB autoscaling (min 100 RU/s) | €8.11                      | €8.11                       | ~€10                         |
| CosmosDB standard (min 400 RU/s)    | €21.63                     | €21.63                      | €21.63                       |
| PostgreSQL (burstable)              | €12.17                     | €12.17                      | €12.17                       |
| PostgreSQL (provisioned, basic)     | €133.59                    | €133.59                     | €133.59                      |

Assumptions made:
- Data will only be kept 1 month in the database.
- For 100x production load, we might get some 200 RU/s peaks, but since this is never consistent
  but only occasionally, the autoscaling will be triggered only occasionally.
- The basic version of PostgreSQL (burstable) might not be sufficient for 100x production load.

##### Conclusion

1. CosmosDB is affordable and future proof, so no need to check other databases.
2. Future test: Consider doing provisioned autoscaling instead of serverless for a
  potential speed improvement.
3. Even when we consider 100x production load, the database does not seem to significantly impact
  the final costs.

#### Blob storage costs

##### Monthly usage estimate

For our current production usage, we'll use the actual estimate of 10 000 runs a month, see the
details of the [database usage estimates](#database-monthly-usage-estimate) for more detail.

Since two out of three runs succeeded, we'll use an estimate of 6500 runs that
require protobuf binaries to save. We're using an estimate of 5MB per protobuf
binary (I've seen 3.4MB and 7MB binaries), resulting in:
- $5 \text{ MB} * 6500 = 32.5 \text{ GB}$ of protobuf binaries every month

**Scenario 1: Saving only the protobuf**

For every binary, we'll assume 1 write and 1.2 read operations. Therefore:

| Load            | Storage (GB) | #writes | #reads  | Reads (per GB) |
| --------------- | ------------ | ------- | ------- | -------------- |
| 1x prod. load   | 32.5         | 6500    | 8125    | 39             |
| 10x prod. load  | 325          | 65 000  | 30 000  | 390            |
| 100x prod. load | 3250         | 650 000 | 300 000 | 3900           |

**Scenario 2: Saving the protobuf + run_info.json**

The run_info.json file is negligible in terms of size (4 KB at most), so we'll just assume that the
number of reads and writes are doubled (but no significant increase in the reads per GB or storage).

| Load            | Storage (GB) | #writes   | #reads    | Reads (per GB) |
| --------------- | ------------ | --------- | --------- | -------------- |
| 1x prod. load   | 32.5         | 13 000    | 16 250    | 39             |
| 10x prod. load  | 325          | 130 000   | 162 500   | 390            |
| 100x prod. load | 3250         | 1 300 000 | 1 625 000 | 3 900          |

##### Pricing

For [Blob Storage](https://azure.microsoft.com/en-gb/pricing/details/storage/blobs/#pricing), there
are multiple price tiers with different trade offs.

The relevant ones for us:

| Storage tier | Price per GB | Write (per 10 000) | Read (per 10 000) | Read (per GB) |
| ------------ | ------------ | ------------------ | ----------------- | ------------- |
| Hot          | €0.0204      | €0.0602            | €0.0047           | Free          |
| Cool¹        | €0.00927     | €0.1204            | €0.0121           | €0.0093       |
| Cold²        | €0.00334     | €0.2168            | €0.1204           | €0.0278       |

¹There exists a penalty for deleting files. If you delete it before 30 days, you will get charged for 30 days.
²Similar to cool, but then the penalty is for 90 days.

##### Results

**Scenario 1:** Saving only the protobuf

Monthly costs (deleting artifacts after 1 month):

| Configuration | Cost/month [1x prod. load] | Cost/month [10x prod. load] | Cost/month [100x prod. load] |
| ------------- | -------------------------- | --------------------------- | ---------------------------- |
| Hot storage   | €0.71                      | €7.06                       | €70.59                       |
| Cool storage  | €0.75                      | €7.52                       | €75.21                       |
| Cold storage  | €1.43                      | €14.31                      | €143.15                      |

Monthly costs (deleting artifacts after 1 year):

| Configuration | Cost/month [1x prod. load] | Cost/month [10x prod. load] | Cost/month [100x prod. load] |
| ------------- | -------------------------- | --------------------------- | ---------------------------- |
| Hot storage   | €8.00                      | €79.99                      | €799.89                      |
| Cool storage  | €4.07                      | €40.66                      | €406.61                      |
| Cold storage  | €2.63                      | €26.26                      | €262.55                      |

Monthly costs (deleting artifacts after 3 years):

| Configuration | Cost/month [1x prod. load] | Cost/month [10x prod. load] | Cost/month [100x prod. load] |
| ------------- | -------------------------- | --------------------------- | ---------------------------- |
| Hot storage   | €23.91                     | €239.11                     | €2391.09                     |
| Cool storage  | €11.30                     | €112.97                     | €1129.67                     |
| Cold storage  | €5.23                      | €52.31                      | €523.07                      |

**Scenario 2:** Saving the protobuf + run_info.json

Monthly costs (deleting artifacts after 1 month):

| Configuration | Cost/month [1x prod. load] | Cost/month [10x prod. load] | Cost/month [100x prod. load] |
| ------------- | -------------------------- | --------------------------- | ---------------------------- |
| Hot storage   | €0.75                      | €7.49                       | €74.89                       |
| Cool storage  | €0.84                      | €8.40                       | €84.02                       |
| Cold storage  | €1.67                      | €16.70                      | €167.02                      |

Monthly costs (deleting artifacts after 1 year):

| Configuration | Cost/month [1x prod. load] | Cost/month [10x prod. load] | Cost/month [100x prod. load] |
| ------------- | -------------------------- | --------------------------- | ---------------------------- |
| Hot storage   | €8.04                      | €80.42                      | €804.19                      |
| Cool storage  | €4.15                      | €41.54                      | €415.42                      |
| Cold storage  | €2.86                      | €28.64                      | €286.43                      |

Monthly costs (deleting artifacts after 3 years):

| Configuration | Cost/month [1x prod. load] | Cost/month [10x prod. load] | Cost/month [100x prod. load] |
| ------------- | -------------------------- | --------------------------- | ---------------------------- |
| Hot storage   | €23.95                     | €239.54                     | €2395.39                     |
| Cool storage  | €11.38                     | €113.85                     | €1138.48                     |
| Cold storage  | €5.47                      | €54.69                      | €546.95                      |

##### Conclusion

1. Cold storage is the better option for us if we intend to keep the artifacts for longer than a
  month. Rough estimate:
   - Hot storage is the cheapest if we keep artifacts <=1 month
   - Cool storage is the cheapest if we keep artifacts between 1 month and 4.5 months
   - Cold storage is the cheapest for artifacts >4.5 months
2. For our current production usage, costs are negligible, even if we store the artifacts for three
   years.
3. For the 10x production scenario, storage has an impact to the total cost.
4. For the 100x production scenario, if we want to keep the artifacts for multiple years,
  storage is going to be our main contributor to the total cost.
5. We could optimize the scenario even more if we expect more reads, by mixing Hot/Cool/Cold storage
  together. (Where artifacts >3 months are placed into cold storage.)
6. Saving an additional `run_info.json` does not impact the pricing in any meaningful way.

Open question: is there any performance difference between Hot/Cool/Cold storage?

#### Event Processing Costs

##### Monthly usage estimates

For our current production usage, we'll use an estimate of 10 000 runs a month, see the details of
the [database usage estimates](#database-monthly-usage-estimate) for more detail.

For events we need to store the whole field as a request. An average field in
the Diagnostic folders is roughly 31.2KB, so we'll take 32KB as an estimate per request.

Every run has two requests (one event push, one event read), so we get a total of:
- 20 000 operations a month.
- $20 000 * 32 \text{ KB}$ = 640 MB of data

| Load            | Operations | Data (GB) |
| --------------- | ---------- | --------- |
| 1x prod. load   | 20 000     | 0.64      |
| 10x prod. load  | 200 000    | 6.40      |
| 100x prod. load | 2 000 000  | 64        |


##### Pricing

[Queue storage](https://azure.microsoft.com/en-us/pricing/details/storage/queues/):
- €0.0417 per GB
- €0.38 per million operations

[Event Grid](https://azure.microsoft.com/en-us/pricing/details/event-grid/)
- €0.556 per million operations (per 64 KB, which one of our operation fits in)
- First 100 000 requests are free.

[CosmosDB Serverless](https://azure.microsoft.com/en-us/pricing/details/cosmos-db/serverless/)
- €0.263 per million requests
- €0.232 storage per GB

For Durable Functions, we assume the same pricing model holds as Queue storage, since it uses
a queue storage in the background.

##### Results

| Service               | Cost/month [1x prod. load] | Cost/month [10x prod. load] | Cost/month [100x prod. load] |
| --------------------- | -------------------------- | --------------------------- | ---------------------------- |
| Queue Storage         | €0.03                      | €0.34                       | €3.43                        |
| Event Grid            | €0.00                      | €0.06                       | €1.06                        |
| CosmosDB (serverless) | €0.15                      | €1.54                       | €15.37                       |

For the storage, it's assumed that we delete all requests after one month.

##### Conclusion

1. All triggers are very affordable and the cost shouldn't bottleneck the choice of triggers.

#### Cost Summary

In this cost analysis, we compared the various factors that impact our costs. The main assumptions
that were made:
1. Our team is responsible for storing route binaries.
2. Our production usage is roughly 10 000 requests per month, based on the usage in April.

Where possible, we used production data to create our estimations, otherwise we
would model the number of requests that we would use.

We see that in our current production usage, the main bottleneck for pricing is the compute
environment. All other costs remain $\le 10$, so the choice for compute environment will be the
main factor in the cost.

Once we increase the costs to 10x or 100x our current load, the binary storage
become increasingly expensive. Depending on how long we want to save these, it
might result in being roughly 70% of the total cost of our team.

In all scenarios, both the trigger and database costs are negligible, as even when the production
load gets 100 times higher, the costs still remain $\le 10$ euros per month.

Some other findings:
1. We should use "hot" storage if we want to store route binaries <1 month
2. We should use "cool" storage if we want to store route binaries between 1 month and 4.5 months
3. We should use "cold" storage if we want to store route binaries >4.5 months

For a total cost estimate, let's go for the following scenario:
1. Azure Functions -> Event Grid -> Azure Functions architecture
2. Flex Consumption plan with 1x always-ready instance (4GB)
3. Keep binaries for 1 year, following cold storage plan, saving `run_info.json`.
4. Use auto-scaling database.

Then we'll have the following production cost estimates:
- 1x production usage: €50.75     (Compute: 78.4%, storage: 5.6%)
- 10x production usage: €83.34    (Compute: 55.8%, storage: 34.4%)
- 100x production usage: €431.52  (Compute: 31.1%, storage: 66.4%)


#### Limitations

This calculation took only into account the following endpoints:
1. `/routes`
2. `/routes/:id/status`
3. `/routes/:id/result`

It did not take into account other calls, like getting the boundary corners or
uploading all calls to blob storage. This might increase the cost a little bit,
but we believe the majority of the costs will be covered by just focussing on
the motion planning part of the API.

This model also did not take into account any full-downloads of the dataset that
happen when we want to analyze the data. If we want to occasionally download all
route binaries, this might increase the cost of our storage significantly and a
re-analysis of storage type might be recommended.

## 5. Performance Evaluation

### Introduction

This section compares the performance of different Azure components for our routing service
infrastructure. The goal is to determine the most efficient and scalable architecture for handling
requests of varying complexity. We evaluate Azure Functions, Azure Web Apps, database connectivity
(CosmosDB), and various async job triggers (Queue Storage, Event Grid, and Durable Functions) to
optimize our service architecture.

### Method

The tests utilize three test cases:

- **Short-running task** (~200ms best case processing time): Representing simple routing requests
  - `TestData/task_200ms.json`
- **Moderate task** (~2s best case processing time): Representing a "normal" routing request
  - `TestData/task_2s.json`
- **Longer-running task** (~6s best case processing time): Representing more complex routing scenarios
  - `TestData/task_6s.json`

Testing is performed using [scripted HTTP requests (in Julia)](https://bitbucket.org/agxeed/agx_routing_snippets/src/add-api-testing-scripts/ApiTesting.jl/).

The primary endpoint tested in all cases is a routing service endpoint similar to the production
service. It's based on the routing method used in the Diagnostic Tests in AgxRoute.Tests. The
service takes tasks in JSON format and returns the task id and route recipe in protobuf encoding.

When running the tests, we've noticed that the running times change depending on the test.
An example test run could look like: 1100 ms, 800 ms, 520 ms, 480 ms, 500 ms, 510 ms.
This occurs for two reasons:
- The JIT compiler optimizes the code after a few runs, and the code gets faster
- The serverless infrastructure (Azure Functions) is not always ready, and the first run
  takes longer than the subsequent runs. This is also known as the *cold-start problem*

We analyze this in more detail in the [cold-start analysis](#cold-start-analysis) section.
For the performance tests, the first run is not representative of the
performance of the system. Therefore, we run the tests in two different ways:
- *Warm tests:*  Tests are run after a few warmup runs, and the average of the last 10 runs is taken
         with 2 seconds between requests.
- *Cold start tests:* Requests are run with a service restart and 2 minute delay between requests,
            and the average of 10 runs is taken.

Most tests are run as warm tests unless stated otherwise. Finally, tests with `-` indicate that the
test was not performed, because we felt that it did not add any additional value to the test.

We will divide our tests into different test cases to limit the scope of the tests.

**Test 1:** Azure Functions vs WebApp
- Research question: How does Azure Functions compare to WebApp in terms of raw performance?
- Goal: Set a baseline for other tests and find the best Request-Response architecture.

**Test 2:** Overhead CosmosDB and Blob storage
- Research question: How does adding CosmosDB and Blob storage affect the performance?
- Goal: Understand the overhead of adding storage to the architecture.

**Test 3:** Best asynchronous job architecture.
- Research question 1: What's the best Asynchronous architecture?
- Research question 2: How does it compare to the request-response architecture?
- Goal: Find the best asynchronous job architecture.

**Test 4:** Proposed architecture vs existing architecture
- Research question: How does the existing architecture perform vs the new architecture?
- Goal: Show the performance improvements of the new architecture.

**Test 5:** Cold-start analysis
- Research question: How does the cold-start time affect the performance?
- Goal: Understand the consequences of cold-start.

**Test 6:** Scalability
- Research question: How well does the proposed architecture scale?
- Goal: Understand the future-proofing of the proposed architecture.

**Test 7:** Memory usage
- Research question: Should we use 2GB RAM or 4GB RAM instances?
- Goal: See if we can save money by using 2GB instances instead of 4GB instances.

**Note:** these tests have been performed under different circumstances with different
configurations. While similar, the results between tests are not directly comparable, please keep
this in mind when reading the results.

### Experiments

#### Test 1: Azure Functions vs Web App

*Research question:* How does Azure Functions compare to Azure Web Apps in terms of raw performance?

<details>
  <summary>Configurations explained</summary>

  All configuration details can be found [in Azure](https://portal.azure.com/#@agxeed.com/resource/subscriptions/26047d6f-4e24-41ec-ba57-cad424fa1e04/resourceGroups/dev-motionplan-resgrp-northeu/providers/Microsoft.Web/serverfarms/dev-motionplan-asp-northeu/pricingTier).

  The following configurations were used for the Web App tests:

  | Service Plan | ACU | vCPU | Memory  | Pricing (monthly) |
  | ------------ | --- | ---- | ------- | ----------------- |
  | Basic B1     | 100 | 1    | 1.75 GB | $13.14            |
  | Basic B2     | 100 | 2    | 3.5 GB  | $25.55            |
  | Basic B3     | 100 | 4    | 7 GB    | $51.50            |
  | Premium P0v3 | 195 | 1    | 4 GB    | $61.32            |
  | Premium P1v3 | 195 | 2    | 8 GB    | $122.64           |

  Where:
  - ACU = Azure Compute Unit, a measure of performance.
  - vCPU = virtual CPU, the number of virtual processors allocated to the service.

  For comparison, the Azure Functions Flex Consumption Plan has 210 ACU, 4GB of memory, and 1 vCPU,
  but can scale out to 500 instances if needed.

</details>

| Test Method             | Input Data        | Azure Functions | Web App B1 | Web App B2 | Web App B3 | Premium v3 P0V3 | Premium v3 P1V3 |
| ----------------------- | ----------------- | --------------- | ---------- | ---------- | ---------- | --------------- | --------------- |
| Sequential Request      | `task_200ms.json` | 0.47s           | -          | 1.336s     | 1.306s     | 0.816s          | 0.74s           |
| Sequential Request      | `task_2s.json`    | 9.95s           | 39.37s     | 16.009s    | 14.254s    | 10.233s         | 8.587s          |
| Sequential Request      | `task_6s.json`    | 11.8s           | 88.56s     | 34.295s    | 27.683s    | 19.909s         | 17.476s         |
| 3x Concurrent Requests  | `task_2s.json`    | 15.298s         | -          | 40.01s     | 24.69s     | -               | 23.78s          |
| 10x Concurrent Requests | `task_6s.json`    | 134.9s          | Timeout    | Timeout    | Timeout    | Timeout         | Timeout         |
| Cold Start              | `task_200ms.json` | 6.09s           | -          | -          | -          | -               | 5.1686s         |
| Cold Start              | `task_6s.json`    | 13.78s          | -          | -          | -          | -               | 22.564s         |

A timeout happens when the computation takes more than 230 seconds to complete.

Answer to the research question: *How does Azure Functions compare to Azure Web Apps in terms of
raw performance?*
- For simple tasks, the Azure Functions performs similar to the Premium P1v3 webapp.
- The more complex the task, the better the Azure Functions perform.
- Given concurrent requests, Azure Functions outperforms Web Apps significantly in all cases.

Finally, we've noticed that the cold start between Azure Functions and Web Apps is similar, however:
- For WebApps, there is an option to keep the application always-on, eliminating this problem.
- For Azure Functions, an always-ready instance reduces the cold-start, but does not eliminate it.

Therefore, Azure Functions has a downside compared to WebApps that when 20 minutes pass without
request, the next request can take ~5s longer. See the section about Azure Functions cold start
for more info.

#### Test 2: Overhead CosmosDB and Blob storage

*Research question:* How does adding CosmosDB and Blob storage affect the performance?

**Database (CosmosDB) Performance Overhead**

Azure Functions provides two methods for interacting with CosmosDB, an Input binding and an Output
binding. The input binding is used to read or write data to/from the database, and the output
binding is used only to write data to the database. We test both combinations and check for any
performance overhead.

Test scenario:
- Task: `task_2s.json`
- Test: Sequential tests, using Functions Request-Response architecture.
- Cold start: executing routing with 20 minutes between requests.
- Warm start: tests are run after a few warmup runs.

| Configuration   | Warm Start (Mean response time) | Cold Start (Mean response time) |
| --------------- | ------------------------------- | ------------------------------- |
| No Database     | 5.55 sec (SD: 0.25 sec)         | 10.45 sec (SD: 0.38 sec)        |
| Database Input  | 5.44 sec (SD: 0.26 sec)         | 10.86 sec (SD: 0.14 sec)        |
| Database Output | 5.44 sec (SD: 0.09 sec)         | 10.80 sec (SD: 0.19 sec)        |

Since the response time for both cold and warm starts are very close together, we can conclude that
adding CosmosDB connectivity to Azure Functions has minimal performance impact.

**Blob Storage Performance Overhead**

Test scenario:
- Task: `task_6s.json`
- Test: Warm tests, sequential tests
- Architecture: Request-Response architecture with CosmosDB.
- *Note:* The configuration with blob storage only returns the URL, whereas the blob storage
  configuration with database output returns the binary.

| Configuration   | Mean response time   |
| --------------- | -------------------- |
| No blob storage | 10.91 (SD: 0.25 sec) |
| Blob storage    | 10.01 (SD: 0.24 sec) |


More tests are needed if we want to take into account the difference of returning the binary, but
since the time increase is very small, we can assume that the blob storage does not add a
significant overhead to the response time.

Answer to the research question: *How does adding CosmosDB and Blob storage affect the performance?*
- Adding CosmosDB connectivity and Blob Storage to Azure Functions has minimal performance impact.

#### Test 3: Best Asynchronous Job architecture

*Research question 1:* What's the best Asynchronous architecture?

*Research question 2:* How does it compare to the request-response architecture?

Test scenario:
- Test: Mean of 10 sequential tests after warmup, 2s between requests.
- All configurations are using CosmosDB (but no blob storage), unless specified otherwise.
- All Functions have 2 always-ready instances available.

All these configuration names follow the same naming convention:
"Functions + Queue Storage + Functions" means Functions for the orchestrator, Queue Storage
for the async job trigger, and Functions for worker. See for more info.

**Task:** task_200ms.json

| Configuration                                                      | Mean response time (s) | Time increase |
| ------------------------------------------------------------------ | ---------------------- | ------------- |
| Functions (Request Response) (no CosmosDB)                         | 0.47                   | 0%            |
| Functions (Request Response)                                       | 0.48                   | 2%            |
| Functions   + Event Grid        + Functions                        | 0.69                   | 47%           |
| Functions   + Queue Storage     + Functions                        | 1.43                   | 204%          |
| Functions   + Durable Functions + Durable Functions                | 1.64                   | 249%          |
| Functions   + Durable Functions + Durable Functions (always-ready) | 1.71                   | 264%          |
| Functions   + CosmosDB trigger  + Functions                        | 4.90                   | 942%          |
| WebApp (B1) + Queue Storage     + Functions                        | 9.25                   | 1868%         |


**Task:** task_6s.json

| Configuration                                                      | Mean response time (s) | Time increase |
| ------------------------------------------------------------------ | ---------------------- | ------------- |
| Functions (Request Response) (no CosmosDB)                         | 10.32                  | 0%            |
| Functions (Request Response)                                       | 10.66                  | 3%            |
| Functions   + Event Grid        + Functions                        | 10.86                  | 5%            |
| Functions   + Durable Functions + Durable Functions (always-ready) | 12.70                  | 23%           |
| Functions   + CosmosDB trigger  + Functions                        | 15.61                  | 51%           |
| Functions   + Queue Storage     + Functions                        | 20.05                  | 94%           |
| WebApp (B1) + Queue Storage     + Functions                        | 20.55                  | 99%           |
| Functions   + Durable Functions + Durable Functions                | 26.92                  | 151%          |


All relative time increases have been rounded to the nearest integer.

Answer to research question 1: What's the best Asynchronous architecture?
- "Functions + Event Grid + Functions" provides the best result in the asynchronous job model, with
  the fastest time in both the 200ms and 6s tasks.

Answer to research question 2: How does it compare to the request-response architecture?
- There is a minor, sub-second overhead in using the Async Job Trigger architecture for small tasks.
- For larger tasks, there is basically no overhead.

Other interesting observations:
- Durable Functions also has reasonable performance.
- The bad performance for Durable Functions without always-ready instance was most likely
  caused by a random occurrence of the cold-start problem.

#### Test 4: Proposed architecture vs existing architecture

*Research question:* How does the existing architecture perform vs the new architecture?

For the current environment:
- Dev environment: used existing development environment: `agx-we-d-azfun-field-0-aav`
- Production environment: Duplicated the production environment with the same settings as the
  current production environment.

For the new environment:
- Proposed architecture: Functions + EventGrid + Functions
- Control architecture: Functions + QueueStorage + Functions

Since it was not easily possible to run the `task_2s.json` task or other tasks on our system,
we instead created two new tasks that are similar to the ones used in the other tests:
- `task_black_diamond_one.json`: a complex task that takes ~2s to run
- `task_black_diamond_two.json`: a simple task that takes ~200ms to run

Cold starts are tested by executing routing with 20 minutes between requests.
Sequential and Parallel tests are run with a warmup period, and are considered warm start tests.

**Complex Portal Test: `task_black_diamond_one.json`**

| Environment                          | Warm Start (average response time) | 3x Concurrent (total time) |
| ------------------------------------ | ---------------------------------- | -------------------------- |
| Legacy (Dev)                         | 5.00 s                             | 16.33 s                    |
| Legacy (Prod)                        | 8.24 s                             | 24.06 s                    |
| Functions + QueueStorage + Functions | 3.95 s                             | 17.99 s                    |
| Functions + EventGrid + Functions    | 2.90 s                             | 3.47 s                     |

**Simple Portal Test: `task_black_diamond_two.json`**

| Environment                          | Warm Start (average response time) | Cold Start (average response time) | 3x Concurrent (total time) | 10x Concurrent (total time) |
| ------------------------------------ | ---------------------------------- | ---------------------------------- | -------------------------- | --------------------------- |
| Legacy (Dev)                         | 0.76 s                             | -                                  | 3.43 s                     | 4.85 s                      |
| Legacy (Prod)                        | 1.13 s                             | 1.05 s                             | 5.27 s                     | 8.90 s                      |
| Functions + QueueStorage + Functions | 0.90 s                             | 11.05 s                            | 2.45 s                     | 2.40 s                      |
| Functions + EventGrid + Functions    | 0.79 s                             | 1.43 s                             | 1.50 s                     | 3.23 s                      |

Answer to the research question: *How does the existing architecture perform vs the new
architecture?*
- Functions + EventGrid + Functions performs 2x–3x as fast on complex tasks compared to the existing
  architecture, while performing similarly on simple tasks.
- Functions + EventGrid + Functions performs 4x-7x as fast on complex concurrent tasks compared to
  the existing architecture, while performing 1.5x-3x as fast on simple concurrent tasks.

Other interesting findings:
- Production environment is significantly slower than the development environment.
- The portal is significantly slower than if we directly call the API.
  - Perhaps portal only polls once every 10 seconds on develop, instead of once every second?
- The existing production architecture does not have the cold-start problem.

#### Test 5: Cold-start analysis

##### Background reading

Proper cold-start tests are incredibly time consuming. While we dabbled on some cold-start
tests here and there, we noticed that to do proper tests, we need weeks of time to get a good
estimate of the cold-start time.

We refer to [this blog post](https://mikhail.io/serverless/coldstarts/azure/)
([archived version](https://web.archive.org/web/20250105124137/https://mikhail.io/serverless/coldstarts/azure/)),
that contains the following interesting results:

<div class="figure">
  <img src="Images/coldstart.svg" alt="image" width="500">
  <div class="caption">
        Probability of cold-start occurring as a function of time since last request.
  </div>
</div>

Note that this blog-post comes from 2021, and [since 2024](https://techcommunity.microsoft.com/blog/appsonazureblog/our-latest-work-to-improve-azure-functions-cold-starts/4164500)
improvements have been made to significantly reduce the run-time (by ~53%).

Based on these posts, we can conclude:
1. Cold-start is random, but is most likely to occur between 20 and 30 minutes.
2. Linux C# cold-start is smaller than Windows cold-starts.
3. Zip size of our library might matter for cold-starts.

##### Experiments

*Research question:* How does the cold-start time affect the performance?

For these tests, we consider the worst-case scenario: restart the server after every test to
ensure that we start from a complete cold-start. This not only includes the Azure Functions
spinning up, but also the JIT compilation of the code.

While we mainly did the tests for the chosen architecture, we also did some tests for the
other architectures to see how they compare.

Scenario:
- Test: Mean of 10 tests after restart.
- All configurations are using CosmosDB and blob storage, except for the request-response
  architecture.

**Task: `task_200ms.json`**

| Configuration                   | Cold Start (mean response time) | Warm Start (mean response time) | Cold Start Impact |
| ------------------------------- | ------------------------------- | ------------------------------- | ----------------- |
| Function + EventGrid + Function | 3.89 s (SD: 0.21)               | 0.71 s                          | 3.18 s            |
| Function Request-Response       | 6.09 s (SD: 0.81)               | 0.47 s                          | 5.53 s            |

**Task: `task_6s.json`**`

| Trigger Type                       | Cold Start (mean response time) | Warm Start (mean response time) | Cold Start Impact |
| ---------------------------------- | ------------------------------- | ------------------------------- | ----------------- |
| Function + EventGrid + Function    | 19.63 s (SD: 0.50)              | 11.66 s                         | 7.97 s            |
| Function Request-Response          | 13.78 s (SD: 5.19)              | 11.80 s                         | 1.98 s            |
| Function + QueueStorage + Function | 21.66 s (SD: 2.81)              | 17.95 s                         | 3.71 s            |
| Function + Durable Functions x2    | 16.97 s (SD: 0.71)              | 12.70 s                         | 4.27 s            |

Answer to the research question: *How does the cold-start time affect the performance?*
- For the proposed architecture, in the worst case, the cold-start time adds around 3s of response
  time or simple tasks, and around 8s for complex tasks.

<details>
  <summary>Informal testing always-ready instances</summary>

  We did some informal testing on the cold-start time of the proposed architecture with always-ready
  instances enabled for the `task_6s.json`, which resulted in:
  - always-ready cold-start time: ~13.78 s

  Considering that a warm start takes ~11.66 s, while a fully cold start takes 19.63 s, it is a
  considerable improvement in the reduction of the cold-start time.

  However, as this is an optimization, we will not go more into detail in this evaluation.

</details>

#### Test 6: Scalability

*Research question:* How well does the proposed architecture scale?

**Task:** `task_200ms.json`

| Concurrent Requests | Event Grid             | Durable Functions      | Queue Storage          |
| ------------------- | ---------------------- | ---------------------- | ---------------------- |
| 1x                  | 0.77s (0.77 s/request) | 1.87s (1.87 s/request) | 2.56s (2.56 s/request) |
| 3x                  | 1.29s (0.43 s/request) | 2.64s (0.88 s/request) | 3.30s (1.10 s/request) |
| 10x                 | 4.3s  (0.43 s/request) | 7.8s (0.78 s/request)  | 9s    (0.90 s/request) |

**Task:** `task_6s.json`

| Concurrent Requests | Event Grid               | Durable Functions        | Queue Storage            |
| ------------------- | ------------------------ | ------------------------ | ------------------------ |
| 1x                  | 10.96s (10.96 s/request) | 15.23s (15.23 s/request) | 24.08s (24.08 s/request) |
| 3x                  | 32.97s (10.99 s/request) | 52.32s (17.44 s/request) | 87s    (29 s/request)    |
| 10x                 | 65.2s  (6.52 s/request)  | 182.6s (18.26 s/request) | 155.8s (15.58 s/request) |

Answer to the research question: *How well does the proposed architecture scale?*
- The proposed architecture scales relatively well, especially compared to the other architectures.
- A user still has to wait longer when there are more concurrent requests, so while this setup
  is good for scaling, if our usage increases significantly, we might need to consider more
  always-ready instances during periods of congestion.

#### Test 7: Memory usage

*Research question:* Should we use 2GB or 4GB instances?
- Task: `task_6s.json`
- Test: Mean of 10 sequential tests after warmup, 2s between requests.
- All configurations are using CosmosDB and blob storage.


| Memory Configuration | Event Grid (seconds) | Durable Functions (seconds) | Queue Storage (seconds) |
| -------------------- | -------------------- | --------------------------- | ----------------------- |
| 2048 MB              | 10.095 (SD: 2.4437)  | 10.885 (SD: 0.5201)         | 20.042 (SD: 3.1327)     |
| 4096 MB              | 5.176 (SD: 0.187)    | 6.67 (SD: 0.3022)           | 15.264 (SD: 3.3373)     |

Answer to the research question: *Should we use 2GB or 4GB instances?*
- Using 4GB instances is significantly faster than using 2GB instances.

## 6. Conclusion

We propose the following architecture for the routing service:
- Orchestrator: Azure Functions
- Async job trigger: Event Grid
- Worker: Azure Functions

The results in [test 3](#test-3-best-asynchronous-job-architecture) show that this architecture is
the best performing asynchronous job model, and the results in [test 4](#test-4-proposed-architecture-vs-existing-architecture)
shows that this architecture is faster than the existing architecture, and scales significantly
better than the existing architecture.

Specifically, the proposed architecture performed 2x-3x faster than the existing
architecture for complex tasks, and 4x-7x faster for complex concurrent tasks. For simple tasks,
the performance was comparable, but did scale better when multiple requests were made.

The [cost evaluation](#cost-summary) shows that the proposed architecture is significantly cheaper
than the current setup based on current production data. Even in a 100x usage scenario, the
proposed architecture remains affordable. Interestingly, in such a scenario, storage costs would
exceed computation costs — unlike the current situation, where computation costs dominate.

One downside of the proposed architecture is that it has a cold-start time. As shown in
[test 5](#test-5-cold-start-analysis), in the worst-case scenario, we can expect a cold-start
every ~25 minutes, which adds ~3s of response time for simple tasks, and ~8s for complex tasks.
Informal testing shows that the cold-start time can be reduced significantly with always-ready
instances, but more testing is needed to confirm this.

We believe that this is an acceptable trade-off, since — considering that this is the worst-case
scenario — for small tasks the occasional waiting of 3s is acceptable, and for complex tasks, the
user has to wait a lot longer, making this increase less noticeable.

Finally, [test 6](#test-6-scalability) showed that the proposed architecture scales well, but does
not scale perfectly. We recommend to monitor the usage of the system, and if we see that the
system is used significantly more, we should consider adding more always-ready instances to
the system during congestion.

In future work, we would like to test the performance of different always-on configurations, so
we can optimize the performance and deployment of our system. This would also allow us to give
more accurate estimates of the cold-start time with always-ready instances.
