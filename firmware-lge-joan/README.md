# firmware-lge-joan

Firmware for the LG V30 (joan), packaged the way postmarketOS packages firmware
for other devices: an empty-but-for-the-licence parent with one subpackage per
subsystem, modelled on `firmware-fxtec-qx1050` and `firmware-qcom-adreno`.

| package | contents |
|---|---|
| `-adreno-h930` | A540 GPU + zap shader, **H930, US998, H932PR, all non-H932** |
| `-adreno-h932` | A540 GPU + zap shader, **exact LG-H932 only** |
| `-bluetooth` | QCA Bluetooth firmware |
| `-modem` | modem images (cellular) |
| `-adsp` | audio DSP images |
| `-ipa` | IPA images (needed for cellular data) |
| `-wifi` | WLAN firmware and board data |
| `-initramfs` | mkinitfs file list for early GPU/BT firmware |

## Choosing the Adreno package

The two `-adreno-` packages conflict, and there is deliberately no default.
Install exactly one:

```sh
apk add firmware-lge-joan-adreno-h930   # almost everyone
apk add firmware-lge-joan-adreno-h932   # an exact LG-H932, nothing else
```

`LG-H932PR` is **not** an H932 and takes the h930 package. Both packages print
a warning on install, and read the bootloader model from `/proc/cmdline` to
tell you if you picked the wrong one. After install, `joan-firmware-variant
--explain` reports what the system will actually load.

The zap payload is signed per model. Installing the wrong one damages nothing,
but the GPU refuses it and the display does not come up.

Only `a540_zap.mdt` and `a540_zap.b01` actually differ between the two:
`.b00` and `.b02` are byte-identical. What varies is the signature, not the
shader. Each package still ships a complete set, so there is no way to end up
with a half-installed one.

## Where the firmware comes from

Two sources. The GPU, zap and Bluetooth files are redistributable and fetched
from commit-pinned [TheMuppets](https://github.com/TheMuppets) vendor trees.

The modem, ADSP, IPA and WLAN images are not in any vendor tree — they live in
the device's `modem` and `dsp` partitions, and 47 of those 48 files appear
nowhere in TheMuppets' joan repositories. They are hosted in
[`firmware-lge-joan-blobs`](https://github.com/ShapeShifter499/firmware-lge-joan-blobs)
and fetched by pinned commit, the way `sm6115-mainline`, `TheMuppets` and
`FairBlobs` host blobs for other devices.

Integrity is checked twice: the archive by `sha512`, then every file inside
against `MANIFEST.tsv` by `sha256` and size.

## A caveat on the modem images

They were extracted from an **LG-US998**. Modem firmware can be region or
carrier specific, and there is no second dump to compare against, so whether
they are correct for an H930 or H932 is unverified. If cellular misbehaves on a
non-US998 V30, this is the first thing to suspect.

## Licence

`LICENSE` and `NOTICE` are installed to `/usr/share/licenses/firmware-lge-joan/`,
the same files and layout `firmware-qcom-adreno` uses. The Qualcomm licence
grants a limited right to redistribute binary code, conditioned on shipping the
terms file and not removing notices, so both are a condition of the grant.

The A540 zap shader is signed by LG rather than Qualcomm, and these images came
off a retail device rather than from QTI; that is not covered by the above.
