# lg-v30-joan-pmos-packages

postmarketOS packages for the **LG V30 (joan, msm8998)** mainline port.

Signed-off-by: Lance <Gero3977@gmail.com>
Assisted-by: Claude-Code:claude-opus-5
Date: 2026-08-21

These are the bits joan needs that do not belong in the kernel tree and are not
carried by upstream pmaports: proprietary firmware pulled from the vendor
image, and board-specific audio configuration.

**2026-10-03:** packages renamed `lge-joan` -> `lg-joan` to match postmarketOS's
`lg-<codename>` convention; each carries `provides`/`replaces` for its old name, so
`apk upgrade` migrates an installed system. The recipes here are synced from
`device/testing/` of the pmaports fork (branch `joan/readme-build-guide`), which is
where they are built from; edit them there first.

## Packages

| directory | package | what it is |
|---|---|---|
| `firmware-lg-joan/` | `firmware-lg-joan` (+ `-h930` / `-h932`) | Shared GPU/BT plus one signing-family package. Fetched from commit-pinned `firmware-lge-joan-blobs`. **No owner tarball.** |
| `alsa-ucm-conf-lg-joan/` | `alsa-ucm-conf-lg-joan` | ALSA UCM profile so PipeWire exposes the sound card instead of a dummy output |
| `joan-imsd/` | `joan-imsd` | 3GPP IMS SIP UA (VoLTE). OpenRC `joan-imsd`, CLI `joan-ims dial` |
| `lg-joan-volte/` | `lg-joan-volte` | First-boot metapackage: MM + 81voltd + rmtfs + calls + joan-imsd |
| `lg-joan-cellular-data/` | `lg-joan-cellular-data` | Cellular-data defaults: rmnet DAD-off udev rule + carrier-agnostic NetworkManager profile |
| `ffmpeg/` | `ffmpeg` (8.1.2-r6, all `ffmpeg-lib*` subpackages) | FFmpeg with the V4L2 patches Firefox needs to hardware-decode video on the Venus block |
| `nfc-tags/` | `nfc-tags` | GTK4/libadwaita NFC tag reader/writer on neard's D-Bus API (URL and text NDEF records) |

A new pmOS user follows `FIRST-INSTALL-VOLTE.md`.

## Why the audio package exists

joan's kernel audio works — the card enumerates and `aplay` produces sound on
both channels — but PipeWire will not expose a card it has no UCM profile for.
Without this package pmOS Settings shows "dummy output" and `wpctl status`
lists no audio devices at all.

The profile is board-specific because joan is unusual: the WCD9340's analog
outputs are **not wired to a transducer** on this board. The headphone jack is
driven by an ES9218P "Quad DAC" over quaternary MI2S, so the routing points
MultiMedia1 at `QUAT_MI2S_RX` rather than the usual `SLIMBUS_0_RX`.

Validated on hardware:

```
$ alsaucm -c 0 set _verb HiFi list _devices
  0: Headphones
    Headphone jack (ES9218P Quad DAC)
```

Note that UCM matches on the card's **long name** (`LG-V30`), not its id
(`LGV30`) — `alsaucm -c LGV30` will fail while `-c 0` and `-c LG-V30` work.

## Why the cellular-data package exists

The modem and ModemManager bring the T-Mobile (or any carrier) LTE bearer up
fine, but the IPA data netdev (`qmapmux0.0` on `rmnet_ipa0`) never answers
IPv6 neighbor solicitation, so kernel DAD fails, the carrier address stays
`tentative`, and every route stays dead — the desktop shows the
no-connection triangle while `mmcli` reports a connected bearer. The udev
rule turns DAD off on these netdevs at creation; the NM profile
(carrier-agnostic, no APN — NM/MM pick the APN from
mobile-broadband-provider-info) gives a fresh install a data connection out
of the box.

Validated on hardware 2026-10-03 (T-Mobile US, SIM in): profile activates,
IPv6 address clean, 55-67 ms ping to 2001:4860:4860::8888, DNS + HTTP
egress over the carrier.

## Why the ffmpeg package exists

joan's Venus video block is a hardware decoder (H.264 and VP9 verified;
the driver also advertises HEVC, VP8, MPEG-2/4, H.263 and VC-1), exposed by
the kernel as a standard V4L2 mem2mem decoder. GStreamer uses it
out of the box (`v4l2h264dec` and friends outrank `avdec_*`, so Showtime
and anything else built on playbin already decode on Venus). Firefox does
not, even with `media.hardware-video-decoding.force-enabled`. Its V4L2 path
goes through the system FFmpeg and was written against the Raspberry Pi
FFmpeg fork, which does two things stock FFmpeg 8.1 does not:

- **Timestamps.** Firefox sets no timebase on the codec context, so stock
  `h264_v4l2m2m` hands back every frame with `pts=NOPTS`.
  `v4l2-m2m-default-timebase.patch` falls back to V4L2's own microsecond
  unit.
- **dmabuf output.** Firefox wants `AV_PIX_FMT_DRM_PRIME` frames, which are
  exported dmabufs it can hand straight to the GPU. Stock v4l2m2m only
  returns mmap'd NV12, so Firefox decodes one frame, fails with
  `CreateImageV4L2: V4L2 dmabuf allocation error` and switches to software
  for the rest of the video. `v4l2-m2m-drmprime.patch` is Lukas Rusak's
  series as LibreELEC.tv ships it
  (`packages/multimedia/ffmpeg/patches/v4l2-drmprime`, written for 9.0.2,
  applied unmodified).

Measured on hardware 2026-10-06 (US998, Firefox 154, 1080p30, steady
state, 35 s window):

| | dropped frames | decoder CPU | keeps real time |
|---|---|---|---|
| H.264, this package | 1.1% | 0.14 core | yes |
| H.264, stock FFmpeg | 99.5% | 4.43 cores | no (21 s of video per 35 s) |
| VP9, this package | 0.0% | 0.14 core | yes |
| VP9, stock FFmpeg | 99.5% | 3.58 cores | no |

Decoded colours match the reference to within 1/255, and frames reach the
compositor as dmabufs.

Two things to know:

- **Looping or seeking a video needs kernel `linux-lg-joan` r49 or newer**
  (Venus commit "resume decoding on a seek after a completed drain"; not
  on GitHub yet as of 2026-10-06). On older kernels a video plays through once in hardware, then drops every
  frame after the first loop or seek.
- **This package shadows Alpine's `ffmpeg` by pkgrel.** When Alpine ships
  a newer ffmpeg, `apk upgrade` replaces this one and Firefox quietly goes
  back to software decode. Rebase the recipe (copy Alpine's APKBUILD, keep
  the two V4L2 patches) whenever that happens. `about:support` cannot
  tell the two apart (it lists HWDEC either way). Check with
  `apk info -v ffmpeg-libavcodec`, or confirm that Firefox's `RDD Process`
  holds the `qcom-venus-decoder` `/dev/video*` node open while a video plays.

## Building

All of these recipes are already in
[`pmaports-lge-joan`](https://github.com/ShapeShifter499/pmaports-lge-joan)
under `device/testing/`. A clone of that fork plus `pmbootstrap init` /
`install` fetches firmware blobs and builds firmware, UCM, and VoLTE.
There is no copy-in and no `owner-firmware-lge-joan.tar`.

This repo is the working copy. To rebuild one package after editing it here:

```sh
cp -r firmware-lg-joan alsa-ucm-conf-lg-joan joan-imsd lg-joan-volte \
      pmaports-lge-joan/device/testing/   # only if you edited it
pmbootstrap build firmware-lg-joan alsa-ucm-conf-lg-joan joan-imsd
```

Pick the device at `pmbootstrap init`: `joan` pulls `-h930`, `joan-h932` pulls
`-h932`. Do not install both family packages.

## Status

- `firmware-lg-joan` — split by signing family (pkgrel 8). In-tree on
  `pmaports-lge-joan`. This directory is the working copy. Do **not** copy
  `ShapeShifter499/firmware-lge-joan` (retired owner-tarball recipe).
- `alsa-ucm-conf-lg-joan` — in-tree on `pmaports-lge-joan`; device packages
  depend on it. Profile validated with `alsaucm`; PipeWire still needs one
  boot with the package installed.
- `joan-imsd` / `lg-joan-volte` — in-tree; pulled by both device packages.
  See `FIRST-INSTALL-VOLTE.md`. r5 declines incoming INVITEs with 480 (the
  network sends callers to voicemail) instead of answering silently; r6 finds
  the IMS bearer, interface, mux id and WDS service at runtime.
- `nfc-tags` — in-tree; device-lg-joan (r23+) ships it with neard D-Bus
  activation. Adapter discovery and the poll loop are bench-tested; reading
  and writing a physical tag is not yet.
- `ffmpeg` — lives under `temp/ffmpeg` on `pmaports-lge-joan`, not
  `device/testing/` (pushed 2026-10-07).
  Hardware decode verified in Firefox 154 and with the `ffmpeg` CLI
  (`h264_v4l2m2m`, 1800 1080p frames in 16 s vs 37 s in software).

## Related

- Kernel: `ShapeShifter499/linux-lg-v30-joan`
- Port notes and audio bring-up history: `ShapeShifter499/lg-v30-port`

## Firmware redistribution and takedown

This repository is text-only. The proprietary modem, ADSP, IPA and WLAN
firmware is hosted separately in
[`ShapeShifter499/firmware-lge-joan-blobs`](https://github.com/ShapeShifter499/firmware-lge-joan-blobs)
and fetched by commit-pinned URL, the way TheMuppets, FairBlobs and
sdm845-mainline host blobs for other postmarketOS devices.

Those images are extracted from retail device partitions and are hosted because
there is nowhere else to fetch them from: they are not in the vendor trees
TheMuppets publishes, and LG has left the mobile handset market and no longer
distributes them. Hosting them is what makes cellular, audio DSP and WLAN work
without every builder owning a second V30 to dump.

No ownership of this firmware is claimed and no licence to it is granted or
implied. All rights remain with their respective holders.

Both firmware packages install Qualcomm's licence and third-party attribution
notice to `/usr/share/licenses/<package>/`, the same files and layout
`firmware-qcom-adreno` uses. That licence grants a limited right to redistribute
binary code, conditioned among other things on shipping the terms file and on
not removing or obscuring notices, so those files are a condition of the grant
rather than a courtesy. The notice additionally carries attribution for the
open-source code embedded in the firmware — OpenSSL, SSLeay, zlib and others —
whose licences require it on binary redistribution.

What that does not cover: the A540 zap shader is signed by LG rather than
Qualcomm, and these images came off a retail device rather than from QTI.

If you hold rights to any file here and want it removed, open an issue on this
repository or contact the maintainer and it will be taken down. Please say
which files are affected so the rest can keep working.
