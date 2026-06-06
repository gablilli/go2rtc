# EZVIZ / Hik-Connect cloud P2P wire protocol

Self-contained reference for the protocol `pkg/ezviz` implements. It was
reverse-engineered from Hikvision's `libezstreamclient.so` and the iVMS-4200
desktop client with Ghidra, and validated against live packet captures from a
4K NVR. Opcodes and field tags below are exactly what the code emits and parses.

## V3 control protocol

The control plane ("V3") is a small binary request/response protocol carried in
UDP datagrams. Every message is a 12-byte header followed by an optional body of
TLV attributes (which may be AES-128-CBC encrypted).

### Header (12 bytes, big-endian multi-byte fields)

```
off  size  field
0    1     magic     high nibble 0xE; first byte emitted is 0xE2
1    1     mask      flag bitfield (see below)
2    2     msgType   opcode
4    4     seqNum    sequence number
8    2     reserved  protocol-version constant 0x6234
10   1     headerLen 0x0C (12); larger when an expand header is present
11   1     crc8      CRC-8 over the whole packet with this byte zeroed
12   ..    body      TLV attributes, optionally AES-128-CBC encrypted
```

### Mask byte

```
bit 7  encrypt        body is AES-128-CBC encrypted
bit 6  saltVersion
bit 5..3  saltIndex   3-bit server salt index
bit 2  expandHeader   an expand header follows the 12-byte base header
bit 1  is2BLen        tag 0x07 attributes use a 2-byte length
bit 0  unused
```

### CRC-8

A custom Hikvision bitwise CRC-8 (not a standard polynomial). Computed over the
entire packet with byte 11 zeroed, then written back to byte 11. See `CRC8` in
`v3.go`; the test vectors in `v3_test.go` pin the exact output.

### TLV attributes

```
tag(1) len(1) value(len)                      # normal attributes
tag(0x07) len_be16(2) value(len)              # only when the is2BLen mask bit is set
```

### Encryption

When the encrypt mask bit is set the body is AES-128-CBC with a fixed IV of
`"01234567"` followed by 8 zero bytes. The key is the per-session P2P secret
fetched from the cloud (never hardcoded; the server rotates 8 salt-indexed keys
and returns one per session, identified by `saltIndex`/`saltVersion`).

### Opcodes used by this implementation

| Opcode   | Name           | Role                                                        |
| -------- | -------------- | ----------------------------------------------------------- |
| `0x0B02` | P2P_SETUP      | Register our NAT-mapped address with a P2P server           |
| `0x0B03` | TRANSFOR_CTRL  | Server response carrying the device's stream port           |
| `0x0B04` | TRANSFOR_DATA  | Wrapper used to relay a control message through the server  |
| `0x0C00` | hole-punch req | Sent by the device to open the NAT path to us               |
| `0x0C01` | hole-punch rsp | Our reply (sent 10×) that completes the punch               |
| `0x0C02` | PLAY_REQUEST   | Start the stream (busType/channel/streamType/session)       |
| `0x0C04` | TEARDOWN       | Stop the stream                                             |

### Other opcodes seen in the protocol (not implemented)

These exist in `libezstreamclient.so` but are not needed for cloud streaming, so
the code does not define or send them. Kept here as a reverse-engineering
reference:

| Opcode   | Name            | Purpose (observed)                          |
| -------- | --------------- | ------------------------------------------- |
| `0x0B00` | TRANSFOR_SETUP  | Alternate setup variant                     |
| `0x0B05` | TRANSFOR_DATA2  | Alternate relay-data variant                |
| `0x0C07` | VOICE_TALK      | Two-way audio backchannel                   |
| `0x0C08` | CT_CHECK        | Capability/connection check                 |
| `0x0C0A` | STREAM_CTRL     | In-stream control (pause/seek family)       |
| `0x0C0B` | DATA_LINK       | Data-link negotiation                       |
| `0x0D00` | TRANSPARENT     | Transparent ISAPI passthrough               |
| `0x0D02` | TRANSPARENT2    | Transparent passthrough variant             |

The two SRT control subtypes `0x8003` (DATA_REF / NAK) and `0x8006` (SHORT_ACK /
ACK2) are likewise part of the dialect but are only ever received and ignored,
so they have no dedicated constant.

### Other attribute tags seen in the protocol (not used)

PLAY_REQUEST and the setup messages only populate the tags listed in `v3.go`.
The wider tag space observed during reverse engineering, for reference:

| Tag    | Name             | Tag    | Name             |
| ------ | ---------------- | ------ | ---------------- |
| `0x09` | CT_STEP          | `0x87` | DATA_LINK_VAL    |
| `0x0A` | CT_DATA          | `0x8D` | TRANSPARENT_EXT  |
| `0x79` | STREAM_INFO      | `0xAE` | EXT_PARAM1       |
| `0x7C` | STREAM_PARAM     | `0xAF` | EXT_PARAM2       |
| `0x80` | STREAM_CONTROL   | `0xB4` | OPT_META3        |
| `0x81` | VOICE_ENCODING   | `0xB5` | STREAM_FLAG      |
| `0xB6` | OPT_META4        | `0xB8` | SEARCH_EXT       |

## P2P session flow

```
1. P2P_SETUP (0x0B02) to every P2P server registers our NAT-mapped address
   (the server reads the source of our UDP packet, STUN-style — no public IP
   needed).
2. The 0x0B03 response carries the device's stream port; we pre-punch to it.
3. The device sends a hole-punch request (0x0C00); we reply 0x0C01 ten times.
4. PLAY_REQUEST (0x0C02) is sent two ways (see "Transport mix" below).
5. The device opens an SRT connection and streams media; we ACK and reassemble.
```

### Transport mix: direct vs relayed

The media path is always **direct, hole-punched P2P** — once the NAT punch
completes, SRT media flows device → client over the punched UDP socket. There is
no TCP media-relay fallback in this implementation.

PLAY_REQUEST itself is sent on **two paths** for reliability:

- **Path A — direct:** PLAY_REQUEST (0x0C02) straight to the device over the
  punched socket.
- **Path B — relayed control:** the same request wrapped in a TRANSFOR_DATA
  (0x0B04) message addressed to the P2P server, which forwards it to the device.

Path B only relays the *control* message (a belt-and-suspenders so the device
still receives the play request if the first direct datagram is dropped). Media
never traverses the relay.

## Device SRT dialect

The device speaks a proprietary SRT variant: an induction → conclusion handshake,
then media data packets. It multiplexes **two SRT sub-sessions on one socket** —
a control sub-session (type `0x807f` keepalives) and the video sub-session — with
**independent sequence spaces**. ACKs must be routed by inner payload type so the
control sequence space cannot pollute the video flow-control window; mixing them
stalls the device's sender. See `handleSrtDataPacket` / `sendSrtAck` in
`session.go`.

## Media framing (Hik-RTP)

Media data packets carry a Hik-RTP framing layer. `hikrtp.go` strips the
Hik-RTP/sub headers and reassembles RFC 7798 fragmentation units (FU, NAL type
49) into Annex-B H.265 access units. Interleaved G.711 A-law (PCMA) audio is
demuxed onto a second track. Codec parameters are probed from the live stream.
