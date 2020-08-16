---
title: "Production Grade Web Services with Go - Transports"
date: 2020-07-04T15:30:44+05:30
tags: ["development", "go", "webservice", "tutorial"]
categories: ["Development"]
---

Note : This is part three of a series of posts describing how to write "Production Grade Webservices in Go". Here's [Part - 1, The Service ](/posts/production-grade-svc-1/) and [Part - 2 The Store ](/posts/production-grade-svc-2) if you haven't read those.

We've reached a point where we have properly laid out the business logic and the storage implementation for our service. Now, we're going to move on and talk about **transports**. As the name suggests, a transport is essentially transporting data over the network in a pre-defined format. JSON over HTTP is one such transport. gRPC is another. We're going to start with JSON over HTTP simply because of how popular it is. I may write a separate post that adds a gRPC transport to this very service later, but for now, HTTP-JSON. 

Go comes with a very nice and usable HTTP server right out of the box. We'll use the server that `net/http` provides us with a third-party router (`gorilla/mux`) from some added features.

Let's define a struct that's going to hold all of this together. The bare-minimum looks something like this:

```go
// RainbowHTTP is an HTTP server for the underlying RainbowService
type RainbowHTTP struct {
	svc    RainbowService
	server *http.Server
	router *mux.Router
}
```

The business-logic is contained in `svc`. The `server` is the actual HTTP Server from the `net/http` package. And finally, we've got our router from `gorilla/mux`.
I'm sure you've noticed that this is where our service goes from a Go interface to a **webservice**. The transport is going to glue all the things together and do the proper plumbing. 

Alright, let's define some methods to deal with HTTP requests. In Go, a method that follows the signature `func(w http.ResponseWriter, r *http.Request)` is potentially capable of serving web requests. Read more about it [here](https://golang.org/pkg/net/http/#HandlerFunc). So let's write a method with a signature like this:

```go
func (s *RainbowHTTP) GetHash() http.HandlerFunc
```

There's a reason why we actually *return* a HandlerFunc instead of making this method a handler func itself. Doing it this way, allows us to do some setup before the actual handler begins. Mat Ryer explains this concept in his famous [post](https://pace.dev/blog/2018/05/09/how-I-write-http-services-after-eight-years.html).

The handler that we return needs to do a bunch of stuff:
- Extract the business logic request from HTTP params/body to a single struct.
- Call the underlying business logic service with the appropriate parameters.
- Form and return a suitable response with proper error handling.

All three of the above can be seen the code below. Check out the full function (along with a helper utility) definition and see if you can decipher what it's doing.

```go
// respond is a internal utility to set proper HTTP responses
func (s *RainbowHTTP) respond(w http.ResponseWriter, req *http.Request, data interface{}, statusCode int, err error) {
	w.WriteHeader(statusCode)
	// Log the supplied error later if it's not nil
	if data != nil {
		err := json.NewEncoder(w).Encode(data)
		if err != nil {
			// Log this later
		}
	}
}

// GetHash returns the HandlerFunc for the Hash route.
func (s *RainbowHTTP) GetHash() http.HandlerFunc {
	// Do some potential setup here.
	// Not required in this case.

	return func(w http.ResponseWriter, req *http.Request) {
		// These structs are limited to the handler scope.
		// They may seem overly verbose in this case, but they're worth it when you're dealing with more complex requests.
		type Request struct {
			Str string `json:"str,omitempty"`
		}
		type Response struct {
			Hash string `json:"hash,omitempty"`
			Err  string `json:"err,omitempty"`
		}

		r := Request{}

		qparams := req.URL.Query()
		if _, ok := qparams["str"]; !ok {
			s.respond(w, req, nil, http.StatusBadRequest, errors.New("No string supplied"))
			return
		}

		// Grab the first "str" query parameter
		r.Str = qparams["str"][0]

		resp := Response{}

		hash, err := s.svc.Hash(r.Str)
		if err != nil {
			resp.Err = err.Error()
			s.respond(w, req, resp, http.StatusInternalServerError, err)
			return
		}
		resp.Hash = hash
		s.respond(w, req, resp, http.StatusOK, nil)

	}
}
```

A similar method, `GetReverseHash` should complete the service. It's going to be identical, so take a try at it yourself. You'll find it in the final code anyway.

Great, so we have two methods that can tied to HTTP routes. To do that, we'll use the `router` from the struct above and define the routes there. I like to do this in a separate file (usually called `routes.go`) so that I can see an overview of all the routes in my service in one place. It also serves as a good starting place for anyone new to this code. In this service we have two simple *GET* methods but the same logic applies to any method.

```go
// routes registers the handlers to the specific routes
func (s *RainbowHTTP) routes() {
	s.router.HandleFunc("/hash", s.GetHash()).Methods("GET")
	s.router.HandleFunc("/reverse", s.GetReverseHash()).Methods("GET")
}
```

Almost there now. Like any good server, we'll need a `Start` method and `Shutdown` method. These aren't too complex and something as basic as the following works well for me:

```go
// Start begins listening for requests on the bindAddr. Blocks.
func (s *RainbowHTTP) Start() error {
	return s.server.ListenAndServe()
}

// Shutdown gracefully terminates the server
func (s *RainbowHTTP) Shutdown(ctx context.Context) error {
	return s.server.Shutdown(ctx)
}
```

If you're confused about the `context` parameter, you may want to read a little bit about what it does [here](https://golang.org/pkg/context/). It'll play an important role later. The rest is fairly straight forward. We use the `ListenAndServe` method that the Go std lib provides us (no need for Tomcat etc).

Let's finish off with a constructor for our server and we'll be done.

```go
// NewRainbowHTTP returns a new HTTP server for the Rainbow Service.
func NewRainbowHTTP(listenAddr string, svc RainbowService) *RainbowHTTP {
	router := mux.NewRouter()

	// Build the server
	server := &RainbowHTTP{
		svc:    svc,
		router: router,
		server: &http.Server{
			Addr:    listenAddr,
			Handler: router,
		},
	}

	// Initialize the routes
	server.routes()

	return server
}
```

We have a working webservice! (far from production ready, but still). But is it really working? Only one way to tell. Tests!
Testing webservices is something that I've usually done externally (via something like Postman). That was until I started writing Go services. Go has a pretty nice testing mechanism built straight into the standard library and that's what we'll use.
Here's a snippet to show how the `httptest` package works:

```go
func TestHashHandler(t *testing.T) {
	// This is a method to demonstrate how to test the handlers.
	// Should ideally be a table driven test with multiple requests later.
	server := getServer()
	rr := httptest.NewRecorder()

	// Create a request
	req, err := http.NewRequest("GET", "/hash", nil)
	if err != nil {
		t.Errorf("Failed to create an HTTP Request: %v", err)
		t.FailNow()
	}
	q := req.URL.Query()
	q.Add("str", "thisisastring")
	req.URL.RawQuery = q.Encode()

	server.Handler().ServeHTTP(rr, req)

	if status := rr.Code; status != http.StatusOK {
		t.Errorf("Handler returned non OK status: got %d, want %d", rr.Code, http.StatusOK)
		t.FailNow()
	}

	// Check the data
	expectedHash := "572642d5581b8b466da59e87bf267ceb7b2afd880b59ed7573edff4d980eb1d5"
	resp := struct {
		Hash string
		Err  string
	}{}
	err = json.NewDecoder(rr.Body).Decode(&resp)
	if err != nil {
		t.Errorf("Error Decoding Response JSON:%v", err)
		t.FailNow()
	}
	if resp.Hash != expectedHash {
		t.Errorf("Wrong hash returned in response. Got - %s Want - %s", resp.Hash, expectedHash)
	}
}
```

While this is not a great test, it has been kept simple intentionally to demonstrate the point. It creates a request and then tests against the response against the expected by actually using the same handler that our HTTP server would. 
Let's give it a spin:

```
17:35:27 tchaudhry@QuasExort93 rainbow master ? go test -v ./...                                                                                                                              1 ↵
=== RUN   TestServiceHash
--- PASS: TestServiceHash (0.00s)
=== RUN   TestServiceHashReverse
--- PASS: TestServiceHashReverse (0.00s)
=== RUN   TestInMemStore
--- PASS: TestInMemStore (0.00s)
=== RUN   TestHashHandler
--- PASS: TestHashHandler (0.00s)
PASS
ok      github.com/tchaudhry91/rainbow/service
```

Mmmm..Nice. Play around with the tests for a while and see if you can write them in a more table driven way.

Now, that we have our transport done. We only need to tie all of this together in our `main` function. Let's break here and do that next time.

Here's the [git tree](https://github.com/tchaudhry91/rainbow/commit/8b776b609d60419191be454a36531b018dcc864d) for the code that we've already written.