# Application features

UI reference for `speculor_app`. For first-pipeline orientation, see [getting-started.md](getting-started.md). For what each plugin does, see [plugins.md](plugins.md).

## Three views

The main window has three top-level views, switched via the toolbar or `Ctrl+1` / `Ctrl+2` / `Ctrl+3`:

- **Pipeline Configuration** — node-graph editor (built on QtNodes). Drag plugins from the Modules dock onto the canvas, connect ports, edit parameters in Node Properties.
- **Visualization** — gadget canvas (free-form layout of video, plot, indicator, control, and utility widgets). Multiple named layouts per project; presentation mode (F11) hides handles.
- **Recordings** (`Ctrl+3`) — browser for recorded sessions: cards with a poster cover, project name, and hover event thumbnails. From here you open a session into Session Replay or run the pipeline from a recording (reinjection replay). Personal tier. See [recording.md](recording.md).

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
- **DDS exposure** (Personal tier) — right-click a node → **Expose via DDS** to publish output ports onto a Fast DDS domain for other Speculors. Exposed nodes carry a blue **DDS** badge, and each port's encoding + QoS is set from **Configure…** in the Node Properties → DDS group. See [dds.md](dds.md).

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
- **User recipes**: writable, stored under `<install_dir>/user/recipes/` (falls back to the per-user data location if the install dir is read-only), deletable from the browser.
- **Validation**: missing plugins are detected and reported before instantiation.

## Visualization

- **Gadget-based** — display widgets (video, data, plots, histograms, maps, timelines, heatmaps), control widgets (button, slider, combobox, radio, text, color picker, XY pad), indicators (label, LED, gauge), and utilities (group frame, profiler overlay, log viewer).
- **Self-registering gadget architecture** — new gadget types are added without touching framework files; each gadget registers itself and exposes its own settings.
- **Properties panel** — auto-generated settings UI from each gadget's declared properties.
- **Multiple layouts per project** — create, rename, delete, switch, set a default; persisted alongside the graph.
- **Canvas features** — background images, resolution presets, snap-to-grid, zoom controls (Ctrl+wheel zooms to the cursor; middle-mouse drag pans; scroll bars appear when zoomed in).
- **Playback controls** — **Pause/Resume** freezes the canvas display while the pipeline keeps running underneath (mirrors the Preview pane's Live/Freeze); **Screenshot** captures the dashboard at full resolution, copies it to the clipboard, and offers to save it to a PNG/JPEG file.
- **Fullscreen presentation** (`F11`) — reparents the canvas into a top-level window, hides handles, keeps gadgets interactive, auto-hiding overlay with close + layout-switcher.
- **MJPEG streaming** — each layout has its own port / quality / FPS / resolution + an `enabled` flag, persisted inside the `.speculor` project file. Multiple layouts can stream simultaneously; toggling on a layout pops up an HTTP MJPEG server while the pipeline is running. Same broadcaster works headlessly under `speculor_cli` (offscreen Qt). Render uses libjpeg-turbo; clean-canvas frames are reused via `QImage::cacheKey()` so idle dashboards don't re-encode.

## Bottom dock panels

Four tabified docks at the bottom of the main window, toggleable from **View → Pipeline**:

| Panel | Purpose |
|-------|---------|
| **Preview** | Live OpenGL video frame display with live/freeze controls and a frame info overlay (size, format, GPU vs CPU origin). |
| **Stats** | Selected node statistics (FPS, latency, frames processed/dropped, errors, health state) plus an all-nodes breakdown with cumulative pipeline latency. |
| **Data Inspector** | Live view of any non-frame node-port output: scalars, tables, records (JSON), plus signals (audio/SDR samples — waveform sparkline + metadata + sample table), bitstream packets (codec / flags / resolution / measured bitrate + hex payload peek), and control messages (message type + parameter entries). |
| **Profiler** | Per-node timing visualization with click-to-dim filtering. |
| **Log** | Severity-filtered log viewer fed by both engine and plugin output. |

Visibility is gated by the engine's resource model — hidden / tabbed-behind panels don't poll the engine, so they consume zero CPU until shown again.

The side docks — **Modules** (left) and **Node Properties** (right) — are likewise toggleable from **View → Pipeline**, so a panel closed via its title-bar ✕ can always be reopened from the menu.

## Tools

- **Mask Editor** — modeless dialog for drawing polygon, rectangle, and ellipse masks on captured frames. Imports/exports SVG.
- **Zone Editor** — modeless dialog for defining detection / exclusion zones.
- **Camera Azimuth Finder** — modeless side-by-side dialog (camera frame preview + OpenStreetMap pane) for discovering a camera's real GPS / azimuth / elevation / horizontal & vertical FOV by pointing it at a known landmark. Uses Open-Meteo DEM (Copernicus ~90 m) for true-elevation alignment and horizon-dip correction. Writes the discovered values back onto the selected camera node via a parameter-alias map.
- **Pipeline Validator** — graph-level checks: unconnected blocking ports, schema mismatches, cycles, missing plugins, fan-in (two or more sources targeting the same input port), unreachable nodes, missing mandatory parameters. Available manually via Tools → Validate Pipeline, and run automatically on every Play — errors block start and pop the validator dialog; warnings/info do not block.
- **SAPIENT Console** (Tools → Extensions → SAPIENT Console) — live status of the **SAPIENT** interoperability extension (role, endpoint, connection state, detections sent/received, tasks) plus a reachability probe. SAPIENT (NATO BSI Flex 335 v2.0) is configured under Settings → Extensions and is **Team-tier** gated. See [sapient.md](sapient.md).
- **DDS Console** (Tools → Extensions → DDS Console) — browses live Speculor instances on the Fast DDS domain and lists their exposed streams. Select a stream, pick a target `dds_subscribe` node from the **Bind to:** dropdown, and click **Bind stream** to wire it. Works with the engine stopped (its own read-only participant). **DDS** interoperability is configured under Settings → Extensions and is **Personal-tier** gated. See [dds.md](dds.md).

## Extensions

Interoperability **extensions** plug the engine into other systems. Each one self-registers its own console (Tools → Extensions) and its own settings page (Preferences → Extensions), and is licence-tier gated:

| Extension | Tier | What it does |
|-----------|------|--------------|
| **Fast DDS** | Personal | Speculor ↔ Speculor: expose any node's output ports on a DDS domain, subscribe to remote streams as ordinary source nodes, share one disciplined clock, opt-in remote parameters. See [dds.md](dds.md). |
| **SAPIENT** | Team | NATO/UK Dstl BSI Flex 335 v2.0 sensor interoperability: act as a sensor (ASM), a fusion node (HLDMM), or both; fusion nodes chain. See [sapient.md](sapient.md). |

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
- **Plugin health watchdog** — automatic detection of hung or crashed plugins, automatic restart up to N retries, cascade-disable of downstream nodes on persistent failure. Tunable in Preferences → Reliability — see [preferences.md](preferences.md#reliability--node-watchdog).
- **`crash_guard`** — wraps each plugin's `process()` in an SEH (Windows) / signal-handler (POSIX) trap, captures up to 32 stack frames on crash, symbolicates via DbgHelp / libunwind. See [troubleshooting.md → crash_guard](troubleshooting.md#crash_guard-stack-traces-show-raw-addresses).

## GPU compute

- **Vulkan acceleration** — optional per-plugin GPU compute for image filters and motion analysis. Auto-detected at startup; per-plugin toggle (`gpu_enabled`).
- **GPU-resident frame passing** — frames stay on the GPU between GPU-capable plugins; no PCIe round-trip.
- **Zero-cost CPU readback** — when an output has a CPU consumer, GPU work copies to a persistently-mapped staging buffer in the same command buffer; CPU consumers get data via `memcpy`, no extra GPU submission. Outputs with only GPU consumers stay resident-only (no staging copy recorded).
- **Automatic CPU↔GPU transfers** — the engine auto-uploads CPU frames to GPU for GPU consumers and auto-downloads for CPU consumers. Plugins don't manage transfers.
- **GPU pipeline depth** — a coalesced GPU subgraph can run several frames in flight. Tunable in Preferences → Performance (default `2`, range `1`–`3`) — see [preferences.md](preferences.md#performance).
- **Mid-run `DEVICE_LOST` recovery** — the engine rebuilds a profile's Vulkan device and restarts the affected nodes when a plugin reports a lost device during execution.
- **Preview GPU/CPU indicator** — the Preview info label shows which side the displayed frame originated on.

## Help dialogs

- **About Speculor** — name, version, copyright, icon.
- **System Information** — OS, CPU (with SIMD extensions), memory, GPU, build/compiler info. Copy-to-clipboard for bug reports.
- **Keyboard Shortcuts** — HTML-formatted reference of all shortcuts.
- **GitHub** — opens the project repository in a browser.

## Project file format

Pipelines save as JSON with the `.speculor` extension — graph, parameters, visualization layouts, connection waypoints, presets, node groups, per-node display options, and DDS exposure. See [project-format.md](project-format.md) for the schema.

## Theme

Dark, Catppuccin-inspired UI. Disabled menu items render in a muted colour so the view-aware Edit menu's enabled state is visible at a glance.
