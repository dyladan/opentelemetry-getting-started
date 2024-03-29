# Getting Started with OpenTelemetry JS
This guide will walk you through the setup and configuration process for a tracing backend (in this case [Zipkin](https://zipkin.io), but [Jaeger](https://www.jaegertracing.io) would be simple to use as well), a metrics backend like [Prometheus](https://prometheus.io), and auto-instrumentation of NodeJS.

## Tracing Your Application with OpenTelemetry
This guide assumes you are going to be using Zipkin as your tracing backend, but modifying it for Jaeger should be straightforward.

This guide and the associated example application can be found at <https://github.com/dyladan/opentelemetry-getting-started>. For a complete working example, check out the branch named `traced` [here](https://github.com/dyladan/opentelemetry-getting-started/tree/traced).

### Setting up a Tracing Backend
The first thing we will need before we can start collecting traces is a tracing backend like Zipkin that we can export traces to. If you already have a supported tracing backend (Zipkin or Jaeger), you can skip this step. If not, you will need to run one.

In order to set up Zipkin as quickly as possible, run the latest [Docker Zipkin](https://github.com/openzipkin/docker-zipkin) container, exposing port `9411`. If you can’t run Docker containers, you will need to download and run Zipkin by following the Zipkin [quickstart guide](https://zipkin.io/pages/quickstart.html).

```sh
$ docker run --rm -d -p 9411:9411 --name zipkin openzipkin/zipkin
```

Browse to <http://localhost:9411> to ensure that you can see the Zipkin UI.

![](images/zipkin.png)

### Instrument Your NodeJS Application
This guide uses the example application [here](https://github.com/dyladan/opentelemetry-getting-started), but the steps to instrument your own application should be broadly the same. Here is an overview of what we will be doing.

1. Install the required OpenTelemetry libraries
2. Initialize a global tracer
3. Initialize and register a trace exporter

#### Install the required OpenTelemetry libraries
To create traces on NodeJS, you will need `@opentelemetry/node`, `@opentelemetry/core`, and any plugins required by your application such as gRPC, or HTTP. If you are using the example application, you will need to install `@opentelemetry/plugin-http`.

```sh
$ npm install \
  @opentelemetry/core \
  @opentelemetry/node \
  @opentelemetry/plugin-http
```

#### Initialize a global tracer
All tracing initialization should happen before your application’s code runs. The easiest way to do this is to initialize tracing in a separate file that is required using node’s `-r` option before application code runs.

Create a file named `tracing.js` and add the following code:

```javascript
'use strict';

const opentelemetry = require("@opentelemetry/core")
const { NodeTracer } = require("@opentelemetry/node")

const tracer = new NodeTracer({
	logLevel: opentelemetry.LogLevel.ERROR
});

opentelemetry.initGlobalTracer(tracer);
```

If you run your application now with `node -r ./tracing.js app.js`, your application will create and propagate traces over HTTP. If an already instrumented service that supports [Trace Context](https://www.w3.org/TR/trace-context/) headers calls your application using HTTP, and you call another application using HTTP, the Trace Context headers will be correctly propagated.

If you wish to see a completed trace, however, there is one more step. You must register an exporter to send traces to a tracing backend.

#### Initialize and Register a Trace Exporter
This guide uses the Zipkin tracing backend, but if you are using another backend like [Jaeger](https://www.jaegertracing.io), this is where you would make your change.

To export traces, we will need a few more dependencies. Install them with the following command:

```sh
$ npm install \
  @opentelemetry/tracing \
  @opentelemetry/exporter-zipkin

$ # for jaeger you would run this command:
$ # npm install @opentelemetry/exporter-jaeger
```

After these dependencies are installed, we will need to initialize and register them. Modify `tracing.js` so that it matches the following code snippet, replacing the service name `"getting-started"` with your own service name if you wish.

```javascript
'use strict';

const opentelemetry = require("@opentelemetry/core")
const { NodeTracer } = require("@opentelemetry/node")

const { SimpleSpanProcessor } = require("@opentelemetry/tracing")
const { ZipkinExporter } = require("@opentelemetry/exporter-zipkin");

const tracer = new NodeTracer({
	logLevel: opentelemetry.LogLevel.ERROR
});

opentelemetry.initGlobalTracer(tracer);

tracer.addSpanProcessor(
  new SimpleSpanProcessor(
    new ZipkinExporter({
      serviceName: "getting-started",
      // If you are running your tracing backend on another host,
      // you can point to it using the `url` parameter of the
      // exporter config.
    })
  )
);


console.log("tracing initialized")
```

Now if you run your application with the `tracing.js` file loaded, and you send requests to your application over HTTP (in the sample application just browse to http://localhost:8080), you will see traces exported to your tracing backend that look like this:

```sh
$ node -r ./tracing.js app.js
```

![](images/zipkin-trace.png)

**Note:** Some spans appear to be duplicated, but they are not. This is because the sample application is both the client and the server for these requests. You see one span that is the client side request timing, and one span that is the server side request timing. Anywhere they don’t overlap is network time.

## Collect Metrics Using OpenTelemetry
This guide assumes you are going to be using Prometheus as your metrics backend. It is currently the only metrics backend supported by OpenTelemetry JS.

### Set up a Metrics Backend
Now that we have end-to-end traces, we will collect and export some basic metrics.

Currently, the only supported metrics backend is [Prometheus](https://prometheus.io). Head to the [Prometheus download page](https://prometheus.io/download/) and download the latest release of Prometheus for your operating system.

Open a command line and `cd` into the directory where you downloaded the Prometheus tarball. Untar it and change into the newly created directory.

```sh
$ cd Downloads

$ # Replace the file name below with your downloaded tarball
$ tar xvfz prometheus-2.14.0.darwin-amd64.tar

$ # Replace the dir below with your created directory
$ cd prometheus-2.14.0.darwin-amd64

$ ls
LICENSE           console_libraries data              prometheus.yml    tsdb
NOTICE            consoles          prometheus        promtool
```

The created directory should have a file named `prometheus.yml`. This is the file used to configure Prometheus. For now, just make sure Prometheus starts by running the `./prometheus` binary in the folder and browse to <http://localhost:9090>.

```sh
$ ./prometheus
# some output elided for brevity
msg="Starting Prometheus" version="(version=2.14.0, branch=HEAD, revision=edeb7a44cbf745f1d8be4ea6f215e79e651bfe19)"
# some output elided for brevity
level=info ts=2019-11-21T20:39:40.262Z caller=web.go:496 component=web msg="Start listening for connections" address=0.0.0.0:9090
# some output elided for brevity
level=info ts=2019-11-21T20:39:40.383Z caller=main.go:626 msg="Server is ready to receive web requests."
```

![](images/prometheus.png)


### Instrument Your NodeJS Application
#### Initialize a global meter
#### Initialize and register a metrics exporter
#### Create and keep updated any metrics you wish to collect