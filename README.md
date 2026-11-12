# AccessoryLocate

AccessoryLocate is a local, experimental iAP2 accessory-location client with a
macOS reference implementation. It connects to an iPhone over USB through
`com.apple.carkit.service`, performs MFi-v3 identification and authentication
with a local helper, and sends GGA/RMC only after the phone requests a location
subscription.

The project is designed for local interoperability research and authorized
device testing. It does not use a cloud MFi service, a commercial location
subscriber, or a developer disk image.

> This repository is a maintained research distribution based on
> [aur3l14no/accessory-location](https://github.com/aur3l14no/accessory-location).
> It is not an official Apple project and is not affiliated with Apple.

## What it does

- Connects to an unlocked, trusted iPhone over USB.
- Requests the CarKit lockdown service.
- Completes iAP2 identification and MFi-v3 authentication locally.
- Waits for the device to subscribe to GGA/RMC location messages.
- Sends bounded, user-specified coordinates at the requested interval.
- Emits machine-readable JSON events for automation and diagnostics.
- Can run its sender and authentication helper with remote networking denied.

It does not read back the phone's selected location, change arbitrary device
settings, or guarantee that iOS will adopt an accessory-provided fix.

## Requirements

- macOS with Apple Command Line Tools.
- Python 3.11 or later.
- [uv](https://docs.astral.sh/uv/).
- An unlocked iPhone that already trusts the Mac.
- A compatible MFi-v3 certificate and matching private key. These two files are
  **not included in this repository**; the next step explains automatic import.

An Apple Developer Disk Image and Developer Mode are not required.

## Platform compatibility

The published implementation is supported and tested on macOS on Apple
Silicon. macOS on Intel is experimental. Windows 10/11 x64 is a USB-first
porting target, not a currently supported platform; Linux is unsupported.
Classic Bluetooth, the native bridge, and `sandbox-exec` network isolation are
macOS-specific.

See [docs/compatibility.md](docs/compatibility.md) for the tested hardware,
feature portability, Windows prerequisites, and status definitions. To port the
project on Windows, give the repository and
[agent/WINDOWS_PORT.md](agent/WINDOWS_PORT.md) to an Agent. That task preserves
the local protocol implementation and requires real-device evidence before
Windows is marked as tested.

## Quick start

```sh
git clone https://github.com/yokeman-young/AccessoryLocate.git
cd AccessoryLocate
uv sync --locked
```

The repository does not ship `certificate.p7b` or `identity.pk8`. Import the
pinned experimental identity directly from the DiPlay v0.2.7 release:

```sh
IDENTITY_DIR="$HOME/Library/Application Support/AccessoryLocate/identity"

uv run python tools/import_diplay_identity.py \
  --download --output-dir "$IDENTITY_DIR"

export ACCESSORY_MFI_CERT_FILE="$IDENTITY_DIR/certificate.p7b"
export ACCESSORY_MFI_KEY_FILE="$IDENTITY_DIR/identity.pk8"
```

The importer can instead read an APK already on disk:

```sh
uv run python tools/import_diplay_identity.py \
  --apk /absolute/path/to/DiPlay-0.2.7.apk \
  --output-dir "$IDENTITY_DIR"
```

Both modes require the exact pinned APK and keep the identity outside the
checkout. The destination must not already exist.

Connect and unlock the iPhone, tap **Trust This Computer**, then start a bounded
session:

```sh
uv run accessory-location send --transport usb \
  --auth-helper tools/local_mfi_v3_helper.py \
  --latitude 22.3193 --longitude 114.1694 --duration 120
```

Open Maps on the phone to trigger a location subscription. A successful session
should produce events similar to:

```json
{"event":"device_connected","transport":"usb"}
{"event":"identification_accepted"}
{"event":"authentication_succeeded"}
{"event":"location_requested","sentences":[1,2]}
{"event":"location_link_acknowledged","count":1}
```

`location_link_acknowledged` confirms delivery on the iAP2 link. It does not
confirm that iOS has adopted the coordinates. Verify the displayed map position
before reporting success.

For a local-only run, install dependencies first and then use the included
macOS sandbox profile:

```sh
/usr/bin/sandbox-exec -f tools/local-only.sb \
  uv run --no-sync accessory-location send --transport usb \
  --auth-helper tools/local_mfi_v3_helper.py \
  --latitude 22.3193 --longitude 114.1694 --duration 120
```

## Identity material

You must supply an accessory identity accepted by the target device. The
repository does not include a certificate, private key, paired record, device
identifier, or captured session.

The DiPlay importer supports the pinned public preview APK from DiPlay v0.2.7.
It verifies the complete APK SHA-256, reads only the two expected identity
assets, validates their format and key match, and writes them outside the
checkout with owner-only permissions. It never installs or executes the APK.

```sh
IDENTITY_DIR="$HOME/Library/Application Support/AccessoryLocate/identity"

uv run python tools/import_diplay_identity.py \
  --download --output-dir "$IDENTITY_DIR"
```

The pinned APK SHA-256 is:

```text
27b434462da7e78b1ff0987d129ed0a4b0a0e56beceaf8ec221d47150d4675bd
```

Review [docs/authentication.md](docs/authentication.md) before using a shared
experimental identity. Such an identity is public material, not a personal
Apple account key, and device acceptance is not guaranteed.

## Verification states

Keep these states separate:

| State | Evidence |
| --- | --- |
| USB connected | `device_connected` event |
| Accessory identified | `identification_accepted` event |
| MFi-v3 authenticated | `authentication_succeeded` event |
| Location requested | `location_requested` event |
| Packets delivered | increasing `location_link_acknowledged` count |
| Coordinates adopted | Maps visibly shows the expected position |
| Real location restored | Maps visibly returns to its real position after stopping |

If packets are acknowledged but Maps does not move, keep the session running,
toggle Location Services off for about 10 seconds, turn it on again, and request
the current location in Maps. iOS may still reject the fix.

## Agent-assisted workflow

[agent/SKILL.md](agent/SKILL.md) contains a Chinese, agent-readable runbook for
authorized macOS/iPhone/Apple Watch setups. It covers local MFi-v3 authentication,
USB CarKit/iAP2 location testing, a 10-second Location Services restart, health
notification setup, recovery, and evidence-based reporting.

The runbook intentionally separates technical success from UI success and
requires the user to answer medical eligibility questions.

For a Windows port, use [agent/WINDOWS_PORT.md](agent/WINDOWS_PORT.md). It
starts with USB CarKit/iAP2, explicitly excludes the macOS Bluetooth bridge,
and defines the tests and hardware evidence needed before claiming support.

## Safety and scope

- Operate only devices you own or are authorized to test.
- Use synthetic or consented target coordinates.
- Keep sessions bounded and stop them when verification is complete.
- Do not submit identity files, paired records, device identifiers, or raw
  device logs to issues or pull requests.
- Do not use the project to disturb another person, defeat consent, or imply
  affiliation with Apple.

See [SECURITY.md](SECURITY.md) and [PRIVACY.md](PRIVACY.md).

## Development

```sh
uv sync --locked
uv run ruff check src tests tools
uv run pytest -q
make native
```

Protocol tests use local loopback peers and synthetic identities. They do not
connect to a phone or use real credentials.

## Repository layout

```text
agent/       Agent runbook and Windows port task
docs/        Protocol, authentication, compatibility, and source notes
native/      Experimental macOS Bluetooth bridge
src/         Python client
tests/       Loopback and synthetic-identity tests
tools/       Local MFi helper, importer, and sandbox profile
```

## Authors

AccessoryLocate is authored by yokeman-young and Codex. Codex contributed
research, implementation, documentation, testing, and release automation as an
AI collaborator under yokeman-young's direction. yokeman-young is the human
maintainer responsible for publication, device testing, and legal or licensing
decisions.

## License and attribution

AccessoryLocate is distributed under GPL-3.0-only. Upstream copyright and
license notices are retained. See [LICENSE](LICENSE) and [NOTICE.md](NOTICE.md).
