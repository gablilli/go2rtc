# EZVIZ / Hik-Connect (cloud P2P)

Streams Hikvision/EZVIZ cameras and NVRs through the Hik-Connect cloud using the
proprietary P2P protocol — no LAN access, port-forwarding, or RTSP required. Fills
the gap noted in the main README ("Hikvision — a lot of proprietary streaming
technologies"); the existing `isapi:` source is two-way-audio backchannel only.

```yaml
streams:
  camera1: ezviz://ACCOUNT:PASSWORD@api.hik-connect.com/SERIAL?channel=1&subtype=main
```

- `ACCOUNT:PASSWORD` — Hik-Connect account credentials
- `SERIAL` — device serial
- `channel` — 1-based channel (default `1`)
- `subtype` — `main` or `sub` (default `main`)

`hikconnect:` is an alias for `ezviz:`.

## Status

Data plane (codec probe → HEVC/H264 NAL handoff into go2rtc) is wired and tested.
The cloud P2P transport (`pkg/ezviz/client.go`) is implemented in a follow-up;
see the responsibilities documented on the `Client` type. The protocol is
credentials-only (no hardcoded keys), HEVC main + sub, live preview.
