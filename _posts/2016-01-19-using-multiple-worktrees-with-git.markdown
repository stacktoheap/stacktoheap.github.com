---
layout: post
title: "Using multiple worktrees with git"
excerpt: "Use multiple worktrees to make working with branches and bug fixes much easier with git"
reading_time: "4 mins"
date: 2016-01-19 12:07
comments: true
categories: [git]
tags: [git, worktree, productivity]
---

One of my recent observations is that when working with multiple branches at the same time, people clone the whole git repository again. Mostly, there are no issues with this, and this could simplify the workflow a lot.

But, when you use git hooks, new clones can be inconvenient. As is the standard practice, we checkin our hooks to the repo. These are then symlinked into `.git/hooks` via a bootstrap script that the user has to run after the initial clone. When cloning a repo again, the user has to remember to install the hooks in the new clone as well.

With or without hooks in the picture, the ideal, and  the most efficient, approach to working with multiple independent working copies is to make use of the `git worktree` feature. This was added in git 2.5, and is the focus of this article.

## Working with multiple branches

Say you are on a feature branch, and want to make a bug fix in `master`. You can create a whole new working copy based on master as follows:

{% highlight bash %}
git fetch
git worktree add -b bugfix-1234 ../bugfix origin/master
{% endhighlight %}

The above sets up a new worktree at `../bugfix` (assuming, and recommended that you are at the root of your original cloned repository). The `-b bugfix-1234` option creates a new branch, named `bugfix-1234`, based off master. If the `-b` option is not given, a new branch matching the directory of your worktree, `bugfix`, is created. You can now `cd ../bugfix` and work on this new worktree (and branch) as though it were a new clone of the repository. Your original clone and work is not interrupted

And best of all, your hooks are carried over as well!

## Long running tasks

Another use case for the worktree feature is when you have long running tasks (say, running build, unit tests, integration tests and functional test) that you want to run while continuing to work on the codebase. In that case, you can:

{% highlight bash %}
git worktree add --detach ../project-build HEAD
cd ../project-build
./run_build.sh
{% endhighlight %}

The above creates a worktree with detached branch off current `HEAD`. We can use this temporary worktree to run our long running task, and continue to make changes in our original worktree. Using `--detach` is ideal in this situation, as this is a temporary worktree, and we don't want to create a branch for this.

## Cleanup

As of this writing, there are no helpers to remove worktrees, so you can just `rm -rf ../bugfix && git worktree prune` when you are done with your new worktree. The `git worktree prune` is optional, and will remove metadata around your worktree from the repository immediately. If not used, the normal git garbage collection process will eventually clean it up.
