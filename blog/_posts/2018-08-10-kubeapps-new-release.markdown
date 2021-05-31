---
layout:	post
title:	"Kubeapps New Release"
date:	2018-08-10
---

  ![](/img/1*D-netGmC8_qX2dXeeBFLoQ.png)Kubeapps, your applications dashboard for KubernetesKubeapps is an application dashboard for your Kubernetes cluster. As opposed to the Kubernetes dashboard, Kubeapps provides a central location for your applications and their full life cycle.

Over the past few weeks we have been working on a set of new features in Kubeapps and we are happy to announce that those new features are available in the [latest release of Kubeapps](https://github.com/kubeapps/kubeapps/releases/tag/v1.0.0-alpha.5).

Some of these features are not backwards compatible, so we recommend that you [use the migration guide](https://github.com/kubeapps/kubeapps/blob/master/docs/user/migrating-to-v1.0.0-alpha.5.md) if you need to migrate from the previous Kubeapps version.

The latest features include:

#### Helm Chart to deploy Kubeapps

In this release, we have made the deployment of Kubeapps even easier. We are deprecating the CLI tool, and are giving users a Helm Chart to deploy instead. The Helm chart provides a lot more flexibility in the way you deploy Kubeapps into your cluster and it is a packaging format very popular in the Kubernetes community.

#### Helm Proxy and Security

One of the initial goals for Kubeapps was to be able to provide a secured Tiller (the in-cluster Helm component) deployment, following the Helm security best practices. You can check out our guide on our recommended way to secure Tiller with TLS certificates and RBAC [here](https://github.com/kubeapps/kubeapps/blob/master/docs/user/securing-kubeapps.md).

We wanted to go a step further and make sure that the service account interacting with Kubeapps and installing Helm charts had the needed permissions required to deploy the components of the chart. For this we deploy alongside with Tiller, as a sidecar container, a [proxy](https://github.com/kubeapps/kubeapps/blob/master/cmd/tiller-proxy/README.md) that validates the requests checking that the user is allowed to perform the requested operation and finally redirect it to Tiller. For example, if the chart that the user wants to deploy has a Deployment, a ConfigMap and a Secret, the proxy will check that the service account initiating the request has RoleBindings associated to Roles that allow them to create those objects in the requested namespace before sending the request to Tiller.

#### Service Catalog integration improvements

We have made a lot of improvements with Kubeapps Service Catalog integration in this new release. We have implemented forms that get generated based on the JSON schema for a particular service. Now, when you request an instance for a particular service, you will be prompted with a form that is specific to the data required by the Service Broker for that particular service, instead of having to fill the JSON manually yourself.

For example, Google’s Cloud PubSub service, available through their GCP Service Broker, only requires a topic to fill in to provision an instance. Instead of having to submit a JSON with the data, you can fill that information through Kubeapps in a form:

![](/img/1*Bz7mhDK6GJ5_4G8sKzvkxA.png)

Once you have provisioned an instance of a service you need to request a binding, which you will be able to use with your application. When requesting a binding you will get presented with another form to fill, based again on the specific service:

![](/img/1*_E930kY73h0JjTTyXIxqoA.png)

If you need a dashboard in your cluster to manage your applications and their life cycles, deploy Kubeapps in your Kubernetes cluster today following [these simple instructions](https://github.com/kubeapps/kubeapps/blob/master/docs/user/getting-started.md) and let us know what you think!

  