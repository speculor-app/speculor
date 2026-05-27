# Project file format — `.speculor`

Speculor projects are serialized as JSON via [nlohmann/json](https://github.com/nlohmann/json) v3.11.3. The file extension is `.speculor`. Both the GUI (`File → Save Project…`) and the CLI runner ([cli.md](cli.md)) read the same format.

## Schema

```json
{
  "nodes": [
    {
      "id": 1,
      "plugin_id": "pattern_source",
      "display_name": "Pattern Source",
      "pos_x": 100.0,
      "pos_y": 200.0,
      "disabled": false,
      "color": "#1e1e2e",
      "text_color": "#cdd6f4",
      "overlay_button_position": "top_right",
      "parameters": { "...": "..." },
      "presets": [
        { "name": "Default", "values": { "...": "..." } }
      ]
    }
  ],
  "connections": [
    {
      "id": 1,
      "source_node": 1,
      "source_port": 0,
      "sink_node": 2,
      "sink_port": 0
    }
  ],
  "connection_waypoints": {
    "1:0-2:0": [[250.0, 180.0]]
  },
  "node_groups": [
    {
      "id": 1,
      "name": "My Group",
      "node_ids": [1, 2, 3],
      "locked": false
    }
  ],
  "visualizations": {
    "layouts": [
      {
        "name": "Operator",
        "default": true,
        "gadgets": [ { "...": "..." } ]
      }
    ]
  }
}
```

## Top-level fields

| Field | Required | Description |
|-------|----------|-------------|
| `nodes` | yes | Pipeline nodes. Each entry serialises a single plugin instance with its parameters, customisation, and saved presets. |
| `connections` | yes | Edges between node ports. Both endpoint nodes must exist. |
| `connection_waypoints` | optional | Optional bend points for connection routing. Key format: `outNode:outPort-inNode:inPort`; value is an ordered array of `[x, y]` waypoints. Stale entries (for deleted connections) are pruned lazily on save. |
| `node_groups` | optional | Visual groupings of nodes (lock / move / delete / copy / paste together). |
| `visualizations` | optional | Named gadget layouts for the Visualization view. One layout may be marked `default`. |

`connection_waypoints`, `node_groups`, and `visualizations` are all optional — older project files predate them and continue to load.

## Node fields

| Field | Description |
|-------|-------------|
| `id` | Stable integer node ID. Preserved across copy/paste/undo so connections survive partial restores. |
| `plugin_id` | Plugin descriptor ID (e.g. `"pattern_source"`). Resolved via the discovered plugin registry at load time; missing plugins are reported and the node skipped. |
| `display_name` | User-editable name shown on the node header. |
| `pos_x`, `pos_y` | QtNodes scene coordinates. |
| `disabled` | When true the engine treats the node as bypass / no-op. The pipeline auto-rebuilds when this flag changes during execution. |
| `color`, `text_color` | Per-node overrides for header background and text colour. |
| `overlay_button_position` | Gear-overlay corner: `"off"`, `"top_left"`, `"top_right"`, `"bottom_left"`, `"bottom_right"`. Empty ≡ off. Round-trips through QtNodes copy/paste/undo JSON as well. |
| `parameters` | Map of `name → value`. Values use the JSON type that matches the parameter type (string, number, bool, array). Decimal parameters are always serialised as **strings** (e.g. `"123.45"`) to avoid float round-trip precision loss. |
| `presets` | Saved parameter snapshots, recallable from the parameter panel. Each entry has a `name` and a `values` map. |

## Connection fields

| Field | Description |
|-------|-------------|
| `id` | Connection identifier. Used as the array index when persisting waypoints. |
| `source_node`, `source_port` | Source endpoint. `source_port` is the 0-based output index on the source node. |
| `sink_node`, `sink_port` | Sink endpoint. Same convention. |

The engine validates schema compatibility (`SpcPortSchema`) on connection creation; a mismatch surfaces visually on the connection in the editor.

## Loading and the engine

`spc::ProjectFile::load(graph, path)` populates a `PipelineGraph` from the JSON. The engine then `build()`s a flattened, scheduled execution graph from the model. See [engine-internals.md](engine-internals.md) for the full lifecycle.

When the GUI loads a project it also rebuilds the Visualization layouts, restores connection waypoints into the `ConnectionWaypointStore`, and re-establishes preview subscriptions. The CLI runner skips visualization and preview state.

Duplicate connections — same `(source_node, source_port, sink_node, sink_port, is_control)` tuple — are silently dropped on load. Legacy files saved before `connect_with_id` started enforcing dedup can contain them, and they would otherwise create orphan SpscQueues that stall the engine. The loader counts the drops; the GUI logs a warning, shows a status-bar nudge, and marks the project dirty so the next save persists the cleaned-up file.
