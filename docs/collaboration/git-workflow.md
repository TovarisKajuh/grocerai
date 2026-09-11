# Git workflow

## Distributed, not shared

Both of us clone the repository. Git is distributed: your clone is a full copy of the history, and nothing you do locally is visible to the other person until you push. There is no shared working directory and no shared checkout. GitHub holds the copy we both agree on, and `main` on GitHub is the version of the project.

## main is protected

Nobody pushes to `main` directly and nobody force-pushes it. Every change reaches `main` through a pull request that the other person has approved. The protection rules that enforce this are listed in [setup.md](setup.md).

The one exception is described at the end of that page: while the project is a prototype with nothing deployed, pushing straight to `main` is acceptable. As soon as a broken `main` costs the other person time, that stops.

## One branch per piece of work

Start every piece of work on its own branch off an up-to-date `main`:

```sh
git switch main
git pull
git switch -c aljaz/route-screen-fixes
```

Branch names are `<person>/<short-description>`. The person prefix makes ownership obvious in the branch list and keeps the two of us from ever picking the same name. The description is a few words in kebab-case about the change, not a ticket number.

A branch holds one discrete change. If you notice something unrelated while working, put it on a different branch.

## Push, PR, review, squash, delete

When the work is ready, rebase onto `origin/main` first (see below), then push and open the pull request:

```sh
git push -u origin aljaz/route-screen-fixes
gh pr create --fill
```

The other person reviews it. They pull the branch, open `index.html` in a browser, read the diff, and either approve or ask for changes. Nobody merges their own PR without a review.

Merge with a squash so `main` gets one commit per PR, then delete the branch:

```sh
gh pr merge --squash --delete-branch
```

Because every merge is a squash, the individual commits on your branch do not need to be tidy. Commit as often as you like on the branch; the PR title and description are what survive.

After the merge, bring your clone up to date:

```sh
git switch main
git pull
```

## Rebase before the PR

Before you push a branch for review, rebase it onto the current `origin/main`:

```sh
git fetch origin
git rebase origin/main
```

If there are conflicts, resolve them in the branch, `git add` the files and `git rebase --continue`. The point is that conflicts are resolved by the person who wrote the conflicting change, on their branch, before review. `main` never sees a conflict resolution.

If the branch has already been pushed, the rebase rewrites its commits and you will need `git push --force-with-lease`. That is fine on your own PR branch and only there. See [pitfalls.md](pitfalls.md).

## Pull with rebase

Set this once:

```sh
git config --global pull.rebase true
```

With it, `git pull` replays your local commits on top of what it fetched instead of creating a merge commit. Combined with squash merges, this keeps `main` a straight line.

## Keep branches short-lived

A branch should be open for two days at most. Longer than that and it drifts from `main`, the rebase gets painful and the PR grows past what anyone can review properly. If a piece of work is going to take longer, split it into branches that can each be merged on their own, even if the first ones are not user-visible yet.

## Worktrees for parallel Claude sessions

Running two Claude Code sessions in one clone means both edit the same working directory and the same checked-out branch, and they will trample each other. Use a worktree instead: a second working directory attached to the same repository, on a different branch.

```sh
git worktree add ../GrocerAI-route-screen-fixes aljaz/route-screen-fixes
```

This creates the branch if it does not exist and checks it out in `../GrocerAI-route-screen-fixes`. Start the second Claude session in that directory. When the branch has been merged:

```sh
git worktree remove ../GrocerAI-route-screen-fixes
```

`git worktree list` shows what is currently attached.
