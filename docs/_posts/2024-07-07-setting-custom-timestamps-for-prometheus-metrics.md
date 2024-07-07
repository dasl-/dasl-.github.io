---
layout: post
title: "Setting custom timestamps for prometheus metrics"
author: "David Leibovic"
---

# TLDR

To associate a custom timestamp with a Prometheus metric:

1. Write a [custom collector](https://prometheus.github.io/client_python/collector/custom/) - you can't use the built-in [`Gauge`](https://prometheus.github.io/client_python/instrumenting/gauge/) class in the Python client.
1. Write a [custom exporter](https://prometheus.io/docs/instrumenting/writing_exporters/). This is a web server that exposes the timestamped metrics in the `*.prom` file such that Prometheus can scrape them. The built-in [textfile collector](https://github.com/prometheus/node_exporter?tab=readme-ov-file#textfile-collector) does not support timestamped metrics.
1. Update your Prometheus config scrape targets with the address of the new exporter.
1. Update your Prometheus config to set [`out_of_order_time_window`](https://prometheus.io/docs/prometheus/2.53/configuration/configuration/#tsdb).

**Contents**
* toc
{:toc}

# Background

By default, the timestamp associated with a Prometheus metric is the timestamp at which the metric was scraped by Prometheus. But sometimes, one wants to associate a custom timestamp with a data point. Prometheus is very opinionated and does not make this easy, but it is possible.

My use case for setting custom timestamps was monitoring local air quality. The New York City government has [an API](https://a816-dohbesp.nyc.gov/IndicatorPublic/data-features/realtime-air-quality/) where the city's air quality is reported. While the API is "realtime", in practice the data is 2 - 4 hours delayed. The data returned from the API is formatted as `<timestamp>,<air_quality>`:

```
2024-07-07T10:00:00,5.59
2024-07-07T11:00:00,6.35
2024-07-07T12:00:00,6.75
2024-07-07T13:00:00,6.59
```

I want Prometheus to associate the timestamp of the air quality reading with the data point rather than the timestamp at which the metric was scraped by Prometheus, which may be 2 - 4 hours later.

# Custom collectors

I'll be working with Prometheus's [Python client](https://github.com/prometheus/client_python) in these examples. I was using a [gauge](https://prometheus.github.io/client_python/instrumenting/gauge/) to record the air quality data. Unfortunately, the gauge's [`set`](https://github.com/prometheus/client_python/blob/09a5ae30602a7a81f6174dae4ba08b93ee7feed2/prometheus_client/metrics.py#L432) method exposes no parameter to use a custom timestamp. I discovered on [github](https://github.com/prometheus/client_python/issues/588#issuecomment-1054724554) that we can accomplish our goal with a [custom collector](https://prometheus.github.io/client_python/collector/custom/). The [`add_metric`](https://github.com/prometheus/client_python/blob/09a5ae30602a7a81f6174dae4ba08b93ee7feed2/prometheus_client/metrics_core.py#L172) method exposes `timestamp` as an optional parameter. The `timestamp` should be a [unix epoch timestamp in milliseconds](https://prometheus.io/docs/instrumenting/exposition_formats/#comments-help-text-and-type-information).

```python
import prometheus_client
from prometheus_client.core import GaugeMetricFamily

class CustomTimestampedGaugeCollector(prometheus_client.registry.Collector):

    def collect(self):
        value, timestamp = get_data()
        gauge_with_custom_timestamp = GaugeMetricFamily(
            'my_metric_name', 'my_metric_description.'
        )
        gauge_with_custom_timestamp.add_metric([], value, timestamp)
        yield gauge_with_custom_timestamp
```

For reference, here's [my air quality monitoring code](https://github.com/dasl-/pitools/blob/e144fc6e9e92bc908eb30556e3575c4f520f0fa5/sensors/measure_city_data) before I started associating timestamps with the data. I was [using](https://github.com/dasl-/pitools/blob/e144fc6e9e92bc908eb30556e3575c4f520f0fa5/sensors/measure_city_data#L20) the gauge's `set` method and unable to pass a custom timestamp. This code would write a `.prom` file that looked like:
```
# HELP city_pm25 Concentration of PM 2.5 in local city. Units: μg/m^3.
# TYPE city_pm25 gauge
city_pm25 6.51
```

And here's [my code](https://github.com/dasl-/pitools/blob/873ff1eac2dd2bbf3e7521c42a3bf2460cb0b6ad/sensors/measure_city_data) after I started associating timestamps with the data via a [custom collector](https://github.com/dasl-/pitools/blob/873ff1eac2dd2bbf3e7521c42a3bf2460cb0b6ad/sensors/measure_city_data#L66). This code writes a `.prom` file that looks like:
```
# HELP city_pm25 Concentration of PM 2.5 in local city. Units: μg/m^3.
# TYPE city_pm25 gauge
city_pm25 6.51 1720375200000
```

# Custom exporters

Problem solved, right? Wrong. I had been using the node_exporter's [textfile collector](https://github.com/prometheus/node_exporter?tab=readme-ov-file#textfile-collector) to export my data. But the textfile collector does not support custom timestamps:

> Note: Timestamps are not supported.

You'll see something like this in logs if you try to use the textfile collector with a `*.prom` file that has timestamps:

```
NODE_EXPORTER[669]: ts=2024-07-06T11:16:03.618Z caller=textfile.go:227 level=error collector=textfile msg="failed to collect textfile data" file=city_data.prom err="textfile \"/tmp/city_data.prom\" contains unsupported client-side timestamps, skipping entire file"
```

Here's the [node_exporter code](https://github.com/prometheus/node_exporter/blob/4cc1c177d05e80176f26fe1ca2a1f193c03c67a0/collector/textfile.go#L299-L301) that explicitly disallows timestamps.

I discovered on the [Prometheus mailing list](https://groups.google.com/g/prometheus-users/c/Zg1EJltYwp0/m/XrZFtClRBQAJ) that a [custom exporter](https://prometheus.io/docs/instrumenting/writing_exporters/) can solve the problem. It's easy enough to write an exporter - the basic idea is that we need to expose the timestamped metrics in the `*.prom` file via a web server such that Prometheus can scrape them. I wrote a custom exporter in Python, using Python's built in [HTTP server](https://docs.python.org/3/library/http.server.html#http.server.ThreadingHTTPServer). Here's [the code](https://github.com/dasl-/pitools/blob/873ff1eac2dd2bbf3e7521c42a3bf2460cb0b6ad/observability/timestamped_exporter) - it exposes the metrics at [`http://<hostname>:9101/metrics`](https://github.com/dasl-/pitools/blob/873ff1eac2dd2bbf3e7521c42a3bf2460cb0b6ad/observability/timestamped_exporter#L79).

The bulk of the work happens in the [`do_GET`](https://github.com/dasl-/pitools/blob/873ff1eac2dd2bbf3e7521c42a3bf2460cb0b6ad/observability/timestamped_exporter#L20C9-L20C15) method. Here's a simplified version of the code:
```python
def do_GET(self):
    content = ''
    prom_dir = '/prom_dir'

    # read all *.prom files in the directory and concatenate their contents
    for file in os.listdir(prom_dir):
        if file.endswith(".prom"):
            path = os.path.join(prom_dir, file)
            content += open(path, 'r').read()

    # serve the data that was read.
    self.send_response(200)
    self.send_header("Content-Type", "text/plain; charset=utf-8")
    self.end_headers()
    self.wfile.write(bytes(content, 'utf-8'))
```

# Updating Prometheus config scrape targets

After creating the custom exporter and exposing the timestamped metrics on port 9101, we need to update the scrape targets in Prometheus config with the address of the new exporter. Here's what [my config](https://gist.github.com/dasl-/c208118d01d4d6b5e826e5fdabaf583d#file-prometheus-yaml-L18) looked like.

# Updating Prometheus config: `out_of_order_time_window`

Problem solved, right? Wrong again. Checking your Prometheus logs, you may see errors like this:
```
PROMETHEUS[1733468]: ts=2024-07-06T13:12:40.583Z caller=scrape.go:1729 level=warn component="scrape manager" scrape_pool=node target=http://study:9101/metrics msg="Error on ingesting samples that are too old or are too far into the future" num_dropped=1
```

I believe if the timestamped metrics you are attempting to ingest have timestamps that are older than approximately 1 hour, you may encounter this error. Prometheus has an experimental feature that solves this problem: [`out_of_order_time_window`](https://prometheus.io/docs/prometheus/2.53/configuration/configuration/#tsdb). See also this [blog post](https://promlabs.com/blog/2022/10/05/whats-new-in-prometheus-2-39/#experimental-out-of-order-ingestion) announcing the feature. Since the air quality data I need to ingest is at most 2 - 4 hours delayed, it would be sufficient to use a value of anything greater than `4h`. I decided to use a value of `1d` just to be conservative. Here's what [my config](https://gist.github.com/dasl-/eb28a9f692fdc76dc2c8033b40557616#file-prometheus-yaml-L44) looked like.

I can now see the NYC air quality data in grafana:
![grafana panel showing air quality data](/assets/posts/2024-07-07-prometheus-timestamps/grafana.png)

# Conclusion

This was a lot harder than I anticipated. We can't use the out-of-the-box features to accomplish our goal - we instead had to write a lot of custom code. Prometheus is quite opinionated, and the maintainers seem to think that there are few valid use cases for setting custom timestamps on metrics.
