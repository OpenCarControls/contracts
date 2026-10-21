# BLE fragmentation

Wire-level contract for sending messages larger than one ATT packet over the
Open Car Control GATT service. The values live in
`opencar/core/v1/transport.toml` (`[ble.payload]` and `[ble.fragmentation]`);
this page explains them. Firmware and app must both follow it.

## Mechanism: a one-byte chunk header

Every write (`app_to_device`) and every notification (`device_to_app`) carries
exactly **one chunk**: a 1-byte header followed by a slice of the encoded
`AppToDevice` / `DeviceToApp` message. The receiver concatenates the slices and
decodes the result once the last chunk has arrived.

```
  bit  7      6 .. 0
     +------+-------------+----------------------+
     | LAST | CHUNK_INDEX | data (≤ chunk_data)  |
     +------+-------------+----------------------+
        header (1 byte)
```

- `CHUNK_INDEX` (7 bits) is `0` for the first chunk of a message and increments
  by one for every following chunk. It never wraps: a message has at most 128
  chunks.
- `LAST` is `1` on the final chunk. A message that fits in one chunk is a single
  chunk with header `0x80`.
- Chunk data size is `negotiated_ATT_MTU - 3 (ATT header) - 1 (chunk header)`.
  With the requested MTU of 247 that is 243 bytes. The receiver does not need
  to know the chunk size: it only looks at the header.
- Chunks of one message are sent back to back; chunks of two messages are never
  interleaved on a characteristic.
- Both characteristics use the same framing. Only device to app is implemented
  first, but app to device (e.g. OTA) is covered by the same format.

Header byte examples: `0x00` first chunk of a multi-chunk message, `0x01`
second, `0x82` third and last, `0x80` a complete single-chunk message.

### Limits

- `ble.payload.max_message_bytes` = **4096**: the largest reassembled message.
  Sized for the ESP32-C3 (400 KB SRAM shared with the BLE host and the cellular
  modem stack): one reassembly buffer per direction costs 4 KiB, and it is
  well above today's E-GMP state (~244 bytes) with room for growth. Raising it
  is a contract change.
- The effective limit for a connection is
  `min(max_message_bytes, 128 * chunk_data_bytes)`. The app must therefore
  negotiate an ATT MTU of at least `ble.fragmentation.min_att_mtu` (64);
  below that, 128 chunks could not carry 4096 bytes.
- A sender must never produce a message above the limit. A receiver drops a
  message that grows beyond it.

### Error handling

BLE notifications and writes are ordered and checked by the link layer, so
corruption is not expected; chunks can only be lost when a stack queue
overflows (notifications) or a write-without-response is dropped, or when the
link drops. No retransmission exists. The receiver keeps at most one partial
message per characteristic and **discards it** when:

1. The link disconnects (reassembly state is also cleared on connect).
2. A chunk arrives with an index other than `previous + 1` (a gap or a
   duplicate), or a chunk with index > 0 arrives with no partial message.
   The receiver then ignores chunks until the next chunk with index 0.
3. A chunk with index 0 arrives while a partial message exists: the old
   partial is dropped and a new message starts.
4. No further chunk arrives within `ble.fragmentation.reassembly_timeout_ms`
   (2000) of the previous one.
5. The reassembled size would exceed `max_message_bytes`.
6. The reassembled bytes fail to decode.

A discarded message is simply lost; nothing is reported on the wire.
Consequences:

- State updates are partial and merged by the app, so a lost one means the
  value is refreshed on the next update that carries it. The device should
  keep re-sending state as it already does.
- Commands carry a `message_id`; a lost command or response surfaces as the
  app's normal response timeout, and the app may retry.

## Why not the alternatives

| Option | Verdict |
| --- | --- |
| **Chunk header per notification** (chosen) | Works on any GATT stack and both characteristics, no new GATT objects, 1 byte overhead per ATT packet, trivial to implement and test on the ESP32-C3 and in Dart. |
| L2CAP connection-oriented channel | Gives a stream and flow control, but support differs per platform and mobile BLE plugin (not verified for the app's plugin), it adds a second pairing/permission path and a new PSM contract, and it is overkill for ~1 KB messages. Could be added later for bulk transfers. |
| Bigger ATT MTU only | ATT caps a value at 512 bytes and phones often negotiate less; it does not remove the limit, only moves it. |
| Fragment inside protobuf (chunk fields in the envelope) | Would force the envelope to be decodable before it is complete, and chunk bookkeeping would live in every message type. |
| Multiple smaller state messages | Already possible through optional-field partial updates, but a single logical update can still not be split atomically and a single oversized command response has no such option. |

## Compatibility

The framing applies to **every** message on both characteristics, including
messages that fit in one packet. It is not compatible with firmware or an app
that send bare protobuf: both must be updated together. Protobuf schemas are
unchanged.
