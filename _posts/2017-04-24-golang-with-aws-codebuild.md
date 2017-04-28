---
layout: post
title:  Go on AWS CodeBuild beyond Hello World
categories: 
tags: Go AWS CodeBuild DevOps
excerpt: "AWS documentation describes how to deal Hello World using CodeBuild. What will happen if we want to build more sophisticated project? Join me in this post if you want to know."
---

AWS CodeBuild supports multiple programming languages including Go. [This page](http://docs.aws.amazon.com/codebuild/latest/userguide/sample-go-hw.html) describes how to build Hello World project. It works exactly as advertized - pull code from S3 bucket, start Docker container with pre-installed GO v 1.7.3, build the code, deploy generated artifact to another S3 bucket.

I got my files compiled, but these days developers as spoiled by lightweight build tools like [Travis CI](https://travis-ci.org/) or [Circle CI](https://circleci.com/). AWS can do a few things better:

*  I used CodePipeline in connection to project on GitHub and . There is about 30 sec delay between commit push and beginning of code cloning. The same story with AWS CodeCommit. Other guys use push notifications and start immediately.
* It would be nice if build status is pushed to GitHub
* Build Process can be faster for such small project. Docker image provisioning takes 1 min 17 secs, compilation takes 28 seconds. 

Hello World works, lets check something more complicated.

## Custom Packages

Lets add not standard package. 

Run `go get golang.org/x/text/language` and update code to use it:  

```go
package main

import (
	"fmt"

	"golang.org/x/text/language"
)

func main() {
	en := language.Make("be")
	fmt.Println(en.Region())
}
```

Build fails with message:

```
hello.go:6:2: cannot find package "golang.org/x/text/language" in any of
/usr/local/go/src/golang.org/x/text/language (from $GOROOT)
/go/src/golang.org/x/text/language (from $GOPATH)
```

That make sense, we try to use package that is not installed in agent image. Lets fix that.

One way to do that is to use package manager like (glide)[https://glide.sh/] or (dep)[https://github.com/golang/dep]. At the moment I am writing this post (April 2017) `dep` is confirmed to be official Go tool, but final version is not release yet. These tools install packages into local `vendor` folder and project can use them directly from that folder.

Another option is to use old good `go get` tool in build server. 
Update `buildspec.yaml`:

```yml
  install: 
    commands:
      - go get golang.org/x/text/language
```
Build is green now and we are ready for next step.

## Sub-packages in Project 

Small project can keep all units in one package. When project became bigger code distributed across multiple packages. Lets create package `service`, add move code to it and use it from main package.

Add `service/regions.go` file:

```go
package service

import (
	"golang.org/x/text/language"
)

// Find returns Region and Confidence for Language
func Find(lang string) (language.Region, language.Confidence) {
	en := language.Make(lang)
	return en.Region()
}
```
Update main file to use new package:

```go
package main

import (
	"fmt"

	"github.com/dharnitski/golang-codebuild/service"
)

func main() {
	fmt.Println(service.Find("be"))
}
```

Console still shows `BY Low` if we run this code.

Lets commit the code and run the build. Build fails:

```
cannot find package "github.com/dharnitski/golang-codebuild/service" in any of: 
/usr/local/go/src/github.com/dharnitski/golang-codebuild/service (from $GOROOT)
/go/src/github.com/dharnitski/golang-codebuild/service (from $GOPATH)
```

To see why that happens we need to add two lines to `buildspec.yml`:

```yml
      - echo CODEBUILD_SRC_DIR - $CODEBUILD_SRC_DIR
      - echo GOPATH - $GOPATH
```      

In build logs you can find these values:

    CODEBUILD_SRC_DIR - /tmp/src469578921/src
    GOPATH - /go

CodeBuild drops source files in `tmp` location. That is fine for many languages where you can compile code in any location. Go is different. Code should be stored in right location under `$GOPATH`.

Unfortunately, CodeBuild does not allow control over `$CODEBUILD_SRC_DIR`. Instead, we have to copy source files to right location during `install` phase of build.

After files are copied, we can call Go tools to compile code. CodeBuild runs each command in a separate shell against the root of source code folder. We have to set current dir in every command to execute it against files in GOPATH folder.

Final version of buildspec.yml:

```yml
version: 0.1

environment_variables:
  plaintext:
    PACKAGE: "github.com/dharnitski/golang-codebuild"

phases:
  install: 
    commands:
      - echo CODEBUILD_SRC_DIR - $CODEBUILD_SRC_DIR
      - echo GOPATH - $GOPATH

      - echo Create dir in GOPATH for sources
      - mkdir -p ${GOPATH}/src/${PACKAGE}

      - echo Copy source files into GOPATH
      - echo cp -a ${CODEBUILD_SRC_DIR}/.  ${GOPATH}/src/${PACKAGE}
      - cp -a ${CODEBUILD_SRC_DIR}/.  ${GOPATH}/src/${PACKAGE}

      - cd ${GOPATH}/src/${PACKAGE} && go get ./...

  build:
    commands:
      - cd ${GOPATH}/src/${PACKAGE} && go build -o ${CODEBUILD_SRC_DIR}/application

artifacts:
  files:
    - application
```

