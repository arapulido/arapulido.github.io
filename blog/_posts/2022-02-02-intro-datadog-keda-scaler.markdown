---
layout: post
title: "Introducing the Datadog KEDA scaler"
date: 2022-02-02
---

![Keda Logo](/img/keda-horizontal-color.png)

# What is KEDA

KEDA is a Kubernetes based Event Driven Autoscaler. It allows driving the horizontal pod autoscaling of any deployment in Kubernetes, acting as an [Metrics Server](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#support-for-metrics-apis) for the [Horizontal Pod Autoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/).

What makes KEDA interesting is that it has a pluggable architecture, so there are a large (and increasing) number of ["scalers"](https://keda.sh/docs/latest/scalers/) to choose from to drive your scaling events. KEDA scalers allow you to drive your scaling events based on a different types of events, like an [AWS SQS queue length](https://keda.sh/docs/latest/scalers/aws-sqs/), or an [Apache Kafka topic](https://keda.sh/docs/latest/scalers/apache-kafka/).

If you need to gather events from different sources for your scaling events, using KEDA might be a good option, as, currently, there is a limitation of [one custom metric server per Kubernetes cluster](https://github.com/kubernetes-sigs/custom-metrics-apiserver/issues/70).

# Introducing the Datadog KEDA scaler

When I was researching this project, I realized that there wasn't yet a Datadog scaler, and thought that it would be [a fun project to hack on](https://github.com/kedacore/keda/pull/2354), and learn how KEDA works in the process. KEDA community was very helpful and welcoming, and they provided very useful feedback while I was working on the PR.

Available since [KEDA 2.6](https://github.com/kedacore/keda/discussions/2588), there is a [Datadog KEDA scaler](https://keda.sh/docs/latest/scalers/datadog/), so you can use KEDA to drive scaling events based on any Datadog metric, allowing to express a full Datadog query in the trigger specification:

```yaml
triggers:
- type: datadog
  metadata:
    query: "sum:trace.redis.command.hits{env:none,service:redis}.as_count()"
    queryValue: "7"
    type: "global"
    age: "60"
```

The Datadog KEDA scaler uses Datadog's public API to retrieve the metrics that will then be used to update the corresponding HPA object, created and managed by KEDA.

Right now the KEDA scaler for Datadog can only drive scaling events based on metric values, but in the future it could be expanded to scale based on events, monitor status, etc.

# Datadog's Cluster Agent Custom Metrics Server

Even though there is now a Datadog KEDA scaler that can be used, the default, official, HPA implementation for Datadog continues to be the [Cluster Agent](https://docs.datadoghq.com/agent/cluster_agent/).

Datadog recommends using the [Cluster Agent as Custom Metrics Server](https://docs.datadoghq.com/agent/cluster_agent/external_metrics/?tab=helm), when possible, to drive the HPA based on Datadog metrics.

# Summary

KEDA version 2.6 includes a new Datadog scaler that can be used to drive horizontal scaling events based on any metric available in Datadog. I will write a follow up blog post with a step by step how to use the Datadog KEDA scaler with a real example.
