---
title: Kubernetes Prometheus and OpenMetrics metrics collection
kind: documentation
aliases:
    - /getting_started/prometheus
    - /getting_started/integrations/prometheus
    - /agent/openmetrics
    - /agent/prometheus
further_reading:
- link: "agent/kubernetes/log"
  tag: "Documentation"
  text: "Collect your application logs"
- link: "/agent/kubernetes/apm"
  tag: "Documentation"
  text: "Collect your application traces"
- link: "/agent/kubernetes/integrations"
  tag: "Documentation"
  text: "Collect automatically your applications metrics and logs"
- link: "/agent/guide/autodiscovery-management"
  tag: "Documentation"
  text: "Limit data collection to a subset of containers only"
- link: "/agent/kubernetes/tag"
  tag: "Documentation"
  text: "Assign tags to all data emitted by a container"
---

Collect your exposed Prometheus and OpenMetrics metrics from your application running inside Kubernetes by using the Datadog Agent, and the [Datadog-OpenMetrics][1] or [Datadog-Prometheus][2] integrations.

## Overview

Starting with version 6.5.0, the Agent includes [OpenMetrics][3] and [Prometheus][4] checks capable of scraping Prometheus endpoints. Datadog recommends using the OpenMetrics check since it is more efficient and fully supports Prometheus text format. For more advanced usage of the `OpenMetricsCheck` interface, including writing a custom check, see the [Developer Tools][5] section. Use the Prometheus check only when the metrics endpoint does not support a text format.

This page explains the basic usage of these checks, which enable you to scrape custom metrics from Prometheus endpoints.

## Setup

### Installation

[Deploy the Datadog Agent in your Kubernetes cluster][6]. OpenMetrics and Prometheus checks are included in the [Datadog Agent][7] package, so you don't need to install anything else on your containers or hosts.

### Configuration

Configure your OpenMetrics or Prometheus check using Autodiscovery, by applying the following `annotations` to your OpenMetrics/Prometheus **pod**:

```yaml
# (...)
metadata:
    #(...)
    annotations:
        ad.datadoghq.com/<CONTAINER_IDENTIFIER>.check_names: |
            ["openmetrics"]
        ad.datadoghq.com/<CONTAINER_IDENTIFIER>.init_configs: |
            [{}]
        ad.datadoghq.com/<CONTAINER_IDENTIFIER>.instances: |
            [
              {
                "prometheus_url": "http://%%host%%:%%port%%/<PROMETHEUS_ENDPOINT> ",
                "namespace": "<METRICS_NAMESPACE_PREFIX_FOR_DATADOG>",
                "metrics": [{"<METRIC_TO_FETCH>":"<NEW_METRIC_NAME>"}]
              }
            ]
spec:
    containers:
        - name: '<CONTAINER_IDENTIFIER>'
```

With the following configuration placeholder values:

| Placeholder                              | Description                                                                                                                                                                                                    |
| ---------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `<PROMETHEUS_PORT>`                      | Port to connect to in order to access the Prometheus endpoint.                                                                                                                                                 |
| `<PROMETHEUS_ENDPOINT>`                  | URL for the metrics served by the container, in Prometheus format.                                                                                                                                             |
| `<METRICS_NAMESPACE_PREFIX_FOR_DATADOG>` | Set namespace to be prefixed to every metric when viewed in Datadog.                                                                                                                                           |
| `<METRIC_TO_FETCH>`                      | Prometheus metrics key to be fetched from the Prometheus endpoint.                                                                                                                                             |
| `<NEW_METRIC_NAME>`                      | Optional parameter which, if set, transforms the `<METRIC_TO_FETCH>` metric key to `<NEW_METRIC_NAME>` in Datadog. If you choose not to use this option, pass a list of strings rather than `key:value` pairs. |

**Note**: See the [sample openmetrics.d/conf.yaml][8] for all available configuration options.

## Getting Started

### Simple metric collection

1. [Launch the Datadog Agent][9].

2. Use [this Prometheus DaemonSet `prometheus.yaml`][10] to launch a Prometheus pod with already the Autodiscovery configuration in it:

    Autodiscovery configuration:

    ```yaml
     # (...)
    spec:
      replicas: 2
      selector:
        matchLabels:
          app: prometheus
          purpose: example
      template:
        metadata:
          labels:
            app: prometheus
            purpose: example
          annotations:
              ad.datadoghq.com/prometheus-example.check_names: |
                ["openmetrics"]
              ad.datadoghq.com/prometheus-example.init_configs: |
                [{}]
              ad.datadoghq.com/prometheus-example.instances: |
                [
                  {
                    "prometheus_url": "http://%%host%%:%%port%%/metrics",
                    "namespace": "documentation_example_kubernetes",
                    "metrics": [ {"promhttp_metric_handler_requests_total": "prometheus.handler.requests.total"}]
                  }
                ]
      # (...)
    ```

     Command to create the Prometheus pod:

    ```shell
    kubectl create -f prometheus.yaml
    ```

3. Go into your [Metric summary][11] page to see the collected metrics: `prometheus_target_interval_length_seconds*`

    {{< img src="integrations/guide/prometheus_kubernetes/prometheus_collected_metric_kubernetes.png" alt="Prometheus metric collected kubernetes">}}

## From custom to official integration

By default, all metrics retrieved by the generic Prometheus check are considered custom metrics. If you are monitoring off-the-shelf software and think it deserves an official integration, don't hesitate to [contribute][5]!

Official integrations have their own dedicated directories. There's a default instance mechanism in the generic check to hardcode the default configuration and metrics metadata. For example, reference the [kube-proxy][12] integration.

## Further Reading

{{< partial name="whats-next/whats-next.html" >}}

[1]: /integrations/openmetrics
[2]: /integrations/prometheus
[3]: https://github.com/DataDog/integrations-core/tree/master/openmetrics
[4]: https://github.com/DataDog/integrations-core/tree/master/prometheus
[5]: /developers/prometheus
[6]: /agent/kubernetes/#installation
[7]: /tagging
[8]: https://github.com/DataDog/integrations-core/blob/master/openmetrics/datadog_checks/openmetrics/data/conf.yaml.example
[9]: https://app.datadoghq.com/account/settings#agent/kubernetes
[10]: /resources/yaml/prometheus.yaml
[11]: https://app.datadoghq.com/metric/summary
[12]: https://github.com/DataDog/integrations-core/tree/master/kube_proxy
