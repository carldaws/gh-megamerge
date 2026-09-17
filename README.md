# gh-megamerge

One holding PR composed from many layer branches.

A cycle-long project normally forces a choice: one giant PR that product and QA
can follow at a single link, or a stack of small PRs and a link that keeps
moving. This tool removes the choice. Each slice of work is an ordinary small
PR off the default branch, reviewed and merged on its own. A **holding PR** —
a draft that is never merged — owns one branch that is always rebuilt as
`base + every open layer`, so anything that deploys per-branch (review apps,
preview deploys) shows the whole project at one stable URL from day one.

A layer merging is invisible at that URL: the same code arrives via the base
branch, the composed tree is unchanged, and nothing is pushed.

## Install

```sh
gh extension install carldaws/gh-megamerge
gh alias set mm megamerge   # optional: gh mm <slug> ...
```

Requires git ≥ 2.38 (`merge-tree --write-tree`).

## Usage

```sh
gh megamerge init command-centre        # creates megamerge/command-centre + draft holding PR
gh megamerge command-centre add         # current branch becomes a layer, then syncs
gh megamerge command-centre             # sync: rebuild holding branch from base + open layers
gh megamerge command-centre status      # layers, their PRs, holding-branch currency
gh megamerge command-centre close       # when the last layer has merged
```

The target is the slug, the `megamerge/<slug>` branch, or the holding PR
number — whichever is to hand. `add` and `remove` take a branch name or PR
number; `add` defaults to the current branch. `sync --dry-run` composes and
reports without pushing.

## How it works

- The layer list lives in a fenced block in the holding PR's body. There is no
  other state: any machine with `gh` auth can drive a project, and `sync`
  drops layers whose PRs have merged or closed.
- Composition uses `git merge-tree --write-tree` and `commit-tree` — nothing
  is checked out, so your working tree is never touched and no layer's code is
  executed. The result is one octopus commit whose parents are the base and
  every layer tip.
- `sync` pushes only when the composed tree differs from what the holding
  branch already holds, so a layer merging into the base does not redeploy.
- Only stack tips are merged: a layer contained in another layer, or already
  in the base, is skipped.
- Stacked layers are detected through their PR base ref. If a parent layer is
  reworked and a child still carries its old commits, `sync` refuses — that
  case merges cleanly and silently resurrects superseded code, so it must not
  compose. Rebase the child, then sync.
- `sync` refuses to overwrite the holding branch if its tip commit was not
  made by megamerge, so real work accidentally pushed there is never
  destroyed.

## Guard rails

Every command validates its target: the PR must be open, a draft, and carry
the megamerge block that only `init` writes. Passing the wrong PR number is an
error, not an incident.

## Per-repo config

```sh
git config megamerge.addLabels "No Review App"   # applied to each layer PR on add
git config megamerge.holdingLabels "Persist DB"  # applied to the holding PR on init
```

Both are comma-separated label lists, useful where labels steer your deploy
pipeline.

## Conventions

- The holding branch is `megamerge/<slug>`; never commit to it or merge its PR.
- Layers are added when they are ready for review and QA, so nobody watching
  the link sees work mid-construction.
- A branch must be pushed, with an open PR, before it can become a layer.

## Tests

```sh
test/run
```

The suite drives the compose, filter, and stale-stack plumbing against a real
throwaway git repository.
