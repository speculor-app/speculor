# Plugin catalog

Every block in a Speculor pipeline is a **plugin** — a dynamically loaded library that crosses a stable C ABI. The app discovers plugins in the `plugins/` directory next to the app binary and lists them, categorised, in the **Modules** dock.

Plugins ship as **bundles**, one archive per domain. Install only the domains you need — the app picks up whatever is present.

## Installing bundles

Each bundle is a single `speculor-bundle-<name>-<version>-<platform>.zip` (or `.tar.xz`) on the [Releases page](https://github.com/speculor-app/speculor/releases). Extract it **at the Speculor install root** — the same directory as `speculor_app`. It drops plugin libraries under `plugins/<bundle-name>/` and any vendor libraries (RTL-SDR, WinRadio, embedded Python, ONNX Runtime, Postgres/Mongo clients, …) under `plugins/<bundle-name>/vendor/`. The app discovers both automatically; no configuration.

Bundles assume the Speculor app is already installed — it provides the shared runtime (Qt, OpenCV, FFmpeg, ONNX Runtime, the Vulkan loader, libcurl, …), so bundles never duplicate those.

Two notes:

- **Match the bundle version to the app version.** Upgrading the app without re-extracting the bundles leaves older plugin binaries under a newer app.
- **Plugins are licence-tier gated.** Each plugin declares the minimum tier it needs; anything above your active tier is skipped during the scan and never appears in the browser. See [licensing.md](licensing.md).

Several plugins are GPU-accelerated with Vulkan compute shaders and fall back to CPU when Vulkan is unavailable.

## Bundles at a glance

| Bundle | Plugins | What you get |
|--------|---------|--------------|
| [`adsb`](#adsb) | 13 | ADS-B / Mode S: sources, decode, filtering, enrichment, map + camera overlays, feeding |
| [`audio`](#audio) | 8 | Audio capture/playback, mixing, filtering, noise reduction, scope + spectrum |
| [`sdr`](#sdr) | 15 | SDR receivers, spectrum/waterfall, demod, signal detection & modulation classification |
| [`passive-radar`](#passive-radar) | 4 | Coherent passive radar: KrakenSDR source, array calibration, range-Doppler correlator, map |
| [`database`](#database) | 9 | SQLite / PostgreSQL / MongoDB — lookup, query, write |
| [`video-sources`](#video-sources) | 5 | Cameras (USB/RTSP/file), astronomy cameras, stills, test patterns |
| [`misc-sources`](#misc-sources) | 13 | GPS, weather, Home Assistant, system stats, scalars, DDS + SAPIENT ingest |
| [`image-filters`](#image-filters) | 13 | Resize, morphology, dewarp, homography, masking, stabilization, sync |
| [`data-filters`](#data-filters) | 3 | Field extraction, ground-plane mapping, GPS smoothing |
| [`calibration`](#calibration) | 2 | Astrophotography master dark / bias / flat build + apply |
| [`motion-analysis`](#motion-analysis) | 4 | Background subtraction and optical flow (mostly GPU) |
| [`detection`](#detection) | 19 | Detection, classification, tracking, and multi-sensor fusion |
| [`renderers`](#renderers) | 5 | Overlays and dashboard renderers |
| [`output`](#output) | 6 | Recorders and streaming servers |
| [`automation`](#automation) | 6 | Triggers, logic, scheduling, PTZ control |
| [`scripting`](#scripting) | 1 | Embedded Python script node |
| [`oss`](#oss) | 3 | Open-source plugins (ViBe, SuBSENSE, RTL-SDR) |

Counts are the plugins in each distribution bundle for this release.

## adsb

ADS-B plugins. Airspace/airport reference data ships in `plugins/adsb/data/`.

| Plugin | Category | What it does |
|--------|----------|--------------|
| **ADS-B Decoder** | ADS-B/Sources | Decodes Mode S / ADS-B messages from 1090 MHz I/Q signal and tracks aircraft |
| **Dump1090 Source** | ADS-B/Sources | Fetches aircraft data from a dump1090/readsb/tar1090 aircraft.json HTTP API |
| **SBS-1 BaseStation Source** | ADS-B/Sources | Receives ADS-B aircraft data from an SBS-1 Kinetic (or compatible) receiver via BaseStation TCP port 30003 format |
| **ADS-B Simulator** | ADS-B/Sources | Generates simulated ADS-B aircraft data for testing map plugins |
| **ADS-B Replay** | ADS-B/Sources | Replays recorded ADS-B data from JSON files produced by Data Recorder. Supports variable-speed playback, pause, and looping |
| **ADS-B Filter** | ADS-B/Filters | Filters aircraft table by altitude, distance, squawk codes, emergency status, military flag, and airborne state. Outputs matching and rejected aircraft as separate tables |
| **ADS-B Statistics** | ADS-B/Filters | Computes aggregate statistics from ADS-B aircraft data: total count, altitude band distribution, unique ICAOs, message rate, emergency/military/ground counts |
| **Aircraft Enricher** | ADS-B/Filters | Enriches aircraft data with registration, type, airline, and route information from adsbdb.com with hexdb.io fallback. Connect between any ADS-B source and display |
| **ADS-B Display** | ADS-B/Visualization | Renders ADS-B aircraft positions on an OpenStreetMap tile map |
| **ADS-B Overlay** | ADS-B/Visualization | Projects ADS-B aircraft positions onto a camera image using bearing/elevation geometry. Click aircraft for info panel |
| **ADS-B Exchange Feeder** | ADS-B/Integration | Feeds raw Mode-S/ADS-B data to ADS-B Exchange in Beast binary format |
| **ADS-B Exchange Stats** | ADS-B/Integration | Posts aircraft statistics to ADS-B Exchange for feeder credit and the globe/anywhere map |
| **MLAT Client** | ADS-B/Integration | Connects to ADS-B Exchange MLAT server, sends timestamped Mode-S messages, receives multilateration positions |

## audio

| Plugin | Category | What it does |
|--------|----------|--------------|
| **Audio Input** | Signal/Audio/Sources | Captures stereo audio from a system input device |
| **Wave Generator** | Signal/Audio/Sources | Generates audio waveform samples at real-time rate |
| **Audio Output** | Signal/Audio/Output | Plays audio signal through a system output device |
| **Audio Mixer** | Signal/Audio/Filters | Mixes two audio signals with per-channel and master gain control |
| **Frequency Filter** | Signal/Audio/Filters | Biquad cascade frequency filter with lowpass, highpass, bandpass, and notch modes |
| **Noise Reduction** | Signal/Audio/Filters | Reduces noise via noise gate or spectral subtraction |
| **Audio Spectrum** | Signal/Audio/Renderers | Audio frequency spectrum display with stereo L/R split view (20 Hz – 20 kHz, extends to Nyquist at higher sample rates) |
| **Oscilloscope** | Signal/Audio/Renderers | Renders signal input as an oscilloscope waveform with trigger and measurements |

## sdr

WinRadio receivers ship their vendor libraries in this bundle. For RTL-SDR see the [`oss`](#oss) bundle; for KrakenSDR and the coherent passive-radar chain see [`passive-radar`](#passive-radar).

| Plugin | Category | What it does |
|--------|----------|--------------|
| **WinRadio G3xDDC** | Signal/SDR/Sources | Streams I/Q data from WinRadio G31DDC / G33DDC / G35DDCi receivers |
| **WinRadio G39DDCi** | Signal/SDR/Sources | Streams I/Q data from WinRadio G39DDCi (Excalibur Ultra) receivers |
| **SDR Simulator** | Signal/SDR/Sources | Generates synthetic I/Q baseband samples simulating an SDR receiver |
| **SDR Control Panel** | Signal/SDR/Control | Dynamic interactive control panel for any SDR receiver — auto-discovers parameters |
| **SDR Spectrum** | Signal/SDR/Renderers | SDR waterfall display with spectrum plot, persistence, and 3D waterfall |
| **IQ Demodulator** | Signal/SDR/Filters | Demodulates I/Q baseband signal to audio (AM, FM, USB, LSB, CW) |
| **Radio Tuner** | Signal/SDR/Filters | AM/FM broadcast radio tuner with interactive UI, stereo, scanning, and presets |
| **Signal Classifier** | Signal/SDR/Analysis | Analyzes I/Q signal and classifies modulation type (CW, AM, FM, SSB) using spectral features. Detects signals above the noise floor and outputs center frequency, bandwidth, modulation type, and confidence |
| **SDR Direction Finder** | Signal/SDR/Analysis | Estimates direction of arrival (bearing) by comparing phase between two coherent SDR receivers. Requires two SDRs with synchronized clocks (external reference) and known antenna spacing |
| **Signal Detector** | Signal/SDR/Detection | Detects signals in an I/Q stream via Welch-averaged power spectrum and CFAR thresholding. Emits center frequency, bandwidth, power and SNR per emission — a "where in the band" front end for modulation classification |
| **Burst Detector** | Signal/SDR/Detection | Time-domain pulse/burst detector for intermittent signals (ADS-B/Mode S, radar, transponders, ISM bursts) that an averaged-PSD detector misses. Correlates the Mode S preamble to label ADS-B |
| **Modulation Classifier** | Signal/SDR/Detection | Detects emissions in an I/Q stream and classifies each one's modulation (CW/AM/SSB/FM and digital PSK/QAM/FSK/OFDM) from instantaneous statistics and higher-order cumulants |
| **Deep Signal Classifier** | Signal/SDR/Detection | Detects emissions and classifies each one's modulation with a trained CNN (ONNX). High-accuracy path for digital modulations; runs on DirectML/CUDA/CoreML/CPU |
| **Signal Fusion** | Signal/SDR/Detection | Joins the classical and deep modulation-classifier outputs into one consolidated signal table (frequency-clustered), with both verdicts and an agreement flag per emission |
| **Signal Tracker** | Signal/SDR/Detection | Tracks fused signals across frames into stable, persistent stations: smoothed frequency, voted modulation, and hysteresis so detections stop jumping and flickering |

## passive-radar

Coherent **passive radar**: illuminate targets with an existing transmitter (an FM or DAB broadcast) and detect their echoes across a coherent multi-channel receiver. The chain is `KrakenSDR → Range-Doppler → Passive Radar Map`; add the **Coherent Array Calibrator** only when you want target *bearing* (direction finding), not just detection at a range and speed.

This domain is young — the KrakenSDR source is `PREVIEW`, the correlator, calibrator and map are `EXPERIMENTAL`. **KrakenSDR** is open source and ships in the [speculor-plugins-oss](https://github.com/speculor-app/speculor-plugins-oss) passive-radar archive; the correlator, calibrator and map ship in the closed passive-radar archive. Both extract into `plugins/passive-radar/`.

| Plugin | Category | What it does |
|--------|----------|--------------|
| **KrakenSDR** | Signal/SDR/Passive Radar/Sources | Streams five frequency-coherent I/Q channels from a KrakenSDR (5× R820T2 array sharing one TCXO), one signal port per channel. This is *frequency-coherent* capture — sample and phase alignment are done downstream, which is exactly what a passive-radar correlator needs. Open source; loads `librtlsdr` at runtime |
| **Range-Doppler (Passive Radar)** | Signal/SDR/Passive Radar/Filter | The passive-radar correlator: cross-correlates a surveillance channel against a reference channel into a bistatic range-Doppler map plus target detections, with ECA direct-path/clutter cancellation. With ≥2 surveillance channels it also estimates each target's bearing and geolocation. Both inputs must come from the same coherent receiver |
| **Coherent Array Calibrator** | Signal/SDR/Passive Radar/Filter | Measures each channel's integer sample lag and complex gain against the array's injected noise source and emits aligned channels. Needed for bearing / beamforming — not for plain detection, which the correlator self-calibrates |
| **Passive Radar Map** | Signal/SDR/Passive Radar/Renderers | A slippy OpenStreetMap view centred on the receiver: range rings, the illuminator and its baseline, and each geolocated detection drawn with its bearing-uncertainty arc (a wedge, not a dot) |

Both `range_doppler` and `pr_map` render to a **frame** port — bind a Video gadget (an Image Sink discards frames). KrakenSDR needs `librtlsdr` like RTL-SDR does; see the [`oss`](#oss) note.

## database

Vendor client libraries ship in this bundle.

| Plugin | Category | What it does |
|--------|----------|--------------|
| **SQLite Query** | Databases | Query a SQLite database and output results as table and/or JSON |
| **SQLite Lookup** | Databases | Query a SQLite database using values from input data as parameters |
| **SQLite Write** | Databases | Write table or JSON data to a SQLite database |
| **PostgreSQL Query** | Databases | Query a PostgreSQL database and output results as table and/or JSON |
| **PostgreSQL Lookup** | Databases | Query a PostgreSQL database using values from input data as parameters |
| **PostgreSQL Write** | Databases | Write table or JSON data to a PostgreSQL database |
| **MongoDB Query** | Databases | Query a MongoDB collection and output results as table and/or JSON |
| **MongoDB Lookup** | Databases | Query a MongoDB collection using values from input data as parameters |
| **MongoDB Write** | Databases | Write table or JSON documents to a MongoDB collection |

## video-sources

| Plugin | Category | What it does |
|--------|----------|--------------|
| **Video Source** | Sources/Video | Reads video from USB cameras, URLs/streams, or video files |
| **Image Source** | Sources/Video | Loads a static image from disk and outputs it as a frame (color or grayscale) |
| **Pattern Source** | Sources/Video | Generates a scrolling gradient test pattern |
| **QHY Camera** | Sources/Video | Captures video from QHY astronomy cameras (QHY183C, QHY5III178, …) |
| **ZWO ASI Camera** | Sources/Video | Captures from ZWO ASI astronomy cameras (ASI183, ASI294, ASI2600, ASI678, …) |

## misc-sources

| Plugin | Category | What it does |
|--------|----------|--------------|
| **GPS Source** | Sources/GPS | Reads GPS data via serial port or TCP socket (auto-detects NMEA and UBX binary) |
| **Static GPS** | Sources/GPS | Outputs a fixed GPS position with optional NTP time synchronization |
| **Weather Source** | Sources/General | Fetches weather forecast and current conditions from the Open-Meteo API |
| **System Stats** | Sources/General | Collects cross-platform system statistics: CPU, memory, disk, network, process info |
| **Scalar Source** | Sources/General | Outputs a constant scalar value |
| **Home Assistant Source** | Sources/Automation | Connects to Home Assistant and streams entity states for a selected device |
| **Track Simulator** | Sources | Emits a single synthetic track at a configurable normalized position. Test fixture for the geolocation / interoperability path |
| **DDS Subscribe (Table)** | Sources/Interop | Emits a table stream received from another Speculor instance over the Fast DDS domain — see [dds.md](dds.md). Personal tier |
| **DDS Subscribe (Scalar)** | Sources/Interop | Emits a scalar stream received from another Speculor instance. Personal tier |
| **DDS Subscribe (Record)** | Sources/Interop | Emits a record (JSON) stream received from another Speculor instance. Personal tier |
| **DDS Subscribe (Video)** | Sources/Interop | Emits a raw video-frame stream received from another Speculor instance (best-effort, latest-wins — intended for LAN deployments). Personal tier |
| **DDS Subscribe (Packet)** | Sources/Interop | Emits a compressed bitstream packet stream from another Speculor instance, for a local decoder node. Personal tier |
| **World-Track Ingest** | Sources/Interop | Emits world tracks received from other SAPIENT nodes (an HLDMM's fused output) into the pipeline — see [sapient.md](sapient.md). Team tier |

## image-filters

| Plugin | Category | What it does |
|--------|----------|--------------|
| **Image Resize** | Filters/Image | Scales image to a target height, preserving aspect ratio |
| **Debayer** | Filters/Image | Demosaics raw bayer frames and optionally converts bit depth |
| **Morphology** | Filters/Image | Morphological operations: erode, dilate, open, close, gradient, top hat, black hat |
| **Dual Morph** | Filters/Image | Morphological CLOSE then OPEN noise cleanup. GPU-accelerated; designed for cleaning up background-subtraction foreground masks without leaving the GPU |
| **Mask Merge** | Filters/Image | Per-pixel boolean combine of two binary masks. Inputs at different resolutions are nearest-neighbor upscaled before combining. GPU-accelerated |
| **ROI Filter** | Filters/Image | Applies a mask image to filter regions of interest. Use the Mask Editor to create masks, then load the saved PNG here |
| **Homography** | Filters/Image | Applies a perspective transform to warp frames. Define 4 source and 4 destination points in normalized coordinates |
| **Fisheye Dewarper** | Filters/Image | Reprojects equidistant fisheye images to a rectilinear perspective view. Drag to pan/tilt, scroll to zoom, arrow keys for fine control, R to reset |
| **Image Stabilizer** | Filters/Image | Electronic image stabilization using feature tracking to estimate camera motion and warp frames to cancel it. Smooth mode damps shake on a moving camera; Lock mode registers every frame onto a fixed reference so the background stays pixel-static for background subtraction |
| **Image Overlay** | Filters/Image | Composites an overlay image onto a background at a configurable position |
| **Camera Sync** | Filters/Image | Synchronizes two camera streams by timestamp. Buffers recent frames and outputs the closest-matching pair |
| **Frame Limiter** | Filters/Image | Limits frame rate by dropping frames. Target FPS mode sleeps to match output cadence; Decimation mode passes every Nth frame |
| **Image Sink** | Filters/Image | Consumes image frames and discards them. Attach to a node to keep it running without a preview or recorder wired up |

## data-filters

| Plugin | Category | What it does |
|--------|----------|--------------|
| **Field Extractor** | Filters/Data | Extracts a named field from table or record data as a typed scalar |
| **Perspective Mapper** | Filters/Data | Transforms tracked object coordinates from pixel space to real-world ground-plane coordinates using a 4-point calibration homography. Adds world position (meters), speed (m/s), and distance fields to tracks |
| **GPS Stabilizer** | Filters/GPS | Filters and stabilizes GPS data: rejects invalid fixes, smooths position jitter, and applies dead-zone suppression |

## calibration

Astrophotography calibration. Optional FITS support ships in this bundle.

| Plugin | Category | What it does |
|--------|----------|--------------|
| **Master Frame Builder** | Filters/Calibration | Stacks N calibration frames into a master dark / bias / flat and writes it to disk (FITS or 32-bit TIFF). Passes frames through so it can sit inline in a capture graph |
| **Frame Calibrator** | Filters/Calibration | Applies master dark / bias / flat to each light frame, matching the master to the light's exposure / gain / offset / read-mode / binning / temperature. Operates on raw Bayer (preserves the mosaic) |

## motion-analysis

| Plugin | Category | What it does |
|--------|----------|--------------|
| **MOG2 BGS** | Analysis/Motion | Gaussian Mixture Model (MOG2) background subtraction — adaptive multi-modal per-pixel model |
| **WMV BGS** | Analysis/Motion | Weighted Moving Variance background subtraction — outputs foreground mask |
| **Optical Flow** | Analysis/Motion | Horn-Schunck dense optical flow |
| **Optical Flow (NVIDIA NVOFA)** | Analysis/Motion | Hardware optical flow via `VK_NV_optical_flow` (NVIDIA Turing/Ampere+ NVOFA engine) |

## detection

ONNX Runtime and its providers ship in this bundle.

| Plugin | Category | What it does |
|--------|----------|--------------|
| **Blob Detect** | Analysis/Detection | Connected-component blob detection with optional merging and shape filtering |
| **Detection Zone** | Analysis/Detection | Defines polygon zones on the video frame and classifies tracked objects by zone membership. Outputs entry/exit/dwell events, per-zone occupancy counts, and an alert when any zone exceeds its maximum occupancy |
| **Dim Target Detect** | Analysis/Detection | Sensitive small/dim-target front-end: morphological top-hat + adaptive threshold + top-K candidates. Pair with Coherence Tracker to confirm faint movers (distant drones) that a BGS+blob chain erases |
| **Patch Classifier** | Analysis/Detection | Learned per-detection clutter filter: crops a temporal patch (3 frames) around each detection and runs a small ONNX CNN that scores drone-vs-clutter, passing through only above-threshold detections |
| **YOLO Detector (ONNX Runtime)** | Analysis/Detection | Object detection via YOLO ONNX model (DirectML on Windows, CoreML on macOS, CUDA on Linux). Outputs bboxes with class id + name |
| **RF-DETR Detector (ONNX Runtime)** | Analysis/Detection | Real-time detection via Roboflow's RF-DETR transformer on ONNX Runtime. Schema-compatible with the YOLO detector |
| **YOLO Detector (RKNN / RK3588 NPU)** | Analysis/Detection | Object detection via YOLO on the Rockchip RK3588 NPU using the native RKNN runtime. aarch64-Linux only |
| **ncnn Detector (Vulkan, GPU-resident)** | Analysis/Detection | YOLO detector via ncnn running on Speculor's own Vulkan device; consumes a GPU-resident frame with no CPU copy (CPU fallback otherwise) |
| **Object Classifier** | Analysis/Classification | Classifies cropped tracks via an ONNX model; smooths labels per track id |
| **Day/Night Classifier** | Analysis/Classification | Classifies frame as Day or Night based on HSV brightness |
| **Cloud Estimator** | Analysis/Classification | Estimates cloud cover percentage using a day or night algorithm |
| **SORT Tracker** | Analysis/Tracking | OC-SORT enhanced multi-object tracker — assigns stable track IDs to bounding boxes |
| **Bbox Merger** | Analysis/Fusion | Combines bbox tables from up to 4 detector sources into a single table. Overlapping boxes merge into one spanning box; output feeds directly into the SORT Tracker |
| **Track Union** | Analysis/Fusion | Combines track tables from up to 4 per-regime trackers on one camera into a single track set, preserving the full track schema. Track ids are offset per source so merged IDs stay globally unique |
| **Track Merger** | Analysis/Fusion | Merges track tables from multiple cameras into a unified table with cross-camera global track IDs. Matches tracks by proximity in world coordinates. Connect up to 4 track sources |
| **Track Class Enricher** | Analysis/Fusion | Assigns class labels to tracks by IoU-matching them to detections and smoothing the class vote per track id |
| **Geolocation** | Analysis/Fusion | Projects pixel-space tracks to geodetic lat/lon/alt + ENU covariance from camera pose, sensor GPS, and a range model. Emits the canonical world-track schema consumed by [SAPIENT](sapient.md) |
| **Geo Correlator** | Analysis/Fusion | Correlates visual tracks from cameras with ADS-B aircraft positions by projecting both to bearing/elevation space. Outputs matched pairs and unmatched visual tracks (potential unknown aircraft) |
| **Audio Analyzer** | Analysis/Audio | Computes audio metrics: RMS, peak, crest factor, spectral centroid, dominant frequency, zero-crossing rate |

## renderers

| Plugin | Category | What it does |
|--------|----------|--------------|
| **BB Display** | Renderers/Overlays | Draws bounding boxes on an image frame |
| **Track Display** | Renderers/Overlays | Draws tracked bounding boxes, trails, and classification labels |
| **Signal Overlay** | Renderers/Overlays | Labels detected signals on an SDR spectrum/waterfall frame: a marker and modulation label at each signal's frequency column |
| **Stats Monitor** | Renderers/General | Renders system statistics as a dashboard monitor image |
| **Weather Display** | Renderers/General | Renders weather data from Open-Meteo as a visual dashboard |

## output

H.264/HEVC encoder libraries ship in this bundle.

| Plugin | Category | What it does |
|--------|----------|--------------|
| **Video Recorder** | Output/Recording | Records video to file with start/stop control via parameter or scalar input |
| **Data Recorder** | Output/Recording | Records table, scalar, and record data to JSON files with pre/post buffer |
| **Merged Recorder** | Output/Recording | Records synchronized video and data to paired `.mp4` and `.json` files |
| **I/Q Recorder** | Output/Recording | Records a raw I/Q signal stream to an interleaved `.cs16`/`.cs8` file with a JSON sidecar (sample rate, center frequency) for offline analysis or ML |
| **MJPEG Server** | Output/Streaming | Streams video as MJPEG over HTTP. Connect a browser to `http://localhost:<port>/stream`. Supports multiple simultaneous clients |
| **H.264 RTSP Server** | Output/Streaming | RTSP server streaming H.264 video. Connect with VLC or ffplay at `rtsp://localhost:<port>/stream`. Uses NVENC/QSV hardware encoding when available |

These are terminal side-effect sinks — they're skipped during reinjection replay, so replaying an incident never re-records or re-broadcasts it. See [recording.md](recording.md).

## automation

| Plugin | Category | What it does |
|--------|----------|--------------|
| **Alert Trigger** | Automation/Triggers | Monitors a scalar or table input and triggers alerts when conditions are met. Can log messages, output alert state, and send parameter commands to other nodes |
| **Logic Gate** | Automation/Logic | Boolean logic on up to 4 scalar inputs (AND, OR, XOR, NAND, NOR, MAJORITY, THRESHOLD). Emits a control message on rising/falling edges to automate parameters on other nodes |
| **Scheduler** | Automation/Logic | Changes parameters or applies presets on other nodes based on time-of-day schedules or day/night classification |
| **PTZ Tracker** | Automation/Control | Closed-loop PTZ controller: selects a track and outputs velocity or position commands. Supports centering (PID) and acquisition (wide-to-PTZ) modes |
| **ONVIF PTZ** | Automation/Control | Sends PTZ commands to an ONVIF-compatible camera via SOAP/HTTP. Accepts a command table with velocity or position values |
| **Auto Tuner** | Automation/Control | Hill-climbs one integer parameter on a downstream node to maximise a tracking objective — live, in-engine |

## scripting

| Plugin | Category | What it does |
|--------|----------|--------------|
| **Python Script** | Scripting | Executes a Python script for arbitrary data processing. Ships a self-contained CPython + NumPy runtime next to the plugin, so scripts see the stdlib + NumPy out of the box. To use additional installed libraries, create a venv against a matching Python version and set *Python User Prefix*. The script defines `process(inputs, outputs)` receiving dicts with frame (NumPy), table, scalar, and record data |

This bundle is the largest — the embedded CPython runtime adds roughly 70 MB.

## oss

Open-source plugins, developed in the public [speculor-plugins-oss](https://github.com/speculor-app/speculor-plugins-oss) repository. They install exactly like any other bundle.

| Plugin | Category | What it does | Licence notes |
|--------|----------|--------------|---------------|
| **RTL-SDR** | Signal/SDR/Sources | Streams I/Q data from RTL-SDR Blog V3 (R820T2) and V4 (R828D) receivers | Apache-2.0 plugin code; links librtlsdr (LGPL-2.1+) at runtime |
| **ViBe BGS** | Analysis/Motion | ViBe background subtraction (CPU + Vulkan GPU) — outputs foreground mask | Apache-2.0; LITIV-derived; **ViBe is patent-encumbered** in some jurisdictions |
| **SuBSENSE BGS** | Analysis/Motion | SuBSENSE background subtraction (CPU + Vulkan GPU) — high-quality color+texture BGS with adaptive per-pixel sensitivity (quality mode, not real-time at full resolution) | Apache-2.0; LITIV-derived |

**KrakenSDR** is also open source (developed in the same repo) but ships in the [`passive-radar`](#passive-radar) bundle, alongside the correlator it feeds.

RTL-SDR and KrakenSDR both load `librtlsdr` at runtime — if it's absent the device list shows `(no devices found)`. See [troubleshooting.md](troubleshooting.md#rtl-sdr-device-list-shows-no-devices).

## Hardware that needs a vendor driver

Most plugins are self-contained. These need hardware support present on the machine:

| Plugin | Requirement |
|--------|-------------|
| **QHY Camera** | QHYCCD driver (vendor library ships in `video-sources`) |
| **ZWO ASI Camera** | ZWO ASI driver |
| **WinRadio G3x / G39** | WinRadio driver (vendor library ships in `sdr`) |
| **RTL-SDR** / **KrakenSDR** | `librtlsdr` — see above |
| **YOLO Detector (RKNN)** | Rockchip RK3588 NPU, aarch64-Linux only |
| **Optical Flow (NVIDIA NVOFA)** | NVIDIA Turing/Ampere or newer |
