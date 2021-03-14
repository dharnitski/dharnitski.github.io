---
layout: post
title: "Golang APM with Opentracing and Datadog"
categories: 
tags: Tracing APM DataDog Opentracing Go
excerpt: Opentracing is getting popular as a vendor-agnostic APM API for microservices. This article describes how to use it in Go service and with Datadog.
---

## Why we need both

Datadog provides many features, one of which is application performance management (APM). It comes with SDK for many languages including [Go](https://github.com/DataDog/dd-trace-go). That library is native to DataDog and provides the best possible experience. Why should we try something else?

You do not need to do that if you are not afraid to lock your tracing implementation with Datadog. Usually, custom tracing code is added to service in many different places. If one day you decide to switch from Datadog to something else it will be an expensive and error-prone exercise. 

The typical solution for this issue is to wrap the vendor-specific code and use that wrapper instead of using a third party library directly. In that case, you can switch to another vendor just by replacing the implementation of the wrapper without touching (except for recompilation) the service code.

How Opentracing can help us? Opentracing is a wrapper we just discussed that already exists.

Before we dive into the implementation, let's discuss the limitations we have to deal with.

## Limitation 1 - incompatibility

Datadog provides Opentracing integration out of the box - https://pkg.go.dev/gopkg.in/DataDog/dd-trace-go.v1/ddtrace/Opentracing.

As a developer, you can emit tracing items (Spans) using either opentracing or a native Datadog interface.

Both spans below are delivered to Datadog and provide the same result. 

```go
import (
	"github.com/opentracing/opentracing-go"
	"gopkg.in/DataDog/dd-trace-go.v1/ddtrace/Opentracing"
	"gopkg.in/DataDog/dd-trace-go.v1/ddtrace/tracer"
)

func someFunc() {
	// Start a Datadog tracer
	t := Opentracing.New()
	// Use it with the Opentracing API.
	opentracing.SetGlobalTracer(t)

	// start opentracing Span
	opentracing.StartSpan("http.request", Opentracing.ResourceName("/posts"))

	// start datadog Span
	tracer.StartSpan("http.request", tracer.ResourceName("/posts"))
}
```

Problems start when we try to connect Datadog and Opentracing spans. Soon we realize that spans created using different frameworks are disconnected.
Datadog receives both of them, but they are rendered as two independent items. Tracing losses a big portion of its power if we cannot connect its pieces.

That happens because Datadog and Opentracing internally use different IDs to store data in Context. They both have code to find parent Span in `context.Context` but that code see only parents save by the same library. 

From a practical perspective, it makes sense to use either native Datadog libraries or opentracing, but do not mix them.
Decide if you are ready to commit to Opentracing before writing custom traces.

## Limitation 2 - missing features

Opentracing implements a generic interface and that interface does not include some Datadog features.

For example, Span can be created with a custom Span ID in Datadog. That is implemented as [func WithSpanID(id uint64) StartSpanOption](https://github.com/DataDog/dd-trace-go/blob/fc56ee060d048ef77b8e380575d30efbfe185f8b/ddtrace/tracer/option.go#L456). It is not possible to do with the Opentracing interface.

I suggest you check the [options.go](https://github.com/DataDog/dd-trace-go/blob/v1/ddtrace/tracer/option.go) file to see how Datadog implements options for Span Start and Finish. Some of these options assign custom tags to Span and these options can be implemented using Opentracing. Otherwise, you only can apply options globally for all Spans using Tracer.

Now we can jump into implementation if the limitations above did not scare you.

## Global Tracer configuration

Typically companies use some shared library linked into every service to reuse common functionality. 

In Go shared package referred to as a [kit](http://peter.bourgon.org/go-kit/). 

All references to Datadog libraries stay in the kit. Tracing code inside services is to be implemented using only Opentracing APIs.

`kit` includes a function that every service needs to call on app start. That function creates and starts a global tracer. 

Two essential parameters in the code below enable slicing tracing data per environment (DEV, QA, STAGING, PROD, etc) and service. You can extend configuration as necessary for your setup.


```go
import (
	"github.com/opentracing/opentracing-go"
	"gopkg.in/DataDog/dd-trace-go.v1/ddtrace/Opentracing"
	"gopkg.in/DataDog/dd-trace-go.v1/ddtrace/tracer"
)

func Initialize(env string, serviceName string) {
	// start Datadog tracer and wrap in into Opentracing interface
	t := Opentracing.New(
		// inject all required tracing options here
		tracer.WithEnv(env),
		tracer.WithService(service))
	// Set tracer as a Global Tracer
	opentracing.SetGlobalTracer(t)
}
```

In addition to the global tracer, the `kit` can include reusable wrappers for common IO operations.

### gorilla/mux

Function `NewMux()` creates a new HTTP handler that records all incoming requests.

```go
import (
	"github.com/gorilla/mux"
	"github.com/opentracing-contrib/go-gorilla/gorilla"
	"github.com/opentracing/opentracing-go"
)

func opentracingMiddleware(next http.Handler) http.Handler {
	tracer := opentracing.GlobalTracer()
	return gorilla.Middleware(tracer,next)
}

func NewMux() *mux.Router {
	router := mux.NewRouter()
	router.Use(opentracingMiddleware)
	// here we can add common routes like health check
	return router
}
```

### aws

Opentracing includes AWS wrapper for Go - [opentracing-contrib/go-aws-sdk](https://github.com/opentracing-contrib/go-aws-sdk). That library works great, but it does miss some features [Datadog wrapper](https://github.com/DataDog/dd-trace-go/tree/v1/contrib/aws/aws-sdk-go/aws) provides. First of all, it is a generic library that does not implement Datadog specific tags. Secondly, it implements instrumentation for individual AWS services. That may be fine, but it is just simpler to instrument the whole AWS session including all services. I created a little package that reproduces Datadog wrapper implementation but do it using Opentracing API:

```go
import (
	"log"

	"github.com/aws/aws-sdk-go/aws/session"
	awsdd "github.com/dharnitski/opentracing-aws-dd"
)

func ExampleWrapSession() {
	session, err := session.NewSession()
	if err != nil {
		log.Fatal(err)
	}
	// session is instrumented with global tracer
	session = awsdd.WrapSession(session)
}
```

### sqlx with Postgres

We need to do some tricks to get an instrumented Postgres DB connection. 

Let's pick SQL driver for Postgres. Golang library does not include any SQL drivers and we have to rely on a third-party. `github.com/lib/pq` was a standard de facto for Postgres, but currently, it is in maintenance mode and recommends an actively maintained alternative `github.com/jackc/pgx/v4/stdlib` on its homepage. We proceed with it. That library resters its driver for name `pgx`.

Service creates DB Connection using `sql(x).Open()` function. That function takes the driver's name as the first argument. We get working but not instrumented version of driver if we use `pgx`. To solve that issue, we create an instrumented version of the driver using `otsql.NewTracingDriver()` function. Once it is created, we register a new driver with some unique name to make it available for `sql(x).Open()` function. The trick here is to carefully pick the driver name because not every name works the same way in SQLX. SQLX respects different flavors of SQL parameters and it uses driver name to decide on the syntax for parameters binding. We have to use one of Postgres related driver names to make SQLX parse query parameters using native Postgres dollar sign syntax. You can find a full list of supported driver names in [sqlx sources](https://github.com/jmoiron/sqlx/blob/ee514944af4b0d1b90212831674e03df1b850998/bind.go#L26). We use `ql` name for our wrapper.

Now it is time to talk about options available for `otsql.NewTracingDriver()` function. By default, with an empty list of options that function creates a driver that does not log the body of SQL query and every operation logged using a verbose name that includes `opentracing-sql` package name and struct. For example, this is the name for the `Ping` function - `github.com/inkbe/opentracing-sql.(*conn).Ping`. We can make that name shorter with a custom function injected as `SpanNameFunction` option.

Here is a final version of code:

```go
import (
	"context"
	"database/sql"
	"fmt"
	"log"
	"runtime"
	"strings"
	"sync"

	otsql "github.com/inkbe/opentracing-sql"
	"github.com/jackc/pgx/v4/stdlib"
	"github.com/jmoiron/sqlx"
	"github.com/opentracing/opentracing-go"
)

// Call stack at the moment of call to the function has the following frames (digits represent the depth from the top):
// 0 - name function itself.
// 1 - newSpan.
// 2 - wrapper function in this package, e.g. QueryContext.
func nameFunc(ctx context.Context) string {
	pc, _, _, ok := runtime.Caller(2)
	if !ok {
		return ""
	}
	f := runtime.FuncForPC(pc)
	if f == nil {
		return ""
	}
	// name = github.com/inkbe/opentracing-sql.(*conn).Ping
	name := f.Name()
	parts := strings.Split(name, ".")
	if len(parts) < 2 {
		return name
	}

	// name = Ping
	return parts[len(parts)-1]
}

var once sync.Once

// NewDB creates DB and tests it
// Detailed error information written in log
// Returned error is generic and can be returned to caller
func NewDB(ctx context.Context, dataSourceName string) (*sqlx.DB, error) {
	once.Do(func() {
		// stdlib.GetDefaultDriver() registers `pgx` driver and there is no way to unregister it
		// sqlx rely on driver name to resolve binding type:
		// https://github.com/jmoiron/sqlx/blob/ee514944af4b0d1b90212831674e03df1b850998/bind.go#L26
		// using `ql` as one that is Postgres compatible
		sql.Register("ql", otsql.NewTracingDriver(stdlib.GetDefaultDriver(), opentracing.GlobalTracer(),
			otsql.SpanNameFunction(nameFunc),
			// nameFunc here makes no sense and ignored but it is how it is built
			otsql.SaveQuery(nameFunc),
		))
	})
	conn, err := sqlx.ConnectContext(ctx, "ql", dataSourceName)
	if err != nil {
		// error message can contain connection string. Returning generic error and log details
		log.Printf("db connection error: %v", err)
		return nil, fmt.Errorf("cannot Connect to DB. See logs for details")
	}
	return conn, err
}
```

### Custom Entries

Opentracing connects spans using [opentracing.SpanContext](https://pkg.go.dev/github.com/opentracing/opentracing-go#SpanContext). This structure is separate from Go native [context.Context](https://golang.org/pkg/context/#Context). Despite that, opentracing for Go injects `opentracing.SpanContext` into `context.Context` and in the majority of cases developer needs to deal only with `context.Context` to connect spans.

Lets see how that works for HTTP handler:

```go
// myHandler handles `/my-path` route
func myHandler(w http.ResponseWriter, r *http.Request) {
	// getting context from request
	// `opentracingMiddleware` already created span for HTTP handler 
	// and injected `opentracing.SpanContext` into `ctx`
	ctx := r.Context()
	// send context down the stack to connect all the pieces
	// we have to do it on every layer now
	doWork(ctx)
}

func doWork(ctx context.Context) {
	// create a child span for request span 
	span, ctx := opentracing.StartSpanFromContext(ctx, "my_work",
		// tags are span wide pieces of data
		opentracing.Tag{Key: "tag_key", Value: "tag_value"},
	)
	// close the span when we exit from function
	defer span.Finish()
	// do actual work here
}

func initialize() {
	router := mux.NewRouter()
	router.Use(opentracingMiddleware)
	// add route for API
	router.HandleFunc("/my-path", myHandler).Methods("GET")
}

```

### Testing

We can test instrumented code by starting [DogStatsD](https://docs.datadoghq.com/developers/dogstatsd/?tab=go) and posting data to Datadog servers. Manual testing is a working solution, but it is not efficient. Tracer can be mocked and tested with unit tests.
Opentracing includes a built-in mock for tracing - `github.com/opentracing/opentracing-go/mocktracer`.
This is how we can test trace inside `doWork` function we just wrote:

```go
import (
	"context"
	"testing"

	"github.com/opentracing/opentracing-go"
	"github.com/opentracing/opentracing-go/mocktracer"
	"github.com/stretchr/testify/assert"
)

func TestTracing(t *testing.T) {
	// uses shared tracer, tests cannot be run in parallel
	// t.Parallel()

	// use mocked tracer instead of real Datadog connector
	tr := mocktracer.New()
	// make mocked tracer globally available
	opentracing.SetGlobalTracer(tr)
	// teardown, reset mock state
	defer tr.Reset()

	// emulate HTTP request wrapper
	// it creates span for request and register it in r.Context
	root, ctx := opentracing.StartSpanFromContext(context.Background(), "request")

	// act, execute function with charged context
	doWork(ctx)
	// emulate end of request execution, request span is closed when request is over
	root.Finish()

	// get finished spans from mock
	spans := tr.FinishedSpans()
	// cast `opentracing.SpanContext` to get access to TraceID
	// TraceID is an identifier that connects spans
	sc := root.Context().(mocktracer.MockSpanContext)
	// we expect 2 spans to be finished - request and one inside doWork func
	assert.Len(t, spans, 2)
	// check that all spans are tied together
	for _, s := range spans {
		assert.Equal(t, sc.TraceID, s.SpanContext.TraceID, s)
	}
	// inner span is closed first
	s := spans[0]
	// checking span data
	assert.Equal(t, "my_work", s.OperationName)
	assert.Equal(t, "tag_value", s.Tag("tag_key"))
}
```

That is it for today.

Happy tracing!