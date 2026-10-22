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
first. Problems are shown as annotations on the lines they are about.

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

This repository is built from `apps/theme-deploy-action` in the Novinzo monorepo.
