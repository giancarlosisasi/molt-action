# molt-action

The composite GitHub Action that runs [molt](https://github.com/giancarlosisasi/molt)'s release
loop.

molt is changeset-driven versioning and changelogs for Python monorepos. This Action installs the
`molt-release` CLI with [uv](https://docs.astral.sh/uv/) and runs it on every push to your base
branch. It keeps a "Version Packages" pull request in sync with your pending changesets, and
publishes to PyPI over OIDC when that pull request merges.

Full documentation: [CI: GitHub Action](https://molt.gio-labs.com/guides/ci-github-action).

## Usage

```yaml
name: Release

on:
  push:
    branches: [main]

# No ambient permissions; the job grants only what it needs.
permissions: {}

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: write        # push tags, push the release branch, create Releases
      pull-requests: write   # open and update the "Version Packages" PR
      id-token: write        # OIDC token for PyPI Trusted Publishing
    steps:
      - uses: actions/checkout@v5
        with:
          fetch-depth: 0              # molt reads history; a shallow clone gives it none
          persist-credentials: false  # the Action brings its own push credential

      - uses: giancarlosisasi/molt-action@v1
        with:
          molt-version: "0.1.2"   # pin molt; keeps the Action and the tool in lockstep
          publish: molt publish   # the command to run in the publish phase
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

That is the whole release automation, for a single package or a monorepo. There is no separate
`astral-sh/setup-uv` step: the Action installs uv itself, from a commit-pinned copy it owns.

## The two phases

Every push to your base branch runs one phase, and molt picks which:

- **Version phase**, when changesets are pending. The Action opens or updates a release pull
  request titled "Version Packages", carrying bumped manifests, rewritten changelogs and an updated
  lockfile. The same pull request is reused as more changesets merge, so its number and its review
  history survive.
- **Publish phase**, when no changesets are pending. That is the state right after the release pull
  request merges. The Action builds and publishes the changed packages, tags each one, and creates
  a GitHub Release per package with that package's changelog entry as the body.

Merging the release pull request is the release. There is no tag to push by hand.

## Inputs

Every input is optional. Supplying only a token runs the version phase with the defaults below.

| Input | Default | What it does |
|---|---|---|
| `molt-version` | latest release | The exact `molt-release` version to install, for example `"0.1.2"`. Empty installs the latest release. |
| `publish` | *(empty)* | The command that publishes the release, for example `molt publish`. Leave it empty to run the version phase only. Supplying it is also what makes the publish phase reachable at all. |
| `version-command` | *(empty)* | The command that versions the release, for example `molt version --snapshot canary`. Empty runs `molt version`. |
| `title` | `Version Packages` | The title of the release pull request. |
| `commit` | `Version Packages` | The commit message molt writes on the release branch. Separate from the title. |
| `create-releases` | `true` | Create one GitHub Release per published package. `true` or `false`; anything else fails the run rather than being read as `false`. |
| `base-branch` | the branch the workflow runs on | The branch being released. The release branch is `changeset-release/<base-branch>`. |
| `commit-mode` | `git-cli` | How the version commit reaches the remote. `git-cli` commits and force-pushes from the runner; `api` sends the changes to GitHub, which authors and signs the commit. Anything else fails the run. |
| `github-token` | `${{ github.token }}` | The token molt uses to open the pull request and create releases. A `GITHUB_TOKEN` already in the step's environment wins over this input. |

## Outputs

| Output | Value |
|---|---|
| `published` | `"true"` when at least one package was published this run. |
| `published_packages` | A JSON array of `{ "name", "version" }` objects for what went out. `[]` when nothing was published. |
| `has_changesets` | `"true"` when pending changesets exist. |
| `pull_request_number` | The number of the created or updated release pull request. Empty when the run opened none, which is every publish run. |

`published` says at least one package reached the index. It is as true after a half-publish as
after a clean one, so a workflow that treats it as "the release worked" must also check the step's
own outcome. A failing run still writes its outputs, so guard a reader with `if: always()`.

## Versioning

`v1` is a moving reference. It tracks the newest 1.x action and moves when `action.yml` changes on
`main`. Pin a commit in production, `uses: giancarlosisasi/molt-action@<sha>`, to decide for
yourself when the action changes. Dependabot updates a commit pin; it cannot tell you a moving tag
moved.

The Action's version and the CLI's version are separate things. `molt-version` pins the
`molt-release` distribution the Action installs from PyPI, and it does not read the Action's own
ref: turning a ref like `v1` into a PyPI specifier is guesswork, and guessing a version is the
opposite of pinning one.

## Repository settings

**Settings → Actions → General → Workflow permissions → Allow GitHub Actions to create and approve
pull requests.** This is off by default on personal-account repositories, and without it the
version phase fails with `GitHub Actions is not permitted to create or approve pull requests`.
Granting `pull-requests: write` in the workflow does not substitute for it.

Three further settings shape what the loop can do: branch protection on `changeset-release/*`, a
signed-commits rule, and whether checks start on the release pull request. See
[Repository settings](https://molt.gio-labs.com/forges/github#repository-settings).

## Conformance test

`action.yml` and the `molt-action` entry point are one surface split across two repositories, so an
input renamed on one side and not the other would be silent until a user's runner hit it.
`tests/test_action_yml.py` closes that: it reads `action.yml` and resolves every declared input
against the options an **installed** `molt-release` really declares, along with the four output
names, the composite `runs.using`, and the commit-SHA pin on every `uses:`.

```bash
uv run --no-project --with molt-release --with pytest pytest
```

CI runs it on every push and pull request, weekly on a schedule, and again before `v1` moves. The
weekly run is the one that matters: a release of `molt-release` in the other repository can break
the pairing without a single commit landing here.

## License

MIT. See [LICENSE](./LICENSE).
