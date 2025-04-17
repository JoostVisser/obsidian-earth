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

Where $43.07\text{ billion MB-ms} / 1\ 024\ 000 = 42 060\text{ GB-s}$

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
    - #executions [on-demand]: $359 170 / 2 - 250 000 = 109 170$ executions
    - **Total:** $109 170 * 0.371 / 1 000 000$ = €0.04

    Calculation details (10x production load):
    - GB-s [on-demand]: $420 600 - 100 000 = 320 600$ GB-s
    - #executions [on-demand]: $3 591 700 - 250 000 = 3 341 700$ executions
    - **Total:** $320 600 * 0.000025 + 3 341 700 * 0.371 / 1 000 000$ = €9.25

    Calculation details (100x production load):
    - GB-s [on-demand]: $4 206 000 - 100 000 = 4 106 000$ GB-s
    - #executions [on-demand]: $35 917 000 - 250 000 = 33 667 000$ executions
    - **Total:** $4 106 000 * 0.000025 + 33 667 000 * 0.371 / 1 000 000$ = €115.14

    Case: 1x always ready instance (4GB)

    Calculation details (1x production load):
    - GB-s [on-demand]: $42 060 / 2 - 100 000 = 0$ GB-s [free-grant our execution time]
    - #executions [on-demand]: $359 170 - 250 000 = 0$ executions (on-demand) [free-grant exceeds our #executions]
    - GB-s [always-active]: $42 060 / 2 = 21 030$ GB-s (always-active)
    - #executions [always-active]: $359 170 / 2 = 179 585$ executions (on-demand)
    - **Total (without baseline):** $21 030 * 0.0000149 + 179 585 * 0.370439 / 1 000 000 = \text{€}0.38$
    - **Total:** $39.40 + 0.04 = \text{€}39.78$

    Calculation details (10x production load):
    - GB-s on-demand: $420 600 / 2 - 100 000 = 110 300$ GB-s (on-demand)
    - #executions on-demand: $3 591 700 / 2 - 250 000 = 1 545 850$ executions (on-demand)
    - GB-s always-active: $420 600 / 2 = 210 300$ GB-s (always-active)
    - #executions always-active: $3 591 700 / 2 = 1 795 850$ executions (on-demand)
    - **Total (without baseline):** $110 300 * 0.000025 + 210 300 * 0.0000149 + (1 795 850 * 0.371 + 1 545 850 * 0.370439) / 1 000 000 = \text{€}7.13$
    - **Total:** $39.40 + 7.13 = \text{€}46.53$

    Calculation details (100x production load):
    - GB-s on-demand: $4 206 000 / 2 - 100 000 = 2 003 000$ GB-s (on-demand)
    - #executions on-demand: $35 917 000 / 2 - 250 000 = 17 708 500$ executions (on-demand)
    - GB-s always-active: $4 206 000 / 2 = 2 103 000$ GB-s (always-active)
    - #executions always-active: $35 917 000 / 2 = 17 958 500$ executions (on-demand)
    - **Total (without baseline):** $2 003 000 * 0.000025 + 2 103 000 * 0.0000149 + (17 708 500 * 0.371 + 17 958 500 * 0.370439) / 1 000 000 = \text{€}94.63$
    - **Total:** $39.40 + 94.63 = \text{€}134.03$

    (Although in the final scenario, perhaps we need more than 1x always ready instance.)

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
| CosmosDB Serverless                 | €0.08                      | €0.76                       | €7.63                       |
| CosmosDB autoscaling (min 100 RU/s) | €8.11                      | €8.11                       | ~€10                        |
| CosmosDB standard (min 400 RU/s)    | €21.63                     | €21.63                      | €21.63                      |
| PostgreSQL (burstable)              | €12.17                     | €12.17                      | €12.17                      |
| PostgreSQL (provisioned, basic)     | €133.59                    | €133.59                     | €133.59                     |

Assumptions made:
- Data will only be kept 1 month in the database.
- For 100x production load, we might get some 200 RU/s peaks, but since this is never consistent
  but only occasionally, the autoscaling will be triggered only occasionally.
- The basic version of PostgreSQL (burstable) might not be sufficient for 100x production load.

#### Conclusion

1. CosmosDB is affordable and future proof, so no need to check other databases.
2. Future test: Consider doing provisioned autoscaling instead of serverless for a
   potential speed improvement.
3. Even when we consider 100x production load, the database does not seem to significantly impact
   the final costs.


#### Monthly usage estimate

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

The run_info.json a file is negligible in terms of size (4 KB at most), so we'll just assume that the number reads and writes are doubled (but no significant increase in the reads per GB or storage).

| Load            | Storage (GB) | #writes   | #reads    | Reads (per GB) |
| --------------- | ------------ | --------- | --------- | -------------- |
| 1x prod. load   | 32.5         | 13 000    | 16 250    | 39             |
| 10x prod. load  | 325          | 130 000   | 162 500   | 390            |
| 100x prod. load | 3250         | 1 300 000 | 1 625 000 | 3 900          |

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

##### Scenario 1: Saving only the protobuf

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

##### Scenario 2: Saving the protobuf + run_info.json

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


#### Conclusion

1. Cold storage is the better option for us if we intend to keep the artifacts for longer than a
   month. Rough estimate:
    - Hot storage is the cheapest if we keep artifacts <=1 month
    - Cool storage is the cheapest if we keep artifacts between 1 month and 4.5 months
    - Cold storage is the cheapest for artifacts >4.5 months
2. For our current production usage, costs are negligible, even if we store the artifacts for three years.
3. For the 10x production scenario, storage has an impact to the total cost.
4. For the 100x production scenario, if we want to keep the artifacts for multiple years,
   storage is going to be our main contributor to the total cost.
5. We could optimize the scenario even more if we expect more reads, by mixing Hot/Cool/Cold storage
   together. (Where artifacts >3 months are placed into cold storage.)
6. Saving an additional `run_info.json` does not impact the pricing in a meaningful way.
7. Future test: is there any performance difference between Hot/Cool/Cold storage?

### Event Processing Costs

#### Monthly usage estimates

For our current production usage, we'll use an estimate of 10 000 runs a month, see the details of
the [database usage estimates](#database-monthly-usage-estimate) for more detail.

For events we need to store the whole field as a request. An average field in
the Diagnostic folders is roughly 31.2KB, so we'll take 32KB as an estimate per request.

Every run will have two requests (one event push, one event read), so we get a total of:
- 20 000 operations a month.
- $20 000 * 32 \text{ KB}$ = 640 MB of data

| Load            | Operations | Data (GB) |
| --------------- | ---------- | --------- |
| 1x prod. load   | 20 000     | 0.64      |
| 10x prod. load  | 200 000    | 6.40      |
| 100x prod. load | 2 000 000  | 64        |


#### Pricing

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

#### Results

| Service               | Cost/month [1x prod. load] | Cost/month [10x prod. load] | Cost/month [100x prod. load] |
| --------------------- | -------------------------- | --------------------------- | ---------------------------- |
| Queue Storage         | €0.03                      | €0.34                       | €3.43                        |
| Event Grid            | €0.00                      | €0.06                       | €1.06                        |
| CosmosDB (serverless) | €0.15                      | €1.54                       | €15.37                       |

For the storage, it's assumed that we delete all requests after one month.

#### Conclusion

1. All triggers are very affordable and the cost shouldn't bottleneck the choice of triggers.

### Summary

In this cost analysis, we compared the various factors of . The main assumptions that were made:
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


### Limitations

This calculation took only into account the following endpoints:
1. `/route/create`
2. `/route/{id}/status`
3. `/route/{id}/get`

It did not take into account other calls, like getting the boundary corners or
uploading all calls to blob storage. This might increase the cost a little bit,
but we believe the majority of the costs will be covered by just focussing on
the motion planning part of the API.

This model also did not take into account any full-downloads of the dataset that
happen when we want to analyse the data. If we want to occasionally download all
route binaries, this might increase the cost of our storage significantly and a
re-analysis of storage type might be recommended.