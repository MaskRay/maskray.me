---
layout: post
title: Reflections on LLVM's switch to GitHub pull requests
author: MaskRay
tags: [llvm,github]
---

Since 2012, LLVM has relied on its self-hosted Phabricator instance on Google Cloud Platform for code review, but now it's making a transition to GitHub pull requests.
In this post, I'll share my perspective on this switch, highlighting GitHub offers significant benefits in some areas while having major drawbacks in the review process.

I may update this article as the process stabilizes further.

## Transition to GitHub pull requests

The move to GitHub pull requests has been a topic of discussion over the past few years.
Several lengthy threads on the subject have emerged:

<!-- more -->

* November 2019 [Enable Contributions Through Pull-request For LLVM](https://discourse.llvm.org/t/enable-contributions-through-pull-request-for-llvm/53627)
* January 2020 [Phabricator -> GitHub PRs?](https://discourse.llvm.org/t/phabricator-github-prs/54145)
* November 2020 [Notes from GitHub Pull Requests round table](https://discourse.llvm.org/t/notes-from-github-pull-requests-round-table/56952)
* August 2021 [Phabricator Creator Pulling the Plug](https://discourse.llvm.org/t/phabricator-creator-pulling-the-plug/58775)
* June 2022 [Update on GitHub pull requests](https://discourse.llvm.org/t/update-on-github-pull-requests/71540)
* November 2022 [Pull Request Progress Update](https://discourse.llvm.org/t/pull-request-progress-update/66790)

This transition could very well be the most contentious infrastructure change in LLVM's history.
If you have the patience to delve into the discussions within the mentioned threads, you'll come across [numerous remarks](https://discourse.llvm.org/t/enable-contributions-through-pull-request-for-llvm/53627/3) [critiquing](https://discourse.llvm.org/t/enable-contributions-through-pull-request-for-llvm/53627/5) [GitHub pull requests](https://discourse.llvm.org/t/enable-contributions-through-pull-request-for-llvm/53627/8) as being subpar.
I'd like to express my gratitude to Joerg Sonnenberger for sharing a post by Gregory Szorc titled [Problems with Pull Requests and How to Fix Them](https://gregoryszorc.com/blog/2020/01/07/problems-with-pull-requests-and-how-to-fix-them/), which made a great analysis comparing several code review tools.

Nevertheless, a decision has been made, and the targeted transition date was set for September 1, 2023.
The negotiation during the decision-making process could have been handled more strategically to [mitigate some confusion](https://discourse.llvm.org/t/update-on-github-pull-requests/71540/9).

On September 1, 2023 (or 2nd on some parts of the world), [the pull request lockdown was completely removed](https://github.com/llvm/llvm-project/pull/65161).

## Accessibility

In general, I believe that the majority of contributors consider GitHub to offer better accessibility.
However, I have heard quite a few opinions suggesting that GitHub's code review capability is significantly worse than that of Phabricator.
I will delve into this topic further later in this post.

Having contributed patches to more than 200 projects, many of which are one-off and even trivial, I genuinely appreciate it when a project uses GitHub pull requests. This is because I am already familiar with the system.
On the other hand, if a project relies on a self-hosted code review website, I might find it less convenient as I'm not particularly keen on registering a username on a website I may never visit again.
Even worse, I might need to invest time in getting acquainted with the system if I haven't encountered similar instances before.

The same argument applies to LLVM's self-hosted Phabricator instance.
Many contributors have not used Phabricator before and would consider both the website and the command-line tool (<https://github.com/phacility/arcanist>), to be challenging to use.
In general, I believe that GitHub is more contributor-friendly but perhaps not as reviewer-friendly.
Within the LLVM community, reviewer resources are extremely limited. Consequently, being reviewer-friendly is likely of critical importance.

## Automation

GitHub provides GitHub Apps and GitHub Actions to extend its functionality.
With these, we can automate pull request labelling, testing, code analysis, code coverage, and potentially even a [merge queue](https://github.blog/2023-07-12-github-merge-queue-is-generally-available/) in the future.
Some fantastic tooling is available for free and widely used by other open source projects, making it easy to emulate how other projects have set up automation.

Phabricator can also handle automation, but there are far fewer resources available for it.
LLVM's self-hosted Phabricator instance, for instance, relies on <https://github.com/google/llvm-premerge-checks> and Buildkite.

## Patch subscription

The llvm-project repository is vast. With a code frequency of 100+ commits every day, it's practically impossible for anyone to monitor every new commit. Nonetheless, many people wish to stay informed about changes to specific components, making patch subscription essential.

One way to achieve this is through mailing lists, such as [llvm-commits](https://lists.llvm.org/pipermail/llvm-commits/).
This list contains emails about new pull requests, edits, GitHub actions, labelling, resolved issues, and more, making it quite noisy.

The other method is to utilize the code review tool, formerly Phabricator and now GitHub.
With Phabricator, users can set up fairly complex subscription rules known as Herald.
When a patch title, description, affected files, or the acting user matches certain criteria, you can take actions like adding yourself as a reviewer/subscriber or sending a one-off email.

GitHub, however, is less flexible in this regard.
Individual users can choose to [watch all pull requests](https://docs.github.com/en/account-and-profile/managing-subscriptions-and-notifications-on-github/managing-subscriptions-for-activity-on-github/managing-your-subscriptions), but they can't do so selectively.
Interestingly, users can [watch issues with a specific label](https://discourse.llvm.org/t/label-based-notification/60517) but not pull requests with a specific label.

To enable component-based subscription, the llvm organization on GitHub has created multiple `pr-subscribers-*` teams, which users can freely join them.
(
[Note](https://discourse.llvm.org/t/update-on-github-pull-requests/71540/87):
A maintainer is supposed to accept every join request. Unfortunately, GitHub only displays the "pending request" information on the `https://github.com/orgs/llvm/teams/pr-subscribers-*` page and not on any other page. It's not reasonable to expect a maintainer to routinely check the `https://github.com/orgs/llvm/teams/pr-subscribers-*` pages for pending join requests.
So if they miss the email notification that says `would like to join "LLVM"`, the request may remain pending indefinitely.
)

Then we use [label-based subscription](https://discourse.llvm.org/t/changes-to-pull-request-subscription-system/73296/14).
A GitHub action is set up to [label every pull request](https://github.com/llvm/llvm-project/pull/65308), and these labels are used to [notify `pr-subscribers-*` teams](https://github.com/llvm/llvm-project/pull/64913).
For example, a pull request affecting `clang/lib/Driver/XX` will receive the labels `clang` and `clang:driver`, and the `pr-subscribers-clang` and `pr-subscribers-clang:driver` teams will be notified.

On <https://github.com/notifications>, the "Team mentioned" tab lists these patches.

### `.github/CODEOWNERS`

Previously, [we added `pr-subscribers-*` teams to `.github/CODEOWNERS`](https://discourse.llvm.org/t/input-needed-teams-for-pull-request-subscriptions/73116).
Due to GitHub's CODEOWNERS mechanism, the `pr-subscribers-clang` team would be added as a reviewer when a pull request affecting `clang/xx` was created.
`pr-subscribers-clang` members would receive an email notification about the pull request with a full diff.

However, a complication arose when a member of the `pr-subscribers-*` team approved a change. It resulted in a message saying `$user approved these changes on behalf of llvm/pr-subscribers-xx`, which could be misleading if the user did not wish to assume such authority.
In addition, the team was automatically removoed as a team reviewer, adding to the confusion.
This use case wasn't in line with GitHub's intended functionality, and there was a risk that GitHub's future changes might disrupt our workflow.

### Email filtering

Filtering is another crucial aspect of managing notifications.
GitHub supports [numerous notification reasons](https://docs.github.com/en/account-and-profile/managing-subscriptions-and-notifications-on-github/setting-up-notifications/configuring-notifications#filtering-email-notifications).

I have joined several `pr-subscribers-*` teams that I am interested in as replacements for my Herald rules on Phabricator.
I am still in the process to make a curated list of Gmail filters.
Here are some filters I currently use:

* Apply label "Mention": `from:(notifications@github.com) to:(mention@noreply.github.com)`
* Apply label "Team mention": `from:(notifications@github.com) to:(team_mention@noreply.github.com)`
* Apply label "Review request": `to:(review_requested@noreply.github.com)`
* Mark as read: `to:(push@noreply.github.com)`
* Skip Inbox, Apply label "llvm/issues": `from:(notifications@github.com) subject:(llvm-project "Issue #")`
* Skip Inbox: `from:(notifications@github.com) (-cc:(review_requested@noreply.github.com OR mention@noreply.github.com OR team_mention@noreply.github.com) "this pull request" KEYWORDS_I_WANT_TO_IGNORE)`

However, it's worth noting that these notifications also appear on <https://github.com/notifications>.
I haven't thought hard about the issue of receiving double notifications.
Perhaps <https://github.com/notifications> notifications can be used as a reminders and mute them in batch.

Another option is to switch to a pull-based workflow if LLVM provides a [public-inbox](https://public-inbox.org/INSTALL.html) instance.
<https://github.com/pulls> provides a dashboard where one can list review requests.

## Code review

Code review is the top reason we pick a code review tool.
Let's assess how GitHub pull requests fare in addressing this challenge.

### Patch evolution

In Phabricator, the body a differential (Phabricator's term for a patch) contains is a patch file.
The patch file is based on a specific commit, but Phabricator is not required to know the base commit.
A stable identifier, `Differential Revision: https://reviews.llvm.org/Dxxxxx`, in the commit message connects a patch file to a differential.
When you amend a patch, Phabricator recognizes that the differential has evolved from patch X to patch Y.
The user interface allows for comparisons between any two revisions associated with a differential. Additionally, review comments are confidently associated with the source line.

On the other hand, GitHub structures the concept of pull requests around branches and enforces a branch-centric workflow. A pull request centers on the difference (commits) between the base branch and the feature branch.
GitHub does not employ a stable identifier for commit tracking. If commits are rebased, reordered, or combined, GitHub can easily become confused.

When you force-push a branch after a rebase, the user interface [displays a line](https://github.blog/changelog/2018-11-15-force-push-timeline-event/) such as "force-pushed the BB branch from X to Y".
Clicking the "compare" button in GitHub presents something like `git diff X..Y`, which includes unrelated commits. Ideally, GitHub would show the difference between the two patch files, as Phabricator does, but it only displays the difference between the two head commits.
These unrelated in-between commits might be acceptable for projects with lower commit frequency but can be challenging for a project with a code frequency of 100+ commits every day.

The fidelity of preserving inline comments after a force push has always been a weakness. The comments may be presented as "outdated". In the past, there was a notorious "lost inline comment" problem.
Nowadays, the situation has improved, but some users still report that inline comments may occasionally become misplaced.

Due to the difficulties in comparing revisions and the lack of confidence in preserving inline comments, some recommendations suggest adopting [less flexible and less preferred workflows](https://github.com/orgs/community/discussions/3478), which involve only appending new commits and discouraging rebases.
This approach sometimes results in a cluttered commit history, with commit messages like "fix typo," "add test," and "fix ci".

In a large repository, avoiding pulling upstream commits may not be realistic due to other commits frequently modifying nearby lines.
Some people use a remote branch to save their work. Having to worry about whether a rebase could cause spam makes the branch more difficult to use.
When working with both the latest main branch and the pull request branch, switching between branches results in numerous rebuilds.
Rebasing follow-up commits could lead to merge conflicts and more pain.
In addition, a popular convention among many LLVM contributors is to commit tests before landing the functional change, which also mandates force-push.
(The GitHub UI will show a merge conflict.)
Sometimes, only through rebasing can one notice that the patch needs adjustments to more code or tests due to its interaction with another landed patch:

* Another landed patch added some tests that would be changed by the current patch.
* Another landed patch added a new use of the function that is renamed by the current patch.

There are two workflows you can do.
One is the merge-based workflow recommended by <https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/working-with-forks/syncing-a-fork>:

```sh
git switch pr1
git merge origin/main
git push $username pr1
```

You can invoke `git log --first-parent` to list the fixup commits.

Personally I prefer the rebase-based alternative: perform a rebase and force push, then make a functional change and either append a new commit or amend the old commit.

When the functional change is made, leave a comment so that reviewers can locate the useful "compare" button and disregard the "compare" button associated with the rebase push.

### Commit message

GitHub's repository setting allows three options for pull requests: "Allow merge commits", "Allow squash merging", and "Allow rebase merging".

The default effect is quite impactful.

* Many projects utilize merge commits when a linear history is preferred, even for just a single commit.
* When "Squash and merge" is selected, the default commit message becomes a concatenation of all commits, often resulting in a message filled with "fix typo," "add test," and "fix ci". In addition, the committer changes to `GitHub <noreply@github.com>`.

In 2022, GitHub finally [introduced an option called "Pull request title and description"](https://github.blog/changelog/2022-08-23-new-options-for-controlling-the-default-commit-message-when-merging-a-pull-request/) for the "Allow squash merging" option.
This new option mitigates some problems.

If you land a patch in a manual way, it is easy for the pull request to end up the "Closed" status (red), even if the commit has the `#xxxxx` identifier.
I am not sure whether this status might discourage people from manually landing patches, which I believe should be fully supported and not discouraged.
[[Feature Request] Mark PR as Merged from Pull Request API](https://github.com/orgs/community/discussions/12437)

### Patch series

I am not well-versed in reviewing patch series on GitHub, but this is a widely acknowledged pain point.
Numerous projects are exploring ways to alleviate this issue.

In Phabricator, since a differential consists of a patch file and doesn't have a fixed base branch, we can freely reorder two differentials.

The issue [Write a guide for doing "Stacked Reviews" with GitHub Pull Requests](https://github.com/llvm/llvm-project/issues/56636) tracks the documentation requirement.

User branches to the `llvm-project` repository are [allowed](https://discourse.llvm.org/t/update-on-github-pull-requests/71540/146) since 2023-11-01 to enable stacked pull requests.
After a month it was [reported](https://discourse.llvm.org/t/user-created-branches-in-the-monorepo-are-often-misused/75544) that some user branches were misused.

### Miscellaneous

Pros

* Fetching a pull request is more straightforward. `git fetch origin pull/xx/head:local_branch`. Phabricator allows `arc patch Dxxxxx` and the less-known method: `curl -L 'https://reviews.llvm.org/Dxxxxx?download=1' | patch -p1`
* `@mention` applies to every user on GitHub, which is convenient when you need to seek for a user's opinions. You cannot expect every user to have an account on a self-hosted instance.
* `gh` is more powerful than `arcanist`.
* There is a per-issue/PR "unsubscribe" button.

Cons

* Fetching a pull request at an old revision can be challenging.
* The narrow display can be somewhat inconvenient.
* Viewing diff and conversations requires switching between tabs.
* Collapsed comments are challenging to locate and often require multiple clicks, [makeing it difficult for reviewers to confirm whether a comment has been resolved](https://discourse.llvm.org/t/rfc-github-pr-resolve-conversation-button/73178).
* One cannot [comment on a source line a few lines out of the affected lines](https://github.com/orgs/community/discussions/4452).
* The inability to search for lines in nearby context when they are not displayed.
* A contributor without write access cannot add reviewers for a pull request.
* A merged pull request [cannot be reopened](https://github.com/orgs/community/discussions/37189). Let's say the patch has been reverted. In contrast, a closed pull request can be reopened.
* The commit message template does not contain an URI pointing to the pull request. [Feature request](https://github.com/orgs/community/discussions/5955)

Note: the button below the "Files changed" tab allows switching between unified and split diff views.
Users who are used to Phabricator's side-by-side diff view may want to adjust this setting.

---

I've noticed that my review productivity has decreased, a sentiment shared by many others.
It's disheartening that [alternative code review](https://discourse.llvm.org/t/phabricator-github-prs/54145/117) solutions haven't been thoroughly considered.
However, we must move forward and adapt to the new workflow while striving to mitigate the loss in productivity.

I hope that future iterations of GitHub will incorporate some ideas from [Pull Request feature requests for GitHub #56635](https://github.com/llvm/llvm-project/issues/56635).

I've voiced numerous concerns regarding GitHub pull requests, and for that, I apologize.
It's essential to acknowledge that GitHub contributes significantly to open source projects in many positive ways.
My intention in sharing these concerns is to express a genuine hope that GitHub pull requests can be enhanced to better support large projects.

I would also like to express my gratitude to [Phorge](https://phorge.it/), a community-driven fork of Phabricator, for their dedication and contributions, even as LLVM decided to migrate to the free but proprietary solution provided by GitHub.
Phorge's commitment to providing alternatives and nurturing an open-source community for organizations that favor self-hosted solutions is truly commendable.

The thread "How’s it going with pull requests?" has some nice analysis.

* <https://discourse.llvm.org/t/hows-it-going-with-pull-requests/73467/2>

## My workflow

I use <https://getcord.github.io/spr/> to [manage pull requests authored by me](https://discourse.llvm.org/t/using-getcord-spr/76097).
```
% cargo install spr
% spr --version
spr 1.3.4
% git remote -v
maskray git@github.com:MaskRay/llvm-project.git (fetch)
maskray git@github.com:MaskRay/llvm-project.git (push)
origin  git@github.com:llvm/llvm-project.git (fetch)
origin  git@github.com:llvm/llvm-project.git (push)
```

My `~/.gitconfig`:
```
[spr]
        githubAuthToken = ghp_xxxxxxxxxxx
        githubRemoteName = origin
        githubRepository = llvm/llvm-project
        githubMasterBranch = main
        branchPrefix = users/MaskRay/spr/
        requireTestPlan = false
```

After a rebase and an amend, `spr diff` will create a merge commit. GitHub's UI gives a `You are viewing a condensed version of this merge commit` message.

To review an incoming pull request, I run `gh pr checkout -b pr$id $id`.

I have enabled a browser extension [Refined GitHub](https://github.com/refined-github/refined-github) and adopted [octo.nvim](https://github.com/pwntester/octo.nvim).

`Alt + Click` on a resolved inline comment expands all resolved inline comments.
Another `Alt + Click` will hide all inline comments.
Note that some inline comments may become out-of-line, possibly due to a large local change or a force push.

I primarily look at the "Files changed" tab. I use "Conversation" for reading standalone comments, but not inline comments.
I want all resolvable comments to be expanded by default, so I have installed this userscript:
```javascript
// ==UserScript==
// @name         GitHub pull: expand all resolvable comments by default
// @version      2024-01-21
// @author       You
// @match        https://github.com/*/files
// @icon         https://www.google.com/s2/favicons?sz=64&domain=github.com
// @grant        none
// ==/UserScript==

document.querySelectorAll('.js-resolvable-timeline-thread-container').forEach(e => { e.setAttribute('open', '') })
```

## Future of reviews.llvm.org

On September 5, 2023, I added a red banner to reviews.llvm.org to discourage new patches.

Transitioning existing differentials to GitHub pull requests could potentially cause disruption.
[Resurrecting old patches is a practice that people regularly engage in](https://discourse.llvm.org/t/update-on-github-pull-requests/71540/73).
As per the current schedule, new differentials will be disallowed on October 1, 2023.

It is anticipated that at some point next year [reviews.llvm.org will become a read-only website](https://discourse.llvm.org/t/reviews-llvm-org-read-only-mode/73289).
To preserve `/Dxxxxx` pages (referenced by many commits), we can utilize [phab-archive from Mercurial](https://foss.heptapod.net/mercurial/phab-archive/-/tree/branch/default/archive).

As activities on Phabricator wind down, maintenance should become more lightweight.

In the past two weeks, there have been different IP addresses crawling `/source/llvm-github/` pages.
It looks very like a botnet as adjacent files are visited by IP addresses from very different autonomous systems.
Such a visit will cause Phabricator to spawn a process like `git log --skip=0 -n 30 --pretty=format:%H:%P 988a16af929ece9453622ea256911cdfdf079d47 -- llvm/lib/Demangle/ItaniumDemangle.cpp` that takes a few seconds (llvm-project is huge).
I have redirected some pages to <https://github.com/llvm/llvm-project/>.

On September 8, 2023, I resized the database disk to 850GB to [fix a full disk issue](https://discourse.llvm.org/t/phabricator-is-very-slow/73132/11).
Hopefully, we won't need to resize the disk again!

On September 18, 2023, I noticed an aggressive spider that made the website slow. I blocked the user agents for some aggresstive spiders.

On October 19, 2023, we received an attack with malicious using fake but strange user agents.
I blocked two related IP ranges and updated `/etc/php/fpm/pool.d/www.conf` (seemed un-configured before) to reserve more processes.

On November 23, 2023, SendGrid used by the website [stopped sending email notifications](https://discourse.llvm.org/t/no-emails-from-phabricator-since-nov-15/75455).
This was related to unidentified spammers.

I am adapting Mercurial's phab-archive project to mirror `reviews.llvm.org/Dxxxx` pages.
I just need to make a few adjustments (python,js,css,html).

The "History" button functions properly, although there isn't a "Compare" button.
