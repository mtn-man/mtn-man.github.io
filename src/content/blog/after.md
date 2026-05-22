---
title: "after: a terminal timer that behaves"
description: "A walkthrough of after, a small CLI countdown timer I built in Go, and some of the implementation decisions that made it surprisingly interesting to write."
pubDate: 2026-05-22
---

<!-- TODO: Opening paragraph — set the scene. What problem were you actually solving?
     e.g. "I wanted something I could drop in a script and not think about.
     sleep(1) doesn't show you anything. watch is for polling. Nothing quite fit."
     Keep it short — one or two sentences of motivation before diving in. -->

The result is [`after`](https://github.com/mtn-man/after) — a small Go program that counts down to a duration or time of day, shows you a live timer, plays a sound when it's done, and gets out of the way.

```bash
after 10m        # countdown from now
after 2:30pm     # countdown to next 2:30 PM
after 1.5h       # decimals work too
```

Install it with Homebrew:

```bash
brew install mtn-man/tools/after
```

Or with Go:

```bash
go install github.com/mtn-man/after@latest
```

## What it does

<!-- TODO: Brief description of the interactive experience — what you see when you run it,
     what the countdown looks like, what happens when it finishes.
     You can reference the demo.gif from the README if you want to embed it here. -->

The display stays quiet about irrelevant fields: `1:23` for 83 seconds, `1:02:03` for just over an hour. Once the time is up, it plays an alert and exits. Hit `q`, `esc`, or `ctrl+c` to cancel early.

It also handles wall clock times, wrapping to tomorrow if the time has already passed:

```bash
after 9am   # if it's 10am, this counts down ~23 hours
```

## A few things that were interesting to build

### Keeping stdout clean

Most timer tools mix status output and timing output on the same stream. `after` sends the countdown and lifecycle messages (`after: started`, `after: complete`, `after: cancelled`) exclusively to `stderr`, leaving `stdout` untouched. That means it behaves correctly in pipelines and scripts without any extra flags:

```bash
after 10m 2> /tmp/timer.log    # log lifecycle, silent countdown
after -s 10m 2> /dev/null &    # run in background, still plays alarm
```

When `stderr` is not a TTY (redirected to a file or pipe), the live countdown is suppressed automatically — only the one-line lifecycle events are emitted. No flickering escape sequences in your log files.

### Two TTY checks for two different things

The tool makes two independent TTY checks that control different behaviors:

- **`stderrIsTTY()`** — controls whether to render the live countdown and use ANSI escape sequences
- **`stdoutIsTTY()`** — gates side effects: alarm playback and macOS sleep inhibition (`caffeinate`)

The split matters. If you redirect stderr to a log file but leave stdout attached to your terminal, the countdown disappears but the alarm still plays when the timer finishes. If you redirect stdout into a pipeline, the alarm is suppressed. The behavior matches what you'd intuitively expect, but getting the two checks right required thinking carefully about which behavior belongs to which stream.

### Playing the alarm without blocking your prompt

When the timer completes, you want your shell prompt back immediately — not after waiting for an audio file to finish playing. The naive approach (just call `afplay` and wait) blocks the process until the sound is done.

`after` solves this with a self-re-execution trick: when the timer completes, the main process spawns a detached child instance of itself with a sentinel argument (`__after_internal_alarm_worker`). The child plays the alarm; the parent exits immediately and returns your prompt. The child is detached from the process group (`Setpgid: true`) so it survives the parent's exit and doesn't receive terminal signals.

<!-- TODO: Any personal note on this — was this the first approach you tried, or did you
     arrive at it after something else didn't work? A sentence of context makes it more
     interesting than just explaining the mechanic. -->

### Cross-platform audio with graceful fallback

<!-- TODO: Brief personal note on why cross-platform support mattered to you
     (do you use Linux on your homelab? BSDs? Both?) -->

Rather than hard-coding one audio command, `after` builds a platform-specific candidate list at startup and filters it through `exec.LookPath`:

- **macOS**: `afplay` with the built-in `Submarine.aiff`
- **Linux**: `canberra-gtk-play` → `speaker-test` fallback
- **FreeBSD / OpenBSD / NetBSD**: `beep` or `canberra-gtk-play`

If nothing is available, the alarm fails silently — no error, no fuss. You can also supply your own file with `-f ~/sounds/bell.mp3`.

## Using it in scripts

<!-- TODO: Share a real example from your own workflow. What do you actually use this for?
     e.g. a reminder to check a build, a reminder to take a break, a deploy wait, etc. -->

A few patterns that work well:

```bash
# wait for something, then run a command
after 5m && notify-send "Check the build"

# background timer with alarm, no terminal output
after -s 30m 2>/dev/null &

# quiet mode — suppress status, keep alarm
after -qs 10m
```

## Get it

Source and releases are on GitHub: [github.com/mtn-man/after](https://github.com/mtn-man/after).

<!-- TODO: Closing line — anything you're planning to add, or just a
     "it does what I need it to do" sign-off. -->
