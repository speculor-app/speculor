# Passive radar

**Passive radar** detects moving targets without transmitting anything. It listens to an existing broadcast transmitter — an FM or DAB station — and looks for the faint, time‑delayed, Doppler‑shifted echoes that same signal produces when it bounces off an aircraft. A *reference* antenna hears the transmitter directly; one or more *surveillance* antennas hear the echoes; cross‑correlating the two recovers each target's **bistatic range** and **range‑rate**, and with a coherent antenna array, its **bearing**.

Speculor's passive‑radar chain is a domain of four plugins in the [`passive-radar`](plugins.md#passive-radar) bundle. It is a young domain — the source is `PREVIEW`, the correlator, calibrator and map are `EXPERIMENTAL`. There is no licence‑tier gate; you need the hardware, not a higher tier.

```
                  reference antenna (at the transmitter)
                          │
KrakenSDR ──iq_ch0 (ref)──┼───────────────► iq_ref  ┐
          ──iq_ch1 (surv)─┘  surveillance  ► iq_surv ┴─► range_doppler ──rd_map──────► Video gadget
          ──iq_ch2..4 ─────  antennas       (optional extra surv)   └──detections──► pr_map / tracker
                                                                                        │
static_gps ─────────────────────────────────────────────────────► gps_in (receiver pos)
```

For direction finding (bearing, not just range+speed) insert **coherent_calibrator** between the source and the correlator — see [Bearing](#bearing-and-position) below.

## The chain at a glance

| Plugin | Role | Needed for |
|---|---|---|
| **KrakenSDR** (`kraken_sdr`) | Five frequency‑coherent I/Q channels from a 5‑channel array | Always — the coherent source |
| **Range‑Doppler** (`range_doppler`) | The correlator: reference × surveillance → range‑Doppler map + detections | Always — this *is* the radar |
| **Passive Radar Map** (`pr_map`) | Slippy map of geolocated detections | Viewing results on a map |
| **Coherent Array Calibrator** (`coherent_calibrator`) | Sample/phase‑aligns the array against its noise source | Bearing / beamforming **only** |

Detection needs only the source and the correlator. The calibrator is for bearing.

## The hardware: KrakenSDR

A [KrakenSDR](https://www.krakenrf.com/) is five R820T2 tuners (RTL‑SDR) sharing one 28.8 MHz TCXO, so all five channels are **frequency‑coherent**. It presents as five USB RTL‑SDR devices with serials `1000`–`1004`; Speculor maps **channel k → serial 1000+k**, and **channel 0 (serial 1000) is the reference / control dongle** — it owns the calibration noise source and the bias‑tee GPIO. Wire your reference antenna (pointed at the transmitter) to that SMA port.

- **Frequency‑coherent, not sample‑ or phase‑aligned.** The shared clock keeps all five tuners on frequency, but the USB streams start at different instants (a fixed integer‑sample offset, ~31 000 samples at 2.4 MSPS) and each PLL locks at a random phase. That is fine for detection — the correlator self‑calibrates the range origin from the direct‑path peak, and a constant phase offset rides along on the correlation. It is **not** fine for bearing, which is what the calibrator fixes.
- **`librtlsdr` at runtime.** KrakenSDR loads `librtlsdr` exactly like the [RTL‑SDR](plugins.md#oss) plugin. If it's missing the device list shows `(no devices found)` — see [troubleshooting.md](troubleshooting.md#rtl-sdr-device-list-shows-no-devices).
- **Power and USB.** Five channels at 2.4 MSPS is ~24 MB/s over the unit's internal USB 2.0 hub. It needs a solid **5 V / 2.4 A+** supply; under‑power or a weak hub drops samples. If you see overflow errors, lower `sample_rate`.
- **Dithering off, one universal gain.** Leave `dithering` off — with it on, each tuner's phase drifts independently and the array decorrelates. All tuning/gain parameters apply identically to all five tuners.

## Choosing an illuminator

Any strong, wideband broadcast you can also receive directly works. Two common choices:

- **FM broadcast** (~100 MHz) — everywhere, strong, but only a few hundred kHz of usable bandwidth, so coarse range resolution. The default `center_freq` sits in the FM band. Decimate hard (an FM station needs nowhere near 2.4 MSPS).
- **DAB** (Band III, ~195–230 MHz) — a 1.536 MHz ensemble gives ~**195 m** bistatic range resolution, far finer than FM. Excellent PR waveform, with one caveat: its frame structure shows up as artifact lines on the map (see [Known artifacts](#known-artifacts-dab)).

Point the reference antenna at the transmitter; point the surveillance antenna(s) at the sky/volume you want to watch, away from the direct path.

## Building the pipeline

The fastest start is the two ready‑made projects that ship with the plugin's `hardware_test/` kit (and `KRAKEN_HARDWARE_TEST.md`, a first‑contact checklist):

1. **`KrakenChannelCheck.speculor`** — KrakenSDR → five spectrum views. Confirms the device is detected, all five channels stream, and the noise source really is on channel 0. Run this first on any new install.
2. **`KrakenPassiveRadar.speculor`** — ch0 reference + ch1 surveillance → `range_doppler` → map + detections, pre‑tuned for an FM illuminator (300 ms CPI, decimation 8 ≈ 1 km range bins, ECA taps 8).

Build it by hand as: `kraken_sdr` → wire `iq_ch0` to the correlator's `iq_ref` and `iq_ch1` to `iq_surv_0` → `range_doppler` → bind a **Video** gadget to `rd_map` and a **Passive Radar Map** to `detections`. Both `range_doppler` and `pr_map` render to a **frame** port — bind a Video gadget to see it; an Image Sink discards frames.

## Tuning the correlator

`range_doppler` runs the *batches* algorithm: within each CPI it correlates surveillance against the reference at every delay, then FFTs across batches to resolve Doppler.

| Knob | What it controls | Rule of thumb |
|---|---|---|
| `cpi_ms` | Coherent processing interval | Longer → finer Doppler (≈ 1/CPI), more work per map |
| `decimation` | Boxcar decimate before correlation | The **main cost lever** — match it to the illuminator's bandwidth |
| `range_bins` | Bistatic range extent (delay samples) | One bin = `c / (sample_rate / decimation)` metres |
| `doppler_bins` | Doppler FFT length | Unambiguous Doppler = ±`sample_rate / (2·batch_len)` |
| `clutter_mode` | Direct‑path removal | Leave on **ECA** (below) |
| `eca_taps` | Reference delays ECA cancels | **Sets the minimum detectable range** (below) |
| `threshold_db` | Detection threshold over noise floor | Raise if the map is speckly, lower to reach fainter targets |
| `doppler_guard_bins` | Reject cells near zero Doppler | Discards static clutter; genuine 0‑Doppler targets are on the baseline anyway |

**Cost** ≈ `range_bins × doppler_bins × batch_len` complex MACs per map — the dominant expense of the chain. `decimation` divides it directly, so decimate to the illuminator's real bandwidth.

### Clutter cancellation decides whether you see anything

The direct path arrives tens of dB stronger than any echo; removing it is not optional. Three modes:

- **None** — raw surface, diagnostics only.
- **Zero‑Doppler** — subtracts each range bin's mean; cheap, but its range sidelobes remain and bury weak echoes.
- **ECA** *(default)* — least‑squares projects the reference's delayed copies out of surveillance, removing the direct path **and its range sidelobes** (measured ~60 dB suppression).

**`eca_taps` sets your minimum range.** ECA cancels reference delays `0 … eca_taps‑1`; anything inside that span is cancelled along with the clutter, including real targets. It is a hard edge, not a rolloff:

> minimum detectable range ≈ `eca_taps × c / (sample_rate / decimation)`

Set `eca_taps` to cover the clutter extent and no further. At 2.4 MSPS undecimated a bin is ~125 m, so 16 taps blinds roughly the first 2 km — fine for targets tens of km out, worth shrinking if you care about close‑in range.

## Bearing and position

Range alone puts a target on an **ellipse** with the transmitter and receiver at the foci — you need bearing to collapse that to a point.

- **Bearing needs ≥ 2 surveillance channels** (`surv_channels`). With one, the correlator emits range/Doppler only and the bearing/position columns are `NaN` with `has_fix = 0`. With two or more, the per‑channel complex CAF at a detected cell is treated as an **array snapshot** and a Bartlett estimator recovers `bearing_deg`.
- **Array geometry must match reality.** `array_type` is `UCA` (default, unambiguous over 360°) or `ULA` (front/back ambiguous, blind at endfire — it reports both twins rather than guessing). `array_radius_m` is the circle **radius**, not element spacing. Most important, **`array_elements`**:
  - `0` — the surveillance channels *are* the whole ring, evenly spread over 360°.
  - `N` — a stock KrakenSDR circle where ch0 is the reference *inside* the ring; with 5 antennas the four surveillance channels sit at 72/144/216/288°, not 90/180/270/360°.

  Getting the ring wrong dominates the error budget — on a 5‑element KrakenSDR, `array_elements=5` measured 0° bearing error where `array_elements=0` measured 45°. Correcting only the radius makes it *more confident and still wrong*, so set the ring first.
- **Positions.** Give `tx_latitude`/`tx_longitude` (the illuminator) or no position can be solved. Receiver position comes from `rx_latitude`/`rx_longitude`, or wire the `gps_in` input ([`static_gps`](plugins.md#misc-sources) for a fixed mast, `gps_source` for a mobile one). For airborne targets set `target_height_m` — the solved range is a *slant* range while the bearing is a *ground* azimuth, and assuming ground level puts a 10 km‑high target kilometres off.
- **The reported uncertainty is deliberately pessimistic.** `bearing_sigma_deg` combines the noise‑only bound with a `bearing_bias_floor_deg` (default 5°) for multipath, coupling and residual calibration — real errors that don't shrink with SNR. Expect 5–8°, which at 20 km is 1.7–3.5 km of cross‑range smear. **This is a track source, not a position fix — draw the wedge, not a dot.**

## Calibrating the array (for bearing)

Bearing needs the *relative phase between channels* to be known, which the shared clock does **not** give you. Insert **coherent_calibrator** between `kraken_sdr` and `range_doppler`:

```
kraken_sdr ──iq_ch0..4──► coherent_calibrator ──iq_out_0..4──► range_doppler
                 ▲                          └── cal_status ──► Data gadget
                 └──── control_out (is_control edge → drives noise_source) ────┘
```

It toggles the array's onboard **noise source** — a common tone into all channels — measures each channel's integer sample lag and complex gain against it, and emits aligned channels. Two wiring rules:

- **`control_out` must be wired to the source as an `is_control` edge**, or calibration can never start (the plugin says so). This is how it drives `noise_source`.
- **Bind a Data gadget to `cal_status`** — one row per channel with `delay_samples`, `gain_db`, `phase_deg`, `quality`, `locked`. This is how you see whether calibration actually locked; it's the first thing to check if bearing misbehaves.

The default `max_delay` (65536) already exceeds a KrakenSDR's ~31 000‑sample startup skew — don't trim it below the skew or the delay is never found. Retunes and ring overflows force a recalibration automatically. You do **not** need the calibrator for plain detection; the correlator self‑calibrates the range origin.

## The map

**Passive Radar Map** (`pr_map`) is a deliberately dumb plotter: it reads `latitude`/`longitude` from the detections table and does no radar maths, so anything that geolocates can drive it. It draws OSM tiles centred on the receiver, range rings, the illuminator and its **baseline**, and each detection as a blob coloured by Doppler sign (red closing, blue receding, white stationary) sized by SNR, with a **bearing‑uncertainty arc**. Drag to pan, scroll to zoom (cursor‑anchored); editing `center_lat`/`center_lon`/`radius_km` snaps the view back.

Watch the **baseline**: near it the geometry collapses (range error scales as `1/(1−cos β)`) and a target there has zero Doppler at any speed — the line tells you which detections to distrust. An empty map is usually the data, not the display: only rows with `has_fix = 1` are plotted, and both plugins log why rows were dropped.

## Known artifacts (DAB)

A DAB ensemble is a great PR waveform but its deterministic frame structure appears on the map:

- **Frame‑periodic Doppler lines** at multiples of **≈10.42 Hz** (1 / 96 ms frame) around every strong response. Ghost detections at ±10–11 Hz around the clutter ridge are the frame rate, not targets — widen `doppler_guard_bins`, or shift the CPI so the lines fall between bins.
- **Cyclic‑prefix peaks** at ±1 ms delay (≈300 km bistatic) — normally outside the processed range window.

Reference remodulation (decode DAB, re‑synthesise a clean reference) is the literature fix and is not implemented.

## Status and limits

Experimental, and honest about it:

- **CPU only** — the correlation is a natural GPU candidate but runs on the CPU today.
- **Integer sample calibration only** — no fractional‑delay correction (matters for full‑span wideband use, not for bearing at broadcast bandwidths).
- **Global‑median thresholding**, not 2D CA‑CFAR — a locally varying clutter residue is thresholded globally.
- **Bearing is not yet proven in‑pipeline against synthetic data** (identical simulated channels carry zero inter‑element phase); the DSP cores are unit‑validated and the chain has run against a real mast‑mounted KrakenSDR on DAB, with channels streaming, calibration locking, and maps and detections flowing.

## Licensing

The **KrakenSDR** source is open source (Apache‑2.0, developed in [speculor-plugins-oss](https://github.com/speculor-app/speculor-plugins-oss)) and, like RTL‑SDR, loads `librtlsdr` (LGPL‑2.1+) at runtime. The correlator, calibrator and map ship in the closed passive‑radar archive. Both archives extract into `plugins/passive-radar/`. See the [plugin catalog](plugins.md#passive-radar).
