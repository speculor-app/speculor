# Troubleshooting

Common build, startup, and runtime issues. See [installation.md](installation.md) for the build guide and [README.md](../README.md) for an overview.

## Build

### `SDK repository not found at ../speculor-sdk`

The build script aborts because `speculor-sdk` is missing. Clone it as a sibling of `speculor-app/` (see [installation.md → repository layout](installation.md#repository-layout)). Other layouts are not supported.

### First build is slow

The SDK build pulls and compiles FFmpeg, OpenCV, x264, x265, Eigen, and libjpeg-turbo from source on Linux/macOS. This is a one-time cost — subsequent builds reuse the cached results in `build/sdk-build/_deps/`. On Windows the same dependencies come from vcpkg, which still pays a similar cost on first install (cached in `build/vcpkg_installed/`).

The Linux dev container caches the build inside the image, so reopening it stays fast.

### Qt not found

- **Linux**: install `qt6-base-dev` and `qt6-base-dev-tools`.
- **macOS**: after `brew install qt@6`, the build script picks Qt up automatically. To override: `export CMAKE_PREFIX_PATH=$(brew --prefix qt@6)`.
- **Windows**: ensure Qt is installed under `C:\Qt\<version>\msvc*_64` (the script auto-detects), or set `CMAKE_PREFIX_PATH` / `Qt6_DIR` to the install root.

### vcpkg errors (Windows)

- The script bootstraps `%USERPROFILE%\vcpkg` automatically when no other clone is found. To use an existing one, set `VCPKG_ROOT` first.
- Run from a **Developer Command Prompt for VS 2022** so MSVC is on the PATH.
- `-Force` re-runs `vcpkg install` from the manifest in `vcpkg.json`.

### VAAPI not found (Linux)

Install `libva-dev` (Ubuntu/Debian) or `libva-devel` (Fedora) for hardware-accelerated video encode/decode in FFmpeg. Optional — the build succeeds without it but FFmpeg won't have VAAPI support.

### SDR plugin headers not found

The WinRadio G3x / G39 and RTL-SDR plugins live in `speculor-plugins` and download their SDK headers at CMake configure time. If the download fails (no internet, unreachable mirror), the affected plugin is skipped with a status message and the rest of the build succeeds.

For RTL-SDR runtime support after build, install:

- Windows: `rtlsdr.dll` from the [RTL-SDR releases](https://github.com/osmocom/rtl-sdr/releases).
- Linux: `apt install librtlsdr0` (or distro equivalent).

The plugin loads the library at runtime; if absent, the device list shows `(no devices found)`.

### ADS-B Display airspace / airports data

The ADS-B Display plugin (in `speculor-plugins`) fetches airspace polygons and airport markers from [OpenAIP](https://www.openaip.net/) at configure time. The fetch needs a free API key.

- Sign up at [openaip.net](https://www.openaip.net/), generate a client key from account settings.
- Pass it via `OPENAIP_API_KEY=<key>` in the environment, or rely on the embedded key in `scripts/build.ps1` / `scripts/build.sh`.
- Without a key the plugin still builds — the airspace and airport layers just render empty.
- Force a refetch with `cmake -DOPENAIP_FORCE_REFETCH=ON -B build/plugins-build` and reconfigure.

## Runtime

### Vulkan: GPU plugins fall back to CPU

Speculor needs **Vulkan API 1.3+** (for Vulkan Video decode). On AMD hybrid laptops, an implicit Vulkan layer registered in the system can clamp the reported API version to 1.1 and break `VulkanContext::init()`.

**Symptoms**: log contains `Vulkan: initialization failed at startup`; GPU-capable plugins fall back to CPU; `vulkaninfo` reports `vkEnumeratePhysicalDevices failed with INCOMPLETE` and warns that an implicit layer uses an older API version than requested.

**Fix from the GUI**: **Edit → Preferences… → Graphics**. Keep "Disable AMD Switchable Graphics Vulkan layer" checked (default). Optionally pick a specific GPU adapter from the dropdown. Changes apply after restart.

**Fix from the environment** (CI, headless, CLI):

| Variable | Purpose |
|----------|---------|
| `VK_LOADER_LAYERS_DISABLE=VK_LAYER_AMD_switchable_graphics,…` | Disables the offending implicit layers before `vkCreateInstance`. Comma-separated. Requires Vulkan loader 1.3.234+ (the SDK bundles 1.4.309). |
| `SPC_GPU_DEVICE_INDEX=<n>` | Selects the Nth adapter (0-based) returned by `vkEnumeratePhysicalDevices`. Overrides the default `discrete > integrated > first` heuristic. |

QSettings values under `preferences/gpu/*` take precedence over the env fallback when launched via the UI. The env fallback is the only mechanism for the CLI runner.

### Where do logs go?

Both `speculor_app` and `speculor_cli` write a rolling log file under a `logs/` directory next to the executable:

- `build/bin/logs/speculor.log` — GUI log.
- `build/bin/logs/speculor_cli.log` — CLI log.

The GUI also surfaces live entries in the **Log** dock (View → Pipeline → Log).

### Pipeline appears hung

The engine ships a watchdog that detects hung or crashed plugins and restarts them automatically. Defaults:

- Hang timeout: 10 s for processing nodes, 30 s for source / GPU nodes.
- Max restarts: 3 (then the node is disabled and downstream nodes are cascade-disabled).
- Restart cooldown: 5 s.

Tune all of these under **Edit → Preferences… → Reliability**. See [preferences.md → reliability](preferences.md#reliability) for the QSettings keys.

A node disabled by the watchdog can be re-enabled manually from the right-click context menu in the node graph.

### `crash_guard` stack traces show raw addresses

The engine wraps each plugin's `process()` call in an SEH/`signal()` guard that catches crashes and prints up to 32 frames using DbgHelp (Windows) or libunwind (POSIX). The output format is `module!function+0xOFF [file:line]`.

By default Release builds **do not** ship debug info, so you'll see addresses only. Rebuild with `--with-pdb` (Linux/macOS) or `-WithPDB` (Windows) to emit side-file PDBs / DWARF without changing the Release binary itself:

```bash
./scripts/build.sh --with-pdb
scripts/build.ps1 -WithPDB
```

Then reproduce the crash — frames now resolve to `module!function [file:line]`. Drop the flag for shipped builds (PDBs roughly double the package size).

### Shutdown hangs past a few seconds

`speculor_app` arms an exit watchdog when the Qt event loop returns: if cleanup doesn't finish within 3 s, the process force-exits via `_Exit(0)`. The grace period can be overridden with `SPC_EXIT_TIMEOUT_MS` (clamped to `[0, 30000]`).

If the watchdog fires, `speculor.log` records `exit watchdog fired — process did not terminate cleanly`.
