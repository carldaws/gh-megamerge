# gh-megamerge

Keep one target branch merged from many source branches.

Each source is an ordinary branch with an ordinary PR, reviewed and merged on
its own schedule. The **target** is a branch the tool owns, continuously
rebuilt as `base + every open source`, with a never-merged draft PR that
carries the target's state and gives it a stable number. A source merging into
the base is invisible at the target: the same code arrives via the base, the
composed tree is unchanged, and nothing is pushed.

What that buys depends on the repo. Anywhere, the target is the integrated
state of your in-flight work — check it out to run everything together, point
CI at it, and hear about cross-branch conflicts when they appear rather than
at merge time. In a repo that deploys per branch (review apps, preview
deploys), the target PR gives product and QA one stable URL for a
project's whole life, however many small PRs it ships as.

## Install

```sh
gh extension install carldaws/gh-megamerge
gh alias set mm megamerge   # optional: gh mm <slug> ...
```

Requires git ≥ 2.38 (`merge-tree --write-tree`).

## Usage

```sh
gh megamerge init dark-mode        # creates megamerge/dark-mode + its draft target PR
gh megamerge dark-mode add         # current branch becomes a source, then syncs
gh megamerge dark-mode             # sync: rebuild the target from base + open sources
gh megamerge dark-mode status      # sources, their PRs, target currency
gh megamerge dark-mode close       # when the last source has merged
```

The target argument is the slug, the `megamerge/<slug>` branch, or the target
PR's number — whichever is to hand. `add` and `remove` take a branch name or
PR number; `add` defaults to the current branch. `sync --dry-run` composes and
reports without pushing.

## How it works

- The source list lives in a fenced block in the target PR's body. There is no
  other state: any machine with `gh` auth can drive a project. When a source's
  PR merges, `sync` moves it to a Merged section in the block, so the target PR
  records the project's full history; a PR closed without merging is dropped.
- Composition uses `git merge-tree --write-tree` and `commit-tree` — nothing
  is checked out, so your working tree is never touched and no source's code
  is executed. The result is one octopus commit whose parents are the base and
  every source tip.
- `sync` pushes only when the composed tree differs from what the target
  already holds, so a source merging into the base does not redeploy.
- Only stack tips are merged: a source contained in another source, or already
  in the base, is skipped.
- Stacked sources are detected through their PR base ref. If a parent is
  reworked and a child still carries its old commits, `sync` refuses — that
  case merges cleanly and silently resurrects superseded code, so it must not
  compose. Rebase the child, then sync.
- `sync` refuses to overwrite the target if its tip commit was not made by
  megamerge, so real work accidentally pushed there is never destroyed.

## Guard rails

Every command validates its target: the PR must be open, a draft, and carry
the megamerge block that only `init` writes. Passing the wrong PR number is an
error, not an incident.

## Per-repo config

```sh
git config megamerge.addLabels "skip-preview"    # applied to each source PR on add
git config megamerge.targetLabels "keep-data"    # applied to the target PR on init
```

Both are comma-separated label lists, useful where PR labels steer a deploy
pipeline — for example, suppressing per-source preview deploys so a project
costs one deployment rather than one per PR, or marking the target's
deployment as long-lived.

## Conventions

- The target branch is `megamerge/<slug>`; never commit to it or merge its PR.
- A branch must be pushed, with an open PR, before it can become a source.
- Where someone is watching a deploy of the target, add sources when they are
  ready for review, so nobody sees work mid-construction.

## Tests

```sh
test/run
```

The suite drives the compose, filter, and stale-stack plumbing against a real
throwaway git repository.
