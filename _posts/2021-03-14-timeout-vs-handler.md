---
layout: post
title: "http.Server.WriteTimeout vs http.TimeoutHandler"
categories: 
tags: Go HTTP REST
excerpt: By default, the Go HTTP Server puts no time limits for execution. It is the developer's responsibility to set appropriate timeouts to prevent connection leaks. This article compares two options to manage timeouts.     
---

In my current project team faced a strange issue. Some requests to our service failed without leaving any traces in logs and APM. When we dig into it, the first clue that we found was higher data size these requests process that requires more time to execute the request. It ended up to be standard behavior for not empty `http.Server.WriteTimeout` parameter. This article describes `http.Server.WriteTimeout` behavior and compares it with an alternative solution - `http.TimeoutHandler`

## Similarities

**Purpose** `WriteTimeout` and `TimeoutHandler` are both designed to solve the same problem. They limit request execution time in HTTP Server to protect resources on client and server. Without limiting, open connections live forever slowing down the application and eventually making it not reachable.   

**Handler code interruption** Both these utilities controls the max time client waits for a response. On another side, server execution is not interrupted when a timeout is reached. `WriteTimeout` and `TimeoutHandler` have different internal logic, but they both do not terminate the execution of the handler goroutine. You can expect that long-running work inside the handler can be finished even though the request is terminated for the client.

## Differences

**Scope** `WriteTimeout` defined in `http.Server` and its value applied to all handlers with not [Hijacked](https://golang.org/pkg/net/http/#example_Hijacker) connections. The `TimeoutHandler` function is more flexible. It implements a Handler Middleware pattern. A developer can apply it to all handlers if it wraps `http.Server.Handler` or it could wrap individual handlers using different timeout values.

**Visibility for clients** `WriteTimeout` simply closes TCP connection if response writing exceeds timeout. If you use `curl` utility, for example, an interrupted connection looks like `curl: (52) Empty reply from server` error. `TimeoutHandler` does not close the connection. Instead, it keeps a connection open but returns an HTTP response with 503 Service Unavailable status. A developer can specify the body of that response as a static string when `TimeoutHandler` is declared. 

**Async execution** `WriteTimeout` is synchronous. Its value checked at the end of `readRequest()` function. If the timeout is not equal to 0, the server sets a writing deadline on the network file descriptor. This implementation has an interesting side effect. The connection deadline is evaluated only at the moment when someone writes into that connection. It means that `WriteTimeout` does exactly what its name means, but not what the developer may expect from it. The long operation to prepare the data can take longer than `WriteTimeout` but the connection will be closed only when we start writing the first response byte to the connection.

`TimeoutHandler` is asynchronous. It starts a new goroutine that does two things when it reaches timeout. At first, it finishes the request by sending 503 Service Unavailable response to the caller. Secondly, it cancels request Context providing a callback to the handler that the developer can use to react in timeout event inside the handler. Due to its asynchronous nature, `TimeoutHandler` always terminates requests on-time even if the handler executes a blocking long-running task. 

**Timeout mitigation in handler** `WriteTimeout`  hides all timeout mechanics from handler code. There is no easy way to catch timeout events in server code. The handler receives no Context cancellation, `w.Write()` returns no error and even returns the number of bytes greater than zero. `TimeoutHandler` from the other side allows catching timeout events. First of all, the handler can listen to request Context cancellation. For example, we can rollback transactions or delete temporary files inside that callback function. Secondly, `w.Write()` returns an `ErrHandlerTimeout` error if we try to write to the response after the timeout event.

## Conclusion

Go was initially shipped with `http.Server.WriteTimeout` to put limits on request writing time. That parameter controls low-level connection deadline and does what is intended to do just fine. `http.TimeoutHandler` was added later in Go v1.8 after Go received `context.Context`. `http.TimeoutHandler` provides better SLA, development interface, and flexibility. It is a better fit for many Go applications.

Happy time-outing!