+++
pubdate = "2026-06-21"
lastmod = "2026-06-21"
draft = false
title = "From Python Logging Library to `nnlogging2` - Episode 1"
+++

![python-logo](/nnlogging2/python-logo.png)

Let's talk about [`nnlogging2`](https://codeberg.org/amas127/nnlogging2) (currently
*v0.2.0*) — a logging library built on top of Python standard
[`logging`](https://docs.python.org/3/library/logging.html) module.

## Why Logging Matters

In any non-trivial project, three things tend to go wrong:

- Critical information drowns in a flood of debugging noises.
- Different modules spit out messages that look identical.
- Messages rarely carry an explicit severity.

The built-in [`logging`](https://docs.python.org/3/library/logging.html) module solves
them:

- You can route logs to different destinations — console, files, syslog, wherever.
- Each module can tag its output with its own name, so `"connection failed"` from the
  database layer doesn't look the same as `"connection failed"` from the HTTP client.
- Every message carries a severity level — `DEBUG`, `INFO`, `WARNING`, you name it.

So far so good. The standard library has text logging covered. But here's the thing: if
you're doing machine learning, you're not just logging strings.

## Logging in Machine Learning Projects

From what I've seen, most serious ML projects end up stuffing a homegrown logger
somewhere in their source tree. It usually does the followings:

- **Message handler** — wraps `print()` with timestamps and log levels. Fine, but barely
  better than `print`.
- **Metrics tracer** — tracks loss, accuracy, learning rate, and writes them to a file
  or prints them every few steps.
- **Production manager** — the kitchen sink. Saves checkpoints, sends Slack alerts, logs
  to a database, sets up TensorBoard.

I've lost count of how many codebases I've seen where `Logger` instances get passed
around like a hot potato:

```python
def train(..., logger: MyLogger): ...
def eval(..., logger: MyLogger): ...
```

And most of them lean on the *Singleton* pattern — one logger to rule them all. That's
not inherently wrong. [`loguru`](https://loguru.readthedocs.io/en/stable/overview.html)
(another popular logging solution) recommends the exact same thing. But here's the
problem: a "logger" is supposed to handle text. When you start bolting on **tracers**
and **managers**, you're quietly reinventing a management tree inside your logger —
never mind that the `logging` module already comes with a tree structure built in.

That's where I got stuck the first time around.

## What Went Wrong with `nnlogging`

Before `nnlogging2`, there was [`nnlogging`](https://codeberg.org/amas127/nnlogging). It
worked, but developing it was a slog. Its scope crept way beyond text handling. I pulled
in [`dvc`](https://dvc.org/) for experiment versioning and
[`duckdb`](https://duckdb.org/) for storing metrics — and they never really played
nicely together. On top of that, logging was at least **10x slower** than it needed to
be, because the library was busy managing its own internal tree on every single call.
Making it extensible or sellable was just as painful.

The library is now archived — you can still find it on [PyPI](https://pypi.org/) or
[my repo](https://codeberg.org/amas127/nnlogging) if you're curious:

```bash
$ pip install "nnlogging==0.2.0a1"  # Last release version
```

## Design Rationale

### Record Injection

`nnlogging` failed because it tried to be too many things. A logger should **log**. It
shouldn't embed a database, run a manager tree, or double as your experiment tracker.
Those tools don't belong inside a logging library — they belong alongside it.

`nnlogging2` starts from a different place: **the standard `logging` module is already
good at what it does.** We don't replace it; we extend it.

Here's the trick. Python's `LogRecord` has an `extra` dictionary that can hold arbitrary
data — but standard formatters have no idea what to do with it. Our approach is dead
simple:

- Call `logger.update(loss=0.42)` and the value drops into an internal sliding window.
- Call `logger.log_metric(level, msg)` and `nnlogging2` copies a snapshot of the current
  window into `record.extra` under a configurable key (default: `"metrics_ext"`).

Most ML loops track per-step values — loss at step *t*, accuracy at step *t*. But when
you only log every 100 steps, the last value is usually not what you want. You want the
**mean** of the last 100. That's what the sliding window is for.

### Instant Computation

The sliding window is just a `collections.deque` with a fixed `maxlen`. When the
formatter asks for a metric, `nnlogging2` computes `mean`, `median`, and `max` on the
fly with NumPy. No pre-aggregation, no extra state.

Two variants come out of the box:

- **`NullableMetric`** — fine with `None` values (they get turned into `NaN`). Handy
  when a metric only makes sense part of the time — like validation accuracy that you
  only compute every few epochs.
- **`NonnullableMetric`** — rejects `None`. For metrics that always have a value, like
  training loss.

Window size and per-metric format strings (`".4f"`, `".2e"`, whatever you need) are all
configurable through the fallback system.

### Extensibility

Pretty much every knob in `nnlogging2` — window size, format strings, the key name used
to inject data into `LogRecord` — should be tweakable per-instance but still ship with
sensible defaults. Classic Python problem: class attributes get clobbered the moment a
subclass or instance sets them.

`nnlogging2` solves this with a `Fallback` descriptor and a `FallbackProtectorMeta`
metaclass. The gist: when you set `MyFormatter.fmtstr`, instead of replacing the
descriptor with a plain string, the metaclass quietly updates the descriptor's internal
fallback value. Your instance override stays an override; the class-level default stays
a default. No surprises.

### Commitment

No database. No DVC. No manager tree. Each of those deserves its own tool, built and
tested on its own terms. `nnlogging2` does one thing — metric logging — and that's the
whole point.

## Conclusion

`nnlogging2` is a deliberate step back from its predecessor. Where `nnlogging` shoved
databases, experiment managers, and DVC into logging system, `nnlogging2` strips all of
that away and does exactly one thing well: tracking numerical metrics through the
standard logging pipeline.

### What's Next?

This post covers the *why* and the high-level *how*. Future episodes will dig into:

- **Fallback System** - A closer look at the descriptor and metaclass.
- **Customization** - Further extension on top of `nnlogging2`.
- **Benchmarks** - Performance benchmarking compared to standard logging library.
- **Reflection** - How to make Machine Learning project development easier.

### Try It Out

```bash
pip install nnlogging2
```

Source, docs, and issues live over at
[codeberg.org/amas127/nnlogging2](https://codeberg.org/amas127/nnlogging2).
