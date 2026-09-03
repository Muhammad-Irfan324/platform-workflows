# platform-workflows

Centralised reusable GitHub Actions workflows for platform infrastructure.

Module repos and deployment repos carry only a small caller file per workflow; everything
else lives here.

## Layout

```
.github/workflows/     the reusable workflows
actions/               composite actions used by the workflows
configs/               shared config: lint defaults copied into module repos,
                       plus the pinned npm toolchain the release workflows read
versions.env           CI tool versions
```

## How it fits together

### Module repo — terraform-aws-ecr, terraform-aws-eks, …

```
┌──────────────────────────────────────────────────────────────┐
│  Developer changes a module                                  │
└────────────────────┬─────────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────────┐
│  Open a pull request                                         │
└────────────────────┬─────────────────────────────────────────┘
                     │
        ┌────────────┴────────────────┐
        ▼                             ▼
┌───────────────────────┐   ┌──────────────────────────────┐
│ terraform-module-ci   │   │ checkov-terraform            │
│ ├─ terraform fmt      │   │ ├─ validate .checkov.yaml    │
│ ├─ terraform init     │   │ │   (skip-check / skip-path  │
│ ├─ terraform validate │   │ │    only — no soft-fail)    │
│ ├─ tflint             │   │ ├─ scan                      │
│ ├─ terraform-docs     │   │ └─ annotate findings on PR   │
│ ├─ validate examples  │   └──────────────┬───────────────┘
│ └─ terraform test     │                  │
└───────────┬───────────┘                  │
            └──────────────┬───────────────┘
                           ▼
                    ┌─────────────┐
                    │  All pass?  │
                    └──────┬──────┘
                           │ yes
                           ▼
                 ┌─────────────────────┐
                 │ Review & merge      │
                 │ to main             │
                 └──────────┬──────────┘
                            │
                            ▼
                 ┌─────────────────────┐
                 │ terraform-release   │
                 │ reads commit types  │
                 └──────────┬──────────┘
                            │
                            ▼
                    ┌───────────────┐
                    │ fix / feat /  │
                    │ BREAKING ?    │
                    └───────┬───────┘
                            │ yes
                            ▼
                 ┌─────────────────────┐
                 │ Tag v1.0.0          │
                 │ + CHANGELOG.md      │
                 │ + GitHub release    │
                 └─────────────────────┘
```

### Deployment repo — infra

Plan and apply are independent. Neither calls the other, and neither assumes a trigger —
the caller decides. Today that is a dispatch with an action picker:

```
┌──────────────────────────────────────────────────────────────┐
│  Deployment repo pins a published module                     │
│  source = "git::…/terraform-aws-ecr.git?ref=v1.0.0"          │
└────────────────────┬─────────────────────────────────────────┘
                     │
                     ▼
        ┌────────────────────────────────┐
        │ workflow_dispatch              │
        │ resource · environment · action│
        └───────────────┬────────────────┘
                        │
                 ┌──────┴───────┐
                 │   action ?   │
                 └──────┬───────┘
          ┌─────────────┴──────────────┐
          │ Terraform Plan             │ Terraform Apply
          ▼                            ▼
┌──────────────────────┐    ┌──────────────────────────┐
│ terraform-plan       │    │ terraform-apply          │
│ ├─ assume role, OIDC │    │ ├─ assume role, OIDC     │
│ ├─ init              │    │ ├─ environment gate      │
│ ├─ validate          │    │ │   (approval / timer)   │
│ ├─ plan              │    │ ├─ init                  │
│ └─ comment on the PR │    │ ├─ plan -out=tfplan      │
│    (pass or fail)    │    │ └─ apply tfplan          │
└──────────────────────┘    └────────────┬─────────────┘
     no infrastructure                   │
     is changed                          ▼
                                  ┌─────────────┐
                                  │     AWS     │
                                  └─────────────┘
```

A repo wanting plan on pull requests and apply on merge wires it that way instead — the
workflows carry no trigger policy of their own:

```
  Pull request ──────────►  terraform-plan
  Merge to main ─────────►  terraform-apply  ──────►  AWS
```

The module tag is the only thing crossing between the two repo types: modules are
published, deployments consume them at a pinned version. This repo never touches AWS,
module repos never create infrastructure, and the deployment repo holds no reusable logic.

### Application repo — sample-node-app, …

```
┌──────────────────────────────────────────────────────────────┐
│  Developer pushes code                                       │
└────────────────────┬─────────────────────────────────────────┘
                     │
                     ▼
┌──────────────────────────────────────────────────────────────┐
│  Open a pull request                                         │
└────────────────────┬─────────────────────────────────────────┘
                     │
        ┌────────────┴────────────────┐
        ▼                             ▼
┌───────────────────────┐   ┌──────────────────────────────┐
│ lint-and-test         │   │ security-scan                │
│ ├─ npm ci             │   │ ├─ SonarQube (SAST)          │
│ ├─ lint               │   │ ├─ OWASP Dependency-Check    │
│ ├─ unit tests         │   │ └─ Gitleaks (secret scan)    │
│ └─ integration tests  │   └──────────────┬───────────────┘
└───────────┬───────────┘                  │
            │                              │
            ▼                              │
┌──────────────────────────┐               │
│ docker-build-scan        │               │
│ ├─ Docker build (Buildx) │               │
│ ├─ Trivy (image scan)    │               │
│ └─ SARIF → Security tab  │               │
└───────────┬──────────────┘               │
            └──────────────┬───────────────┘
                           ▼
                    ┌─────────────┐
                    │  All pass?  │
                    └──────┬──────┘
                           │ yes
                           ▼
                 ┌─────────────────────┐
                 │ Review & merge      │
                 │ to main             │
                 └──────────┬──────────┘
                            │
                            ▼
                 ┌──────────────────────────┐
                 │ docker-build-scan        │
                 │ ├─ rebuild + scan        │
                 │ ├─ SARIF → Security tab  │
                 │ └─ push to GitLab CR     │
                 │     + provenance + SBOM  │
                 └──────────────────────────┘
```

### Central configuration feeds both

```
                    ┌──────────────────────────────┐
                    │ versions.env                 │
                    │ tflint · terraform-docs      │
                    │ node · actionlint            │
                    └───────────────┬──────────────┘
                                    │
                    ┌───────────────┴──────────────┐
                    │ configs/                     │
                    │ .tflint.hcl.tmpl             │
                    │ .terraform-docs.yaml         │
                    │ .releaserc.json              │
                    └───────────────┬──────────────┘
                                    │
                 ┌──────────────────┴──────────────────┐
                 ▼                                     ▼
      ┌─────────────────────┐               ┌─────────────────────┐
      │ terraform-module-ci │               │ terraform-release   │
      │ applies lint config │               │ applies release cfg │
      │ if repo has none    │               │ if repo has none    │
      └─────────────────────┘               └─────────────────────┘
```

## What each workflow buys you

| Workflow | Without it | With it |
|---|---|---|
| `terraform-module-ci` | Module quality depends on the reviewer noticing | Formatting, validity, linting, docs drift and examples checked on every PR |
| `checkov-terraform` | Security findings surface after deploy | Findings annotated inline on the PR, with a suppression schema a repo cannot widen |
| `terraform-release` | Versions tagged by hand, changelog written by hand | Version and changelog derived from commit messages |
| `terraform-plan` | Reviewers approve a diff they never opened | Plan posted into the PR, and a failing plan fails the check |
| `terraform-apply` | Apply recalculates its own diff after approval | Plan and apply are one atomic pair behind an environment gate |
| `application-ci` | Each app repo builds its own scanning pipeline | Lint, test, SonarQube, OWASP, Gitleaks, Docker build and Trivy in one call |
| `versions.env` + `configs/` | Every module repo carries its own lint config and they drift | One definition, applied on the next run everywhere |

## Workflows

| Workflow | Purpose | Used by |
|----------|---------|---------|
| [`terraform-module-ci.yaml`](.github/workflows/terraform-module-ci.yaml) | fmt, init, validate, tflint, terraform-docs, examples, tests | Module repos |
| [`terraform-release.yaml`](.github/workflows/terraform-release.yaml) | Publish a module version from conventional commits | Module repos |
| [`terraform-plan.yaml`](.github/workflows/terraform-plan.yaml) | Plan with OIDC auth, comment the result on the PR | Deployment repos |
| [`terraform-apply.yaml`](.github/workflows/terraform-apply.yaml) | Apply the reviewed plan behind an environment gate | Deployment repos |
| [`application-ci.yaml`](.github/workflows/application-ci.yaml) | Lint, test, security scan (SonarQube, OWASP, Gitleaks, Trivy), Docker build & push | Application repos |
| [`checkov-terraform.yaml`](.github/workflows/checkov-terraform.yaml) | Checkov scan with enforcement modes and inline annotations | Any repo |
| [`ci.yaml`](.github/workflows/ci.yaml) | actionlint, yamllint, SHA-pin gate, release guard | This repo |
| [`release.yaml`](.github/workflows/release.yaml) | Version this repo and move the `v1` tag | This repo |

Each file opens with a header covering its purpose, what it requires, what it grants
and a usage example. The authoritative input list is the `on: workflow_call: inputs:`
block in the file itself — it carries a `description` per input and cannot drift from
the code the way a second copy in a header would.

## Pinning

```yaml
uses: Muhammad-Irfan324/platform-workflows/.github/workflows/terraform-module-ci.yaml@v1
```

`v1` moves forward with each release, so fixes arrive automatically. A breaking change
cuts `v2`, and repos migrate when they choose.

Use `@v1` everywhere, including apply. `v1` does not track `main` — it only moves when a
release is cut, which requires a reviewed, merged pull request here. Pinning apply to an
exact tag instead sounds safer but is not: nobody bumps a manual pin, so apply drifts
behind plan and the two end up running different versions of the logic against the same
directory. Consistency is worth more than the small extra margin.

Pin an exact tag only to hold a specific repo back while you investigate something. When
you do, set `workflows-ref` to the same tag. GitHub tells a reusable workflow the
*caller's* ref, never its own, so the workflows that read `versions.env` and `configs/` at
runtime cannot derive it — the input is how they are told. Left at its `v1` default they
read the latest `v1`, which is right for everyone on `@v1` and wrong for a repo held back.

Because every example here pins `@v1`, nothing in this README carries a version that goes
stale — there is no version string to keep updated.

## Releasing this repo

`release.yaml` runs on every push to main and decides what to do from the commit messages
since the last release. What keeps a broken commit from being tagged is the required checks
on the pull request, not `release.yaml` itself — the two workflows fire on the same push and
run alongside each other, so a release is never waiting on that run. `ci.yaml` also triggers
on the push so a direct push or admin bypass is still linted, but that is a record, not a
gate:

| Commit starts with | Result |
|---|---|
| `fix:` | patch — `1.2.0` → `1.2.1` |
| `feat:` | minor — `1.2.0` → `1.3.0` |
| `feat!:` or a `BREAKING CHANGE:` footer | major — `1.2.0` → `2.0.0` |
| `chore:` `docs:` `ci:` `refactor:` `test:` `style:` | **no release** |

A release creates the version tag, publishes a GitHub release, and moves `v1` to point at
it. Consumers on `@v1` pick it up on their next run.

Dependabot's own pull requests are prefixed `fix`, not `ci`, so a bumped action or npm pin
cuts a patch release and reaches consumers. Prefixed `ci` they merged, released nothing,
and left every repo on `@v1` running the superseded version.

For the current version and what changed in it, see
[Releases](https://github.com/Muhammad-Irfan324/platform-workflows/releases) — the notes
are generated from the commit messages.

This repo keeps no `CHANGELOG.md`. The Releases page already carries the same information,
and skipping it means a release never has to commit back to `main`. Module repos do get
one, because their changelog is documentation for the teams consuming the module.

Three things worth knowing:

**Every commit message counts, not just the pull request title.** This repo squashes with
`squash_merge_commit_message = COMMIT_MESSAGES`, so the squashed commit's body is every
commit message in the PR concatenated. A `BREAKING CHANGE:` footer in any one of them
reaches `main` and cuts a major — even under a PR titled `chore:`. The subject follows
`COMMIT_OR_PR_TITLE`: a PR with one commit uses that commit's own message, and only a PR
with two or more commits falls back to the PR title.

So a PR is not safely "just a chore" because its title says so. Read the commits.

**`ci.yaml` guards the one thing that cannot be derived.** `workflows-ref` carries a
hardcoded major, so the release guard reads the same signals — PR title plus every commit
— and fails the PR if a major is coming while that default still names the old one. The
fix is then one line in the branch that caused it, rather than a surprise after release.

**A major bump does not move `v1`.** `2.0.0` creates `v2`; repos stay on `v1` until they
change their `uses:` line deliberately. That is the whole point of the moving tag.

## Module repos

```yaml
name: Module CI
on:
  pull_request:
    branches: [main]

permissions:
  contents: read

jobs:
  module_ci:
    name: "Module CI"
    uses: Muhammad-Irfan324/platform-workflows/.github/workflows/terraform-module-ci.yaml@v1
    with:
      working-directory: "."          # optional, default "."
      terraform-version: "~1.15.9"     # optional, default "~1.15.9"
      terraform-docs-check: true      # optional, default true
      validate-examples: true         # optional, default true
      require-tests: false            # optional, default false
      workflows-ref: "v1"         # optional, default "v1" — match the @ref above
    secrets:
      AUTOMATION_PAT: ${{ secrets.AUTOMATION_PAT }}

  checkov:
    name: "Checkov Terraform"
    uses: Muhammad-Irfan324/platform-workflows/.github/workflows/checkov-terraform.yaml@v1
    with:
      enforcement: blocking           # optional, default "blocking"; or report-only
```

Checkov runs as its own job, not inside Module CI — running both means two scans per PR
under different configurations.

Do not add a `paths:` filter. The workflow skips absent examples and tests on its own, and
a filter means a PR touching only `README.md` or `.checkov.yaml` runs nothing — and once
these are required checks, one that never starts leaves the pull request unmergeable,
waiting on a check that will never report.

Both jobs cancel superseded runs: push twice in quick succession and the first run stops
rather than finishing alongside the second.

Turn on `require-tests` once a module has `.tftest.hcl` files. With it off, a module with
no tests reports ⏭️ and passes — which is right while none exist, but later it would let
someone delete the test file and still get a green check.

### Release

```yaml
name: Release
on:
  push:
    branches: [main]

permissions:
  contents: write        # semantic-release pushes the tag, CHANGELOG and release
  issues: write          # "included in release" comments on shipped issues
  pull-requests: write   # and on shipped pull requests

jobs:
  release:
    uses: Muhammad-Irfan324/platform-workflows/.github/workflows/terraform-release.yaml@v1
    with:
      workflows-ref: "v1"         # optional, default "v1" — match the @ref above
    secrets:
      TOKEN: ${{ secrets.GITHUB_TOKEN }}
      AUTOMATION_PAT: ${{ secrets.AUTOMATION_PAT }}
```

`fix:` → patch, `feat:` → minor, `BREAKING CHANGE` → major, anything else → no release.

All three permissions are required. A reusable workflow can only *narrow* the token scope
its caller grants, never widen it, so `contents: write` alone is not enough — the release
publishes, then `@semantic-release/github` fails posting its "included in release" comments
with `Resource not accessible by integration`, and the job goes red after the tag already
exists. That is exactly how this repository's own `v1.0.0` shipped.

## Deployment repos

```yaml
name: Plan
on:
  pull_request:
    branches: [main]

permissions:
  id-token: write        # OIDC; never granted by default, must be explicit
  contents: read
  pull-requests: write   # the plan comment

jobs:
  plan:
    uses: Muhammad-Irfan324/platform-workflows/.github/workflows/terraform-plan.yaml@v1
    with:
      working-directory: "./deployments/service"
      aws-role-arn: ${{ vars.AWS_ROLE_ARN }}
      terraform-version: "~1.15.9"
      backend-config: "config/backend.hcl"
      var-file: "env/staging.tfvars"
      aws-region: "eu-west-1"         # optional, default "eu-west-1"
    secrets:
      AUTOMATION_PAT: ${{ secrets.AUTOMATION_PAT }}
```

Plan and apply are independent and trigger-agnostic — the caller decides when each runs.
The two patterns in use:

**Dispatch** — a person picks resource, environment and action:

```yaml
on:
  workflow_dispatch:
    inputs:
      action:
        type: choice
        options: ["Terraform Plan", "Terraform Apply"]

permissions:
  id-token: write        # OIDC; never granted by default, must be explicit
  contents: read
  pull-requests: write   # the plan comment

jobs:
  plan:
    if: inputs.action == 'Terraform Plan'
    uses: Muhammad-Irfan324/platform-workflows/.github/workflows/terraform-plan.yaml@v1
    # …
  apply:
    if: inputs.action == 'Terraform Apply'
    uses: Muhammad-Irfan324/platform-workflows/.github/workflows/terraform-apply.yaml@v1
    # …
```

**Merge-triggered** — apply follows the merge:

```yaml
name: Apply
on:
  push:
    branches: [main]

permissions:
  id-token: write        # OIDC; never granted by default, must be explicit
  contents: read

jobs:
  apply:
    uses: Muhammad-Irfan324/platform-workflows/.github/workflows/terraform-apply.yaml@v1
    with:
      working-directory: "./deployments/service"
      aws-role-arn: ${{ vars.AWS_ROLE_ARN }}
      terraform-version: "~1.15.9"
      backend-config: "config/backend.hcl"
      var-file: "env/production.tfvars"
      environment: "production"
    secrets:
      AUTOMATION_PAT: ${{ secrets.AUTOMATION_PAT }}
```

Apply writes a plan file and applies that file, so it never executes a diff recalculated
between the two steps. A failing plan stops the job before anything is applied, and the
plan it did apply is recorded in the run summary.

**Plan and apply handle collisions differently.** Apply queues per environment and
directory and never cancels — a half-finished apply has already changed real
infrastructure. Plan cancels superseded runs, because a second push to the same pull
request makes the running plan obsolete before anyone can read it.

**Terraform's version is set per repo, not here.** Terraform writes `terraform_version`
into remote state and refuses state written by a newer minor, so `terraform-version` is a
required input on plan and apply.

**`permissions:` belongs on the caller, not here.** A reusable workflow can only reduce
the token scope it is given, never raise it, and `id-token` is `none` under both default
settings — so OIDC fails unless the calling workflow grants `id-token: write` itself. The
blocks in the examples above are the minimum each workflow needs.

## Application repos

```yaml
name: Application CI
on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

permissions:
  contents: read
  pull-requests: write
  security-events: write

jobs:
  app_ci:
    name: "Application CI"
    uses: Muhammad-Irfan324/platform-workflows/.github/workflows/application-ci.yaml@v1
    with:
      node-version: "20"
      docker-context: "."
      registry-url: "registry.gitlab.com"
      image-name: "my-group/my-service"
      sonar-project-key: "my-service"
      push-image: ${{ github.ref == 'refs/heads/main' }}
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
      SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
      REGISTRY_USER: ${{ secrets.REGISTRY_USER }}       # GitLab Deploy Token name
      REGISTRY_TOKEN: ${{ secrets.REGISTRY_TOKEN }}     # GitLab Deploy Token value
```

The pipeline runs lint, unit tests, integration tests, SonarQube SAST, OWASP dependency
scanning, Gitleaks secret scanning, Docker build with Trivy container scanning (findings
uploaded as SARIF to the GitHub Security tab), and optionally pushes to GitLab Container
Registry on merge to main with SLSA provenance attestation and SBOM attached to the image.
All security gates run as PR checks — a failing scan blocks the merge.

## Central configuration

### [`versions.env`](versions.env)

Versions of the tools CI installs. Bump a line and it applies everywhere on the next run.

**Nothing bumps this file automatically.** Dependabot reads `uses:` refs and package
manifests; no ecosystem understands TFLint or terraform-docs, so these pins move only when
a person moves them. The file header lists the releases pages to check. The npm and pip
sides *are* watched — see `configs/package.json` and `configs/yamllint-requirements.txt`.

A `*_SHA256` line is the digest of the asset its `*_VERSION` names, checked before the
binary is extracted or run. Bump one without the other and the install step fails the
checksum — loudly, everywhere. That failure is the control working; refresh the digest from
the release's own checksums file rather than from the tarball you just downloaded.

| Value | Read by |
|---|---|
| `TFLINT_VERSION` | terraform-module-ci |
| `TFLINT_AWS_RULESET_VERSION` | terraform-module-ci, rendered into `.tflint.hcl` |
| `TERRAFORM_DOCS_VERSION` | terraform-module-ci |
| `TERRAFORM_DOCS_SHA256` | terraform-module-ci, verified before the binary runs |
| `NODE_VERSION` | release, terraform-release |
| `ACTIONLINT_VERSION` | ci |
| `ACTIONLINT_SHA256` | ci, verified before the binary runs |

### [`configs/`](configs/)

Most of these are copied into a module repo that does not provide its own, and a committed
file in the module repo always wins. `package.json` is the exception: nothing copies it,
and the release workflows read the pins straight out of it.

| File | Applied by | As |
|---|---|---|
| [`.tflint.hcl.tmpl`](configs/.tflint.hcl.tmpl) | terraform-module-ci | `.tflint.hcl`, with `TFLINT_AWS_RULESET_VERSION` substituted |
| [`.terraform-docs.yaml`](configs/.terraform-docs.yaml) | terraform-module-ci | copied verbatim |
| [`.releaserc.json`](configs/.releaserc.json) | terraform-release | copied verbatim |
| [`.releaserc.json`](configs/.releaserc.json) | release (this repo) | changelog and git plugins stripped at runtime |
| [`package.json`](configs/package.json) | both release workflows | pins read with `jq`, passed to `npm install` |
| [`yamllint-requirements.txt`](configs/yamllint-requirements.txt) | ci (this repo) | `pip install --require-hashes`; holds yamllint's version *and* digests |

`.releaserc.json` is one base config serving both. Module repos get it whole — changelog
written, committed and pushed. This repo renders it with the `changelog` and `git` plugins
removed, since it is versioned by tag only. Both keep the `exec` plugin, which writes
`.release-version` so the workflow can read the version back instead of scraping it out of
semantic-release's log output.

### `.checkov.yaml` — stays in the module repo

Per-module skip rules only. `skip-check` and `skip-path` are the only accepted keys; the
file is validated before Checkov runs, so a module repo cannot disable its own gate with
`soft-fail: true`.

## Conventions

- **SHA pinning** — third-party actions pinned to a commit SHA with a `# vX.Y.Z` comment,
  enforced by `ci.yaml`.
- **OIDC auth** — AWS access via role assumption, no long-lived credentials.
- **Naming** — `.yaml` extension, `kebab-case` inputs, `UPPER_SNAKE_CASE` secrets.
- **Permissions** — least-privilege, declared at job level.

## Contributing

A change here lands in every consuming repo the next time one runs, so two things matter
more than they would elsewhere.

**Your commit subject decides whether a release happens.** `fix:` and `feat:` cut one;
`chore:`, `docs:`, `ci:`, `refactor:`, `test:` and `style:` do not. Get this wrong in the
harmless direction and your change sits on `main` unreleased while every repo on `@v1`
keeps running the old version — the failure is silent, and nothing goes red.

**Breaking means breaking for the caller, not for you.** Renaming or removing an input,
making an optional input required, adding a required secret, or changing a default that
callers depend on all break someone's pipeline. Mark those `feat!:` or add a
`BREAKING CHANGE:` footer. The release guard in `ci.yaml` will then fail your PR until
`workflows-ref` names the major you are about to cut — that failure is the reminder,
not an obstacle.

Remember every commit in the PR is read, not just the title (see *Releasing this repo*).

Before opening a pull request:

- `actionlint` and `yamllint` — the same checks `ci.yaml` runs.
- Pin any new third-party action to a commit SHA with a `# vX.Y.Z` comment; the SHA-pin
  gate rejects anything else.
- Any new downloaded binary needs a `*_SHA256` in `versions.env`, verified before it runs.
- Say *why* in the file, not just what. The headers and inline comments here exist because
  the reasoning behind a workflow is the part that gets lost.

## Prerequisites

- `AUTOMATION_PAT` available to the consuming repo, for private module sources.
- OIDC provider configured in AWS, with IAM roles trusted per consuming repo.
- Workflow sharing enabled here: Settings → Actions → General → Access.
- Branch protection requiring these checks, or results are informational only. Best set as
  an org ruleset targeting `terraform-*` so it covers every module repo at once.
- **Application repos** — `SONAR_TOKEN` (project-scoped analysis token from SonarQube);
  `SONAR_HOST_URL` (SonarQube server URL, passed as a secret);
  `REGISTRY_USER` and `REGISTRY_TOKEN` (a GitLab Deploy Token scoped to
  `read_registry` + `write_registry`, not a personal password).
