---
layout: post
title: "Angular code coverage using CircleCI and Coveralls"
categories: 
tags: Angular TDD Coveralls CircleCI
excerpt: Step by steps instruction how to enable Code Coverage check for you Angular CI pipeline.   
---
## Hello Angular

Angular out of the box packed with great tools and templates to scaffold a new project in a few minutes.

Install Angular CLI if you do not have it yet using instructions from [CLI Command Reference](https://angular.io/cli).

Create a new project:

    ng new angular-hello-world
    cd angular-hello-world
    ng test --watch=false --code-coverage

Last command runs unit tests included into scaffolded project and calculates code coverage.

You should see something like output below if everything was done right:

```shell
Chrome 73.0.3683 (Mac OS X 10.14.3): Executed 3 of 3 SUCCESS (0.25 secs / 0.223 secs)
TOTAL: 3 SUCCESS

=============================== Coverage summary ===============================
Statements   : 100% ( 7/7 )
Branches     : 100% ( 0/0 )
Functions    : 100% ( 1/1 )
Lines        : 100% ( 6/6 )
================================================================================
TOTAL: 3 SUCCESS
TOTAL: 3 SUCCESS
```

Code coverage files stored in `/coverage` folder. Open `/coverage/angular-hello-world/index.html` file in browser to see results:

![HTML Code Coverage](/assets/img/2019-04-17/coverage.png)

## Get Coveralls access token

> Hint: Project should be pushed to GitHub to use Coveralls.

Sign in into [Coveralls](https://coveralls.io/) using your GitHub credentials.

Go to [Add Repos](https://coveralls.io/repos/new) page.

> Hint: click `SYNC REPOS` button at the top right corner of the screen if you do not see your repo.

Find your project in the list of available projects and enable it:

![Code Coverage](/assets/img/2019-04-17/coveralls-new.png)

Go to details and record secret access token, you will need it later.

## Install Coveralls package

We need to install `npm` package that enables pushing coverage results to Coveralls:

    npm install coveralls --save-dev

## Post to Coverage from local machine

Let's test if we can post generated test coverage to Coveralls.

> Hint: Coveralls uses `lcov.info` file to read code coverage lines. This file is generated out of the box if you used Angualar project scaffolding.

Create `.coveralls.yml` file in project root and paste secret token into created file:

    repo_token: <coveralls-secret-token>

> Important: Do NOT commit this file into GIT. Delete it or add into `.gitignore` after you are done with testing.

Run command line to push code coverage:

    cat ./coverage/angular-hello-world/lcov.info | ./node_modules/coveralls/bin/coveralls.js

You can see code coverage [being published](https://coveralls.io/github/dharnitski/angular-hello-world) now:

![HTML Code Coverage](/assets/img/2019-04-17/coverage2.png)

## Add .circleci/config.yml

Angular documentation includes sample for testing project [using Circle CI](https://angular.io/guide/testing#configure-project-for-circle-ci).

Check it to see if anything is changed after this post is published.

[Our script](https://github.com/dharnitski/angular-hello-world/blob/master/.circleci/config.yml) adds a few extra lines to integrate with Coveralls:

{% raw %}

```yml
version: 2
jobs:
  build:
    working_directory: ~/my-project
    docker:
      - image: circleci/node:10-browsers
    steps:
      - checkout
      - restore_cache:
          key: my-project-{{ .Branch }}-{{ checksum "package-lock.json" }}
      - run: npm install
      - save_cache:
          key: my-project-{{ .Branch }}-{{ checksum "package-lock.json" }}
          paths:
            - "node_modules"

      - run:
          name: Lint
          command: npm run ng lint
      - run:
          name: Unit Test
          command: npm run ng test -- --watch=false --code-coverage
      - run:
          name: Coveralls
          command: cat ./coverage/angular-hello-world/lcov.info  | ./node_modules/coveralls/bin/coveralls.js

```

{% endraw %}

## Configure CircleCI

Login into CircleCI using GitHub credentials and Set Up yor project.

> Hint: You can skip configuration steps as it is only generates `.circleci/config.yml` file that we already have. You can safely click `Start building` button as soon as you see it.

Go to project setting in CircleCI and add environment variable `COVERALLS_REPO_TOKEN` with value storing secret Coveralls access token.

Now everything is configured. Every push into every branch triggers build using CircleCI and publishes code coverage.

It is also a right time to enforce [coverage rules](https://angular.io/guide/testing#code-coverage-enforcement) or setup [pull request alerts](https://coveralls.io/features).  

## Badges

Do not forget to add Coveralls and CircleCI badges to `README.md` file.

[![CircleCI](https://circleci.com/gh/dharnitski/angular-hello-world.svg?style=svg)](https://circleci.com/gh/dharnitski/angular-hello-world)
[![Coverage Status](https://coveralls.io/repos/github/dharnitski/angular-hello-world/badge.svg?branch=master)](https://coveralls.io/github/dharnitski/angular-hello-world?branch=master)

Happy Covering!