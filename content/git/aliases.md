---
title: "Aliases"
date: 2026-03-03T13:28:00+01:00
description: "My Git aliases."
tags: ["Git", "Aliases"]
draft: true
params:
  neso:
    show_toc: true
---

I prefer to use git in my [terminal](/terminal) --- this way I can see the output of the commands I run, and thanks to the following aliases, it is blazingly fast.

<!--more-->

## Git Aliases
```sh
alias gpr="gh pr create --fill && gh pr view --web"
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

## Motivation
