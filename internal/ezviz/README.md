# EZVIZ / Hik-Connect (cloud P2P)

Streams Hikvision / EZVIZ cameras and NVRs through the cloud using the proprietary
P2P protocol — no LAN access, port-forwarding, or RTSP required. Fills the gap
noted in the main README ("Hikvision — a lot of proprietary streaming
technologies"); the existing `isapi:` source is two-way-audio backchannel only.

**EZVIZ and Hik-Connect are the same Hikvision consumer cloud** (Hik-Connect is the
current name; EZVIZ the older/retail brand). One account and this one source cover
both — register the device in whichever app you have, then use the credentials here.

```yaml
streams:
  camera1: ezviz://ACCOUNT:PASSWORD@api.hik-connect.com/SERIAL?channel=1&subtype=main
```

- `ACCOUNT:PASSWORD` — Hik-Connect / EZVIZ account credentials
- `SERIAL` — device serial
- `channel` — 1-based channel (default `1`)
- `subtype` — `main` (full resolution) or `sub` (low-res substream) (default `main`)

`hikconnect:` and `ezviz:` are interchangeable aliases for the same source — use
whichever matches the app you registered the device in.

## Status

Data plane (codec probe → HEVC/H264 NAL handoff into go2rtc) is wired and tested.
The cloud P2P transport (`pkg/ezviz/client.go`) is implemented in a follow-up;
see the responsibilities documented on the `Client` type. The protocol is
credentials-only (no hardcoded keys), HEVC main + sub, live preview.
