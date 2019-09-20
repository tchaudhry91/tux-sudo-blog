---
title: "Go: Spin-Up Databases for CI Testing"
date: 2019-09-20T06:19:22Z
tags: ["docker", "github", "actions", "ci", "go"]
ccategories: ["Development"]
project_url: "https://github.com/tchaudhry91/spinme"
---

One thing has always bothered me while writing tests is the lack of a real datasource to run tests against. Most projects I've worked with in the past have either mocked responses from a datastore or used a "common" datastore to perform tests against. While mocks are good for quick unit tests, I still prefer using a "real" datasource, especially for integration tests.

While taking Bill Kennedy's Ultimate Go training a while ago, I saw Bill recommend a testing approach which involved "spinning up" a database container right from within your test code, run your tests, and clean-up. See [this](https://github.com/ardanlabs/service/blob/master/internal/platform/database/databasetest/docker.go) for reference. I love this approach. It provides a wonderful way to do everything from within `go test` without meddling with external scripts, thereby providing much greater interoperability. My only gripe with this is the fact that it uses `os.exec` to command communicate with the docker daemon. The approach is sound, however, I find forking out to use docker-cli to be a bit clunky. Docker's client for go is pretty good and offers imho, a better way to achieve the same. It adds a dependency to the code, however, I believe that is a good thing. The real dependency here is the daemon on the underlying system. Taking a dependency in Go code, makes the management of the API easier and offers more control than relying on the cli api silently. Moreover, if you only use the dependency in your `_test.go` files, it won't impact the final build binary (or that's my understanding atleast, someone correct me if I'm wrong). In most cases the exec approach is perhaps good enough and the right thing to do, but I have an alternative.

Recently, I wrote a small project with a bunch of helpers and an approach to achieve the same using the docker client directly.
[Spin Me!](https://github.com/tchaudhry91/spinme) is a simple tool that can be used to "spin up" docker database containers followed by auto-configuration, right from within your go code.
See the example below for [Postgresql](https://godoc.org/github.com/tchaudhry91/spinme/spin#example-Postgres):

```go
out, err := spin.Postgres(context.Background(), nil)
if err != nil {
    fmt.Println(err)
    return
}
defer spin.SlashID(context.Background(), out.ID)
// Give postgres a few seconds to boot-up, sadly there is no "ready" check yet
time.Sleep(5 * time.Second)
connStr, err := spin.PostgresConnString(out)
if err != nil {
    fmt.Println(err)
    return
}
db, err := sql.Open("postgres", connStr)
if err != nil {
    fmt.Println(err)
    return
}
defer db.Close()
err = db.Ping()
if err != nil {
    fmt.Println(err)
    return
}
fmt.Println("Connected!")
```

The above example goes through the entire cycle of going through the entire spin-up, connect, clean-up cycle. See the entire API and more examples here [Go Doc](https://godoc.org/github.com/tchaudhry91/spinme/spin).

Most CI systems ship with an environment that allows applications to create docker containers and this allows `go test` to spawn it's own database on each integration run in a commpletely clean environment.

As a bonus, the package comes with a small command-line utility to manage containers.

```bash
spinme -h
SpinMe is a wrapper around docker to run common applications.
  Use this to easily create dependent services such as databases.

Usage:
  spinme [command]

Available Commands:
  down        Bring down the given container
  help        Help about any command
  status      Status shows the list of all running services spun via spinme
  up          Start a particular service

Flags:
      --db string   Database for local storage (default "/home/tchaudhry/.spinme")
  -h, --help        help for spinme

Use "spinme [command] --help" for more information about a command.
```

See the project [repo](https://github.com/tchaudhry91/spinme) for more details.

Cheers and happy testing.
