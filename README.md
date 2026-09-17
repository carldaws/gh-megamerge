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

## Walkthrough

**1. Start a project.**

```sh
gh megamerge init dark-mode
```

This creates `megamerge/dark-mode` from the repo's default branch (pass
`--base` for another), pushes it, and opens the draft target PR. That PR's
number is now stable for the project's life — share its link, or the deploy
URL its branch gets, with whoever is following along.

**2. Work exactly as you normally do.**

Branch off the base, commit, push, open a PR for each slice of the project.
Nothing about your branches or PRs changes; stacked branches are fine — open
the child's PR against its parent branch, as you would anyway.

**3. When a branch is ready to be seen, make it a source.**

```sh
git switch dark-mode-toggle   # or wherever the work lives
gh megamerge dark-mode add
```

`add` takes the current branch by default (or pass a branch name or PR
number). It checks the branch is pushed and has an open PR, applies any
configured labels, records it in the target PR's body, and syncs — the target
now contains it.

**4. Keep the target current as you iterate.**

After pushing more commits to any source:

```sh
gh megamerge dark-mode
```

The bare command is a sync: it recomposes `base + every open source` and
pushes only if the result actually differs. If two sources conflict, it
refuses and names the files, leaving the target as it was; fix the conflict on
a source branch and sync again. `gh megamerge dark-mode status` shows every
source, its PR's state, and whether the target is current;
`sync --dry-run` reports what a sync would do without pushing.

**5. Merge sources whenever they're approved.**

Merge each source PR exactly as you normally would, in any order. The next
sync notices, moves the source to the Merged section of the target PR's body,
and — because the same code now arrives via the base — the composed tree is
unchanged and nothing is pushed or redeployed. Anyone watching the target
never sees the handover.

**6. Close the project.**

```sh
gh megamerge dark-mode close
```

When the last source has merged, `close` closes the target PR and deletes its
branch. The PR remains as a record: every source the project shipped, listed
in its body. (`close` refuses while sources are still open, unless `--force`.)

Everywhere a target is named, use whichever handle is closest: the slug, the
target branch name, or the target PR's number. To pull a source back out of a
project without closing its PR, `gh megamerge dark-mode remove <branch|pr>`.

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
git config megamerge.branchPrefix "alice/mm"     # target branches become alice/mm/<slug> (default: megamerge)
git config megamerge.addLabels "skip-preview"    # applied to each source PR on add
git config megamerge.targetLabels "keep-data"    # applied to the target PR on init
```

The prefix is for repos with a branch-naming convention the default would
fight; set it before `init` and every command resolves slugs against it.

Both are comma-separated label lists, useful where PR labels steer a deploy
pipeline — for example, suppressing per-source preview deploys so a project
costs one deployment rather than one per PR, or marking the target's
deployment as long-lived.

## Conventions

- The target branch is `<prefix>/<slug>` (default prefix `megamerge`); never
  commit to it or merge its PR.
- A branch must be pushed, with an open PR, before it can become a source.
- Where someone is watching a deploy of the target, add sources when they are
  ready for review, so nobody sees work mid-construction.

## Tests

```sh
test/run
```

The suite drives the compose, filter, and stale-stack plumbing against a real
throwaway git repository.
