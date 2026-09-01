# reusable-actions

Public repository for reusable GitHub Actions workflows.

Callers pin every reusable workflow to the full immutable Git commit SHA resolved from a release tag. The SHA fixes the exact code executed. Replace `REUSABLE_ACTIONS_SHA` in the examples with the commit SHA of the release tag you want, and add the tag name as a trailing comment, for example `@0123456789abcdef... # v2`. [Dependabot updates for GitHub Actions](https://docs.github.com/en/code-security/how-tos/secure-your-supply-chain/secure-your-dependencies/auto-update-actions) can then propose bumps while reviewers see the human-readable release version.

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
    uses: Fatsoma/reusable-actions/.github/workflows/ruby-lint.yml@REUSABLE_ACTIONS_SHA # v2

  ruby-security:
    uses: Fatsoma/reusable-actions/.github/workflows/ruby-security.yml@REUSABLE_ACTIONS_SHA # v2

  ruby-test:
    uses: Fatsoma/reusable-actions/.github/workflows/ruby-test.yml@REUSABLE_ACTIONS_SHA # v2
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

You can use custom docker build instructions with a `ci-docker-build` make target:

```make
CI_DOCKER_IMAGE=

.PHONY: ci-docker-build
ci-docker-build:
	docker build --tag "$(CI_DOCKER_IMAGE)" .
```

## Ruby CI

The Ruby CI workflows standardize `Lint`, `Test`, and `Security` jobs for the Fatsoma Ruby cohort migrating from CircleCI. Each call uses an immutable `REUSABLE_ACTIONS_SHA`. Test result and coverage data come from rspec's built-in JSON formatter and SimpleCov's default `.last_run.json` — no extra reporting gems.

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

The `if:` guard on each caller job skips the job for pull requests opened from forks; without it, the same change runs twice on pull requests from branches inside the repository.

### Private gems

If the `Gemfile` git-sources private Fatsoma gems, pass the GitHub App credentials and a newline-delimited allowlist of the private repositories to read. Both `app-client-id` and `gem-allowlist` are required together; when `gem-allowlist` is omitted, no token is created and no private-gem access is configured.

```yml
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

`secrets: inherit` passes the caller repository's secrets to the reusable workflow, including `FATSOMA_DEPENDENCIES_APP_PRIVATE_KEY` when it is configured. Repositories without private gem dependencies can omit `secrets: inherit` entirely.

The workflow rewrites `git@github.com:Fatsoma/` URLs to token-authenticated HTTPS before `bundle install`; `Gemfile.lock` keeps recording the SSH remotes and needs no changes.

### Inputs

All three Ruby workflows accept the same optional inputs:

| Input | Default | Purpose |
| --- | --- | --- |
| `ruby-version` | `.ruby-version` | Ruby version for setup-ruby. |
| `app-client-id` | — | GitHub App client ID for private gem access. Required with `gem-allowlist`. |
| `gem-allowlist` | — | Newline-delimited private gem repositories for the GitHub App token. |

### What each workflow does

`ruby-lint.yml` checks out with full history, then runs rubocop with `--fail-level convention --force-exclusion` on the Ruby files changed in the pull request (merge-base of the base branch) or push (`github.event.before`); when no base resolves it lints the whole repo. The caller's `.rubocop.yml` is used when present; when absent, rubocop falls back to the default configuration inherited from the `rubocop-fatsoma-config` gem via the cohort Gemfiles (v2-spec-helpers is the one such repo — its gem is immutable, ADR-0006).

`ruby-test.yml` distributes the bundled fatsoma-settings gem's `.env.circle` into `$CONFIG_PATH/.env.{test,development,local}` (mirroring CircleCI's settings step), then runs `bundle exec rspec --format json --format documentation spec`, uploading the JSON test results and the coverage/artifacts directory. With `COVERAGE=true` set by the workflow, SimpleCov writes its HTML report and `.last_run.json` under `coverage/`; the workflow appends `total: **N%**` to the job's step summary and copies the report into `$CI_ARTIFACTS/coverage` for upload (the spec-helpers gem is immutable — no `CI_ARTIFACTS` starter exists in it, ADR-0006).

`ruby-integration-rabbitmq.yml` runs the full spec suite with `INTEGRATION=true` against a `rabbitmq:3.12-management-alpine` service container (ports 5672/15672; a broker health check replaces CircleCI's `dockerize -wait`), mirroring v2-queue's CircleCI integration job. Before the specs, it runs the caller's in-repo broker config script (input `rabbitmq-config-script`, default `./.circleci/config_rabbitmq.sh`) which declares the test topic exchange, dead-letter exchange, lazy retry queue and policies via the `rabbitmqadmin` CLI served on 15672. `WORKERS_MESSAGE_QUEUE_RETRY_{BASE,MULTIPLIER,LIMIT}` are set to `1/2/10` as in CircleCI. Results upload as `integration-results` / `integration-coverage`. Currently used by v2-queue only.

`ruby-security.yml` runs brakeman (`~> 5.0`, pinned for Ruby 2.7) with `--exit-on-warn` for Rails applications (detected by the presence of `app/`), uploading the HTML report as `security-scan-results`. The `EOLRails` and `EOLRuby` checks are excluded: the whole cohort runs Rails 5.2 on Ruby 2.7 (both EOL), those warnings are known and tracked, and upgrading is a programme outside CI migration scope — without the exclusion every app-containing repo would fail on exactly those two warnings (cohort-governance discovery 011). All other checks still gate. Dependency CVE scanning is deliberately absent: Dependabot alerts cover it. Non-Rails callers can omit the security job entirely.
