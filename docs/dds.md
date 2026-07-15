# DDS interoperability (Speculor ↔ Speculor)

The **DDS extension** connects multiple Speculor instances over a [Fast DDS](https://fast-dds.docs.eprosima.com/) domain: expose any node's output ports to the network, subscribe to remote streams as ordinary source nodes, share one disciplined clock, and (opt-in) change parameters across instances.

DDS is gated at **Personal tier and above** — below Personal the DDS settings page and per-port exposure are disabled. See [licensing.md](licensing.md).

```
INSTANCE A (publisher)                        INSTANCE B (subscriber)
cam → detector → tracker                      [dds_subscribe_data] → zones → alerts
        │ "Expose via DDS" (per port)                ▲ engine-fed source
        ▼                                            │
  egress tap → egress pump ───────────────▶ ingest router → fill on tick
                       Fast DDS domain (UDP/SHM, builtin discovery)
```

## Topics and QoS

| Topic | Purpose | QoS |
|---|---|---|
| `spc/directory` | instance cards: identity, exposed streams (encoding + QoS + table schemas), capabilities | RELIABLE, TRANSIENT_LOCAL, KEEP_LAST 1, keyed by `origin_id` |
| `spc/stream/<origin>/<node>/<port>` | one topic per exposed stream — carries that port's samples only | **per-port** (chosen when exposing): reliability, history + depth / keep-all, durability, lifespan, on-full. Defaults: frames best-effort/latest-wins, packets reliable/deep, tables reliable |
| `spc/time/request` + `reply` | master/follower clock sync | BEST_EFFORT, KEEP_LAST 1 |
| `spc/params/request` + `reply` | remote parameter LIST/GET/SET | RELIABLE, KEEP_LAST 16 |

Each exposed stream gets **its own topic** (`spc/stream/<origin_id>/<node_id>/<output_port>`). Per-stream topics give each stream independent QoS and full isolation — a slow subscriber to one stream never even receives another's samples. A subscriber reads the publisher's advertised reliability/durability from the directory card and creates a matching reader, so QoS lines up automatically.

## Exposing node outputs

Right-click a node in the graph editor → **Expose via DDS** → tick output ports (or use the **DDS** group in Node Properties, which shows the same per-port checkboxes). Only publishable output classes are offered (frame/table/scalar/record/packet — not the direct-call signal/control ports). A blue **DDS** badge appears on exposed nodes.

Exposure persists in the `.speculor` project (`interop.dds.exposed_ports`, keyed by node ID — duplicating a node never silently duplicates egress; see [project-format.md](project-format.md)). Toggling **Expose via DDS** takes effect live — the DDS pump re-seeds without restarting the pipeline — and the CLI needs no extra flags. An exposed-but-unwired port still runs its node (the tap counts as a consumer). Table streams advertise their runtime schema on the directory once the first sample flows.

Egress is fully suppressed during any replay (session replay or reinjection) — recorded data never re-emits onto the domain. See [recording.md](recording.md).

### Per-port encoding & QoS

Each exposed port has its own **encoding** and **QoS** — set them from the **Configure…** button next to the port in the Node Properties → DDS group (configuring also exposes the port). They persist per port in the project file, and the DDS Console shows them per stream.

**Encoding** (frame ports only — packets are already a bitstream, tables carry no pixels):

| Mode | Codec | Lossless | Notes |
|---|---|---|---|
| Raw | none | ✓ | same-host/SHM or low-res — highest bandwidth |
| Lossless (fast) | LZ4, delta-filtered (level 0=fast … 12) | ✓ | ~3× at real-time, lowest CPU |
| Lossless (best) | Zstd, delta-filtered (level 1 … 22) | ✓ | ~5× at real-time (level 1) — the science default |
| Lossy | JPEG (quality 1–100; GRAY8/RGB24/BGR24) | ✗ | previews — much smaller, not bit-exact |

Both lossless codecs first apply a **format-aware horizontal delta predictor** (PNG "Sub"-style): raw byte compressors barely dent real textured frames (~1.0×), but delta+compress reaches 3–6×. The predictor is intrinsic to the encoding, so no extra wire field. Compression runs on the extension's egress pump, not the node worker, so frames dropped by a best-effort/latest-wins stream are never compressed. An unsupported codec/format silently falls back to Raw. Decode happens on the subscriber's ingest thread; the plugin never touches a codec.

**QoS** (all ports): `reliability` (reliable / best-effort), `history` (keep-last *depth* / keep-all), `durability` (volatile / transient-local — late joiners get the last *depth*), `lifespan_ms`, and the reliable **on-full** policy (drop-oldest for live, or block briefly so a science stream never drops — the pump absorbs the block, the engine never stalls). Defaults are kind-aware: frames best-effort/keep-last-1, packets reliable/keep-last-64, tables/scalars/records reliable/keep-last-8.

**Bandwidth** is the key design constraint for many-sensor deployments. Measured on textured RGB24 frames (figures are content-dependent and vary by machine). The real-time bar for 4 MP 30 fps is ~351 MB/s encode:

| Encoding | 2 MP ratio | 4 MP ratio | 4 MP encode | Real-time @4 MP? |
|---|---|---|---|---|
| Raw | 1.0× | 1.0× | — | n/a (1493 / 2949 Mbit/s — exceeds GbE) |
| delta+LZ4 fast | 3.5× | 2.9× | ~600 MB/s | ✓ (lowest CPU) |
| **delta+Zstd L1** | **5.6×** | **5.1×** | ~400 MB/s | ✓ (best real-time lossless) |
| delta+Zstd L3 | 8.7× | 6.3× | ~310 MB/s | ✗ — fine at 2 MP or lower fps |
| JPEG q85 (lossy) | 6.7× | 6.7× | ~1000 MB/s | ✓ |

Two things to note: **the delta filter is essential** — plain LZ4/Zstd get only ~1.0× on real textured frames; and **raw frames can't cross a GbE link** (one 4 MP30 stream = ~2.9 Gbit/s). Over a network prefer **compressed video packets** (a few Mbit/s each — 100+ per GbE) or **delta+Zstd** frames; keep **raw** to same-host shared memory or low-res, and reserve **high Zstd/LZ4-HC levels** for sub-HD or non-real-time. Per-stream byte counters (post-compression) are in the extension status and the DDS Console. A one-host loopback sustains ~2 GB/s, above any NIC, so in practice the link is the limit, not the transport.

## Subscribing to remote streams

Drop a **DDS Subscribe** source (Sources/Interop) matching the stream class — `dds_subscribe_data` (table), `_scalar`, `_record`, `_video` (raw frames), `_packet` (bitstream, wire a decoder after it) — and set its parameters:

- `remote_instance` — the publisher's origin id (e.g. `spc-1a2b3c4d`)
- `remote_node` / `remote_port` — the exposed node id + output-port ordinal

The fastest way to fill these is the **DDS Console** (Tools → Extensions → DDS Console): it browses live instances on the domain and lists their exposed streams by name; select a stream, pick the target `dds_subscribe` node from the **Bind to:** dropdown, and click **Bind stream** to fill the three parameters. The console works with the engine stopped (it uses its own read-only participant), which is when you wire graphs. Manual entry always works too.

The engine fills the node from the DDS extension (the plugin never links Fast DDS). Table outputs declare a **dynamic schema** — the real schema latches from the publisher's directory card at runtime and downstream nodes resolve fields by name; Validate Pipeline flags such edges as informational. A packet subscriber joining mid-stream automatically receives the publisher's cached codec config packet (SPS/PPS) and decodes from the next keyframe. Subscribing to your own origin, or exposing a `dds_subscribe_*` node's output, is refused (loop prevention).

## Configuration

Edit in the GUI at **Preferences → Extensions → Fast DDS**, or set the machine-level settings directly (`extensions/dds/*`, shared by GUI + CLI; `--dds-config=<ini>` overrides in the CLI). Preferences/ini is the sole source of configuration — there are no environment-variable overrides:

| Key | Default | Meaning |
|---|---|---|
| `enabled` | `false` | master switch |
| `domain` | `0` | DDS domain id — instances must match |
| `origin_id` | generated once (`spc-xxxxxxxx`) | stable instance identity; remote bindings key on it |
| `friendly_name` | hostname | shown on the directory |
| `partition` | empty | DDS partition (traffic isolation within a domain) |
| `initial_peers` | empty | CSV `ip[:port]` unicast locators when multicast is unavailable |
| `discovery_server` | empty | `ip:port` — Discovery Server mode (recommended >~10 instances) |
| `shm_enabled` | `true` | shared-memory transport for same-host peers |
| `socket_buf_mb` | `0` | send/receive socket buffers (raise for frame streams) |
| `frame_max_fps` | `0` | per-stream frame decimation ceiling (0 = off) |
| `time_role` | `off` | `master` \| `follower` |
| `time_master_id` | empty | follower's master (empty = auto-latch a single advertised master) |
| `time_poll_s` | `10` | follower poll interval |
| `allow_remote_params` | `false` | accept remote parameter SETs (reads are always allowed) |

## Time sync

One instance runs `time_role=master` (advertised on its directory card); followers poll it NTP-style over DDS — a fresh random nonce must echo in the reply and the responder must be the selected master, so blind injection and rogue responders are rejected. Corrections feed the process-wide disciplined clock with its normal slew/step policy, so all instances stamp data on one timeline.

**Followers should set Preferences → Time Sync to Off** — running both disciplines steers the clock from two sources (see [preferences.md](preferences.md#time-sync)). A GPS/NTP-disciplined master transparently propagates its quality (error bound, lock state).

## Remote parameters

Every instance answers **LIST** (full parameter catalog with live values) and **GET** over `spc/params/*`. **SET** is rejected with a well-formed NACK unless the target set `allow_remote_params=true` — the capability is advertised on the directory card so remote UIs grey out editing preemptively. Applied SETs go through the engine's normal parameter write path (STOP_ONLY and MANDATORY are enforced exactly as for local writes) and refresh any open parameter panels.

The **DDS Console → Parameters tab** is the client for this: pick a remote instance, and it fetches that instance's catalog (LIST) into a node/parameter tree with live values. Select a parameter and **Get** re-reads its current value; when the instance advertises `params_writable`, scalar parameters (int / float / bool / string / enum) become editable and **Set** applies the value. Composite types (color / list / decimal / text / trigger) are catalog-only. The tab rides the Console's own browser participant, so — like stream browsing — it operates while the **local** pipeline is stopped; the target instance must be running to answer.

## Security posture

By default the DDS domain is a **trusted-LAN segment**: domain id + partition give isolation, not authentication (`extensions/dds/security/mode` = `off`, the default). That is the right posture when every participant sits on one trusted network — it needs no certificates and costs nothing.

Security is **opt-in** and never auto-detected — the trust boundary is always an explicit choice. Configure it under `extensions/dds/security/*` (Preferences → Extensions → Fast DDS, or the shared ini / `--dds-config`):

- **`mode = secure`** enables DDS-Security **PKI-DH mutual authentication**: every participant presents an X.509 identity certificate, and only those whose certificate chains to the configured `ca_cert` are admitted — any other participant is rejected at the handshake and never even sees the domain's endpoints. Requires `ca_cert`, `identity_cert` and `private_key` (PEM paths); `key_password` if the key is encrypted. Set `require_secure = true` to turn a misconfiguration (missing certs, or `mode` left `off`) into a hard startup failure instead of a silent fall-back to an open participant.
- **Encryption + per-topic access control** layer on top of `secure` when you also supply a signed **governance** + **permissions** pair (`governance` / `permissions`, signed by `permissions_ca` — blank uses `ca_cert`). The governance turns on AES-GCM protection and join access control; the permissions grant each participant — by its certificate subject DN — publish/subscribe on specific topics. **Both are optional**: without them, `secure` is authentication-only (outsiders rejected, but traffic between authenticated peers is cleartext on the domain); add them only when you need the wire encrypted.
- **`mode = transport`** (TLS-over-TCP) is a deliberately deferred future option — it is not offered in the Preferences mode picker (only `off` / `secure`), and is strictly weaker than `secure`; if forced via the ini it refuses to start rather than running open.

Leave `allow_remote_params` off on exposed hosts unless you need remote writes.

### Generating the PKI (encryption tier)

Create a CA, per-instance identity certs whose **CN identifies the instance**, and S/MIME-sign the governance + permissions with the CA:

```sh
# CA (serves as both identity CA and permissions CA)
openssl req -x509 -newkey rsa:2048 -days 3650 -nodes \
  -keyout ca.key -out ca.pem -subj "/C=XX/O=YourOrg/CN=speculor-ca"

# per-instance identity cert (repeat per box; CN must be unique)
openssl req -newkey rsa:2048 -nodes -keyout node.key -out node.csr \
  -subj "/C=XX/O=YourOrg/CN=spc-1a2b3c4d"
openssl x509 -req -in node.csr -CA ca.pem -CAkey ca.key -CAcreateserial \
  -days 3650 -out node.pem

# sign governance + permissions — the <subject_name> in permissions.xml must be
# the node cert's RFC2253 subject, e.g. "CN=spc-1a2b3c4d,O=YourOrg,C=XX"
openssl smime -sign -in governance.xml  -text -out governance.p7s  -signer ca.pem -inkey ca.key
openssl smime -sign -in permissions.xml -text -out permissions.p7s -signer ca.pem -inkey ca.key
```

Point the instance at `ca.pem`, `node.pem` / `node.key`, `governance.p7s`, `permissions.p7s`.

### Scoped encryption (constrained deployments)

AES-GCM is per-sample CPU, multiplied across every stream — costly on raw megapixel frames on low-end boxes. The governance can encrypt **where it matters** and leave bulk streams cheaper, via per-topic `data_protection_kind`:

```xml
<!-- control/attribution: encrypt -->
<topic_rule>
  <topic_expression>spc/directory</topic_expression>
  <data_protection_kind>ENCRYPT</data_protection_kind>
  ...
</topic_rule>
<!-- bulk frame streams: sign-only (access control + integrity, no per-pixel crypto) -->
<topic_rule>
  <topic_expression>spc/stream/*</topic_expression>
  <data_protection_kind>NONE</data_protection_kind>
  ...
</topic_rule>
```

Pay crypto for identity and control (`spc/directory`, `spc/params/*`, `spc/time/*`) and keep the megapixel frame topics (`spc/stream/*`) sign-only where the LAN is the trust boundary.

## Troubleshooting

- **No discovery**: instances must share `domain` (and `partition` if set). Multicast-blocked networks need `initial_peers` or a `discovery_server`.
- **Table subscriber emits nothing**: check the subscriber log for `dds_subscribe node N bound to stream ...`; the schema latches from the publisher's directory within ~1 s of the first sample — a schema mismatch drops samples (counted in the extension's subscription status).
- **Frames stutter on constrained links**: raise `socket_buf_mb`, set `frame_max_fps`, or prefer the packet path (compressed) with a decoder on the subscriber.
- Live counters: the extension's status (per-stream `{samples, bytes, drops, decimated}`, subscription `{received, filled, dropped}`, time-sync offset/RTT) — surfaced in logs and in the DDS Console.

More general startup and runtime issues: [troubleshooting.md](troubleshooting.md).
