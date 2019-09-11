---
layout: post
title: "Can new Go errors wrapper replace pkg/errors?"
categories: 
tags: Go
excerpt: Go version 1.13 includes new functions to work with errors. These functions are expired by popular github.com/pkg/errors package. Let's see if new wrapper can replace existing library.  
---

[github.com/pkg/errors](https://github.com/pkg/errors) is very popular extension for Go errors. Check Dave Cheney's [blog post](https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully) to see what issues that package is solving. Now, starting from Go v1.13 support for error wrapping is included into base library.

## How both wrappers work

With `github.com/pkg/errors` you can wrap error with function `Wrap` and extract original error with function `Cause`:

    internal := errors.New("internal error")
    wrapped := errors.Wrap(internal, "wrapper")
    unwrapped := errors.Cause(wrapped)

In Go 1.13 you can do the same using `%w` formatting verb and `Unwrap` function:

    internal := errors.New("internal error")
    wrapped := fmt.Errorf("wrapper: %w", internal)
    unwrapped := errors.Unwrap(wrapped)

## Wrappers are not compatible

What if I need to use both wrappers in my code. Can I wrap error using one wrapper and unwrap using another one? Answer to this question is NO, you cannot do that.

Internally standard and external libraries use different interfaces and just does not recognize each other.

Moreover, if you try to do that you can see different results. Standard `Unwrap` function returns `nil` while external `Cause` function returns its argument.

## Deep nesting

Sometimes it is necessary to add context to the error (wrap) multiple times. Is there any differences between libraries in that case?

Answer is yes. External `Cause` recursively checks all nested errors and returns the last one while standard `Unwrap` function returns first one without recursion.

## Error call stack

`github.com/pkg/errors` record a stack trace at the point errors are created if they are created using New, Errorf, Wrap, and Wrapf.
Later call stack can be rendered with `%+v` formatting verb.

Unfortunately, this great feature is not included into standard library.
