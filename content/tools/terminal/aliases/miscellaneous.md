---
title: "miscellaneous"
date: 2026-04-09T12:00:00+02:00
description: "Miscellaneous shell aliases and functions."
tags: ["Aliases", "Python", "Clipboard"]
draft: false
params:
  neso:
    show_toc: true
---

A few more aliases and shell functions that I reach for constantly --- from Python virtual environments to clipboard tricks.

<!--more-->

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

## Other

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
