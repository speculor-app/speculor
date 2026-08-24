# `speculor_cli` — headless runner

`speculor_cli` runs a saved `.speculor` project without the Qt GUI. Useful for batch processing, server deployments, regression testing, and CI. It ships in the same release archive as the GUI; the binary sits alongside `speculor_app` in the extracted install directory.

## Usage

```
speculor_cli <project.speculor> [plugin_dir] [options]
```

| Argument / option | Default | Description |
|-------------------|---------|-------------|
| `project.speculor` | required | Path to a project file. Same format as the GUI saves — see [project-format.md](project-format.md). |
| `plugin_dir`      | `./plugins` | Directory scanned recursively for `.so` / `.dll` / `.dylib` plugins. |
| `--no-stream`     | off | Run engine only — skip binding any MJPEG ports even when layouts are flagged `stream.enabled`. |
| `--stream-only=<layout-name>` | (all enabled) | Repeatable. Restrict streaming to the named layouts. |
| `--log-stderr`    | off | Mirror log entries to stderr in addition to the log file. By default the CLI is silent on stderr (one startup banner aside) so stdout stays clean for redirection / piping. |
| `--pipeline-depth=<N>` | (preference) | GPU coalesced-subgraph frames kept in flight; clamped to `[1, 3]`. Overrides Preferences → Performance → "GPU pipeline depth". |
| `--pool-frames=<N>` | (preference) | Frame pool depth in slots, clamped to `[4, 64]`. Frame pools are shared per `(width, height, format)` across **all** nodes and chains, so raise this when running many concurrent same-resolution sources. Costs `N × frame-size` of RAM per distinct geometry. Overrides Preferences → Performance. |
| `--pool-budget-mb=<N>` | (preference) | Cap pool RAM per geometry; the pool depth is derived from it and the frame size. `0` = unlimited. Overrides Preferences → Performance. |
| `--no-preferences` | off | Ignore the shared settings store entirely and run on built-in defaults — for reproducible CI runs that must not inherit local configuration. Command-line flags still apply. See [Configuration](#configuration). |
| `--run-seconds=<N>` | off | Auto-stop cleanly after `N` seconds. |
| `--license-file=<path>` | (cached file) | Use a licence file from an explicit path instead of the cached one. See [licensing.md](licensing.md#cli-usage). |
| `--activate=<key>` | — | Activate this machine against a licence key, write the signed licence file, and exit. Optionally with `--machine-name=<name>`. |

### Recording & replay

Experimental, and gated to a **Personal** licence or higher in their entirety — see [recording.md](recording.md).

| Option | Default | Description |
|--------|---------|-------------|
| `--record[=<dir>]` | off | Record the whole pipeline to a session folder under `<dir>/<timestamp>/`. Destination: `<dir>` if given, else the folder set in Preferences → Recording, else `recordings/session` relative to the working directory. Captures compressed video per source node, plus tables/scalars/records/control/signals/parameter writes/events/health as structured traffic. Long runs segment automatically; if the disk can't keep up, the loss window is recorded as a gap rather than stalling the pipeline. |
| `--record-frames=all` | (preference) | With `--record`: capture **every** node's frame outputs (intermediate stages), not just sources. Forces Preferences → Recording → Capture mode = **Full** for this run. |
| `--record-standby=<sec>` | (preference) | Pre-record standby (implies `--record`): the last `<sec>` seconds ring in RAM and nothing hits disk until the incident trigger fires — then past + live are recorded. An untriggered session is discarded on stop. Overrides the configured pre-roll window. |
| `--record-trigger-after=<s>` | — | Fire the incident trigger `<s>` seconds into the run. |
| `--mark=<sec>[-<sec>]:<label>[@<lat>,<lon>]` | — | With `--record`: inject a manual **event** at `<sec>` into the run (a `<start>-<end>` range makes an interval); repeatable. An optional `@<lat>,<lon>` suffix attaches a geo point. |
| `--replay=<session-dir>` | — | **Reinjection replay**: rebuild the pipeline from the session's embedded graph and *run* it, with the recorded source nodes replay-driven — their plugins never start; the engine feeds them decoded video plus recorded data at recorded pace, and everything downstream recomputes live. No project argument (an optional positional arg is the plugin dir). Interoperability extensions (SAPIENT/DDS) are disabled so a replayed incident can't re-publish to live peers. Combine with `--record` to capture the recomputed run. |
| `--replay-speed=<x>` | `1.0` | Replay pacing multiplier for `--replay` (0.05–50; e.g. `4` replays a 20 s session in 5 s). |
| `--replay-set=<n>.<p>=<v>` | — | With `--replay`: override node `<n>`'s parameter `<p>` to `<v>` before the run starts (repeatable, applied after the session's saved params). The what-if knob — same recorded input, new settings. |
| `--replay-dump=<session-dir>` | — | Open a recorded session read-only, print its channels plus sampled values across the timeline, then exit. No project or plugins needed. |
| `--export=<session-dir>` | — | Extract a session segment to standard files and exit: video **remuxed without re-encode** to MP4 (H.264/HEVC) or MKV (other codecs), tables to CSV, everything else to JSONL. Range: `--export-from=<sec>` / `--export-to=<sec>`; output dir: `--export-out=<dir>` (default `<session-dir>/export`). No project or plugins needed. |

Example:

```bash
./speculor_cli my_pipeline.speculor ./plugins
```

The CLI loads plugins from the directory, builds the pipeline, starts the engine, and (unless `--no-stream`) brings up one MJPEG HTTP server per layout marked `stream.enabled` in the project. It runs until interrupted with `Ctrl+C` (`SIGINT`) or `SIGTERM`. On shutdown it stops every streamer, calls `engine.stop()`, and exits with status 0.

Record a run, then re-run the recording at 4× with one parameter changed and capture the outcome:

```bash
./speculor_cli my_pipeline.speculor ./plugins --record=/data/sessions --run-seconds=60
./speculor_cli --replay=/data/sessions/20260715-101500 ./plugins \
    --replay-speed=4 --replay-set=3.confidence_threshold=0.4 --record
```

## Configuration

`speculor_cli` reads the **same settings store the GUI writes**, so a machine configured once behaves the same however it is launched. Configure it in the GUI's Preferences, then run the CLI on the same box and the pipeline uses those values — no flags, no environment variables.

| Configured in | Applies headlessly |
|---------------|--------------------|
| Preferences → **Reliability** | Watchdog: hang timeouts, auto-restart, max restarts, restart cooldown, poll interval. |
| Preferences → **Performance** | Frame pool size and budget, GPU pipeline depth, GPU submit diagnostics. |
| Preferences → **GPU** | Preferred device (index / UUID), disabled Vulkan loader layers, validation layers. |
| Preferences → **Recording** | Destination folder, capture mode, pre-roll, part / spool / min-free / budget sizes, raw codec. |
| Preferences → **Time Sync** | Clock source, NTP server, manual offset, estimated-error floor. |
| Preferences → **Extensions** | SAPIENT and DDS configuration. |

Precedence is **command line > environment variable > preference > built-in default**. Every flag in the tables above overrides the corresponding preference for that run only; nothing is written back.

Interface settings (UI scale, zoom-fit, panel refresh cadence) are GUI-only — they have no headless meaning.

**Reproducible runs.** Pass `--no-preferences` to ignore the store completely and start from built-in defaults. Use it in CI, where inheriting whatever a developer last configured would make runs non-reproducible. Flags still apply, so a CI job can pin exactly the values it wants:

```bash
./speculor_cli my_pipeline.speculor ./plugins \
    --no-preferences --pool-frames=32 --run-seconds=60
```

> The store is per-machine, not per-project. Values that belong to a pipeline — node parameters, stream ports, layouts — live in the `.speculor` file and travel with it.

## Streaming

Stream config (port / JPEG quality / max FPS / output resolution / enabled) lives **per layout** inside the `.speculor` project file. Configure each layout's stream from the GUI (right-click the layout selector in the Visualization view → **Stream settings…**, plus the **Streaming enabled** toggle), save the project, then run `speculor_cli` against it — the same servers come up without the GUI.

Multiple layouts can stream simultaneously, each on its own port. If a port is already in use the streamer for that layout logs the error and the engine continues running the rest of the project.

Headless rendering uses Qt's **offscreen QPA platform** — no display server, no window manager, so streaming works over SSH and in containers. The release archive ships everything the offscreen platform needs alongside the binary; no extra packages are required.

## Exit codes

| Code | Meaning |
|------|---------|
| `0`  | Clean shutdown via signal. |
| `1`  | Project file missing, failed to parse, or the engine could not build / start the pipeline. |
| `2`  | Licence check failed — `speculor_cli` needs a tier above Community. See [licensing.md](licensing.md#cli-usage). |

## Logs

The CLI writes to `<exe_dir>/logs/speculor_cli.log`, in the format:

```
YYYY-MM-DD HH:MM:SS.mmm [LEVEL] source: message
```

`LEVEL` is one of `DEBUG`, `INFO`, `WARN`, `ERROR`. Plugin output (via `SPC_LOG_*` macros) flows through the same sink. The startup banner printed to stderr names the exact file.

`speculor_cli.log` is always the current run. When a run starts and finds a log left from an earlier day, that file is renamed to a dated archive (`speculor_cli-2026-08-23.log`) and archives older than 7 days are deleted; the live file is also archived once it passes 256 MB. Repeated messages are rate-limited. See [troubleshooting.md → Where do logs go?](troubleshooting.md#where-do-logs-go) for the full policy.

By default stdout and stderr are quiet — only a single banner is printed to stderr at startup so interactive users know the process is alive. Pass `--log-stderr` to mirror every log entry to stderr (useful when running under a process supervisor that captures stderr, or for live debugging). Stdout is never written to, so `speculor_cli ... > some.pipe` stays clean for piping into other tools.

## Environment variables

| Variable | Purpose |
|----------|---------|
| `SPC_GPU_DEVICE_INDEX` | Pick the Nth Vulkan adapter (0-based) returned by `vkEnumeratePhysicalDevices`. Overrides the `discrete > integrated > first` heuristic. The CLI already applies the device chosen in Preferences → GPU (see [Configuration](#configuration)); use this to override it for one run, or on a machine that was never configured through the GUI. |
| `VK_LOADER_LAYERS_DISABLE` | Comma-separated list of Vulkan loader layers to disable before `vkCreateInstance`. Most often used to drop `VK_LAYER_AMD_switchable_graphics` on hybrid laptops — see [troubleshooting.md → Vulkan](troubleshooting.md#vulkan-gpu-plugins-fall-back-to-cpu). |
| `SPC_VULKAN_VALIDATION` | Set to `1` to enable the `VK_LAYER_KHRONOS_validation` layer (requires the Vulkan SDK installed locally). Off by default. |

The GUI's `SPC_EXIT_TIMEOUT_MS` (forced-exit grace period) does **not** apply to the CLI — `speculor_cli` exits cleanly via the signal handler and doesn't arm the GUI's exit watchdog.

### Recording storage budgets

Both frontends read these from **Preferences → Recording** — configure them once in the GUI and `speculor_cli` on the same machine uses the same values. See [recording.md](recording.md).

| Setting | Default | Purpose |
|---------|---------|---------|
| Video part size | `1024` MB | Recordings segment (keyframe-aligned) at this size. |
| Data part size | `512` MB | Structured-traffic part size. |
| Memory spool | `64` MB | Recorder memory. Overflow is recorded as a gap rather than stalling the pipeline. |
| Minimum free disk | `2048` MB | Stop recording when free disk falls below this. |
| Session budget | unlimited | Total on-disk budget for a session. |
| Data compression | `zstd` | Codec for `traffic.mcap`. A throughput choice before a size one — see [recording.md](recording.md#when-the-recorder-cannot-keep-up). |

These were previously settable through `SPC_RECORD_*` environment variables. They no longer are: a recording knob reachable only through the environment is invisible to the person doing the recording, and these had no UI trace at all.

## Interoperability

The DDS and SAPIENT extensions run headless too, reading the same machine-level configuration the GUI writes:

| Option / variable | Purpose |
|-------------------|---------|
| `--dds-config=<ini>` | Override the DDS extension's configuration with an ini file. See [dds.md](dds.md#configuration). |
| `SPC_SAPIENT_ROLE`, `SPC_SAPIENT_HOST`, `SPC_SAPIENT_PORT`, `SPC_SAPIENT_PARENT_HOST`, `SPC_SAPIENT_PARENT_PORT` | SAPIENT role and endpoints. See [sapient.md](sapient.md#configuration). |

Both are disabled automatically during `--replay`, so a replayed incident never re-publishes to live peers.

## GPU acceleration

The CLI supports the same Vulkan-accelerated plugins the GUI does; plugins flagged `gpu_enabled` route through it. When Vulkan isn't available the CLI silently falls back to CPU, same as the GUI. Use `--pipeline-depth=<N>` to set how many frames a coalesced GPU subgraph keeps in flight (default `2`, range `1`–`3`).

## Limitations

- **Preview/Data Inspector/Profiler panels** — GUI-only. The CLI doesn't bring up the bottom-dock panels; their subscriptions don't run.
- **No interactive controls** — control gadgets (button, slider, combobox, etc.) and parameter widgets are GUI-only. The CLI renders display gadgets headlessly for streaming, but anything that would need user input at runtime needs to be baked into the project's stored parameter values.
- **Numeric / TextInput / PluginConfig gadgets are skipped headlessly** — those gadget types pull `app/`-only widget helpers and aren't compiled into the streaming runtime. They simply don't appear in CLI-rendered streams; the rest of the layout still streams. Use display alternatives (Label / Gauge / LED) where the value needs to be visible.
- **No mid-run parameter editing** — the CLI doesn't expose a remote-control interface. Parameter changes happen only via what's saved in the project.
- **No pipeline restart on disable change** — supported in the GUI's auto-rebuild path; not wired in the CLI.
