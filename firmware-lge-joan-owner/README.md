# firmware-lge-joan-owner

The LG V30 (joan) modem, ADSP, IPA and WLAN firmware.

## Where these come from

[`ShapeShifter499/firmware-lge-joan-blobs`](https://github.com/ShapeShifter499/firmware-lge-joan-blobs),
fetched as a commit-pinned archive. That is how blobs are hosted for other
postmarketOS devices — TheMuppets, FairBlobs, sdm845-mainline and others all
keep firmware in its own repository rather than in the packaging tree.

These images live in the phone's `modem` and `dsp` partitions rather than the
vendor partition, so unlike the GPU and Bluetooth firmware there is nothing
upstream to fetch: of the 48 files, 47 appear nowhere in TheMuppets' joan
trees. Without them there is no cellular, no audio DSP and no WLAN.

## Integrity

Two independent checks. The archive is pinned by `sha512` like any other
source, and every file inside is then verified against `MANIFEST.tsv` by
`sha256` and size. A truncated or swapped blob fails the build and names
itself, rather than producing a package with a hole in it.

## Building

```sh
pmbootstrap build firmware-lge-joan-owner
```

Nothing else is needed — no device, no extraction.

## Licence

Installs Qualcomm's licence and third-party attribution notice to
`/usr/share/licenses/firmware-lge-joan-owner/`, the same files and layout
`firmware-qcom-adreno` uses. The licence's redistribution conditions require
shipping the terms file and leaving notices intact.
