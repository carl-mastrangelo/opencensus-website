---
title: "Server"
date: 2018-11-27T01:12:31-07:00
draft: false
aliases: [/integrations/http/go/net_http/server]
weight: 2
logo: /images/gopher.png
---

- [Introduction](#introduction)
- [Traces](#traces)
- [Metrics](#metrics)
- [End-to-end example](#end-to-end-example)
- [Viewing traces](#view-traces)
- [Viewing metrics](#view-metrics)
- [References](#references)

### Introduction

OpenCensus provides a package [go.opencensus.io/plugin/ochttp](https://godoc.org/go.opencensus.io/plugin/ochttp) which has a custom HTTP handler [ochttp.Handler](https://godoc.org/go.opencensus.io/plugin/ochttp#Handler) that can wrap your [HTTP handlers](https://golang.org/pkg/net/http/#Handler) while running your [HTTP servers](https://golang.org/pkg/net/http/#Server). For example

```go
package main

import (
	"log"
	"net/http"

	"go.opencensus.io/plugin/ochttp"
)

func main() {
	originalHandler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.Write([]byte("Hello, World!"))
	})
	och := &ochttp.Handler{
		Handler: originalHandler, // The handler you'd have used originally
	}

	// Now use the instrumnted handler
	if err := http.ListenAndServe(":9999", och); err != nil {
		log.Fatalf("Failed to run the server: %v", err)
	}
}
```

### Traces
The instrumented handler creates a trace automatically for each request that is received. We shall need a [Trace exporter](/exporters/supported-exporters/go) to examine the traces.
If the request that it received contains a trace with the same context propagation format, it will continue the trace as a root, for example

![](/images/ochttp-server-trace-non-link.png)

However, if the handler is started with the field [IsPublicEndpoint](https://godoc.org/go.opencensus.io/plugin/ochttp#Handler.IsPublicEndpoint), then the span of the propagated span will be recorded as a [Parent Link](/tracing/span/link/) instead for example

On the server side
![](/images/ochttp-server-trace-link-server.png)

which is the parent link of
![](/images/ochttp-server-trace-link-client.png)


### Metrics
The instrumented handler records metrics automatically for each request that is received. We shall need a [Stats exporter](/exporters/supported-exporters/go) to examine the metrics.

Metrics can be enabled by simply registering the [ochttp.DefaultServerViews](https://godoc.org/go.opencensus.io/plugin/ochttp#DefaultServerViews)
```go
        // In our main, register ochttp Server views
        if err := view.Register(ochttp.DefaultServertViews...); err != nil {
                log.Fatalf("Failed to register server views for HTTP metrics: %v", err)
        }
```

which then provides the following metrics

Metric|Prefix|Description|Tags|Aggregation
---|---|---|---|---
Requests count by method|"opencensus.io/http/server/request_count_by_method"|The number of requests received|"http.method"|Count
Responses count by method|"opencensus.io/http/server/response_count_by_method"|The number of responses sent|"http.method"|Count
Size distribution of HTTP request bodies|"opencensus.io/http/server/request_bytes"|The number of bytes received per request||Distribution
Size distribution of HTTP response bodies|"opencensus.io/http/server/response_bytes"|The number of bytes sent per request||Distribution
Server Latency|"opencensus.io/http/server/latency"|The latency distribution of HTTP responses||Distribution


### End to end example

The following example uses the ochttp.Handler to process HTTP requests that are received every 5 seconds. It replies back with "Hello, World!". It exports traces and metrics to
Trace and Metrics exporters: Zipkin and Prometheus respectively. Of course you can use [any other exporters of your choice](/exporters/supported-exporters/go)

{{% notice tip %}}
For assistance setting up any of the exporters, please refer to:

Exporter|URL
---|---
Prometheus|[Prometheus codelab](/codelabs/prometheus)
Zipkin|[Zipkin codelab](/codelabs/zipkin)
{{% /notice %}}

```go
package main

import (
	"io"
	"io/ioutil"
	"log"
	"net/http"
	"net/http/httptest"
	"time"

	openzipkin "github.com/openzipkin/zipkin-go"
	zipkinHTTP "github.com/openzipkin/zipkin-go/reporter/http"

	"go.opencensus.io/exporter/prometheus"
	"go.opencensus.io/exporter/zipkin"
	"go.opencensus.io/plugin/ochttp"
	"go.opencensus.io/stats/view"
	"go.opencensus.io/trace"
)

func main() {
	// Firstly, we'll register ochttp Server views
	if err := view.Register(ochttp.DefaultServerViews...); err != nil {
		log.Fatalf("Failed to register server views for HTTP metrics: %v", err)
	}

	// Enable observability to extract and examine traces and stats.
	enableObservabilityAndExporters()

	// The handler containing your business logic to process requests.
	originalHandler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		io.Copy(ioutil.Discard, r.Body)
		w.Write([]byte("Hello, World!"))
	})
	och := &ochttp.Handler{
		Handler: originalHandler, // The handler you'd have used originally
	}
	cst := httptest.NewServer(och)
	defer cst.Close()

	client := &http.Client{}
	for {
		req, _ := http.NewRequest("POST", cst.URL, nil)
		res, _ := client.Do(req)
		io.Copy(ioutil.Discard, res.Body)
		<-time.After(5 * time.Second)
	}
}

func enableObservabilityAndExporters() {
	// Stats exporter: Prometheus
	pe, err := prometheus.NewExporter(prometheus.Options{
		Namespace: "ochttp_tutorial",
	})
	if err != nil {
		log.Fatalf("Failed to create the Prometheus stats exporter: %v", err)
	}

	view.RegisterExporter(pe)
	go func() {
		mux := http.NewServeMux()
		mux.Handle("/metrics", pe)
		log.Fatal(http.ListenAndServe(":8888", mux))
	}()

	// Trace exporter: Zipkin
	localEndpoint, _ := openzipkin.NewEndpoint("ochttp_tutorial", "localhost:0")
	reporter := zipkinHTTP.NewReporter("http://localhost:9411/api/v2/spans")
	ze := zipkin.NewExporter(reporter, localEndpoint)
	trace.RegisterExporter(ze)

	// Lastly, let's always sample for the purposes of this demo
	trace.ApplyConfig(trace.Config{DefaultSampler: trace.AlwaysSample()})
}
```

and to run the example

```shell
go run main.go
```

and then start Prometheus with this configuration file `prometheus.yaml`

```yaml
scrape_configs:
  - job_name: 'ochttp_tutorial'

    scrape_interval: 10s

    static_configs:
      - targets: ['localhost:8888']
```


### Viewing traces

* All traces
![](/images/ochttp-server-traces-all.png)

* Single trace
![](/images/ochttp-server-traces-single.png)

* Single trace detail
![](/images/ochttp-server-traces-single-detail.png)

### Viewing metrics

* All metrics
![](/images/ochttp-server-metrics-all.png)

* Response bytes buckets
![](/images/ochttp-server-metrics-response-bytes-bucket.png)

* Request bytes buckets
![](/images/ochttp-server-metrics-request-bytes-bucket.png)

* Response latency buckets
![](/images/ochttp-server-metrics-server-latency-bucket.png)


### References

Resource|URL
---|---
ochttp.Handler GoDoc|https://godoc.org/go.opencensus.io/plugin/ochttp#Handler
ochttp.DefaultServerViews|https://godoc.org/go.opencensus.io/plugin/ochttp#DefaultServerViews
net/http Godoc|https://golang.org/pkg/net/http
