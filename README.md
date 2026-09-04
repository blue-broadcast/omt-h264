# omt-h264

An unadopted proposal to add **H.264** as a second video codec to the
[OMT (Open Media Transport)](https://github.com/openmediatransport) protocol,
alongside VMX1 — aimed at shared Wi-Fi links where bandwidth budget is the
dominant constraint (e.g. several phones broadcasting at once over the same
router).

**This repo is documentation, not code.** It describes the frame format,
the FourCC, the bitrate model, and above all the capability negotiation
question that the current protocol leaves unanswered — see [SPEC.md](SPEC.md)
for the full technical detail.

## Why there's no code here

The sending implementation lives in a proprietary Android SDK
(`blue-broadcast/omt-android`), which isn't a public repo. What this repo
gives instead:

- the frame format specification (enough to reimplement either the sending
  or receiving side independently),
- a signed **release APK** of the demo app (`blue-broadcast/sample-app`) to
  test H.264 sending under real conditions, without access to the SDK code,
- the receiving side, on the other hand, **is** open: an MIT fork of
  `libomtnet` plus a GPL-2.0 fork of the OMT OBS plugin, both in the
  `blue-broadcast` organization.

## Trying it out

1. Grab the latest release of this repo (APK).
2. Install it on an Android phone (minSdk 24).
3. In the app's settings, choose the **H.264** codec and a resolution/quality.
4. Start broadcasting — the source shows up as a standard OMT source on the
   local network.
5. On the receiving end: OBS with the forked OMT plugin
   (`blue-broadcast/omtplugin` + `blue-broadcast/libomtnet`) decodes the
   H.264 stream. A receiver that only knows VMX1 (vMix, for example) will
   see the source but won't be able to decode it — that's exactly the
   capability negotiation gap [SPEC.md](SPEC.md) documents.

## Links

- [blue-broadcast/libomtnet](https://github.com/blue-broadcast/libomtnet) — H.264 receiving (MIT)
- [blue-broadcast/omtplugin](https://github.com/blue-broadcast/omtplugin) — OBS plugin (GPL-2.0)
- [blue-broadcast/omt-android](https://github.com/blue-broadcast/omt-android) — sending SDK (proprietary, not public)

## License

This repo's text (README, SPEC) is CC-BY licensed. The APK distributed in
releases stays governed by the proprietary SDK's license — see the terms
inside the app itself.
