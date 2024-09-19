# OpenTelemetryApp Checklist
## Bootstrapping your application
1. **Install the following packages**:
   ```go
   "go get "go.opentelemetry.io/otel" \
   "go.opentelemetry.io/otel/exporters/stdout/stdoutmetric" \
   "go.opentelemetry.io/otel/exporters/stdout/stdouttrace" \
   "go.opentelemetry.io/otel/exporters/stdout/stdoutlog" \
   "go.opentelemetry.io/otel/sdk/log" \
   "go.opentelemetry.io/otel/log/global" \
   "go.opentelemetry.io/otel/propagation" \
   "go.opentelemetry.io/otel/sdk/metric" \
   "go.opentelemetry.io/otel/sdk/resource" \
   "go.opentelemetry.io/otel/sdk/trace" \
   "go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"\
   "go.opentelemetry.io/contrib/bridges/otelslog" 
 
   ```
2. **Initialize the OpenTelemetry SDK**: Create otel.go with OpenTelemetry SDK bootstrapping code
   ```go
   package main

   import (
	  "context"
  	"errors"
	  "time"

 	"go.opentelemetry.io/otel"
 	"go.opentelemetry.io/otel/exporters/stdout/stdoutlog"
 	"go.opentelemetry.io/otel/exporters/stdout/stdoutmetric"
 	"go.opentelemetry.io/otel/exporters/stdout/stdouttrace"
 	"go.opentelemetry.io/otel/log/global"
 	"go.opentelemetry.io/otel/propagation"
 	"go.opentelemetry.io/otel/sdk/log"
 	"go.opentelemetry.io/otel/sdk/metric"
 	"go.opentelemetry.io/otel/sdk/trace"
   )

   // setupOTelSDK bootstraps the OpenTelemetry pipeline.
   // If it does not return an error, make sure to call shutdown for proper cleanup.
   func setupOTelSDK(ctx context.Context) (shutdown func(context.Context) error, err error) {
   	var shutdownFuncs []func(context.Context) error
 
   	// shutdown calls cleanup functions registered via shutdownFuncs.
   	// The errors from the calls are joined.
   	// Each registered cleanup will be invoked once.
   	shutdown = func(ctx context.Context) error {
   		var err error
   		for _, fn := range shutdownFuncs {
   			err = errors.Join(err, fn(ctx))
   		}
  		shutdownFuncs = nil
  		return err
  	}

	// handleErr calls shutdown for cleanup and makes sure that all errors are returned.
	handleErr := func(inErr error) {
		err = errors.Join(inErr, shutdown(ctx))
	}

	// Set up propagator.
	prop := newPropagator()
	otel.SetTextMapPropagator(prop)

	// Set up trace provider.
	tracerProvider, err := newTraceProvider()
	if err != nil {
		handleErr(err)
		return
	}
	shutdownFuncs = append(shutdownFuncs, tracerProvider.Shutdown)
	otel.SetTracerProvider(tracerProvider)

	// Set up meter provider.
	meterProvider, err := newMeterProvider()
	if err != nil {
		handleErr(err)
		return
	}
	shutdownFuncs = append(shutdownFuncs, meterProvider.Shutdown)
	otel.SetMeterProvider(meterProvider)

	// Set up logger provider.
	loggerProvider, err := newLoggerProvider()
	if err != nil {
		handleErr(err)
		return
	}
	shutdownFuncs = append(shutdownFuncs, loggerProvider.Shutdown)
	global.SetLoggerProvider(loggerProvider)

	return
   }
   
   func newPropagator() propagation.TextMapPropagator {
   	return propagation.NewCompositeTextMapPropagator(
   		propagation.TraceContext{},
   		propagation.Baggage{},
   	)
   }
   
   func newTraceProvider() (*trace.TracerProvider, error) {
   	traceExporter, err := stdouttrace.New(
   		stdouttrace.WithPrettyPrint())
   	if err != nil {
   		return nil, err
   	}
   
   	traceProvider := trace.NewTracerProvider(
   		trace.WithBatcher(traceExporter,
   			// Default is 5s. Set to 1s for demonstrative purposes.
   			trace.WithBatchTimeout(time.Second)),
   	)
   	return traceProvider, nil
   }
   
   func newMeterProvider() (*metric.MeterProvider, error) {
   	metricExporter, err := stdoutmetric.New()
   	if err != nil {
   		return nil, err
   	}
   
   	meterProvider := metric.NewMeterProvider(
   		metric.WithReader(metric.NewPeriodicReader(metricExporter,
   			// Default is 1m. Set to 3s for demonstrative purposes.
   			metric.WithInterval(3*time.Second))),
   	)
   	return meterProvider, nil
   }
   
   func newLoggerProvider() (*log.LoggerProvider, error) {
   	logExporter, err := stdoutlog.New()
   	if err != nil {
   		return nil, err
   	}
   
   	loggerProvider := log.NewLoggerProvider(
   		log.WithProcessor(log.NewBatchProcessor(logExporter)),
   	)
   	return loggerProvider, nil
   }
   ```
3. **Instrument the HTTP server**: Modify main.go to include code that sets up OpenTelemetry SDK and instruments the HTTP server using the otelhttp instrumentation library
   ```go
   package main

   import (
   	"context"
   	"errors"
   	"log"
   	"net"
   	"net/http"
   	"os"
   	"os/signal"
   	"time"
   
   	"go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"
   )
   
   func main() {
   	if err := run(); err != nil {
   		log.Fatalln(err)
   	}
   }
   
   func run() (err error) {
   	// Handle SIGINT (CTRL+C) gracefully.
   	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt)
   	defer stop()
   
   	// Set up OpenTelemetry.
   	otelShutdown, err := setupOTelSDK(ctx)
   	if err != nil {
   		return
   	}
   	// Handle shutdown properly so nothing leaks.
   	defer func() {
   		err = errors.Join(err, otelShutdown(context.Background()))
   	}()
   
   	// Start HTTP server.
   	srv := &http.Server{
   		Addr:         ":8080",
   		BaseContext:  func(_ net.Listener) context.Context { return ctx },
   		ReadTimeout:  time.Second,
   		WriteTimeout: 10 * time.Second,
   		Handler:      newHTTPHandler(),
   	}
   	srvErr := make(chan error, 1)
   	go func() {
   		srvErr <- srv.ListenAndServe()
   	}()
   
   	// Wait for interruption.
   	select {
   	case err = <-srvErr:
   		// Error when starting HTTP server.
   		return
   	case <-ctx.Done():
   		// Wait for first CTRL+C.
   		// Stop receiving signal notifications as soon as possible.
   		stop()
   	}
   
   	// When Shutdown is called, ListenAndServe immediately returns ErrServerClosed.
   	err = srv.Shutdown(context.Background())
   	return
       }
   
   func newHTTPHandler() http.Handler {
   	mux := http.NewServeMux()
   
   	// handleFunc is a replacement for mux.HandleFunc
   	// which enriches the handler's HTTP instrumentation with the pattern as the http.route.
   	handleFunc := func(pattern string, handlerFunc func(http.ResponseWriter, *http.Request)) {
   		// Configure the "http.route" for the HTTP instrumentation.
   		handler := otelhttp.WithRouteTag(pattern, http.HandlerFunc(handlerFunc))
   		mux.Handle(pattern, handler)
   	}
   
   	// Register handlers.
   	handleFunc("/rolldice/", rolldice)
   	handleFunc("/rolldice/{player}", rolldice)
   
   	// Add HTTP instrumentation for the whole server.
   	handler := otelhttp.NewHandler(mux, "/")
   	return handler
   }

   ```
## Checklist to create metrics using OpenTelemetry (OTel) in a Go application
1.	**Install OpenTelemetry Go SDK**: Begin by installing the OpenTelemetry Go SDK in your Go environment. You can use the following command to install the SDK:
```go
 go get go.opentelemetry.io/otel
```
2.	**Instrument Your Application**: Import the necessary packages from the OpenTelemetry Go SDK and instrument your application code to create and record metrics. This typically involves adding code snippets at specific points in your application's logic to capture relevant metrics data.
3.	**Create a Meter**: Create an instance of the Meter interface from the OpenTelemetry SDK. The Meteris responsible for creating and managing metrics. You can use the following code snippet as a starting point: 
```go
import (
  "go.opentelemetry.io/otel"
  "go.opentelemetry.io/otel/metric"
  "go.opentelemetry.io/otel/metric/global"
)

func setupMeter() metric.Meter {
    otel.SetErrorHandler(otel.StdErrorHandler)
    return global.Meter("your-metric-meter")
}
```

4.	**Define Metrics**: Define the specific metrics you want to capture. OpenTelemetry provides various types of metrics, such as counters, gauges, and histograms. Choose the appropriate metric type based on your requirements. This example creates a counter metric named "requests_total". Here's an example of defining a counter metric:
```go
func defineMetrics(meter metric.Meter) {
     counter := metric.Must(meter).NewInt64Counter("requests_total")
}
```
 

5.	**Record Metric Data**: At relevant points in your application's code, record metric data using the metrics you defined. For example, to increment the counter metric from the previous step, you can use the following code, This code increments the "requests_total" counter by 1 for each request handled:
```go
func handleRequest() {
  // Your request handling logic

 counter.Add(context.Background(), 1)
}
```

6.	**Configure Exporters**: Configure exporters to send the captured metrics data to your backend or monitoring system. OpenTelemetry provides various exporters for popular systems like Prometheus, Jaeger, and more. Refer to the documentation of your chosen exporter to configure it correctly.
7.	**Run and Monitor**: Run your Go application and monitor the exported metrics in your chosen backend or monitoring system. Verify that the metrics are being recorded and displayed correctly.
By following this checklist, you should be able to successfully create metrics using OpenTelemetry in your Go application. Remember to consult the OpenTelemetry Go documentation for more detailed information and examples.

## Checklist to create logs using OpenTelemetry (OTel) in a Go application

1. **Initializing the logger using logrus**:

```go
import (
    "github.com/sirupsen/logrus"
)

func main() {
    // Initialize logrus logger
    logger := logrus.New()

    // Set log level
    logger.SetLevel(logrus.InfoLevel)

    // Set log output destination
    logger.SetOutput(os.Stdout)

    // Example log statements
    logger.Info("Application started")
    logger.Warn("Something unexpected happened")
    logger.Error("An error occurred")
}
```

2. **Initializing the logger using zap**:

```go
import (
    "go.uber.org/zap"
    "go.uber.org/zap/zapcore"
)

func main() {
    // Initialize zap logger configuration
    config := zap.Config{
        Level:            zap.NewAtomicLevelAt(zap.InfoLevel),
        Encoding:         "json",
        OutputPaths:      []string{"stdout"},
        ErrorOutputPaths: []string{"stderr"},
        EncoderConfig: zapcore.EncoderConfig{
            TimeKey:        "timestamp",
            LevelKey:       "level",
            MessageKey:     "message",
            EncodeTime:     zapcore.ISO8601TimeEncoder,
            EncodeLevel:    zapcore.LowercaseLevelEncoder,
            EncodeDuration: zapcore.SecondsDurationEncoder,
        },
    }

    // Initialize zap logger
    logger, _ := config.Build()

    // Example log statements
    logger.Info("Application started")
    logger.Warn("Something unexpected happened")
    logger.Error("An error occurred")
}
```

3. **Initializing OpenTelemetry tracer and logging with zap**:

```go
import (
    "go.opentelemetry.io/otel"
    "go.opentelemetry.io/otel/exporters/stdout"
    "go.opentelemetry.io/otel/sdk/trace"
    "go.uber.org/zap"
)

func main() {
    // Initialize OpenTelemetry tracer with stdout exporter
    exporter, _ := stdout.NewExporter(stdout.WithPrettyPrint())
    provider := trace.NewTracerProvider(trace.WithBatcher(exporter))
    otel.SetTracerProvider(provider)

    // Initialize zap logger
    logger, _ := zap.NewDevelopment()

    // Example log statements with OpenTelemetry integration
    tracer := otel.Tracer("example")
    ctx, span := tracer.Start(context.Background(), "exampleSpan")
    defer span.End()

    logger.Info("Application started", zap.String("spanID", span.SpanContext().SpanID().String()))
    logger.Warn("Something unexpected happened", zap.String("spanID", span.SpanContext().SpanID().String()))
    logger.Error("An error occurred", zap.String("spanID", span.SpanContext().SpanID().String()))
}
```

Please note that these snippets are just examples, and you may need to modify them based on your specific requirements and the logging library you choose to use.

## Checklist to create distributed traces using OpenTelemetry (OTel) in a Go application
1. **Install OpenTelemetry Go SDK**: Begin by installing the OpenTelemetry Go SDK in your Go environment. You can use the following command to install the SDK:

   ```
   go get go.opentelemetry.io/otel
   ```

2. **Instrument Your Application**: Import the necessary packages from the OpenTelemetry Go SDK and instrument your application code to create and propagate traces. This typically involves adding code snippets at specific points in your application's logic to capture relevant trace data.

3. **Create a Tracer Provider**: Create an instance of the `TracerProvider` from the OpenTelemetry SDK. The `TracerProvider` is responsible for managing and creating traces. You can use the following code snippet as a starting point:

   ```go
   import (
       "go.opentelemetry.io/otel"
       "go.opentelemetry.io/otel/trace"
       "go.opentelemetry.io/otel/exporters/trace/jaeger"
   )

   func setupTracerProvider() (trace.TracerProvider, error) {
       otel.SetErrorHandler(otel.StdErrorHandler)

       // Create and configure a Jaeger exporter
       exporter, err := jaeger.New(jaeger.WithCollectorEndpoint("http://localhost:14268/api/traces"))
       if err != nil {
           return nil, err
       }

       // Create a tracer provider with the exporter
       tp := trace.NewTracerProvider(
           trace.WithBatcher(exporter),
           trace.WithSampler(trace.AlwaysSample()),
       )

       return tp, nil
   }
   ```

   Replace `"http://localhost:14268/api/traces"` with the appropriate endpoint for your chosen trace backend, such as Jaeger.

4. **Create a Tracer**: Create an instance of the `Tracer` interface from the `TracerProvider`. The `Tracer` is for creating spans to represent individual traces. You can use the following code snippet as a starting point:

   ```go
   func createTracer(tp trace.TracerProvider) trace.Tracer {
       return tp.Tracer("your-trace-service")
   }
   ```

   Replace `"your-trace-service"` with a meaningful name for your trace service.

5. **Start and End Spans**: At relevant points in your application's code, start and end spans to represent the trace. Use the `Tracer` instance you created to create spans. Here's an example of starting and ending a span:

   ```go
   func handleRequest(tracer trace.Tracer) {
       // Start a new span
       ctx, span := tracer.Start(context.Background(), "handle_request")
       defer span.End()

       // Your request handling logic
   }
   ```

   This code starts a new span named "handle_request" and ends it when the request handling is complete.

6. **Propagate Context**: If your application makes outgoing requests or spawns goroutines, ensure that the trace context is propagated. This ensures that the trace is carried across different components of your application. Refer to the OpenTelemetry Go documentation for specific instructions on propagating context in your particular use case.

7. **Configure Exporters**: Configure exporters to send the captured trace data to your backend or tracing system. OpenTelemetry provides various exporters for popular systems like Jaeger, Zipkin, and more. Refer to the documentation of your chosen exporter to configure it correctly.

8. **Run and Monitor**: Run your Go application and monitor the distributed traces in your chosen backend or tracing system. Verify that the traces are being captured and displayed correctly.

By following this checklist, you should be able to successfully create distributed traces using OpenTelemetry in your Go application. Remember to consult the OpenTelemetry Go documentation for more detailed information and examples.

OpenTelemetry Documentation: https://opentelemetry.io/docs/

Please connect me on LinkedIn if you have more questions: https://www.linkedin.com/in/monalisha-singh-85ba1091/



