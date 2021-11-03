---
layout:	post
title:	"Getting cert-manager certificates time to expiration in Datadog"
date:	2021-11-03
---

# TL;DR;

[Datadog cert-manager integration](https://github.com/DataDog/integrations-extras/tree/master/cert_manager) [version 2.2.0](https://github.com/DataDog/integrations-extras/releases/tag/cert_manager-2.2.0) includes a new widget in its default dashboard that gets the days to expiration for each certificate in your clusters, color-coding the ones close to expiration:

![Screenshot of the new expiration widget](/img/cert_expiration.png)

If you are using the cert-manager integration, make sure to update it to its latest version to get this new widget to populate correctly. If you are not using the integration and/or want to learn more about the `certmanager_clock_time_seconds` metric, continue reading.

# certmanager_clock_time_seconds

[cert-manager 1.5.0](https://github.com/jetstack/cert-manager/releases/tag/v1.5.0) introduced a new metric, `certmanager_clock_time_seconds` that returns the timestamp of the current time. Thanks to this new metric, we are able to calculate the time left before the expiration of the certificate, thanks to the existance of the `certmanager_certificate_expiration_timestamp_seconds` metric.

The problem is that when `certmanager_clock_time_seconds` metric was added, it was added as a `counter`, where [it should have been a `gauge`](https://github.com/jetstack/cert-manager/issues/4560). If this metric is a `counter`, it gets converted to a Datadog `monotonic counter`, and Datadog only offers the delta between the previous reported value and the current value, making the metric value of the timestamp pretty useless.

Fortunately, Datadog offers overriding [its OpenMetrics default mapping](https://docs.datadoghq.com/integrations/guide/prometheus-metrics/#how-prometheusopenmetrics-metrics-map-to-datadog-metrics), so we are able to fix this on Datadog's side, while it gets fixed upstream.

If you are using the OpenMetrics integration, you can override this metric to be a `gauge` by adding the following annotations in your cert-manager deployment (adding other metrics you want to add as well):

```yaml
ad.datadoghq.com/cert-manager.check_names: '["openmetrics"]'
ad.datadoghq.com/cert-manager.init_configs: '[{}]'
ad.datadoghq.com/cert-manager.instances: |
  [{
    "openmetrics_endpoint": "http://%%host%%:9402/metrics",
    "namespace": "cert_manager",
    "metrics": ['certmanager_clock_time_seconds': 'clock_time', 'certmanager_certificate_expiration_timestamp_seconds': 'certificate.expiration_timestamp'],
    "type_overrides": {'certmanager_clock_time_seconds': 'gauge'}
  }]
```

If you are using the [Datadog cert-manager integration](https://github.com/DataDog/integrations-extras/tree/master/cert_manager), this mapping is already done for you, if you are using version `2.2.0` or newer of the integration.

# Calculating time to expiration for a certificate

Once we have the `certmanager_clock_time_seconds` metric correctly reporting as a `gauge` in Datadog, we are able to start calculating the time to expiration of any given certificate. As an example, let's create a widget that gets the list of certificates, ordered by days to expiration.

Typing `G` in our Datadog environment, the Quick Graph modal open:

![Screenshot of the new expiration widget](/img/quick_graph.png)

For the type of widget, select "Top List" and create the following formula:

![Screenshot of the new expiration widget](/img/days_to_expiration.png)

You will get the number of days for your certificates to expire. This same visualization is already part of the Cert Manager Overview default Dashboard:

![Screenshot of the new expiration widget](/img/cert_expiration.png)

# Conclusion

Thanks to the addition of the `certmanager_clock_time_seconds` metric to the OpenMetrics exporter from cert-manager, we are now able to make calculations related to other timestamp metrics in cert-manager.