---
layout:	post
title:	"Bitnami Kubernetes Production Runtime 1.1"
date:	2019-01-04
---

  At Bitnami we have been working on the [Bitnami Kubernetes Production Runtime](https://github.com/bitnami/kube-prod-runtime/), a curated collection of services needed to deploy on top of your Kubernetes cluster (currently supporting GKE and AKS) to enable logging, monitoring and certificate and DNS management. Gus Lees wrote [a great blog post](https://engineering.bitnami.com/articles/announcing-the-bitnami-kubernetes-production-runtime-bkpr.html) explaining why we created BKPR and what problem are we trying to solve.

One of the goals when we released BKPR 1.0 was that each minor release of BKPR would support 2 versions of Kubernetes, allowing users to follow a tick-tock model for upgrades (they can upgrade BKPR first, with the same Kubernetes version, and then upgrade Kubernetes itself, without having to upgrade both at the same time).

![](/img/1*RWIncsroYwLBNwZw320PxA.png)BKPR K8s versions support modelBKPR 1.0 had support for Kubernetes 1.9 and 1.10, as 1.10 was the default Kubernetes version for AKS and GKE at the time of the release. As both platforms moved to 1.11 we prepared the [BKPR 1.1 release](https://github.com/bitnami/kube-prod-runtime/releases/tag/v1.1.0) with support for 1.10 and 1.11 to keep up with our promise.

Aside from keeping up with Kubernetes releases, we want to continue adding components, features and documentation to BKPR. These are some of the new stuff you will be able to find:

* Grafana support. We have added Grafana to the monitoring stack, making it the perfect companion to Prometheus
* We have improved [our troubleshooting guide](https://github.com/bitnami/kube-prod-runtime/blob/master/docs/troubleshooting.md), with additions on how to debug certificate management and DNS issues.
* Although the default use case for BKPR is to deploy it on a new clean cluster, we also want to support migrating to BKPR if you are already running some of these components in your cluster. We have published [a migration guide for Prometheus](https://github.com/bitnami/kube-prod-runtime/blob/master/docs/migration-guides/prometheus-migration.md), so people can migrate their Prometheus deployment to the BKPR one without losing data from the time series database. We have applied that migration guide ourselves in our production clusters.
* To continue improving the security of BKPR we have moved the Kibana image to a non-root image.
  