# Prometheus Notes

The following is a summary of the notes and the links to external resources I have collected while working with Prometheus.

## Overview

Prometheus is an open-source systems monitoring and alerting toolkit originally built at SoundCloud. [...] It is now a standalone open source project and maintained independently of any company. To clarify the project's governance, Prometheus joined the [Cloud Native Computing Foundation](https://www.cncf.io/) in 2016 as the second hosted project, after Kubernetes, and graduated in 2018.

### Features

Prometheus's main features are:

- a multi-dimensional data model (time series identified by __metric name__ and __key/value pairs__)
- a flexible query language to leverage this dimensionality
- no reliance on distributed storage; single server nodes are autonomous
- time series collection happens via a pull model over HTTP
- pushing time series is supported via an intermediary gateway
- targets are discovered via service discovery or static configuration
- multiple modes of graphing and dashboarding support

Most Prometheus components are written in Go, making them easy to build and deploy as static binaries.

Ref: [Prometheus Overview](https://prometheus.io/docs/introduction/overview/)

## Installation

### Install Prometheus and enable self-monitoring

- Go to the [download page](https://prometheus.io/download/) and get the latest stable release of Prometheus for your architecture
- Untar the tarball
- cd into the directory

Setup Prometheus to monitor itself by putting the following in *prometheus.yml*:
~~~
global:
    scrape_interval: 10s
scrape_configs:
  - job_name: 'prometheus'

    # Override the global default and scrape targets from this job every 5 seconds.
    scrape_interval: 5s

    static_configs:
      - targets: ['localhost:9090']
~~~

**NOTE**: indentation is very important since a malformed YAML file will be not executed by Prometheus.

Run it:
~~~
./prometheus -config.file=prometheus.yml
~~~

Prometheus status page should now be accessible on http://localhost:9090/

After a couple of minutes, you can also verify that Prometheus is serving metrics about itself by navigating to its metrics endpoint: http://localhost:9090/metrics

It can also be queried directly from the command line using cURL:
~~~
curl http://localhost:9090/metrics
~~~

The answer looks somehow like the following:
~~~
[...]
# HELP http_requests_total Total number of HTTP requests made.
# TYPE http_requests_total counter
http_requests_total{code="200",handler="label_values",method="get"} 9
http_requests_total{code="200",handler="prometheus",method="get"} 11770
http_requests_total{code="200",handler="query",method="get"} 7
http_requests_total{code="200",handler="query_range",method="get"} 8237
http_requests_total{code="200",handler="series",method="get"} 60
http_requests_total{code="200",handler="static",method="get"} 4
http_requests_total{code="400",handler="query_range",method="get"} 3
http_requests_total{code="503",handler="query_range",method="get"} 131
[...]
~~~

### Debian Packages

**NOTE**: the RobustPerception APT repository (`deb.robustperception.io`) is deprecated; RobustPerception was acquired by Grafana Labs in 2020.

The official [installation documentation](https://prometheus.io/docs/prometheus/latest/installation/) now recommends pre-compiled binaries or [Docker images](https://hub.docker.com/r/prom/prometheus/). Debian packages may also be found in distribution repositories or community sources such as [prometheus-community](https://github.com/prometheus-community).

The following components can be installed the same way:

* Node Exporter 
* Pushgateway 
* Alertmanager

### Using the expression browser

Let us try looking at some data that Prometheus has collected about itself. To use Prometheus's built-in expression browser, navigate to ``http://localhost:9090/query`` and use the "Graph" tab.

Just check one of the metric exporter by Prometheus itself:
~~~
prometheus_target_interval_length_seconds
~~~

This should return a lot of different time series (along with the latest value recorded for each), all with the metric name prometheus_target_interval_length_seconds, but with different labels. These labels designate 
different latency percentiles and target group intervals.

Another example based on Prometheus functions: enter the following expression to graph the per-second rate of chunks being created in the self-scraped Prometheus.
~~~
rate(prometheus_tsdb_head_chunks_created_total[1m])
~~~

Ref: [Getting started with Prometheus](https://prometheus.io/docs/introduction/getting_started/)

Additional query examples in combination with Prometheus functions can be found [here](https://prometheus.io/docs/querying/examples/).

### Export additional host metrics through node_exporter

Monitoring only Prometheus itself is not very useful. The main monitoring agent utilized by Prometheus is called *node_exporter*: it exports hardware and OS metrics exposed by *NIX kernels, it's written in Go with pluggable metric collectors.

Ref: [Node Exporter (Github repo)](https://github.com/prometheus/node_exporter)

### Install and Enable the node_exporter

Add the following lines at the end of your *prometheus.yml* and reload the Prometheus configuration.

~~~
  - job_name: node

    static_configs:
      - targets: ['localhost:9100']
~~~

**NOTE**: keep the indentation, since the format is in **YAML**.

Reload the configuration:
~~~
curl -X POST http://localhost:9090/-/reload
~~~

Note: the ``--web.enable-lifecycle`` flag that was required as of Prometheus 2.0 has been removed again in 2.1; the lifecycle endpoints (``/-/reload``, ``/-/quit``) are enabled by default since then.

Node Exporter:

- Go to the [download page](https://prometheus.io/download/) and get the latest stable release of *node_exporter* for your architecture
- Untar the tarball
- cd into the directory

Run it: 
~~~
./node_exporter
~~~

The node exporter should now be listening on port **9100** and the metrics can be obtained from the command line with: ``curl http://localhost:9100/metrics``.

## Data Model

- All Prometheus data points are 64bit floating point
- Timestamps have millisecond resolution

Time Series structure:
~~~
metric_name, key/values,(...), samples
~~~

- metric_name: aspect of the system being measured
- key/values pairs form a *label* (a sub-dimension of a metric)
- samples: timestamp + a 64 bit float value

**NOTES**
- Labels can have any UTF8 value
- ``__name__`` is the reserved label holding the metric name; labels starting with ``__`` are reserved for metadata

### Gauges
- Snapshot of state
- Can go up and down
- You care about absolute value
- Export raw data

### Counters
- Count events, you care about rate of increase
- Start at 0, can only be incremented
- Resets are automatically handled by Prometheus counter functions
- End in *_total* by convention

### Summary & Histogram
- Compound types to make common case easier
- Can be used to track distributions
- Summary without quantiles is cheap, use it freely

### Native Histograms
- New metric type (GA in Prometheus 3.x) complementing classic histograms
- Exponentially spaced buckets: constant memory regardless of value range
- Fully aggregable (summable), unlike classic histogram buckets
- See: [Native Histograms specification](https://prometheus.io/docs/specs/native_histograms/)

Ref: [Prometheus Metric Types](https://prometheus.io/docs/concepts/metric_types/)

## Querying Prometheus

Prometheus comes with its own query language called *PromQL*.

PromQL is __strongly typed__:
- Instant Vector: a set of time series containing a single sample for each time series, all sharing the same timestamp
- Range Vector: a set of time series containing a range of data points over time for each time series
- Scalar: a simple numeric floating point value
- String: a simple string value

There are four parts to every metric:

1. Metric_name (e.g. ``varnish_main_client_req``).
2. One or more labels, which are simply key-value pairs that distinguish each metric with the same name (e.g. namespace="section-b4a199920b24b"). Each metric will have at least a job label, which corresponds to the scrape ``config`` in the Prometheus config.
3. The value, which is a float64. When querying in the Prometheus console, the value shown is the value for the most recent timestamp collected.
4. The timestamp, which has millisecond precision. The timestamp doesn't appear in the query console, but if you switch to the Graph tab within Prometheus, you can see the values for each timestamp in the chart.

### Matchers
- =, !=, =~, !~
- Can mix and match
- Regexes are anchored

### Functions
- Full list in official docs
- Use rate, increase, irate, and resets __only on counters__.
- Use rate/irate before histogram_quantile
- Absent is for alerting on time series not existing
- Don't use drop_common_labels or count_scalar

### Aggregators
- sum, count, min, max, avg, stddev and stdvar
- Use without to select output labels
- Can also use *by*, but it makes sharing expressions harder
- Always rate and then sum, never sum then rate

### Operators
- Arithmetic, Comparison and Logical
- Using ignoring to choose which labels not to bucket on. Can also use on.
- group_left/group_right for many-to-one matches
- Bool modifier for comparisons to return 0/1, otherwise filters
- Logical operators always many-to-many

### HTTP API
- Query range, each step is independently calculated

### Recording Rules
Recording rules allow you to precompute frequently needed or computationally expensive expressions and save their result as a new set of time series. Querying the precomputed result will then often be much faster than executing the original expression every time it is needed. This is especially useful for __dashboards__, which need to query the same expression repeatedly every time they refresh.

- Run regularly by Prometheus
- For precomputation, when you need a range vector, and for aggregates for federation
- See official docs for naming rules best practices. aggregation:metric:operations.

### Alerting Rules

[Alerting rules](https://prometheus.io/docs/alerting/rules/) allow you to define alert conditions based on Prometheus expression language expressions and to send notifications about firing alerts to an external service. Whenever the alert expression 
results in one or more vector elements at a given point in time, the alert counts as active for these elements' label sets.

### Query References

- [Query Basics](https://prometheus.io/docs/querying/basics/)
- [Querying Examples](https://prometheus.io/docs/querying/examples/)
- [Functions](https://prometheus.io/docs/querying/functions/)
- [Operators](https://prometheus.io/docs/querying/operators/)

## Practical Deployment

- 800K samples/s was the practical record for a single node in 2016; modern single nodes routinely ingest millions of samples/s.
- 2016-era planning figures: ~3-4kB of RAM per active time series, ~1.3 bytes/sample of disk with varbit encoding.
- SSD recommended.
- Ultra-granular monitoring isn't free, tradeoff against time and cost.
- 60s resolution is a good starting point.
- Run at least one Prometheus per datacenter.
- One Prometheus per team/service works well, and lets you scale.
- Global Prometheus servers can pull in aggregated stats via federation.
- For HA, run two identical Prometheus servers.
- Alertmanager has supported high-availability clustering (gossip-based) since v0.15: run several replicas that form a cluster. See the [Alertmanager HA documentation](https://prometheus.io/docs/alerting/latest/high_availability/).
- [Agent mode](https://prometheus.io/docs/prometheus/latest/prometheus_agent/) (stable since 3.x, `--agent` flag) is a scrape-only mode that just remote-writes: use it to scale collection away from the primary servers.
- Remote write/read integrations cover long-term storage and centralizing data from many nodes.
- Prometheus focuses on reliability over perfection.
- Use cross, meta and 3rd part monitoring to detect monitoring system failure.
- Prometheus storage is considered ephemeral. Long term storage will handle historical data.
- For migration start with exporters, low risk with immediate value.
- Use reverse proxy like nginx for auth. 
- Prometheus supports credentials when scraping.

## Pushgateway

The Pushgateway is an intermediary service which allows you to push metrics from jobs which cannot be scraped:
- [When to use the Pushgateway](https://prometheus.io/docs/practices/pushing/)
- [Pushing Metrics](https://prometheus.io/docs/instrumenting/pushing/)
- [Prometheus and anti-pattern pushgateway timeouts (Craftware)](https://www.craftware.info/2017/01/26/prometheus-and-anti-pattern-pushgateway-timeouts/)

## Prometheus Exporters

Exporters are independent programs, which can extract metrics from the monitored systems and convert them to a Prometheus-readable format.
The most famous of these is ``node_exporter``, which reads and provides operating system metrics such as memory usage and network load.
But there are other exporters for a wide range of protocols and services, such as Apache, MySQL, Postgresql, SNMP, etc.

- [Officially supported exporters](https://prometheus.io/download/)
- [Third party exporters](https://prometheus.io/docs/instrumenting/exporters/)

### Writing an exporter by yourself

- [Things to consider when writing an exporter or custom collector](https://prometheus.io/docs/instrumenting/writing_exporters/)
- [PromCom 2016: "So you want to write an exporter"](https://www.youtube.com/watch?v=KXq5ibSj2qA)
- [Tips to develop your own exporter (Brian Brazil)](https://www.slideshare.net/brianbrazil/so-you-want-to-write-an-exporter)
- [Write a Jenkins exporter in Python (Brian Brazil)](https://www.robustperception.io/writing-a-jenkins-exporter-in-python/)
- [A Noob's guide to custom Prometheus exporters ](https://rsmitty.github.io/Prometheus-Exporters-Revamp/)

## Data visualization

One of the best ways to visualize data saved in Prometheus is to use a dynamic dashboard like *Grafana*.

- [Grafana support for Prometheus](https://prometheus.io/docs/visualization/grafana/)
- [Prometheus data source in Grafana](https://grafana.com/docs/grafana/latest/datasources/prometheus/)
- [Grafana main web site](https://grafana.com)

## Useful Resources

- [A curated list of Prometheus resources (articles/videos/etc.)](https://github.com/roaldnefs/awesome-prometheus)
- [Prometheus Presentation at CERN (Fabian Reinartz/CoreOS)](https://cds.cern.ch/record/2253468?ln=en)
- [Five things you should be monitoring: Latency, Errors, Throughput, Utilization, Blackbox monitoring (B.Brazil)](https://www.oreilly.com/ideas/5-things-you-should-be-monitoring)
- [Sysadmin Chris Siebenmann on Prometheus at University of Toronto](https://utcc.utoronto.ca/~cks/space/blog/__TopicPrometheus)
- [The Zen of Prometheus](https://prometheus.io/docs/practices/the_zen/)

### Forums

- [Prometheus channel on Reddit](https://www.reddit.com/r/PrometheusMonitoring/)
- [Questions tagged 'promql' on StackOverflow](https://stackoverflow.com/questions/tagged/promql)

### Tutorials

- [Robust Perception (Prometheus company) YouTube channel](https://www.youtube.com/channel/UCtiFWOeRSTP3M6QUnTEKwpw)
- [Monitoring a Machine with Prometheus: A Brief Introduction (Brian Brazil)](https://www.youtube.com/watch?v=WUkNnY65htQ)
- [PromCom (Prometheus Monitoring Conference)](https://www.youtube.com/channel/UC4pLFely0-Odea4B2NL1nWA)

## PromQL

For day to day use, there's only a handful of PromQL patterns you need to know. Let's look at them (Brian Brazil):
- https://www.robustperception.io/common-query-patterns-in-promql/

With the instant rate if scrapes become more frequent, graphs automatically improve in resolution (Brian Brazil):
- https://www.robustperception.io/irate-graphs-are-better-graphs/

Which targets have the most samples? (Brian Brazil)
- https://www.robustperception.io/which-targets-have-the-most-samples/

Let's look at the Graphite, InfluxDB and Prometheus query languages and see how the same ideas are represented in each (Brian Brazil):
- https://www.robustperception.io/translating-between-monitoring-languages/

Composing range vector functions in PromQL:
- https://www.robustperception.io/composing-range-vector-functions-in-promql

Aggregating across batch job runs with ``push_time_seconds`` (Pushgateway):
- https://www.robustperception.io/aggregating-across-batch-job-runs-with-push_time_seconds

### Queries Examples

PromQL cheat sheet - Usage and examples of basics, aggregations and functions:
- https://github.com/jitendra-1217/promql.cheat.sheet

A short collection of PromQL queries and alert rules for different use cases:
- https://github.com/infinityworks/prometheus-example-queries

Walkthrough on how to build a query and narrows it down to aggregate the values of multiple time series:
- https://www.weave.works/blog/promql-queries-for-the-rest-of-us/

**NOTE**: Weaveworks ceased operations in 2024; the weave.works blog articles are no longer available (the link now resolves to the company home page).

PromQL deep dive (WeaveWorks, still available on Slideshare):
- https://www.slideshare.net/weaveworks/promql-deep-dive-the-prometheus-query-language

### How to query Prometheus

An introduction from scratch on how to build PromQL queries on an Ubuntu 14.04 test installation (Julius Volz, Prometheus developer; historical):
- https://www.digitalocean.com/community/tutorials/how-to-query-prometheus-on-ubuntu-14-04-part-1
- https://www.digitalocean.com/community/tutorials/how-to-query-prometheus-on-ubuntu-14-04-part-2

Modern equivalents (maintained):
- [How To Use Alertmanager and Blackbox Exporter to Monitor Your Web Server (DigitalOcean)](https://www.digitalocean.com/community/tutorials/how-to-use-alertmanager-and-blackbox-exporter-to-monitor-your-web-server-on-ubuntu-16-04) — steps compatible with current Ubuntu LTS releases (22.04, 24.04)
- [Getting started with Prometheus](https://prometheus.io/docs/prometheus/latest/getting_started/) + [Monitoring Linux host metrics with the Node Exporter](https://prometheus.io/docs/guides/node-exporter/) — official step-by-step query walkthrough
- [How to Install Prometheus Monitoring Stack for DOKS Cluster (DigitalOcean developer center)](https://www.digitalocean.com/community/developer-center/how-to-install-prometheus-monitoring-stack-for-doks-cluster) — recent, Kubernetes-focused

### Prometheus instrumentation details

Blog posts from Brian Brazil (main developer of Prometheus):
- [How does a counter work?](https://www.robustperception.io/how-does-a-prometheus-counter-work/)
- [How does a gauge work?](https://www.robustperception.io/how-does-a-prometheus-gauge-work/)
- [Counting with Prometheus (YouTube)](https://www.youtube.com/watch?v=67Ulrq6DxwA)

## Operations

- [Prometheus Configuration](https://prometheus.io/docs/operating/configuration/)
- [Prometheus Storage (on disk)](https://prometheus.io/docs/operating/storage/)
- [Prometheus external storage integration](https://prometheus.io/docs/operating/integrations/#remote-endpoints-and-storage)
- [Prometheus Security Model](https://prometheus.io/docs/operating/security/)
- [promtool (validate configuration and rules)](https://prometheus.io/docs/prometheus/latest/command-line/promtool/)

### Blog posts from Brian Brazil

- [Reload Prometheus configuration](https://www.robustperception.io/reloading-prometheus-configuration/)
- [Dealing with "too many open files" errors](https://www.robustperception.io/dealing-with-too-many-open-files)
- [Alerting on approaching open file limits](https://www.robustperception.io/alerting-on-approaching-open-file-limits)
- [Using the Textfile collector from a shell script](https://www.robustperception.io/using-the-textfile-collector-from-a-shell-script)
- [Quick sensor metrics with the Textfile collector](https://www.robustperception.io/quick-sensor-metrics-with-the-textfile-collector)

### Deploying and HA

- [Scaling and federating Prometheus](https://www.robustperception.io/scaling-and-federating-prometheus/)
- [HA Prometheus for alerting and notifications](https://www.robustperception.io/high-availability-prometheus-alerting-and-notification/)
- [Deploying Prometheus at the Wikimedia Foundation (FOSDEM 2017)](https://fosdem.org/2017/schedule/event/deploying_prometheus_at_wikimedia_foundation/)
