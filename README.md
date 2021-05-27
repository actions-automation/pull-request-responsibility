# pull-request-responsibility

`pull-request-responsibility` is a set of actions that work on pull requests to ensure they continue moving towards merge. Automatically request and assign reviewers, update assignees based on review status, and automatically merge when approval criteria are met.

## Supported actions

Each of these actions are discrete, and can be enabled separately if desired.

### `assign`: Automatically adjust PR assignees based on who is most responsible for pushing it forward

The heart and soul of the toolset: this action will automatically adjust assignees on a pull request such that the current assignees are the ones that must take action to move the pull request forward. Here's an example timeline of actions for a PR submitted by author X:

- PR from author X arrives. Any requested reviewers (let's say reviewers A, B, and C) are assigned.
- Reviewer A leaves a review requesting changes. Reviewers A, B, and C are unassigned, and author X is assigned to address A's review.
- X makes changes to the PR and re-requests review from reviewer A. Since there are no outstanding Changes Requested reviews, all requested reviewers that haven't yet approved (A, B, C) are put back on the PR.
- Reviewer B approves the PR, and is unassigned. Reviewers A and C are still assigned.
- Reviewer C approves the PR, and is unassigned. Reviewer A is still assigned.
- etc...

In effect, this turns Github's Assignee field from a vague "point person" to a clear indicator of responsibility.

### `request`: Automatically request reviewers on new pull requests

When a new PR arrives, this action will request a (configurable) number of reviewers, pulling from a mixture of:

- Github's suggested reviewers for that PR
- A team from the repository's parent organization (configurable)

Reviewer availability is considered: if the user marks themselves as having "limited availability" via Github's user status, they won't be automatically requested for review.

Inputs:

- `reviewers`: the team name to pull reviewers from in the parent repo's organization
- `num_to_request`: the number of reviewers to request on new PRs

You can see an example of how inputs should be specified in the Usage section below.

### `copy-labels-linked`: Copy any labels present on linked issues to PRs

When an issue is linked to a PR with a [keyword](https://docs.github.com/en/github/managing-your-work-on-github/linking-a-pull-request-to-an-issue#linking-a-pull-request-to-an-issue-using-a-keyword), either in the PR description or in PR commits, this action ensures that any labels present on any such linked issues are also present on the PR.

### `merge`: Automatically merge pull requests when they meet merge criteria

When a pull request meets a repo's configured criteria for [mergeability](https://docs.github.com/en/github/administering-a-repository/defining-the-mergeability-of-pull-requests) (which may include number of approvals, passing tests, as well as other criteria), this action will automatically merge them.

## Usage

In most cases, all you need to start using `actions-automation` is a single workflow!

Create a new workflow `yml` under `.github/workflows` with the following structure:

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
    name: pull-request-responsibility
    steps:
      - uses: actions-automation/pull-request-responsibility@main
        with:
          actions: "assign,copy-labels-linked,merge" # The actions to run.
          token: ${{ secrets.GITHUB_TOKEN }}
```

Note that you'll need to configure the action's inputs accordingly. If none are provided, only the `assign` action will run by default; the rest are opt-in via the `actions` input. Additionally, you'll need to provide a Github access token; the `GITHUB_TOKEN` should suffice for most cases.

Once this is in place, you're done! `pull-request-responsibility` should begin working on your repo
when certain events (ex. an issue/PR gets opened) happen.

### Enabling the `request` action

You may have noticed that the `request` action is not included in the workflow above. That's because it performs cross-repo API calls and needs some additional setup.

### Github authentication with a personal access token

The Actions-provided `GITHUB_TOKEN` secret is good for most of what `pull-request-responsibility` does, but it does not have permission to read or write data from outside the repository that spawned it. This poses a problem with the `request` action, which looks at things like user availability and organization-level teams.

In order for `request` to function correctly, we'll need to authenticate with Github using a **personal access token** instead of the `GITHUB_TOKEN`. These are user-generated and effectively allow scripts to act as a user through the Github API. You can manage your active tokens [here](https://github.com/settings/tokens).

Provision one with the `public_repo`, `read:org`, and `write:org` scopes enabled and make it available as a [shared secret](https://docs.github.com/en/free-pro-team@latest/actions/reference/encrypted-secrets)
to your repository, using the name `BOT_TOKEN`. (Organization-wide secrets also work.)

Github _strongly_
[recommends](https://docs.github.com/en/free-pro-team@latest/actions/learn-github-actions/security-hardening-for-github-actions#considering-cross-repository-access)
creating a separate account for PATs like this, and scoping access to that
account accordingly. It's recommended to heed their advice; PATs are dangerous if leaked, and can result in malicious actors abusing the account that created it.

If you're using any other `actions-automation` tools, chances are this step will be necessary as well. Once the token is available as a secret, however, it should be straightforward to use it in perpetuity.

### Enabling `request` in your workflow

With the repository secret available, replace the `token` input with your new PAT:

```yml
token: ${{ secrets.BOT_TOKEN }}
```

You'll also need to provide some additional inputs for the action to work (documented in its description):

```yml
reviewers: "reviews" # The team to pull reviewers from for `request`.
num_to_request: 3 # The number of reviewers that `request` should request on new PRs.
```

And, finally, enable the action in the `actions` input:

```yml
actions: "request,assign,copy-labels-linked,merge" # The actions to run.
```

Your completed step should look something like this:

```yml
- uses: actions-automation/pull-request-responsibility@main
  with:
    actions: "request,assign,copy-labels-linked,merge" # The actions to run.
    token: ${{ secrets.BOT_TOKEN }}
    reviewers: "reviews" # The team to pull reviewers from for `request`.
    num_to_request: 3 # The number of reviewers that `request` should request on new PRs.
```

And you should be done!

## FAQ

### Why do I need to listen for every single Actions event in my workflow?

`pull-request-responsibility` filters the actions it takes based on the event that occurs. For
example, it will request new reviewers when a new PR is opened, but it won't
attempt to do label management until that PR has been labeled.

The action doesn't _need_ every trigger to work properly -- in fact, it doesn't
use most of them at all. However, there's little downside to enabling all of
them, and doing so avoids the need to change workflows if new functionality using a new trigger is added.

## Credits

This action was originally developed for the Enarx project as part of [Enarxbot](https://github.com/enarx/bot), and is now available for any repo to use here.
