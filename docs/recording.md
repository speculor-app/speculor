# Session recording & replay

Speculor can record **everything a pipeline produces** — compressed video, tables, scalars, records, control traffic, parameter changes, interaction events, and health events — into a self-contained *session* folder, then scrub through it on a timeline or **re-run the whole pipeline from the recording** as if the cameras were live.

The recording & replay feature set is **experimental** and requires a **Personal** licence or higher in its entirety. Below Personal the controls are disabled with an explanatory tooltip and the CLI modes refuse to start. See [licensing.md](licensing.md).

## Recording a session

- **GUI**: arm the **● Record** button on the transport bar (next to Play/Stop). Armed before Play, the recording starts with the run; toggled while running, it attaches/detaches live. Sessions land under `Documents/Speculor/recordings/<timestamp>/` by default — the folder, raw-image codec, pre-record ring, and all storage budgets are configurable under **Edit → Preferences → Recording** (see [preferences.md](preferences.md#recording)).
- **CLI**: `speculor_cli project.speculor --record[=<dir>]` — see [cli.md](cli.md#recording--replay) for every knob.

Recording is designed to never slow the pipeline down: taps copy into bounded spools and background threads do the writing. If the disk can't keep up, the newest data is dropped and the loss window is recorded as a **gap** (see below) — the pipeline itself is unaffected.

While a session is recording, **every node is captured** — including a node whose only output is left unconnected (e.g. an audio or signal branch you want recorded but not processed downstream). The recorder counts as a consumer, so such nodes run and record instead of being skipped as idle. This only applies while recording; a normal run still skips nodes nobody consumes.

### Capture modes

A session is recorded in one of two explicit modes, chosen in **Preferences → Recording → Capture mode** (CLI: `--record-frames=all` selects Full):

- **Sources** (default) — only **data-source** nodes are tapped, and *all* their outputs are (frames, packets, and structured data). A data source is a plugin that ingests from *outside* the pipeline — camera, SDR, audio, GPS, file, network, synthetic generator. Derived nodes (spectrum, oscilloscope, detectors, overlays) are **not** recorded. Leanest on disk. Intended for **reinjection replay**: the recorded sources are fed back through the live pipeline so every derived view **recomputes**.
- **Full** — **every** node's outputs are tapped (one video per frame-producing node, plus all structured traffic), so the session can be **scrubbed forward/back over every view**. Much larger.

The mode decides the replay backend automatically: opening a **Sources** recording runs it as a reinjection replay (its derived views were never recorded, so faithful scrub isn't possible), while a **Full** recording scrubs faithfully.

### What lands in the session folder

| File | Content |
|------|---------|
| `session.json` | Manifest: capture mode, session span, clock anchor, channel inventory, per-part time origins, gap windows |
| `node-<id>[-partN].mkv` | A recorded node's video (data sources in Sources mode; every frame-producing node in Full mode). Compressed sources (RTSP/file) are stored **without re-encoding** (bitstream passthrough, ~0 CPU); sources with no compressed stream (USB/synthetic/astro) are encoded recorder-side (H.264 when an encoder is available, lossless FFV1 otherwise). Audio is muxed alongside when the source carries it. |
| `traffic[-partN].mcap` | Structured traffic, one topic per node/port (`/nodes/3/ports/0/table`, `/nodes/5/params`, `/health`, `/gaps`, `/events`, …) — data-source ports only in Sources mode, every port in Full mode. Standard [MCAP](https://mcap.dev) — opens in Foxglove or the `mcap` CLI. Table schemas are embedded, so files are self-describing. |
| `events.json` | Event summary written at stop: logical count + a downsampled timeline. Lets the Recordings browser preview events without opening the mcap. |
| `poster.jpg` | A 16:9 poster cover frame captured for the session, shown on the Recordings card (absent when the recorder captured no frame). |
| `event_thumbs.json` | Downsampled event thumbnail crops — the Recordings card / event timeline reads these off-thread for hover previews, never the multi-GB mcap. |
| `graph.speculor` | The exact pipeline that produced the session — replay rebuilds from this. |

Long recordings **segment automatically**: video parts roll (keyframe-aligned) at the video part size (default 1024 MB) and MCAP parts at the data part size (default 512 MB). Recorder memory is bounded by the memory spool (default 64 MB); overflow drops are recorded as gaps in `session.json` and as messages on the `/gaps` topic so losses are visible on the timeline instead of silent. All budgets live in **Preferences → Recording**; environment variables override them for headless use — see [cli.md](cli.md#recording-storage-budgets).

Crash tolerance: the MCAP writer flushes every 2 s, so a hard kill loses at most that window; finished video parts are always playable.

### Pre-record ("save the incident")

Standby mode keeps the last N seconds in a RAM ring without touching disk; firing the *incident trigger* flushes the ring and keeps recording live — you capture what led up to the event, not just what follows it.

- **GUI**: set **Preferences → Recording → Pre-record ring** (> 0). On Play a standby ring arms automatically; clicking **● Record** mid-run is the trigger. Stopping without triggering discards the session.
- **CLI**: `--record-standby=<sec>` (+ `--record-trigger-after=<s>` for headless tests).

Compressed (passthrough) sources and structured traffic ring in full; frame-encoded sources start at the trigger — raw frames don't fit a RAM ring.

## Events

Plugins (and users) flag **notable events of interest** — "active track acquired", "object entered zone", "alert" — as structured events with a graduated severity (info/notice/warning/critical), an optional frame region or geo point, and BEGIN/END pairing so an *interval* (a track active for a span of time) shows as a bar rather than a dot. Events are recorded to the `/events` stream in `traffic.mcap` and delivered live to the GUI.

- In **Session Replay** they appear as a strip above the scrubber (dots for instants, colored bars for intervals, click/hover to seek) and in an **Events** tab. Events carrying a frame region also draw their bounding box — severity-coloured, labelled — over the decoded video for as long as they're active at the scrub cursor.
- A live run shows events in the **Events** dock panel (timeline + log); **instant** events with a geo point pin on the **Map** gadget (right-click the map to drop one).

An event can also carry a small **event snapshot** — a JPEG crop of the frame at the moment/region of interest (e.g. the SORT tracker attaches a crop of a track's bounding box when it's first confirmed). The engine crops the region, downscales the long side to ≤128 px, and JPEG-encodes it on a worker thread off the hot path. GPU-only frames with no CPU-readable pixels are delivered as plain events in this release. The crop shows in the event tooltip on the live timeline / log and, once recorded, on the Session Replay event strip; the live view bounds thumbnail memory with a byte-budget LRU so a long, event-heavy run can't grow without limit.

You can drop a **manual event** from the Events panel (live) or the Session Replay **Add Event at Cursor** button (stored in a `manual_events.jsonl` sidecar), or, headless, with `speculor_cli … --record --mark=<sec>[-<sec>]:<label>`.

## Browsing recordings (Recordings view)

**Windows → Recordings** (`Ctrl+3`) is the home for captured sessions and the single entry point for everything recording-related: a full-page browser that scans the recordings folder and lists every session newest-first as a card — a poster cover frame, the project name, a type badge (SOURCES / FULL), start → end and duration, the pipeline summary (node count + names), and an event count with a preview timeline whose points reveal thumbnail crops on hover, plus **Open**, **Reveal** and **Delete** actions.

Open (or double-click) enters Replay Mode with the right backend for the recording's mode. Cards read only the small sidecar files (`session.json`, `graph.speculor`, `events.json`, `poster.jpg`, `event_thumbs.json`), so the list builds without touching the mcap. The header's **Open Other…** button opens a session that lives outside the configured folder, and **Open Folder** reveals the root in your file manager.

## Scrubbing a session (Session Replay)

Opening a **Full** recording from the Recordings view scrubs it faithfully in the main window under the DVR transport: decoded video with play/pause and 0.25–4× speed, plus the latest recorded values of every structured channel at the cursor.

**Tools → Session Replay…** is the standalone floating variant (also read-only, independent of the live engine) and additionally lets you **add manual events** while reviewing.

Headless check: `speculor_cli --replay-dump=<session-dir>` prints a session's channels and sampled values across the timeline.

## Re-running a session (reinjection replay)

Opening a **Sources** recording from the Recordings view (or `speculor_cli --replay=<session-dir>` on the CLI) rebuilds the pipeline from the session's embedded graph and **runs it**: the recorded source nodes are fed decoded video and recorded data at recorded pace (CLI: `--replay-speed=<x>`), and everything downstream — detectors, trackers, gadgets, panels — recomputes live. Peek into any node exactly as during a live run.

Notes:

- Source plugins never start during replay, so a session recorded against an RTSP camera replays on a machine that can't reach that camera. The other plugins in the graph must be installed.
- Interoperability extensions (SAPIENT/DDS) stay disabled — a replayed incident can't publish tracks to live peers.
- Terminal side-effect sinks (network / DB / file recorders, streaming servers, MLAT upload, external feeds) are **skipped** during reinjection replay, so replaying a recorded incident never re-broadcasts, re-uploads, or re-records it.
- Downstream nodes genuinely recompute, so results can differ in small ways run-to-run (tracker IDs etc.). For a bit-faithful view of what happened, use Session Replay; for "what would happen if", reinjection is the tool — combine `--replay-set=<n>.<p>=<v>` with `--record` to change a setting and capture the outcome.
- Recorded compressed video **and audio** are **re-emitted on the source's bitstream port** during replay (timestamps rebased to the current clock), so wired recorder nodes keep working and a record-of-replay stores a bit-exact packet copy — no generation loss.
- Recorded **signal edges** (PCM audio, IQ) are reinjected on replay too, so downstream signal consumers recompute live.

## Exporting

`speculor_cli --export=<session-dir> [--export-from=<sec> --export-to=<sec> --export-out=<dir>]` extracts a segment to standard files: video remuxed (no re-encode) to MP4 (H.264/HEVC) or MKV (FFV1/MJPEG), tables to CSV with columns from the recorded schema, everything else to JSONL. Handy for sharing an incident clip or analyzing detections in a spreadsheet.

## Current limits

- The encode fallback covers RGB24/BGR24/GRAY8/RGBA and full-depth GRAY16/RGB48 (16-bit always lands in lossless FFV1 so astro data is never crushed to 8 bits); NV12/FLOAT32 frames record structured data only.
- Audio is recorded (passthrough) and plays in the MKVs, but replay does not yet feed audio back into signal edges.
- Replay drives *source* nodes; mid-graph replay boundaries are future work.
