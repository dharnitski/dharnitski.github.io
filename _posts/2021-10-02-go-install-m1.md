---
layout: post
title: "Setup Golang in 2021 (still on MacOS)"
categories: 
tags: Go Setup vscode TDD
excerpt: Just received a new laptop and realized that my notes from 2019 are not up to date. We do not use dep and GO workspaces. Time to make a new checklist. 
---
## Prerequisites

This setup tested with for macOS Monterey Version 12.0.1.

We will use `brew` to install software. Follow instructions on [Homebrew](https://brew.sh/) main page to install it if it is not installed yet.

## Install Go

Run this command line to install Go:

    brew install go

Go binary distribution is installed now. It can be tested by checking Go version:

    go version

## Add go/bin to the $PATH

We need to register `go/bin` in $PATH to make sure that all go programs work:

    "export PATH=\$PATH:\$(go env GOPATH)/bin" >> ~/.zshrc

Alternative, the manual option is to open file ~/.zshrc and append the line

export PATH=$PATH:$(go env GOPATH)/bin
In any case, you need to restart your shell to start using this update.

## linter

These days all linters are available in one nice package. Use this command line to install:

    brew install golangci-lint

This is how installation can be checked:

    golangci-lint --version


## Tools

* Git(Hub) UI client: `brew install --cask github`
* Docker: `brew install --cask docker`
* VsCode: `brew install --cask visual-studio-code`

Open/create GO file in VsCode and follow IDE hints to install Go plugin and tools.  

## Access to private repos

Extra configuration may be necessary to use GO module stored in private GIT repository.

Often access to that private repo requires MFA and GO tools cannot use such credentials.

In GitHub issue can be solved by generating Access Token and using it in GIT client.


At first step we need to generate an access token with repo scope as described [here](https://docs.github.com/en/github/authenticating-to-github/creating-a-personal-access-token).

After we got the token we can update update git configuration to force its use.

Replace below ${user} with your GitHub login (not email) and ${personal_access_token} with the access token and run the script:

    git config \
        --global \
        url."https://${user}:${personal_access_token}@github.com".insteadOf \
        "https://github.com"

Configuration can be tested by getting a list of references from private repo. Replace ${account} and ${repo} with real values and run:

    git ls-remote -q https://github.com/${account}/${repo}.git

## Bypass Modules proxy for private repos

Since Go 1.13, the go command by default downloads and authenticates modules using the [Go module mirror and Go checksum database](https://proxy.golang.org) for accelerating modules download. 

Private reports are not available through that service and you can see `version not found` error when you try to get it. We need to add the repo path to `$GOPRIVATE` to exclude that repo from the public Go module list:

    echo "export GOPRIVATE=github.com/${account}" >> ~/.zshrc

You can test applied change with command below:

    go env | grep "PRIVATE"


Additional reading on the topic - [https://golang.org/ref/mod#private-modules](https://golang.org/ref/mod#private-modules)


Happy Coding!