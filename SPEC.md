# H.264 as an OMT video codec — proposal

Status: **proposal**, not adopted upstream. Implemented and tested end to
end (Android sending + OBS receiving) in the `blue-broadcast` ecosystem,
distinct from the proprietary SDK itself (see [README.md](README.md) for
what is and isn't open here).

## Motivation

OMT's native video codec, **VMX1**, is intra-frame and targets a constant
bits-per-pixel rate independent of content. That's the right default for a
high-capacity local link (Ethernet, dedicated Wi-Fi 6): very low
decompression latency, no dependency on a hardware decoder.

On a **shared** Wi-Fi link between multiple sources (the case this targets:
several phones broadcasting at once over the same router), bandwidth
budget becomes the dominant constraint. H.264 with a hardware encoder
(`MediaCodec` on Android) delivers, at comparable perceived quality, a much
lower bitrate than VMX1 for the same content — at the cost of inter-frame
compression (higher decode latency, a dependency on an H.264 decoder on the
receiving end).

This proposal doesn't replace VMX1: it adds H.264 as a **second possible
video codec**, to be negotiated or detected based on receiver capabilities.

## FourCC

```c
constexpr int32_t H264 = 0x34363248;  // ASCII "H264", little-endian
```

Chosen by analogy with the video FourCCs already defined in the OMT
protocol (`VMX1`, `UYVY`, `YUY2`, `NV12`, `BGRA`) — same convention of
4 ASCII characters packed into a little-endian `int32_t`. **Not officially
reserved** by the upstream project; needs confirming or arbitrating before
adoption, to avoid colliding with an existing or future use.

## Frame format

The standard OMT video frame (`FrameHeader`, 16 bytes + `VideoExtHeader`,
32 bytes, existing TCP transport, ports 6400-6600) is reused with no
structural change. Only the payload content and one field differ:

```c
struct VideoExtHeader {
    int32_t codec;         // = H264 FourCC instead of VMX1/UYVY/etc.
    int32_t width;
    int32_t height;
    int32_t frameRateN;
    int32_t frameRateD;
    float   aspectRatio;
    int32_t flags;
    int32_t colorSpace;
};
```

The **payload** is no longer a decompressed image (planar or interleaved)
but an **H.264 Annex B access unit** already compressed by the hardware
encoder — used as-is, with no further re-encapsulation (no MP4/fMP4-style
container, just the start-code-delimited Annex B stream `MediaCodec`
outputs).

This is a **pass-through**: the sender does *no* processing of the H.264
stream beyond wrapping it in the existing OMT header — it never decodes
what it just encoded itself. CPU cost on the sending side is therefore just
that of the hardware encoding.

## Capability negotiation — the most useful angle of this proposal

**The current OMT protocol has no mechanism for a receiver to announce,
before connecting, which video codecs it can decode.** The receiver
discovers the codec actually in use by reading `VideoExtHeader.codec`
**after** getting the first video frame — there's no negotiation step ahead
of that, comparable to the one that already exists for channels
(`<OMTSubscribe video="true" audio="true" .../>`).

Practical consequence observed during implementation: a receiver that
can't decode H.264 (vMix, for example, which only expects VMX1) still
receives the frames and can only silently discard them — no signal back to
the sender, no automatic fallback.

What the reference implementation (the `libomtnet` fork) had to add to
stay robust against this gap:

- Detecting H.264 decoder availability on the receiving end before
  accepting the first frame (`OMTH264Codec.IsAvailable`, which depends on
  FFmpeg being present) — without negotiation, this check happens after
  connecting, not before.
- An explicit fallback on decode failure (`H.264 decoder waiting for a
  keyframe` until the first keyframe arrives), instead of a crash or a
  corrupted stream being shown.

**Open for discussion** (the point where feedback would be most useful):
extending `<OMTSubscribe .../>` with a video capability attribute, for
example:

```xml
<OMTSubscribe Video="true" Audio="true" VideoCodecs="VMX1,H264" />
```

The sender would then pick the first codec in the list it can produce, in
the receiver's order of preference — backward compatible with a receiver
that doesn't send this attribute (current behavior: VMX1 implied).

## Bitrate model (reference implementation)

The target bitrate follows a linear "bits per pixel" model in resolution
and fps, separate from the one used for VMX1:

```
kbps = bpp[quality] × width × height × fps / 1000
```

| Quality | H.264 bpp | VMX1 bpp (reference) |
|---|---|---|
| Low     | 0.08 | 0.20 |
| Medium  | 0.15 | 0.36 |
| High    | 0.24 | 0.65 |

Measured anchors (30 fps): 720p High ≈ 6.5 Mbit/s, 1080p High ≈
15 Mbit/s — versus roughly 18 and 40 Mbit/s for VMX1 at the same settings.
The ratio (~2.5×) reflects H.264's typical inter-frame gain over an
intra-frame codec, at comparable perceived quality on live-camera content
(continuous motion, no graphic animation).

## What this proposal doesn't cover

- No HEVC/AV1: H.264 was chosen for its near-universal availability in
  hardware encoding on Android (`MediaCodec`) and in software/hardware
  decoding on desktop receivers (FFmpeg).
- No dynamic mid-stream renegotiation (hot codec switching): the codec is
  fixed at the start of the broadcast.
- No B-frame support: the reference implementation encodes IPPP (no
  bidirectional frames), to keep decode latency minimal despite the
  inter-frame compression.

## Reference implementation

- **Sending**: proprietary Android SDK (`blue-broadcast/omt-android`, not
  open here) — hardware H.264 `MediaCodec`, pass-through into the OMT frame.
- **Receiving**: MIT fork of `libomtnet` — FFmpeg decoding
  (`OMTH264Codec`), wired into a fork of the official OMT OBS plugin.
- **Demo**: signed Android APK (see [README.md](README.md)), installable
  on a phone to test sending under real conditions against any existing
  OMT receiver.
