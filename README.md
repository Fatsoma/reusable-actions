# reusable-actions

Reusable GitHub Actions workflows for Fatsoma repositories.

Callers pin every reusable workflow to the full immutable Git commit SHA resolved from a release tag — the SHA fixes the exact code executed. Add the tag name as a trailing comment, for example `@0123456789abcdef... # v2`, so reviewers see the human-readable release version. [Dependabot updates for GitHub Actions](https://docs.github.com/en/code-security/how-tos/secure-your-supply-chain/secure-your-dependencies/auto-update-actions) can then propose bumps. Replace `REUSABLE_ACTIONS_SHA` in the examples with the commit SHA of the release tag you want.

Caller workflows typically trigger on pushes to the default branch, pull requests, and manual dispatch:

```yml
on:
  push:
    branches: [master]
  pull_request:
  workflow_dispatch:
```

Each job uploads its reports as workflow-run artifacts, downloadable from the run's summary page.

## Onboarding a new repository

1. Grant the repository access to private dependencies if it needs them — see [Private dependencies](#private-dependencies). This step requires an organisation admin.
2. Create `.github/workflows/ci.yml` with the triggers above and `permissions: contents: read`.
3. Copy the quickstart for your language — [Go](#go-ci) or [Ruby](#ruby-ci) — plus an [ECR push](#ecr-push) job if the repository ships a Docker image. All jobs live in the single `ci.yml` — the convention is one caller file per repository, not one file per reusable workflow.
4. Pin every `uses:` line to the release-tag SHA with the trailing tag comment (e.g. `# v2`).

A complete minimal Go `ci.yml` looks like this:

```yml
name: CI

on:
  push:
    branches: [master]
  pull_request:
  workflow_dispatch:

permissions:
  contents: read

jobs:
  test:
    if: github.event_name != 'pull_request' || github.event.pull_request.head.repo.full_name == github.repository
    uses: Fatsoma/reusable-actions/.github/workflows/go-test.yml@REUSABLE_ACTIONS_SHA # v2

  coverage:
    if: github.event_name != 'pull_request' || github.event.pull_request.head.repo.full_name == github.repository
    uses: Fatsoma/reusable-actions/.github/workflows/go-coverage.yml@REUSABLE_ACTIONS_SHA # v2

  security:
    if: github.event_name != 'pull_request' || github.event.pull_request.head.repo.full_name == github.repository
    uses: Fatsoma/reusable-actions/.github/workflows/go-security.yml@REUSABLE_ACTIONS_SHA # v2
```

The `if:` guard on each caller job skips the job for pull requests opened from forks; without it, the same change runs twice on pull requests from branches inside the repository.

## Private dependencies

Both the Go and Ruby workflows follow the same pattern when a repository depends on private Fatsoma repositories: the caller passes the GitHub App client ID plus a newline-delimited allowlist of the private repositories to read, and `secrets: inherit` passes the caller's secrets — including `FATSOMA_DEPENDENCIES_APP_PRIVATE_KEY` — to the reusable workflow. Both `app-client-id` and the allowlist input are required together; when the allowlist is omitted, no token is created and no private access is configured. The workflow rewrites `github.com` remote URLs (HTTPS or SSH) to token-authenticated HTTPS before dependency installation; manifests and lockfiles keep recording their original remotes and need no changes.

Before this works for a new repository, an organisation admin must grant access in two places:

1. Add the repository under the GitHub App's installation repository access: [Fatsoma Dependencies installation settings](https://github.com/organizations/Fatsoma/settings/installations/152920273).
2. Add the repository to the organisation secret's selected repositories: [FATSOMA_DEPENDENCIES_APP_PRIVATE_KEY secret settings](https://github.com/organizations/Fatsoma/settings/secrets/actions/FATSOMA_DEPENDENCIES_APP_PRIVATE_KEY).

`FATSOMA_DEPENDENCIES_APP_CLIENT_ID` is an organisation variable available to all repositories; it needs no per-repository scoping.

Ruby apps whose Gemfile sources Rails LTS also need the `BUNDLE_GEMS__RAILSLTS__COM` organisation secret (`username:password` for gems.railslts.com) granted to the repository; bundler reads the variable natively, so the value needs no mapping.

## Go CI

The Go CI workflows standardize `Test`, `Coverage`, `Security`, and `Integration` jobs. Test results come from `go test -json` and coverage from `go tool cover` — no JUnit conversion or extra tooling.

A minimal repository — public modules only — needs no inputs at all:

```yml
permissions:
  contents: read

jobs:
  test:
    if: github.event_name != 'pull_request' || github.event.pull_request.head.repo.full_name == github.repository
    uses: Fatsoma/reusable-actions/.github/workflows/go-test.yml@REUSABLE_ACTIONS_SHA # v2

  coverage:
    if: github.event_name != 'pull_request' || github.event.pull_request.head.repo.full_name == github.repository
    uses: Fatsoma/reusable-actions/.github/workflows/go-coverage.yml@REUSABLE_ACTIONS_SHA # v2

  security:
    if: github.event_name != 'pull_request' || github.event.pull_request.head.repo.full_name == github.repository
    uses: Fatsoma/reusable-actions/.github/workflows/go-security.yml@REUSABLE_ACTIONS_SHA # v2
```

Repositories with integration tests add an integration job gated behind the unit jobs:

```yml
jobs:
  # test, coverage, security jobs as above

  integration:
    if: github.event_name != 'pull_request' || github.event.pull_request.head.repo.full_name == github.repository
    needs: [test, coverage, security]
    uses: Fatsoma/reusable-actions/.github/workflows/go-integration.yml@REUSABLE_ACTIONS_SHA # v2
```

### Private Go modules

If `go.mod` requires private Fatsoma modules, pass the GitHub App credentials and a newline-delimited allowlist of the private repositories to read (see [Private dependencies](#private-dependencies) for the one-time organisation setup):

```yml
jobs:
  test:
    if: github.event_name != 'pull_request' || github.event.pull_request.head.repo.full_name == github.repository
    uses: Fatsoma/reusable-actions/.github/workflows/go-test.yml@REUSABLE_ACTIONS_SHA # v2
    with:
      app-client-id: ${{ vars.FATSOMA_DEPENDENCIES_APP_CLIENT_ID }}
      module-allowlist: |
        v2-api-auth
        v2-api-httpclient
    secrets: inherit
```

Repositories without private module dependencies can omit `secrets: inherit` entirely.

### Inputs

| Input              | Default              | Workflows                   | Purpose                                                                           |
| ------------------ | -------------------- | --------------------------- | --------------------------------------------------------------------------------- |
| `app-client-id`    | —                    | all                         | GitHub App client ID for private module access. Required with `module-allowlist`. |
| `module-allowlist` | —                    | all                         | Newline-delimited private module repositories for the GitHub App token.           |
| `translate`        | `false`              | test, coverage, integration | Merge goi18n translation files before the run.                                    |
| `env`              | `""`                 | integration                 | Newline-delimited `KEY=VALUE` integration environment variables.                  |
| `test-path`        | `./test/integration` | integration                 | Package path for the integration tests.                                           |

The postgres integration profiles additionally accept `migrate` (run migrations before the tests), `migrations-dir`, and `postgres-db` / `postgres-user` / `postgres-password`.

### What each workflow does

`go-test.yml` runs `go test -json ./...` and uploads the JSON results as the `test-results` artifact. `ZONEINFO` is derived from `go env GOROOT` so tests use the installed Go timezone data. The Go version comes from the caller's `go.mod`.

`go-coverage.yml` runs `go test -v -coverprofile` with `TZ=""`, renders an HTML report with `go tool cover`, uploads it as the `coverage` artifact, and appends `total: **N%**` to the job's step summary.

`go-security.yml` runs gosec with `args: ./...`. It deliberately tracks the mutable `securego/gosec@master` so new checks reach every caller as soon as they land — reduced time to discovery for security issues — and produces no report artifact.

`go-integration.yml` and its per-service variants run `go test -json -tags integration` against the `test-path` package, uploading results as `integration-test-results`. One workflow file exists per service combination — conditional `services:` are not valid in reusable workflows:

| Workflow                                  | Services                     |
| ----------------------------------------- | ---------------------------- |
| `go-integration.yml`                      | none                         |
| `go-integration-redis.yml`                | redis                        |
| `go-integration-valkey.yml`               | valkey                       |
| `go-integration-elasticsearch.yml`        | elasticsearch                |
| `go-integration-elasticsearch-valkey.yml` | elasticsearch + valkey       |
| `go-integration-postgres.yml`             | postgres                     |
| `go-integration-postgres-rabbitmq.yml`    | postgres + rabbitmq          |
| `go-integration-rabbitmq.yml`             | rabbitmq (no current caller) |

All variants keep the job name `Integration` so switching between them keeps the same required check. Service-backed test/coverage variants (`go-test-elasticsearch.yml`, `go-test-elasticsearch-valkey.yml`, `go-coverage-elasticsearch.yml`) likewise keep the `Test`/`Coverage` job names.

## Ruby CI

The Ruby CI workflows standardize `Lint`, `Test`, and `Security` jobs. Test result and coverage data come from rspec's built-in JSON formatter and SimpleCov's default `.last_run.json` — no extra reporting gems. Unlike the Go workflows, Ruby apps generally do not split unit and integration specs into separate CI jobs: service-dependent specs run inside the main `rspec spec` run, so a repository that needs services swaps `ruby-test.yml` for a service-backed variant rather than adding an integration job.

A minimal repository — public gems only — needs no inputs at all:

```yml
permissions:
  contents: read

jobs:
  lint:
    if: github.event_name != 'pull_request' || github.event.pull_request.head.repo.full_name == github.repository
    uses: Fatsoma/reusable-actions/.github/workflows/ruby-lint.yml@REUSABLE_ACTIONS_SHA # v2

  test:
    if: github.event_name != 'pull_request' || github.event.pull_request.head.repo.full_name == github.repository
    uses: Fatsoma/reusable-actions/.github/workflows/ruby-test.yml@REUSABLE_ACTIONS_SHA # v2

  security:
    if: github.event_name != 'pull_request' || github.event.pull_request.head.repo.full_name == github.repository
    uses: Fatsoma/reusable-actions/.github/workflows/ruby-security.yml@REUSABLE_ACTIONS_SHA # v2
```

### Private gems

If the `Gemfile` git-sources private Fatsoma gems, pass the GitHub App credentials and a newline-delimited allowlist of the private repositories to read (see [Private dependencies](#private-dependencies) for the one-time organisation setup):

```yml
jobs:
  test:
    if: github.event_name != 'pull_request' || github.event.pull_request.head.repo.full_name == github.repository
    uses: Fatsoma/reusable-actions/.github/workflows/ruby-test.yml@REUSABLE_ACTIONS_SHA # v2
    with:
      app-client-id: ${{ vars.FATSOMA_DEPENDENCIES_APP_CLIENT_ID }}
      gem-allowlist: |
        v2-spec-helpers
        rubocop-fatsoma-config
    secrets: inherit
```

Repositories without private gem dependencies can omit `secrets: inherit` entirely. The workflow rewrites `git@github.com:Fatsoma/` URLs to token-authenticated HTTPS before `bundle install`; `Gemfile.lock` keeps recording the SSH remotes and needs no changes.

### Inputs

The lint, test, and security workflows accept the same optional inputs; the service-backed test variants add their own (see below).

| Input              | Default         | Purpose                                                                     |
| ------------------ | --------------- | --------------------------------------------------------------------------- |
| `ruby-version`     | `.ruby-version` | Ruby version for setup-ruby.                                                |
| `app-client-id`    | —               | GitHub App client ID for private gem access. Required with `gem-allowlist`. |
| `gem-allowlist`    | —               | Newline-delimited private gem repositories for the GitHub App token.        |
| `prepare-database` | `true`          | Run `db:create db:test:prepare` before running specs (postgres variants).   |

The service-backed test variants run `db:create db:test:prepare` before the specs when `prepare-database` is true (the default); the whole cohort is `schema_format :sql`, so `db:test:prepare` loads `db/structure.sql` after checking for pending migrations.

### What each workflow does

`ruby-lint.yml` runs rubocop with `--fail-level convention --force-exclusion` on the Ruby files changed in the pull request or push; when no base resolves it lints the whole repo. The caller's `.rubocop.yml` is used when present.

`ruby-test.yml` distributes the bundled fatsoma-settings gem's `.env.circle` into `$CONFIG_PATH/.env.{test,development,local}`, then runs `bundle exec rspec --format json --format documentation spec`, uploading the JSON test results and the coverage report. SimpleCov writes its HTML report and `.last_run.json` under `coverage/`; the workflow appends `total: **N%**` to the job's step summary.

`ruby-test-rabbitmq.yml`, `ruby-test-postgres-redis.yml`, `ruby-test-postgres-rabbitmq-redis.yml`, and `ruby-test-postgres-rabbitmq-redis-elasticsearch.yml` are the service-backed variants of `ruby-test.yml` — one workflow per service combination, because conditional `services:` are not valid in reusable workflows. All keep the job name `Test` so switching from `ruby-test.yml` keeps the same required check.

`ruby-security.yml` runs brakeman (`~> 5.0`, pinned for Ruby 2.7) for Rails applications (detected by the presence of `app/`), uploading the HTML report as `security-scan-results`. The scan is advisory, not a gate: the previous CI ran brakeman without failing on warnings, so every repo carries an unmeasured backlog — the report exists for triage, and turning it into a gate is a separate decision. The `EOLRails` and `EOLRuby` checks are excluded because Fatsoma's Rails apps run EOL Rails 5.2 on Ruby 2.7. Dependency CVE scanning is deliberately absent: Dependabot alerts cover it. Non-Rails callers can omit the security job entirely.

### Gem publish

`ruby-gem-publish` builds the gem from `<gem-name>.gemspec` and pushes it to GitHub Packages under the repository's owner. Callers place it in a workflow triggered on push to the default branch rather than in the pull-request CI workflow, and pass the `BUNDLE_RUBYGEMS__PKG__GITHUB__COM` secret:

```yml
on:
  push:
    branches: master

jobs:
  ruby-gem-publish:
    uses: Fatsoma/reusable-actions/.github/workflows/ruby-gem-publish.yml@REUSABLE_ACTIONS_SHA # v2
    with:
      gem-name: example-gem
    secrets: inherit
```

## ECR push

`ecr-push.yml` builds a Docker image and pushes it to an ECR repository:

```yml
jobs:
  ecr-push:
    uses: Fatsoma/reusable-actions/.github/workflows/ecr-push.yml@REUSABLE_ACTIONS_SHA # v2
    with:
      aws-region: us-west-1
      ecr-repository: ${{ github.event.repository.name }}
      environment: staging
      image-tag: latest
```

| Input            | Default        | Purpose                                     |
| ---------------- | -------------- | ------------------------------------------- |
| `aws-account`    | `819738237059` | AWS account ID (numeric form).              |
| `aws-region`     | —              | AWS region of the ECR repository.           |
| `ecr-repository` | —              | ECR repository name.                        |
| `environment`    | —              | App environment (e.g. staging, production). |
| `image-tag`      | —              | Tag for the image build.                    |

The workflow assumes the IAM role `arn:aws:iam::<aws-account>:role/gha-<environment>-<repository>` via OIDC, so the role must exist with a trust policy for the caller repository before the first run.

You can use custom docker build instructions with a `ci-docker-build` make target:

```make
CI_DOCKER_IMAGE=

.PHONY: ci-docker-build
ci-docker-build:
	docker build --tag "$(CI_DOCKER_IMAGE)" .
```
