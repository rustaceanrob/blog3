---
title: "The Lazy Rebase"
date: "April 14 2024"
description: "The dirty secret git doesn't want you to know about!"

---

#### The Situation

You can try this strategy if:
- You have an open pull request
- The upstream `master` branch made new commits
- The upstream repository expects a `rebase` workflow

#### The Steps

Start by syncing up your local `master` branch with the upstream branch, typically `git pull upstream master` while checked out on your `master` branch. Then `git checkout <feature-branch>`. 

Assuming that the upstream branch was not working on a similar feature, most of the conflicts will be unrelated to your pull request. Occasionally the changes made on upstream will break some of your code, but I am of the opinion I would rather fix these situations on a case by case basis. To simply pull all the changes into your feature branch, you can use `git rebase -Xours master`. This will exclude,`X`, the changes on our feature branch, and just naively bring in everything from the remote tracking branch. I prefer getting my hands dirty within the editor to fix broken code that results from upstream changes. 
