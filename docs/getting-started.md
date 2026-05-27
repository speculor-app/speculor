# Getting started

A short walkthrough for building a first pipeline once Speculor is installed and running. See [installation.md](installation.md) for build instructions and [features.md](features.md) for the full UI reference.

## Launch the app

```bash
./build/bin/speculor_app           # Linux / macOS
build\bin\speculor_app.exe         # Windows
```

The main window opens in **Pipeline Configuration** view (`Ctrl+1`). Three docks frame the central node-graph canvas:

- **Modules** (left) — categorised list of every plugin that was discovered in `build/bin/plugins/`. Drag entries onto the canvas, or right-click the canvas to add nodes.
- **Node Properties** (right) — auto-generated parameter editor for the currently selected node.
- **Bottom dock area** — Preview, Data Inspector, Log, and Profiler tabs (toggle via **View → Pipeline**).

If the Modules list is empty, the plugins didn't build. Check that `speculor-plugins` was cloned alongside this repo and re-run the build.

## Build a minimal pipeline

The smallest useful pipeline is a synthetic source feeding a recorder:

1. From the Modules panel, drag **Pattern Source** (under *Generators / Video*) onto the canvas.
2. Drag **Video Recorder** (under *Output / Recording*) below it.
3. Click and drag from the Pattern Source's output port to the Video Recorder's input port. A green connection indicates the port types match.
4. Select the Video Recorder and set a target file path in Node Properties.
5. Press **F5** (or the play button on the transport bar) to start the pipeline.
6. Press **F5** again to stop. The recorded file is written when the recorder receives `stop()`.

To see the source visually, drag a Preview output: select the Pattern Source node and click the eye icon in the Preview dock toolbar to subscribe its output. Frames render in real time.

## Save your work

**File → Save Project…** (`Ctrl+S`) writes a `.speculor` JSON file containing the graph, parameters, visualization layouts, connection waypoints, and per-node display options. **File → Open Project…** (`Ctrl+O`) restores it. See [project-format.md](project-format.md) for the schema.

Speculor also auto-restores the last open project on startup (deferred until the main window is shown so the viewport is laid out correctly).

## Build a dashboard

Switch to **Visualization** view (`Ctrl+2`). The canvas is a free-form layout for **gadgets**: video displays, plots, indicators, controls (buttons, sliders, comboboxes), and utility widgets.

1. Right-click the canvas → **Add Gadget** → **Video**. Bind it to a node + output port via its properties panel.
2. Add a **Slider** gadget. In its properties, bind it to a parameter on any node — the slider drives the parameter live.
3. Drag, resize, and re-layer gadgets in edit mode. Switch to view mode (top-right toggle) to interact without accidentally moving them.
4. Manage multiple layouts (named tabs) from the Visualization toolbar — useful for swapping between an "operator" view and a "diagnostic" view of the same pipeline.

**F11** enters fullscreen presentation mode (gadget handles hidden, gadgets remain interactive). The Visualization view can also broadcast over MJPEG (port / quality / FPS / resolution configurable from the toolbar settings dialog) so a browser or other consumer can pick up the canvas as a live video stream.

## Edit operations

Both views share a view-aware Edit menu — undo / redo / cut / copy / paste / duplicate / delete dispatch to whichever view is active and stay disabled while they don't apply.

| Shortcut | Action |
|----------|--------|
| `Ctrl+Z` / `Ctrl+Y` | Undo / Redo (per-view stack) |
| `Ctrl+X` / `Ctrl+C` / `Ctrl+V` | Cut / Copy / Paste |
| `Ctrl+D` | Duplicate selection |
| `Del` | Delete selection |
| `Ctrl+G` | Group selected nodes (Pipeline view) |
| `F5` | Toggle pipeline play / stop |
| `Ctrl+1` / `Ctrl+2` | Switch between Pipeline and Visualization views |
| `F11` | Fullscreen Visualization view |
| `Ctrl+,` | Open Preferences |

While the pipeline is running, mutating actions (cut/paste/duplicate/delete and undo/redo) are disabled in the Pipeline view — the engine runs against a snapshot of the graph and in-place edits would desync it. Stop the pipeline to edit it.

## Where to next

- [features.md](features.md) — UI reference: panels, tools, recipes, node groups, plugin config panel, azimuth finder.
- [preferences.md](preferences.md) — every QSettings key the Preferences dialog touches.
- [cli.md](cli.md) — running pipelines headless from a `.speculor` file.
- [project-format.md](project-format.md) — `.speculor` JSON schema.
- [plugin-development.md](plugin-development.md) — writing your own plugins against the SDK.
- [architecture.md](architecture.md) → [engine-internals.md](engine-internals.md) — how the engine schedules graph execution.
