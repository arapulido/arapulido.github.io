---
layout: post
title: "The Datadog Kubernetes Admission Controller and Autoinstrumentation Injection"
date: 2023-02-03
---

The [Datadog Cluster Agent](https://docs.datadoghq.com/containers/cluster_agent/) is a specialized Datadog Agent for Kubernetes clusters that implements features specific to Kubernetes and acts as a proxy between Node Agents and the Kubernetes API.

The Datadog Cluster Agent includes a [Kubernentes MutatingAdmissionWebhook](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#mutatingadmissionwebhook) that is able to modify Kubernetes API requests before they are processed. This allows the Cluster Agent to change the definitions of Kubernetes resources with the goal of improving their observability.

In this post we will explain some of the improvements that are injected in our Kubernetes clusters.

# Kubernetes Admission Controllers

**Note:** This section explains briefly what Kubernetes Admission Controllers are and how they work. If you already know this, feel free to skip this section)

When a request is made to the Kubernetes API server it first goes through two phases:

* Authentication. Is this request authenticated or not?
* Authorization. Can this user perform this action against this type of resource in this particular namespace? This is mostly covered by RBAC.

But once the authenticated user is confirmed to be able to perform the selected action, it goes through a third phase: Admission Controllers.

![Kubernetes API requests flow](/img/admission_controllers.jpg)

Admissions Controllers are small pieces of code, embedded in the API server binary, that can further validate a request or even mutate it. There is a set of precompiled Admission Controllers that are enabled or disabled using an API server command line argument.

From all the Admission Controllers available, there are two that are a bit different. These are the ValidatingAdmissionWebhook and MutatingAdmissionWebhook. These allow for processes outside the API Server to validate or mutate API requests, as they are able to register as a webhook.

The Datadog Cluster Agent implements a webhook registered with the MutatingAdmissionWebhook.

# The Cluster Agent MutatingAdmissionWebhook

We will explain how the webhook works using a sample application. You can reproduce all of the examples below following the instructions in [this GitHub repository](https://github.com/arapulido/datadog-admission-example).

## Enabling the MutatingAdmissionWebhook

The first thing that is needed is to enable the MutatingAdmissionWebhook, as it is not enabled by default. The [official Datadog docs explain how to enable it](https://docs.datadoghq.com/containers/cluster_agent/admission_controller/?tab=operator) depending on the method used to deploy the Datadog Agent.

## Basic resource modifications made by the Cluster Agent

Let's check some of the basic modifications that the Datadog MutatingAdmissionWebhook does to a basic Deployment.

Let's take this Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: ecommerce
    tags.datadoghq.com/service: discounts
    tags.datadoghq.com/env: "development"
    tags.datadoghq.com/version: "1.0" 
  name: discounts
spec:
  replicas: 1
  selector:
    matchLabels:
      tags.datadoghq.com/service: discounts
      tags.datadoghq.com/env: "development"
      tags.datadoghq.com/version: "1.0" 
      app: ecommerce
  strategy: {}
  template:
    metadata:
      labels:
        tags.datadoghq.com/service: discounts
        tags.datadoghq.com/env: "development"
        tags.datadoghq.com/version: "1.0" 
        admission.datadoghq.com/enabled: "true"
        app: ecommerce
    spec:
      containers:
      - image: arapulido/discounts_no_instrumentation:latest
        name: discounts
        command: ["flask"]
        args: ["run", "--port=5001", "--host=0.0.0.0"]
        env:
          - name: FLASK_APP
            value: "discounts.py"
          - name: POSTGRES_PASSWORD
            valueFrom:
              secretKeyRef:
                key: pw
                name: db-password
          - name: POSTGRES_USER
            value: "user"
          - name: POSTGRES_HOST
            value: "db"
          - name: DD_LOGS_INJECTION
            value: "true"
          - name: DD_ANALYTICS_ENABLED
            value: "true"
          - name: DD_PROFILING_ENABLED
            value: "true"
        ports:
        - containerPort: 5001
        resources: {}
```

We can see that it has opted-in being modified using the label `admission.datadoghq.com/enabled: "true"`. 

After applying this definition, checking the resource that was actually created in the cluster, we can see that there are some differences, as the request was mutated:

```yaml
    env:
    - name: DD_VERSION
      value: "1.0"
    - name: DD_SERVICE
      value: discounts
    - name: DD_ENV
      value: development
    - name: DD_ENTITY_ID
      valueFrom:
        fieldRef:
          apiVersion: v1
          fieldPath: metadata.uid
    - name: DD_AGENT_HOST
      valueFrom:
        fieldRef:
          apiVersion: v1
          fieldPath: status.hostIP
```

These environment variables are needed to get the right labeled metrics and traces to Datadog. 

Before the Cluster Agent implemented this MutatingAdmissionWebhook, Datadog users needed to remember to add these environement variables themselves, clutering their resources if they remembered, or not getting the most of Datadog, if they didn't.

# Autoinstrumentation library injection

But the most interesting feature in the MutatingAdmissionWebhook is the ability to automatically inject autoinstrumentation libraries into our pods and start getting traces into Datadog without having to modify our code or our pod definitions.

At the time of writing of this blog post, library injection was available for Javascript, Java, and Python. This sections explains how it works for Python.

We take the same Deployment definition as the section above, but we add a new annotation to the pod template to enable library injection:

```yaml
  template:
    metadata:
      annotations:
        admission.datadoghq.com/python-lib.version: "v1.6.6"
```

After creating the resource in our cluster, we can see that, aside from the environment variables, there are more differences in the created pod. Let's dive in:

An `emptyDir` volume is added to the pod definition and a corresponding volume mount is added to the container:

```yaml
[...]
    volumeMounts:
    - mountPath: /datadog-lib
      name: datadog-auto-instrumentation
[...]
  volumes:
  - emptyDir: {}
    name: datadog-auto-instrumentation
```

An init container is also added to the pod definition:

```yaml
  initContainers:
  - command:
    - sh
    - copy-lib.sh
    - /datadog-lib
    image: gcr.io/datadoghq/dd-lib-python-init:v1.6.6
    name: datadog-lib-init
```

The [only thing this init container does](https://github.com/DataDog/dd-trace-py/blob/1.x/lib-injection/copy-lib.sh) is to copy a python file called `sitecustomize.py` into the volume mount `/datadog-lib` of the containers:

```sh
#!/bin/sh

# This script is used by the admission controller to install the library from the
# init container into the application container.
cp sitecustomize.py "$1/sitecustomize.py"
```

Finally, there is a new environment variable in the main container that will force loading that Python module:

```yaml
    env:
    - name: PYTHONPATH
      value: /datadog-lib/
```

Let's check the content of that module to understand what will happen when loading it:

```python
import os
import sys


def _configure_ddtrace():
    # This import has the same effect as ddtrace-run for the current process.
    import ddtrace.bootstrap.sitecustomize

    bootstrap_dir = os.path.abspath(os.path.dirname(ddtrace.bootstrap.sitecustomize.__file__))
    prev_python_path = os.getenv("PYTHONPATH", "")
    os.environ["PYTHONPATH"] = "%s%s%s" % (bootstrap_dir, os.path.pathsep, prev_python_path)

    # Also insert the bootstrap dir in the path of the current python process.
    sys.path.insert(0, bootstrap_dir)
    print("datadog autoinstrumentation: successfully configured python package")


# Avoid infinite loop when attempting to install ddtrace. This flag is set when
# the subprocess is launched to perform the install.
if "DDTRACE_PYTHON_INSTALL_IN_PROGRESS" not in os.environ:
    try:
        import ddtrace  # noqa: F401

    except ImportError:
        import subprocess

        print("datadog autoinstrumentation: installing python package")

        # Set the flag to avoid an infinite loop.
        env = os.environ.copy()
        env["DDTRACE_PYTHON_INSTALL_IN_PROGRESS"] = "true"

        # Execute the installation with the current interpreter
        try:
            subprocess.run([sys.executable, "-m", "pip", "install", "ddtrace"], env=env)
        except Exception:
            print("datadog autoinstrumentation: failed to install python package")
        else:
            print("datadog autoinstrumentation: successfully installed python package")
            _configure_ddtrace()
    else:
        print("datadog autoinstrumentation: ddtrace already installed, skipping install")
        _configure_ddtrace()
```

The module basically runs `pip install` to install the `ddtrace` package and, once installed, imports [the module that starts autoinstrumentation](https://github.com/DataDog/dd-trace-py/blob/1.x/ddtrace/bootstrap/sitecustomize.py) and adds it to the `PYTHONPATH`. This is the same module that is called when running `ddtrace-run`, the previous way to autoinstrumnent your Python applications with Datadog instrumentation libraries.

Once the MutatingAdmissionWebhook injects those libraries, we will start seeing traces coming into Datadog without any code modification:

![Traces coming into Datadog](/img/traces.jpg)

# Summary

Enabling the Datadog MutatingAdmissionController in a Datadog monitored Kubernetes cluster helps improving the observability of the deployed applications. One of the most useful features is the ability to automatically inject and configure the tracing instrumentation libraries, making the process of making your application observable easier than ever.
