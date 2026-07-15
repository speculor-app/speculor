# Licensing

Speculor is sold under a per-user offline-capable licence. The app and CLI will not start without a valid signed licence file.

## Licence tiers

Every licence carries a **tier** encoded in the signed file. In ascending order: **Community**, **Personal**, **Indie**, **Team**, **Enterprise**. The tier drives feature-gating. A valid licence with no tier field falls back to **Community** (the lowest tier).

| Capability                       | Community | Personal / Indie / Team |
|----------------------------------|-----------|-------------------------|
| Build & run pipelines (app)      | Yes       | Yes                     |
| `speculor_cli` (headless runner) | **No**    | Yes                     |
| MJPEG visualization streaming    | Configure only — **cannot enable/start** | Yes |
| Visualization watermark          | Speculor logo, bottom-right | None      |

Feature gates above the base tiers:

| Feature | Minimum tier | Notes |
|---------|--------------|-------|
| **Session recording & replay** | **Personal** | The Record button, the Recordings view (`Ctrl+3`), Tools → Session Replay, the Preferences → Recording page, and every CLI mode (`--record`, `--replay`, `--replay-dump`, `--export`). Gated in its entirety while the feature is **experimental**. Below Personal the controls are disabled with an explanatory tooltip and the CLI modes refuse to start. See [recording.md](recording.md). |
| **DDS interoperability** | **Personal** | Expose node outputs onto a Fast DDS domain, subscribe to remote Speculor streams via the DDS Console, opt-in remote parameters and DDS-Security. Below Personal the DDS settings page and per-port exposure are disabled. See [dds.md](dds.md). |
| **SAPIENT interoperability** | **Team** | Skipped on Community, Personal, and Indie. See [sapient.md](sapient.md). |

**Plugin loading** is gated the same way. Each plugin declares the minimum tier it needs. At startup the app and `speculor_cli` load only plugins at or below the active tier — a higher-tier plugin is skipped during the plugin scan and never appears in the browser. See [plugins.md](plugins.md).

The tier is shown on the **Help → License…** page. Obtain or upgrade a licence via the **Get a license** button (first-run dialog) or **Switch License…** (License page) — both open the customer portal.

## How activation works

1. After purchasing a licence, you receive a licence key by email (e.g. `XXXX-XXXX-XXXX-XXXX-XXXX`).
2. First launch shows an activation dialog. Enter the key and a friendly machine name (defaults to the hostname). Click **Activate**.
3. The app contacts the licensing server once to bind the key to this machine's fingerprint, downloads a signed licence file, and stores it under the OS user's application-data directory.
4. After activation the app works fully offline up to the file's expiry plus a grace period (default 14 days). When the app is online the licence file is silently refreshed in the background ~2 s after the main window appears.

A single licence supports activation on up to **2 machines by default** (e.g. desktop + laptop). Additional machines return a `MACHINE_LIMIT_EXCEEDED` error; deactivate one machine through the account dashboard before activating a third.

## Licence file location

| OS      | Path                                                            |
|---------|-----------------------------------------------------------------|
| Windows | `%LOCALAPPDATA%\Speculor\Speculor\license.lic`                  |
| Linux   | `$XDG_DATA_HOME/Speculor/Speculor/license.lic` (fallback `~/.local/share/...`) |
| macOS   | `~/Library/Application Support/Speculor/Speculor/license.lic`   |

The file is an armored Ed25519-signed blob. The app verifies the signature on every launch with a public key compiled into the binary; tampering produces a `license file invalid or tampered` block.

## CLI usage

`speculor_cli` requires a tier **above Community** (Personal, Indie, or Team). A Community licence (or none) makes the runner exit with code `2` and an "upgrade at &lt;portal&gt;" message. It reads the cached licence file from the same application-data path the GUI uses. Three setup paths for a headless box:

1. **Activate from the CLI directly** (recommended):

   ```bash
   ./speculor_cli --activate=XXXX-XXXX-XXXX-XXXX-XXXX
   # optional: --machine-name="render-rack-01"
   # optional: --license-file=/srv/speculor/license.lic
   ```

   Contacts the licensing server, binds this machine to the key, writes the signed file, and exits.

2. Activate Speculor once on a desktop machine signed in as the same OS user, then hand-copy `license.lic` to the headless machine's application-data path above.

3. Pass `--license-file=<path>` to point the gate at a licence file shipped out-of-band:

   ```bash
   ./speculor_cli project.speculor --license-file=/srv/speculor/license.lic
   ```

Per-machine binding still applies in cases (2) and (3) — the fingerprint hash in the file must match the headless machine, so the file must have been issued for that exact host.

See [cli.md](cli.md) for the rest of the CLI reference.

## Offline behaviour

| Situation                                          | Decision      |
|----------------------------------------------------|---------------|
| No file on disk                                    | Block         |
| Signature invalid / file tampered                  | Block         |
| Fingerprint doesn't match this machine             | Block         |
| `expiry > now`                                     | Allow         |
| `now > expiry`, `now < expiry + grace` (14 days)   | AllowWithWarning |
| `now > expiry + grace`                             | Block         |
| File claims issued > 24 h in the future            | Block         |

The deliberate trade-off for offline tolerance: a licence file stays usable on its fingerprinted machine until it expires plus the grace period, even if it is revoked server-side in the meantime.

## Keep your key private

The registered email is shown in the About dialog and embedded in the signed payload, so publishing a licence key publicly is also publishing the email it was sold to. Activation is capped per licence, and a leaked key can be revoked server-side.
