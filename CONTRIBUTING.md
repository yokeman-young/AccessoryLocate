# Contributing to AccessoryLocate

Use Python 3.11 or later and uv:

```sh
uv sync --locked
uv run ruff check src tests tools
uv run pytest -q
make native
```

The native build requires macOS and Apple Command Line Tools. Protocol tests use
local loopback sockets. Authentication tests generate temporary synthetic keys
and certificates; no phone or real accessory identity is needed.

Keep protocol changes covered by interoperability-focused tests: malformed
frames, fragmentation, retransmission, sequence wrap, subscription handling,
and cryptographic encoding. Do not add captured sessions or real credentials as
fixtures. Use artificial data and remove device identifiers from issue reports.

Keep user instructions in the README, protocol details in `docs/protocol.md`,
helper integration details in `docs/authentication.md`, and platform support
in `docs/compatibility.md`. A Windows port must follow
`agent/WINDOWS_PORT.md` and may not be marked tested without real-device
evidence.

Keep agent workflows in `agent/SKILL.md` and the porting task in
`agent/WINDOWS_PORT.md`. Preserve the distinction between protocol
acknowledgement and visible UI adoption.

Do not submit APKs, certificates, private keys, paired records, device
identifiers, captured iAP2 traffic, or unredacted screenshots. See
`SECURITY.md` before reporting a vulnerability.

## Local troubleshooting

On macOS with Python 3.14, `uv sync` can succeed while imports fail if the
generated editable-install `.pth` file carries the hidden filesystem flag. Use
Python 3.12 for verification or clear the flag before rerunning:

```sh
chflags nohidden .venv/lib/python3.14/site-packages/*.pth
```

This is a Python/macOS site-processing edge case, not a missing dependency.
