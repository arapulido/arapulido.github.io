---
layout:	post
title:	"Understanding Datadog's Tag Cardinality in Kubernetes"
date:	2021-11-15
---

![Screenshot of an example of cardinality](/img/cardinality.png)

# What is metric cardinality and why should I care?

When metrics are emitted, each of the values of a particular metric are sent with a set of tags. Depending on the tags that a metric emits, you can filter and/or aggregate by those, allowing slicing and dicing the data as needed in order to discover information.

The cardinality of a metric is basically the number of tags and possible values of each of those tags. An example of a low cardinality tag could be `region`, if the potential values for that tag are only `APAC`, `EMEA` and `AMER`. An example of a high cardinality (potentially infinite!) tag could be something like `customer_id`, as there is no limit in the potential values of that tag.

There are two main reasons why trying to get granularity right is a good idea:

* If you add a high-cardinality tag to your metric, but that tag is not really adding information to the metric (i.e. you don't care the specific customer, you only care about  aggregated data, like country), it will clutter your metrics (and the Datadog UI).
* If this is part of your [custom metrics, this directly affects your billing](https://docs.datadoghq.com/metrics/custom_metrics/).

# Datadog agent's out-of-the-box tags

To make things easier, the Datadog agent in Kubernetes collects a set of tags related to your Kubernetes environment, like `kube_deployment`, `kube_service`, `cluster_name`, etc. These are tags that will be relevant for almost any metric emitted from Kubernetes.

But there are tags that the agent collects that can increase the cardinality of a metric, for example, `pod_name`. `pod_name` is a tag with a very high cardinality, as pods are ephemeral, and when part of a Deployment or Daemonset, they take a different name every time they get rescheduled.

To make sure that the user is in control of what tags to emit, the set of tags that are sent by default by the agent is configurable, using the environment variables `DD_CHECKS_TAG_CARDINALITY` and `DD_DOGSTATSD_TAG_CARDINALITY`. These variables take the values of `low`, `orchestrator`, `high` and default to `low`. `pod_name`, for example, is only emitted for a check if `DD_CHECKS_TAG_CARDINALITY` is set to `orchestrator` or `high`.

[The full list of out-of-the-box tags the Kubernetes agent emits, and their level of cardinality, is available in the official documentation](https://docs.datadoghq.com/agent/kubernetes/tag/?tab=containerizedagent#out-of-the-box-tags).

# But if the default is `low`, why am I seeing some metrics emitting `pod_name`?

There are several integrations that override the default cardinality value, as for the type of metrics they emit, they require a set of tags to really be useful.

For example, for [the Kubelet integration](https://github.com/DataDog/integrations-core/tree/master/kubelet), which gathers metrics that are specific to a pod, adding tags like `pod_name` is needed, to make sure we are able to pinpoint issues with a specific pod, so [the integration overrides the cardinality of the metrics emitted by this integration to always be `orchestrator`](https://github.com/DataDog/integrations-core/blob/master/kubelet/datadog_checks/kubelet/kubelet.py#L269).

# Conclusion

The Datadog Kubernetes agent attaches some out-of-the-box tags to the metrics it emits, but high cardinality tags are only sent if the user modifies the agent configuration. Before modifying these options it is important to understand the consequences, including a potential increase of custom metrics.
