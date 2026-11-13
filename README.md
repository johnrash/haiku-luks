# LUKS2 disk encryption for Haiku

Patches for [Haiku](https://www.haiku-os.org) that add LUKS2 encrypted
volumes, compatible with Linux's cryptsetup:

- **An encrypted system volume.** Haiku's EFI boot loader asks for the
  passphrase, unlocks the volume and boots from it, and the kernel takes
  the key over without asking again. Installer installs onto an encrypted
  volume as onto any other.
- **Encrypted data volumes** from the desktop:
  - DriveSetup's Partition menu has Encrypt…, Unlock… and Lock;
  - Tracker's Mount menu lists locked volumes, and unmounting one locks it;
  - EncryptedVolume asks for the passphrase.
- **`luks`,** a command line tool to format, inspect, open and close
  volumes and to add and remove passphrases. It also builds on Linux, to
  make encrypted images.
- **Linux interoperability:** the volumes are standard LUKS2
  (AES-XTS-plain64, Argon2id or PBKDF2). cryptsetup opens Haiku's, and
  Haiku opens cryptsetup's.

The code is new and written for Haiku from the LUKS2 specification and the
RFCs. It's all MIT-licensed.

> **Not reviewed by a person.** The crypto and LUKS2 code has passed test
> vectors, interoperability tests with cryptsetup, fuzzing and two AI
> reviews, but no person who knows cryptography has reviewed it. Keep
> backups of anything you put on an encrypted volume.

> **AI-written.** These patches were written with Claude, Anthropic's AI
> model, working for the repository's owner; the commits say so
> (`Co-Authored-By: Claude`). **Haiku does not accept AI-generated
> contributions**, so please don't submit these patches, or code derived
> from them, to Haiku.

> **Not Haiku.** This is not made, endorsed or supported by Haiku, Inc. or
> the Haiku project. "Haiku" and the Haiku logo are trademarks of Haiku,
> Inc.

## Status

- **x86_64, UEFI:** built and tested. Tested in QEMU (OVMF, NVMe disks)
  with scripted boots, GUI tests and RAM searches for leftover keys. On
  current `master` the QEMU tests sometimes hang or stop for reasons
  outside the LUKS code ([`docs/BUILDING.md`](docs/BUILDING.md), end of
  section 4).
  Outside the tests it has run on one laptop (an encrypted install booted
  from its NVMe disk) and in one KVM virtual machine, with the same changes
  on an older Haiku base (late October 2026).
- **arm64:** every changed source compiles, but it hasn't been run. It
  needs an AES accelerator and tests on a board ([`docs/DESIGN.md`](docs/DESIGN.md),
  "What arm64 still needs").
- **BIOS boot:** the BIOS loader can't unlock an encrypted system volume.
  Data volumes work with either loader.

## Getting started

1. [`docs/BUILDING.md`](docs/BUILDING.md): apply the patches to Haiku's
   source, build an image, and run the tests.
2. [`docs/USING.md`](docs/USING.md): encrypt a partition, install Haiku on
   an encrypted volume, the boot prompt, and the `luks` command.
3. [`docs/DESIGN.md`](docs/DESIGN.md): the supported LUKS2 subset, the boot
   flow and key handoff, the driver, the reviews and their fixes, what was
   tested, and the known limits.

## The patches

[`patches/`](patches) holds 15 `git format-patch` files for Haiku's
`master` branch, made on top of `a8694efb7c` (2026-11-12). Together they
change 78 files, most of them new.

| Patch | What |
|---|---|
| 0001 | The LUKS2 library (`src/libs/luks2`): headers, JSON metadata, keyslots, AES (portable and AES-NI), XTS, SHA-2, PBKDF2, Argon2; the `luks` command; host tests against test vectors, OpenSSL and cryptsetup |
| 0002 | `luks_disk`, the kernel driver that publishes an unlocked volume as `/dev/disk/virtual/luks/<n>/raw` |
| 0003 | Booting from an encrypted system volume: the EFI loader's passphrase prompt, the key handoff to the kernel, the EFI watchdog switched off; QEMU boot test |
| 0004 | EncryptedVolume, Tracker's Mount menu, DriveSetup's Partition menu; QEMU GUI and Installer test |
| 0005 | Fixes from the first review: layout and limit checks, wiping, boot volume identity |
| 0006 | Test fix: stop a VM that doesn't power off |
| 0007 | Fixes from a correctness review: ordered and flushed header writes, a lock for changes, no fallback to an older header copy, in-use checks; a power-loss model test |
| 0008 | EFI loader: boot the disk the loader came from, and grow its heap when it runs out |
| 0009 | Test fix: read the whole console line |
| 0010 | Boot loader: a BFS-typed partition that fails to mount is no longer freed while still listed (crash at "no boot path found") |
| 0011 | The "Linux LUKS" partition type; DriveSetup names LUKS2 volumes |
| 0012 | FAT: initialize with the partition's offset as hidden sectors (it was the size) |
| 0013 | Kernel: unmount the volumes it can before shutting down, so FAT volumes are left clean |
| 0014 | Test fix: the EFI loader is in `/boot/system` on current `master` |
| 0015 | DriveSetup: lock the window while scanning the disks (stock `master` dropped into the debugger) |

Patches 0008, 0010, 0012, 0013 and 0015 fix bugs in stock Haiku that the
encryption work ran into, outside the LUKS code. An encrypted install
needs 0010 when its partition is typed "Be File System", and the stick
it's installed from needs 0008 on a machine that has Haiku on another
disk.

## Layout

| Path | What |
|---|---|
| `patches/` | The patches, to apply with `git am` |
| `docs/BUILDING.md` | Applying, building, testing |
| `docs/USING.md` | Using encrypted volumes |
| `docs/DESIGN.md` | Design, reviews, test results, limits |

## Licences

- **This repository's own files** (the docs): MIT, see [`LICENSE`](LICENSE).
- **The patches:** the new files they add are MIT ("Copyright 2026,
  haiku-odroid-c2 port authors."; the work started in an ODROID-C2 port
  of Haiku). Their changes to existing Haiku files are under those files'
  licences, which are MIT.
- **Third-party code:** the AES in `src/libs/luks2/crypto/aes_ct.c` is
  Thomas Pornin's constant-time AES (MIT), as found in Haiku's tree under
  `src/libs/compat/openbsd_wlan/crypto/aes.c`, with its notice kept.
  Nothing else is copied from other projects: cryptsetup is only used as a
  black box in the tests.
