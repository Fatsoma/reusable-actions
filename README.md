# github-actions

Public repository for reusable github actions

Release tags identify published versions. Callers pin every reusable action and workflow to the full immutable Git commit SHA resolved from a release tag. The SHA fixes the exact code executed. Add the corresponding named tag as a trailing comment, for example `@0123456789abcdef... # v1`, so [Dependabot updates for GitHub Actions](https://docs.github.com/en/code-security/how-tos/secure-your-supply-chain/secure-your-dependencies/auto-update-actions) and reviewers can display the human-readable release version.

## Usage

```yml
jobs:
  ecr-push:
    uses: Fatsoma/reusable-actions/.github/workflows/ecr-push.yml@REUSABLE_ACTIONS_SHA # v1
    with:
      aws-region: us-west-1
      ecr-repository: ${{ github.event.repository.name }}
      environment: staging
      image-tag: latest

  ruby-gem-publish:
    on:
      push:
        branches: main
    uses: Fatsoma/reusable-actions/.github/workflows/ruby-gem-publish.yml@REUSABLE_ACTIONS_SHA # v1
    with:
      gem-name: example-gem

  ruby-lint:
    uses: Fatsoma/reusable-actions/.github/workflows/ruby-lint.yml@REUSABLE_ACTIONS_SHA # v1

  ruby-vulnerabilities:
    uses: Fatsoma/reusable-actions/.github/workflows/ruby-vulnerabilities.yml@REUSABLE_ACTIONS_SHA # v1

  ruby-test:
    uses: Fatsoma/reusable-actions/.github/workflows/ruby-test.yml@REUSABLE_ACTIONS_SHA # v1
```

## Go CI

The Go CI workflows standardize `Test`, `Coverage`, and `Security` jobs for approved service profiles. Each call uses an immutable `REUSABLE_ACTIONS_SHA`. The private key is inherited, while the GitHub App client ID and repository-specific private-module allowlist are explicit inputs.

Test workflows use Go's native JSON output and need no reporting tool. Security runs the official `securego/gosec@master` action with its default `./...` arguments, deliberately tracking the latest stable gosec release.

```yml
permissions:
  contents: read

jobs:
  test:
    if: github.event_name != 'pull_request' || github.event.pull_request.head.repo.full_name == github.repository
    uses: Fatsoma/reusable-actions/.github/workflows/go-test.yml@REUSABLE_ACTIONS_SHA # v1
    with:
      app-client-id: ${{ vars.FATSOMA_DEPENDENCIES_APP_CLIENT_ID }}
      module-allowlist: |
        v2-api-auth
        v2-api-fatsoma-api
      translate: true
    secrets: inherit

  coverage:
    if: github.event_name != 'pull_request' || github.event.pull_request.head.repo.full_name == github.repository
    uses: Fatsoma/reusable-actions/.github/workflows/go-coverage.yml@REUSABLE_ACTIONS_SHA # v1
    with:
      app-client-id: ${{ vars.FATSOMA_DEPENDENCIES_APP_CLIENT_ID }}
      module-allowlist: |
        v2-api-auth
        v2-api-fatsoma-api
      translate: true
    secrets: inherit

  security:
    if: github.event_name != 'pull_request' || github.event.pull_request.head.repo.full_name == github.repository
    uses: Fatsoma/reusable-actions/.github/workflows/go-security.yml@REUSABLE_ACTIONS_SHA # v1
    with:
      app-client-id: ${{ vars.FATSOMA_DEPENDENCIES_APP_CLIENT_ID }}
      module-allowlist: |
        v2-api-auth
        v2-api-fatsoma-api
    secrets: inherit
```

All Test and Coverage entrypoints accept the same inputs:

```yml
with:
  app-client-id: ${{ vars.FATSOMA_DEPENDENCIES_APP_CLIENT_ID }}
  module-allowlist: |
    v2-api-auth
    v2-api-fatsoma-api
  translate: false # Optional; defaults to false.
secrets: inherit
```

Use the entrypoint matching the required service contract:

| Job | No services | Elasticsearch | RabbitMQ | Elasticsearch + Valkey |
| --- | --- | --- | --- | --- |
| Test | `go-test.yml` | `go-test-elasticsearch.yml` | `go-test-rabbitmq.yml` | `go-test-elasticsearch-valkey.yml` |
| Coverage | `go-coverage.yml` | `go-coverage-elasticsearch.yml` | `go-coverage-rabbitmq.yml` | Not available |

Profiles fix service images, ports, health checks, readiness probes, and failure diagnostics. They do not accept service-image overrides. Redis and Elasticsearch coverage-with-Valkey profiles are not available yet; keep those jobs local until a published profile supports the exact contract.

Security has no service profile. `go-security.yml` runs the official mutable `securego/gosec@master` action with `args: ./...`. Findings fail the job; accepted exceptions remain versioned source annotations. It does not upload a security-report artifact or read CI-local scanner configuration.

Every Test workflow derives `ZONEINFO` from `go env GOROOT`, runs `go test -json ./...`, and uploads `test-results/go-test.json`. Set `tags: integration` to include build-tag-gated tests that a repository runs inside its Test check. Every Coverage workflow runs `TZ="" go test -v -coverprofile=cover.out ./...` untagged. Set `translate: true` only when the repository contract runs `make translate`; both default to unset/false.

## Go Integration

Dedicated Integration workflows run build-tag-gated tests that need databases. They are separate entrypoints per service profile:

| Services | Entrypoint |
| --- | --- |
| PostgreSQL | `go-integration-postgres.yml` |
| PostgreSQL + RabbitMQ | `go-integration-postgres-rabbitmq.yml` |

Each runs `go test -json -tags integration <test-path>` (default `./test/integration`) and uploads `integration-test-results`. Inputs:

```yml
with:
  app-client-id: ${{ vars.FATSOMA_DEPENDENCIES_APP_CLIENT_ID }}
  module-allowlist: |
    v2-api-auth
    v2-migration-tool
  env: |               # Optional; KEY=VALUE per line, exported before tests.
    APP_NAME=scanner
    DATABASE_HOST=127.0.0.1
    DATABASE_PORT=5432
  migrate: true        # Optional; defaults to false.
  migrations-dir: db   # Optional; defaults to db.
  test-path: ./test/integration  # Optional.
  postgres-db: service_scanner_sync_test
  postgres-user: integration_user
  postgres-password: integration_password
secrets: inherit
```

Migrations use `go run github.com/fatsoma/v2-migration-tool/cmd/migration-tool@latest up` from `migrations-dir`, deliberately tracking the latest migration tool. The caller job owns triggers, the same-repository pull-request guard, and `needs`.

Changing shared CI behavior requires a governed finding, two-repository validation, and an approved reusable-actions release PR.

You can use custom docker build instructions with a `ci-docker-build` make target:

```make
CI_DOCKER_IMAGE=

.PHONY: ci-docker-build
ci-docker-build:
	docker build --tag "$(CI_DOCKER_IMAGE)" .
```
