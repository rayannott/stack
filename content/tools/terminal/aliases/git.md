---
title: "git"
date: 2026-03-03T13:28:00+01:00
description: "My Git aliases."
tags: ["Git", "Aliases"]
draft: false
params:
  neso:
    show_toc: true
---

I prefer to use git in my [terminal](tools/terminal) --- this way I can see the output of the commands I run, and thanks to the following aliases, it is blazingly fast.

<!--more-->

## Git Aliases
Git is a tool that I use every day, and I want to type as little as possible.
```sh
alias gsw="git switch"
alias gswc="git switch -c"
alias gcm="git commit -m"
alias gca="git add -A"
alias gcam="git add -A && git commit -m"
alias gp="git push"
alias gl="git log --graph --decorate --numstat"
alias gpl="git pull"
alias gb="git branch"
alias gfop="git fetch origin --prune"
alias gds="git diff HEAD~1..HEAD"
alias gs="git status"
alias gfix="git add -A && git commit --amend --no-edit"
alias gd="git diff"
```

I also use GitHub CLI (gh) for what makes sense for my day-to-day development. The only alias that I use is `gpr` to create a pull request and view it in the browser:

```sh
alias gpr="gh pr create --fill && gh pr view --web"
```


## Motivation & Workflow
In my workflows, I prefer the use a very lean git strategy. I never commit directly to the `main` branch. Instead, I create a new one, implement the changes, and then "squash and merge" it into the main via a pull request. This way, the main branch is always tracking the latest stable version of the codebase, and the closed PRs read like a changelog.

## Example Workflow
Let's create a new feature branch, make changes, and then create a pull request:
```sh
gs  # where are we now?
gswc feat/git-aliases  # create a new feature branch
# ...make changes...
gd  # let's view the changes
gcam "add new feature"  # this adds all changes to the staging area and commits them
# oops, I forgot to change this one small thing...
gfix  # appends the changes to the previous commit
gl  # this just looks nice
gp  # push to remote
gpr  # create a pull request
```


The pull request will be created and opened in the browser. We can review the changes and merge it into the main branch.

Now, let's go through the standard PR review process, and finally merge it into the main branch. (Don't forget to delete the feature branch after merging!)

The following concludes the feature branch workflow:
```sh
gws -  # switch back
gpl  # pull the latest changes
gfop  # remove local branches that are no longer on remote
```
