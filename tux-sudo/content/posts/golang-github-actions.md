---
title: "Go: Useful Development Workflows"
date: 2019-12-23T05:32:57Z
tags: ["github", "go", "golang", "actions", "goreleaser", "taskfile", "sonar"]
categories: ["Ops", "Development"]
---

Code Coverage, Static Code Analysis, Tests, Releases? What else? Probably a few more things, but you get the point. These are all things that are expected to be part of a modern day development workflow. While this is becoming the norm in companies, a lot of pet open-source projects still skimp on these things. And frankly, I did too. I wasn't going to bother hosting my own Jenkins behemoth just so my soon-to-be abandoned project could run a few builds. But with practically everything available as a SaaS (and in most cases, free for open source projects), I had no excuse anymore.
Travis/Circle/GitlabCI/AzurePipelines etc. were already quite common and with GitHub's entry into the fray, there really is no reason not to have atleast a simple CI workflow on your project.

What I want to talk about in this post however, is about going beyond the regular build and test workflow. Most of the steps below can be extrapolated to pretty much any language, but I'll primarily be operating with Go. All the tools highlighted below cost nothing to operate on open source projects and require no self-hosting.

### Build Automation Tool

Pretty straight forward. If you like writing Makefiles, go ahead and treat yourself. I personally don't. I have a new favourite in [Task](https://taskfile.dev). It feels very clean and simple. Ships as a single binary and can be easily put inside any public CI system.
Here's a sample `Taskfile`:

```yml
# https://taskfile.dev

version: "2"

tasks:
  build-client:
    cmds:
      - mkdir release || true
      - go build -o release/bottle-cli ./cmd/client
  build-server:
    cmds:
      - mkdir release || true
      - go build -o release/crate ./cmd/server
  build:
    cmds:
      - task: build-server
      - task: build-client
  test:
    cmds:
      - go fmt ./...
      - golangci-lint run
      - go test -v ./...
  sonar:
    cmds:
      - sonar-scanner -Dsonar.login=$SONAR_TOKEN
```

Pretty easy to tell what is going on here. The tasks can then simple be run like `task build`. Again, if you'd like to stick to Make or any other language specific tool, do that instead.

### Static Analysis / Code Quality

Lots of candidates here. I'm going to highlight [SonarCloud](https://sonarcloud.io/about). Again, totally free for open source projects and has been a common feature of companies I've worked with.
You need to install the sonar-scanner to be able to scan your projects. This is easily doable on most CI systems (see example later for GitHub Actions) and a simple configuration file which resembles:

```ini
# Organization and project keys are displayed in the right sidebar of the project homepage
sonar.organization=tchaudhry91-github
sonar.projectKey=tchaudhry91_bottle
sonar.host.url=https://sonarcloud.io

sonar.exclusions=crate/pb/crate.pb.go
```

You can setup the organization, projectKey and a grab a sonar token on <https://sonarcloud.io>.

I'm not going to go deep into all the things you can do with Sonar, but it's really easy to get started and get some nice quality gates and badges set-up.

Other quality/linting checks tools can also be integrated as required. [GolangCI-Lint](https://github.com/golangci/golangci-lint) is another quite nice tool that I use in most of my repos. See example [here](https://github.com/tchaudhry91/bottle/blob/master/.golangci.yml).

### Releases

This, I'm going to deliberately keep language specific because of how widely it varies. In short, I've never had to look beyond [GoReleaser](https://github.com/goreleaser/goreleaser) for Go. This directly creates a github release based on git tags and also compiles binaries for specified platforms. Lots of knobs available to tweak, but something basic as below should suffice in most cases:

```yaml
# This is an example goreleaser.yaml file with some sane defaults.
# Make sure to check the documentation at http://goreleaser.com
before:
  hooks:
    # you may remove this if you don't use vgo
    - go mod tidy
    # you may remove this if you don't need go generate
    - go generate ./...
builds:
  - id: "crate"
    main: ./cmd/server/main.go
    binary: crate
    env:
      - CGO_ENABLED=0
  - id: "bottle"
    main: ./cmd/client/main.go
    binary: bottle
    env:
      - CGO_ENABLED=0
archives:
  - replacements:
      darwin: Darwin
      linux: Linux
      windows: Windows
      386: i386
      amd64: x86_64
checksum:
  name_template: "checksums.txt"
snapshot:
  name_template: "{{ .Tag }}-next"
changelog:
  sort: asc
  filters:
    exclude:
      - "^docs:"
      - "^test:"
```

Now, we'll trigger the release whenever a Git tag resembling "v*" gets pushed.

## Bind it together

I'm going to use the example of [this project](https://github.com/tchaudhry91/bottle) that I started a while ago. We'll see here how to tie up all the tools discussed above and have GitHub actions do it's job.

I'm going to be writing two distinct workflows. The `CI workflow` that runs on every push/pull request and does all the quality checks and tests. Additionally, there will be a `release workflow` that will be run only when a specific tag gets pushed.

CI Workflow

```yaml
name: CI
on: [push, pull_request]

jobs:
  ci:
    name: CI
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v1
      - uses: actions/setup-go@v1
        with:
          go-version: "1.13"
      - run: curl -sL https://taskfile.dev/install.sh | sh
      - run: curl -sfL https://install.goreleaser.com/github.com/golangci/golangci-lint.sh | sh -s v1.21.0
      - name: Test
        run: ./bin/task test
        env:
          PATH: $PATH:./bin
          CGO_ENABLED: 0
      - name: Build
        run: ./bin/task build
        env:
          CGO_ENABLED: 0
      - name: Sonar Scan
        uses: sonarsource/sonarcloud-github-action@master
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

and a similar Release workflow but with a different `on` condition:

```yaml
name: Release
on:
  push:
    tags:
      - "v*"

jobs:
  release:
    name: Release
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v1
      - run: curl -sL https://taskfile.dev/install.sh | sh
      - run: curl -sfL https://install.goreleaser.com/github.com/golangci/golangci-lint.sh | sh -s v1.21.0
      - run: curl -sfL https://install.goreleaser.com/github.com/goreleaser/goreleaser.sh | sh
      - name: Test
        run: ./bin/task test
        env:
          PATH: $PATH:./bin
          CGO_ENABLED: 0
      - name: Build
        run: ./bin/task build
        env:
          CGO_ENABLED: 0
      - name: Sonar Scan
        uses: sonarsource/sonarcloud-github-action@master
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      - name: Release
        run: ./bin/goreleaser
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

Supply the `SONARY_TOKEN` secret under your github project settings, add both these files into your `.github/workflows` directory and see the magic happen!

There's a lot more you can do with your workflows. Security scans, deployments, perhaps anicillary build and test services (see [this](https://blog.tux-sudo.com/posts/ci-docker-dbs/) post for more details on how to spin databases for testing in CIs). Lots of possibilities and not a $ spent. Have fun!
