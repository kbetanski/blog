---
title: "GitLab CI/CD and monorepo"
description: "Using GitLab CI/CD as a build system for the monorepository."
date: "2020-03-22"
lastmod: "2022-02-21"
toc: true
slug: gitlab-ci-cd-and-monorepo
tags:
  - GitLab
  - monorepository
  - CI/CD
  - gitlab-ci
links:
  - title: Example Repository
    description: Repository for the code from the blog post.
    website: https://github.com/kbetanski/gitlab-ci-monorepo
  - title: GitLab CI Documentation
    description: Documentation for GitLab CI/CD tool for software development using continuous methodologies.
    website: https://docs.gitlab.com/ee/ci/

---

The microservice-based architecture often leads to a monorepository structure to
handle multiple services in a single codebase. It resolves problems with making
changes in a few places at once in the system. Furthermore, the monorepository
is not a panacea for everything and has cons. Tools are poorly maintained and
often done to solve the workflow of development teams that created them. Not
everybody has time to write their own toolset to solve things around. As I use
GitLab CI/CD in my current development stack, I would like to show a few tips
for working with it and the monorepository.

Structure of the repository in the presented case:
```bash
services/
services/project-one/
services/project-two/
.gitlab-ci.yml
```

Let's pretend `project-one` is a service written javascript and `project-two` in
PHP, by that I can show that these tips are build-system specific and
programming language agnostic.

## Use `include` to organise *.gitlab-ci.yml*

Single configuration file declaring pipelines in the repository with multiple
services will be completely unreadable. It is possible to have separate
configuration files defining CI/CD pipeline per service by including them in the
root *.gitlab-ci.yml*.

>.gitlab-ci.yml
```yaml
include:
  - local: ./services/project-one/.gitlab-ci.yml
  - local: ./services/project-two/.gitlab-ci.yml
```

That way, the root file can be filled with `stages:`, `include:` and generic job
definitions, like in example commit message linter.

## Control workflow by `only:changes`

The jobs are now separated by services but pushing to repository triggers
running pipelines for all of the projects. To prevent this behavior and trigger
jobs by only for services with made changes in catalog add `only:changes`
keyword to jobs definitions:

>services/project-one/.gitlab-ci.yml
```yaml
project-one test:
  stage: test
  before_script:
    - cd services/project-one
    - yarn install
  script:
    - yarn run test
  only:
    changes:
      - services/project-one/**/*
```

Disclaimer, It triggers specified jobs based on changes in the last push to the
branch, not for every commit in the merge request.

## `Cache` dependencies per service

To speed up a CI/CD pipeline, you can cache the dependencies between jobs and
runs. It's useful when the service has a few jobs that require resolving and
downloading dependencies.

>services/project-two/.gitlab-ci.yml
```yaml
project-two install:
  stage: install
  before_script:
    - cd services/project-two
  script:
    - composer install -n
  cache:
    key: $CI_COMMIT_REF_SLUG-project-two
    policy: pull-push
    paths:
      - services/project-two/vendor

project-two test:
  stage: test
  before_script:
    - cd services/project-two
    - composer install -n
  script:
    - composer test
  only:
    changes:
      - services/project-two/**/*
  cache:
    key: $CI_COMMIT_REF_SLUG-project-two
    policy: pull
    paths:
      - services/project-two/vendor
``` 

As a cache key, this example uses branch name and service name. It means that
cache will sustain between commits in that branch only.

Why `composer install` is still in the test running job? The cache can be purged
between the runs CI/CD jobs and pipelines, or the storage service used for the
cache mechanism can fail. This way, you can be sure that the failure of the
caching mechanism will not create failures on the CI pipelines, keeping some
redundancy in the CI/CD.

## Reduce jobs definitions with: `extends`

The job definitions are already growing with lines that are the same between
them. Imagine adding a job linting your code for project two. It would have to
declare cache, changing directory and etc. Let's change that situation and
prevent the repetition of settings.

>services/project-one/.gitlab-ci.yml
```yaml
.project-one:
  before_script:
    - cd services/project-one
  cache:
    key: $CI_COMMIT_REF_SLUG-project-one
    policy: pull
    paths:
      - services/project-one/node_modules
  only:
    changes:
      - services/project-one/**/*

project-one install:
  extends: .project-one
  stage: install
  script:
    - yarn install
  cache:
    policy: pull-push
  
project-one test:
  extends: .project-one
  stage: test
  script:
    - yarn install
    - yarn run test

project-one lint:
  extends: .project-one
  stage: test
  script:
    - yarn install
    - yarn run lint
```

Adding another job to run additional tests no more creates lots of lines and
focuses on what commands it has to run. 

## Define `needs` of jobs to start

The pipeline from example starts to grow, but what if installing dependencies in
`project-one install` job fails? Failure of a CI/CD job in one service fails CI
jobs from another. That behavior can be changed too. By using the `needs`
keyword.

>service/project-two/.gitlab-ci.yml
```yaml
.project-two:
  before_script:
    - cd services/project-two
  cache:
    key: $CI_COMMIT_REF_SLUG-project-two
    policy: pull
    paths:
      - services/project-two/vendor
  only:
    changes:
      - services/project-two/**/*

project-two install:
  extends: .project-two
  stage: install
  script:
    - composer install -n
  cache:
    policy: pull-push
  
project-two test:
  extends: .project-two
  stage: test
  needs:
    - project-two install
  script:
    - composer install
    - composer test
  
project-two lint:
  extends: .project-two
  stage: test
  needs:
    - project-two install
  script:
    - composer install
    - composer lint
```

Under the `needs` keyword, you should specify jobs that must succeed to run this
job. In this case, failure of command in another service's pipeline won't
prevent running jobs in another one.
