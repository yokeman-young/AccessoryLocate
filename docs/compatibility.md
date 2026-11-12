# Compatibility and tested environments

AccessoryLocate currently ships as a macOS reference implementation. The
Python iAP2, MFi-v3, NMEA, and identity-import code has platform-independent
boundaries, but end-to-end operation has only been verified on macOS.

Status meanings:

| Status | Meaning |
| --- | --- |
| Supported and tested | Exercised end to end with the hardware listed below |
| Experimental | Code exists, but the full path has not been verified |
| Porting target | Not supported today; follow the documented porting task before claiming support |
| Not supported | No maintained implementation |

## Host platforms

| Host | Status | Notes |
| --- | --- | --- |
| macOS on Apple Silicon | Supported and tested | USB CarKit/iAP2 is the primary path; the Bluetooth bridge is experimental |
| macOS on Intel | Experimental | The Python path should be testable, but no maintained Intel test result is available |
| Windows 10/11 x64 | Porting target | USB-only support may be feasible; no end-to-end Windows test exists yet |
| Linux | Not supported | No maintained transport or hardware test exists |

Windows users can give the repository to an Agent together with
[`agent/WINDOWS_PORT.md`](../agent/WINDOWS_PORT.md). That document defines the
USB-first scope, platform boundaries, and evidence required before changing
the support matrix.

## Reference test environment

The following is a redacted environment that completed the macOS workflow. It
is a reference configuration, not a minimum requirement.

| Item | Reference |
| --- | --- |
| Mac | MacBook Air, Apple M5, 10-core CPU, 24 GB memory, `Mac17,3` |
| macOS | `27.2.1`, build `26B131` |
| Phone | USB-connected iPhone, unlocked and trusted by the Mac |
| Watch | Paired Apple Watch, required for the high-blood-pressure notification branch |
| Connection | Direct USB data cable; no Developer Disk Image |
| Identity | Shared experimental offline MFi identity imported from the pinned DiPlay v0.2.7 APK |
| Network | Online only for dependency and APK preparation; location sending and local authentication run without remote services |

Device names, serial numbers, UDIDs, certificates, private keys, screenshots,
and packet captures are intentionally excluded.

## Minimum functional requirements

- macOS with Apple Command Line Tools.
- Python 3.11 or later and `uv`.
- An unlocked iPhone that trusts the host computer.
- A USB data cable connected directly to the host.
- A compatible MFi-v3 certificate and matching private key. The repository
  does not distribute these files; see
  [`docs/authentication.md`](authentication.md).
- For the health notification branch, an iPhone paired with an Apple Watch.
  Region and medical eligibility requirements still apply.

The current macOS workflow does not require an Apple Developer Disk Image,
Developer Mode, AnyGo, a cloud MFi service, or a commercial location
subscription.

## Feature portability

| Feature | macOS | Windows |
| --- | --- | --- |
| USB lockdown and CarKit service | Tested | Unverified; first Windows milestone |
| iAP2 identification and framing | Tested | Protocol code is portable; device transport is unverified |
| Local MFi-v3 helper protocol | Tested | Python helper is potentially portable; end-to-end authentication is unverified |
| GGA/RMC location messages | Tested | Protocol code is portable; map adoption is unverified |
| Classic Bluetooth bridge | Experimental | Not implemented |
| `sandbox-exec` network isolation | Supported by the reference workflow | No equivalent is bundled; do not claim enforced network isolation |
| Identity importer and cryptographic validation | Tested | Python code is potentially portable; Windows path handling and permissions need tests |
| Health notification setup | Guided on iPhone | Remains an iPhone/Apple Watch UI workflow, not a host-portable feature |

## Windows prerequisite boundary

Windows support is not available by running `make native` or by replacing
`xcrun` with a Visual Studio compiler. A Windows port should begin with USB
CarKit/iAP2 and must first establish:

- an Apple-supported Windows mobile-device driver stack that allows local
  device discovery;
- a working `pymobiledevice3` USB backend at the pinned dependency version;
- a direct USB connection to an unlocked, trusted iPhone;
- a Windows-native path for the identity and authentication helper.

Windows firewall or security-policy changes must be explicit user actions.
The macOS sandbox profile is not a portable security mechanism.

## What does not count as compatibility

- Passing synthetic unit tests is not evidence that Windows can connect to an
  iPhone.
- Receiving `identification_accepted` or `authentication_succeeded` is not
  evidence that Maps adopted the coordinates.
- Seeing increasing `location_link_acknowledged` counts is not evidence of a
  visible map change.
- Enabling the high-blood-pressure notification switch is separate from the
  host operating system and still depends on the paired Watch, region, and the
  user's medical eligibility answers.
- A temporary map change is not complete until the sender is stopped, Location
  Services is restored, and the real location is visible again.
