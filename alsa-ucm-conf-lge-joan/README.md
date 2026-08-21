# alsa-ucm-conf-lge-joan

ALSA Use Case Manager profile for the LG V30 (joan).

Signed-off-by: Lance <Gero3977@gmail.com>
Assisted-by: Claude-Code:claude-opus-5
Date: 2026-08-21

## Why this exists

The kernel side of joan's audio works — the card enumerates, `aplay` plays,
and both channels are clean. But PipeWire will not expose a card it has no UCM
profile for, so pmOS Settings shows "dummy output" and `wpctl status` lists no
audio devices at all. This package is the missing userspace half.

## What it does

joan does **not** use the WCD9340's analog outputs; they are not wired to a
transducer on this board. The headphone jack is driven by an **ES9218P "Quad
DAC"** fed over **quaternary MI2S**. So the profile routes MultiMedia1 to
`QUAT_MI2S_RX` rather than the usual `SLIMBUS_0_RX`, and hands the sound server
the DAC's own volume and switch controls so the desktop slider drives real
hardware attenuation.

Every control and value in the profile is one verified working on hardware.

## Matching

ALSA looks up `conf.d/<card driver>/<card long name>.conf`. On joan:

```
 0 [LGV30          ]: sdm845 - LG-V30
                      LG-V30
```

driver `sdm845`, long name `LG-V30` → `conf.d/sdm845/LG-V30.conf`.

## Notes and limits

- **Volume ceiling.** `Headphone Playback Volume` is 0..255 where 255 is 0 dB,
  and 0 dB into headphones is uncomfortably loud. The profile comes up at 195
  (about -30 dB) and lets the user raise it.
- **No jack detection.** The card exposes a "Headphone Jack" control, but that
  belongs to the WCD9340's MBHC and does not track the ES9218P path. Wiring
  `JackControl` to it would let UCM hide the device when the jack reports
  absent, which would be wrong here, so it is deliberately omitted. The
  headphone device is always present for now.
- **Fixed 48 kHz.** The MI2S bit clock is hardcoded to 1.536 MHz in the machine
  driver, i.e. 48 kHz stereo 16-bit. PipeWire resamples other rates
  transparently, so this is not user-visible, but it should become
  rate-dependent eventually.
- **No loudspeaker device.** joan's speaker is a TFA9872 on quaternary TDM /
  i2c_7 which has no mainline driver yet. When that lands it gets its own
  `SectionDevice` here.

## Related

Kernel work lives in `ShapeShifter499/linux-lg-v30-joan`; the audio bring-up is
documented in `lg-v30-port/docs/`.
