---
layout: post
title: Maintaining shadow branches for GitHub PRs
author: MaskRay
tags: [git,github]
---

I've created [pr-shadow](https://github.com/MaskRay/pr-shadow) with vibe coding, a tool that maintains a shadow branch for GitHub pull requests (PR) that never requires force-pushing.
This addresses pain points I described in [Reflections on LLVM's switch to GitHub pull requests#Patch evolution](/blog/2023-09-09-reflections-on-llvm-switch-to-github-pull-requests#patch-evolution).

<!-- more -->

## The problem

GitHub structures pull requests around branches, enforcing a branch-centric workflow.
There are multiple problems when you force-push a branch after a rebase:

* The UI displays "force-pushed the BB branch from X to Y". Clicking "compare" shows `git diff X..Y`, which includes unrelated upstream commits—not the actual patch difference. For a project like LLVM with 100+ commits daily, this makes the comparison essentially useless.
* Inline comments may become "outdated" or misplaced after force pushes.
* If your commit message references an issue or another PR, each force push creates a new link on the referenced page, cluttering it with duplicate mentions. (Adding backticks around the link text works around this, but it's not ideal.)

These difficulties lead to recommendations favoring [less flexible workflows](https://github.com/orgs/community/discussions/3478) that only append commits (including merge commits) and discourage rebases.
However, this means working with an outdated base, and switching between the main branch and PR branches causes numerous rebuilds-especially painful for large repositories like llvm-project.

```sh
git switch main; git pull; ninja -C build

# Switching to a feature branch with an outdated base requires numerous rebuilds.
git switch feature0
git merge origin/main  # I prefer `git rebase main` to remove merge commits, which clutter the history
ninja -C out/release

# Switching to another feature branch with an outdated base requires numerous rebuilds.
git switch feature1
git merge origin/main
ninja -C out/release

# Listing fixup commits ignoring upstream merges requires the clumsy --first-parent.
git log --first-parent
```

In a large repository, avoiding rebases isn't realistic—other commits frequently modify nearby lines, and rebasing is often the only way to discover that your patch needs adjustments due to interactions with other landed changes.

In 2022, GitHub introduced "Pull request title and description" for squash merging.
This means updating the final commit message requires editing via the web UI.
I prefer editing the local commit message and syncing the PR description from it.

## The solution

After updating my `main` branch, before switching to a feature branch, I always run

```sh
git rebase main feature
```

to minimize the number of modified files.
To avoid the force-push problems, I use pr-shadow to maintain a shadow PR branch (e.g., `pr/feature`) that only receives fast-forward commits (including merge commits).

I work freely on my local branch (rebase, amend, squash), then sync to the PR branch using `git commit-tree` to create a commit with the same tree but parented to the previous PR HEAD.

```
Local branch (feature)     PR branch (pr/feature)
        A                         A (init)
        |                         |
        B (amend)                 C1 "Fix bug"
        |                         |
        C (rebase)                C2 "Address review"
```

Reviewers see clean diffs between C1 and C2, even though the underlying commits were rewritten.

When `git merge-base origin/main feature` has moved since the previous push, `prs push` splits the update into two commits on `pr/feature`: a "Rebase onto main" merge commit whose tree is the old patch re-applied onto the new upstream (computed via `git merge-tree --write-tree`), followed by a regular commit carrying the functional change.
The first commit's first-parent diff is the pure upstream delta; the second's is the pure functional delta, so `git format-patch -1 <amend>` yields a clean patch reviewers can cherry-pick.
GitHub's PR-level file-diff is also clean, because `git merge-base origin/main pr/feature` resolves to the new upstream commit via the merge commit's second parent.

If the rebase itself required manual conflict resolution, the 3-way merge that reconstructs "old patch on new upstream" fails too. `prs push` then falls back to a single combined merge commit. Running `prs push` right after `git rebase` (before amending) keeps the split clean.

## Usage

```bash
# Initialize and create PR
git switch -c feature
edit && git commit -m feature

# Set `git merge-base origin/main feature` as the initial base. Push to pr/feature and open a GitHub PR.
prs init
# Same but create a draft PR. Repeated `init`s are rejected.
prs init --draft

# Work locally (rebase, amend, etc.)
git fetch origin main:main
git rebase main
git commit --amend

# Sync to PR
prs push "Rebase and fix bug"
# Force push if remote diverged due to messing with pr/feature directly.
prs push --force "Rewrite"

# Update PR title/body from local commit message.
prs desc

# Run gh commands on the PR.
prs gh view
prs gh checks
```

The tool supports both fork-based workflows (pushing to your fork) and same-repo workflows (for branches like `user/<name>/feature`).
It also works with GitHub Enterprise, auto-detecting the host from the repository URL.

## Related work

The name "prs" is a tribute to [spr](https://github.com/spacedentist/spr), which implements a similar shadow branch concept.
However, spr pushes user branches to the main repository rather than a personal fork.
While necessary for stacked pull requests, this approach is discouraged for single PRs as it clutters the upstream repository.
pr-shadow avoids this by pushing to your fork by default.

I owe an apology to folks who receive `users/MaskRay/feature` branches (if they use the default `fetch = +refs/heads/*:refs/remotes/origin/*` to receive user branches). I had been abusing spr for a long time after [LLVM's GitHub transition](/blog/2023-09-09-reflections-on-llvm-switch-to-github-pull-requests#patch-evolution) to avoid unnecessary rebuilds when switching between the main branch and PR branches.

Additionally, spr embeds a PR URL in commit messages (e.g., `Pull Request: https://github.com/llvm/llvm-project/pull/150816`), which can cause downstream forks to add unwanted backlinks to the original PR.

spr handles stacks by keeping every in-flight commit on local `main`, creating one PR per commit whose base is a synthetic branch (tree = `main` + all lower commits in the stack), and tracking commit identity across rebases via the `Pull Request:` trailer.
This depends on pushing base branches to the upstream repository — the property pr-shadow explicitly avoids — so stacks in pr-shadow are not first-class. For now I apply pr-shadow to the base patch and rebase-and-force-push dependent patches.
