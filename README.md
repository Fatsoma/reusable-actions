# reusable-actions

Public repository for reusable GitHub Actions workflows.

Callers pin every reusable workflow to the full immutable Git commit SHA resolved from a release tag. The SHA fixes the exact code executed. Replace `REUSABLE_ACTIONS_SHA` in the examples with the commit SHA of the release tag you want, and add the tag name as a trailing comment, for example `@0123456789abcdef... # v1`. [Dependabot updates for GitHub Actions](https://docs.github.com/en/code-security/how-tos/secure-your-supply-chain/secure-your-dependencies/auto-update-actions) can then propose bumps while reviewers see the human-readable release version.

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

  ruby-lint:
    uses: Fatsoma/reusable-actions/.github/workflows/ruby-lint.yml@REUSABLE_ACTIONS_SHA # v1

  ruby-vulnerabilities:
    uses: Fatsoma/reusable-actions/.github/workflows/ruby-vulnerabilities.yml@REUSABLE_ACTIONS_SHA # v1

  ruby-test:
    uses: Fatsoma/reusable-actions/.github/workflows/ruby-test.yml@REUSABLE_ACTIONS_SHA # v1
```

`ruby-gem-publish` is triggered on push to `main` rather than on `workflow_call` from a job:

```yml
on:
  push:
    branches: main

jobs:
  ruby-gem-publish:
    uses: Fatsoma/reusable-actions/.github/workflows/ruby-gem-publish.yml@REUSABLE_ACTIONS_SHA # v1
    with:
      gem-name: example-gem
```

## Go CI

The Go CI workflows standardize `Test`, `Coverage`, and `Security` jobs for the supported service profiles below. Each call uses an immutable `REUSABLE_ACTIONS_SHA`.

Test workflows use Go's native JSON output and need no reporting tool. Security runs the official `securego/gosec@master` action with its default `./...` arguments, deliberately tracking the latest gosec development.

A minimal repository — public Go modules only, no translations, no integration services — needs no inputs at all:

```yml
permissions:
  contents: read

jobs:
  test:
    if: github.event_name != 'pull_request' || github.event.pull_request.head.repo.full_name == github.repository
    uses: Fatsoma/reusable-actions/.github/workflows/go-test.yml@REUSABLE_ACTIONS_SHA # v1

  coverage:
    if: github.event_name != 'pull_request' || github.event.pull_request.head.repo.full_name == github.repository
    uses: Fatsoma/reusable-actions/.github/workflows/go-coverage.yml@REUSABLE_ACTIONS_SHA # v1

  security:
    if: github.event_name != 'pull_request' || github.event.pull_request.head.repo.full_name == github.repository
    uses: Fatsoma/reusable-actions/.github/workflows/go-security.yml@REUSABLE_ACTIONS_SHA # v1
```

The `if:` guard on each caller job skips the job for pull requests opened from forks; without it, the same change runs twice on pull requests from branches inside the repository.

### Private Go modules

If `go.mod` depends on private Fatsoma modules, pass the GitHub App credentials and a newline-delimited allowlist of the private repositories to read. Both `app-client-id` and `module-allowlist` are required together; when `module-allowlist` is omitted, no token is created and no private-module access is configured.

```yml
  test:
    if: github.event_name != 'pull_request' || github.event.pull_request.head.repo.full_name == github.repository
    uses: Fatsoma/reusable-actions/.github/workflows/go-test.yml@REUSABLE_ACTIONS_SHA # v1
    with:
      app-client-id: ${{ vars.FATSOMA_DEPENDENCIES_APP_CLIENT_ID }}
      module-allowlist: |
        v2-api-auth
        v2-api-fatsoma-api
    secrets: inherit
```

`secrets: inherit` passes the caller repository's secrets to the reusable workflow, including `FATSOMA_DEPENDENCIES_APP_PRIVATE_KEY` when it is configured. Repositories without private module dependencies can omit `secrets: inherit` entirely.

### Test and Coverage inputs

All Test and Coverage entrypoints accept the same optional inputs:

| Input | Default | Purpose |
| --- | --- | --- |
| `app-client-id` | — | GitHub App client ID for private module access. Required with `module-allowlist`. |
| `module-allowlist` | — | Newline-delimited private module repositories for the GitHub App token. |
| `translate` | `false` | Merge `configs/locales/en-gb.*.json` goi18n translation files before running. |

Unit and integration tests are always separated. Test and Coverage run unit tests only; integration tests run in a dedicated Integration job. Test and Coverage are service-free by default, but a repository whose application wiring connects to a datastore at startup needs that service for its unit tests, so it uses a service-backed Test/Coverage profile.

Test/Coverage entrypoints:

| Unit needs | Test | Coverage |
| --- | --- | --- |
| No services | `go-test.yml` | `go-coverage.yml` |
| Elasticsearch | `go-test-elasticsearch.yml` | `go-coverage-elasticsearch.yml` |
| Elasticsearch + Valkey | `go-test-elasticsearch-valkey.yml` | — |

`go-test*.yml` derives `ZONEINFO` from `go env GOROOT`, runs `go test -json ./...`, and uploads `test-results/go-test.json`. `go-coverage*.yml` runs `TZ="" go test -v -coverprofile=cover.out ./...`.

Security has no service profile. `go-security.yml` runs the official mutable `securego/gosec@master` action with `args: ./...`. Findings fail the job; accepted exceptions remain versioned source annotations. It does not upload a security-report artifact or read CI-local scanner configuration.

## Go Integration

Integration tests run in a dedicated job gated on `needs: [test, coverage, security]`, with the service profile the integration suite requires. Each entrypoint runs `go test -json -tags integration <test-path>` (default `./test/integration`) and uploads `integration-test-results`:

| Services | Entrypoint |
| --- | --- |
| None | `go-integration.yml` |
| Redis | `go-integration-redis.yml` |
| Valkey | `go-integration-valkey.yml` |
| RabbitMQ | `go-integration-rabbitmq.yml` |
| Elasticsearch | `go-integration-elasticsearch.yml` |
| Elasticsearch + Valkey | `go-integration-elasticsearch-valkey.yml` |
| PostgreSQL | `go-integration-postgres.yml` |
| PostgreSQL + RabbitMQ | `go-integration-postgres-rabbitmq.yml` |

Shared optional inputs: `app-client-id` and `module-allowlist` (both required together for private module access, as above), `env` (newline `KEY=VALUE`, exported before tests), `translate`, and `test-path`. Postgres profiles additionally accept `migrate`, `migrations-dir`, and `postgres-db` / `postgres-user` / `postgres-password`.

```yml
  integration:
    needs: [test, coverage, security]
    uses: Fatsoma/reusable-actions/.github/workflows/go-integration-postgres.yml@REUSABLE_ACTIONS_SHA # v1
    with:
      app-client-id: ${{ vars.FATSOMA_DEPENDENCIES_APP_CLIENT_ID }}
      module-allowlist: |
        v2-api-auth
        v2-migration-tool
      env: |
        APP_NAME=payment
        DATABASE_HOST=127.0.0.1
        DATABASE_PORT=5432
      migrate: true
      postgres-db: api_payments_test
      postgres-user: integration_user
      postgres-password: integration_password
    secrets: inherit
```

Migrations use `go run github.com/fatsoma/v2-migration-tool/cmd/migration-tool@latest up` from `migrations-dir`, deliberately tracking the latest migration tool. The integration caller job gates on `needs: [test, coverage, security]` and owns triggers and the same-repository pull-request guard.

You can use custom docker build instructions with a `ci-docker-build` make target:

```make
CI_DOCKER_IMAGE=

.PHONY: ci-docker-build
ci-docker-build:
	docker build --tag "$(CI_DOCKER_IMAGE)" .
```
