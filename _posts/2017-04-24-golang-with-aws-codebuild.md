---
layout: post
title:  Go on AWS CodeBuild beyond Hello World
categories: 
tags: Go AWS CodeBuild DevOps
excerpt: "AWS documentation describes how to deal Hello World using CodeBuild. What will happen if we want to build more sophisticated project? Join me in this post if you want to know."
---

AWS CodeBuild supports many programming languages including Go. [Go Hello World Sample for AWS CodeBuild](http://docs.aws.amazon.com/codebuild/latest/userguide/sample-go-hw.html) article describes how to build simple Go project. It works exactly as advertized - pull code from S3 bucket, start Docker container with pre-installed GO v1.7.3, build the code and deploy generated artifact to another S3 bucket.

I followed the article and in a few minutes I got my code being compiled. CodeBuild works, but these days developers as spoiled by lightweight build tools like [Travis CI](https://travis-ci.org/) or [Circle CI](https://circleci.com/). After using these tools I cannot help but notice some differences:

* I put my sample code to GitHub and used CodePipeline to connect source control and CodeBuild. With CodePipeline there is about 30 sec delay between commit push and pipeline start. The story is the same if code is moved to AWS CodeCommit. These days developers expect that CI system uses push notifications and starts builds almost immediately.
* Integration with GitHub can be better. It would be nice if build status is pushed to GitHub
* Build Process itself can be faster for such small project. Docker image provisioning took 1 min 17 secs and compilation took 28 seconds.

Let's continue, we just proved that Hello World works, now it is time to check something more complicated.

## Custom Packages

As first modification we can use not standard package. 

Run `go get golang.org/x/text/language` to install package and update code to use it:  

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
Everything works fine locally but build fails when we try build the code in CodeBuild:

```
hello.go:6:2: cannot find package "golang.org/x/text/language" in any of
/usr/local/go/src/golang.org/x/text/language (from $GOROOT)
/go/src/golang.org/x/text/language (from $GOPATH)
```

We try to use custom package which is not installed in agent image. Lets fix the issue.

One way to mage build green again is to use package manager like (glide)[https://glide.sh/] or (dep)[https://github.com/golang/dep]. At the moment I am writing this post (May 2017) `dep` is confirmed to be official Go tool, but final version is not release yet. 

Both tools install custom packages into project `vendor` folder, package now committed into source control and project can uses it directly from local folder without `go get`.

Another option is to use good old `go get` tool when we install code on build agent. 
We need to update `buildspec.yaml` to include additional command:

```yml
  install: 
    commands:
      - go get golang.org/x/text/language
```
Build is green now and we are ready for next step.

## Sub-packages in Project 

Small project can keep all units in one location. One package cannot handle all files when project became bigger. Every package stores files to address particular domain or layer. 

We create package called `service`. After that we move our code to new package and use service from main package.

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

Console still shows `BY Low` when we run this code.

Now it is time to commit the code and run the build. 

Build server does not like it:

```
cannot find package "github.com/dharnitski/golang-codebuild/service" in any of: 
/usr/local/go/src/github.com/dharnitski/golang-codebuild/service (from $GOROOT)
/go/src/github.com/dharnitski/golang-codebuild/service (from $GOPATH)
```

To see why build failed I added add two lines to `buildspec.yml` file:

```yml
      - echo CODEBUILD_SRC_DIR - $CODEBUILD_SRC_DIR
      - echo GOPATH - $GOPATH
```      

When you check logs you should see these values:

    CODEBUILD_SRC_DIR - /tmp/src469578921/src
    GOPATH - /go

CodeBuild gets the code and drops it into unique location for every build. Path to this location defined in `$CODEBUILD_SRC_DIR` variable. Variable is available at build time, so you can build scripts to work with files. That setup is fine for many languages, as long as these languages do have special requirements where source files should be stored. Go is not one of such languages, it assumes that files located located in workspace direction under `$GOPATH`.

Unfortunately, CodeBuild does not provide control over source code drop location. `$CODEBUILD_SRC_DIR` variable is provided for reference and cannot be reassigned. To fix compilation error we have to copy files into right location. The best place for this action is `install` build phase. This is how we can copy files using `mkdir` and `cp` posix commands:

      mkdir -p ${GOPATH}/src/${PACKAGE}
      cp -a ${CODEBUILD_SRC_DIR}/.  ${GOPATH}/src/${PACKAGE}

After files copied, we can use Go tools to compile code. There is one nuance though that we need to remember. CodeBuild runs every command in a separate shell against the root of source code folder. That means that we cannot simply run `go buid` because it will be executed against original source code location. We also cannot set `cd` once and use it later, because every new command executed in its own context. We have to set current dir every time when we want to run command against files in `$GOPATH` folder.

Keeping files in right location has one additional benefit. Now we can use `go xxx ./...` commands to address all sub-packages in one command.   

This is final version of `buildspec.yml` that includes all tricks we just discussed:

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

With this file we copy sources to right location, install dependencies, compile the code and save generated artifact.

Full version of code can be found in [https://github.com/dharnitski/golang-codebuild](https://github.com/dharnitski/golang-codebuild).
