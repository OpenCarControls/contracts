## What and why

<!-- What changes in the wire format, and why? Link the issue and the matching firmware and mobile-app PRs. -->

## Checklist

- [ ] PR title uses conventional-commit style (`feat:`, `fix:`, `docs:`, `chore:`)
- [ ] `buf format`, `buf lint` and `buf breaking` pass
- [ ] Append-only rules followed: no renumbered, retyped or deleted fields; new fields appended and `optional`
- [ ] BLE message size limit still respected (state updates fit in `ble.payload.max_message_bytes`)
- [ ] Matching firmware and mobile-app changes exist or are linked

## Source of the data

<!-- For new vehicle signals: how do you know what they mean (DBC, capture, documentation)? -->
