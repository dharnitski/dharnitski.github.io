---
layout: post
title:  Go on AWS CodeBuild beyond Hello World
categories: 
tags: Go AWS CodeBuild DevOps
excerpt: "CodeBuild is an AWS managed build service. Out of the box it supports many programming languages including Java, Node, Python, Ruby and Golang. AWS provides good documentation and samples for different frameworks to get developers up to speed. I currently use Golang on my machine. Let me show you how CodeBuild works with Go ecosystem. I will start with a simple hello-world project and later will make it more complicated to demonstrate how CodeBuild deals with dependencies and nested packages."
---

AWS CodeBuild supports many programming languages. For each Amazon recommends to use pre-configured and optimized Docker images that can be found in AWS documentation. Check [this page](http://docs.aws.amazon.com/codebuild/latest/userguide/build-env-ref.html) to see what is currently available. At the moment I am writing this post (June 2017) it supports Golang versions 1.5, 1.6 and 1.7. If you do not see required framework in that list, you can add custom components during `install` build phase. For more information, see [Build Spec Example](http://docs.aws.amazon.com/codebuild/latest/userguide/build-spec-ref.html#build-spec-ref-example).

 [AWS CodeBuild Samples](http://docs.aws.amazon.com/codebuild/latest/userguide/samples.html) page contains samples for different languages.   [A sample for Go](http://docs.aws.amazon.com/codebuild/latest/userguide/sample-go-hw.html) describes how to build a simple Go project. It is pretty straightforward - you manually upload a source code and build definition to S3 bucket, CodeBuild pulls the code from the bucket, launches a Docker container with pre-installed GO v1.7.3, builds the code and deploys a generated artifact to another S3 bucket.

If you follow page instructions, in 30 min you will get everything setup, and will have a compiled and deployed code. 

 You can continue working with CodeBuild using S3 bucket, but without much effort you can also connect CodeBuild to your source control repository. AWS has good [setup documentation for this process](http://docs.aws.amazon.com/codebuild/latest/userguide/how-to-create-pipeline.html#how-to-create-pipeline-add-test). 

This sample proves that AWS does work, but it wold be not fair to skip some areas where lightweight build tools like [Travis CI](https://travis-ci.org/) or [Circle CI](https://circleci.com/) do better job:

* Nowadays many CI systems use push notifications and start builds immediately where when building With AWS CodePipeline there is about 30 sec delay between my GitHub commit push and pipeline start. The story is the same if code is moved to AWS CodeCommit. 
* It is also common across other systems to push build results to GitHub. 
* Build Process in mentioned CI systems is faster for such a simple project. When I measured the time, Docker image provisioning for AWS took 1 min 17 secs and compilation took 28 seconds.

Anyway, we can continue. We just proved that Hello World works, now it is time to check something more complicated.

## Custom Packages

The Hello World sample contains only one file - `main.go` and does not use any dependencies. To see how it works with custom packages,  let's extend our sample with a function that returns the main region for given language. In Go this functionality is available in `golang.org/x/text/language` package.

Run `go get golang.org/x/text/language` to install the package and update the code as shown below to use the package:  

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
Everything works fine locally, but the build process fails when we try to build the code in CodeBuild:

```
hello.go:6:2: cannot find package "golang.org/x/text/language" in any of
/usr/local/go/src/golang.org/x/text/language (from $GOROOT)
/go/src/golang.org/x/text/language (from $GOPATH)
```

What is going on?

The Build process compiles source files in a standard Go docker container. After our last change the source code requires the custom package which is installed locally, but it is not installed in an image used by CodeBuild. 

Let's fix the problem.

One way to make the build green again is to use the package manager tools like [glide](https://glide.sh/) or [dep](https://github.com/golang/dep). At the moment I am writing this post (June 2017) `dep` is confirmed to be an official Go tool, but the final version is not released yet. 

Both tools install custom packages into the project `vendor` folder and these packages are committed into the source control. After that, any tool will use files directly from the `vendor` folder folder without necessity to call `go get`.

The other option is to use a good old `go get` tool during `install` build phase on a build agent. 

To try the last approach, Update `buildspec.yaml` to include additional command:

```yml
  install: 
    commands:
      - go get golang.org/x/text/language
```
It will make the build green. Now we are ready for new challenges.

## Sub-packages in Project 

For a small project like Hello World you can keep all units in one folder. When the project grows, a more organized structure will be needed.   The project structure in Go is defined with packages. Every package stores files to address particular domain or layer. 

We will create the package called `service`, move our code there and will use `service` package in the `main` package.

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
Update the main file to use a new package:

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

The Console will show `BY Low` when you run this code locally.

But when you build the same code using CodeBuild, the build server does not like it:

```
cannot find package "github.com/dharnitski/golang-codebuild/service" in any of: 
/usr/local/go/src/github.com/dharnitski/golang-codebuild/service (from $GOROOT)
/go/src/github.com/dharnitski/golang-codebuild/service (from $GOPATH)
```

To troubleshoot, add two lines to `buildspec.yml` file to see why build fails:

```yml
      - echo CODEBUILD_SRC_DIR - $CODEBUILD_SRC_DIR
      - echo GOPATH - $GOPATH
```      

When you check logs you should see something like this:

    CODEBUILD_SRC_DIR - /tmp/src469578921/src
    GOPATH - /go

CodeBuild gets the code and drops it into a unique location for every build. That path is defined in `$CODEBUILD_SRC_DIR` variable. The variable is available at the build time, so you can build scripts to work with files. 

This implementation is fine for many languages, as long as these languages do not have special requirements where source files should be stored. Go is not one of such languages, it assumes that files are located in the workspace direction under `$GOPATH`.

Unfortunately, CodeBuild does not provide control over the source code drop location. The `$CODEBUILD_SRC_DIR` variable shows where the files are dropped but it cannot be reassigned. 

To fix the compilation error shown above we have to copy files into the right location. The best place for this action is `install` build section. Use `mkdir` and `cp` to copy files:

      mkdir -p ${GOPATH}/src/${PACKAGE}
      cp -a ${CODEBUILD_SRC_DIR}/.  ${GOPATH}/src/${PACKAGE}

After files are copied, we can use Go tools to compile the code. 

There is one caveat to that. CodeBuild runs every command in a separate shell against the root of the source code folder. That means that we cannot simply run `go build` as it will be executed against the original source code location. We also cannot set `cd` once and use it later, because every new command is executed in its own context. We have to set the current dir every time when we want to run the command against files in `$GOPATH` folder.

Keeping files in the right location has one additional benefit. Now we can use `go xxx ./...` pattern to execute the command against all sub-packages recursively.   

This is a final version of `buildspec.yml` that includes all tricks we just discussed:

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

The file above copies sources to the right location, installs dependencies, compiles the code and saves the generated artifact.

The Full version of the code can be found at [https://github.com/dharnitski/golang-codebuild](https://github.com/dharnitski/golang-codebuild).

Happy coding!
