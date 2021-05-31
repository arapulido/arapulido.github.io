---
layout:	post
title:	"Using external cloud services from Kubernetes using the Service Catalog"
date:	2018-11-15
---

  Kubernetes is a great platform to run containerized workloads in production. Developers can express their applications orchestration in a declarative way, and set up their CI/CD workflows to deploy into production continuously.

However, many of the cloud native applications depend on managed services offered by the clouds. Developers use databases, pubsub queues, storage, etc. from the major clouds knowing that those services already implement many of the features they would need to manage themselves otherwise (backups, HA, etc.). Combining Kubernetes with external services available from public cloud providers can be a powerful way to deploy cloud native applications. Developers can focus on deploying their applications to their Kubernetes clusters, while delegating things like database management to a public cloud managed service.

In the past, this integration needed to happen outside Kubernetes. Developers needed to provision those services, then connect them to their applications creating secrets with the right credentials, breaking their declarative GitOps workflows. The Service Catalog bridges these two worlds by allowing developers instantiate services outside Kubernetes directly from their Kubernetes cluster, in a full declarative way.

Service Catalog is an extension API that enables applications running in Kubernetes clusters to easily use external managed software offerings, such as a datastore service offered by a cloud provider. These services are provided by a Service Broker, which is an endpoint talking to these providers. Once the cluster administrator deploys a ClusterServiceBroker, several ClusterServiceClasses and ClusterServicePlans will be available in the cluster for users to provision those services. To provision a service, users will create a ServiceInstance object and to connect it to their application they will create a ServiceBinding object.

![](/img/1*p3LaEmZvZXiA0uSmtL-OHQ.png)The real power of the Service Catalog is that all of those actions, like provisioning a service instance, creating a binding and using that binding to connect to an application can be described as declarative YAML files, as any other Kubernetes object. This means they can be part of your application description and you can create those instances or bindings as part of your CI/CD pipeline or directly from your Helm Charts.

I will be [talking about the Service Catalog and how you can use it with your Helm Charts to deploy Kubernetes applications that use some of these cloud managed services at Kubecon North America](https://kccna18.sched.com/event/GrRR). If you are planning to attending, add the talk to your KubeCon schedule. I hope to see you there!

  