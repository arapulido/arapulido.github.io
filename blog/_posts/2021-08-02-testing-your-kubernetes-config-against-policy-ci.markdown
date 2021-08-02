---
layout:	post
title:	"Testing your Kubernetes configuration against your Gatekeeper policy as part of your CI/CD pipeline"
date:	2021-08-02
---

# Introduction to Gatekeeper

[Gatekeeper](https://github.com/open-policy-agent/gatekeeper) is a CNCF project, part of [Open Policy Agent](https://www.openpolicyagent.org/), that enforces OPA policies in Kubernetes clusters through an Admission Controller Webhook.

Once you have the Gatekeeper controller running in your cluster, you can start writing reusable policies for your cluster through a CustomResourceDefinition object called `ConstraintTemplate`. These objects contain the policy code written in [Rego](https://www.openpolicyagent.org/docs/latest/policy-language/), a domain specific language for OPA. These `ConstraintTemplates` are not policies themselves, but parametrized templates that then get instantiated into policies through `Constraint` objects.

![Diagram showing the relation between ConstraintTemplates and Constraints](/img/constrainttemplate_constraint.png)

This makes Gatekeeper great for Kubernetes and it allows to easily reuse policies. There is even [an open source repository of ready to use `ConstraintTemplates`](https://github.com/open-policy-agent/gatekeeper-library) that contains commonly needed policies for a Kubernetes cluster.

This post assumes that you are familiar with using Gatekeeper in your Kubernetes cluster, as we will be focusing on how to enable it in a CI loop. If you need to learn more about Gatekeeper and how to enable it in your cluster I would recommend you to read [the official documentation](https://open-policy-agent.github.io/gatekeeper/website/docs/).

# Testing your policies as part of CI

This is great, but the following question would be: if I am following a GitOps process for my Kubernetes cluster and all my configuration changes got through a pull request and get committed to a git repository before they get applied to my cluster, how can I also test my configuration against my company policy automatically as part of CI to avoid trying to apply objects that will be rejected?

OPA has a project called [`conftest`](https://www.conftest.dev/) that enables staticly testing a set of Rego policies against structured data, including YAML description files. The problem with using `conftest` if you are writing your policies for Gatekeeper is that you will need to maintain two sets of policies: the ones parametrized as `ConstraintTemplates` and a set of fully written Rego policies that already have the different parameters you have created.

For this reason, if your mainly using OPA with Gatekeeper I would recommend to test your policies against a lightweigth Kubernetes cluster, like [`kind`](https://kind.sigs.k8s.io/), and test against your real policies as part of your CI loop.

Gatekeeper hooks to the ValidationAdmissionWebhook, part of the Kubernetes API server. When a request gets to the API server it goes through authentication, then authorization and then through the list of admission controllers. If the request doesn't follow your cluster policy, it will be rejected at that point. This means that using a server side dry run of the request (available since Kubernetes 1.13) is enough to test your policies, without needing to really apply the objects, making your test pipelines a lot faster.

![Diagram showing an API request with Gatekeeper](/img/api_request_diagram.png)

## Using GitHub Actions with a sample repository

As an example on how this could be implemented in the CI tool of your choice, we will be using a sample GitHub repository and GitHub Actions for our CI pipeline. The repository, called `gatekeeper_testing` is open source and can be found at: [https://github.com/arapulido/gatekeeper_testing/](https://github.com/arapulido/gatekeeper_testing/).

The repository has the following structure:

```
.
├── kubernetes-config
└── policies
    ├── constrainttemplates
    └── constraints
```

* `kubernetes-config` includes all the Kubernetes objects we deploy in our sample cluster
* `policies/constrainttemplates` includes the Gatekeeper constraint templates we have in our sample cluster
* `policies/constraints` includes the Gatekeeper constraints we have in our sample cluster

We are assuming that `kubernetes-config` is the configuration we want to apply in our cluster and that, following GitOps best practices, new or modified objects will be committed to this repository before applying them to our cluster.

To test that any changes made to the objects are allowed by our Gatekeeper policies we create [a GitHub Actions workflow](https://github.com/arapulido/gatekeeper_testing/blob/main/.github/workflows/test-policies.yaml) that will test any changes to the repository against the policy:

```yaml
name: Test Kubernetes objects against Gatekeeper policies

on: push 

jobs:
  test-policies:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v2
        with:
          fetch-depth: 0

      -  name: Install kubectl
         uses: azure/setup-kubectl@v1
         id: install

      - name: Create kind cluster
        uses: helm/kind-action@v1.2.0

      - name: Deploy Gatekeeper 3.5
        run: kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/release-3.5/deploy/gatekeeper.yaml

      - name: Wait for Gatekeeper controller
        run: kubectl -n gatekeeper-system wait --for=condition=Ready --timeout=60s pod -l control-plane=controller-manager

      - name: Apply all ConstraintsTemplates 
        run: kubectl apply --recursive -f policies/constrainttemplates/
      
      - name: Wait
        run: sleep 2

      - name: Apply all Constraints 
        run: kubectl apply --recursive -f policies/constraints/

      - name: Wait
        run: sleep 2

      - name: Try to apply all of our Kubernetes configuration
        run: kubectl apply --recursive -f kubernetes-config --dry-run=server
```

After creating the Kubernetes cluster, deploying Gatekeeper and applying the policy, in the last step we try to create all of our Kubernetes configuration, but using a server side dry-run (the objects that follow the policy won't be really created, making the pipeline faster):

```yaml
- name: Try to apply all of our Kubernetes configuration
  run: kubectl apply --recursive -f kubernetes-config --dry-run=server
```

You can see that [the tests fail if any of the objects dont't follow the cluster policy](https://github.com/arapulido/gatekeeper_testing/runs/3202208785?check_suite_focus=true#step:11:1):

![Screenshot of tests failing](/img/test-fail.png)

And [they pass once the objects follow the cluster policy](https://github.com/arapulido/gatekeeper_testing/runs/3202940754?check_suite_focus=true#step:11:1):

![Screenshot of tests failing](/img/test-pass.png)

# Summary

In this blog post we have seen how Gatekeeper policies can be integrated in a CI loop to be able to test new or modified Kubernetes objects against policy using a lightweight Kubernetes cluster and server side dry runs.

Take into account that having policy checks as part of your CI loop is an addition to having the Gatekeeper controller running in your clusters, and that you should always be running Gatekeeper in your clusters to make sure your company's policy is being followed.


