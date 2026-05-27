# `speculor_cli` — headless runner

`speculor_cli` runs a saved `.speculor` project without the Qt GUI. Useful for batch processing, server deployments, regression testing, and CI. Build it via the standard build script ([installation.md](installation.md)); the binary lands in `build/bin/speculor_cli`.

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

Example:

```bash
./build/bin/speculor_cli my_pipeline.speculor ./build/bin/plugins
```

The CLI loads plugins from the directory, builds the pipeline, starts the engine, and (unless `--no-stream`) brings up one MJPEG HTTP server per layout marked `stream.enabled` in the project. It runs until interrupted with `Ctrl+C` (`SIGINT`) or `SIGTERM`. On shutdown it stops every streamer, calls `engine.stop()`, and exits with status 0.

## Streaming

Stream config (port / JPEG quality / max FPS / output resolution / enabled) lives **per layout** inside the `.speculor` project file. Configure each layout's stream from the GUI (right-click the layout selector in the Visualization view → **Stream settings…**, plus the **Streaming enabled** toggle), save the project, then run `speculor_cli` against it — the same servers come up without the GUI.

Multiple layouts can stream simultaneously, each on its own port. If a port is already in use the streamer for that layout logs the error and the engine continues running the rest of the project.

Headless rendering uses Qt's **offscreen QPA platform** — no display server, no window manager. The CLI sets `QT_QPA_PLATFORM=offscreen` before constructing `QApplication`. The packaging scripts ship `qoffscreen.dll` (Windows) / `libqoffscreen.so` (Linux) alongside the binary; building from source gets it automatically via `windeployqt` / `linuxdeployqt`. As a result `speculor_cli` now depends on Qt6 (Core / Gui / Widgets / Network) and `libjpeg-turbo` at runtime — the install bundle is ~50 MB larger than a pre-streaming CLI build.

## Exit codes

| Code | Meaning |
|------|---------|
| `0`  | Clean shutdown via signal. |
| `1`  | Project file missing, failed to parse, or the engine could not build / start the pipeline. |

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

## GPU acceleration

The CLI supports the same Vulkan-accelerated plugins the GUI does. If the SDK was built with Vulkan, the CLI calls `spc::gpu::create_engine_bridge()` at startup; plugins flagged `gpu_enabled` route through it. When Vulkan isn't available the CLI silently falls back to CPU, same as the GUI.

## Limitations

- **Preview/Data Inspector/Profiler panels** — GUI-only. The CLI doesn't bring up the bottom-dock panels; their subscriptions don't run.
- **No interactive controls** — control gadgets (button, slider, combobox, etc.) and parameter widgets are GUI-only. The CLI renders display gadgets headlessly for streaming, but anything that would need user input at runtime needs to be baked into the project's stored parameter values.
- **Numeric / TextInput / PluginConfig gadgets are skipped headlessly** — those gadget types pull `app/`-only widget helpers and aren't compiled into the streaming runtime. They simply don't appear in CLI-rendered streams; the rest of the layout still streams. Use display alternatives (Label / Gauge / LED) where the value needs to be visible.
- **No mid-run parameter editing** — the CLI doesn't expose a remote-control interface. Parameter changes happen only via what's saved in the project.
- **No pipeline restart on disable change** — supported in the GUI's auto-rebuild path; not wired in the CLI.
