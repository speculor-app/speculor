# Speculor

**A real-time sensor and signal-processing engine.** Speculor connects any device that produces data — cameras, microphones, SDR radios, GPS receivers, ADS-B feeds, weather stations, smart-home devices, databases — and routes it through low-latency processing pipelines assembled from dynamically loaded plugin nodes. The engine runs each node on its own thread, moves typed packets between them with lock-free transports, and keeps GPU work GPU-resident across multi-stage Vulkan pipelines.

The engine is headless. Frontends ride on top of it: **`speculor_app`** — the Qt6 pipeline builder you'll see in the screenshots — is one of them; **`speculor_cli`** — the headless runner for servers and CI — is another; more are on the roadmap.

![Platform](https://img.shields.io/badge/Platform-Linux%20%7C%20Windows%20%7C%20macOS-lightgrey)
![License](https://img.shields.io/badge/License-Proprietary-blue)

## What it is

Speculor is a modular data-processing engine. Every block in a pipeline is a **plugin** — a dynamically loaded `.dll` / `.so` / `.dylib` that crosses a stable C ABI. The engine is the schedule, transport, and lifecycle layer; plugins do the actual work. Frontends (the Qt6 pipeline builder, the headless CLI runner, and future ones) embed the engine and feed it a graph. The engine multithreads the nodes, and data moves between them through a transport picked for the data type:

- **Video frames** travel through reference-counted shared pools — no allocations on the hot path, zero-copy when the consumer doesn't mutate.
- **Tables** (detections, tracks, I/Q batches, sensor records) carry a typed schema so the engine validates port compatibility before the pipeline starts and routes whole batches without per-element overhead.
- **Signals** (audio, SDR samples, high-rate sensor streams) bypass queues entirely — the engine direct-calls the consumer's `on_signal()` on the producer's thread for sample-accurate, zero-copy transport, with a lock-free SPSC ring buffer available when async hand-off is needed.
- **GPU frames** stay GPU-resident across chains of GPU-capable plugins. The engine fuses every GPU node on the same Vulkan profile into a single `vkQueueSubmit2` per frame and injects inter-plugin barriers from the graph edges — N driver round-trips collapse to 1.
- **Control & parameter feedback** flows back through a dedicated connection class, so a downstream node can drive an upstream node's parameters live (e.g. a tracker steering a PTZ camera, or a classifier retuning a radio).

The same `.speculor` project runs under any frontend — graph, parameters, dashboard layouts, and MJPEG streams all carry over. Build and tune interactively in `speculor_app`; deploy on a server with `speculor_cli`.

## Why this design

- **Heterogeneous sensors, one graph.** An ADS-B feed at 1 Hz, a camera at 30 Hz, and an SDR at 2 MHz can converge on the same fusion node. Each connection picks the transport that fits its data, so rates don't fight each other.
- **Predictable latency.** Per-node threads, lock-free SPSC queues, and pre-allocated frame / table pools mean a frame doesn't wait on a contended lock or a heap allocation. The graph *is* the schedule.
- **GPU when it pays off.** Vulkan compute is opt-in per plugin. When multiple GPU nodes share a profile, the engine coalesces their submits so the win compounds across the chain instead of being eaten by per-plugin driver overhead. Mid-run `DEVICE_LOST` recovers automatically.
- **Hot-pluggable extensibility.** Drop a new `.dll` / `.so` into the plugin folder and it appears in the Modules panel — no host recompile, no rebuild, no restart of unrelated nodes.
- **Crash-resistant.** Every `process()` and `record_gpu` runs under an SEH (Windows) / signal-handler (POSIX) guard. A misbehaving plugin gets stack-traced, restarted, and after N retries cascade-disables itself — the rest of the pipeline keeps running.

## What ships in the plugin catalog

| Domain                  | Examples                                                                                              |
|-------------------------|-------------------------------------------------------------------------------------------------------|
| **Video sources**       | FFmpeg / RTSP cameras, V4L2, QHY astronomical cameras, image folders, pattern generators              |
| **Audio**               | System input/output, mixer, frequency filtering, noise reduction, oscilloscope, spectrum analyzer     |
| **SDR**                 | RTL-SDR, WinRadio G3x / G39, I/Q demod, radio tuner, direction finder, signal classifier              |
| **ADS-B**               | dump1090 source, SBS source, decoder, MLAT client, filter, statistics, aircraft enricher, replay      |
| **GPS & telemetry**     | NMEA sources, static GPS, system stats, weather feeds, Home Assistant bridge                          |
| **Motion analysis**     | MOG2 / SubSense / ViBe / WMV background subtraction, optical flow, SORT tracker                       |
| **Image filters**       | GPU-accelerated colour conversion, scaling, masking, blending                                         |
| **Renderers & overlays**| HUD overlays, weather display, stats monitor                                                          |
| **Databases**           | PostgreSQL / MongoDB / SQLite — lookup, query, write                                                  |
| **Automation**          | Logic gates, control nodes, alert triggers                                                            |
| **Scripting**           | Embedded Python script node                                                                           |
| **Output**              | Video recording, MJPEG / H.264 HTTP streaming                                                         |

## At a glance

- Visual node-graph editor with port-type validation, undo/redo, multi-instance plugins, node groups, reusable recipes, and per-node customisation.
- Auto-generated parameter UI for every plugin.
- Free-form visualization canvas with display, control, indicator, and utility **gadgets** — video, plots, histograms, maps, timelines, heatmaps, gauges, LEDs, sliders, XY pads. Multiple named layouts per project. F11 presentation mode.
- Built-in MJPEG broadcaster per layout — multiple layouts stream simultaneously on different ports.
- Live monitoring: preview dock, profiler, data inspector, severity-filtered log, per-node FPS / latency on the graph itself.
- Plugin health watchdog: hang detection, automatic restart with retry budget, cascade-disable on persistent failure, mid-run Vulkan device recovery.
- Tooling: Mask Editor, Zone Editor, Camera Azimuth Finder, Pipeline Validator.

## Install

Releases for Windows, Linux, and macOS are on the [Releases page](https://github.com/speculor-app/speculor/releases). Each release ships:

- **`speculor-app-<version>-<platform>.{zip,tar.xz}`** — the app + UI + Qt + SDK + shared runtime DLLs. Extract anywhere and run `speculor_app`.
- **`speculor-bundle-<name>-<version>-<platform>.zip`** (one per plugin domain) — extract at the same install root to add capability bundles (adsb, sdr, audio, video-sources, image-filters, motion-analysis, detection, output, automation, scripting, …). The host app discovers them automatically.

### Windows

Extract the `.zip`, double-click `speculor_app.exe`. SmartScreen may warn on first launch (the release isn't code-signed yet); click **More info → Run anyway**.

### Linux

```bash
tar -xJf speculor-app-<version>-Linux-x64.tar.xz
cd speculor-app
./speculor_app
```

### macOS

Releases are unsigned (signing + notarization is on the roadmap). Extract the `.tar.xz`, then clear the Gatekeeper quarantine flag before launching:

```bash
tar -xJf speculor-app-<version>-macOS-arm64.tar.xz
cd speculor-app
xattr -dr com.apple.quarantine .
./speculor_app
```

## License key

Speculor uses license-key activation. On first launch you'll be prompted for a key — paste it into the activation dialog and the app authenticates against the licensing service. See [docs/licensing.md](docs/licensing.md) for trial keys and offline activation.

## Basic usage

Once the GUI is open in **Pipeline Configuration** view (`Ctrl+1`):

1. Drag a plugin from the **Modules** panel onto the canvas (or right-click the canvas).
2. Drag between ports to connect nodes — port type validation is live.
3. Select a node to edit its parameters in the **Node Properties** panel.
4. Press **F5** to start / stop the pipeline.
5. Switch to **Visualization** view (`Ctrl+2`) to assemble a dashboard.
6. **File → Save Project** writes a `.speculor` JSON file.

Replay headless from a saved project:

```bash
./speculor_cli my_pipeline.speculor ./plugins
```

See [docs/getting-started.md](docs/getting-started.md) for a longer walkthrough.

## Documentation

| Topic | Document |
|-------|----------|
| First-pipeline tutorial | [docs/getting-started.md](docs/getting-started.md) |
| Application UI reference | [docs/features.md](docs/features.md) |
| Headless CLI runner | [docs/cli.md](docs/cli.md) |
| Preferences | [docs/preferences.md](docs/preferences.md) |
| `.speculor` project file format | [docs/project-format.md](docs/project-format.md) |
| License-key activation | [docs/licensing.md](docs/licensing.md) |
| Common errors / Vulkan / crash symbolication | [docs/troubleshooting.md](docs/troubleshooting.md) |

## Support

Bug reports and feature requests: open an issue on this repo's [issue tracker](https://github.com/speculor-app/speculor/issues).

## License

Speculor is proprietary software. Copyright © 2026 Fabio Barros. All rights reserved. See [`LICENSE`](LICENSE) for the full End User License Agreement.

Speculor incorporates third-party software under their own license terms — attribution and license texts are reproduced in [`THIRD_PARTY_NOTICES.md`](THIRD_PARTY_NOTICES.md). All bundled components are LGPL or permissive; Speculor's FFmpeg build is LGPL-only (no `--enable-gpl`, no `libx264`/`libx265`).
