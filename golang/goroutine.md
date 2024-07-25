## Goroutine - TIL

Các kiến thức đã học được về goroutine

### Goroutine introduction

<https://go.dev/tour/concurrency/1>

A goroutine is a lightweight thread managed by the Go runtime.

```go
go f(x, y, z)
```
starts a new goroutine running

```go
f(x, y, z)
```
The evaluation of `f, x, y, and z` happens in the current goroutine and the execution of f happens in the new goroutine.

Goroutines run in the same address space, so access to shared memory must be synchronized. The sync package provides useful primitives, although you won't need them much in Go as there are other primitives. (See the next slide.)

### Channels

Channels are a typed conduit through which you can send and receive values with the channel operator, <-.

```go
ch <- v    // Send v to channel ch.
v := <-ch  // Receive from ch, and
           // assign value to v.
```

(The data flows in the direction of the arrow.)
Like maps and slices, channels must be created before use:

```go
ch := make(chan int)
```
By default, **sends and receives block until the other side is ready**. This allows goroutines to synchronize without explicit locks or condition variables.

The example code sums the numbers in a slice, distributing the work between two goroutines. Once both goroutines have completed their computation, it calculates the final result.

<https://dev.to/gophers/what-are-goroutines-and-how-are-they-scheduled-2nj3>

Goroutines are cheap to create and very easy to use. Any function in go can be run concurrently by simply appending the go keyword to the function call.

**So goroutines are threads?**

To people new to Go, the word goroutine and thread get used a little interchangeably. This makes sense if you come from a language such as Java where you can quite literally make new OS threads. Go is different, and a goroutine is not the same as a thread. Threads are much more expensive to create, use more memory and switching between threads takes longer.

Goroutines are an abstraction over threads and a single Operating System thread can run many goroutines.

**So How are goroutines scheduled?**

- <https://www.youtube.com/watch?v=YHRO5WQGh0k>
- 


### Goroutine example: HTTPTimeoutHandler in Go Built-in HTTP Server Package

- <https://ieftimov.com/posts/make-resilient-golang-net-http-servers-using-timeouts-deadlines-context-cancellation/>
- <https://cs.opensource.google/go/go/+/master:src/net/http/server.go;l=3620>
- <https://ieftimov.com/posts/make-resilient-golang-net-http-servers-using-timeouts-deadlines-context-cancellation/>

```go

// TimeoutHandler returns a [Handler] that runs h with the given time limit.
//
// The new Handler calls h.ServeHTTP to handle each request, but if a
// call runs for longer than its time limit, the handler responds with
// a 503 Service Unavailable error and the given message in its body.
// (If msg is empty, a suitable default message will be sent.)
// After such a timeout, writes by h to its [ResponseWriter] will return
// [ErrHandlerTimeout].
//
// TimeoutHandler supports the [Pusher] interface but does not support
// the [Hijacker] or [Flusher] interfaces.
func TimeoutHandler(h Handler, dt time.Duration, msg string) Handler {
	return &timeoutHandler{
		handler: h,
		body:    msg,
		dt:      dt,
	}
}

// ErrHandlerTimeout is returned on [ResponseWriter] Write calls
// in handlers which have timed out.
var ErrHandlerTimeout = errors.New("http: Handler timeout")

type timeoutHandler struct {
	handler Handler
	body    string
	dt      time.Duration

	// When set, no context will be created and this context will
	// be used instead.
	testContext context.Context
}

func (h *timeoutHandler) errorBody() string {
	if h.body != "" {
		return h.body
	}
	return "<html><head><title>Timeout</title></head><body><h1>Timeout</h1></body></html>"
}

func (h *timeoutHandler) ServeHTTP(w ResponseWriter, r *Request) {
	ctx := h.testContext
	if ctx == nil {
		var cancelCtx context.CancelFunc
		ctx, cancelCtx = context.WithTimeout(r.Context(), h.dt)
		defer cancelCtx()
	}
	r = r.WithContext(ctx)
	done := make(chan struct{})
	tw := &timeoutWriter{
		w:   w,
		h:   make(Header),
		req: r,
	}
	panicChan := make(chan any, 1)
	go func() {
		defer func() {
			if p := recover(); p != nil {
				panicChan <- p
			}
		}()
		h.handler.ServeHTTP(tw, r)
		close(done)
	}()
	select {
	case p := <-panicChan:
		panic(p)
	case <-done:
		tw.mu.Lock()
		defer tw.mu.Unlock()
		dst := w.Header()
		for k, vv := range tw.h {
			dst[k] = vv
		}
		if !tw.wroteHeader {
			tw.code = StatusOK
		}
		w.WriteHeader(tw.code)
		w.Write(tw.wbuf.Bytes())
	case <-ctx.Done():
		tw.mu.Lock()
		defer tw.mu.Unlock()
		switch err := ctx.Err(); err {
		case context.DeadlineExceeded:
			w.WriteHeader(StatusServiceUnavailable)
			io.WriteString(w, h.errorBody())
			tw.err = ErrHandlerTimeout
		default:
			w.WriteHeader(StatusServiceUnavailable)
			tw.err = err
		}
	}
}
```

Usage this handler

```go
package main

import (
	"fmt"
	"io"
	"net/http"
	"time"
)

func slowHandler(w http.ResponseWriter, req *http.Request) {
	time.Sleep(2 * time.Second)
	io.WriteString(w, "I am slow!\n")
}

func main() {
	srv := http.Server{
		Addr:         ":8888",
		WriteTimeout: 5 * time.Second,
		Handler:      http.TimeoutHandler(http.HandlerFunc(slowHandler), 1*time.Second, "Timeout!\n"),
	}

	if err := srv.ListenAndServe(); err != nil {
		fmt.Printf("Server failed: %s\n", err)
	}
}
```

<https://groups.google.com/g/golang-nuts/c/kvP6VlWtZlg>


### How to cancel a running goroutine when using with TimeoutHandler

<https://stackoverflow.com/questions/60737889/http-timeouthandler-returns-but-handlerfunc-keeps-running>

> When the timeout happens and your handler function still runs (haven't returned), the request's context will be cancelled. Your handler is responsible to monitor the Context's Done channel, and abort its work if cancel is requested. Each handler runs in its own goroutine, and goroutines cannot be killed or interrupted from the "outside"

<https://groups.google.com/g/golang-nuts/c/kvP6VlWtZlg>

Your inner request handler needs to use the request context to cancel its work.

```go
package main

import (
	"log"
	"net/http"
	"time"
)

type foo struct{}

func (f foo) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	log.Print("New request")
	for i := 0; i < 10; i++ {
		select {
		case <-r.Context().Done():
			log.Print("Aborted")
			return
		case <-time.After(1 * time.Second):
			log.Print("Tick")
		}
		w.Write([]byte(".\n"))
	}
	w.Write([]byte("hello world\n"))
	log.Print("Completed")
}

func main() {
	fooHandler := foo{}
	timeoutHandler := http.TimeoutHandler(fooHandler, 5*time.Second, "Too slow!\n")
	http.Handle("/foo", timeoutHandler)
	log.Fatal(http.ListenAndServe(":8080", nil))
}

```

To test:

`curl localhost:8080/foo`

### Fire and forget goroutines

<https://stackoverflow.com/questions/68359637/fire-and-forget-goroutine-golang>

**Opinion 1:**

I would recommend always having your goroutines under control to avoid memory and system exhaustion. If you are receiving a spike of requests and you start spawning goroutines without control, probably the system will go down soon or later.

In those cases where you need to return an immediate 200Ok the best approach is to create a message queue, so the server only needs to create a job in the queue and return the ok and forget. The rest will be handled by a consumer asynchronously.

Producer (HTTP server) >>> Queue >>> Consumer

Normally, the queue is an external resource (RabbitMQ, AWS SQS...) but for teaching purposes, you can achieve the same effect using a channel as a message queue.

In the example you'll see how we create a channel to communicate 2 processes. Then we start the worker process that will read from the channel and later the server with a handler that will write to the channel.

**Opinion 2:**

There is no "goroutine cleaning" you have to handle, you just launch goroutines and they'll be cleaned when the function launched as a goroutine returns. Quoting from Spec: Go statements

> When the function terminates, its goroutine also terminates. If the function has any return values, they are discarded when the function completes.

<https://go.dev/ref/spec#Go_statements>

A "go" statement starts the execution of a function call as an independent concurrent thread of control, or goroutine, within the same address space.

The expression must be a function or method call; it cannot be parenthesized. Calls of built-in functions are restricted as for expression statements.

The function value and parameters are evaluated as usual in the calling goroutine, but unlike with a regular call, program execution does not wait for the invoked function to complete. Instead, the function begins executing independently in a new goroutine. When the function terminates, its goroutine also terminates. If the function has any return values, they are discarded when the function completes.

### Go routines leaks problems

- <https://brainbaking.com/post/2024/03/the-case-of-a-leaky-goroutine/>
- <https://www.uber.com/en-BE/blog/leakprof-featherlight-in-production-goroutine-leak-detection/>

Programmatic errors (e.g., complex control flow, early returns, timeouts), can lead to mismatch in communication between goroutines, where one or more goroutines may block but no other goroutine will ever create the necessary conditions for unblocking. Goroutine leaks prevent the  garbage collector from reclaiming the associated channel, goroutine stack, and all reachable objects of the permanently blocked goroutine. Long-running services where small leaks accumulate over time exacerbate the problem. 

Go distribution does not offer any out-of-the-box solution for detecting goroutine leaks, whether during compilation or at runtime. Detecting goroutine leaks is non-trivial as they may depend on complex interactions/interleavings between several goroutines, or otherwise rare runtime conditions. Several proposed static analysis techniques [1, 2, 3] are prone to imprecision, both reporting false positives or incurring false negatives. Other proposals such as goleak employ dynamic analysis during testing, which may reveal several blocking errors, but their efficacy depends on the comprehensive coverage of code paths and thread schedules. Exhaustive coverage is infeasible at a large scale; for example, certain configuration flags changing code paths in production are not necessarily tested by unit-tests. 

<https://www.ardanlabs.com/blog/2018/11/goroutine-leaks-the-forgotten-sender.html>

Concurrent programming allows developers to solve problems using more than one path of execution and is often used in an attempt to improve performance. Concurrency doesn’t mean these multiple paths are executing in parallel; it means these paths are executing out-of-order instead of sequentially. Historically, this type of programming is facilitated using libraries that are either provided by a standard library or from 3rd party developers.

Concurrent programming allows developers to solve problems using more than one path of execution and is often used in an attempt to improve performance. Concurrency doesn’t mean these multiple paths are executing in parallel; it means these paths are executing out-of-order instead of sequentially. Historically, this type of programming is facilitated using libraries that are either provided by a standard library or from 3rd party developers.

**Leaking Goroutines**

When it comes to memory management, Go deals with many of the details for you. The Go compiler decides where values are located in memory using escape analysis. The runtime tracks and manages heap allocations through the use of the garbage collector. Though it’s not impossible to create memory leaks in your applications, the chances are greatly reduced.

A common type of memory leak is leaking Goroutines. If you start a Goroutine that you expect to eventually terminate but it never does then it has leaked. It lives for the lifetime of the application and any memory allocated for the Goroutine can’t be released. This is part of the reasoning behind the advice **Never start a goroutine without knowing how it will stop**.

**Never start a goroutine without knowing how it will stop**

<https://dave.cheney.net/2016/12/22/never-start-a-goroutine-without-knowing-how-it-will-stop>

In Go, goroutines are cheap to create and efficient to schedule. The Go runtime has been written for programs with tens of thousands of goroutines as the norm, hundreds of thousands are not unexpected. But goroutines do have a finite cost in terms of memory footprint; you cannot create an infinite number of them.

Every time you use the go keyword in your program to launch a goroutine, you must know how, and when, that goroutine will exit. If you don’t know the answer, that’s a potential memory leak.

Consider this trivial code snippet:

```go
ch := somefunction()
go func() {
        for range ch { }
}()
```

This code obtains a channel of int from somefunction and starts a goroutine to drain it. When will this goroutine exit? It will only exit when ch is closed. When will that occur? It’s hard to say, ch is returned by somefunction. So, depending on the state of somefunction, ch might never be closed, causing the goroutine to quietly leak.

In your design, **some goroutines may run until the program exits**, for example a background goroutine watching a configuration file, or the main conn. Accept loop in your server. **However, these goroutines are rare enough I don’t consider them an exception to this rule.**

Every time you write the statement go in a program, you should consider the question of how, and under what conditions, the goroutine you are about to start, **will end**.

### References

- <TBA>