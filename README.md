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
concurrency:
  group: novinzo-theme
  cancel-in-progress: true
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - uses: emergente-labs/novinzo-theme-deploy@v1
        with:
          store: your-store
```

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
    if: github.event.pull_request.head.repo.full_name == github.repository
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - uses: emergente-labs/novinzo-theme-deploy@v1
        with:
          store: ${{ vars.NOVINZO_STORE }}
          mode: check
```

A store may have its theme checked 120 times an hour.

To check a theme on your computer, run `npx @novinzo/theme-cli check` in it.

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
  release adds one, stay as they are and are listed too.

Only releases tagged `v<major>.<minor>.<patch>` count; prerelease tags are skipped. When
Novinzo serves another major version than `theme.json` extends, the run fails: a new major
release's changes are taken in by hand.

It then moves `extends` to the release and sets the `version` and `summary` outputs; it
commits nothing itself. The starter's `starter-updates.yml` opens a pull request with them
the first day Novinzo serves a new release, looking once a day:

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
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v5
      - id: update
        uses: emergente-labs/novinzo-theme-deploy@v1
        with:
          mode: update
      - if: steps.update.outputs.version != ''
        env:
          GH_TOKEN: ${{ github.token }}
          VERSION: ${{ steps.update.outputs.version }}
          SUMMARY: ${{ steps.update.outputs.summary }}
        run: |
          branch="novinzo/starter-$VERSION"
          if git ls-remote --exit-code --heads origin "$branch" > /dev/null; then
            echo "A pull request for starter $VERSION is already open."
            exit 0
          fi
          git switch --create "$branch"
          git -c user.name="github-actions[bot]" \
            -c user.email="41898282+github-actions[bot]@users.noreply.github.com" \
            commit --all --message "Take in starter $VERSION"
          git push origin "$branch"
          gh pr create --head "$branch" --title "Take in starter $VERSION" --body "$SUMMARY"
```

GitHub lets a workflow open pull requests only once **Settings → Actions → General →
Allow GitHub Actions to create and approve pull requests** is on. A pull request a
workflow opens does not start other workflows, so close and reopen it to run the check.
GitHub pauses scheduled workflows in a public repository after 60 days without activity;
re-enable it from the **Actions** tab.

This repository is built from `apps/theme-deploy-action` in the Novinzo monorepo.
