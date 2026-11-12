# Release checklist

Before the first public push or tag:

- Confirm clone URLs and repository metadata use the intended GitHub owner.
- Run `uv sync --locked`, `uv run ruff check src tests tools`,
  `uv run pytest -q`, and `make native`.
- Confirm the working tree contains no `.venv`, cache directories, APK,
  certificate, private key, paired record, screenshot, or captured session.
- Search the tree for local usernames, home-directory paths, device names,
  UDIDs, serial numbers, map history, and author-specific configuration.
- Verify `LICENSE`, `NOTICE.md`, third-party references, and the upstream
  GPL-3.0-only attribution.
- Review `SECURITY.md` and `PRIVACY.md`.
- Verify `docs/compatibility.md` marks only actually tested platforms as
  supported and that `agent/WINDOWS_PORT.md` retains the USB-first boundary.
- Confirm issue templates or repository settings direct security reports to
  private vulnerability reporting.
- Test from a fresh checkout with a synthetic identity only.
- Review the final rendered README and agent runbook on GitHub.
