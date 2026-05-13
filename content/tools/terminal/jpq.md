---
title: "jpq"
date: 2026-05-13T15:22:00+02:00
description: "jq-style JSON filtering with Python expressions."
tags: ["Python", "JSON", "CLI"]
draft: false
weight: 15
params:
  neso:
    show_toc: true
---

[`jpq`](https://github.com/rayannott/jpq) is a tiny CLI I wrote because I have `jq` reflexes but a Python brain. JSON comes in on stdin, gets bound to `this`, an expression runs against it, and the result goes back out as JSON. No DSL to memorise --- just Python.

<!--more-->

## Install

```bash
uv tool install jpq
```

`pipx` and `pip` also work --- see the [README](https://github.com/rayannott/jpq#install).

## The model

Three things to remember, and then you're done:

- Stdin JSON is parsed and bound to `this`.
- The expression you pass is `eval`'d with `re`, `os`, `collections`, `itertools`, `statistics`, `math`, `datetime`, `pathlib` pre-imported, plus every builtin, plus an `env("NAME")` helper for env vars.
- The result is dumped back to stdout as JSON. `set`, `datetime`, and `pathlib.Path` are coerced automatically; `-c` / `--compact` strips the indentation when you want to pipe further.

That's it. Anything you can write as a single Python expression, you can pass to `jpq`.

## A demo dataset

The examples below run against a small fake log file --- 18 events across four days. Stick the curl behind an alias so the commands stay readable:

```bash
alias events='curl -s https://gist.githubusercontent.com/rayannott/8c29315b01c55903c25c0a4337a2e041/raw/d0ccb1a979c889c968fa014c14124f92ae2d245f/logs.json'

events | jpq 'len(this["events"])'
# 18
```

The schema, as a 3-event preview (one auth, one build, one deploy):

```json
{
  "fetched_at": "2026-05-13T14:00:00+00:00",
  "events": [
    {"id": "evt_001", "ts": "2026-05-10T08:14:22+00:00", "level": "INFO",  "category": "auth",   "score": 0.91, "message": "user alice logged in from 10.0.0.42",             "tags": ["mfa", "prod"]},
    {"id": "evt_013", "ts": "2026-05-12T14:20:00+00:00", "level": "ERROR", "category": "build",  "score": 0.20, "message": "build #1239 failed after 420s on agent 10.0.0.7", "tags": ["nightly", "dev"]},
    {"id": "evt_015", "ts": "2026-05-13T06:00:00+00:00", "level": "INFO",  "category": "deploy", "score": 0.93, "message": "deploy v0.4.3 to staging by dave",                "tags": ["dev"]}
  ]
}
```

Four field axes that don't overlap: `level` (outcome severity), `category` (event class), `score` (continuous health metric), and `tags` (orthogonal context --- environment, build trigger, MFA flag, deploy strategy). No `"success"` / `"failure"` in tags --- that's what `level` is for.

## Examples

### 1. Count the events --- builtins

```bash
events | jpq 'len(this["events"])'
```

```json
18
```

### 2. Level distribution --- `collections.Counter`

```bash
events | jpq 'collections.Counter(e["level"] for e in this["events"])'
```

```json
{
  "INFO": 12,
  "WARN": 4,
  "ERROR": 2
}
```

### 3. Tag frequency --- `Counter` over a flattened generator

The nested `for` in a generator expression is one of the parts of Python that earns its keep:

```bash
events | jpq 'collections.Counter(t for e in this["events"] for t in e["tags"])'
```

```json
{
  "mfa": 7,
  "prod": 8,
  "nightly": 3,
  "flaky": 2,
  "dev": 10,
  "pr": 4,
  "canary": 1,
  "new-user": 1,
  "rollback": 1
}
```

### 4. Mean score per category --- `itertools.groupby` + `statistics`

`itertools.groupby` only groups *consecutive* equal keys. If you forget to `sorted()` first you get garbage, and there's no one to blame but yourself:

```bash
events | jpq '{k: round(statistics.mean(e["score"] for e in g), 3)
 for k, g in itertools.groupby(
     sorted(this["events"], key=lambda e: e["category"]),
     key=lambda e: e["category"])}'
```

```json
{
  "auth": 0.604,
  "build": 0.649,
  "deploy": 0.775
}
```

### 5. Daily buckets --- `groupby` keyed by `datetime.date`

```bash
events | jpq '{str(d): [e["id"] for e in g]
 for d, g in itertools.groupby(
     sorted(this["events"], key=lambda e: e["ts"]),
     key=lambda e: datetime.datetime.fromisoformat(e["ts"]).date())}'
```

```json
{
  "2026-05-10": ["evt_001", "evt_002", "evt_003", "evt_004", "evt_005"],
  "2026-05-11": ["evt_006", "evt_007", "evt_008", "evt_009"],
  "2026-05-12": ["evt_010", "evt_011", "evt_012", "evt_013", "evt_014"],
  "2026-05-13": ["evt_015", "evt_016", "evt_017", "evt_018"]
}
```

### 6. Parse build messages --- `re.search` with named groups

Free-text messages become structured records. The non-greedy `.*?` handles the three different build-message shapes (`finished in Ns`, `finished in Ns with K flaky tests`, `failed after Ns on agent IP`) in one regex:

```bash
events | jpq '[re.search(r"build #(?P<num>\d+).*?(?P<dur>\d+)s", e["message"]).groupdict()
 for e in this["events"]
 if e["category"] == "build" and e["level"] != "INFO"]'
```

```json
[
  {"num": "1234", "dur": "312"},
  {"num": "1237", "dur": "510"},
  {"num": "1239", "dur": "420"}
]
```

### 7. IP-bearing events, partitioned by `level`

Filter to events whose message contains an IP, then bucket by whether they're `INFO` or not:

```bash
events | jpq '{k: [e["id"] for e in this["events"]
     if re.search(r"\d+\.\d+\.\d+\.\d+", e["message"])
     and ("info" if e["level"] == "INFO" else "non-info") == k]
 for k in ("info", "non-info")}'
```

```json
{
  "info": ["evt_001", "evt_007", "evt_010", "evt_012", "evt_018"],
  "non-info": ["evt_003", "evt_013", "evt_014"]
}
```

### 8. Score summary --- `statistics` + `math`

```bash
events | jpq '{"avg": round(statistics.mean(e["score"] for e in this["events"]), 3),
 "stdev": round(statistics.stdev(e["score"] for e in this["events"]), 3),
 "log2_n": round(math.log2(len(this["events"])), 3)}'
```

```json
{
  "avg": 0.659,
  "stdev": 0.285,
  "log2_n": 4.17
}
```

### 9. Pipe `jpq` into `jpq` to keep each stage small

`jpq`'s output is JSON, which makes it valid `jpq` input. Splitting a heavy transformation across two pipes keeps each stage trivially debuggable (run the first one alone and eyeball the result) and lets you reuse the intermediate shape.

Compare aggregating build durations as **one expression** versus as **two pipes**. Stage 1 parses the messages into structured records; stage 2 aggregates over them:

```bash
events \
  | jpq '[re.search(r"build #(?P<num>\d+).*?(?P<dur>\d+)s", e["message"]).groupdict()
          for e in this["events"] if e["category"] == "build"]' \
  | jpq '{"mean_dur_s": round(statistics.mean(int(r["dur"]) for r in this), 1),
          "max_dur_s": max(int(r["dur"]) for r in this),
          "n_builds": len(this)}'
```

```json
{
  "mean_dur_s": 334.3,
  "max_dur_s": 510,
  "n_builds": 7
}
```

Drop the second `| jpq ...` and you see what stage 1 produced --- a clean list of `{"num": ..., "dur": ...}` records --- which is also the answer to "why is my final number wrong?". Try writing the same logic as a single nested expression and you'll appreciate the pipe.

The one cost: each `|` re-serialises and re-parses JSON. For 18 events that's free; for a million log lines you'd want to fuse the stages back together (or, honestly, stop using `jpq` and write a script).

## Tips

- `-c` / `--compact` strips indentation. Useful when piping into another tool that wants single-line JSON.
- Exit codes mean something: `3` on bad stdin, `4` on an expression that fails to parse or raises at runtime, `5` on a non-JSON-serialisable result. Scripts can branch on these.
- Pipe `jpq | jq` if you really want jq's colourisation on the way out. The two of them are friends, not rivals.

> [!TIP]
> If a one-liner gets long, break it across multiple lines inside the same quoted string --- the expression compiler doesn't care about whitespace, and your shell will keep the quote open until you close it.
