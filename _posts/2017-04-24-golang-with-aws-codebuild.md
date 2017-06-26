---
layout: post
title:  Go on AWS CodeBuild beyond Hello World
categories: 
tags: Go AWS CodeBuild DevOps
excerpt: "CodeBuild is an AWS managed build service. Out of the box it supports many programming languages including Java, Node, Python, Ruby and Golang. AWS provides good documentation and samples for different frameworks to get developers up to speed. I currently use Golang on my working place and want to see how CodeBuild works with Go ecosystem. I am going to start with simple hello-world project and later make it more complicated to see how CodeBuild deals with dependencies and nested packages."
---

AWS CodeBuild supports many programming languages. Amazon recommends to use pre-configured and optimized Docker images that can be found on [this page](http://docs.aws.amazon.com/codebuild/latest/userguide/build-env-ref.html). Check that page to see what is available right now. In June 2017 there was support for Golang versions 1.5, 1.6 and 1.7. If you do not see required framework in that list, you can install required components during `install` build phase. For more information, see [Build Spec Example](http://docs.aws.amazon.com/codebuild/latest/userguide/build-spec-ref.html#build-spec-ref-example).

 [AWS CodeBuild Samples](http://docs.aws.amazon.com/codebuild/latest/userguide/samples.html) page contains sample for different languages. As you can see, it includes [Go Hello World Sample for AWS CodeBuild](http://docs.aws.amazon.com/codebuild/latest/userguide/sample-go-hw.html). That article describes how to build simple Go project. It is pretty straightforward - you manually upload source code and build definition to S3 bucket, CodeBuild pulls code from that bucket, starts Docker container with pre-installed GO v1.7.3, builds the code and deploys generated artifact to another S3 bucket.

I am not going to add much here. Everything works as it should if you follow page instructions. In 30 min you should get everything setup, code compiled and deployed. 

Next step is optional. You can continue use CodeBuild using S3 bucket, but without much effort you can connect CodeBuild to your source control repository. Again, I am not going to describe that in details, AWS has good [setup documentation for that process](http://docs.aws.amazon.com/codebuild/latest/userguide/how-to-create-pipeline.html#how-to-create-pipeline-add-test). 

AWS has lots of advantages, but it wold be not fair to skip some areas where lightweight build tools like [Travis CI](https://travis-ci.org/) or [Circle CI](https://circleci.com/) do better work:

* With CodePipeline there is about 30 sec delay between my GitHub commit push and pipeline start. The story is the same if code is moved to AWS CodeCommit. These days developers expect that CI system uses push notifications and starts builds immediately.
* It would be great if build results are pushed to GitHub. Many other CI systems do that out of the box.
* Build Process itself can be faster for such simple project. When I measured the time, Docker image provisioning took 1 min 17 secs and compilation took 28 seconds.

But let's continue, we just proved that Hello World works, now it is time to check something more complicated.

## Custom Packages

Hello World sample contains only one file - `main.go` and does not use any dependencies. Lets check what will happen if we add custom package to the project. 

Let's extend our sample to build application that takes language and returns region where that language is used.

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
Everything works fine locally with that change, but process fails when we try to build the code in CodeBuild:

```
hello.go:6:2: cannot find package "golang.org/x/text/language" in any of
/usr/local/go/src/golang.org/x/text/language (from $GOROOT)
/go/src/golang.org/x/text/language (from $GOPATH)
```

What is going on?

Build process tries to compile source files in bare Go docker container. Our custom package installed locally, but it is not installed in image used by CodeBuild. 

Lets fix the problem.

One way to make build green again is to use package manager like [glide](https://glide.sh/) or [dep](https://github.com/golang/dep). At the moment I am writing this post (June 2017) `dep` is confirmed to be official Go tool, but final version is not release yet. 

Both tools install custom packages into project `vendor` folder and these packages are committed into source control. After that, any tool uses files directly from local folder without necessity to call `go get`.

Another option is to use good old `go get` tool during `install` build phase on build agent. 

Let's try that approach.

Update `buildspec.yaml` to include additional command:

```yml
  install: 
    commands:
      - go get golang.org/x/text/language
```
Build is green now and we are ready for new challenges.

## Sub-packages in Project 

Small project can keep all units in one folder. It make sense to add some structure when project grow. project structure in Go defined using packages. Every package stores files to address particular domain or layer. 

We are going to create package called `service`, move our code and use `service` from `main` package.

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

Now it is time to buind the code using CodeBuild. 

Build server does not like it:

```
cannot find package "github.com/dharnitski/golang-codebuild/service" in any of: 
/usr/local/go/src/github.com/dharnitski/golang-codebuild/service (from $GOROOT)
/go/src/github.com/dharnitski/golang-codebuild/service (from $GOPATH)
```

I added add two lines to `buildspec.yml` file to see why build fails:

```yml
      - echo CODEBUILD_SRC_DIR - $CODEBUILD_SRC_DIR
      - echo GOPATH - $GOPATH
```      

When you check logs you should see something like this:

    CODEBUILD_SRC_DIR - /tmp/src469578921/src
    GOPATH - /go

CodeBuild gets the code and drops it into unique location for every build. That path is defined in `$CODEBUILD_SRC_DIR` variable. Variable is available at build time, so you can build scripts to work with files. 

This implementation is fine for many languages, as long as these languages do have special requirements where source files should be stored. Go is not one of such languages, it assumes that files located in workspace direction under `$GOPATH`.

Unfortunately, CodeBuild does not provide control over source code drop location. `$CODEBUILD_SRC_DIR` variable is provided for reference and cannot be reassigned. To fix compilation error we have to copy files into right location. The best place for this action is `install` build phase. This is how we can copy files using `mkdir` and `cp`:

      mkdir -p ${GOPATH}/src/${PACKAGE}
      cp -a ${CODEBUILD_SRC_DIR}/.  ${GOPATH}/src/${PACKAGE}

After files copied, we can use Go tools to compile code. 

There is one nuance though that we need to remember. CodeBuild runs every command in a separate shell against the root of source code folder. That means that we cannot simply run `go buid` because it will be executed against original source code location. We also cannot set `cd` once and use it later, because every new command executed in its own context. We have to set current dir every time when we want to run command against files in `$GOPATH` folder.

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

Happy coding!
