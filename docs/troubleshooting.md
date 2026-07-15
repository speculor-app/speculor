# Troubleshooting

Common startup and runtime issues. For a first-pipeline walkthrough see [getting-started.md](getting-started.md).

## Runtime

### The Modules panel is empty

The app discovers plugins in the `plugins/` directory next to the app binary. An empty Modules list means no plugin bundles are installed.

Extract the `speculor-bundle-*.zip` / `.tar.xz` files from the same release at the install root — they drop plugins under `plugins/<bundle-name>/` and any vendor libraries beside them, and the app picks both up automatically on the next launch. See [plugins.md](plugins.md) for which bundle carries which plugin.

If only *some* plugins are missing, check the tier on **Help → License…**: plugins declare a minimum licence tier and any plugin above the active tier is skipped during the scan and never appears in the browser. See [licensing.md](licensing.md).

### A plugin disappeared after upgrading the app

Plugin bundles are versioned with the release they ship in. Upgrading the app without re-extracting the bundles leaves older plugin binaries under a newer app. Re-download and extract the bundles from the same release as the app.

### RTL-SDR: device list shows no devices

The RTL-SDR plugin loads the `rtlsdr` library at runtime. If it's missing, the device list shows `(no devices found)`. Install it:

- **Windows** — drop `rtlsdr.dll` from the [RTL-SDR releases](https://github.com/osmocom/rtl-sdr/releases) next to the app.
- **Linux** — `apt install librtlsdr0` (or your distro's equivalent).

### ADS-B Display shows no airspace or airports

The airspace polygon and airport marker layers are optional reference data. Without them the plugin runs normally — aircraft, trails, filters, MLAT indicators and tooltips all work — and those two layers simply render empty.

### Vulkan: GPU plugins fall back to CPU

Speculor needs **Vulkan API 1.3+** (for Vulkan Video decode). On AMD hybrid laptops, an implicit Vulkan layer registered in the system can clamp the reported API version to 1.1 and break Vulkan initialization.

**Symptoms**: log contains `Vulkan: initialization failed at startup`; GPU-capable plugins fall back to CPU; `vulkaninfo` reports `vkEnumeratePhysicalDevices failed with INCOMPLETE` and warns that an implicit layer uses an older API version than requested.

**Fix from the GUI**: **Edit → Preferences… → GPU**. Keep "Disable AMD Switchable Graphics Vulkan layer" checked (default). Optionally pick a specific GPU adapter from the dropdown. Changes apply after restart.

**Fix from the environment** (CI, headless, CLI):

| Variable | Purpose |
|----------|---------|
| `VK_LOADER_LAYERS_DISABLE=VK_LAYER_AMD_switchable_graphics,…` | Disables the offending implicit layers before `vkCreateInstance`. Comma-separated. Requires Vulkan loader 1.3.234+ (the release bundles 1.4.309). |
| `SPC_GPU_DEVICE_INDEX=<n>` | Selects the Nth adapter (0-based). Overrides the default `discrete > integrated > first` heuristic. |

The GUI's settings under `preferences/gpu/*` take precedence over the env fallback when launched via the UI. The env fallback is the only mechanism for the CLI runner — see [preferences.md](preferences.md#environment-overrides).

### `Illegal instruction (core dumped)` on Linux

The release binaries assume a portable x86-64 baseline, but a CPU older than that baseline traps on the first unsupported instruction — a raw `SIGILL`, usually with no app log because the fault can fire during dynamic linking before logging is up.

**Confirm**: on the affected box, `grep -m1 -o -E 'avx2|avx|sse4_2' /proc/cpuinfo | sort -u` shows which SIMD levels it actually has. Report the output with your issue — a plugin `.so` can trap just as easily as the main binary.

### Where do logs go?

Both `speculor_app` and `speculor_cli` write a rolling log file under a `logs/` directory next to the executable:

- `logs/speculor.log` — GUI log.
- `logs/speculor_cli.log` — CLI log.

The GUI also surfaces live entries in the **Log** dock (View → Pipeline → Log). The CLI is quiet on stdout/stderr by default; pass `--log-stderr` to mirror entries to stderr — see [cli.md](cli.md#logs).

### Pipeline appears hung

The engine ships a watchdog that detects hung or crashed plugins and restarts them automatically. Defaults:

- Hang timeout: 10 s for processing nodes, 30 s for source / GPU nodes.
- Max restarts: 3 (then the node is disabled and downstream nodes are cascade-disabled).
- Restart cooldown: 5 s.

Tune all of these under **Edit → Preferences… → Reliability**. See [preferences.md → reliability](preferences.md#reliability--node-watchdog) for the keys.

A node disabled by the watchdog can be re-enabled manually from the right-click context menu in the node graph.

### `crash_guard` stack traces show raw addresses

The engine wraps each plugin's `process()` call in an SEH (Windows) / signal-handler (POSIX) guard that catches crashes and prints up to 32 frames. The output format is `module!function+0xOFF [file:line]`.

Release builds ship without debug info, so frames show raw addresses. Capture the full log — it records the module and offset for every frame — and include it with your report; the frames are resolved to `module!function [file:line]` against the matching release build's symbols.

### Shutdown hangs past a few seconds

`speculor_app` arms an exit watchdog when the Qt event loop returns: if cleanup doesn't finish within 3 s, the process force-exits. The grace period can be overridden with `SPC_EXIT_TIMEOUT_MS` (clamped to `[0, 30000]`).

If the watchdog fires, `speculor.log` records `exit watchdog fired — process did not terminate cleanly`.

### Recording controls are greyed out

Recording & replay are **experimental** and require a **Personal** licence or higher in their entirety — the transport Record button, the Recordings view, Tools → Session Replay, the Preferences → Recording page, and the CLI's `--record` / `--replay` / `--replay-dump` / `--export` modes. Check the tier on **Help → License…**. See [recording.md](recording.md) and [licensing.md](licensing.md).

### DDS: instances don't discover each other

Instances must share the same `domain` (and `partition`, if set). Multicast-blocked networks need `initial_peers` or a `discovery_server`. DDS is **Personal**-tier gated — below that the settings page and per-port exposure are disabled. See [dds.md → troubleshooting](dds.md#troubleshooting).

### SAPIENT connection drops or no detections (Team tier)

Open the **SAPIENT Console** (Tools → Extensions) and the **Log** dock for the connection state. The most common causes are firewalling, a mutual-TLS CA mismatch, a pipeline with no `geolocation` stage after the tracker, and running below the Team tier. See [sapient.md → troubleshooting](sapient.md#troubleshooting).
