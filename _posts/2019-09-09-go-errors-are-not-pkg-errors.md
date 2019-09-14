---
layout: post
title: "Can new Go errors wrapper replace pkg/errors?"
categories: 
tags: Go
excerpt: Go version 1.13 includes new functions to work with errors. These functions are expired by popular github.com/pkg/errors package. Let's see if new wrapper can replace existing library.  
---

[github.com/pkg/errors](https://github.com/pkg/errors) is a popular errors package that adds context to the failure path in a way that does not destroy the original value of the error. Dave Cheney explains in his [blog post](https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully) why it is not possible to add context and check error type with vanilla Go errors. But that was not possible till Go v1.13 was released. Now Go has built in support for errors wrapping in base library.

## How both wrappers work

I will refer `github.com/pkg/errors` package as external package to distinguish if from new standard Go errors.

Errors warped using `Wrap` function and original error extracted using function called `Cause` with external library:

```go
import "github.com/pkg/errors"

internal := errors.New("internal error")
// add additional context to an error
wrapped := errors.Wrap(internal, "wrapper")
// get original error
unwrapped := errors.Cause(wrapped)
```

In Go 1.13 you can do the same using `%w` formatting verb and `Unwrap` function:

```go
import "errors"

internal := errors.New("internal error")
// add additional context to an error
wrapped := fmt.Errorf("wrapper: %w", internal)
// get original error
unwrapped := errors.Unwrap(wrapped)
```

## Wrappers are not compatible

What if I need to use both wrappers in my code. Can I wrap an error using one wrapper and unwrap using different one? Answer to this question is NO. You cannot do that.

Internally standard and external libraries use different interfaces to check if error has parent. Incompatibility can be added on purpose to push developer refactor whole errors chain.

Let's see if there is any functional differences between libraries.

## Unwrapping if there is no base error

Unwrapping without base error is first case when libraries work differently.

External `Cause` function returns error itself if it does not wrap another error:

```go
import "github.com/pkg/errors"

func TestUnwrapBare(t *testing.T) {
    err := errors.New("some error")
    unwrapped := errors.Cause(err)
    require.NotNil(t, unwrapped)
    assert.Equal(t, "some error", unwrapped.Error())
}
```

Standard `Unwrap` function returns `nil` if error does not wrap another error:

```go
import "errors"

func TestUnwrapBare(t *testing.T) {
    err := errors.New("some error")
    unwrapped := errors.Unwrap(err)
    require.Nil(t, unwrapped)
}
```

## Deep nesting

Sometimes it is necessary to add context to the error (wrap) multiple times. Is there any differences between libraries in that case?

Answer is YES. External `Cause` function checks recursively all nested errors and returns the last one while standard `Unwrap` function returns first one without recursion.

## Error call stack

`github.com/pkg/errors` package records a stack trace at the point error created if it created using `New`, `Errorf`, `Wrap`, and `Wrapf` functions.
Later call stack can be rendered with `%+v` formatting verb:

```go
import (
    "fmt"
    "os"

    "github.com/pkg/errors"
)

func main() {
    f, err := os.Open("notes.txt")
    if err != nil {
        err = errors.Wrap(err, "Error opening file")
        fmt.Printf(" %+v", err)
    }
    defer f.Close()
}
```

This little program fails with open file error. Later that error is wrapped into another error that adds mode details and call stack.

Printing with verb `%+v` returns full error call-stack:

```bash
 open notes.txt: no such file or directory
Error opening file
main.main
        /Users/dharnitski/go/src/github.com/dharnitski/go-new-errors/main.go:13
runtime.main
        /usr/local/Cellar/go/1.13/libexec/src/runtime/proc.go:203
runtime.goexit
        /usr/local/Cellar/go/1.13/libexec/src/runtime/asm_amd64.s:1357
```

Standard library does not track call-stack and prints much less information.

```go
import (
    "fmt"
    "os"
)

func main() {
    f, err := os.Open("notes.txt")
    if err != nil {
        err = fmt.Errorf("Error opening file: %w", err)
        fmt.Printf(" %+v", err)
    }
    defer f.Close()
}
```

Console output for standard library:

```bash
 Error opening file: open notes.txt: no such file or directory
```

Source code for this post available on [github](https://github.com/dharnitski/go-new-errors).

Happy wrapping!
