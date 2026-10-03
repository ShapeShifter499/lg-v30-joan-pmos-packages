# lg-v30-joan-pmos-packages

postmarketOS packages for the **LG V30 (joan, msm8998)** mainline port.

Signed-off-by: Lance <Gero3977@gmail.com>
Assisted-by: Claude-Code:claude-opus-5
Date: 2026-08-21

These are the bits joan needs that do not belong in the kernel tree and are not
carried by upstream pmaports: proprietary firmware pulled from the vendor
image, and board-specific audio configuration.

## Packages

| directory | package | what it is |
|---|---|---|
| `firmware-lge-joan/` | `firmware-lge-joan` (+ `-h930` / `-h932`) | Shared GPU/BT plus one signing-family package. Fetched from commit-pinned `firmware-lge-joan-blobs`. **No owner tarball.** |
| `alsa-ucm-conf-lge-joan/` | `alsa-ucm-conf-lge-joan` | ALSA UCM profile so PipeWire exposes the sound card instead of a dummy output |
| `joan-imsd/` | `joan-imsd` | 3GPP IMS SIP UA (VoLTE). OpenRC `joan-imsd`, CLI `joan-ims dial` |
| `lge-joan-volte/` | `lge-joan-volte` | First-boot metapackage: MM + 81voltd + rmtfs + calls + joan-imsd |
| `lg-joan-cellular-data/` | `lg-joan-cellular-data` | Cellular-data defaults: rmnet DAD-off udev rule + carrier-agnostic NetworkManager profile |

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

## Building

All of these recipes are already in
[`pmaports-lge-joan`](https://github.com/ShapeShifter499/pmaports-lge-joan)
under `device/testing/`. A clone of that fork plus `pmbootstrap init` /
`install` fetches firmware blobs and builds firmware, UCM, and VoLTE.
There is no copy-in and no `owner-firmware-lge-joan.tar`.

This repo is the working copy. To rebuild one package after editing it here:

```sh
cp -r firmware-lge-joan alsa-ucm-conf-lge-joan joan-imsd lge-joan-volte \
      pmaports-lge-joan/device/testing/   # only if you edited it
pmbootstrap build firmware-lge-joan alsa-ucm-conf-lge-joan joan-imsd
```

Pick the device at `pmbootstrap init`: `joan` pulls `-h930`, `joan-h932` pulls
`-h932`. Do not install both family packages.

## Status

- `firmware-lge-joan` — split by signing family (pkgrel 8). In-tree on
  `pmaports-lge-joan`. This directory is the working copy. Do **not** copy
  `ShapeShifter499/firmware-lge-joan` (retired owner-tarball recipe).
- `alsa-ucm-conf-lge-joan` — in-tree on `pmaports-lge-joan`; device packages
  depend on it. Profile validated with `alsaucm`; PipeWire still needs one
  boot with the package installed.
- `joan-imsd` / `lge-joan-volte` — in-tree; pulled by both device packages.
  See `FIRST-INSTALL-VOLTE.md`.

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
