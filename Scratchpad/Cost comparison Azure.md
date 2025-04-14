# Cost Comparison

## Introduction

This document provides a cost analysis of the different Azure components
evaluated for our routing service infrastructure. The goal is to understand the
financial implications of various architectural choices, identify the most
cost-effective solutions, and project future costs as our service scales. By
examining pricing tiers, scaling behaviors, and resource consumption patterns,
we can make informed decisions that balance performance needs with budget
constraints.

## Method

(Improve methods, use architecture as reference.)

We evaluated multiple configurations including:
- Azure Functions (Consumption Plan vs. Premium Plan)
- Azure Web Apps (Basic, Standard, and Premium tiers)
- Azure CosmosDB
- Storage solutions (Queue Storage, Event Grid)
- Hybrid architectures combining multiple services

All cost figures are based on current Azure pricing (as of evaluation date) and reflect North Europe region pricing unless otherwise specified.

## Results

### Azure Functions vs. Azure Web Apps
Azure uses **ACU**, "Active Compute Units", as a measure for "performance". All Azure Functions run on 210 ACU, for the web apps it depends per plan.
#### Azure Web Apps
For a look at all the different plans, check the [WebApp pricing page](https://azure.microsoft.com/en-us/pricing/details/app-service/linux/)

Here's a highlight of the plans that are important to us:

| Plan            | Price per month | RAM   | CPUs | Storage | Performance |
| --------------- | --------------- | ----- | ---- | ------- | ----------- |
| Basic B2        | $25             | 3.5GB | 2    | 30GB    | 100 ACU     |
| Premium v3 P0V3 | $61             | 4GB   | 1    | 250GB   | 195 ACU     |
| Premium v3 P1V3 | $123            | 8GB   | 2    | 250GB   | 195 ACU     |

It's possible to run multiple app services on the same "App Service Plan", then multiple web-apps share the same computer.
#### Functions
##### Pricing list
###### Flex-consumption pricing
For [Functions](https://azure.microsoft.com/en-us/pricing/details/functions/),
it's a bit more difficult to calculate, as it uses two metrics:
- €0.000025 GB-s (seconds compute time for every GB of memory used)
- €0.371 per million function executions

The flex-consumption pricing consists of two parts:
1. **On-demand** — A computer that spins up once the event is (has cold-start problem)
2. **Always-ready instance** -- An active computer that continuously runs on the background, but results in a fixed monthly costs.

| Meter                         | Free Grant (Per Month) | Pay as you go                    |
| ----------------------------- | ---------------------- | -------------------------------- |
| On Demand Execution Time      | 100,000 GB-s           | €0.000025/GB-s                   |
| On Demand Total Executions    | 250,000 executions     | €0.371 per million executions    |
| Always Ready Baseline         |                        | €0.0000038/GB-s                  |
| Always Ready Execution Time   |                        | €0.0000149/GB-s                  |
| Always Ready Total Executions |                        | €0.370439 per million executions |
Per always ready instance, this results in a baseline price of:
- For 2GB instances: $\text{€}0.0000038 * (60*60*24*30) \text{ sec/month} * 2 \text{ GB} = \text{€} 19.70$ a month
- For 4GB instances: $\text{€}0.0000038 * (60*60*24*30) \text{ sec/month} * 4 \text{ GB} = \text{€} 39.40$ a month
###### Elastic Premium pricing

| Meter             | vCPU | RAM    | Price / hour | Price per month |
| ----------------- | ---- | ------ | ------------ | --------------- |
| Elastic Premium 1 | 1    | 3.5 GB | 0.211        | €135.86         |
| Elastic Premium 2 | 2    | 7 GB   | 0.423        | €271.72         |
|                   |      |        |              |                 |
Depending on how high we set the scale-out, the costs could be even higher. A higher scale-out does mean a better throughput when multiple tasks are planned, but any computing minutes need to be paid for. 
Since we need a minimum of 1 active instance, the price per month is the lower bound of the costs.
##### Comparison

For this comparison, we're using the [data used in our production server of the last 30 days](https://portal.azure.com/#@agxeed.com/blade/Microsoft_Azure_MonitoringMetrics/Metrics.ReactView/Referer/MetricsExplorer/ResourceId/%2Fsubscriptions%2Fd71a458e-0b74-41b5-8179-d0fe92d2c546%2FresourceGroups%2Fagx-we-p-rg-aav%2Fproviders%2FMicrosoft.Web%2Fsites%2Fagx-we-p-azfun-field-1-aav/TimeContext/%7B%22relative%22%3A%7B%22duration%22%3A2592000000%7D%2C%22showUTCTime%22%3Afalse%2C%22grain%22%3A1%7D/ChartDefinition/{"isCompressed":true,"value":"eJzVk0lPwzAQhf9K5HNMSNTS0iurBOXCdkAcJvYktZTakZd0U%2F87dkIQBQoSByRyssbv%2BZt5o2xIk7EZaGvI5GlD5mi1YN1Zo1FOM5yiBQ4WyGRDBCcTkhiXG6ZFbYWSJuGjFAbDMdLDfDSggzQf0nE6Oqb8sMDjjGdsODhK%2BscutHK1SaBc0gXSmuqSAjRJrVUjOGqTTAXTyqjCHjxinhhh8Z0a1oWTtBBYcZoGI9nGRMIcfVfnTrLQ0NkSmQuHE%2BWkJTGBstRYQijdrWovTTuPqYEF4%2FyNuOiJ3tUl8SCMg0qsW3cIgAtTV7C62WVGb9Cop%2FYDn%2B4YvptkG%2F%2F%2FzO%2BlsObPM%2B%2Bpv8j8OSZW2Crobt082rfQCCSP9ghaelQoHX1DesVcCcnbMJqPQ7Y%2FYZfVMCYVlij5pySE8SWRh3atdhiTWvns2sssJjO%2FzUvVoD4B7SkFVAa76jXkWIU4TGf0G4Rl%2B9ju%2B8svIUHadZZ54%2BoHTbr13%2FP2BTlPiW0%3D"}) (snapshot on April 11, 2025):
- 43.07 billion execution units
- 359 170 executions

Execution Units are given in MB-milliseconds. To convert to GB-seconds, divide by 1 024 000. Therefore, our production last month is:
- 42 060 GB-s
- 359 170 executions

| Service                                                            | Cost per month for 1x Production load | Cost per month at 10x production load |
| ------------------------------------------------------------------ | ------------------------------------- | ------------------------------------- |
| Azure Functions (Flex Consumption)                                 | €0.04                                 | €9.25                                 |
| Azure Functions (Flex Consumption) + 1 always ready instance (4GB) | €39.78                                | €46.53                                |
| Azure Functions (Premium, current plan)                            | €271.72                               | ~€500 (assuming we scale-out a bit)   |

We took into account the free grant of 250 000 requests and 100,000 GB-s per month for Flex Consumption. For the always-ready instance, we assumed that half the computation happens on the always-ready instance (doesn't really matter much, to be honest).
### Database Costs

CosmosDB offers flexible pricing models that align well with our scaling needs:

| Configuration                      | Monthly Cost  | Notes                       |
| ---------------------------------- | ------------- | --------------------------- |
| Provisioned throughput (5500 RU/s) | $446          | For 500GB storage           |
| Serverless                         | Pay-as-you-go | Good for variable workloads |
| Auto-scale (avg 30% utilization)   | $240          | For same 5500 RU/s capacity |

CosmosDB's cost is comparable to traditional SQL options:

- CosmosDB (5500 RU/s, 500GB): $446
- Azure SQL (P1 125 DTUs, 500GB): $456

The serverless option provides excellent cost optimization for variable workloads, and auto-scaling can reduce costs by up to 46% with typical utilization patterns.

### Event Processing Costs

| Service           | Base Cost                | Cost per Million Operations  | Notes                                     |
| ----------------- | ------------------------ | ---------------------------- | ----------------------------------------- |
| Queue Storage     | $0.0175/GB               | $0.04 per million operations | Low cost, simple implementation           |
| Event Grid        | $0.60/million events     | $0.60 per million events     | Higher cost but lower processing overhead |
| Durable Functions | Execution cost + storage | Varies by execution duration | Higher processing overhead                |

Event Grid provides the best balance between cost and performance, with only a 5.2% performance impact on long-running tasks compared to direct processing, while Queue Storage adds significant overhead (94% increase) but at a lower cost per operation.

### Scaling Cost Implications

As workloads increase, the cost profiles change significantly:

- **Azure Functions Consumption Plan**: Scales linearly with request volume, but provides automatic scaling with no minimum instances
- **Azure Functions Premium Plan**: Higher base cost but can be more economical for high-throughput scenarios due to enhanced performance and pre-warmed instances
- **Azure Web Apps**: Requires manual scaling decisions, potentially leading to either over-provisioning (wasted resources) or under-provisioning (performance issues)

For high-volume scenarios (>10M requests/month), the Premium Plan for Azure Functions may become more cost-effective than the Consumption Plan due to execution efficiency.

## Conclusion

Based on our comprehensive analysis, we can draw several key conclusions about the cost implications of our architecture choices:

1. **Azure Functions provides the best performance-to-cost ratio** for our routing workloads, particularly for variable or unpredictable traffic patterns. The Consumption Plan offers excellent entry pricing with its free grant, while the Premium Plan provides cost predictability for higher volumes.

2. **CosmosDB with auto-scaling or serverless option** offers the most cost-effective database solution, with costs potentially reduced by up to 46% compared to fixed provisioning. The pay-as-you-go model aligns well with our variable workload patterns.

3. **Event Grid provides the optimal balance** between cost and performance for asynchronous processing, adding minimal overhead (5.2%) at a reasonable cost per million events.

4. **Cold start mitigation** through Premium Plan pre-warmed instances should be considered only when response time consistency becomes critical, as it significantly increases base costs.

5. **Scaling economics favor Azure Functions** over Web Apps as volume increases, due to its automatic scaling capabilities and consumption-based pricing model.

For our current projected workload, the most cost-effective architecture would be Azure Functions (Consumption Plan) with CosmosDB (auto-scale) and Event Grid for asynchronous processing. As our volume increases beyond 10M requests per month, we should re-evaluate the potential benefits of the Premium Plan for Azure Functions.
