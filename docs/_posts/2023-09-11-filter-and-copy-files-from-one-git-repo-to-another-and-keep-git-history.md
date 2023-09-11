---
layout: post
title:  "Filter and copy files from one git repo to another and keep git history"
date:   2023-09-11 10:47:12 -0800
categories: git
---

I find myself in situations regularly where I need to make a smaller copy of a Git repo, which has just a selection of the files from the source, but I need to preserve the commit history of those files. After plugging away at some source and remote branch stuff, I was able to find a few examples and then make a quick guide on how to do this relatively easily.

# Prerequisites

git
[git-filter-repo](https://github.com/newren/git-filter-repo)

# Basic Process

This really isn't too bad actually, it's about 3 steps.

* From your source repo, set up a filter in a new branch of all the files you want in the new repo
* In the target repo, add the source repo as a remote
* In the target repo, create a new branch, referencing the filtered source repo branch
* In the target repo, merge the branch and publish it

# Getting Started

Create a workdir to do your syncing to, keeping everything separate from your primary work dirs.

```bash
mkdir workdir
cd workdir
git clone git@github.com:alexholliz/source.git
git clone git@github.com:alexholliz/target.git
```

# Setting up the filtered source branch

```bash
# in the source git repo
git checkout -b filter-source
git filter-repo --path docker/configurables.yaml --path modules/k8s --refs refs/heads/filter-source --force
```

Keep in mind that you can filter both for files themselves, _and_ whole directories/subdirectories. The `--path` flags represent the content you want to end up in your filtered branch. Your branch is updated to remove everything that isn't in the paths you've specified. Of course if you break stuff with this, just checkout main again, delete your branch, and start over with new filters.

# Setting up the target branch with your filtered source branch

```bash
# in the target git repo
git checkout -b filter-target
git remote add repo-source ../source
git fetch repo-source
git branch branch-source remotes/repo-source/filter-source
git merge branch-source --allow-unrelated-histories
```

# Cleanup

Now that you've merged your branches into the target, you can publish that branch to the repo, and do your PR to get it into main, or whatever you like. History will come with it!