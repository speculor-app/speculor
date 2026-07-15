# SAPIENT interoperability

Speculor speaks **SAPIENT** (UK Dstl, **BSI Flex 335 v2.0** — the NATO-track open standard for autonomous-sensor interoperability) as a first-class **extension**. A Speculor node can act as a SAPIENT **sensor (ASM)**, a **fusion node (HLDMM)**, or both, and fusion nodes chain — so Speculors federate by speaking the standard itself, with no bespoke inter-Speculor protocol.

SAPIENT is a **Team-tier** feature: the extension is skipped on lower licences. See [licensing.md](licensing.md).

## Architecture

```
 per-camera pipeline:
   camera → detector → tracker → [geolocation] → world tracks
                                        │ lat/lon/alt + covariance, class, velocity
                                        ▼ engine tap
                              ┌────────────────────┐
                              │   world-track bus   │  protocol-neutral
                              └────────────────────┘
                                        │ subscribe
                              ┌────────────────────┐    TCP / mutual-TLS   ┌──────────────┐
                              │  SAPIENT extension  │◄────────────────────►│   SAPIENT    │ protobuf
                              │  role: asm | hldmm  │                      │  codec       │ (BSI 335 v2.0)
                              └────────────────────┘
```

- The **`geolocation`** plugin turns pixel tracks into the protocol-neutral **world-track** contract (geodetic lat/lon/alt + ENU covariance, spherical fallback, velocity, class). See [plugins.md](plugins.md).
- The **world-track bus** is a push pub/sub fed by an engine tap on registered world-track source ports; extensions subscribe.
- The **SAPIENT extension** maps world tracks ⇄ SAPIENT messages over TCP or mutual-TLS.
- The **`world_track_ingest`** source plugin brings an HLDMM's fused output back *into* a pipeline.

## Roles

| Role | Behaviour |
|------|-----------|
| **`asm`** (default) | Connects out to a fusion node; sends Registration → periodic StatusReport (with optional `node_location`) + DetectionReports built from the world-track bus. Reconnects with exponential backoff. |
| **`hldmm`** / **`fusion`** | Listens for ASM connections; ingests their detections and **fuses** across sensors (association + bearing triangulation) into global tracks, which it republishes on the local bus. With a parent configured it also **uplinks** its fused output upward as an ASM (chaining). |

A "central" Speculor is simply `speculor_cli` (or the app) running a project whose SAPIENT extension is in `hldmm` role — not a separate binary.

## Configuration

SAPIENT config is **machine-level** — a property of this deployed box (role, endpoints, identity, position, TLS material), not the pipeline. In the GUI, edit it in **Preferences → Extensions** (`Ctrl+,`, Team-tier gated); it persists to the app's settings and applies at the next pipeline Start. Headless (`speculor_cli`) reads the `SPC_SAPIENT_*` environment variables instead. The field names below are what the extension consumes (and the env-var suffixes / settings fields map onto them one-to-one).

```jsonc
{
  "extensions": {
    "modules": {
      "sapient": {
        "role": "asm",                // "asm" (default) | "hldmm" | "fusion"
        "enabled": true,

        // --- ASM role ---
        "host": "10.0.0.5",           // fusion-node host to connect to
        "port": 7000,
        "name": "Speculor Camera 1",
        "node_id": "",                // UUID; empty -> generated
        "status_interval_s": 5.0,
        "lat": 51.5074,               // fixed-install sensor position (optional):
        "lon": -0.1278,               //   reported as StatusReport node_location so
        "alt": 35.0,                  //   an HLDMM can triangulate bearing-only tracks

        // --- HLDMM role ---
        "listen_port": 7000,          // port to accept ASM connections on
        "parent_host": "",            // optional: uplink fused output to a parent HLDMM
        "parent_port": 0,

        // --- mutual-TLS (both roles + uplink); omit for plain TCP ---
        "tls": false,
        "tls_ca": "/etc/speculor/ca.pem",      // trust store to verify the peer
        "tls_cert": "/etc/speculor/node.pem",  // this node's certificate chain
        "tls_key": "/etc/speculor/node.key",   // this node's private key
        "tls_verify_hostname": false           // also require the peer cert's
                                               // SAN/CN to match the host (not
                                               // just chain to the CA); enable
                                               // only if node certs carry a
                                               // matching hostname
      }
    }
  }
}
```

Environment fallbacks: `SPC_SAPIENT_ROLE`, `SPC_SAPIENT_HOST`, `SPC_SAPIENT_PORT`, `SPC_SAPIENT_PARENT_HOST`, `SPC_SAPIENT_PARENT_PORT`. See [cli.md](cli.md#interoperability).

## Fusion (HLDMM)

The HLDMM fuses detections from multiple ASMs:

- **Association** — `(sensor, object)` continuity first, then a world-geometry gate (geodetic distance, ray-perpendicular distance, or ray-ray closest approach for bearing-only) → stable **global track IDs**. The same target seen by two sensors becomes one world track.
- **Estimation** —
  - ≥2 **bearing** rays from different sensors → **triangulation** (least-squares closest point of the rays → a full 3D fix with no single-sensor altitude assumption; covariance from the normal-equations inverse). Each ASM's origin comes from its StatusReport `node_location` (configure `lat`/`lon`/`alt`).
  - **geodetic** fixes → confidence-weighted mean.
  - a lone bearing → spherical passthrough until a second ray arrives.

## Federation / chaining

Set `parent_host`/`parent_port` on an `hldmm` node and it presents as an ASM to a parent HLDMM, forwarding its fused output upward:

```
ASM ─┐
ASM ─┼─►  HLDMM (child)  ──►  HLDMM (parent)  ──►  external C2
ASM ─┘     (fuses)              (fuses again)
```

No bespoke protocol — every hop is SAPIENT. The uplink reuses the node's identity cert (`tls_cert`/`tls_key`) when TLS is enabled.

## Inbound tasking

A C2 (or parent HLDMM) can re-task a Speculor over SAPIENT. The extension decodes the `Task` `Command`, maps it to a neutral task, and routes it onto live engine parameters via the canonical alias convention, replying with a `TaskACK`:

| SAPIENT command | Effect |
|---|---|
| `look_at` (range-bearing) | **Slew** → `cam_azimuth` / `cam_elevation` (+ `heading`/`pitch`/… aliases) |
| `detection_threshold` / `detection_report_rate` | detector sensitivity (`confidence_threshold` …); LOW/MEDIUM/HIGH → 0.7 / 0.5 / 0.3 |
| `classification_threshold` | classifier sensitivity |

Unmapped commands (e.g. free-form `mode_change`) get a well-formed `TaskACK(rejected)` so conformance stays green. A task is accepted if at least one node exposed the targeted parameter.

## Mutual-TLS

Set `tls: true` plus `tls_ca`/`tls_cert`/`tls_key` to wrap every SAPIENT connection in **mutual TLS** (each side presents *and* verifies a certificate against the CA, TLS 1.2+). Plain TCP is the default.

By default a peer is trusted if its certificate chains to the CA — so in a mesh where every node holds a cert from the same private CA, any node can present a valid cert for a connection to another. Set `tls_verify_hostname: true` (or tick **Verify peer hostname** in Preferences → Extensions) to additionally require the server certificate's identity (SAN/CN) to match the configured `host`, closing that gap. It is off by default because it requires your node certificates to carry a hostname that matches how peers address them.

Generate a private CA + per-node certs (example with the `openssl` CLI):

```bash
# one-time CA
openssl genrsa -out ca.key 4096
openssl req -x509 -new -key ca.key -days 3650 -subj "/CN=Speculor SAPIENT CA" -out ca.pem

# per node (repeat per Speculor / C2)
openssl genrsa -out node.key 2048
openssl req -new -key node.key -subj "/CN=speculor-node-1" -out node.csr
openssl x509 -req -in node.csr -CA ca.pem -CAkey ca.key -CAcreateserial -days 825 -out node.pem
```

Ship `ca.pem` to every node; give each node its own `node.pem`/`node.key`. Verification is against the CA chain (machine certs pinned to a private CA), not hostname — appropriate for a federated sensor mesh.

## Conformance (dstl BSI-Flex-335-v2 Test Harness)

The official **`dstl/BSI-Flex-335-v2-Test-Harness`** (Apache-2.0) is the counterpart node for validating both roles. It is an external tool. Procedure:

1. **ASM:** run the harness as an HLDMM; point a Speculor `asm`-role project at it (`host`/`port`). The harness validates Registration → Status cadence → DetectionReport → Alert, plaintext first, then under mutual-TLS.
2. **HLDMM:** run a Speculor `hldmm`-role project; point the harness's ASM simulator at its `listen_port` and confirm ingest + fused output.
3. **Tasking:** issue `Task`s from the harness and confirm `TaskACK`s (accepted for slew/threshold, well-formed rejected for unsupported commands).

## Resilience notes

- ASM reconnects with exponential backoff (`reconnect_backoff_max_ms`); detections queue (bounded, oldest-dropped) while disconnected.
- HLDMM accepts per-client threads with a two-phase stop (shutdown-to-unblock, then join); the TLS handshake runs per client so a slow client can't stall `accept`.
- Bus delivery is decoupled from the engine hot path (bounded ring + delivery thread), so a slow/blocked network peer never stalls processing.
- Heartbeat-timeout eviction at the HLDMM and message archive/replay are future hardening items.

## Troubleshooting

If a SAPIENT link won't establish or drops repeatedly, open the **SAPIENT Console** (Tools → Extensions) and the **Log** dock for the connection state, then check:

- **Reachability** — the fusion-node host/port (ASM) or `listen_port` (HLDMM) is open through any firewall; the console has a built-in probe.
- **TLS** — if mutual-TLS is enabled, every node must trust the same CA and present its own cert/key; a CA or path mismatch surfaces as a handshake failure.
- **No detections leaving** — the pipeline only transmits world tracks, so it needs a `geolocation` stage after the tracker (a bare tracker has no geographic fix to send).
- **Tier** — the extension is Team-gated; on a lower tier it stays idle even when configured.

More general startup and runtime issues: [troubleshooting.md](troubleshooting.md).
