---
layout: post
title: "Uninstall Go on macOS"
categories: 
tags: Go Setup
excerpt: After working for several years on different Go versions and using variety of different tools it is time to reset my system to "clean" state. Just to be clear, my machine works just fine, it is just perfectionist inside me wants to remove stuff that is not in use anymore.
---
## Check if Go is installed

Let's start with checking if Go actually installed on machine.

To see version of Go run this command:

    go version

To see Go location run this one:

    which go

Default value is `/usr/local/bin/go`.

## Go installation options

There are multiple ways to setup Go dev environment. Developer can pick option that provides more flexibility or more simplicity. Flexibility presented by packaged Go binaries that can be  [.darwin-amd64.tar.gz archive](https://golang.org/dl/) when simplicity achieved with package management tools like [Homebrew](https://brew.sh/). Option in between is [.darwin-amd64.pkg macOS package installer](https://golang.org/dl/)

What is important to understand is that you can have several versions of Go installed on machine using several different methods. In that case, there is no single solution to uninstall all these versions and multiple places to be checked.

## If Go installed using Homebrew

Homebrew is a smart package manager that is capable to clean up after after himself.

It is recommended to use built-in `uninstall` command if Go was installed using `brew`.

Command below uninstalls Dep and Go if it was installed using `brew`:

    brew uninstall dep
    brew uninstall go

Homebrew cleanups `$PATH` and other config files. No extra steps are necessary if Go was never installed using different options.

> tip: `brew` keeps Go files in `/usr/local/Cellar/go/x.x.x/` folder  

## If Go installed using macOS package

Run command line to see if Go was installed using macOS package

    pkgutil --pkgs

Go macOS package is presented as `com.googlecode.go` in result list.
macOS package stores files in predefined location. For Go version 1.12 these are the files to be deleted:

* file `/etc/paths.d/go`
* folder `usr/local/go`

There is a command line to check if files are moved to different location in future Go versions:

    pkgutil --only-files --files com.googlecode.go


Next step is to remove the system record of Go package:

    sudo pkgutil --forget com.googlecode.go

Cleaning up `$PATH` environment variable to be performed manually as described below.

## Removing files manually

* `/etc/paths.d/go` check if that file exists and remove. It is added by [macOS package](https://golang.org/dl/).
* `/usr/local/go` check if that folder exists and remove. [macOS package]
(https://golang.org/dl/) keeps Go files in that folder.
* Check `$PATH` for `*/go/bin`. That is a good hint find where Go is installed. Delete that `go` folder.
* `$HOME/go` or `$GOPATH` - Go Workspace. **Important!** This folder may contain not pushed code in `/src` sub-folder. Be careful to not lost your work.  

## Environment Variables

* `$PATH` contains path to Go binaries (`/usr/local/go/bin` by default). Update  `$PATH` to exclude path to Go binaries.
* `$GOPATH` may contain override for workspace path default value `$HOME/go`. Remove `$GOPATH` if it exists.

Happy uninstalling!