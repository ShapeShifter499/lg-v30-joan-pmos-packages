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
| `firmware-lge-joan/` | `firmware-lge-joan` | GPU, Bluetooth, modem, ADSP, IPA and WLAN firmware, fetched from the vendor blobs at build time |
| `alsa-ucm-conf-lge-joan/` | `alsa-ucm-conf-lge-joan` | ALSA UCM profile so PipeWire exposes the sound card instead of a dummy output |

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

## Building

Each directory is a standard Alpine `APKBUILD`. With pmbootstrap:

```sh
pmbootstrap build --src=. alsa-ucm-conf-lge-joan
```

or point pmaports at this tree as an extra aports directory.

## Status

- `firmware-lge-joan` — in use, also maintained standalone at
  `ShapeShifter499/firmware-lge-joan`. Copied here so joan's packages live
  together; the standalone repo is unchanged.
- `alsa-ucm-conf-lge-joan` — profile validated with `alsaucm`; not yet
  confirmed end to end through PipeWire, because the running pmOS image is an
  Alpine/OpenRC rootfs with no `systemctl`, so the sound server was never
  restarted to pick it up. Needs one boot with the package installed.

## Related

- Kernel: `ShapeShifter499/linux-lg-v30-joan`
- Port notes and audio bring-up history: `ShapeShifter499/lg-v30-port`
