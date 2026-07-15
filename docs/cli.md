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
| `--pipeline-depth=<N>` | `2` | GPU coalesced-subgraph frames kept in flight; clamped to `[1, 3]`. Headless equivalent of Preferences → Performance "GPU pipeline depth"; the CLI doesn't read the GUI's settings store, so this flag is the only headless control. |
| `--run-seconds=<N>` | off | Auto-stop cleanly after `N` seconds. |
| `--license-file=<path>` | (cached file) | Use a licence file from an explicit path instead of the cached one. See [licensing.md](licensing.md#cli-usage). |
| `--activate=<key>` | — | Activate this machine against a licence key, write the signed licence file, and exit. Optionally with `--machine-name=<name>`. |

### Recording & replay

Experimental, and gated to a **Personal** licence or higher in their entirety — see [recording.md](recording.md).

| Option | Default | Description |
|--------|---------|-------------|
| `--record[=<dir>]` | off | Record the whole pipeline to a session folder under `<dir>/<timestamp>/` (default `recordings/session`). Captures compressed video per source node, plus tables/scalars/records/control/signals/parameter writes/events/health as structured traffic. Long runs segment automatically; if the disk can't keep up, the loss window is recorded as a gap rather than stalling the pipeline. |
| `--record-frames=all` | off | With `--record`: capture **every** node's frame outputs (intermediate stages), not just sources — the headless equivalent of Preferences → Recording → Capture mode = **Full**. |
| `--record-standby=<sec>` | off | Pre-record standby (implies `--record`): the last `<sec>` seconds ring in RAM and nothing hits disk until the incident trigger fires — then past + live are recorded. An untriggered session is discarded on stop. |
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

The CLI writes a rolling log file at `<exe_dir>/logs/speculor_cli.log` in the format:

```
[HH:MM:SS.mmm] [LEVEL] [source] message
```

`LEVEL` is one of `DEBUG`, `INFO`, `WARN`, `ERROR`. Plugin output (via `SPC_LOG_*` macros) flows through the same sink.

By default stdout and stderr are quiet — only a single banner is printed to stderr at startup so interactive users know the process is alive. Pass `--log-stderr` to mirror every log entry to stderr (useful when running under a process supervisor that captures stderr, or for live debugging). Stdout is never written to, so `speculor_cli ... > some.pipe` stays clean for piping into other tools.

## Environment variables

| Variable | Purpose |
|----------|---------|
| `SPC_GPU_DEVICE_INDEX` | Pick the Nth Vulkan adapter (0-based) returned by `vkEnumeratePhysicalDevices`. Overrides the `discrete > integrated > first` heuristic. The GUI's per-profile device selection is a `QSettings` value and only applies when launched from the GUI; the CLI uses this env var instead. |
| `VK_LOADER_LAYERS_DISABLE` | Comma-separated list of Vulkan loader layers to disable before `vkCreateInstance`. Most often used to drop `VK_LAYER_AMD_switchable_graphics` on hybrid laptops — see [troubleshooting.md → Vulkan](troubleshooting.md#vulkan-gpu-plugins-fall-back-to-cpu). |
| `SPC_VULKAN_VALIDATION` | Set to `1` to enable the `VK_LAYER_KHRONOS_validation` layer (requires the Vulkan SDK installed locally). Off by default. |

The GUI's `SPC_EXIT_TIMEOUT_MS` (forced-exit grace period) does **not** apply to the CLI — `speculor_cli` exits cleanly via the signal handler and doesn't arm the GUI's exit watchdog.

### Recording storage budgets

The GUI reads these from Preferences → Recording; headless, they are environment variables. See [recording.md](recording.md).

| Variable | Default | Purpose |
|----------|---------|---------|
| `SPC_RECORD_PART_MB` | `1024` | Video part size — recordings segment (keyframe-aligned) at this size. |
| `SPC_RECORD_MCAP_PART_MB` | `512` | Structured-traffic part size. |
| `SPC_RECORD_SPOOL_MB` | `64` | Recorder memory spool. Overflow is recorded as a gap rather than stalling the pipeline. |
| `SPC_RECORD_MIN_FREE_MB` | — | Stop recording when free disk falls below this. |
| `SPC_RECORD_BUDGET_MB` | — | Total on-disk budget for a session. |

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
