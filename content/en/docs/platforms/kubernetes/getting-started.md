---
title: Getting Started
weight: 1
# prettier-ignore
cSpell:ignore: filelog filelogreceiver kubelet kubeletstats kubeletstatsreceiver loggingexporter otlpexporter sattributes sattributesprocessor sclusterreceiver sobjectsreceiver
---

This page will walk you through the fastest way to get started monitoring your
Kubernetes cluster using OpenTelemetry. It will focus on collecting metrics and
logs for Kubernetes clusters, nodes, pods, and containers, as well as enabling
the cluster to support services emitting OTLP data.

If you're looking to see OpenTelemetry in action with Kubernetes, the best place
to start is the [OpenTelemetry Demo](/docs/demo/kubernetes-deployment/). The
demo is intended to illustrate the implementation of OpenTelemetry, but it is
not intended to be an example of how to monitor Kubernetes itself. Once you
finish with this walkthrough, it can be a fun experiment to install the demo and
see how all the monitoring responds to an active workload.

If you're looking to start migrating from Prometheus to OpenTelemetry, or if
you're interested in using the OpenTelemetry Collector to collect Prometheus
metrics, see
[Prometheus Receiver](../collector/components/#prometheus-receiver).

## Overview

Kubernetes exposes a lot of important telemetry in many different ways. It has
logs, events, metrics for many different objects, and the data generated by its
workloads.

To collect all of this data we'll use the
[OpenTelemetry Collector](/docs/collector/). The collector has many different
tools at its disposal which allow it to efficiently collect all this data and
enhance it in meaningful ways.

To collect all the data, we'll need two installations of the collector, one as a
[Daemonset](/docs/collector/deployment/agent/) and one as a
[Deployment](/docs/collector/deployment/gateway/). The Daemonset installation of
the collector will be used to collect telemetry emitted by services, logs, and
metrics for nodes, pods, and containers. The deployment installation of the
collector will be used to collect metrics for the cluster and events.

To install the collector, we'll use the
[OpenTelemetry Collector Helm chart](../helm/collector/), which comes with a few
configuration options that will make configure the collector easier. If you're
unfamiliar with Helm, check out [the Helm project site](https://helm.sh/). If
you're interested in using a Kubernetes operator, see
[OpenTelemetry Operator](../operator/), but this guide will focus on the Helm
chart.

## Preparation

This guide will assume the use of a [Kind cluster](https://kind.sigs.k8s.io/),
but you're free to use any Kubernetes cluster that you deem fit.

Assuming you already have
[Kind installed](https://kind.sigs.k8s.io/#installation-and-usage), create a new
kind cluster:

```sh
kind create cluster
```

Assuming you already have [Helm installed](https://helm.sh/docs/intro/install/),
add the OpenTelemetry Collector Helm chart so it can be installed later.

```sh
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
```

## Daemonset Collector

The first step to collecting Kubernetes telemetry is to deploy a daemonset
instance of the OpenTelemetry Collector to gather telemetry related to nodes and
workloads running on those node. A daemonset is used to guarantee that this
instance of the collector is installed on all nodes. Each instance of the
collector in the daemonset will collect data only from the node on which it is
running.

This instance of the collector will use the following components:

- [OTLP Receiver](https://github.com/open-telemetry/opentelemetry-collector/tree/main/receiver/otlpreceiver):
  to collect application traces, metrics and logs.
- [Kubernetes Attributes Processor](../collector/components/#kubernetes-attributes-processor):
  to add Kubernetes metadata to incoming application telemetry.
- [Kubeletstats Receiver](../collector/components/#kubeletstats-receiver): to
  pull node, pod, and container metrics from the API server on a kubelet.
- [Filelog Receiver](../collector/components/#filelog-receiver): to collect
  Kubernetes logs and application logs written to stdout/stderr.

Let's break these down.

### OTLP Receiver

The
[OTLP Receiver](https://github.com/open-telemetry/opentelemetry-collector/tree/main/receiver/otlpreceiver)
is the best solution for collecting traces, metrics, and logs in the
[OTLP format](/docs/specs/otel/protocol/). If you are emitting application
telemetry in another format, there is a good chance that
[the Collector has a receiver for it](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/receiver),
but for this tutorial we'll assume the telemetry is formatted in OTLP.

Although not a requirement, it is a common practice for applications running on
a node to emit their traces, metrics, and logs to a collector running on the
same node. This keeps network interactions simple and allows easy correlation of
Kubernetes metadata using the `k8sattributes` processor.

### Kubernetes Attributes Processor

The
[Kubernetes Attributes Processor](../collector/components/#kubernetes-attributes-processor)
is a highly recommended component in any collector receive telemetry from
Kubernetes pods. This processor automatically discovers Kubernetes pods,
extracts their metadata such as pod name or node name, and adds the extracted
metadata to spans, metrics, and logs as resource attributes. Because it adds
Kubernetes context to your telemetry, the Kubernetes Attributes Processor lets
you correlate your application's traces, metrics, and logs signals with your
Kubernetes telemetry, such as pod metrics and traces.

### Kubeletstats Receiver

The [Kubeletstats Receiver](../collector/components/#kubeletstats-receiver) is
the receiver that gathers metrics about the node. It will gather metrics like
container memory usage, pod cpu usage, and node network errors. All of the
telemetry includes Kubernetes metadata like pod name or node name. Since we're
using the Kubernetes Attributes Processor, we'll be able to correlate our
application traces, metrics, and logs with the metrics produced by the
Kubeletstats Receiver.

### Filelog Receiver

The [Filelog Receiver](../collector/components/#filelog-receiver) will collect
logs written to stdout/stderr by tailing the logs Kubernetes writes to
`/var/log/pods/*/*/*.log`. Like most log tailers, the filelog receiver provides
a robust set of actions that allow you to parse the file however you need.

Someday you may need to configure a Filelog Receiver on your own, but for this
walkthrough the OpenTelemetry Helm Chart will handle all the complex
configuration for you. In addition, it will extract useful Kubernetes metadata
based on the file name. Since we're using the Kubernetes Attributes Processor,
we'll be able to correlate the application traces, metrics, and logs with the
logs produced by the Filelog Receiver.

---

The OpenTelemetry Collector Helm chart makes configuring all of these components
in a daemonset installation of the collector easy. It will also take care of all
of the Kubernetes-specific details, such as RBAC, mounts and host ports.

One caveat - the chart doesn't send the data to any backend by default. If you
want to actually use your data in your favorite backend you'll need to configure
an exporter yourself.

The following `values.yaml` is what we'll use

```yaml
mode: daemonset

image:
  repository: otel/opentelemetry-collector-k8s

presets:
  # enables the k8sattributesprocessor and adds it to the traces, metrics, and logs pipelines
  kubernetesAttributes:
    enabled: true
  # enables the kubeletstatsreceiver and adds it to the metrics pipelines
  kubeletMetrics:
    enabled: true
  # Enables the filelogreceiver and adds it to the logs pipelines
  logsCollection:
    enabled: true
## The chart only includes the loggingexporter by default
## If you want to send your data somewhere you need to
## configure an exporter, such as the otlpexporter
# config:
#   exporters:
#     otlp:
#       endpoint: "<SOME BACKEND>"
#   service:
#     pipelines:
#       traces:
#         exporters: [ otlp ]
#       metrics:
#         exporters: [ otlp ]
#       logs:
#         exporters: [ otlp ]
```

To use this `values.yaml` with the chart, save it to your preferred file
location and then run the following command to install the chart

```sh
helm install otel-collector open-telemetry/opentelemetry-collector --values <path where you saved the chart>
```

You should now have a daemonset installation of the OpenTelemetry Collector
running in your cluster collecting telemetry from each node!

## Deployment Collector

The next step to collecting Kubernetes telemetry is to deploy a deployment
instance of the Collector to gather telemetry related to the cluster as a whole.
A deployment with exactly one replica ensures that we don't produce duplicate
data.

This instance of the Collector will use the following components:

- [Kubernetes Cluster Receiver](../collector/components/#kubernetes-cluster-receiver):
  to collect cluster-level metrics and entity events.
- [Kubernetes Objects Receiver](../collector/components/#kubernetes-objects-receiver):
  to collect objects, such as events, from the Kubernetes API server.

Let's break these down.

### Kubernetes Cluster Receiver

The
[Kubernetes Cluster Receiver](../collector/components/#kubernetes-cluster-receiver)
is the Collector's solution for collecting metrics about the state of the
cluster as a whole. This receiver can gather metrics about node conditions, pod
phases, container restarts, available and desired deployments, and more.

### Kubernetes Objects Receiver

The
[Kubernetes Objects Receiver](../collector/components/#kubernetes-objects-receiver)
is the Collector's solution for collecting Kubernetes objects as logs. Although
any object can be collected, a common and important use case is to collect
Kubernetes events.

---

The OpenTelemetry Collector Helm chart streamlines the configuration for all of
these components in a deployment installation of the Collector. It will also
take care of all of the Kubernetes-specific details, such as RBAC and mounts.

One caveat - the chart doesn't send the data to any backend by default. If you
want to actually use your data in your preferred backend, you'll need to
configure an exporter yourself.

The following `values.yaml` is what we'll use:

```yaml
mode: deployment

image:
  repository: otel/opentelemetry-collector-k8s

# We only want one of these collectors - any more and we'd produce duplicate data
replicaCount: 1

presets:
  # enables the k8sclusterreceiver and adds it to the metrics pipelines
  clusterMetrics:
    enabled: true
  # enables the k8sobjectsreceiver to collect events only and adds it to the logs pipelines
  kubernetesEvents:
    enabled: true
## The chart only includes the loggingexporter by default
## If you want to send your data somewhere you need to
## configure an exporter, such as the otlpexporter
# config:
# exporters:
#   otlp:
#     endpoint: "<SOME BACKEND>"
# service:
#   pipelines:
#     traces:
#       exporters: [ otlp ]
#     metrics:
#       exporters: [ otlp ]
#     logs:
#       exporters: [ otlp ]
```

To use this `values.yaml` with the chart, save it to your preferred file
location and then run the following command to install the chart:

```sh
helm install otel-collector-cluster open-telemetry/opentelemetry-collector --values <path where you saved the chart>
```

You should now have a deployment installation of the collector running in your
cluster that collects cluster metrics and events!
