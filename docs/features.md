# Application features

UI reference for `speculor_app`. For the engine internals that back these features, see [architecture.md](architecture.md) → [engine-internals.md](engine-internals.md). For first-pipeline orientation, see [getting-started.md](getting-started.md).

## Two views

The main window has two top-level views, switched via the toolbar or `Ctrl+1` / `Ctrl+2`:

- **Pipeline Configuration** — node-graph editor (built on QtNodes). Drag plugins from the Modules dock onto the canvas, connect ports, edit parameters in Node Properties.
- **Visualization** — gadget canvas (free-form layout of video, plot, indicator, control, and utility widgets). Multiple named layouts per project; presentation mode (F11) hides handles.

A floating **Mask Editor** and **Camera Azimuth Finder** are launched from the Tools menu and are not view tabs.

## Pipeline editor

- **Visual node graph** — drag-and-drop construction with QtNodes. Right-click the canvas to add nodes from a categorised submenu.
- **Multi-instance plugins** — same plugin, multiple nodes, independent state.
- **Auto-generated parameter UI** — every plugin parameter type (`int`, `float`, `float64`, `decimal`, `bool`, `string`, `enum`, `color`, `list`, `file`) renders automatically. Parameters can be marked `STOP_ONLY` (greyed out while the pipeline runs) or hidden behind a group.
- **Parameter presets** — save and recall named parameter configurations per plugin, persisted in the `.speculor` project file.
- **Schema-described data system** — each port carries a typed schema (`SpcPortSchema`). The engine validates compatibility when ports are connected; mismatches surface as visual cues on the connection.
- **Connection waypoints** — double-click a connection to add a draggable bend point; double-click a handle to remove it. Waypoints persist in the project file with full undo/redo, and rendering uses piecewise cubic-bezier curves through them.
- **Node customisation** — per-node colour, text colour, display name, disable / bypass flag, and an opt-in gear-overlay button (corner configurable per node).
- **Plugin browser** — categorised tree with live search, drag-and-drop to canvas, right-click context menu, and three independently collapsible accordion sections: **Modules**, **Recipes**, **Virtual Plugins**.

### Edit operations (view-aware)

| Shortcut | Action | Notes |
|----------|--------|-------|
| `Ctrl+Z` / `Ctrl+Y` | Undo / Redo | Per-view undo stack. |
| `Ctrl+X` / `Ctrl+C` / `Ctrl+V` | Cut / Copy / Paste | Pipeline view: nodes. Visualization view: gadgets (MIME `application/speculor-gadget`). |
| `Ctrl+D` | Duplicate selection | |
| `Del` | Delete selection | |
| `Ctrl+G` | Group selected nodes | Pipeline view only. |

Edit menu state tracks selection, clipboard contents, and the active view's undo stack. While the pipeline is running, mutating actions in the Pipeline view are force-disabled — the engine runs against a snapshot and in-place edits would desync it. Copy stays enabled.

### Node groups

Visual grouping with lock / move / delete / copy / paste support:

- **Create**: select 2+ nodes → right-click → "Group Selection" (`Ctrl+G`) → enter name.
- **Operations**: Lock/Unlock (move as one), Rename, Ungroup, Delete Group, Save as Recipe.
- **Persistence**: stored in the project file as a `node_groups` JSON array. See [project-format.md](project-format.md).

### Pipeline recipes

Reusable, pre-wired node-group templates stored as JSON in `templates/recipes/`. Instantiate with one action; save an existing group as a recipe.

- **Browse**: Recipes section in the Modules dock; canvas right-click menu.
- **Instantiate**: double-click, drag onto canvas, or right-click canvas → Recipes submenu.
- **Built-in recipes**: shipped under `<exe_dir>/templates/recipes/`, shown muted (read-only).
- **User recipes**: writable, stored under `QStandardPaths::AppDataLocation + "/recipes/"`, deletable from the browser.
- **Validation**: missing plugins are detected and reported before instantiation.

## Visualization

- **Gadget-based** — display widgets (video, data, plots, histograms, maps, timelines, heatmaps), control widgets (button, slider, combobox, radio, text, color picker, XY pad), indicators (label, LED, gauge), and utilities (group frame, profiler overlay, log viewer).
- **Self-registering gadget architecture** — a new gadget type is added with `SPC_REGISTER_GADGET` and an entry in `app/CMakeLists.txt`; no framework files change.
- **Properties panel** — auto-generated settings UI from each gadget's `declare_properties()`.
- **Multiple layouts per project** — create, rename, delete, switch, set a default; persisted alongside the graph.
- **Canvas features** — background images, resolution presets, snap-to-grid, zoom controls.
- **Fullscreen presentation** (`F11`) — reparents the canvas into a top-level window, hides handles, keeps gadgets interactive, auto-hiding overlay with close + layout-switcher.
- **MJPEG streaming** — each layout has its own port / quality / FPS / resolution + an `enabled` flag, persisted inside the `.speculor` project file. Multiple layouts can stream simultaneously; toggling on a layout pops up an HTTP MJPEG server while the pipeline is running. Same broadcaster works headlessly under `speculor_cli` (offscreen Qt). Render uses libjpeg-turbo; clean-canvas frames are reused via `QImage::cacheKey()` so idle dashboards don't re-encode.

## Bottom dock panels

Four tabified docks at the bottom of the main window, toggleable from **View → Pipeline**:

| Panel | Purpose |
|-------|---------|
| **Preview** | Live OpenGL video frame display with live/freeze controls and a frame info overlay (size, format, GPU vs CPU origin). |
| **Stats** | Selected node statistics (FPS, latency, frames processed/dropped, errors, health state) plus an all-nodes breakdown with cumulative pipeline latency. |
| **Data Inspector** | Tabular view of `SpcTable` / `SpcRecord` / `SpcScalar` data from any node port. |
| **Profiler** | Per-node timing visualization with click-to-dim filtering. |
| **Log** | Severity-filtered log viewer fed by both engine and plugin output. |

Visibility is gated by the engine's resource model — hidden / tabbed-behind panels don't poll the engine, so they consume zero CPU until shown again.

## Tools

- **Mask Editor** — modeless dialog for drawing polygon, rectangle, and ellipse masks on captured frames. Imports/exports SVG.
- **Zone Editor** — modeless dialog for defining detection / exclusion zones.
- **Camera Azimuth Finder** — modeless side-by-side dialog (camera frame preview + OpenStreetMap pane) for discovering a camera's real GPS / azimuth / elevation / horizontal & vertical FOV by pointing it at a known landmark. Uses Open-Meteo DEM (Copernicus ~90 m) for true-elevation alignment and horizon-dip correction. Writes the discovered values back onto the selected camera node via a parameter-alias map.
- **Pipeline Validator** — graph-level checks: unconnected blocking ports, schema mismatches, cycles, missing plugins, fan-in (two or more sources targeting the same input port), unreachable nodes, missing mandatory parameters. Available manually via Tools → Validate Pipeline, and run automatically on every Play — errors block start and pop the validator dialog; warnings/info do not block.

## Plugin config panel

Every real (non-virtual) plugin gets a dedicated **Configure…** dialog for free — no plugin-side opt-in. Two entry points open the same shared modeless `PluginConfigPanel` per node (Node Properties is itself the parameter editor, so it doesn't double as an opener):

- **Preview dock gear overlay** — a small ⚙ button anchored to the actual frame-draw rect (letter-box-aware). Visible when the node's gear-overlay corner is set to anything other than `off`. Configurable in Node Properties → Customization → Gear Overlay.
- **Video gadget gear overlay** — same glyph painted into the visualization canvas; only active in view mode.

A separate **Plugin Config Gadget** embeds the same panel directly in the Visualization canvas via `QGraphicsProxyWidget` — useful for plugins with no display output (math nodes, table emitters, sinks, control-only plugins) that have no Preview / VideoGadget surface to hang the gear icon on.

All three surfaces (Node Properties, Plugin Config Panel, Plugin Config Gadget) and any control gadget bound to the same parameter stay in lockstep — every edit goes through `MainWindow::apply_param_change`, the single write path that persists to the graph, calls `engine_->set_parameter`, and broadcasts the change to every other open editor.

## Monitoring & debugging

- **Live preview** with live/freeze controls and a per-frame info overlay (GPU vs CPU origin, format, size).
- **Stats panel** with cumulative pipeline latency calculation.
- **Profiler panel** — per-node timing visualization.
- **Log panel** — severity-filtered, fed by engine + plugin output.
- **Node graph stats overlay** — per-node FPS / latency rendered directly on the graph.
- **Plugin health watchdog** — automatic detection of hung or crashed plugins, automatic restart up to N retries, cascade-disable of downstream nodes on persistent failure. Tunable in Preferences → Reliability — see [preferences.md](preferences.md#reliability).
- **`crash_guard`** — wraps each plugin's `process()` in an SEH (Windows) / signal-handler (POSIX) trap, captures up to 32 stack frames on crash, symbolicates via DbgHelp / libunwind. Build with `--with-pdb` / `-WithPDB` to resolve to file:line — see [troubleshooting.md → crash_guard](troubleshooting.md#crash_guard-stack-traces-show-raw-addresses).

## GPU compute

- **Vulkan acceleration** — optional per-plugin GPU compute for image filters and motion analysis. Auto-detected at startup; per-plugin toggle (`gpu_enabled`).
- **GPU-resident frame passing** — frames stay on the GPU between GPU-capable plugins; no PCIe round-trip.
- **Zero-cost CPU readback** — GPU work always copies to a persistently-mapped staging buffer in the same command buffer; CPU consumers get data via `memcpy`, no extra GPU submission.
- **Automatic CPU↔GPU transfers** — the engine auto-uploads CPU frames to GPU for GPU consumers and auto-downloads for CPU consumers. Plugins don't manage transfers.
- **Mid-run `DEVICE_LOST` recovery** — the engine rebuilds a profile's `VkDevice` and restarts the affected nodes when a plugin sets `device_lost` during execution. See [engine-internals.md](engine-internals.md).
- **Preview GPU/CPU indicator** — the Preview info label shows which side the displayed frame originated on.

## Help dialogs

- **About Speculor** — name, version, copyright, icon.
- **System Information** — OS, CPU (with SIMD extensions), memory, GPU, build/compiler info. Copy-to-clipboard for bug reports.
- **Keyboard Shortcuts** — HTML-formatted reference of all shortcuts.
- **GitHub** — opens the project repository in a browser.

## Project file format

Pipelines save as JSON with the `.speculor` extension — graph, parameters, visualization layouts, connection waypoints, presets, node groups, per-node display options. See [project-format.md](project-format.md) for the schema.

## Theme

Dark UI with a Catppuccin-inspired QSS in `app/resources/styles/dark_theme.qss`. Disabled menu items render in a muted colour so the view-aware Edit menu's enabled state is visible at a glance.
