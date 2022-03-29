---
layout: post
title: "Setting up OpenTelemetry with Datadog and Heroku"
date: 2022-02-07
---

The OpenTelemetry Collector is a vendor agnostic agent to collect and export telemetry data. Datadog has an [exporter available](https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/exporter/datadogexporter) to receive traces and metrics from the OpenTelemetry SDKs and forward them to Datadog.

In this blog post we explain how to set this up in Heroku. If you want to check an end-to-end Python application sample, you can check [this GitHub repository](https://github.com/arapulido/heroku-otel-example).

# Adding the OTEL Collector Buildpack

The first thing that we need to do is to add the OTEL Collector to our Heroku application. I looked around and discovered [this buildpack](https://github.com/Djiit/heroku-buildpack-otelcol) that adds the collector to your application. Unfortunately, this buildpack is a bit outdated, as in the latest versions of the collector the project has split into two distributions: core, and contrib, and most of the exporters are no longer available in the core distribution.

I have created [a fork of the mentioned buildpack](https://github.com/arapulido/heroku-buildpack-otelcol) adding the possibility to deploy the contrib distribution instead. I have sent [a PR upstream](https://github.com/Djiit/heroku-buildpack-otelcol/pull/3), and I will update this blog post once it is accepted. But, for now, we will be using my fork for this.

> 📝 EDIT: The PR has now been merged, so you can now used the [original buildpack](https://github.com/Djiit/heroku-buildpack-otelcol) instead of my fork.

Add this buildpack to your application. The `OTELCOL_CONTRIB` environment variable tells the buildpack to use the Collector contrib distribution:

```
heroku config:add OTELCOL_CONTRIB=true
heroku buildpacks:add https://github.com/arapulido/heroku-buildpack-otelcol
```

# Configuring the OTEL Collector to send data to Datadog

In the root of your application, create a new `otelcol` and add your OTEL configuration as a `config.yml` file there. You can find an example of a working configuration for Datadog in [the sample application git repository](https://github.com/arapulido/heroku-otel-example/blob/main/otelcol/config.yml).

You should add your API key to that file under [`api.key`](https://github.com/arapulido/heroku-otel-example/blob/main/otelcol/config.yml#L53-L59):

![OTEL Configuration](/img/otel_config.png)

# Getting traces in Datadog

Once the buildpack has been added and the `otelcol/config.yml` pushed, the OTel Collector will start automatically when your dyno starts and will collect traces and metrics and send them to Datadog.

In the [sample application repository](https://github.com/arapulido/heroku-otel-example), you can see how this application was instrumented with [the OpenTelemetry SDK](https://github.com/arapulido/heroku-otel-example/blob/main/flask_example.py).

The following screenshot shows one of the traces from the sample application, collected by the OpenTelemetry Collector and pushed to Datadog:

![A trace in Datadog collected by OpenTelemetry](/img/otel_trace.png)

# Summary

Datadog maintains a [Heroku Buildpack](https://docs.datadoghq.com/agent/basic_agent_usage/heroku/) that deploys the Datadog Agent to gather telemetry (metrics, logs and traces) from your Heroku Dynos, but it is also possible to use the OpenTelemetry Collector to collect metrics and traces from your Heroku application and send them to Datadog through the Datadog exporter.