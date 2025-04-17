# Cost Comparison

## Introduction

This document provides a cost analysis of the different Azure components evaluated for our routing
service infrastructure. The goal is to understand the financial implications of various
architectural choices, identify the most cost-effective solutions, and project future costs as our
service scales. By examining pricing tiers, scaling behaviors, and resource consumption patterns, we
can make informed decisions that balance performance needs with budget constraints.

## Method

(Improve methods, use architecture as reference.)

We evaluated multiple configurations including:

- Compute environments (Azure Functions, Azure Web Apps)
- Storage (Azure Blob Storage)
- Databases (Azure CosmosDB, PostgreSQL)
- Event triggers (Event Grid, Queue Storage, CosmosDB)

All cost figures are based on current Azure pricing (as of evaluation date) and reflect North Europe
region pricing unless otherwise specified.

## Results

### Compute environments

To limit the scope of this comparison, we'll only consider flex-consumption and elastic premium.

The regular consumption plan is slightly cheaper than the flex-consumption plan,
but we would like to have the option of always-ready instances (and the
performance of flex-consumption is better).

#### Monthly usage estimate

For WebApps, we pay for the server per month, so we don't need to consider any
usage estimate for the costs. (In practice, the server will respond slower with a higher load.)

For functions, we have two measures of computations:
- GB-s: amount of computation we perform per second per GB RAM.
- #executions: number of calls to the functions being made in total

For this comparison, we're using the [data used in our production server of the last 30 days](da.gd/azurestats)
(snapshot on April 11, 2025):
- 43.07 billion execution units (MB-ms)
- 359 170 executions

Where $43.07\text{ billion MB-ms} / 1\ 024\ 000 = 42 060\text{ GB-s}$.

Therefore:

| Load            | Computation (GB-s) | Function executions |
| --------------- | ------------------ | ------------------- |
| 1x prod. load   | 42 060             | 359 170             |
| 10x prod. load  | 420 600            | 3 591 700           |
| 100x prod. load | 4 206 000          | 35 917 000          |

#### Azure Web Apps

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

#### Functions Flex-consumption pricing

The [flex-consumption pricing](https://azure.microsoft.com/en-us/pricing/details/functions/) consists of two parts:
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
| Azure Functions (Flex Consumption)                                 | €0.04                      | €9.25                       |                              |
| Azure Functions (Flex Consumption) + 1 always ready instance (4GB) | €39.78                     | €46.53                      |                              |
| Azure Functions (Premium, current plan)                            | €271.72                    | €300 - €500                 | €500–€3000                   |
| WebApp Basic B1                                                    | €12.17                     | N/A                         | N/A                          |
| WebApp Basic B2                                                    | €23.66                     | N/A                         | N/A                          |
| WebApp Premium v3 P0V3 (less than our current production plan)     | €56.79                     | €56.79                      | €56.79 (too slow?)           |
| WebApp Premium v3 P1V3 (similar to current production plan)        | €113.58                    | €113.58                     | €113.58 (too slow?)          |

The premium plans are rough estimates, but it assumes we need to scale-out a
little bit due to the extra requests.

For the flex-consumption, we took into account the free grant of 250 000
requests and 100,000 GB-s per month. For the always-ready instance, we assumed
that half the computation happens on the always-ready instance (doesn't really
matter much, to be honest).

<details>
    <summary>Calculation details</summary>

    Free grant contains 100 000 GB-s and 250 000 executions.

    Calculation details (1x production load, no always ready instance):
    - GB-s [on-demand]: $42 060 - 100 000 = 0$ GB-s [free grant exceeds our execution time]
    - #executions [on-demand]: $359 170 / 2 - 250 000 = 109 170$ executions
    - **Total:** $109 170 * 0.371 / 1 000 000$ = €0.04

    Calculation details (10x production load, no always ready instance):
    - GB-s [on-demand]: $420 600 - 100 000 = 320 600$ GB-s
    - #executions [on-demand]: $3 591 700 - 250 000 = 3 341 700$ executions
    - **Total:** $320 600 * 0.000025 + 3 341 700 * 0.371 / 1 000 000$ = €9.25

    Calculation details (1x production load, 1x always ready instance 4GB):
    - GB-s [on-demand]: $42 060 / 2 - 100 000 = 0$ GB-s [free-grant our execution time]
    - #executions [on-demand]: $359 170 - 250 000 = 0$ executions (on-demand) [free-grant exceeds our #executions]
    - GB-s [always-active]: $42 060 / 2 = 21 030$ GB-s (always-active)
    - #executions [always-active]: $359 170 / 2 = 179 585$ executions (on-demand)
    - **Total (without baseline):** $21 030 * 0.0000149 + 179 585 * 0.370439 / 1 000 000 = \text{€}0.38$
    - **Total:** $39.40 + 0.04 = \text{€}39.78$

    Calculation details (10x production load, 1x always ready instance 4GB):
    - GB-s on-demand: $420 600 / 2 - 100 000 = 110 300$ GB-s (on-demand)
    - #executions on-demand: $3 591 700 / 2 - 250 000 = 1 545 850$ executions (on-demand)
    - GB-s always-active: $420 600 / 2 = 210 300$ GB-s (always-active)
    - #executions always-active: $3 591 700 / 2 = 1 795 850$ executions (on-demand)
    - **Total (without baseline):** $110 300 * 0.000025 + 210 300 * 0.0000149 + (1 795 850 * 0.371 + 1 545 850 * 0.370439) / 1 000 000 = \text{€}7.13$
    - **Total:** $39.40 + 7.13 = \text{€}46.53$

    All costs have been rounded to the nearest cent.

</details>

#### Conclusion

1. For development/testing:
  - Choosing Azure Functions (Flex Consumption) saves money over using webapps B1 and B2 and is more future-proof.
2. For production:
  - Choosing Azure Functions (Flex Consumption with an always active instance)
    saves a lot of money over the other production options.
  - The WebApp Premium v3 P1V3 might be a superior option over Elastic Premium 2 if scaling-out is not needed.
    (This does change the architecture of the source code, however.)
3. Future test: Consider testing 2GB flex consumption, this saves half the money of 4GB per always-ready instance.

### Database Costs

#### Database monthly usage estimate

Using our production data, we get a rough estimate for the usage of our database:
- $116 000$ Request Units (RUs) per month
- $4 MB$ of storage
- $1$ RU/s consistent throughput

| Load            | Requests Units (RU) | Storage (GB) | Consistent throughput (RU/s) |
| --------------- | ------------------- | ------------ | ---------------------------- |
| 1x prod. load   | 116 000             | 0.004        | 1                            |
| 10x prod. load  | 1 160 000           | 0.04         | 10                           |
| 100x prod. load | 11 600 000          | 0.4          | 100                          |


<details>
    <summary> Rough estimation of database costs</summary>

    For the Database costs, we'll make use of the production data of 2025 January, February and March.

    In those three months, we got 10955 runs. Per month this has an average of
    around 3650 runs a month (of which 66% succeeded). For ease of calculations,
    we'll use a higher estimate of 4000 runs per month as base value.

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
    - RUs: $4000 * 29 = 116000$ RUs
    - Storage: $4000 * 1\text{ KB} = 4\text{ MB}$

    Which would translate to roughly: $112 000 RUs / (30 * 8 * 3600\text{ s}) ≈ 0.134 RU/s
    (Assuming a uniform distribution for requests for 9AM - 5PM, 7 days a week, for 30 days.)

    Since requests will never be uniformly distributed over the day, let's just consider 1 RU/s as a
    baseline to catch a peak moment for requests.

</details>

#### Database pricing

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

#### Results

| Configuration                       | Cost/month [1x prod. load] | Cost/month [10x prod. load] | Cost/month [100x prod. load |
| ----------------------------------- | -------------------------- | --------------------------- | --------------------------- |
| CosmosDB Serverless                 | €0.03                      | €0.30                       | €3.05                       |
| CosmosDB autoscaling (min 100 RU/s) | €8.11                      | €8.11                       | €8.11                       |
| CosmosDB standard (min 400 RU/s)    | €21.63                     | €21.63                      | €21.63                      |
| PostgreSQL (burstable)              | €12.17                     | €12.17                      | €12.17                      |
| PostgreSQL (provisioned, basic)     | €133.59                    | €133.59                     | €133.59                     |

This assumes that runs will only be kept 1 month in the database. The resulting binary (plus perhaps a run-info) will be kept in blob storage for longer, if needed.
#### Conclusion

1. CosmosDB is affordable and future proof, so no need to check other databases.
2. Future test: Consider doing provisioned autoscaling instead of serverless for a
potential speed improvement.
3. Even when we consider 100x production load, the database does not seem to significantly impact the final costs.

### Storage Pricing

#### Monthly usage estimate

For our current production usage, we'll use the actual estimate of 3650 runs a month, see the details of
the [database usage estimates](#database-usage-estimate) for more detail.

Since two out of three runs succeeded, we'll use an estimate of 2500 runs that
require protobuf binaries to save. We're using an estimate of 5MB per protobuf
binary (I've seen 3.4MB and 7MB binaries), resulting in:
- $5 \text{ MB} * 2500 = 12.5 \text{ GB}$ of protobuf binaries every month

**Scenario 1: Saving only the protobuf**

For every binary, we'll assume 1 write and 1.2 read operations. Therefore:

| Load            | Storage (GB) | #writes | #reads  | Reads (per GB) |
| --------------- | ------------ | ------- | ------- | -------------- |
| 1x prod. load   | 15           | 2500    | 3000    | 18             |
| 10x prod. load  | 150          | 25 000  | 30 000  | 180            |
| 100x prod. load | 1500         | 250 000 | 300 000 | 1 800          |
**Scenario 2: Saving the protobuf + run_info.json**

The run_info.json a file is negligible in terms of size (4 KB at most), so we'll just assume that the number reads and writes are doubled (but no significant increase in the reads per GB or storage).

| Load            | Storage (GB) | #writes | #reads  | Reads (per GB) |
| --------------- | ------------ | ------- | ------- | -------------- |
| 1x prod. load   | 15           | 5000    | 6000    | 18             |
| 10x prod. load  | 150          | 50 000  | 60 000  | 180            |
| 100x prod. load | 1500         | 500 000 | 600 000 | 1 800          |
#### Pricing

For [Blob Storage](https://azure.microsoft.com/en-gb/pricing/details/storage/blobs/#pricing), there are multiple price tiers with different trade offs.

The relevant ones for us:

| Storage tier | Price per GB | Write (per 10 000) | Read (per 10 000) | Read (per GB) |
| ------------ | ------------ | ------------------ | ----------------- | ------------- |
| Hot          | €0.0204      | €0.0602            | €0.0047           | Free          |
| Cool¹        | €0.00927     | €0.1204            | €0.0121           | €0.0093       |
| Cold²        | €0.00334     | €0.2168            | €0.1204           | €0.0278       |

¹There exists a penalty for deleting files. If you delete it before 30 days, you will get charged for 30 days.
²Similar to cool, but then the penalty is for 90 days.
#### Results
##### Scenario 1

Monthly costs (deleting artifacts after 1 month):

| Configuration | Cost/month [1x production load] | Cost/month [10x production load] |
| ------------- | ------------------------------- | -------------------------------- |
| Hot storage   | €0.32                           | €3.22                            |
| Cold storage  | €0.34                           | €3.40                            |
|               |                                 |                                  |
|               |                                 |                                  |

Monthly costs (deleting artifacts after 1 year):

| Configuration | Cost/month [1x production load] | Cost/month [10x production load] |
| ------------- | ------------------------------- | -------------------------------- |
| Hot storage   | €3.69                           | €36.88                           |
| Cold storage  | €1.87                           | €18.70                           |

Deleting artifacts after 3 years:

| Configuration | Cost/month [1x production load] | Cost/month [10x production load] |
| ------------- | ------------------------------- | -------------------------------- |
| Hot storage   | €11.03                          | €110.32                          |
| Cold storage  | €5.21                           | €52.07                           |




#### Conclusion

1. Cold storage is the better option for us if we intend to keep the artifacts for longer than a month.
2. For our current production usage, costs are negligible, even if we store the artifacts for three years.
3. For the 10x production scenario, storage has an impact to the total cost.

### Event Processing Costs

#### Monthly usage estimates

For our current production usage, we'll use an estimate of 4000 runs a month, see the details of
the [database usage estimates](#database-usage-estimate) for more detail.

For events we need to store the whole field as a request. An average field in
the Diagnostic folders is roughly 31.2KB, so we'll take 32KB as an estimate per request.

Every run will have two requests (one event push, one event read), so we get a total of:
- 8 000 operations a month.
- $8 000 * 32 \text{ KB}$ = 256 MB of data

#### Pricing

[Queue storage](https://azure.microsoft.com/en-us/pricing/details/storage/queues/):
- €0.0417 per GB
- €0.38 per million operations

[Event Grid](https://azure.microsoft.com/en-us/pricing/details/event-grid/)
- €0.556 per million operations (per 64 KB, which one of our operation fits in)
- First 100 000 requests are free.

For Durable Functions, we assume the same pricing model holds as Queue storage, since it uses
a queue storage in the background.

#### Results

| Service               | Cost/month [1x prod. load] | Cost/month [10x prod. load] | Cost/month [100x prod. load] |
| --------------------- | -------------------------- | --------------------------- | ---------------------------- |
| Queue Storage         | €0.01                      | €0.13                       | €1.37                        |
| Event Grid            | €0.00                      | €0.00                       | €0.39                        |
| CosmosDB (serverless) | €0.08                      | €0.60                       | €6.15                        |

For the storage, it's assumed that we delete all requests after one month.
#### Conclusion

1. All triggers are very affordable and the cost shouldn't bottleneck the choice of triggers.

### Summary

In our scenario, the compute resources are the significant factor in the cost of the infrastructure.

In the event of an increasing production load (x10), the storage costs start to
play a more significant role in the overall cost of the infrastructure.
