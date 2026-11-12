# Security policy

## Supported versions

Security fixes are applied to the current main branch and the latest tagged
release.

## Reporting

After the public repository is created, use GitHub private vulnerability
reporting or a GitHub Security Advisory. Do not open a public issue containing:

- certificates or private keys;
- pairing records or device identifiers;
- raw iAP2 sessions;
- account, location, or device names;
- credentials embedded in logs or screenshots.

Include the affected version, macOS version, reproduction steps with synthetic
data, impact, and any proposed mitigation.

## Credential handling

The repository must never contain:

```text
*.p7b
*.pk8
*.p12
*.pfx
*.pem
*.key
*.apk
identity/
credentials/
captures/
```

The optional importer must keep identities outside the checkout, create the
destination with mode `0700`, and create files with mode `0600`.

The public experimental identity referenced by the importer is shared material
from a third-party APK. It is not a personal Apple key and must not be described
as secret or uniquely assigned to a user.

## Network boundary

The sender and local authentication helper need only local USB and Unix-socket
transport. Run them through `tools/local-only.sb` after dependency installation
when a network-isolated session is required. The importer's download mode is a
separate setup step and is not part of the sender process.

## Threat model

The project assumes:

- the Mac and connected device are controlled by an authorized operator;
- the USB transport and macOS sandbox are not hostile;
- the phone's displayed UI is the authority for whether a location fix or
  health notification is actually enabled;
- device firmware may reject an otherwise valid authentication or location
  exchange at any time.

Protocol acknowledgement is not proof of UI adoption. Reports must preserve that
distinction.
