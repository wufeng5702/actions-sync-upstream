# actions-sync-upstream

A GitHub Action that keeps a fork in sync with its upstream repository: it rebases your branch on top of upstream and pushes with `--force-with-lease`.

- Auto-detects the upstream (the fork's parent) and your default branch via the GitHub API — most forks need no configuration.
- Fetches only the branch being synced; tags are never fetched.
- Scheduled runs are cheap: when the branch is already up to date nothing is pushed, and a rebase conflict fails the job instead of pushing a broken state.

## Usage

Create `.github/workflows/sync-fork.yml` in your fork:

```yaml
name: Sync fork with upstream

on:
  schedule:
    - cron: "0 8 * * *" # 08:00 UTC (16:00 UTC+8)
  workflow_dispatch:

permissions:
  contents: write # the action needs to push

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - uses: wufeng5702/actions-sync-upstream@main
```

## Inputs

All inputs are optional.

| Name            | Default | Description                                                                                                                                                              |
| --------------- | ------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `upstream`      | `""`    | Upstream repository as `owner/name` or a github.com URL. Empty = auto-detect the fork's parent (exits successfully if the repository is not a fork).                        |
| `default_branch` | `""`   | Branch to sync. Empty = the repository's default branch.                                                                                                                  |
| `fetch_depth`   | `"0"`   | Fetch depth for checkout and upstream fetch. `0` = full history, always safe for rebase. Smaller values are faster but the rebase fails if the merge base is deeper.       |
| `token`         | `""`    | Token used for API calls and pushing. Empty = the workflow's `GITHUB_TOKEN`.                                                                                              |

```yaml
      - uses: wufeng5702/actions-sync-upstream@main
        with:
          upstream: GopeedLab/gopeed # optional, auto-detected otherwise
          default_branch: main       # optional, auto-detected otherwise
          fetch_depth: "50"          # optional speed-up for frequently synced forks
          token: ${{ secrets.SYNC_TOKEN }} # optional, see notes below
```

## How it works

1. Resolve `upstream` (input, or the fork's parent from the API) and the branch (input, or the repository default branch). If the repository is not a fork and no `upstream` input is given, the action exits successfully.
2. Fetch that single branch from `origin` (your fork) and from `upstream`.
3. Reset the local branch to your fork's copy — or create it from upstream if the fork doesn't have it yet — then `git rebase upstream/<branch>`.
4. Push with `git push --force-with-lease`, which refuses to overwrite anything the action hasn't seen.

## Notes

- `contents: write` is required so the action can push.
- Pushing with `GITHUB_TOKEN` does not trigger other workflows in your fork. If you want CI to run on sync commits, pass a personal access token as `token`.
- A rebase conflict fails the job and nothing is pushed. Rebase manually, resolve the conflicts, push, then re-run the workflow.
- `fetch_depth: 0` (the default) fetches all branches of your fork during checkout; set a smaller value on very large repositories and the rebase will still work as long as the merge base is within the depth.
- Requires the `gh` CLI, which is preinstalled on GitHub-hosted runners.
- Pin the action to a tag (e.g. `@v1`) instead of `@main` once a release exists, if you want reproducible runs.
