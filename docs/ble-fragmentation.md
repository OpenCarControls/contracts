# BLE fragmentation

Wire-level contract for sending messages larger than one ATT packet over the
Open Car Control GATT service. The values live in
`opencar/core/v1/transport.toml` (`[ble.payload]` and `[ble.fragmentation]`);
this page explains them. Firmware and app must both follow it.

## Mechanism: a two-byte chunk header and a CRC

Every write (`app_to_device`) and every notification (`device_to_app`) carries
exactly **one chunk**: a 2-byte header followed by a slice of the **message
stream**. The message stream is the encoded `AppToDevice` / `DeviceToApp`
followed by a 4-byte CRC-32 of the encoded message. The receiver concatenates
the slices, checks the CRC and decodes the message once the last chunk has
arrived.

```
  byte 0                  byte 1
  bit  7      6 .. 0
     +------+-------------+-----------+-------------------------+
     | LAST | CHUNK_INDEX |  MSG_SEQ  | data (≤ chunk_data)     |
     +------+-------------+-----------+-------------------------+
        header (2 bytes)

  message stream = encoded message || CRC-32(encoded message), big-endian
```

- `CHUNK_INDEX` (7 bits) is `0` for the first chunk of a message and increments
  by one for every following chunk. It never wraps: a message has at most 128
  chunks.
- `LAST` is `1` on the final chunk. A message that fits in one chunk is a single
  chunk with `LAST` set and index `0`.
- `MSG_SEQ` (8 bits) is the same on every chunk of a message. The sender keeps
  one counter per direction, starts it at any value on connect and increments
  it (mod 256) for every message. The receiver does **not** require messages to
  be consecutive (lost messages are expected); it only requires every chunk of
  one message to carry the sequence number of its chunk 0.
- The CRC is CRC-32/ISO-HDLC (the zlib / IEEE 802.3 CRC-32: polynomial
  `0x04C11DB7` reflected, init and final xor `0xFFFFFFFF`; the check value of
  `"123456789"` is `0xCBF43926`), computed over the encoded message only, and
  sent big-endian at the end of the stream. The 4 bytes may be split across
  the last two chunks like any other data.
- Chunk data size is `negotiated_ATT_MTU - 3 (ATT header) - 2 (chunk header)`.
  With the requested MTU of 247 that is 242 bytes. The receiver does not need
  to know the chunk size: it only looks at the headers.
- Chunks of one message are sent back to back; chunks of two messages are never
  interleaved on a characteristic.
- Both characteristics use the same framing. Only device to app is implemented
  first, but app to device (e.g. OTA) is covered by the same format.

Examples (MTU 247): a 3-byte message `AA BB CC` is one chunk,
`80 <seq> AA BB CC <crc32 ×4>`. A 500-byte message has a 504-byte stream, so
three chunks with headers `00 <seq>`, `01 <seq>` and `82 <seq>` carrying 242,
242 and 20 stream bytes.

### Limits

- `ble.payload.max_message_bytes` = **4096**: the largest reassembled message.
  Sized for the ESP32-C3 (400 KB SRAM shared with the BLE host and the cellular
  modem stack): one reassembly buffer per direction costs 4 KiB, and it is
  well above today's E-GMP state (~244 bytes) with room for growth. The CRC is
  not counted in it. Raising it is a contract change.
- The effective limit for a connection is
  `min(max_message_bytes, 128 * chunk_data_bytes - crc_bytes)`. The app must therefore
  negotiate an ATT MTU of at least `ble.fragmentation.min_att_mtu` (64);
  below that, 128 chunks could not carry 4096 bytes.
- A sender must never produce a message above the limit. A receiver drops a
  message that grows beyond it.

### Error handling

BLE notifications and writes are ordered and checked by the link layer, so
corruption is not expected; chunks can only be lost when a stack queue
overflows (notifications) or a write-without-response is dropped, or when the
link drops. The sequence number and the CRC catch the case where lost chunks
would otherwise let pieces of two messages be glued together. No
retransmission exists. The receiver keeps at most one partial
message per characteristic and **discards it** when:

1. The link disconnects (reassembly state is also cleared on connect).
2. A chunk arrives with an index other than `previous + 1` (a gap or a
   duplicate), or a chunk with index > 0 arrives with no partial message.
   The receiver then ignores chunks until the next chunk with index 0.
3. A chunk with index 0 arrives while a partial message exists: the old
   partial is dropped and a new message starts.
4. A chunk with index > 0 carries a different `MSG_SEQ` than chunk 0 of the
   partial message.
5. No further chunk arrives within `ble.fragmentation.reassembly_timeout_ms`
   (2000) of the previous one.
6. The reassembled message would exceed `max_message_bytes`, or the stream is
   shorter than the CRC.
7. The CRC does not match the message.
8. The reassembled bytes fail to decode.

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
| **Chunk header per notification** (chosen) | Works on any GATT stack and both characteristics, no new GATT objects, 2 bytes overhead per ATT packet plus a 4-byte CRC per message, trivial to implement and test on the ESP32-C3 and in Dart. |
| L2CAP connection-oriented channel | Gives a stream and flow control, but support differs per platform and mobile BLE plugin (not verified for the app's plugin), it adds a second pairing/permission path and a new PSM contract, and it is overkill for ~1 KB messages. Could be added later for bulk transfers. |
| Bigger ATT MTU only | ATT caps a value at 512 bytes and phones often negotiate less; it does not remove the limit, only moves it. |
| Fragment inside protobuf (chunk fields in the envelope) | Would force the envelope to be decodable before it is complete, and chunk bookkeeping would live in every message type. |
| Multiple smaller state messages | Already possible through optional-field partial updates, but a single logical update can still not be split atomically and a single oversized command response has no such option. |

## Compatibility

The framing applies to **every** message on both characteristics, including
messages that fit in one packet. It is not compatible with firmware or an app
that send bare protobuf: both must be updated together. Protobuf schemas are
unchanged.
