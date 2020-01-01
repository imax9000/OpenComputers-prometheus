# Prometheus metric library for OpenComputers

This is a Lua library that can be used with OpenComputers in Minecraft to keep
track of metrics and push them to [Prometheus](https://prometheus.io)
Pushgateway.

WARNING: it's still in "I just got it working" state, so there are some sharp
edges.

## Installation

```
oppm register https://github.com/gelraen/OpenComputers-prometheus
oppm install prometheus
```

## Quick start guide

```lua
-- Initialize with pushgateway endpoint and a set of grouping labels:
local prom = prometheus.init("http://localhost:9091/metrics", {owner="someone"})

--[[ Callback that is called once with a single argument - `Prometheus` object
that will contain a bunch of metrics.

Return value will be passed as is to `update` callback.
]]
local init = function(prom)
  return {
    count=prom:counter("test_counter", "Test counter", {"foo"}),
  }
end

--[[ Callback that will be called on timer to update metric values.

The only argument is the value returned by `init` callback.
]]
local update = function(metrics)
  metrics.count:inc(1, {"a"})
  metrics.count:inc(2, {"b"})
end

--[[ Every 60 seconds push new set of metric values. Labels passed in second
argument are merged together with labels passed to prometheus.init() to create
a set of grouping labels. This set is the going to be used in Pushgateway
endpoint URL.
]]
prom:timer(60, {job="foobar"}, init, update)
```

TODO: update stuff below.

To track request latency broken down by server name and request count
broken down by server name and status, add the following to the `http` section
of `nginx.conf`:

```
lua_shared_dict prometheus_metrics 10M;
lua_package_path "/path/to/nginx-lua-prometheus/?.lua";
init_by_lua '
  prometheus = require("prometheus").init("prometheus_metrics")
  metric_requests = prometheus:counter(
    "nginx_http_requests_total", "Number of HTTP requests", {"host", "status"})
  metric_latency = prometheus:histogram(
    "nginx_http_request_duration_seconds", "HTTP request latency", {"host"})
  metric_connections = prometheus:gauge(
    "nginx_http_connections", "Number of HTTP connections", {"state"})
';
log_by_lua '
  metric_requests:inc(1, {ngx.var.server_name, ngx.var.status})
  metric_latency:observe(tonumber(ngx.var.request_time), {ngx.var.server_name})
';
```

This:
* configures a shared dictionary for your metrics called `prometheus_metrics`
  with a 10MB size limit;
* registers a counter called `nginx_http_requests_total` with two labels:
  `host` and `status`;
* registers a histogram called `nginx_http_request_duration_seconds` with one
  label `host`;
* registers a gauge called `nginx_http_connections` with one label `state`;
* on each HTTP request measures its latency, recording it in the histogram and
  increments the counter, setting current server name as the `host` label and
  HTTP status code as the `status` label.

Last step is to configure a separate server that will expose the metrics.
Please make sure to only make it reachable from your Prometheus server:

```
server {
  listen 9145;
  allow 192.168.0.0/16;
  deny all;
  location /metrics {
    content_by_lua '
      metric_connections:set(ngx.var.connections_reading, {"reading"})
      metric_connections:set(ngx.var.connections_waiting, {"waiting"})
      metric_connections:set(ngx.var.connections_writing, {"writing"})
      prometheus:collect()
    ';
  }
}
```

Metrics will be available at `http://your.nginx:9145/metrics`. Note that the
gauge metric in this example contains values obtained from nginx global state,
so they get set immediately before metrics are returned to the client.

If you experience problems indicating that nginx doesn't know how to interpret
lua-commands and you use an external module for nginx-lua-support (e.g. the
`libnginx-mod-http-lua` package on Debian) try adding

    load_module modules/ndk_http_module.so;
    load_module modules/ngx_http_lua_module.so;

to the beginning of `nginx.conf` to ensure the modules are loaded.


## API reference

### init()

**syntax:** require("prometheus").init(*dict_name*, [*prefix*])

Initializes the module. This should be called once from the
[init_by_lua](https://github.com/openresty/lua-nginx-module#init_by_lua)
section in nginx configuration.

* `dict_name` is the name of the nginx shared dictionary which will be used to
  store all metrics. Defaults to `prometheus_metrics` if not specified.
* `prefix` is an optional string which will be prepended to metric names on output


Returns a `prometheus` object that should be used to register metrics.

Example:
```
init_by_lua '
  prometheus = require("prometheus").init("prometheus_metrics")
';
```

### prometheus:counter()

**syntax:** prometheus:counter(*name*, *description*, *label_names*)

Registers a counter. Should be called once from the
[init_by_lua](https://github.com/openresty/lua-nginx-module#init_by_lua)
section.

* `name` is the name of the metric.
* `description` is the text description that will be presented to Prometheus
  along with the metric. Optional (pass `nil` if you still need to define
  label names).
* `label_names` is an array of label names for the metric. Optional.

[Naming section](https://prometheus.io/docs/practices/naming/) of Prometheus
documentation provides good guidelines on choosing metric and label names.

Returns a `counter` object that can later be incremented.

Example:
```
init_by_lua '
  prometheus = require("prometheus").init("prometheus_metrics")
  metric_bytes = prometheus:counter(
    "nginx_http_request_size_bytes", "Total size of incoming requests")
  metric_requests = prometheus:counter(
    "nginx_http_requests_total", "Number of HTTP requests", {"host", "status"})
';
```

### prometheus:gauge()

**syntax:** prometheus:gauge(*name*, *description*, *label_names*)

Registers a gauge. Should be called once from the
[init_by_lua](https://github.com/openresty/lua-nginx-module#init_by_lua)
section.

* `name` is the name of the metric.
* `description` is the text description that will be presented to Prometheus
  along with the metric. Optional (pass `nil` if you still need to define
  label names).
* `label_names` is an array of label names for the metric. Optional.

Returns a `gauge` object that can later be set.

Example:
```
init_by_lua '
  prometheus = require("prometheus").init("prometheus_metrics")
  metric_connections = prometheus:gauge(
    "nginx_http_connections", "Number of HTTP connections", {"state"})
';
```

### prometheus:histogram()

**syntax:** prometheus:histogram(*name*, *description*, *label_names*,
  *buckets*)

Registers a histogram. Should be called once from the
[init_by_lua](https://github.com/openresty/lua-nginx-module#init_by_lua)
section.

* `name` is the name of the metric.
* `description` is the text description. Optional.
* `label_names` is an array of label names for the metric. Optional.
* `buckets` is an array of numbers defining bucket boundaries. Optional,
  defaults to 20 latency buckets covering a range from 5ms to 10s (in seconds).

Returns a `histogram` object that can later be used to record samples.

Example:
```
init_by_lua '
  prometheus = require("prometheus").init("prometheus_metrics")
  metric_latency = prometheus:histogram(
    "nginx_http_request_duration_seconds", "HTTP request latency", {"host"})
  metric_response_sizes = prometheus:histogram(
    "nginx_http_response_size_bytes", "Size of HTTP responses", nil,
    {10,100,1000,10000,100000,1000000})
';
```

### prometheus:collect()

**syntax:** prometheus:collect()

Presents all metrics in a text format compatible with Prometheus. This should be
called in
[content_by_lua](https://github.com/openresty/lua-nginx-module#content_by_lua)
to expose the metrics on a separate HTTP page.

Example:
```
location /metrics {
  content_by_lua 'prometheus:collect()';
  allow 192.168.0.0/16;
  deny all;
}
```

### prometheus:metric_data()

**syntax:** prometheus:metric_data()

Returns metric data as an array of strings.

### counter:inc()

**syntax:** counter:inc(*value*, *label_values*)

Increments a previously registered counter. This is usually called from
[log_by_lua](https://github.com/openresty/lua-nginx-module#log_by_lua)
globally or per server/location.

* `value` is a value that should be added to the counter. Defaults to 1.
* `label_values` is an array of label values.

The number of label values should match the number of label names defined when
the counter was registered using `prometheus:counter()`. No label values should
be provided for counters with no labels. Non-printable characters will be
stripped from label values.

Example:
```
log_by_lua '
  metric_bytes:inc(tonumber(ngx.var.request_length))
  metric_requests:inc(1, {ngx.var.server_name, ngx.var.status})
';
```

### counter:del()

**syntax:** counter:del(*label_values*)

Delete a previously registered counter. This is usually called when you don't 
need to observe such counter (or a metric with specific label values in this 
counter) any more. If this counter has labels, you have to pass `label_values` 
to delete the specific metric of this counter. If you want to delete all the 
metrics of a counter with labels, you should call `Counter:reset()`.

* `label_values` is an array of label values.

The number of label values should match the number of label names defined when
the counter was registered using `prometheus:counter()`. No label values should
be provided for counters with no labels. Non-printable characters will be
stripped from label values.

### counter:reset()

**syntax:** counter:reset()

Delete all metrics for a previously registered counter. If this counter have no 
labels, it is just the same as `Counter:del()` function. If this counter have labels, 
it will delete all the metrics with different label values.

### gauge:set()

**syntax:** gauge:set(*value*, *label_values*)

Sets the current value of a previously registered gauge. This could be called
from [log_by_lua](https://github.com/openresty/lua-nginx-module#log_by_lua)
globally or per server/location to modify a gauge on each request, or from
[content_by_lua](https://github.com/openresty/lua-nginx-module#content_by_lua)
just before `prometheus::collect()` to return a real-time value.

* `value` is a value that the gauge should be set to. Required.
* `label_values` is an array of label values.

### gauge:inc()

**syntax:** gauge:inc(*value*, *label_values*)

Increments or decrements a previously registered gauge. This is usually called 
when you want to observe the real-time value of a metric that can both be 
increased and decreased.

* `value` is a value that should be added to the gauge. It could be a negative 
value when you need to decrease the value of the gauge. Defaults to 1.
* `label_values` is an array of label values.

The number of label values should match the number of label names defined when
the gauge was registered using `prometheus:gauge()`. No label values should
be provided for gauges with no labels. Non-printable characters will be
stripped from label values.

### gauge:del()

**syntax:** gauge:del(*label_values*)

Delete a previously registered gauge. This is usually called when you don't 
need to observe such gauge (or a metric with specific label values in this 
gauge) any more. If this gauge has labels, you have to pass `label_values` 
to delete the specific metric of this gauge. If you want to delete all the 
metrics of a gauge with labels, you should call `Gauge:reset()`.

* `label_values` is an array of label values.

The number of label values should match the number of label names defined when
the gauge was registered using `prometheus:gauge()`. No label values should
be provided for gauges with no labels. Non-printable characters will be
stripped from label values.

### gauge:reset()

**syntax:** gauge:reset()

Delete all metrics for a previously registered gauge. If this gauge have no 
labels, it is just the same as `Gauge:del()` function. If this gauge have labels, 
it will delete all the metrics with different label values.

### histogram:observe()

**syntax:** histogram:observe(*value*, *label_values*)

Records a value in a previously registered histogram. Usually called from
[log_by_lua](https://github.com/openresty/lua-nginx-module#log_by_lua)
globally or per server/location.

* `value` is a value that should be recorded. Required.
* `label_values` is an array of label values.

Example:
```
log_by_lua '
  metric_latency:observe(tonumber(ngx.var.request_time), {ngx.var.server_name})
  metric_response_sizes:observe(tonumber(ngx.var.bytes_sent))
';
```

### Built-in metrics

The module increments the `metric_errors_total` metric if it encounters
an error (for example, when `lua_shared_dict` becomes full). You might want
to configure an alert on that metric.


## Credits

- Created and maintained by Anton Tolchanov (@knyar)
- Metrix prefix support contributed by david birdsong (@davidbirdsong)
- Gauge support contributed by Cosmo Petrich (@cosmopetrich)
- Adapted to OpenComputers by Max Ignatenko (@gelraen)

## License

Licensed under MIT license.
