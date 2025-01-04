+++
date = '2024-03-09T22:35:17+08:00'
draft = false
title = 'Bits and pieces of HTTP in Go'
+++

The benefit of being a young programming language is that its design will incorporate the latest software engineering needs from the ground up.
Concepts like multi-thread processing, building RESTful API become the first class citizens in Go.

In fact, this is part of the motivation in designing Golang, as one of the authors commented about older programming languages in the keynote talk 
[Go at Google: Language Design in the Service of Software Engineering](https://go.dev/talks/2012/splash.article)

`The problems introduced by multicore processors, networked systems, massive computation clusters, and the web programming model were being worked around rather than addressed head-on.`

`Go was designed to address a set of software engineering issues exposed in the construction of large server software, more than most general-purpose programming languages."`

As part of its manifestation, Go provides a neat `net/http` package to quickly build a performant HTTP server where incoming requests will be served by dedicated goroutines, 
which are scalable to handle growing throughput as the server hardware resources scales.

## Let's build one

The first thing we need is a **request router** or **multiplexer** that matches the URL of each incoming request to the right handler. Go has provided a default implementation and we can instantiate it in one line.

```go
svr := http.NewServeMux()
```

Then, we need to be aware of an important interface for the request **handler**.

```go
// This should look intuitive at a low level on how HTTP server actually works: 
// given a raw HTTP request object, process it however we need, and write 
// response back to the same connection.
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```

Go is a functional programming language, so we often write out the handling logic as a function and make use of an adaptor type `http.HandlerFunc`.

```go
// It essentially helps us implement the method to satisfy Handler interface
// so long as we provide a function that conforms with the below signature
type HandlerFunc func(ResponseWriter, *Request)

func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
    f(w, r)
}
```

Therefore, we can go ahead to implement our handler logic just as a pure function, and make it a handler by converting the function into a `HandlerFunc` type.

```go
okFunc := func(w http.ResponseWriter, r *http.Request) {
	// it will implicitly be set on the first call to "Write"
	w.WriteHeader(http.StatusOK)

	n, err := w.Write([]byte("ok handler responded OK!"))
	if err != nil {
		w.WriteHeader(http.StatusInternalServerError)
	}
	log.Printf("written %d bytes to the response\n", n)
}

okHandler := http.HandlerFunc(okFunc)
```

Now, we already have all the pieces needed to run a functioning HTTP server!

{{<details title="To see the full source code" >}}
```go
package main

import (
	"log"
	"net/http"
)

func main() {
	okFunc := func(w http.ResponseWriter, r *http.Request) {
		// it will implicitly be set on the first call to "Write"
		w.WriteHeader(http.StatusOK)

		n, err := w.Write([]byte("ok handler responded OK!"))
		if err != nil {
			w.WriteHeader(http.StatusInternalServerError)
		}
		log.Printf("written %d bytes to the response\n", n)
	}

	okHandler := http.HandlerFunc(okFunc)

	svr := http.NewServeMux()
	svr.Handle("/ok", okHandler)
	// serveMux also provides a shortcut to take in the function directly
	// and convert it to handler for you
	svr.HandleFunc("ok_again", okFunc)
	// the http package offers a few default handlers as well
	svr.Handle("/", http.NotFoundHandler())

	// receive incoming request from TCP network address, 
	// and hand it to serveMux to route to the closest
	// matching handler for processing
	err := http.ListenAndServe(":8080", svr)
	if err != nil {
		log.Printf("http server exited: %v\n", err)
	}
}
```
{{</details>}}
&nbsp;

## Middleware

As the HTTP service grows more complex, we may reach a point where request pre-processing or post-processing becomes necessary.
We wish to chain up a series of operations in a predefined order. We can build the middleware pattern with what `net/http` package has offered.

The full execution order will look like:

```text
             ( can be swapped )
request —> serveMux —> middleware request handler —> handler function
                                                           |
response <———————————— middleware response handler <———————|
```

The key idea of a middleware function is to:
>1. take in the handler that will execute next, 
>2. define some work on top of the input handler, 
>3. hand over the control to the next handler by calling its`ServeHTTP`method
>4. return the new handler with the middleware work wrapped around

Here are some practical examples:

```go
// stop the invalid request from proceeding further to optimise 
// the server load
func middlewareDropInvalid(next http.Handler) http.Handler {
	f := func(w http.ResponseWriter, r *http.Request) {
		if contentType := r.Header.Get("Content-Type"); contentType == "" {
			http.Error(w, "no content type specified", http.StatusBadRequest)
			return
		}
		next.ServeHTTP(w, r)
	}
	return http.HandlerFunc(f)
}

// produce access logs for traceability
func middlewareLogging(next http.Handler) http.Handler {
	f := func(w http.ResponseWriter, r *http.Request) {
		log.Printf("remote client address: %s\n", r.RemoteAddr)
		next.ServeHTTP(w, r)
	}
	return http.HandlerFunc(f)
}

// set the response header when the response is in JSON format
func middlewareHeader(next http.Handler) http.Handler {
	f := func(w http.ResponseWriter, r *http.Request) {
		next.ServeHTTP(w, r)
		w.Header().Set("Content-Type", "application/json")
	}
	return http.HandlerFunc(f)
}
```

Depending on whether the middleware is handler specific or global, we can chain them together at either the serveMux level or individual handler level.

```go
// global middleware - always drop invalid requests and log access before routing
//                     requests to the handler
svr := http.NewServeMux()
wrappedSvr := middlewareDropInvalid(middlewareLogging(svr))

// handler specific middleware - set header only in this path
wrappedSvr.Handle("/ok", middlewareHeader(handlerFunc))

err := http.ListenAndServe("localhost:8080", wrappedSvr)
if err != nil {
	log.Printf("http server exited: %v\n", err)
}
```

## Concurrency

Just to reiterate on the point of having builtin multi-thread programming in Go. It might be worth to take a closer look at what happens under the hood in`http.ListenAndServe`.

This function basically sets up the program to listen at the specified TCP address, and spin up **a dedicated goroutine** to serve each incoming request.

The Go runtime scheduler will then be responsible for distributing these goroutines to run on OS threads across CPU cores, which is defined by `GOMAXPROCS` environment variable and is default
to the number of cores available on the host.

```go
// Go 1.22.2 source code: net/http/server.go
func ListenAndServe(addr string, handler Handler) error {
	server := &Server{Addr: addr, Handler: handler}
	return server.ListenAndServe()
}

// which then calls
func (srv *Server) ListenAndServe() error {
	// redacted for brevity...
	ln, err := net.Listen("tcp", srv.Addr)
	if err != nil {
		return err
	}
	return srv.Serve(ln)
}

// now the fun parts
func (srv *Server) Serve(l net.Listener) error {
	// redacted for brevity...
	ctx := context.WithValue(baseCtx, ServerContextKey, srv)
	for {
		rw, err := l.Accept()
		if err != nil {
            // redacted for brevity...
			return err
        }
		connCtx := ctx
		if cc := srv.ConnContext; cc != nil {
			connCtx = cc(connCtx, rw)
			if connCtx == nil {
				panic("ConnContext returned nil")
			}
		}
		c := srv.newConn(rw)
		c.setState(c.rwc, StateNew, runHooks)
		// ** Each connection context is concurrently served in a goroutine, 
		// which internally calls the serveMux's ServeHTTP method
		go c.serve(connCtx)
    }
}
```

And if we look at the source code of serveMux, it routes the request to the appropriate handler as we have walked through earlier.

```go
// Go 1.22.2 source code: net/http/server.go
func (mux *ServeMux) Handler(r *Request) (h Handler, pattern string) {
	// Side notes: Go introduced breaking changes in the URL pattern matching,
	//             but it allows flag GODEBUG=httpmuxgo121 to restore older behaviours
	//             https://go.dev/blog/routing-enhancements
	if use121 {
		return mux.mux121.findHandler(r)
	}
	h, p, _, _ := mux.findHandler(r)
	return h, p
}
```

## Going further

The builtin `net/http` package in Go has already offered us the ability to build rather involved HTTP server with relatively small amount of code.
However, the real production application often poses **more challenges** that require an uplift from the standard library. 

* The `Handler` interface requires to process raw HTTP request object. In the case when we need to differentiate methods`GET`,`POST`,`PUT`for the same URl path, the handler code will get very cluttered.
* It takes extra steps to extract resource identifier from the URL path with`http.StripPrefix()`or`strings.TrimPrefix()`. 

Luckily, the above two are addressed in the Go 1.22 release, which supports pattern matching such as

```go
// the routing will be based on request method as well
svr.Handle("GET /resources/{id}", getHandler)
svr.Handle("POST /resources/{id}", postHandler)

// the resource identifier can be retrieved in the handler 
// from the request object using the key
id := r.PathValue("id")
```

There can be performance related issue as well. 

* If the request payload contains large JSON data (e.g. OpenRTB bid request can have more than 150 fields), the parsing overhead can be substantial.
* The default `serveMux` implements a simple routing (Go 1.21 and before) by sorting all registered paths from the longest to shortest. And the pattern matching is performed sequentially. This can incur latency overhead when the server routing is complex (e.g. having over 100+ registered patterns).

{{<details title="Source code" >}}
```go
// Go 1.21.11 soruce code: net/http/server.go
type ServeMux struct {
	mu    sync.RWMutex
	m     map[string]muxEntry
	es    []muxEntry // slice of entries sorted from longest to shortest.
	hosts bool       // whether any patterns contain hostnames
}

func appendSorted(es []muxEntry, e muxEntry) []muxEntry {
	n := len(es)
	i := sort.Search(n, func(i int) bool {
		return len(es[i].pattern) < len(e.pattern)
	})
	if i == n {
		return append(es, e)
	}
	// we now know that i points at where we want to insert
	es = append(es, muxEntry{}) // try to grow the slice in place, any entry works.
	copy(es[i+1:], es[i:])      // Move shorter entries down
	es[i] = e
	return es
}

func (mux *ServeMux) match(path string) (h Handler, pattern string) {
	// Check for exact match first.
	v, ok := mux.m[path]
	if ok {
		return v.h, v.pattern
	}

	// Check for longest valid match.  mux.es contains all patterns
	// that end in / sorted from longest to shortest.
	for _, e := range mux.es {
		if strings.HasPrefix(path, e.pattern) {
			return e.h, e.pattern
		}
	}
	return nil, ""
}
```
{{</details>}}

In those cases, we can look out for open source frameworks to help us. Some popular ones that I personally like are: 
[Gin](https://github.com/gin-gonic/gin) and  [Echo](https://github.com/labstack/echo)

They make use of techniques such as [fastjson](https://github.com/valyala/fastjson) to avoid reflection and allocating extra space for structs to speed up parsing JSON data;
and Trie-based routing that breaks up URL segments into hierarchical tree nodes for faster lookup.

