# github-actions

Public repository for reusable github actions 

As is customary for GitHub Actions, we provide release tags for you to reference in your repository's workflow files. The `v1` tag is a moving tag that will always apply to the latest version 1 train release.

## Usage

### ECR Push

```yml
jobs:
  ecr-push:
    uses: Fatsoma/reusable-actions/.github/workflows/ecr-push.yml@v1
    with:
      aws-region: us-west-1
      ecr-repository: ${{ github.event.repository.name }}
      environment: staging
      image-tag: latest
```

You can use custom docker build instructions with a `ci-docker-build` make target:

```make
CI_DOCKER_IMAGE=

.PHONY: ci-docker-build
ci-docker-build:
	docker build --tag "$(CI_DOCKER_IMAGE)" .
```

### Go Tests

Go lang test, syntax check, security check:

```yml
name: Go

on: [push]

jobs:
  syntax:
    uses: Fatsoma/reusable-actions/.github/workflows/go-syntax.yml@v1

  test:
    uses: Fatsoma/reusable-actions/.github/workflows/go-test.yml@v1
    with:
      enable-translate: true

  security:
    uses: Fatsoma/reusable-actions/.github/workflows/go-security.yml@v1
```
