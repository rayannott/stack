---
title: "aliases"
date: 2026-03-03T13:28:00+01:00
description: "Shell aliases I use to speed up my terminal workflow."
tags: ["Git", "Aliases", "Python", "Clipboard"]
draft: false
params:
  neso:
    show_toc: true
---

I prefer to use git in my [terminal](tools/terminal) --- this way I can see the output of the commands I run, and thanks to aliases, it is blazingly fast. Type less, do more.

<!--more-->

## Git

Git is a tool that I use every day, and I want to type as little as possible.
```sh
alias gsw="git switch"
alias gswc="git switch -c"
alias gcm="git commit -m"
alias gcam="git add -A && git commit -m"
alias gp="git push"
alias gl="git log --graph --decorate --numstat"
alias gpl="git pull"
alias gb="git branch"
alias gfop="git fetch origin --prune"
alias gsh="git show"
alias gs="git status"
alias gfix="git add -A && git commit --amend --no-edit"
alias gd="git diff"
```

I also use GitHub CLI (gh) for what makes sense for my day-to-day development. The only alias that I use is `gpr` to create a pull request and view it in the browser:

```sh
alias gpr="gh pr create --fill && gh pr view --web"
```

### Motivation & Workflow
In my workflows, I prefer the use a very lean git strategy. I never commit directly to the `main` branch. Instead, I create a new one, implement the changes, and then "squash and merge" it into the main via a pull request. This way, the main branch is always tracking the latest stable version of the codebase, and the closed PRs read like a changelog.

### Example Workflow
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

> [!TIP]
> `gcam` may fail on the first run if a [pre-commit](tools/terminal/pre-commit) hook auto-fixes files (e.g. ruff formatting). Just run `gcam` again --- the fixes are already staged.

Now, let's go through the standard PR review process, and finally merge it into the main branch. (Don't forget to delete the feature branch after merging!)

The following concludes the feature branch workflow:
```sh
gws -  # switch back
gpl  # pull the latest changes
gfop  # remove local branches that are no longer on remote
```

## Python

Working with Python projects means activating virtual environments and syncing dependencies all the time. These two aliases cut that down to a couple of keystrokes:

```sh
alias va="source .venv/bin/activate"
alias uvs="uv sync --all-groups"
```

- `va` --- activate the `.venv` in the current directory.
- `uvs` --- sync all dependency groups with [uv](https://docs.astral.sh/uv/).

A typical flow when jumping into a project:

```sh
cd my-project
va   # activate the venv
uvs  # make sure everything is in sync
```

## Misc

### Clipboard-aware `cp`

The built-in `cp` copies files, but I also want a quick way to pipe command output straight to my clipboard. This function overloads `cp` to do both:

```sh
cp() {
  if [ "$#" -eq 0 ]; then
    if [ -t 0 ]; then
      command cp
    else
      wl-copy
    fi
  else
    command cp "$@"
  fi
}
```

When called with arguments it behaves exactly like regular `cp`. When called with **no** arguments and stdin is a pipe, it forwards the input to `wl-copy` (Wayland's clipboard utility --- on X11 you'd use `xclip` or `xsel`, on macOS `pbcopy`).

A few examples:

```sh
echo "hello" | cp          # copies "hello" to clipboard
cat ~/.ssh/id_ed25519.pub | cp  # copies your public key
some-command | fzf | cp    # interactively pick a line, then copy it
```

The `fzf` combo is especially handy when a command produces multiline output and you only need one entry --- pipe through `fzf`, select the line you want, and it lands in your clipboard.
