---
title: "Quick Start"
weight: 1
---

The examples on this page are written to work with any OpenTelemetry-compatible Tracer,
but as a quick way to get started you can use the [Jaeger all-in-one image](https://www.jaegertracing.io/docs/1.21/opentelemetry/) running locally via Docker:

`$ docker run -d -p 4317:4317 -p 16686:16686 jaegertracing/opentelemetry-all-in-one:latest`

(This image is experimental; for production you can run
the [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/) writing to a Jaeger back-end, 
or use any other OpenTelemetry-compatible Tracer.)

## Setting up your tracer

```golang
import (
    "context"

    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/otlp"
    "go.opentelemetry.io/otel/exporters/otlp/otlpgrpc"
    "go.opentelemetry.io/otel/sdk/resource"
    tracesdk "go.opentelemetry.io/otel/sdk/trace"
    "go.opentelemetry.io/otel/semconv"
)

func setupTracing(ctx context.Context, serviceName, collectorAddr string) (tracesdk.SpanExporter, error) {
    exporter, err := otlp.NewExporter(ctx,
        otlpgrpc.NewDriver(
            otlpgrpc.WithEndpoint(collectorAddr),
            otlpgrpc.WithInsecure(),
        ))
    if err != nil {
        return nil, err
    }
    res := resource.NewWithAttributes(
        semconv.ServiceNameKey.String(serviceName),
    )
    tp := tracesdk.NewTracerProvider(
        tracesdk.WithSyncer(exporter),
        tracesdk.WithResource(res),
    )
    otel.SetTracerProvider(tp)

    return exporter, err
}

...

func main() {
    ctx := context.Background()
    exporter, err := setupTracing(ctx, "my-service", "127.0.0.1:4317")
    ...
    defer exporter.Shutdown(ctx)
    ...
}
```

## Start a Trace

```golang
import (
	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/trace"
)

...

tracer := otel.Tracer("component_name")
ctx, span := tracer.Start(ctx, "say-hello", trace.WithNewRoot())
println(helloStr)
span.End()
```

## Create a Child Span

```golang
import (
	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/trace"
)

...

tracer := otel.Tracer("component_name")

ctx, parentSpan := tracer.Start(ctx, "parent")
defer parentSpan.End()

...

// Create a Child Span - OpenTelemetry finds the parent from the Context.
ctx, childSpan := tracer.Start(ctx, "child")
defer childSpan.End()
```

## Make an HTTP request

To get traces across service boundaries, we propagate [span context](https://github.com/open-telemetry/opentelemetry-specification/blob/main/specification/trace/api.md#spancontext) by injecting the context into http headers. Once the downstream service receives the http request, it must extract the context and continue the trace. (The code example doesn't handle errors correctly, please don't do this in production code; this is just an example)

For a higher-level wrapper see [package otelhttp](https://pkg.go.dev/go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp)

The upstream(client) service:

```golang
import (
    "net/http"

    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/propagation"
    "go.opentelemetry.io/otel/semconv"
    "go.opentelemetry.io/otel/trace"
)

...

// Set up global propagator to W3C Trace Context
otel.SetTextMapPropagator(propagation.TraceContext{})

tracer := otel.Tracer("client")

ctx, clientSpan := tracer.Start(ctx, "client",
    // annotate that it's the client span
    trace.WithSpanKind(trace.SpanKindClient))
defer clientSpan.End()

url := "http://localhost:8082/publish"
req, _ := http.NewRequest("GET", url, nil)

// Additional HTTP tags are useful for debugging purposes.
clientSpan.SetAttributes(
    semconv.HTTPURLKey.String(url),
    semconv.HTTPMethodKey.String("GET"),
)

// Inject the client span context into the headers
otel.GetTextMapPropagator().Inject(ctx, propagation.HeaderCarrier(req.Header))
resp, _ := http.DefaultClient.Do(req)
```

The downstream(server) service:

```golang
import (
    "log"
    "net/http"

    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/propagation"
    "go.opentelemetry.io/otel/trace"
)

func main() {

    // Tracer initialization, etc.

    ...

    // Set up global propagator to W3C Trace Context
	otel.SetTextMapPropagator(propagation.TraceContext{})

	tracer := otel.Tracer("server")

	http.HandleFunc("/publish", func(w http.ResponseWriter, r *http.Request) {
		// Extract the context from the headers
		spanCtx := otel.GetTextMapPropagator().Extract(ctx, propagation.HeaderCarrier(r.Header))
		_, serverSpan := tracer.Start(spanCtx, "server", trace.WithSpanKind(trace.SpanKindServer))
		defer serverSpan.End()
	})

    log.Fatal(http.ListenAndServe(":8082", nil))
}
```

## View your trace

If you have Jaeger all-in-one running, you can view your trace at `localhost:16686`.

## Link to GO walkthroughs / tutorials

* TBD
