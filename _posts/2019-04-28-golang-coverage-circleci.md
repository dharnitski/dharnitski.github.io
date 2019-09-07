---
layout: post
title: "Code Coverage in Go"
categories: 
tags: Go TDD Coverage Coveralls CircleCI CI
excerpt: Getting Golang code coverage and posting to Coveralls.io is simple. Or not?  
---
## Coverage Challenge

Code coverage generation in Go is trivial. There is just one caveat. It is scoped to a single package we are testing.

`go test` command always work with one package. You can send multiple packages into it, but it will put them in queue and  will test these packages one by one. Package level scoping has no impact on test results, but significantly complicates project coverage calculation.

Quick recap how `go test` command runs tests:

1. Determine a list of packages to test. All these  packages will be tested one by one and every action below will be executed for each package.
1. If coverage is enabled, annotates the source code before compilation.
1. Recompile package along with any files with names matching the file pattern `*_test.go`.
1. As part of building a test binary, run `go vet` on the package, but that is not important for our topic.
1. Run tests against compiled code. If coverage is enabled, save coverage results.
1. If a package test passes, print only the final 'ok' summary line. If a package test fails, print the full test output.

Coverage is collected only for the current package if we run `go test` with `-cover` flag. Additional packages can be added using `-coverpkg` flag.

## Solution

Like many other problems, the solution for the coverage issue is implemented as an open source package - [github.com/ory/go-acc](https://github.com/ory/go-acc).

After `go-acc` package installation, correct coverage can be generated using command below:

    go-acc ./...

When we run `go-acc` it works as follows:

* Runs tests for the specified packages and collects coverage for the tested package and its dependencies.
* Transforms coverage information into correct format.
* Appends coverage lines to a single file making it easier to integrate with Coveralls.

## Final Result

CircleCI configuration to publish coverage:

```yml
version: 2
jobs:
  build:
    docker:
      - image: circleci/golang:1.12
    working_directory: /go/src/github.com/dharnitski/go-coverage-sample
    steps:
      - checkout
      - run:
          name: "Install Dependencies"
          command: |
            go get github.com/mattn/goveralls
            go get -u github.com/ory/go-acc
      - run:
          name: "Generate Coverage"
          command: |
            go-acc ./...
      - run:
          name: "Publish Coverage to Coveralls.io"
          command: |
            goveralls -coverprofile=coverage.txt -service semaphore -repotoken $COVERALLS_TOKEN
```

What we are doing here:

* Install Dependencies - loads and installs Go commands for calculating coverage and posting it to Coveralls.
* Generate Coverage - uses `go-acc` with default settings. File coverage is to be saved into `coverage.txt` file.
* Publish Coverage to Coveralls.io - this command does what its name tells us. It requires Coveralls access token to be stored in `COVERALLS_TOKEN` CircleCI Environment Variable.

## -coverpkg flag

Go `test` allows to specify packages we want to cover with tests with `-coverpkg` flag.

Command below tests all packages, calculates coverage for all packages and saves results into `coverage.txt`

    go test -coverpkg=./... -coverprofile=coverage.txt ./...

The only problem with this command, it does not work for situation when there are several `main` packages in your codebase including `vendor` folder.

You will know if that is the case for your project if you see an error similar to this one:

    duplicate symbol main.main (types 1 and 1) in main and /Users/user/Library/Caches/go-build/e5/e53413e54e9e7e079307427e09f0d60657fc6a7179fa93196aee80b0bc550578-d(_go_.o)

You can find working project in [github](https://github.com/dharnitski/go-coverage-sample)

Happy covering!
