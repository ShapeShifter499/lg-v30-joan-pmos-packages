# alsa-ucm-conf-lge-joan

ALSA Use Case Manager profile for the LG V30 (joan).

Signed-off-by: Lance <Gero3977@gmail.com>
Assisted-by: Claude-Code:claude-opus-5
Date: 2026-08-31

## Why this exists

The kernel side of joan's audio works — the card enumerates, `aplay` plays,
and both channels are clean. But PipeWire will not expose a card it has no UCM
profile for, so pmOS Settings shows "dummy output" and `wpctl status` lists no
audio devices at all. This package is the missing userspace half.

## What it does

Two outputs are offered, and neither goes through the WCD9340's analog block,
so the profile routes MultiMedia1 to an MI2S backend rather than the usual
`SLIMBUS_0_RX`:

| device | path | priority |
|---|---|---|
| `Headphones` | `QUAT_MI2S_RX` → **ES9218P "Quad DAC"** → jack | 200 |
| `Speaker` | `TERT_MI2S_RX` → **TFA9872** → loudspeaker | 150 |

Each device enables only its own backend mixer, so selecting one output does
not also feed the other, and each hands the sound server the real hardware
volume control so the desktop slider drives actual attenuation.

Every route and control in the profile is one confirmed by ear on hardware.
The two starting volume values are deliberately conservative choices rather
than measured ones; see the notes below.

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
- **No speaker protection.** mainline's `tfa989x` drives the TFA9872 with the
  CoolFlux DSP bypassed, so SpeakerBoost and the excursion limiter are not
  running. `Speaker Playback Volume` is 0..15; `TDMSPKG` is an *attenuation*
  in hardware (0 = loudest) so the control is registered inverted and the
  scale runs the usual way round, 15 being loudest. The profile comes up at 8
  and lets the user raise it. Keep source levels low.
- **No earpiece device.** joan's earpiece is the WCD9340 EAR PA reached via
  RX INT0. The chain has been made to power up, but nobody has confirmed it
  audible, so it is deliberately not offered here. Add a `SectionDevice` when
  someone has actually listened.

## Related

Kernel work lives in `ShapeShifter499/linux-lg-v30-joan`; the audio bring-up is
documented in `lg-v30-port/docs/`.
