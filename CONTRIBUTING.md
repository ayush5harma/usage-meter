# Contributing to Usage Meter

Thanks for helping improve Usage Meter.

## Before you start

Open an issue for a substantial feature or behavior change so the approach can be discussed early. Keep pull requests focused and explain the user-visible behavior they change.

Do not include account emails, OAuth tokens, Keychain output, cache contents, screenshots of real usage, or personal machine paths in issues, commits, fixtures, or pull requests. Use the existing synthetic fixtures and deterministic preview commands instead.

## Local development

Requirements are macOS and the Xcode Command Line Tools. Build into a scratch directory so your installed app remains untouched:

```sh
APP_DIR=/tmp/usage-meter-dev bash build.sh --force
```

Run the relevant checks before opening a pull request:

```sh
bash test/codex-fixtures/run-all.sh
bash test/agy-statusline/run-all.sh
swift test/contrast/contrast.swift
for script in build.sh install.sh uninstall.sh bin/usage-meter-stats \
  test/codex-fixtures/run-all.sh test/agy-statusline/run-all.sh; do
  bash -n "$script"
  shellcheck -S warning -x "$script"
done
```

The fixture suites must remain offline and account-free. When a visual change affects the menu-bar gauges, regenerate and inspect the deterministic previews described in [docs/reference.md](docs/reference.md#preview-assets).

## Code and documentation

Keep the app and collector separated: the Swift app draws and displays data; the collector obtains it. Preserve the UI's freshness indicators and avoid presenting a fallback cache as a live reading.

Use clear, user-focused documentation. Update the technical reference when changing collector fields, cache behavior, or supported environment variables.

## Pull requests

Include a concise description, the checks you ran, and screenshots made with synthetic preview data when UI changes are visible. A maintainer will review the change before merge.
