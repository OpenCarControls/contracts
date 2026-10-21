# Contributing to Open Car Contracts

Thanks for your interest! This repository is the single source of truth for the wire format shared by the firmware and the mobile app. Mistakes here can break controllers and apps in the field, so changes are reviewed carefully. Please read the warning in the [README](README.md) first.

## Before you start

*   **Open an issue first** for anything beyond a small fix, so we can agree on the approach before you spend time on it.
*   Adding a vehicle or feature? Describe what the car exposes and how you know it. A contract change is only useful with matching [`firmware`](https://github.com/OpenCarControls/firmware) and [`mobile-app`](https://github.com/OpenCarControls/mobile-app) changes; say where those are.
*   Report security problems privately, as described in [SECURITY.md](SECURITY.md).

## The append-only rules (critical)

1.  Never change a field number.
2.  Never change a field's type.
3.  Never delete a field; mark it `[deprecated = true]` instead.
4.  Always append new fields to the bottom of a message.
5.  Use `optional` for all state fields, to allow partial updates.

The [README](README.md) explains why. `buf breaking` enforces most of this in CI.

Also keep in mind the BLE message size limit (`ble.payload.max_message_bytes` in `opencar/core/v1/transport.toml`): every field added to a vehicle's state grows its worst-case update, and the firmware's tests fail if it no longer fits.

## Before opening a pull request

```sh
buf format -w
buf lint
buf breaking --against '.git#branch=main'
```

## Pull requests

*   Keep them small and focused; one topic per PR.
*   Use conventional-commit style for the PR title (`feat:`, `fix:`, `docs:`, `chore:`). CI validates it, and releases are derived from it.
*   Fill in the PR template.
*   AI-assisted contributions are fine, but you must understand and be able to explain every line you submit.
*   CI runs on your first PR only after a maintainer approves it.

By contributing you agree that your work is released under the project's [LICENSE](LICENSE).
