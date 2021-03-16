# pull-request-responsibility

`pull-request-responsibility` is a set of actions that work on pull requests to ensure they continue moving towards merge. Automatically request and assign reviewers, update assignees based on review status, and automatically merge when approval criteria are met.

## Supported actions

Each of these actions are discrete, and can be enabled separately if desired.

### `request`: Automatically request reviewers on new pull requests

When a new PR arrives, this action will request a (configurable) number of reviewers, pulling from a mixture of:

- Github's suggested reviewers for that PR
- A team from the repository's parent organization (configurable)

Reviewer availability is considered: if the user marks themselves as having "limited availability" via Github's user status, they won't be automatically requested for review.

Inputs:

- `reviewers`: the team name to pull reviewers from in the parent repo's organization
- `num_to_request`: the number of reviewers to request on new PRs

You can see an example of how inputs should be specified in the example workflow below.

### `assign`: Automatically adjust PR assignees based on who is most responsible for pushing it forward

The heart and soul of the toolset: this action will automatically adjust assignees on a pull request such that the current assignees are the ones that must take action to move the pull request forward. Here's an example timeline of actions for a PR submitted by author X:

- PR from author X arrives. Any requested reviewers (let's say reviewers A, B, and C) are assigned.
- Reviewer A leaves a review requesting changes. Reviewers A, B, and C are unassigned, and author X is assigned to address A's review.
- X makes changes to the PR and re-requests review from reviewer A. Since there are no outstanding Changes Requested reviews, all requested reviewers that haven't yet approved (A, B, C) are put back on the PR.
- Reviewer B approves the PR, and is unassigned. Reviewers A and C are still assigned.
- Reviewer C approves the PR, and is unassigned. Reviewer A is still assigned.
- etc...

In effect, this turns Github's Assignee field from a vague "point person" to a clear indicator of responsibility.

### `copy-labels-linked`: Copy any labels present on linked issues to PRs

When an issue is linked to a PR with a [keyword](https://docs.github.com/en/github/managing-your-work-on-github/linking-a-pull-request-to-an-issue#linking-a-pull-request-to-an-issue-using-a-keyword), either in the PR description or in PR commits, this action ensures that any labels present on any such linked issues are also present on the PR.

### `merge`: Automatically merge pull requests when they meet merge criteria

When a pull request meets a repo's configured criteria for [mergeability](https://docs.github.com/en/github/administering-a-repository/defining-the-mergeability-of-pull-requests) (which may include number of approvals, passing tests, as well as other criteria), this action will automatically merge them.

## Usage

Setup takes just a few straightforward steps.

### Authenticating with Github

Many `actions-automation` actions rely on a [personal access token](https://docs.github.com/en/free-pro-team@latest/github/authenticating-to-github/creating-a-personal-access-token)
to interact with Github and work their magic. If you're setting up a new
repository for use with `actions-automation` tools, you'll need to provision one with the `public_repo`, `read:org`, and `write:org`
scopes enabled and make it available as a [shared secret](https://docs.github.com/en/free-pro-team@latest/actions/reference/encrypted-secrets)
to your repository, using the name `BOT_TOKEN`. (Organization-wide secrets also work.)

Github _strongly_
[recommends](https://docs.github.com/en/free-pro-team@latest/actions/learn-github-actions/security-hardening-for-github-actions#considering-cross-repository-access)
creating a separate account for PATs like this, and scoping access to that
account accordingly.

### Enabling `pull-request-responsibility` on a repository

Once a token is available to your repo, the final step is to enable `pull-request-responsibility` via
Github Actions. To do so, create a new workflow under `.github/workflows` with
the following structure:

```yml
name: pull-request-responsibility

on:
  check_run:
  check_suite:
  create:
  delete:
  deployment:
  deployment_status:
  fork:
  gollum:
  issue_comment:
  issues:
  label:
  milestone:
  page_build:
  project:
  project_card:
  project_column:
  public:
  pull_request_target:
    types:
      - assigned
      - unassigned
      - labeled
      - unlabeled
      - opened
      - edited
      - closed
      - reopened
      - synchronize
      - ready_for_review
      - locked
      - unlocked
      - review_requested
      - review_request_removed
  push:
  registry_package:
  release:
  status:
  watch:
  schedule:
    - cron: '*/15 * * * *'
  workflow_dispatch:

jobs:
  pull-request-responsibility:
    runs-on: ubuntu-latest
    env:
      BOT_TOKEN: ${{ secrets.BOT_TOKEN }}
    name: pull-request-responsibility
    steps:
      - uses: actions-automation/pull-request-responsibility@main
        with:
          actions: "request,assign,copy-labels-linked,merge" # The actions to run.
          reviewers: "reviews" # The team to pull reviewers from for `request`.
          num_to_request: 3 # The number of reviewers that `request` should request on new PRs.

```

Note that you'll need to configure the action's inputs accordingly. If none are provided, only the `assign` action will run by default; the rest are opt-in. The other parameters are required for the `request` action, if enabled.

Once this is in place, you're done! `pull-request-responsibility` should begin working on your repo
when certain events (ex. an issue/PR gets opened) happen.

## FAQ

### Why do I need to listen for every single Actions event in my workflow?

`pull-request-responsibility` filters the actions it takes based on the event that occurs. For
example, it will request new reviewers when a new PR is opened, but it won't
attempt to do label management until that PR has been labeled.

The action doesn't _need_ every trigger to work properly -- in fact, it doesn't
use most of them at all. However, there's little downside to enabling all of
them, and doing so avoids the need to change workflows if new functionality using a new trigger is added.

### What happens if I don't set `BOT_TOKEN` properly?

If `pull-request-responsibility` can't find a valid token under `BOT_TOKEN`, it will fail gracefully
with a note reminding you to add one if you want to opt into automation. It
should _not_ send you a failure email about it.

This avoids forks of repositories that have `pull-request-responsibility` enabled from spamming their
authors with needless Actions failure emails about improperly-set credentials.

## Credits

This action was originally developed for the Enarx project as part of [Enarxbot](https://github.com/enarx/bot), and is now available for any repo to use here.
