---
layout: post
title:  Go on AWS CodeBuild beyond Hello World
categories: 
tags: Go AWS CodeBuild DevOps
excerpt: "AWS documentation describes how to deal Hello World using CodeBuild. What will happen if we want to build more sophisticated project? Join me in this post if you want to know."
---

AWS CodeBuild supports multiple programming languages including Go. [This page](http://docs.aws.amazon.com/codebuild/latest/userguide/sample-go-hw.html) describes how to build Hello World project. It works exactly as advertized - pull code from S3 bucket, start Docker container with pre-installed GO v 1.7.3, build the code, deploy generated artifact to another S3 bucket.

I got my files compiled, but these days developers as spoiled by lightweight build tools like [Travis CI](https://travis-ci.org/) and [Circle CI](https://circleci.com/). AWS can do a few things better:

*  I used CodePipeline in connection to project on GitHub and . There is about 30 sec delay between commit push and beginning of code cloning. The same story with AWS CodeCommit. Other guys use push notifications and start immediately.
* It would be nice if build status is pushed to GitHub
* Build Process can be faster for such small project. Docker image provisioning takes 1 min 17 secs, compilation takes 28 seconds. 

Hello World works, lets check something more complicated.

Lets add not standard package. 

Run `go get golang.org/x/text/language` and update code to use it:  



```go
package main

import (
	"fmt"

	"golang.org/x/text/language"
)

func main() {
	fmt.Println(language.Russian)
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



