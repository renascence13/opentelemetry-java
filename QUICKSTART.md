# OpenTelemetry QuickStart

<!-- Re-generate TOC with `markdown-toc --no-first-h1 -i` -->

<!-- toc -->

- [Configuration](#configuration)
    + [Sampler](#sampler)
    + [Span Processor](#span-processor)
    + [Exporter](#exporter)
- [Tracing](#tracing)
  * [Create basic Span](#create-basic-span)
  * [Create nested Spans](#create-nested-spans)
  * [Span Attributes](#span-attributes)
  * [Create Spans with events](#create-spans-with-events)
  * [Create Spans with links](#create-spans-with-links)
  * [Context Propagation](#context-propagation)
- [Metrics](#metrics)

<!-- tocstop -->

OpenTelemetry can be used to instrument code for collecting telemetry data.
For more details, check out the [OpenTelemetry Website].

In the following examples, we demonstrate how to configure the OpenTelemetry SDK, create spans and record metrics through the OpenTelemetry API.

[OpenTelemetry Website]: https://opentelemetry.io/

# Configuration

**Libraries** that want to export telemetry data using OpenTelemetry only need a dependency on the `opentelemetry-api` package
and should never configure OpenTelemetry themselves. The configuration must be provided by **Applications** which should also depend on the 
`opentelemetry-sdk` package, or any other implementation of the OpenTelemetry API. This way, libraries will obtain a real tracer
implementation only if the user application is configured for it. For more details, check out the [Library Guidelines].
The configuration examples reported in this document only apply to the SDK provided by `opentelemetry-sdk`. 
Other implementation of the API might provide different configuration mechanisms.

[Library Guidelines]: https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/library-guidelines.md

The application has to install a span processor with an exporter and may customize the behavior of the OpenTelemetry SDK.

For example, a basic configuration instantiates the SDK tracer registry and sets to export the traces to a logging stream.

```java
// Get the tracer
TracerSdkRegistry tracerRegistry = OpenTelemetrySdk.getTracerRegistry();

// Set to export the traces to a logging stream
tracerRegistry.addSpanProcessor(
    SimpleSpansProcessor.newBuilder(
        new LoggingExporter()
    ).build());
```

### Sampler

It is not always feasible to trace and export every user request in an application.
In order to strike a balance between observability and expenses, traces can be sampled. 

The OpenTelemetry SDK offers three samplers out of the box:
 - [AlwaysOnSampler] which samples every trace regardless of upstream sampling decisions.
 - [AlwaysOffSampler] which doesn't sample any trace, regardless of upstream sampling decisions.
 - [Probability] which samples a configurable percentage of traces, and additionally samples any trace that was sampled upstream.

[AlwaysOnSampler]: https://github.com/open-telemetry/opentelemetry-java/blob/master/sdk/src/main/java/io/opentelemetry/sdk/trace/Samplers.java#L82--L105
[AlwaysOffSampler]:https://github.com/open-telemetry/opentelemetry-java/blob/master/sdk/src/main/java/io/opentelemetry/sdk/trace/Samplers.java#L108--L131
[Probability]:https://github.com/open-telemetry/opentelemetry-java/blob/master/sdk/src/main/java/io/opentelemetry/sdk/trace/Samplers.java#L142--L203

Additional samplers can be provided by implementing the [`io.opentelemetry.sdk.trace.Sampler`] interface.

```java
TraceConfig alwaysOn = TraceConfig.getDefault().toBuilder().setSampler(
        Samplers.alwaysOn()
).build();
TraceConfig alwaysOff = TraceConfig.getDefault().toBuilder().setSampler(
        Samplers.alwaysOff()
).build();
TraceConfig half = TraceConfig.getDefault().toBuilder().setSampler(
        Samplers.probability(0.5)
).build();
// Configure the sampler to use
tracerRegistry.updateActiveTraceConfig(
    half
);
```

[`io.opentelemetry.sdk.trace.Sampler`]: https://github.com/open-telemetry/opentelemetry-java/blob/master/sdk/src/main/java/io/opentelemetry/sdk/trace/Sampler.java

### Span Processor

Different Span processors are offered by OpenTelemetry. 
The `SimpleSpanProcessor` immediately forwards ended spans to the exporter, while the `BatchSpansProcessor` batches them and sends them in bulk.
Multiple Span processors can be configured to be active at the same time using the `MultiSpanProcessor`.

```java
tracerRegistry.addSpanProcessor(
    SimpleSpansProcessor.newBuilder(new LoggingExporter()).build()
);
tracerRegistry.addSpanProcessor(
    BatchSpansProcessor.newBuilder(new LoggingExporter()).build()
);
tracerRegistry.addSpanProcessor(MultiSpanProcessor.create(Arrays.asList(
            SimpleSpansProcessor.newBuilder(new LoggingExporter()).build(),
            BatchSpansProcessor.newBuilder(new LoggingExporter()).build()
)));
```

### Exporter

Span processors are initialized with an exporter which is responsible for sending the telemetry data a particular backend.
OpenTelemetry offers four exporters out of the box:
- In-Memory Exporter: keeps the data in memory, useful for debugging.
- Jaeger Exporter: prepares and sends the collected telemetry data to a Jaeger backend via gRPC.
- Logging Exporter: saves the telemetry data into log streams.
- OpenTelemetry Exporter: sends the data to the [OpenTelemetry Collector] (not yet implemented).

Other exporters can be found in the [OpenTelemetry Registry].

[OpenTelemetry Collector]: https://github.com/open-telemetry/opentelemetry-collector
[OpenTelemetry Registry]: https://opentelemetry.io/registry/?s=exporter

```java
tracerRegistry.addSpanProcessor(SimpleSpansProcessor.newBuilder(
    InMemorySpanExporter.create()
).build());
tracerRegistry.addSpanProcessor(SimpleSpansProcessor.newBuilder(
    new LoggingExporter()
).build());

ManagedChannel jaegerChannel = ManagedChannelBuilder.forAddress([ip:String], [port:int]).usePlaintext().build();
JaegerGrpcSpanExporter jaegerExporter = JaegerGrpcSpanExporter.newBuilder()
    .setServiceName("example").setChannel(jaegerChannel).setDeadline(30000)
    .build();
tracerRegistry.addSpanProcessor(BatchSpansProcessor.newBuilder(
    jaegerExporter
).build());
```

# Tracing

In the following, we present how to trace code using the OpenTelemetry API.
**Note:** Methods of the OpenTelemetry SDK should never be called.
 
First, a `Tracer` must be acquired, which is responsible for creating spans and interacting with the [Context](#context-propagation).
A tracer is acquired by using the OpenTelemetry API specifying the name and version of the library 
instrumenting the instrumented library or application to be monitored. 
More information is available in the specification chapter [Obtaining a Tracer].

```java
Tracer tracer = OpenTelemetry.getTracerRegistry().get("instrumentation-library-name","semver:1.0.0");
```

[Obtaining a Tracer]: https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/api-tracing.md#obtaining-a-tracer

## Create basic Span
To create a basic span, you only need to specify the name of the span.
The start and end time of the span is automatically set by the OpenTelemetry SDK.
```java
Span span = tracer.spanBuilder("SpanName").startSpan();
// your use case
...
span.end();
```

## Create nested Spans

Most of the time, we want to correlate spans for nested operations.
OpenTelemetry supports tracing within processes and across remote processes.
For more details how to share context between remote processes, see [Context Propagation](#context-propagation).

For a method `a` calling a method `b`, the spans could be manually linked in the following way:
```java
void a() {
  Span parentSpan = tracer.spanBuilder("a")
        .startSpan();
  b(parentSpan);
  parentSpan.end();
}
void b(Span parentSpan) {
  Span childSpan = tracer.spanBuilder("b")
        .setParent(parentSpan)
        .startSpan();
  // do stuff
  childSpan.end();
}
```
The OpenTelemetry API offers also an automated way to propagate the `parentSpan`:
```java
void a() {
  Span parentSpan = tracer.spanBuilder("a").startSpan();
  try(Scope scope = tracer.withSpan(parentSpan)){
    b();
  } finally {
    parentSpan.end();
  }
}
void b() {
  Span childSpan = tracer.spanBuilder("b")
     // NOTE: setParent(parentSpan) is not required anymore, 
     // `tracer.getCurrentSpan()` is automatically added as parent
    .startSpan();
  // do stuff
  childSpan.end();
}
``` 

To link spans from remote processes, it is sufficient to set the [Remote Context](#context-propagation) as parent. 
```java
Span childRemoteParent = tracer.spanBuilder("Child").setParent(remoteContext).startSpan();
```

## Span Attributes
In OpenTelemetry spans can be created freely and it's up to the implementor to annotate them with attributes specific to the represented operation. 
Attributes provide additional context on a span about the specific operation it tracks, such as results or operation properties.
```java
Span span = tracer.spanBuilder("/resource/path").setSpanKind(Span.Kind.CLIENT).startSpan();
span.setAttribute("http.method", "GET");
span.setAttribute("http.url", url.toString());
```

Some of these operations represent calls that use well-known protocols like HTTP or database calls. 
For these, OpenTelemetry requires specific attributes to be set. The full attribute list is available in the [Semantic Conventions] in the cross-language specification.

[Semantic Conventions]: https://github.com/open-telemetry/opentelemetry-specification/blob/master/specification/data-semantic-conventions.md

## Create Spans with events

Spans can be annotated with named events that can carry zero or more [Span Attributes](#span-attributes),
each of which is itself a key:value map paired automatically with a timestamp.

```java
span.addEvent("Init");
...
span.addEvent("End");
```
```java
Map<String, AttributeValue> eventAttributes = new HashMap<>();
eventAttributes.put("key", AttributeValue.stringAttributeValue("value"));
eventAttributes.put("result", AttributeValue.longAttributeValue(0L));

span.addEvent("End Computation", eventAttributes);
```

## Create Spans with links
A Span may be linked to zero or more other Spans that are causally related.
Links can be used to represent batched operations where a Span was initiated by multiple initiating Spans, each representing a single incoming item being processed in the batch.

```java
Link link1 = SpanData.Link.create(parentSpan1.getContext());
Link link2 = SpanData.Link.create(parentSpan2.getContext());
Span child = tracer.spanBuilder("childWithLink")
        .addLink(link1)
        .addLink(link2)
        .addLink(parentSpan3.getContext())
        .addLink(remoteContext)
    .startSpan();
```

For more details how to read context from remote processes, see [Context Propagation](#context-propagation).

## Context Propagation

OpenTelemetry provides a text-based approach to propagate context to remote services using the [W3C Trace Context](https://www.w3.org/TR/trace-context/) HTTP headers.
The following presents an example of an outgoing HTTP request using `HttpURLConnection`.
 
```java
// Tell OpenTelemetry to inject the context in the HTTP headers
HttpTextFormat.Setter<HttpURLConnection> setter =
  new HttpTextFormat.Setter<HttpURLConnection>() {
    @Override
    public void put(HttpURLConnection carrier, String key, String value) {
        // Insert the context as Header
        carrier.setRequestProperty(key, value);
    }
};

URL url = new URL("http://127.0.0.1:8080/resource");
Span outGoing = tracer.spanBuilder("/resource").setSpanKind(Span.Kind.CLIENT).startSpan();
// Semantic Convention
outGoing.setAttribute("http.method", "GET");
outGoing.setAttribute("http.url", url.toString());
HttpURLConnection transportLayer = (HttpURLConnection) url.openConnection();
// Inject the request with the context
tracer.getHttpTextFormat().inject(outGoing.getContext(), transportLayer, setter);
// Make outgoing call
...
```

Similarly, the text-based approach can be used to read the W3C Trace Context from incoming requests.
The following presents an example of processing an incoming HTTP request using `HttpExchange`.
```java
HttpTextFormat.Getter<HttpExchange> getter =
  new HttpTextFormat.Getter<HttpExchange>() {
    @Override
    public String get(HttpExchange carrier, String key) {
      if (carrier.getRequestHeaders().containsKey(key)) {
        return carrier.getRequestHeaders().get(key).get(0);
      }
      return null;
    }
};
...
public void handle(HttpExchange he) {
    // Extract the context from the request
    SpanContext ctx = tracer.getHttpTextFormat().extract(he, getter);
    Span serverSpan = tracer.spanBuilder("/resource").setSpanKind(Span.Kind.SERVER)
        .setParent(ctx)
        .startSpan();
    // Add the attributes defined in the Semantic Conventions
    serverSpan.setAttribute("http.method", "GET");
    serverSpan.setAttribute("http.scheme", "http");
    serverSpan.setAttribute("http.host", "localhost:8080");
    serverSpan.setAttribute("http.target", "/resource");
    // Serve the request
    ...
    serverSpan.end();
}
```

# Metrics

