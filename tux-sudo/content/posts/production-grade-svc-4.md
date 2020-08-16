---
title: "Production Grade Web Services with Go - Tying it Together"  
date: 2020-08-16T15:02:39+05:30
tags: ["development", "go", "webservice", "tutorial"]
categories: ["Development"]
---

Note : This is part four of a series of posts describing how to write "Production Grade Webservices in Go". Here's [Part - 1, The Service ](/posts/production-grade-svc-1/), [Part - 2, The Store ](/posts/production-grade-svc-2/), [Part - 3, Transports](/posts/production-grade-svc-3).

In part-3, we built a working HTTP service that passed the tests. But we're still lacking the option to actually "run" it. While, this may appear like a trivial task, it's actually quite important to get this right. 
Something I feel rather strongly about is the need to avoid package level variables and global state in Go programs. Like plenty of other things, Peter Bourgon says it perfectly in [this post](https://peter.bourgon.org/blog/2017/06/09/theory-of-modern-go.html):

*tl;dr: magic is bad; global state is magic → no package level vars; no func init*

In my experience, while writing services, this principle gets violated most often in the `main` package. Especially with initializations and configuration. So, I'm going to present a simple method, where everything is initialized in `func main` itself and flows down from there. And by everything, I do mean everything. There is no magic. Unless there is an explicit parameter on something, it's not receiving that resource. In other words, all the dependencies are explicitly visible and injected from `func main`.

Starting off, we're attempt to initialize our HTTP Server (since that's the final goal). Through this, you'll realize everything you need to get the server up. Check out the constructor for the http server:

```go
func NewRainbowHTTP(listenAddr string, svc RainbowService) *RainbowHTTP 
```

Hm, so we need a `listenAddr`, which should probably be an input parameter. So, let's begin with that. There are plenty of flag/environment parsing libraries out there and any one of them should fit the bill as long as they don't violate the "no package scoped variables" principle. The one I prefer is [ff](https://github.com/peterbourgon/ff). It's extremely simple, yet effective. So let's write a simple flag/env parser to fetch our `listenAddr`.

```go
func main() {
	fs := flag.NewFlagSet("rainbow", flag.ExitOnError)
	var (
		listenAddr = fs.String("listen-addr", "localhost:8080", "listen address")
	)

	ff.Parse(fs, os.Args[1:],
		ff.WithEnvVarPrefix("RAINBOW"),
	)
}
```

Here, we populate the variable `listenAddr` based on the flag value (or the environment variable RAINBOW_LISTEN_ADDR if the flag isn't specified).
This can be expanded to all the configuration values you need to get the application up and running. If you have too many options, it's probably best to encapsulate this parsing in another function and call that from main instead. But again, no shared variables please, pass arguments, receive returns.

Let's move on to the second bit now. The transport requires an implementation of the `RainbowService` in addition to this listen address. This is our `RainbowSHA256Service` struct that we fleshed out earlier. Let's try to initialize that now:

```go
func NewSHA256RainbowService(store Store) *SHA256RainbowService {
	return &SHA256RainbowService{store}
}
```

Hm, this in turn requires an implementation of a `Store`. Alright let's get a store first. We could use either our `InMemStore` or the `RedisStore`, but it's probably better to go with Redis if we want persistance. Let's see what a store needs: 

```go
func NewRedisStore(addr string, password string, db int) *RedisStore
```

Only configuration parameters. Okay, this is stuff we can add to our flag set.

```go
	var (
		listenAddr = fs.String("listen-addr", "localhost:8080", "listen address")
		dbAddr     = fs.String("db-addr", "localhost:6379", "Database address")
		dbPassword = fs.String("db-password", "", "Database password")
		dbNumber     = fs.String("db-number", 1, "Database number")
	)
```
With this, we get a nice cascading effect. We have enough to initialize a store. Which means we have enough to initialize our RainbowService, which in turn will allow us to initialize the HTTP Transport. No magic. Everything flows directly and transparently. Let's tie it together:

```go
	store := service.NewRedisStore(*dbAddr, *dbPassword, *dbNumber)
	svc := service.NewSHA256RainbowService(store)
    server := service.NewRainbowHTTP(*listenAddr, svc)

    server.Start()
```

And there you go. That's a working `func main` for you. I also like to put this in a package of it's own. See the repository structure and the main file here at [this commit](https://github.com/tchaudhry91/rainbow/commit/e1d9cf089fd69c71d83ef1d9b94c088fc9addaf3) which shows the progress so far.

If you go ahead and build this with `go build -o rainbow-server ./cmd/server`, you'll have a working configurable service. Sure, it's awful quiet in the output and logging department (we'll cover that in Part 5 soon), but it works. Try it out.

There are a few more things to make this service proper. We need to make sure our service is interruptible, shuts down gracefully and displays some basic start-up information.
To do this, we must listen for Operating System signals that ask the application to terminate.

```go
    shutdown := make(chan error, 1
	interrupt := make(chan os.Signal, 1)
	signal.Notify(interrupt, os.Interrupt, syscall.SIGTERM)
```

The snippet above is going to create a new `interrupt` Channel and `signal.Notify` is going to post to that channel whenever an interrupt signal is received. We also create a `shutdown` channel to track fatal errors that should lead to the application shutting down.

So, we need to *listen* to this channel for signals AND run our server as before. This means concurrency, and that means go routines.

```go
	go func() {
		err = server.Start()
		shutdown <- err
	}()
```

Okay, that's one part handled. Our service is up and running. Now, let's listen to the channels:

```go
	select {
    case signalKill := <-interrupt:
        // log to taste
    case err := <-shutdown:
        // log to taste
	}
```

This select statement is going to block until one of two things happens. Either we receive an interrupt signal or we have an error on our shutdown channel. In both cases, we need to go ahead and shutdown our service gracefully, which is going to look something like this:

```go
	err = server.Shutdown(context.TODO())
	if err != nil {
        // log to taste
	}
```

The actual graceful shutdown is left the to the `shutdown` method on our transport. This exercise is to ensure that it gets called.

Also, there are logging statements that need to be sprinkled in. The library choice is up to you again, however, I like using [kit/log](https://github.com/go-kit/kit/tree/master/log).
Go ahead an initialize a logger, and fill out this the log statements. A sample implementation can be found in this [commit](https://github.com/tchaudhry91/rainbow/commit/12f33c643cfdb5ec723c69b0b32de8d148d64943) with a completed `func main`

All done! We're going to use the logger we initialized for request/response logging with **Middlewares** in the next post. See you!
