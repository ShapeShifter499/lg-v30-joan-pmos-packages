# firmware-lge-joan

postmarketOS/Alpine package recipe for the LG V30 family (`joan`) early-boot
Adreno 540 and WCN3990 Bluetooth firmware.

Status: pre-alpha. This repository contains packaging code and source metadata;
it intentionally does not contain proprietary firmware binaries.

## What the package installs

- Qualcomm A540 GPMU firmware shared by Joan variants.
- LG H930-compatible signed A540 ZAP firmware (used by H930, US998, and other
  non-H932 Joan models).
- LG H932-specific signed A540 ZAP firmware.
- Joan-tested WCN3990 Bluetooth TLV and NVM (`crbtfw21.tlv`, `crnv21.bin`),
  needed before the SD root filesystem is mounted.
- An owner-extracted, hash-verified Joan modem/ADSP/IPA/WLAN tarball kept out
  of Git (`owner-firmware-lge-joan.tar`).
- A postmarketOS mkinitfs file list containing both ZAP sets, both Bluetooth
  files, and the official `firmware-qcom-adreno-a530` PM4/PFP files.
- `joan-firmware-variant`, a diagnostic command that reports the same model
  selection used by the pre-alpha kernel integration.

The two signed sets live at distinct paths:

```text
/usr/lib/firmware/qcom/lge/joan/H930/a540_zap.*
/usr/lib/firmware/qcom/lge/joan/H932/a540_zap.*
/usr/lib/firmware/qca/crbtfw21.tlv
/usr/lib/firmware/qca/crnv21.bin
```

They are not switched with late userspace symlinks. Mainline probes the GPU too
early for that to be race-free. The associated pre-alpha kernel reads the exact
bootloader model from:

```text
androidboot.vendor.lge.model.name=LG-H932
```

Only exact `LG-H932` selects the H932 files; `LG-H932PR` and all other models
select the H930-compatible set. This follows LineageOS's Joan firmware-selection
intent while avoiding its substring match. A recovery override is available:

```text
pmos.joan_firmware_variant=h930
pmos.joan_firmware_variant=h932
```

Reference implementation: LineageOS
[`android_device_lge_joan/releasetools/device_check.sh`](https://github.com/LineageOS/android_device_lge_joan/blob/0053e5025795da63f8aa94ce86bd831a1004ca4a/releasetools/device_check.sh).

## Firmware provenance

The remotely available GPU/BT files are hash-pinned to the established
LineageOS vendor repositories maintained by TheMuppets:

- `proprietary_vendor_lge_joan` commit
  `2489e95801110695e991394245b6d7ae66670c4e`
- `proprietary_vendor_lge_joan-common` commit
  `274a1e49a783b971eb967f6afd303b3ec38c9a1b`

`firmware-sources.tsv` records each remote URL, size, and SHA-256 digest.
`owner-firmware-sources.tsv` records all owner-extracted modem/ADSP/IPA/WLAN
inputs by source path, destination path, size, and SHA-256. The deterministic
owner tarball stays Git-ignored. The APKBUILD carries SHA-512 checksums for all
package inputs.

No explicit redistribution license was found for the vendor firmware. The
firmware and owner extraction tarball remain proprietary and are not covered by
this repository's MIT license. This recipe grants no redistribution rights;
each builder is responsible for obtaining device firmware lawfully and for any
redistribution decision.

## Verify and prepare source inputs

```sh
./scripts/fetch-firmware.sh
./scripts/import-owner-firmware.sh PREPARED-SOURCE-DIRECTORY
./tests/test-selector.sh
```

The remote fetch and local import both fail closed on size/hash mismatches. The
importer refuses to overwrite existing output and creates the deterministic
ignored `owner-firmware-lge-joan.tar` expected by the APKBUILD.

## Use in pmaports

Copy the package files into a pmaports checkout:

```sh
./scripts/install-into-pmaports.sh /path/to/pmaports
pmbootstrap checksum firmware-lge-joan
pmbootstrap build firmware-lge-joan
```

The Joan device package should depend on `firmware-lge-joan-initramfs` so both
GPU variant sets and the early Bluetooth files are available before the SD
root filesystem is mounted.

## Licensing

The original scripts, tests, and package metadata in this repository are MIT
licensed. Downloaded firmware is proprietary and excluded from Git. Source
projects and copyright holders retain all rights to their respective files.
