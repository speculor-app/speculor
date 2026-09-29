# Licensing

Speculor is sold under a per-user offline-capable licence, and **the licence comes with your Speculor account**: a machine gets it by signing in. The app and CLI will not start without a valid signed licence file.

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

The account and its tier are shown on the **Help → Account…** page. Upgrade through **Manage account** there, which opens the customer portal; the machine follows the account's licence at its next launch.

## How signing in works

1. First launch shows **Sign in to Speculor**. Click **Sign in** (or **Create an account** — every account comes with a free Community licence). The sign-in opens in your browser, so Speculor never sees your password.
2. Once you have signed in, the app picks the account's best active licence, binds it to this machine's fingerprint, downloads a signed licence file, and stores it under the OS user's application-data directory. The machine name defaults to the hostname.
3. After that the app works fully offline up to the file's expiry plus a grace period (default 14 days). When the app is online it renews the sign-in and refreshes the licence file in the background ~2 s after the main window appears; if the account now carries a different licence (an upgrade, a purchase), the machine moves to it.
4. **Help → Account… → Sign out** signs the machine out and gives its licence slot back, so another machine can use it.

The app keeps only a sign-in token, never your password: in DPAPI on Windows, in the desktop keyring (GNOME Keyring, KWallet, KeePassXC) on Linux, and otherwise in a file only your OS user can read.

A single licence supports activation on up to **2 machines by default** (e.g. desktop + laptop). Additional machines return a `MACHINE_LIMIT_EXCEEDED` error; sign out on one machine, or remove it through the account dashboard, before signing in on a third.

A machine activated with a typed key before accounts arrived keeps running on it, and asks you to sign in at each launch until you do.

## Licence file location

| OS      | Path                                                            |
|---------|-----------------------------------------------------------------|
| Windows | `%LOCALAPPDATA%\Speculor\Speculor\license.lic`                  |
| Linux   | `$XDG_DATA_HOME/Speculor/Speculor/license.lic` (fallback `~/.local/share/...`) |
| macOS   | `~/Library/Application Support/Speculor/Speculor/license.lic`   |

The file is an armored Ed25519-signed blob. The app verifies the signature on every launch with a public key compiled into the binary; tampering produces a `license file invalid or tampered` block.

## CLI usage

`speculor_cli` requires a tier **above Community** (Personal, Indie, or Team). A Community licence (or none) makes the runner exit with code `2` and an "upgrade at &lt;portal&gt;" message. It reads the cached licence file from the same application-data path the GUI uses. Three setup paths for a headless box:

1. **Sign in from the CLI directly** (recommended):

   ```bash
   ./speculor_cli --sign-in
   # optional: --machine-name="render-rack-01"
   # optional: --license-file=/srv/speculor/license.lic
   ```

   Prints an address and a short code. Open the address in a browser on any device, sign in and enter the code; the CLI then activates this machine with the account's licence, writes the signed file, and exits. Only approve a code you started yourself.

2. Sign in to Speculor once on a desktop machine as the same OS user, then hand-copy `license.lic` to the headless machine's application-data path above.

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

## Keep your licence file private

The account's email is embedded in the signed payload, so publishing a licence file publicly is also publishing that address. Activation is capped per licence, and a leaked licence can be revoked server-side.
