# Mounts and cued optical tracking

**Cued optical tracking** points a motorised mount at something another sensor already found, then hands over to the camera to keep it centred. A cue — an ADS-B aircraft, a radar track, a world track — gives a position in the sky. The mount slews there. The mount camera picks the target out of the frame, and a visual lock takes over, correcting the aim from what the camera actually sees rather than from where the cue said the target would be.

The result is a telescope or PTZ head that finds and follows a moving target on its own, with a heads-up display you can steer by hand at any moment.

Speculor's mount chain is the [`mounts`](plugins.md#mounts) bundle plus two supporting plugins from other domains. **Every plugin in this chain is `EXPERIMENTAL`** (`image_preprocess` is `PREVIEW`) — the interfaces work and are in daily use, but expect parameters to move between releases. There is no licence-tier gate; you need the hardware, not a higher tier.

## The chain at a glance

```
  CUE (where to look)                          MOUNT CAMERA (what is actually there)
  ┌──────────────────────┐                     ┌────────────────────────────────────┐
  │ adsb_decoder    ─┐   │                     │ camera ─► image_preprocess         │
  │ sort_tracker    ─┼─► │ tracks_in           │            │                       │
  │ pr_tracker      ─┘   │                     │            ▼                       │
  └──────────────────────┘                     │        sky_object_detect           │
             │                                 │          │        │                │
  static_gps ─┼─► gps_in                       │     boxes_out  image_out           │
             ▼                                 │          │        │                │
      ┌──────────────┐                         │          ▼        │                │
      │mount_director│ ◄── cam_tracks_in ──────┼──── sort_tracker   │                │
      └──────────────┘                         └───────────────────┬┘
        │         │                                                │
   mount_cmd  target_info ──────────────────────────────────►  mount_hud ─► image_out
        │                                                       ▲   │
        ▼                                          mount_state ─┘   │ director_ctrl
  skywatcher_mount / zwo_mount ──────────────────────────────────┘   │ mount_ctrl
                                                                     │ preproc_ctrl
                          (control edges: HUD buttons back to the nodes above)
```

Two loops. The **cue loop** is open — a track goes in, a slew command comes out. The **visual loop** is closed — the mount camera's own detections feed back into `cam_tracks_in`, and the director corrects the aim until the target sits under the reticle.

## What each stage does

| Plugin | Role |
|---|---|
| `mount_director` | The brain. Turns a geo target into an az/el aim point, leads it, and closes the visual lock. Emits `mount_cmd`. |
| `skywatcher_mount` / `zwo_mount` | The drivers. Take `mount_cmd`, talk to the hardware, report `mount_state`. |
| `image_preprocess` | Live-tunable conditioning ahead of the detector — invert, brightness, contrast, gamma, morphology. |
| `sky_object_detect` | Finds small objects against sky, day or night. Emits `boxes_out` plus the pass-through image. |
| `mount_hud` | The gunsight. Draws the overlay and sends operator commands back up the chain. |
| `onvif_ptz` / `ptz_tracker` | The PTZ equivalent: `ptz_tracker` turns tracks into `ptz_cmd`, `onvif_ptz` drives an ONVIF camera head. |

## Supported mounts

| Mount | Plugin | Connection |
|---|---|---|
| Sky-Watcher AZ-GTi and Synta-protocol mounts | `skywatcher_mount` | WiFi (UDP, default `192.168.4.1:11880`), WiFi (TCP), or USB serial |
| ZWO AM5 / AM3 | `zwo_mount` | USB serial or WiFi TCP (default port `4030`), LX200 protocol |
| ONVIF PTZ heads | `onvif_ptz` | Network, ONVIF PTZ service |

`zwo_mount` runs in **Alt-Az** or **Equatorial** mode (`mount_mode`), and can drive the axes either by per-axis rates or by chasing a GoTo (`rate_backend`) — rate control is smoother, GoTo chase is the fallback for firmware that misbehaves under continuous rate commands.

Both drivers share a common core, so `slew_speed`, `nudge_step`, `track_ff` (feed-forward), `update_rate_hz` and the alignment/catalog parameters behave identically across them.

## Building the pipeline

Minimum viable chain — slew to an ADS-B aircraft, no camera:

1. `adsb_decoder` (or any track source) → `mount_director.tracks_in`
2. `static_gps` → `mount_director.gps_in` — **the observer position is mandatory**; without it the director cannot convert lat/lon/alt into az/el
3. `mount_director.mount_cmd` → `skywatcher_mount.mount_cmd`
4. Set `mode` to **Track Input** and `selection` to **Nearest** (or **Manual ID** with a callsign/hex in `target_ident`)

Add the visual loop:

5. mount camera → `image_preprocess` → `sky_object_detect`
6. `sky_object_detect.boxes_out` → `sort_tracker` → `mount_director.cam_tracks_in`
7. `sky_object_detect.image_out` → `mount_hud.image_in`
8. `skywatcher_mount.mount_state` → `mount_hud.mount_state_in`, `mount_director.target_info` → `mount_hud.target_info_in`
9. Control edges from `mount_hud`: `director_ctrl` → `mount_director`, `mount_ctrl` → the driver, `preproc_ctrl` → `image_preprocess`

The camera's field of view matters: set `cam_h_fov` on the director to the mount camera's true horizontal FOV, or the visual lock will over- or under-correct. If the camera publishes its FOV in metadata, wire `cam_meta_in` and the director reads it from there instead.

## Cueing modes

`mode` selects where the aim point comes from:

- **Off** — no commands emitted; the mount stays where it is.
- **Manual Geo** — a fixed `target_lat` / `target_lon` / `target_alt`.
- **Track Input** — a track from `tracks_in`, chosen by `selection`: Nearest, Highest Confidence, or Manual ID.
- **Manual Az/El** — point-and-shoot at a fixed `target_az` / `target_el`.

Cues arrive late. ADS-B in particular is seconds behind reality, so the director **extrapolates the target's position to now** (`extrapolate`, on by default) and adds `lead_time` on top to account for the mount's own slew delay. It also counts pipeline latency in the estimate. `max_fix_age` discards a cue that has gone stale; `target_hold` keeps aiming at the last known position for a few seconds after the track disappears, so a brief ADS-B dropout doesn't abandon the target.

## Visual lock

`lock_mode` controls when the camera takes over:

- **Off** — cue only, no visual correction.
- **Designated Target** — lock only onto the track the director is already aiming at.
- **Any Track** — lock onto whatever the mount camera finds near the centre.

The sequence is *slew-to-cue → acquire → lock*. A camera track within `lock_radius` of centre for `confirm_frames` consecutive frames acquires the lock. Once locked, the director drives the error to zero with a proportional-integral servo (`lock_gain`, `lock_ki`), rate-limited by `max_lock_rate` and smoothed by `smoothness`. Inside `center_deadband` it stops correcting, so the mount doesn't hunt around a target that is already centred.

The lock breaks if the target drifts past `break_radius`. If the camera loses the target entirely — a cloud, a dropout — the director **coasts** on the cue for `coast_timeout` seconds before giving up, then re-acquires at `reacquire_slew_rate`.

`center_offset_x` / `center_offset_y` trim the aim point away from geometric centre — useful when the optical axis and the mount axis are not quite aligned. The HUD's double-click-to-trim writes these for you.

## Pointing limits

The director refuses to command the mount outside a configured envelope, which keeps a telescope off its own tripod legs and a camera out of the sun:

- `min_elevation` / `max_elevation` — the elevation band (default 0–85°).
- `az_limit` with `az_min` / `az_max` — an allowed azimuth sector.
- `limit_hysteresis` — stops the mount chattering on and off the limit.
- `view_gate` with `view_margin` — only track targets that are inside a *view camera's* field of view, wired via `view_meta_in`. This is how you keep a narrow-FOV mount slaved to whatever a wide-FOV camera can actually see.

A target outside the envelope is dropped rather than clamped: the mount stops instead of parking against a limit.

## The HUD

`mount_hud` draws over the mount camera image and is the operator's control surface. It needs no separate window — the buttons are on the video.

- **Reticle** and **lock box** show the aim point and the locked target.
- **Target zoom** — a magnified inset of the locked target (`zoom_size`, `zoom_position`), so a distant aircraft is identifiable without touching the optics.
- **Target info panel** — identity and telemetry of the aimed target: callsign, registration, type, altitude, speed, plus track class and model confidence when a classifier is wired in. `units` switches between imperial (ft, kt, NM) and metric.
- **Steer pad** — a D-pad that nudges the mount by `nudge_deg` per press. Double-clicking the image trims the aim point instead.
- **LOCK / MANUAL / TUNE** toggles — engage the visual lock, take manual control (node parameters beat the command port), and open the preprocess tuning panel.

The TUNE panel adjusts `image_preprocess` live over the `preproc_ctrl` control edge — invert, brightness, contrast, gamma, morphology — so you can dial in the detector against the sky you actually have. It stays hidden until the target node answers the state query, so an unwired `preproc_ctrl` simply means no button.

## Status and limits

- **Everything here is `EXPERIMENTAL`.** Parameters have moved between releases and will again.
- **Alignment is the mount's job, not Speculor's.** `skywatcher_mount` keeps stored references and can auto-align on connect (`auto_align_on_connect`), but a badly aligned mount will point badly no matter what the director computes.
- **The visual lock needs a detector that works on your sky.** `sky_object_detect` is size-agnostic and handles day and night, but a sunlit target against bright cloud is genuinely hard; `image_preprocess` and the TUNE panel exist because this needs tuning per site.
- **WiFi mounts drop commands.** `skywatcher_mount` retries Synta commands over lossy links, but a mount on a marginal WiFi link will track less smoothly than one on USB.
- **No tier gate**, but the hardware is the barrier: a supported mount, a camera with a usable focal length, and a GPS position.

## Licensing

The mounts bundle carries no licence-tier requirement — it runs on **Community**. The cue sources you are likely to pair it with may not: check [licensing](licensing.md) for the tier each source needs.
