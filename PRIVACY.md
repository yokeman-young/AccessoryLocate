# Privacy

AccessoryLocate does not include telemetry, analytics, accounts, cloud storage,
or a remote MFi service.

## Local data

The sender communicates locally with an iPhone over USB. Its JSON output goes to
the invoking terminal. The repository does not create a location history,
device database, or diagnostic upload.

The optional identity importer may:

- read a user-selected APK;
- download the pinned DiPlay v0.2.7 APK from GitHub in `--download` mode;
- write the two extracted identity files to a user-selected directory outside
  the checkout.

Download mode contacts GitHub during setup. The sender and authentication helper
do not need remote networking.

## User-controlled paths

Certificate and key paths are supplied through environment variables. Error
output is designed not to print credential contents, credential paths, raw
challenges, signatures, or private-key material.

## Shared artifacts

Screenshots, terminal logs, and device traces can contain names, locations, map
history, device identifiers, or other personal information. Review and redact
them before publishing.

Do not include a real certificate, private key, paired record, or captured iAP2
session in issues, pull requests, releases, or documentation.
