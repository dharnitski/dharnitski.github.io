---
layout: post
title: "Skip tests in Go"
categories: 
tags: Go TDD
excerpt: Different types of tests require different schedules. We run tests for our code as soon as we make changes, after that we run the whole test suite to commit code to  the source control. Database modification requires running DB tests. How can we control what tests to run?   
---
## Run all the tests

Before we dive into managing tests let's remind how to run the test in GO.

This command line runs all the tess in all project packages:

    go test ./...

VS Code with [Go plugin](https://code.visualstudio.com/docs/languages/go) provides the command called `Go: Test All Packages in Workspace` to do the same.

## Run tests for current package

We can control which tests to run by scoping them by package.

Run tests in current package:

    go test

Another option to do the same is to use `.` (dot) to point to current folder:

    go test .

With VS Code we can run tests in current package upon file save event. Check [Environment Setup]({{ site.baseurl }}{% post_url 2019-04-08-setup-go-on-mac %}) instructions to see how to properly configure it if you practice TDD.

## Run tests for specific package

There are two ways to run package tests. One way is to use file system relative path to package folder. Second option is full package name.

This is how you can test specific package from command line:

    go test ./path/to/my/package
    go test github.com/account/project/path/to/my/package

Testing for multiple packages can be combined in one call:

    go test ./package1 ./package2
    go test github.com/account/project/package1  github.com/account/project/package2

It is also possible to combine package paths and names:

    go test ./package1 github.com/account/project/package2

## Skip long running tests

Often we want to separate short and long tests and `test` Go command supports `-short` flag built specifically for this purpose.

It is a little bit counter intuitive because we have to use flag called `-short` to exclude long tests, so let's talk about how that flag works.

Go implements logic below for `-short` flag:

* Flag is **not set** - run **long and short** tests
* `-short` slag is **set** - run **only short** tests

Option to run only long tests is not supported.

Syntax for using `-short` flag:

    go test -short

In code, we use `testing.Short()` function to see if `-short` flag is set:

```go
func TestTimeConsuming(t *testing.T) {
  if testing.Short() {
    t.Skip("skipping test in short mode.")
  }
  // rest of the test
}
```

Go conveniently reports skipped tests if we run tests in verbose mode (`go test -short -v`):

```bash
--- SKIP: TestTimeConsuming (0.00s)
    handler_test.go:261: skipping test in short mode.
```

## Skip test based on condition

`-short` flag is not the only option to skip tests. You can check any condition and skip test if condition returns true. Let's check a few samples.

Skip tests based on Environment variable:

```go
func TestEnvVar(t *testing.T) {
  if os.Getenv("TEST_WIP") != "" {
    t.Skip("Skipping not finished test")
  }
  // rest of the test
}
```

ENV variable can be set globally or run command with local variable:

    TEST_WIP=true go test -v

Skip tests based on custom flag:

```go
var foo string

func init() {
  flag.StringVar(&foo, "foo", "", "the foo bar bang")
}

func TestFoo(t *testing.T) {
  if foo == "bar" {
    t.Skip("Skipping for foo=bar")
  }
}
```

Run tests with flag:

    go test -foo=bar -v

## Build Constraints

Another convenient method to control tests is based on [Build Constrains](https://golang.org/pkg/go/build/#hdr-Build_Constraints).

A build constraint, also known as a build tag, it is a condition under which a file should be included in the package.

Build constrains can be used for any Go files and tests are not an exception.

Build constrains syntax is similar to comment with special `+build` word. Sample below shows how to set `sql` build constrain for file.

```go
// +build sql

package mypackage

import (
  "testing"
)

func TestFoo(t *testing.T) {
}
```

Constrains are injected into GO using `-tags` command line argument. String below injects `sql` constrain into test command:

    go test ./... -tags=sql

Command line above runs test for files without and with `// +build sql` constrain.

Although, if we run `go test ./...` (withoiut `sql` tag) our file is not compiled and tests are not executed.

Constrains syntax allows to use negative expressions and combine them using AND and OR operators:

    // +build linux,386 darwin,!cgo

corresponds to the boolean formula:

    (linux AND 386) OR (darwin AND (NOT cgo))

You can find information how to configure VS Code to enable testing constrains in my another post [Setup Go Dev environment]({{ site.baseurl }}{% post_url 2019-04-08-setup-go-on-mac %}).

## Disable test caching

In package list mode, `go test` caches successful package test
results to avoid unnecessary repeated running of tests.

Caching test results is not always desired behavior. For example, test can read input data from text file. Textual file is not compiled into test binary and therefore text file changes does not trigger cache eviction.

The idiomatic way to disable test caching explicitly
is to use `-count=1`:

    go test -count=1