# benchsum

**Frame-time statistics for MangoHud logs: average FPS, 1% / 0.1% lows, frame-time percentiles,
spike counts, and whether a difference between configurations is bigger than run-to-run noise.**
For comparing Linux gaming settings (Proton versions, ntsync vs fsync, DXVK options, CPU topology,
kernels, drivers) with repeated benchmark runs instead of single screenshots of an FPS counter.

Part of a small toolkit for investigating Linux gaming performance:
[gamerun](https://github.com/JasonBurnsInfosec/gamerun) (run and record) · **benchsum** (frame-time statistics) ·
[threadwatch](https://github.com/JasonBurnsInfosec/threadwatch) (per-thread CPU and wait profiling)

## Why

Two runs of the same benchmark rarely give the same number. A "3% faster" result can be noise, and
a real 3% gain can hide in it. benchsum groups repeated runs per configuration, reports the mean
with a 95% confidence interval, and runs a Welch's t-test against a baseline, so you can say "+8.8%,
p = 3×10⁻⁵" instead of "seems faster". It also cuts the benchmark out of each log, because averages
over a whole session are dominated by menus and loading screens.

## Features

- Reads MangoHud CSV logs, folders of them, or [gamerun](https://github.com/JasonBurnsInfosec/gamerun)
  run directories (label, env vars, file edits and benchmark window come from `manifest.json`)
- **Benchmark windows**: auto-detect a fixed-length benchmark after the loading screen, or use fixed
  seconds or gamerun markers
- **Grouping** by label, experiment, environment variable, file edits or any manifest field
- Metrics: average FPS, 1% and 0.1% lows, p50/p95/p99 frame time, frames over 33 ms and 50 ms per
  minute, CPU/GPU load
- Statistics: mean ±95% CI per group, Welch's t-test with p-values against a baseline
- Output: aligned text, Markdown (paste into GitHub issues), JSON, or CSV (one row per run)
- Python 3.8+, standard library only, a single file

## Install

```sh
git clone https://github.com/JasonBurnsInfosec/benchsum ~/src/benchsum
ln -s ~/src/benchsum/benchsum ~/.local/bin/benchsum
```

## Quick start

```sh
# MangoHud logging (gamerun does this for you): MANGOHUD=1 MANGOHUD_CONFIG=output_folder=$HOME/logs/fsync,autostart_log=1,log_interval=0 %command%
benchsum ~/logs/ntsync ~/logs/fsync --baseline ntsync                 # one folder per configuration
benchsum ~/logs/ntsync ~/logs/fsync --baseline ntsync --auto-window 112 --markdown
benchsum runs/214950-*/* --group-by env:PROTON_NO_NTSYNC --baseline 'PROTON_NO_NTSYNC=(unset)'
```

Example output (Total War: ROME II built-in benchmark, 4 interleaved runs each):

```
config            n    avg FPS     1% low  0.1% low  p50 ms  p95 ms  p99 ms  >33ms/min  >50ms/min  CPU%  GPU%
----------------  -  ---------  ---------  --------  ------  ------  ------  ---------  ---------  ----  ----
fsync             4  58.5 ±1.0  29.8 ±3.4      18.6   16.63   24.29   30.06        9.5        1.1     6    36
default-windowed  4  53.8 ±0.7  29.8 ±2.9      18.1   18.29   25.82   29.65        9.4        1.1     6    36

window: auto:112:250

vs baseline "default-windowed" (Welch t-test; * = significant at 95%):
config  avg FPS            1% low        0.1% low      p99 ms       >33ms/min
------  -----------------  ------------  ------------  -----------  ------------
fsync   +8.8% p=2.8e-05 *  +0.2% p=0.97  +2.6% p=0.47  +1.4% p=0.8  +1.4% p=0.98
```

Reading it: fsync is 8.8% faster on average, and with p = 0.00003 that is very unlikely to be noise.
The lows and spikes are the same in both modes.

## Inputs

| input | label |
|---|---|
| a MangoHud CSV | file name without the timestamp |
| a folder of MangoHud CSVs | the folder name; every CSV inside is a repeat |
| a gamerun run directory | `label` from `manifest.json`; the longest log in it is used |

## Options

| option | |
|---|---|
| `--baseline GROUP` | compare every other group against this one |
| `--group-by KEYS` | `+`-separated: `label` (default), `experiment`, `run`, `append` (gamerun file edits), `env:NAME`, `field:KEY` (any manifest key) |
| `--auto-window D[:MS]` | measure a D-second benchmark that starts right after the last loading stall (frame slower than MS ms, default 250) with D seconds of log after it |
| `--window A-B` | measure log seconds A to B |
| `--markers S:E` | measure between gamerun markers S and E |
| `--no-manifest-window` | ignore the experiment's `bench_window` |
| `--write-window` | save the window used as `<run dir>/window.json` (used by gamerun and threadwatch) |
| `--skip-start S`, `--skip-end S` | without a window: trim the start / end of each log |
| `--runs` | also list every run, with its window and sync mode |
| `--include-failed` | keep gamerun runs whose hook failed or was cut off (`hook_status` not `ok`; skipped by default) |
| `--markdown` | tables ready for GitHub |
| `--json` | per-run metrics, per-group stats and per-metric tests against the baseline |
| `--csv` | one row per run with every metric |

Window precedence: `--window`, `--auto-window`, `--markers`, the manifest's `bench_window`, whole log.
Group names combine the keys, e.g. `--group-by env:WINE_CPU_TOPOLOGY+env:PROTON_NO_NTSYNC` gives
`WINE_CPU_TOPOLOGY=3 / PROTON_NO_NTSYNC=1`; use the full name as `--baseline`.

### How `--auto-window` finds the benchmark

Loading screens produce frames of hundreds of milliseconds. `--auto-window D` takes the last such
stall (frame > 250 ms by default) that still has D seconds of log after it, and measures the D seconds
starting 0.3 s later. It suits fixed-length built-in benchmarks (set D to the benchmark's length).
Check the reported window with `--runs` the first time you use it on a new game.

## Metric definitions

| metric | definition |
|---|---|
| avg FPS | frames ÷ total frame time (time-weighted, not the mean of per-frame FPS) |
| 1% / 0.1% low | 1000 ÷ mean of the slowest 1% / 0.1% of frame times |
| p50 / p95 / p99 | frame-time percentiles (ms) |
| >33ms/min, >50ms/min | frames slower than 33.3 / 50 ms per minute: absolute spike counts, comparable across configurations |
| rel stutter (JSON/CSV only) | frames slower than 2.5× the median **and** more than 4 ms over it. Relative to the median, so it rises when a configuration gets *faster*; don't compare it across configurations |
| ±N | 95% confidence interval of the group mean (t-distribution) |
| p | two-sided Welch's t-test p-value against the baseline |
| CPU% / GPU% | MangoHud's system CPU load / GPU load columns, averaged |

## Statistics, briefly

- **n**: number of runs in the group. With n < 3 there is no useful noise estimate.
- **±95% CI**: if you repeated the whole experiment many times, the true mean would fall inside this
  range 95% of the time. Overlapping ranges don't automatically mean "no difference"; use the test.
- **Welch's t-test**: compares two group means without assuming equal variance. `*` marks p < 0.05.
  No star does not prove there is no difference; it means you can't tell with these runs.
- **Interleave runs** (A B B A …) rather than all A then all B, so slow drift (heat, background
  activity, caches) affects both groups equally. `gamerun batch` does this.
- 1% and 0.1% lows are much noisier than averages; expect to need more runs to show changes in them.

## Limitations

- MangoHud CSV only (the format with a system-info header, `fps,frametime,…,elapsed` columns). Tested
  with MangoHud 0.8.x.
- Marker windows rely on the log's file-name timestamp (1 s resolution).
- One log per run: in a gamerun run directory with several logs (relaunches), the longest is used.

## Related

- [gamerun](https://github.com/JasonBurnsInfosec/gamerun): records runs and calls benchsum automatically
- [threadwatch](https://github.com/JasonBurnsInfosec/threadwatch): explains *why* a configuration is faster (per-thread CPU, waits)
- [MangoHud](https://github.com/flightlessmango/MangoHud): produces the logs

## License

MIT, see [LICENSE](LICENSE).
