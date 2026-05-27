# Licensing

Speculor is sold under a per-user offline-capable license. The app and CLI
will not start without a valid signed license file. This page covers the
user-visible mechanics; threat model and dev notes are below.

## How activation works

1. After purchasing a license, the customer receives a license key by email
   (e.g. `XXXX-XXXX-XXXX-XXXX-XXXX`).
2. First launch shows an activation dialog. Enter the key and a friendly
   machine name (defaults to the hostname). Click **Activate**.
3. The app contacts the licensing server once to bind the key to this
   machine's fingerprint, downloads a signed license file, and stores it
   under the OS user's application-data directory.
4. After activation the app works fully offline up to the file's expiry
   plus a grace period (default 14 days). When the app is online the
   license file is silently refreshed in the background ~2 s after the
   main window appears.

A single license supports activation on up to **2 machines by default**
(e.g. desktop + laptop). Additional machines return an
`MACHINE_LIMIT_EXCEEDED` error; the user must deactivate one machine
through the account dashboard before activating a third.

## License file location

| OS      | Path                                                            |
|---------|-----------------------------------------------------------------|
| Windows | `%LOCALAPPDATA%\Speculor\Speculor\license.lic`                  |
| Linux   | `$XDG_DATA_HOME/Speculor/Speculor/license.lic` (fallback `~/.local/share/...`) |
| macOS   | `~/Library/Application Support/Speculor/Speculor/license.lic`   |

The file is a Keygen-style armored Ed25519-signed blob. The app verifies
the signature on every launch with a public key compiled into the binary;
tampering produces a `license file invalid or tampered` block.

## CLI usage

`speculor_cli` reads the cached license file from the same AppData path
the GUI uses. Three setup paths for a headless box:

1. **Activate from the CLI directly** (recommended):
   ```bash
   speculor_cli --activate=XXXX-XXXX-XXXX-XXXX-XXXX
   # optional: --machine-name="render-rack-01"
   # optional: --license-file=/srv/speculor/license.lic
   ```
   Contacts the licensing server, binds this machine to the key, writes
   the signed file, and exits.

2. Activate Speculor once on a desktop machine signed in as the same OS
   user, then hand-copy `license.lic` to the headless machine's AppData
   path above.

3. Pass `--license-file=<path>` to point the gate at a license file
   shipped out-of-band:
   ```bash
   speculor_cli project.speculor --license-file=/srv/speculor/license.lic
   ```

Per-machine binding still applies in cases (2) and (3) — the fingerprint
hash in the file must match the headless machine, so the file must have
been issued for that exact host.

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

## What this scheme protects against

- Casual key sharing → server-side `maxMachines` activation cap.
- Forum-leaked keys → admin revokes server-side; license file expires
  within ~30 days and the next refresh fails.
- Off-the-shelf keygens → impossible without the Ed25519 private key.
- Trivial rename / file tamper → signature verification fails.
- Copying a friend's `license.lic` → machine fingerprint mismatch.
- Forward clock attacks → 24 h issued-in-future guard.

## What this scheme does **not** protect against

By design, for the prosumer tier:

- A determined attacker with Ghidra/IDA can patch the verification call
  out of the binary. We do not ship anti-debug, no integrity self-check,
  no packer.
- The Ed25519 public key sits in the binary's read-only data; an
  attacker can replace it with their own and sign their own licenses.
- A leaked license file remains usable on its original fingerprinted
  machine for up to ~44 days (file TTL + grace) even after server-side
  revocation. This is the deliberate trade-off for offline tolerance.

The registered email is shown in the About dialog and embedded in the
signed payload, so publishing a license key publicly is also publishing
the email it was sold to.

## Developer notes

### Bypass during development

In Debug builds (`scripts/build.ps1 Debug -WithPDB`), set
`SPECULOR_LICENSE_DEV=1` in the environment to short-circuit the gate
to `Allow`. The bypass logs a `WARN` line on every launch and is
**compiled out entirely** in Release builds (`!defined(NDEBUG)` guard
in `core/include/speculor/license_config.h`).

### Local Keygen URL + public key

Each licensing setting is a CMake CACHE variable, overridable in three
places:

1. CLI: `cmake -DSPECULOR_LICENSE_URL=https://api.keygen.sh ...`
2. `ccmake` / `cmake-gui` (each var carries a description string).
3. `cmake/license.local.cmake` (gitignored, FORCE-overrides 1 + 2 so a
   release-machine checkout always wins).

Variables:

- `SPECULOR_LICENSE_URL` — your Keygen CE / Cloud URL
- `SPECULOR_LICENSE_ACCOUNT_ID` — Keygen account UUID
- `SPECULOR_LICENSE_PUBKEY_BYTES` — 32 comma-separated `0xNN` bytes
- `SPECULOR_LICENSE_FP_SALT` — random per-product salt (bake once)
- `SPECULOR_LICENSE_GRACE_DAYS` — offline grace period (default 14)

Copy `cmake/license.local.cmake.example` to `cmake/license.local.cmake`
for the on-disk path. Without this file (or any -D overrides) the build
uses all-zeros defaults; the gate will always Block unless
`SPECULOR_LICENSE_DEV=1` is set in a Debug build.

For a local end-to-end test against self-hosted Keygen CE, see the
sibling [`speculor-backend`](https://github.com/speculor-app/speculor-backend)
repo — `docs/deployment.md` there walks through the Docker Compose
bootstrap (including a `dev` profile for plain-HTTP localhost) and
license issuance via `keygen/mint-license.sh`. The same repo holds the
production stack and (Phase 2) the admin portal.

### Components

- `core/include/speculor/license_manager.h` — Ed25519 verifier + decision
  rules. No Qt dependency, so the CLI can call it directly.
- `core/include/speculor/machine_id.h` — per-platform stable machine id
  hashed with the product salt.
- `app/license_dialog.{h,cpp}` — Qt activation dialog.
- `app/license_client.{h,cpp}` — Qt network client wrapping the Keygen
  REST API (activate, refresh).
- `app/license_gate.{h,cpp}` — entry point used by `app/main.cpp`.
- `tests/unit/test_license_manager.cpp` — round-trips every decision
  branch through the production verifier with a synthetic keypair.
