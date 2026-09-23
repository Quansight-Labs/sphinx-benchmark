# sphinx-benchmark

This is a Sphinx extension that benchmarks and profiles a docs build process [event](https://www.sphinx-doc.org/en/master/extdev/event_callbacks.html)-wise, handler-wise, and gaps-wise (i.e. gaps in between the events). This extension also provides estimated function-wise breakdown table and function call trees for the whole docs build, individual event, handler and gap. By using this extension one can investigate which function, extension, theme, or part of Sphinx itself is slowing the docs builds.

**Note** : This extension is still in its early stages of development-- so it will change a lot and there might be a lot of bugs in it right now. So don't use it in production yet!

## Usage

1. Install the sphinx-benchmark extension

   ```bash
   pip install sphinx-benchmark
   ```

   Or to install the latest version

   ```bash
   pip install git+https://github.com/Schefflera-Arboricola/sphinx-benchmark.git@main
   ```

2. Add the extension to your `conf.py`:

   ```python
   extensions = ["sphinx_benchmark", ...]
   ```

   Put it first in the list as it minimizes (but doesn't eliminate) the untracked starting time.

   Optionally, you can set how often (in seconds) the docs build's function call stack is sampled for the
   function-wise breakdowns and call trees (default: `0.001`); or you can also switch off sampling if you
   don't want the function-wise breakdowns and call trees (bcoz creating those do add a bit of overhead):

   ```python
   sphinx_benchmark_sampling_interval = 0.005
   disable_sampling = True  # False by default
   ```

   To understand what sampling is, read [this](https://github.com/Schefflera-Arboricola/sphinx-benchmark/blob/main/benchmarking_outputs/README.md#sampling-call-trees-and-function-wise-breakdown).

3. Then build your docs as usual:

   ```bash
   sphinx-build -b html docs/ docs/_build/html
   ```

   The build generates a `sphinx_benchmarks_<date>-<build's start time>_<HEAD commit's last 7 chars>.json` in the present working
   directory (e.g. `sphinx_benchmarks_20260907-143012_306917b.json`).

4. [Recommended format - HTML] Change the directory to the build directory (where the generated `sphinx_benchmarks_*.json` is present) and run
   the following to generate the benchmarks report in html format by running:

   ```bash
   sphinx-benchmark run html
   ```

   Then a `sphinx_benchmark_report` folder will be created in your build directory. Open the `index.html` present inside
   `sphinx_benchmark_report` in your browser to see the overview and events, handlers and gaps breakdown.

   You can also specify the output directory for where you want the `sphinx_benchmark_report` folder to get created, 
   using `--output-dir` option. Or specify a different json file using the `--input` option.

5. To see the benchmarks in the terminal, run:

   ```bash
   sphinx-benchmark run
   ```

   This prints the top 10 events and gaps that take up the most of the build time,
   sorted by % of build descending. Following is the output for the numpy's docs build:
   
   ```bash
   % sphinx-benchmark run         

   Project: NumPy 2.6.dev0  |  HEAD: e29186cf91a8d314002a4085ae7e3c2fe6b4cc6e
   Builder: html  |  Started: 2026-09-23 11:25:09 UTC  |  Sampling interval: 5 ms
   
   Build time: 355.869547s   Inside events: 56.104345s (15.77%)   Outside events (gaps): 299.765201s (84.23%)
   ========================================================================================================
   Top 10 of 70 events and gaps, by % of build
   ========================================================================================================
     Name                                                           Type        Time(s)     Count   % build
   --------------------------------------------------------------------------------------------------------
     html-page-context -> doctree-resolved                           gap     121.891773      2290    34.25%
     autodoc-process-docstring -> object-description-transform       gap      51.052618      2217    14.35%
     doctree-resolved -> html-page-context                           gap      40.945038      2676    11.51%
     html-page-context -> missing-reference                          gap      15.707065       385     4.41%
     autodoc-process-docstring                                     event      15.010674      4251     4.22%
     builder-inited                                                event      13.336338         1     3.75%
     object-description-transform -> doctree-read                    gap      12.621546      2215     3.55%
     autodoc-process-signature                                     event      11.431854      4639     3.21%
     (finish, after last emission)                                   gap      11.065856               3.11%
     source-read -> doctree-read                                     gap      10.347389       268     2.91%
   --------------------------------------------------------------------------------------------------------
     events total                                                             39.778866              11.18%
     gaps total                                                              263.631286              74.08%
     (60 more rows; use --top N to show more)
   ```

   Use `--top` to change how many rows are shown, e.g. `sphinx-benchmark run --top 20`.
   It picks the most recent `sphinx_benchmarks_*.json`, or you can also pass a specific .json 
   by using the `--input` option.
 
   For the full tables (the per-event handler tables plus the gaps summary), run
   `sphinx-benchmark run table`; it also accepts a selector for more focused views:

   ```bash
   sphinx-benchmark run table events                      # only the per-event handler tables
   sphinx-benchmark run table events <event-name>         # every emission of that event, with all details
   sphinx-benchmark run table events <handler-name>       # every call of that handler from any event
   sphinx-benchmark run table events <event-name> <handler-name>    # that handler's calls during that event emission only
   sphinx-benchmark run table gaps                        # the gaps summary table, then function-wise breakdown of where the time of all gaps goes (based on sampling)
   sphinx-benchmark run table gaps <start-event> <end-event>  # details of every individual gap between those two events, then function-wise breakdown of where the time of all gaps with give start and end event goes (based on sampling)
   ```

   You can find the benchmarking outputs for different Scientific Python projects in the 
   [benchmarking_outputs](./benchmarking_outputs/) directory. For more on how to read benchmarking
   output/report and how benchmarks are calculated see [the benchmarking_outputs README](./benchmarking_outputs/README.md).


## Limitations/pain points

### Major ones

Wherever necessary, appropriate warning messages regarding the following are printed:

- No parallel builds: 
   - The recorder lives in the main process only, so `parallel_read_safe` and `parallel_write_safe`
     are both `False`. Sphinx will fall back to a serial build even if you pass `-j auto`, which
     means the wall-clock total won't match what you'd normally see, when you are building with parallelism.
   - Also sampling is done by a background daemon thread and if the docs build thread and the sampler
     daemon thread will run in parallel then we might end up with incorrect function stacks.
- The function-wise breakdowns (gaps, events, handlers, whole build) are estimated from samples, not measured: a function seen in only a few samples is noise, so short gaps and handlers/events could be unreliable, and anything that holds the GIL in C code is accounted to the Python function that called it.
- Sampling (and therefore function-wise breakdown tables and call-tree) only works with GIL enabled: 
the sampler thread reads the docs build thread's function frames while holding the GIL, which is 
only safe because the build thread is paused meanwhile. On a free-threaded Python with the GIL
disabled the sampler is not started (the JSON has `"frames": null`, so no call trees) because otherwise
we might record incorrect function stacks and therefore generate incorrect benchmarking report; 
run with `PYTHON_GIL=1` to ensure that the sampling is done.

### Other minor limitations

- the extension itself also adds a little bit of overhead to the build process. 
- Wall clock time is not CPU time: caches, background processes, and network fetches all are included in the total time. Run benchmarks more than once before concluding anything.
- Some handlers can't be classified: Handlers defined in `conf.py`, or in a package that
doesn't match anything in `app.extensions`, are classified as `unknown` and reported by
module or file name. Partials and callable objects have no `__qualname__`, so they're
labelled by whatever name could be recovered.
- the benchmarks .json can sometimes be too big (depends on the number of samples collected)
- The `build-finished` emission has `duration=None`. The handler that writes the JSON runs
inside that emission, so the emission hasn't ended yet when it's serialised. It's stored
with `duration=None` and the summary skips it. Anything after it, like `builder.cleanup()`
isn't measured-- but it is usually just a few seonds at the end.
- The startup blind spot: Timing begins at the extension's `setup()`. The "startup, before first emission" row in the gaps table covers only what happened after that point. The startup time is usually just a few seconds.

---

Feel free to contribute to sphinx-benchmark-- open issues for any problems you face, or any
feedback you'd like to give, or for new features you'd like to see implemented. And if you'd
like to open a pull request, we'd be happy to review it.

All code in this repository is available under the Berkeley Software Distribution (BSD) 3-Clause License.
(see [LICENSE](LICENSE))


Thank you for stopping by :)
