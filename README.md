# OpenTracing Java Bridge for Zipkin

This library is a Java bridge between the [Brave/Zipkin Api](https://github.com/openzipkin/brave/tree/master/brave#brave-api-v4) and OpenTracing. It allows its users to write portable (in the [OpenTracing](http://opentracing.io) sense) instrumentation that's translated into Brave instrumentation transparently.

## Required Reading

In order to understand OpenTracing Api, one must first be familiar with the [OpenTracing project](http://opentracing.io) and [terminology](http://opentracing.io/spec/) more generally.

To understand how Zipkin and Brave work, you can look at [Zipkin Architecture](http://zipkin.io/pages/architecture.html) and [Brave Api](https://github.com/openzipkin/brave/tree/master/brave#brave-api-v4) documentation.

## Setup

Firstly, you need a Tracer, configured to [report to Zipkin](https://github.com/openzipkin/zipkin-reporter-java).

```java
// Configure a reporter, which controls how often spans are sent
//   (the dependency is io.zipkin.reporter:zipkin-zipkin-sender-okhttp3)
sender = OkHttpSender.create("http://127.0.0.1:9411/api/v1/spans");
reporter = AsyncReporter.builder(sender).build();

// Now, create a Brave tracer with the service name you want to see in Zipkin.
//   (the dependency is io.zipkin.brave:brave)
braveTracer = Tracer.newBuilder()
                    .localServiceName("my-service")
                    .reporter(reporter)
                    .build();

// Finally, wrap this with the OpenTracing Api
tracer = BraveTracer.wrap(braveTracer);

// You can later unwrap the underlying Brave Api as needed
braveTracer = tracer.unwrap();
```

## Usage

Some example code demonstrating how the OpenTracing Api is to be used.

Code in the first process…

```java
    // start a span
    try ( Span span0 = tracer.buildSpan("span-0")
            .withTag("description", "top level initial span in the original process")
            .start() ) {

        // do something

        // start another span
        try ( Span span1 = tracer.buildSpan("span-1")
                .asChildOf(span0)
                .withTag("description", "the first inner span in the original process")
                .start() ) {

            // do something

            // start another span
            try ( Span span2 = tracer.buildSpan("span-2")
                    .asChildOf(span1)
                    .withTag("description", "the second inner span in the original process")
                    .start() ) {

                // do something

                // cross process boundary
                Map<String,String> map = new HashMap<>();
                tracer.inject(span2.context(), Format.Builtin.HTTP_HEADERS, new TextMapInjectAdapter(map));

                // request.addHeaders(map);
                // request.doGet();
            }
        }
    }
```

Code in the second process

```java
    // Map<String,String> map = request.getHeaders();

    try ( Span span3 = tracer.buildSpan("span-3")
            .asChildOf(tracer.extract(Format.Builtin.HTTP_HEADERS, new TextMapExtractAdapter(map)))
            .withTag("description", "the third inner span in the second process")
            .start() ) {

        // do something

        // start another span
        try ( Span span4 = tracer.buildSpan("span-4")
                .asChildOf(span3)
                .withTag("description", "the fourth inner span in the second process")
                .start() ) {

            // do something

            // cross process boundary
            map = new HashMap<>();
            tracer.inject(span4.context(), Format.Builtin.HTTP_HEADERS, new TextMapInjectAdapter(map));

            // request.addHeaders(map);
            // request.doGet();
        }
    }
```

This code repeats from process to process, as far through the stack as required.
