This directory contains the benchmarking results for some of the Scientific Python projects' documentation builds: the terminal tables are in `matplotlib.md`, `networkx.md`, `numpy.md` and `pandas.md`, and `networkx-numpy-benchmarks.zip` has the benchmarks JSON of the networkx and numpy builds. Unzip the `networkx-numpy-benchmarks.zip` and then run `sphinx-benchmark run html --input <path to json file>` to get the HTML report for the corresponding docs build. The plan is to only have the `.zip` files containing the json files, in the future, instead of the .md files.

# Reading the benchmarking output

## HTML report

- Overview(index.html) contains the build information and a pie chart of how much % of the total build was taken up by which event (its own time) and which gap.
- Events & handlers page contains a breakdown of each event into the handlers it ran (the same numbers as Table 1 below).
  - click an event or handler name to see every emission record of that event, or every call record of that handler.
  - from that page, the "Call tree" and "Function-wise breakdown" tabs show where the time inside that event or handler went.
- Gaps page contains the gaps summary table (Table 2 below); click a gap (event pair) to see all of its individual gap records. The same two "Call tree" and "Function-wise breakdown" tabs are there for each gap pair, and on the Gaps page itself for all gaps together.
- Whole build page contains the call tree of the whole build, each box coloured by where the build was: inside a handler, inside an event but outside its handlers, or in a gap between emissions. Its "Function-wise breakdown" tab lists the same per function.

In every call tree, branches under 1% of the tree's total are left out, and the function-wise tables only list functions with a total time of at least 0.5%. Hover a box (or a function name in the table) to see where the function is defined (file:line). Every table can be re-sorted by clicking a column heading. The call trees and function-wise breakdowns are estimated from stack samples, not measured (see [Sampling](#sampling-call-trees-and-function-wise-breakdown)), and are absent if the build wasn't sampled, like if GIL was disabled.

## CLI benchmarking outputs

The `sphinx-benchmark run table` command prints the project and build information followed by the two benchmark summary tables.

### Table 1 : where time goes inside events

The first table has one block per event, sorted by the most expensive event first.
The header line of each block looks like this:

```
builder-inited  -  107.478046s own time (41.46% of build)  |  1 emissions  |  107.494970s duration (including nested event)  |  depth 0
```

- **own time** : total time spent in this event across the whole build, *excluding*
  any other events that fired from inside it. This is what the blocks are sorted by,
  and what the percentage is measured against `total_wall_time`.
- **emissions** : how many times the event fired.
- **duration (including nested event)** : only printed when this event emits other events.
  It is the total time spent in this event *including* the durations of events that were fired
  from inside it.
- **depth** : `0` means the event fired at the top level of the build. `1` means it
  only ever fired from inside another event. A range like `0-1` means both.

Underneath is a row per handler that ran during that event:

- **Handler** : The function's name
- **Kind** : `extension`, `theme`, `sphinx-internal`, `stdlib` or `unknown`
- **Ext/Module** : The extension (e.g. `sphinx.ext.autodoc`), theme, or top-level package (or file, e.g. `conf.py`) the handler belongs to; `-` for Sphinx's own handlers
- **Calls** : How many times this handler function was called
- **Total(s)** : All of its execution times added up
- **Avg(ms)** : Total / Calls

Two numbers at the end of each event block:

- **(sum of handlers)** : the handler rows added together.
- **(unaccounted overhead)** : the event's duration minus the above sum. It's
  normally small. If it's large, the time is going somewhere the per-handler timers
  can't see.

You can further see the break-down of each event emission using
`sphinx-benchmark run table events <event_name>`, and the break-down of each
handler call using either `sphinx-benchmark run table events <handler_name>` or
`sphinx-benchmark run table events <event_name> <handler_name>`.

At the very bottom:

```
Sum of own durations of all events: 131.263775s   Wall clock: 259.262810s   Outside any event: 127.999036s (49.37%)
```

That last figure is usually the surprise. Roughly half of a real build isn't inside
any event at all-- it's Sphinx reading source files, resolving references, and
writing HTML. Events are checkpoints, not phases, and nothing times the work
between them. That's what the second table is for.

### Table 2 : the gaps between events

We take all top-level event emissions (excluding any nested events), sort them by start time, and measure the time between one ending and the next one starting. Each row aggregates every gap that occurred at an event boundary:

```
html-page-context -> missing-reference    30.198548    910    33.185    11.65%
```

Read this as: 910 times during the build, `html-page-context` finished and
`missing-reference` was the next event to fire, and the stretch in between added up
to 30s (about 33ms gap each time, 11.65% of the build).

The table also has a `(startup, before first emission)` row (the time from the extension's setup to the first emission), a `(finish, after last emission)` row, and a `(total outside events)` row, which is the "Outside any event" figure at the bottom of Table 1.

If you ever see a `WARNING: N negative gaps -- top-level emissions overlap` line, two top-level
emissions overlapped. That shouldn't happen, so please report an issue for it.

You can further see the break-down of each gap using the
`sphinx-benchmark run table gaps <start-event> <end-event>` command.

### Table 3 : what's inside a gap

`sphinx-benchmark run table gaps <start-event> <end-event>` (and `run table gaps` for all
gaps together) also prints what the build was running during that gap: the top 15 functions
by self time, each with

- **Self(s)** : time in the function's own code (calls into the Python standard library count towards the caller)
- **Total(s)** : the function and everything it called

and each as a % of the gap. These numbers come from the stack samples (see
[Sampling](#sampling-call-trees-and-function-wise-breakdown)), so they are estimates.
The CLI only prints this for gaps and doesn't draw the call tree; the HTML report has
both, for events and handlers as well.

## About the JSON

We don't expect a user to read the JSON file, but here is how it is organised if you want to get into it.

Every build with this extension enabled creates a `sphinx_benchmarks_*.json` file with five top-level keys:

- `project_info` : `name`, `version` and `copyright` from `conf.py`, plus `HEAD`, the git HEAD commit hash of the docs directory (`null` if it isn't a git repo or git isn't installed)
- `build_info` : `builder` (e.g. `html`), `start_time` (UTC) and `total_wall_time`, the whole build time from the extension's setup to `build-finished`
- `events` : one record per event emission: `event_id`, `event_name`, `call` (1 for the first emission of this event, 2 for the second, and so on), `start` (seconds since build start), `depth`, `duration` (including nested emissions; `null` for `build-finished`), `parent_id` (the `event_id` this one is nested inside, or `null`) and `own_time` (`duration` minus the nested emissions)
- `calls` : one record per handler call: `event`, `handler` (its qualified name), `module`, `kind`, `extension`, `call`, `start` and `duration`
- `frames` : stack snapshots of the whole build (what was running, and when), from which the call trees and function-wise breakdowns are built (`null` if the build wasn't sampled)

### frames

- `sampling_interval`: the interval in seconds the sampler sleeps between two samples
- `samples`: the total number of samples/snapshots collected
- `functions`: one entry per distinct function ever seen in any snapshot, with
  its function name, module, file, line, kind and extension (classified the same way as handlers). Its position in
  this list is its function index.
- `stacks`: one entry per distinct stack of functions ever seen. Each is a
  list of function indexes, innermost function first, i.e. index 0 is the function
  running and the last one is the outermost function.
- `snapshots`: one entry per sample: `[seconds since build start, stack index]`.

For example:

```
"sampling_interval": 0.001,
"samples": 3,
"functions": [
    {"function": "main",         "module": "sphinx.cmd.build", ...},
    {"function": "Sphinx.build", "module": "sphinx.application", ...},
    {"function": "parse",        "module": "docutils.parsers.rst", ...}
],
"stacks":    [[2, 1, 0], [1, 0]],
"snapshots": [[0.0012, 0], [0.0021, 0], [0.0033, 1]]
```

Read it as: the first two samples saw that the docs build thread
was running inside the `parse` function, which was called by `Sphinx.build`,
which was called by `main`. The third sample saw the build inside `Sphinx.build`.

---

# How are benchmarks calculated?

At fixed points in a docs build process (e.g. after config is read, after a page is parsed, before a page is written, etc.) Sphinx emits an **event**, and every extension or theme that registered a **handler** for
that event gets called.

The extension collects benchmarks in two ways, both starts in the extension's `setup()`:

1. **Timing** : it wraps Sphinx's event machinery with perf_counters, so every event emission and every
   handler call is measured. The gaps between emissions are then derived from these start times and durations.
   This gives the event, handler and gap tables.
2. **Sampling** : a background daemon thread takes snapshots of the build thread's function call stack
   throughout the build. These are estimates and gives us the call trees and function-wise breakdowns.

## Timing: wrapping emit, connect and handlers

Nearly everything an extension does in Sphinx goes through `app.events.emit()`.
Sphinx calls it at certain points in the build process, and it runs all the
registered handlers for that event. This extension wraps and puts timers around this path.

`wrap_emit()` wraps `app.events.emit` to time the whole emission, and
`wrap_listener()` swaps each handler for a wrapped and timed copy of it.
So for every event you get the total time it took and the split across its handlers.
`wrap_all_listeners()` wraps everything already registered when the extension loads, and
`wrap_connect()` wraps `app.events.connect` so handlers registered later get wrapped
at the moment they are registered.

Each timed call becomes a `HandlerCall` record and each event emission becomes an
`Event` record, both kept in one `Recorder`. All times are measured from the moment
the extension's `setup()` ran, so everything shares a starting point.

A handler can itself emit events, so a stack of the emissions in progress is kept: each
`Event` record gets its `depth` (how many emissions it is nested inside) and its
`parent_id`. At the end, each emission's `own_time` is its `duration` minus the durations of
the emissions nested directly inside it, so nested time isn't counted twice. Table 1 and the
pie chart use own time.

Nothing is timed between two emissions. A gap is therefore not measured directly: the
top-level emissions (depth 0) are sorted by start time and a gap is the time from one
ending to the next one starting.

At `build-finished` (connected with priority 999, so other extensions' `build-finished`
handlers run first and get recorded) the extension stops the sampler, works out each
event's own time, classifies every handler by the module it is defined in (`sphinx.ext.*`
is an `extension`, anything else under `sphinx` is `sphinx-internal`, a package that
registers an HTML theme is a `theme`, a package in `app.extensions` is an `extension`,
the standard library is `stdlib`, and everything else is `unknown`), and dumps everything
into the JSON.

## Sampling: call trees and function-wise breakdown

The timers only see events and handlers, so they can't tell which functions the time inside
an event, a handler or a gap went to. For that, `setup()` also starts a background daemon thread
(`StackSampler`). It sleeps for 1 ms (the sampling interval), then reads the docs build thread's current
function call stack with `sys._current_frames()`, walks it with `frame.f_back` from the running function up to
the outermost one, and stores it along with the time since the build started. To take a sample the thread needs
the GIL, which the build thread hands over at its next I/O call or after Python's GIL switch interval
(5 ms by default), so two samples can be more than 1 ms apart. The samples go into the JSON as `frames`.

When a report is made, the samples are turned into call trees and function-wise breakdown tables in four steps:

### 1. Placing each sample

Every sample has a start time, so it is compared with the recorded event emissions and handler
calls records and placed in the innermost thing the build was in at that moment: a handler call, an event emission
(outside its handlers) or a gap between two top-level emissions (or before the first, or after the last).
The samples of a gap are exactly those placed in it. An event gets the samples placed in its emissions and in
their handlers, but not those in nested emissions, so they match its own time. A handler gets every sample
taken while one of its calls was open, nested emissions included, so they match its measured duration.
The whole build simply gets all samples.

### 2. Trimming the stacks

A sample's stack runs from the function that was running all the way out to
Sphinx's `main`. For an event, only the part inside `emit()` is kept; for a handler,
only the part inside the handler itself. Gaps and the whole build keep the full stack.

### 3. Building the tree

Every stack is walked from the outermost function inwards. Stacks that start
with the same functions share those nodes, and a new branch begins where they differ. The sample is
counted as "self" time of its innermost function; a node's "total" is its own self plus everything
below it. Each sample is worth `measured time / number of samples` of that event, handler or gap, so a
call tree always adds up to the measured time. (For the whole build this rate is worked out per handler,
event and gap rather than once, because phases that do I/O or call into C hand over the GIL more often
and so get more samples per second.)

### 4. The function-wise table 

The tree is flattened to one row per function, wherever it appears in
the tree. Its self time is the sum of its self times; time in the Python standard library is charged
to the nearest caller that is not in the standard library, so `open` or `re.match` never appear as
hot spots themselves. Its total time is the sum of its nodes' totals, counting a function only once
per stack if it calls itself. Rows are sorted by self time.

For example, with the three samples of the `frames` example above falling in a gap measured at 0.3s,
each sample is worth 0.1s and the tree is

```
main          total 0.3s  self 0.0s
└─ Sphinx.build   total 0.3s  self 0.1s
   └─ parse       total 0.2s  self 0.2s
```

and the table has `parse` first (self 0.2s), then `Sphinx.build` (0.1s), then `main` (0.0s, total 0.3s).

These numbers are estimates, not measured times: so a function seen in only a few samples is considered noise.
