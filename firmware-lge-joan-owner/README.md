# firmware-lge-joan-owner

The LG V30 (joan) modem, ADSP, IPA and WLAN firmware, extracted by the owner of
the device.

## Why this is separate

`firmware-lge-joan` carries the redistributable set: the A540 GPU firmware, the
zap shader payloads and the Bluetooth files, all fetched from commit-pinned
[TheMuppets](https://github.com/TheMuppets) vendor trees. Anyone can build it.

The files in *this* package cannot be fetched. Each builder extracts them from
their own device. Keeping them in a separate package means the GPU and
Bluetooth firmware still builds for someone who has not done that — which was
not true when both sets lived in one package.

## What you need

`owner-firmware-lge-joan.tar`, beside this `APKBUILD` or pointed at by
`JOAN_OWNER_TAR`. It is not in this repository and must not be committed.

`owner-firmware-sources.tsv` lists every file the tarball must contain, with
its path inside the tar, where it is installed, its size and its **sha256**.
Every file is verified against that manifest before packaging; a missing or
altered file fails the build and names itself.

## Why the tarball itself is not checksummed

A tar's hash depends on file ordering, timestamps and uid, so two people
extracting identical firmware get different tarballs. Pinning one would pin one
person's extraction and make the package unbuildable by everyone else — which
is exactly what happened before this split. The payload is verified per file
instead, which is reproducible across builders.

## Building

```sh
cp /path/to/owner-firmware-lge-joan.tar firmware-lge-joan-owner/
pmbootstrap build firmware-lge-joan-owner
```

Signed-off-by: Lance <Gero3977@gmail.com>
Assisted-by: Claude-Code:claude-opus-5
Date: 2026-08-31
