# Canary Analysis in Spinnaker

## Canary Overview
At a high-level, Canary is a developement process in which a change is partially rolled out, then evaluated against the current deployment (baseline) to ensure that he new deployment is operating at least as well as the old. This evaluation is done using key metrics that are chosen when the canary is configured.

Canaries are usually run against deployments containing changes to code, but they can also be used for operational changes including changes to configuration.

**The canary process is not a substitte for other forms of testing.**

### Pre-requisites

1. Have metrics to evaluate
2. Your application might send performance metrics which are published and available by default. You can also install a monitoring agent to collect more comprehensive metrics, and you can instrument your code to generate further metrics for that agent. In any case, you need to have access to a set of metrics, using some telemetry provider, which [Kayenta](https://github.com/spinnaker/kayenta) can then use.

## How to make Canary work in Spinnaker - the high-level process
#### 1. In Spinnaker, create one or many canary configurations
The configuration provides the set of metrics for use with the canary stages that reference it, plus default scoring thresholds and weights - defaults that can be overwritten in a [canary stage]()

You can configue each metric flexibly, to define its scope and whether it fails when it deviates up or down. You can also group metrics logically. (Any that you leave ungrouped are evaluated, but they don't contribute to the success or failure of the canary run.)

You can think of this configuration as a templated set of queries against your metric store.

#### 2. In any deplyment pipeline that will use canary, add one or more [canary stages]().
The canary stage includes information that scopes the templated query (canary config) to a specified set of time boundaries.

## Best Practices for configuring canary

### Don't put too many metrics in one group
Especially for critical metrics, if you have any metrics in the group and one critical metric fails, but the rest pass, the group gets a passing score overall.

You can put a critical metric in a group of only one to ensure that if it fails, the whole group fails every time.

### Compare canary against baseline, not against production
You might be tempted to compare the canary deployment against your current production deployment. Instead always compare the canary against an equivalent baseline, deployed at the same time.

The baseline uses the same version and configuration that is currently running in production, but is otherwise identical to the canary:

* Same time of deployment
* Same size of deployment
* Same type and amount of traffic

This way, you control for version and configuration only, and you reduce factors that could affect the analysis, like the cache warmup time, the heap size, and so on.

### Run the canary for enough time
You need at leaast 50 pieces of time series data per metric for the statisitical analysis to produce accurate results. That is 50 data points per canary run, with potentially several runs per canary analysis. In the end, you should plan for canary analyses several hours long.

You will need to tune the time parameters to your particular application. A good starting point is to have a canary lifetime of 3 hours, an interval of 1 hour and no warm-up period (unless you already know your application needs one). This gives you 3 caanry runs, each 1 hour long.

### Carefully choose your thresholds
You need to configure two thresholds for a canary analysis,

* Marginal - If a canary run has a score below this threshold, then the whole canary fails
* Pass - The last canary run of the analysis must score higher than this threshold for the whole scenario to be successful. Otherwise, it fails

These thresholds are very important for the analysis to give an accurate result, you will need to experiment with them, in the context of your own application, its traffic, and its metrics.

Keep in mind, that your ocnfiguration will be refined over time. Don't think of it as something you set once and never think about again.

Good starting points:
* Marginal Threshold of 75
* Pass Threshold of 95

### Carefully choose the metrics to analyse
You can get started with a single metric, but in the long run, your canaries will use several.

Use a variety of metrics that reflect different aspects of the health of your application. Use these three out of the four "Golden Signals" of SRE.

* latency
* errors
* saturation

If you consider some other specific metrics critical, place them in their own group in the canary configuration. This allows you to fail the whole caanry analysis if there is a problem with one of those specific metrics. You can also the criticality flag on the individual metric.

### Create a set of standad, re-usable canary configs
Configuring a canary is difficult, and not every developer in your organization will be able to do so. Also, if you let all teams within an org manage their own configs, you will likley end up with too many configs, too many metrics, nobody will know what is happening, and people will be afarid to change anything.

For these reasons, it is a good idea to curate a set of configs that all the teams can re-use.

### Use retrospective analysis to make debugging faster
It takes a long time to configure a canary analysis. It can take a long timr to debug it too, partyly because with a [long-running canary analysis]() you have to wait a long time for the analysis to finish before you can refine it.

Fortunately, a Canary Analysis stage can be configured to use a [retorspective analysis]() instead of a real-time analysis. This analysis is based on past monitoring data, without having to wait for the data points to be generated. With this mode, you can iterate more quickly on the development of the canary configuration

!!! tip

    Use the copy-to-clipboard icons in the canary stage execution details view to capture the needed timestamps

### Compare equivalent deployments
To compare the metrics between baseline and canary, Kayenta needs the exact same metrics for both. This means that metrics should have the same labels. Problems can arise if the metrics are labeled with their respective instance names.

If you see that Kayenta is not actually comparing the metrics, confirm that the queries that it executes against your monitoring system return metrics with the same labels.

### Some config values to start with
Although these values are not necessarily "best-practices", they are resoanbale starting points for your canary configs:

| Setting          | Value      |
| ---------------- | ---------- |
| canary lifetime  | 3 hours    |
| successful score | 95         |
| unhealthy score  | 75         |
| warmup period    | 0 minutes  |
| frequency        | 60 minutes |
| use lookback     | no         |
 






