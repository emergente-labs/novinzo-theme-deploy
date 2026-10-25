# Deploy a Novinzo theme

Deploys the theme in this repository to a Novinzo store on every push to the store's
linked branch. The store owner links the repository and branch on **Settings → Theme** in
the Novinzo dashboard; the deploy proves it runs for that repository and branch with
GitHub's OIDC token, so no secret is stored here.

```yaml
name: Deploy theme
on:
  push:
    branches: [main]
permissions:
  id-token: write
  contents: read
jobs:
  deploy:
    uses: emergente-labs/novinzo-theme-deploy/.github/workflows/deploy.yml@v1
    with:
      store: ${{ vars.NOVINZO_STORE }}
```

The workflow calls this repository's [reusable workflows](#reusable-workflows) rather than
listing the action's steps itself, so what it runs changes with each release of `v1` and
the theme repository never has to edit it. A job can still run the action directly
(`uses: emergente-labs/novinzo-theme-deploy@v1` after `actions/checkout`), with the same
inputs.

The action deploys to Novinzo's API. To deploy to another Novinzo environment, such as
staging, also set `api-url` to that environment's API.

The action runs the theme's `npm run build` when its `package.json` has one (the
starter builds its Tailwind CSS this way) and deploys the theme, which Novinzo checks
first. Problems are shown as annotations on the lines they are about, in the pull request's
diff too when the theme is in a subdirectory set as the `path` input. When `theme.json`
extends the starter, the action first writes the starter's Liquid to `.novinzo/base/` for
the build, so a Tailwind build that names it as a source keeps the classes of the pages the
theme inherits; a theme without a build does not need it.

The first run after linking deploys nothing: a repository name alone does not prove the
repository is the store owner's, so Novinzo shows the run (repository, who started it,
commit) on **Settings → Theme** for the owner to approve. Once they have, re-run the job;
from then on every push deploys. Only `push` and manually started runs deploy.

To roll back, revert the commit and push: Novinzo keeps only the live version.

## Checking pull requests

With `mode: check`, the action has Novinzo check the theme the way a deploy does and
deploys nothing, so a pull request shows its problems before it is merged. Any branch of
the store's approved repository may be checked; GitHub gives pull requests from forks no
OIDC token, so the job skips them.

```yaml
name: Check theme
on:
  pull_request:
permissions:
  id-token: write
  contents: read
jobs:
  check:
    uses: emergente-labs/novinzo-theme-deploy/.github/workflows/check.yml@v1
    with:
      store: ${{ vars.NOVINZO_STORE }}
```

A store may have its theme checked 120 times an hour.

To check a theme on your computer, run `npx @emergente-labs/theme-cli check` in it.

## Taking in starter releases

A theme whose `theme.json` extends the starter (`"extends": "starter@1.0.0"`) gets every
file it does not have from the starter Novinzo serves, so those follow the starter's
releases on their own. With `mode: update`, the action takes the newest starter release
of that major version into the copies the theme keeps, up to the release Novinzo serves
(it asks `api-url`), so the copies change when the inherited files do rather than when a
release is tagged:

- a copy the theme never changed becomes the release's, or is deleted when the release no
  longer has the file (it may have moved it);
- a copy it changed gets the release's changes merged in, as git merges;
- where the theme and the release changed the same lines, the file gets both, between git's
  conflict markers (`<<<<<<< this theme's <file>`, `=======`, `>>>>>>> starter <version>`).
  The pull request lists these files first: keep what you want of each side and delete the
  marker lines before merging. Until then, deploys and checks refuse the theme and annotate
  each marker, so none can reach the store;
- a binary file (such as an image) both changed cannot hold markers, so the theme's stays and
  the pull request lists it for you to take in the release's by hand;
- a copy the theme changed of a file the release no longer has, and a theme file where the
  release adds one, stay as they are and are listed too;
- of `package.json`, only `scripts` change, script by script as files do: one the theme
  never changed becomes the release's (or is added or deleted), one it changed stays and is
  listed. Dependencies stay the theme's, as its lockfile records them.

Only releases tagged `v<major>.<minor>.<patch>` count; prerelease tags are skipped. When
Novinzo serves another major version than `theme.json` extends, the run fails: a new major
release's changes are taken in by hand.

It then moves `extends` to the release and sets the `version` and `summary` outputs; it
commits nothing itself. The reusable `starter-updates.yml` opens a pull request with them
(closing any older one it supersedes) the first day Novinzo serves a new release, looking
once a day:

```yaml
name: Starter updates
on:
  schedule:
    - cron: "0 6 * * *"
  workflow_dispatch:
permissions:
  contents: write
  pull-requests: write
jobs:
  update:
    uses: emergente-labs/novinzo-theme-deploy/.github/workflows/starter-updates.yml@v1
```

GitHub lets a workflow open pull requests only once **Settings → Actions → General →
Allow GitHub Actions to create and approve pull requests** is on. A pull request a
workflow opens does not start other workflows, so close and reopen it to run the check.
GitHub pauses scheduled workflows in a public repository after 60 days without activity;
re-enable it from the **Actions** tab.

## Reusable workflows

`.github/workflows/` holds the jobs a theme repository runs, for its own workflows to call
with `uses: emergente-labs/novinzo-theme-deploy/.github/workflows/<file>@v1`:

| Workflow | Runs | Inputs |
| --- | --- | --- |
| `deploy.yml` | the action, deploying | `store` (required), `api-url`, `path`, `build` |
| `check.yml` | the action with `mode: check`, skipping pull requests from forks | `store` (required), `api-url`, `path`, `build` |
| `starter-updates.yml` | the action with `mode: update`, and opens its pull request | `api-url`, `path` |

Their inputs default as the action's do. The calling workflow grants the permissions (as in
the examples above), names the events, and passes the store, usually from the
`NOVINZO_STORE` repository variable. A workflow's own token may not change a workflow file,
so keeping the steps here is what lets a release change them in every theme repository.
None of them runs in the starter template itself.

This repository is built from `apps/theme-deploy-action` in the Novinzo monorepo.
