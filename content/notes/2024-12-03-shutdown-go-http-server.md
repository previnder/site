---
title: How to shutdown a Go HTTP server gracefully
date: 2024-12-03
slug: /shutdown-go-http-server
---

The following code snippet of Go starts a simple HTTP server that responds with “Hello World” to every request it gets.

```go
func main() {
	server := &http.Server{
		Addr: ":8080",
		Handler: http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			w.Write([]byte("<html><body><h1>Hello World</h1></body></html>"))
		}),
	}
	if err := server.ListenAndServe(); err != nil {
		log.Fatal(err)
	}
}
```

But there’s a small problem: the `Server.ListenAndServe` method blocks until the server terminates—until, to be more specific, one of `Server.Close` or `Server.Shutdown` methods is called, or it encounters a fatal error. In this example, the server never terminates (unless there’s an error), and one would have to terminate the process manually from the outside, by for instance hitting `Ctrl+C` and sending an interrupt [signal](<https://en.wikipedia.org/wiki/Signal_(IPC)>) to the process.

But that would also terminate any HTTP requests that are being read or responses being written at that moment, mid-flight. This is not of course, especially for a production service, the optimal way this should work.

The `Server.Shutdown` method, in fact, exists for this particular use case, for shutting down the server gracefully, after waiting until all connections have returned to idle. In the following snippet, the program waits until it receives a `SIGINT` or a `SIGKILL` signal, and when it does, it shuts down the HTTP server gracefully before exiting the program.

(If you’re unfamiliar with [Go’s channels](https://go.dev/tour/concurrency/2) or [context.Context](https://pkg.go.dev/context), you most definitely won’t understand how the following works. But if you’re familiar with those two things, the following should be easy to read and understand.)

```go
func main() {
	server := &http.Server{
		Addr: ":8080",
		Handler: http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			w.Write([]byte("<html><body><h1>Hello World</h1></body></html>"))
		}),
	}

	// From the documentation of the signal package:
	//
	// NotifyContext returns a copy of the parent context that is marked done
	// (its Done channel is closed) when one of the listed signals arrives,
	// when the returned stop function is called, or when the parent context's
	// Done channel is closed, whichever happens first.
	//
	// ....
	ctx, stop := signal.NotifyContext(context.Background(), os.Interrupt, os.Kill)
	defer stop()

	go func() {
		// ListenAndServe always returns a non-nil error. And if it returns as a
		// result of a call to Server.Close or Server.Shutdown, it returns
		// http.ErrServerClosed.
		if err := server.ListenAndServe(); err != nil && !errors.Is(err, http.ErrServerClosed) {
			log.Printf("ListenAndServe (%s) error: %v\n", server.Addr, err)
		}
	}()

	// Blocks until the process receives a SIGINT or SIGKILL signal.
	<-ctx.Done()

	// From the documentation of the http package:
	//
	// Shutdown gracefully shuts down the server without interrupting any
	// active connections. Shutdown works by first closing all open
	// listeners, then closing all idle connections, and then waiting
	// indefinitely for connections to return to idle and then shut down.
	// If the provided context expires before the shutdown is complete,
	// Shutdown returns the context's error, otherwise it returns any
	// error returned from closing the [Server]'s underlying Listener(s).
	//
	// When Shutdown is called, [Serve], [ListenAndServe], and
	// [ListenAndServeTLS] immediately return [ErrServerClosed]. Make sure the
	// program doesn't exit and waits instead for Shutdown to return.
	//
	// ....
	if err := server.Shutdown(context.Background()); err != nil {
		log.Printf("Error gracefully shutting down server: %v\n", err)
	}
	log.Println("Server exited gracefully")
}
```

Of course, more complex behavior than this can be implemented, which in some cases might be quite useful. For example, receiving a second interrupt signal, while the program waits until `server.Shutdown` to return, could be made to exit the process immediately; or there could be a time limit for how long the program waits for `server.Shutdown` to return; etc.
