---
layout:	post
title:	"Kubeapps for Oracle Container Engine for Kubernetes"
date:	2018-10-24
---

  *This article originally ran on October 22nd on the Oracle Cloud Native Blog.*

Kubeapps is a web-based UI for deploying and managing applications in Kubernetes clusters. It allows your cluster users to deploy applications packaged as Helm charts directly from their browsers.

Bitnami has been working on making the experience of running Kubeapps on top of an [Oracle Container Engine for Kubernetes](https://docs.cloud.oracle.com/iaas/Content/ContEng/Concepts/contengoverview.htm) (OKE) cluster great, including testing and improving Bitnami’s authored Helm charts so they work out of the box in OKE clusters.

In this blog post, we will explain how you can deploy Kubeapps into your OKE cluster and use it to deploy any of the many Bitnami’s Helm charts available. This post assumes that you already have an OKE cluster and kubectl is configured to talk to it.

### Install Helm CLI locally and in your cluster

To deploy Kubeapps you will need to install the Helm CLI tool (`helm`). Follow [the instructions on the Helm Github page to install the Helm CLI into your system](https://github.com/helm/helm/releases/). Take into account that Kubeapps 1.0 requires Helm 2.10 or later.

When creating an OKE cluster you have the option to have Tiller (Helm’s server component) deployed into your cluster.

You can check if Tiller is already running in your cluster:

{% highlight sh %}
$ kubectl get pods -n kube-system
NAME READY STATUS RESTARTS AGE  
kube-dns-66d8df795b-j6jnb 3/3 Running 0 22h  
[...]  
tiller-deploy-5f547b596c-djbnb 1/1 Running 0 22h
{% endhighlight %}

If you have a pod called tiller-deploy-* running, then Tiller is already deployed in your cluster. In that case you will need to upgrade your version running the following command:

{% highlight sh %}
helm init --upgrade --service-account tiller
{% endhighlight %}

If Tiller is not deployed yet in your cluster, you can deploy it easily by running:

{% highlight sh %}
helm init
{% endhighlight %}

### Deploy Kubeapps in your cluster

The next step would be to deploy Kubeapps in your cluster. This can be done with Helm in your terminal by running:

{% highlight sh %}
helm repo add bitnami <https://charts.bitnami.com/bitnami>  
helm install — namespace kubeapps -n kubeapps bitnami/kubeapps
{% endhighlight %}

Kubeapps requires a token to login, then it will be used in any request to make sure that the user has enough permissions to perform the required API calls (if your cluster has RBAC enabled).

For this blog post, we will create a service account with cluster-admin permissions as explained in the Kubeapps documentation.

{% highlight sh %}
kubectl create serviceaccount kubeapps-operator  
kubectl create clusterrolebinding kubeapps-operator — clusterrole=cluster-admin — serviceaccount=default:kubeapps-operator
{% endhighlight %}

With the following command we will reveal the token that we will use to login into the Kubeapps dashboard:

{% highlight sh %}
kubectl get secret $(kubectl get serviceaccount kubeapps-operator -o jsonpath=’{.secrets[].name}’) -o jsonpath=’{.data.token}’ | base64 — decode
{% endhighlight %}

### Accessing the Kubeapps dashboard and logging in

The default values for the options in the Kubeapps Helm chart deploy the Kubeapps main service as a ServiceIP, which cannot be accessed externally. We will use Kubernetes port-forward option to be able to access it locally:

{% highlight sh %}
echo “Kubeapps URL: http://127.0.0.1:8080"  
export POD\_NAME=$(kubectl get pods — namespace kubeapps -l “app=kubeapps” -o jsonpath=”{.items[0].metadata.name}”)  
kubectl port-forward — namespace kubeapps $POD\_NAME 8080:8080
{% endhighlight %}
 
Once the port-forward is running you can access Kubeapps in your browser at <http://localhost:8080>

You will be prompted with a login screen. To log in, you can paste the token you obtained in the previous section:

![](/img/0*Z6oM6t0jL7fFLMEG)Once you are logged in, you can browse all available charts in the Charts link:

![](/img/0*Yxn21kltw_kwcbqM)

### Using Kubeapps to deploy Bitnami charts in your OKE cluster

Bitnami maintains a catalog of more than 50 charts and those have been fully tested in OKE clusters and polished to work out of the box on an OKE cluster. You can have a look to the Helm charts in the Bitnami repo by accessing <http://localhost:8080/charts/bitnami/>

![](/img/0*GpeYul3qumtlWiyg)As an example, we will deploy the Bitnami Wordpress Helm chart through Kubeapps.

![](/img/0*b4LgQrwz2efuYare)After selecting the Wordpress chart, we will deploy it with the default values, which will create a LoadBalancer service and will deploy a MariaDB database in the cluster. You can check that both pods are up and running, and that PVCs, backed by OCI, have been provisioned:

{% highlight sh %}
$ kubectl get pods
NAME READY STATUS RESTARTS AGE  
my-wordpress-mariadb-0 1/1 Running 0 4m
my-wordpress-wordpress-5cfc65b9-dnblz 1/1 Running 0 4m
$ kubectl get pvc
NAME STATUS VOLUME CAPACITY ACCESS MODES STORAGECLASS AGE  
data-my-wordpress-mariadb-0 Bound ocid1.volume.oc1.phx.abyhqljsslsar3caoqzxotoaci3svroeci4g2pkh7rva6gtck4tqbckslmnq 50Gi RWO oci 4m  
my-wordpress-wordpress Bound ocid1.volume.oc1.phx.abyhqljsiewzlxlyuaeulf6nsm4w2wqnzwq3ho3vbisrcjumroga6l765r6q 50Gi RWO oci 4m
{% endhighlight %}

Also, as this is a LoadBalancer service, OKE will provide a load balancer with an IP you can use to access your new Wordpress website:

![](/img/0*J4MYE-ekIJLguoLG)![](/img/0*z-iyoZu222r-_DFw)

### Summary

In this blog post, we explained how you can use Kubeapps in your Oracle Container Engine for Kubernetes cluster to deploy and maintain OKE-ready Kubernetes applications from Bitnami. These applications were specifically tested for the Oracle platform, and you can rest assured that they follow Bitnami’s secure and up-to-date packaging standards. You can visit the [Kubeapps Hub](https://hub.kubeapps.com) to keep track of [what Bitnami charts are available and the supported versions](https://hub.kubeapps.com/charts/bitnami).

  