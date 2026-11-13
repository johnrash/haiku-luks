# LUKS2 disk encryption for Haiku: design

What the patches do, and why they do it that way. The work came in
stages:
- an encrypted system volume that the boot loader unlocks with a
  passphrase (x86_64, tested in QEMU), patches 0001-0003;
- encrypted volumes from Haiku itself (EncryptedVolume, Tracker,
  DriveSetup, Installer), patch 0004;
- the fixes from two reviews, patches 0005 and 0007;
- the fixes from the first tries on real hardware, patches 0008, 0010 and
  0011;
- two fixes outside the LUKS code, found on the way, patches 0012 and 0013;
- three test script fixes, patches 0006, 0009 and 0016;
- two changes for current Haiku `master`, patches 0014 and 0015.

The common code is written so that arm64, which boots through the same
EFI loader, can follow without changes to it.

Licensing: all new files are MIT ("Copyright 2026, haiku-odroid-c2 port
authors."; the work started in an ODROID-C2 port of Haiku). The LUKS2 code
is written from:
- the published LUKS2 on-disk format specification (Milan Brož, "LUKS2
  On-Disk Format Specification");
- Argon2 from RFC 9106, BLAKE2b from RFC 7693, PBKDF2 from RFC 8018;
- SHA-2 from FIPS 180-4, XTS from IEEE 1619.

No cryptsetup source was read or copied; cryptsetup is used only as a
black box, to make reference volumes for the tests. The only reused code is
Thomas Pornin's constant-time AES (MIT, already in Haiku's tree under
`src/libs/compat/openbsd_wlan/crypto/aes.c`).

**The crypto code has had two reviews, both by AI agents (see "Review"
and "Correctness review"), and none by a person who knows cryptography.
Keep backups of anything you put on an encrypted volume.**

## The LUKS2 subset

Supported (anything else is refused, never guessed):

| Part | Supported | Notes |
|---|---|---|
| Binary header | both copies, `sha256` checksum, header sizes 16 KiB to 4 MiB | the valid copy with the higher `seqid` wins; a bad primary falls back to the secondary |
| JSON metadata | `keyslots`, `segments`, `digests`, `config`, `tokens` (kept, not used) | a small JSON parser/serialiser; unknown fields survive a rewrite |
| Segment | one `crypt` segment, `aes-xts-plain64`, 256- or 512-bit key, sector size 512 to 4096, `iv_tweak`, size `dynamic` or fixed | no `integrity`, no reencryption (`config.requirements.mandatory` must be empty) |
| Keyslot | type `luks2`, area `raw` with `aes-xts-plain64`, `priority` 0/1/2 | |
| KDF | `argon2id`, `argon2i`, `pbkdf2` (sha256, sha512) | Argon2 version 0x13 |
| Anti-forensic split | `luks1` AF with `sha256` or `sha512` | |
| Digest | `pbkdf2` digest over the volume key, constant-time compare | |

The IV is plain64: the little-endian sector number in 512-byte units from
the segment start plus `iv_tweak`. That holds for 4096-byte sectors too
(no `iv_large_sectors`), as dm-crypt does for LUKS2. The keyslot areas
always use 512-byte sectors, with IV 0 at the area's start.

Volumes this code creates look like cryptsetup's defaults:
- 16 KiB headers and a 16 MiB data offset;
- keyslots 0-31;
- `aes-xts-plain64` with a 512-bit key;
- `argon2id`, and the `luks1` AF with 4000 stripes;
- sector size 512 (4096 on request).

Linux opens them with `cryptsetup open`, and Haiku opens Linux's.

## Where each part lives, and why it's portable

```
headers/private/luks2/            interfaces: luks2.h, luks2_crypto.h, luks2_json.h,
                                  luks2_driver.h (the driver's ioctls and boot module)
src/libs/luks2/                   common code, no OS or CPU assumptions
    luks2_read.cpp                headers, metadata, keyslot unlock, key check
    luks2_write.cpp               format, add and remove keyslots (tools only)
    luks2_util.cpp, json.cpp      base64, AF merge, KDF dispatch; JSON
    luks2_partitions.cpp          recognising LUKS2 partitions, the driver's ioctls (no crypto)
    luks2_userland.cpp            random numbers, KDF choice (luks, EncryptedVolume)
    crypto/aes_ct.c               Pornin's constant-time AES (bitsliced, 32-bit words)
    crypto/aes_xts.cpp            AES dispatch to an accelerator, XTS
    crypto/sha2_hmac.cpp          SHA-256/512, HMAC, PBKDF2
    crypto/argon2.cpp             BLAKE2b, Argon2
    arch/x86/aes_ni.cpp           optional AES-NI accelerator (x86_64)
src/add-ons/kernel/drivers/disk/virtual/luks_disk/   kernel driver
src/system/boot/loader/luks.cpp   boot loader: detect, prompt, decrypting disk node
src/system/kernel/fs/vfs_boot.cpp kernel: open the boot volume with the loader's key
src/bin/luks/                     command line tool (also built for the build host)
src/apps/encryptedvolume/         EncryptedVolume: unlock, encrypt, lock
src/tests/system/luks2/           host, QEMU and arm64 compile tests (README.md)
```

- **Byte order and types:** the common code reads and writes all on-disk
  integers byte by byte (big-endian header, little-endian IVs). It never
  casts unaligned buffers to wider types, uses `uint32`/`uint64`
  explicitly, and has no inline assembly. It needs `memcpy`, `memset`,
  `strcmp`, a heap, and an "allocate large" hook (for Argon2's memory);
  each environment (kernel, loader, userland, build host) supplies those.
- **The cipher interface:** crypto sits behind a small block-cipher
  interface, `luks2_aes`, with `set_key`, `encrypt_blocks` and
  `decrypt_blocks`. XTS calls it with as many blocks as it has (a whole
  sector), so a bitsliced or pipelined implementation can work on several
  blocks at once.
- **Accelerators:** at key setup an accelerator is picked if the CPU has
  one.
  - On x86 that's AES-NI (CPUID leaf 1, ECX bit 25). It's compiled only for
    x86 targets, with `__attribute__((target("aes,sse2")))`, so the rest of
    the file needs no special flags.
  - arm64 would add `arch/arm64/aes_ce.cpp` (AESE/AESD/AESMC/AESIMC,
    detected through `ID_AA64ISAR0_EL1.AES`); nothing else changes.
  - The constant-time portable AES is always there as the fallback.
- **SIMD in the kernel:** the x86_64 kernel saves the FPU/SSE state on
  every interrupt and kernel entry (`arch/x86/64/interrupts.S`), so kernel
  code may use SSE and AES-NI. arm64 needs the same check before it uses
  NEON in the kernel (see "What arm64 still needs").
- **The loader code** uses only the loader's portable APIs:
  `Node`/`Partition`, `console_wait_for_key()`, `printf()` and `malloc()`.
  Large allocations go to `platform_allocate_heap_region()`, which is EFI's
  `AllocatePages()` on every EFI platform. Nothing is in
  `platform/efi/arch/*`.

## Boot flow and the key handoff

1. **Scan:** the loader scans partitions as before. A partition with no
   partition table and no file system the loader knows, but with a valid
   LUKS2 header, is kept as a *locked volume* instead of being dropped.
2. **Prompt:** if no boot volume was found and there are locked volumes,
   the loader shows the volume's name and asks
   `Passphrase (Esc for the boot menu):`.
   - The prompt has no echo (a `*` per character) and takes Backspace.
   - The loader runs the KDF and tries every keyslot by priority.
   - The boot menu's "Select boot volume" also lists
     "Unlock encrypted volume …" items.
3. **Unlock:** on success it wraps the partition in a decrypting `Node`
   (`EncryptedDisk`, read-only) and scans that like a disk. The BFS volume
   inside is mounted and becomes the boot volume, and the loader reads the
   kernel and modules through it.
4. **The boot volume message:** `register_boot_file_system()` computes the
   disk identifier (size and checksums) on the *decrypted* device, so the
   kernel's existing `DiskBootMethod` recognises it. The loader also adds
   the volume's UUID, offset and size and its key to the boot volume
   message.
   - The message is a `KMessage` rather than a new `kernel_args` field.
     Old kernels ignore its unknown fields, so they still boot from the new
     loader, and old loaders still work with the new kernel.
   - A new `kernel_args` field would make an older kernel stop with
     "Version mismatch", for example when someone boots an older package
     state.
5. **The kernel:** `vfs_mount_boot_file_system()` copies the key out of
   the message and wipes it in place before anything dumps the message.
   After the first device scan it:
   - finds the partition with that UUID, offset and size;
   - asks the `luks_disk` driver's boot module to open it with the key (the
     driver checks the key against the header's digest);
   - rescans, and finds the boot partition by the usual disk identifier and
     partition offset;
   - wipes its own copy of the key.

   There's no second passphrase prompt.
6. **Wiping in the loader:** the passphrase buffer, the KDF output, the
   AF-split keyslot material and Argon2's memory are wiped as soon as a
   keyslot has been tried. Right after the boot message is copied to the
   kernel args, the loader wipes:
   - the volume key and the decrypting node's AES key schedules;
   - the key field in its own copy of the boot message.

   The key is the last field added to the message. A field added after it
   can make the message grow, and the old buffer would then be freed with
   the key in it; the RAM search found exactly that. Nobody else clears
   the EFI loader's memory: it becomes free memory for the kernel.

The volume key then lives only in the kernel driver's AES key schedules, as
with dm-crypt. Kernel memory is not swapped.

## The kernel driver

`drivers/disk/virtual/luks_disk` is a device manager driver on the generic
bus, like `ram_disk`, and is in the boot module set:

- **`/dev/disk/virtual/luks/control`:** ioctls to open a volume (a
  partition path, the volume key, a read-only flag), to close one (by
  device number), and to list the open devices.
- **`/dev/disk/virtual/luks/<n>/raw`:** the decrypted device. Its block
  size is the segment's sector size, and its size is the segment's.
  - Reads and writes go to the underlying partition, which the driver
    keeps open, through a fixed pool of bounce buffers, so the page writer
    never needs an allocation.
  - Unaligned access is handled by read-modify-write under a lock.
  - `B_FLUSH_DRIVE_CACHE` is passed down. Discards are not, since they would
    reveal which blocks are in use.
- **Keys:** the driver checks every key it's given against the LUKS2
  digest. It doesn't run Argon2: userland and the loader do.
- **Removable:** the decrypted device says it's removable, and
  `B_EJECT_DEVICE` locks it (see "Haiku integration").
- **No special handling elsewhere:** the disk device manager picks the
  device up like any disk, so `mount`, DriveSetup and Tracker's Mount menu
  see the BFS volume inside once it's open.

## Argon2 memory in the loader

Argon2 needs `memory` KiB: cryptsetup's default is up to 1 GiB, and at most
half the RAM of the machine that formatted the volume.
- **Allocation:** the loader allocates one block per lane through
  `malloc()`, which goes to EFI `AllocatePages()`. cryptsetup uses up to 4
  lanes, so that's 256 MiB at most per allocation.
- **Speed:** the loader computes the lanes one after another, since it has
  no threads; the result is the same. So it takes about `lanes` times as
  long as a multi-threaded cryptsetup.
- **Too little memory:** if the memory can't be allocated, the loader says
  so and skips that keyslot.

`luks format` and `luks addkey` take at most 1 GiB or a quarter of the RAM,
4 lanes and at least 4 passes. They measure Argon2 single-threaded on the
machine they run on, and use less memory when 4 passes take longer than
the target (2 s, `--iter-time`). So a volume formatted on a small machine
(2 GiB of RAM, a Cortex-A53) gets a keyslot its loader can open in about
2 s. A volume meant for such a machine but formatted on a faster one should
use `--memory 262144` or less.

The EFI loader switches off the firmware's 5 minute watchdog at start
(`platform/efi/start.cpp`), as GRUB and systemd-boot do. Without that, the
firmware resets the machine when someone takes longer than that at the
passphrase prompt or in the boot menu.

## Swap

Haiku's swap file is `/var/swap` on the boot volume, so on an encrypted
system volume it's encrypted with it; there's no separate swap partition to
set up. Haiku has no hibernation.

## Randomness when creating volumes

Only `luks format` and `luks addkey` (and EncryptedVolume's Encrypt) need
random numbers: for the volume key, the salts, the AF stripes and the UUID.
- On Haiku they read `/dev/urandom` (the `random` driver, seeded from
  RDRAND/RDSEED on x86_64 and from timing jitter elsewhere), and fail on a
  short read.
- On the build host they use `getrandom()`.
- The loader and the kernel never create keys.

## Haiku integration

Everything is in userland except one driver change; there's no kernel
disk-system add-on. Such an add-on would make the disk device manager call
a LUKS2 partition a file system. mount_server's "mount all" and DriveSetup's
Mount would then try to mount it and fail. Instead the programs recognise
LUKS2 by its signature, as DriveSetup already did for its "LUKS encrypted"
label and lock icon.

- **EncryptedVolume** (`src/apps/encryptedvolume`, in `/boot/system/apps`)
  has three modes:
  - `EncryptedVolume <partition>` asks for the passphrase, with a
    Read-only option. It runs the KDF in a worker thread, opens the volume
    through the driver, mounts what's inside and opens it in Tracker. A
    wrong passphrase is reported in the window, which stays open.
  - `--encrypt <partition>` asks for a volume name and the passphrase
    twice, and asks for confirmation. Then it formats LUKS2 (Argon2id
    measured for 2 s on this machine), opens the volume, puts BFS on it
    through the disk device manager, and mounts it. Where the partitioning
    system has the "Linux LUKS" type (GPT, MBR), it gives the partition
    that type.
  - `--lock <partition>` unmounts and locks, without a window.
- **Locking on unmount:** mount_server ejects a device after its last
  volume was unmounted (Tracker's Unmount; "eject when unmounting" is on by
  default). Since the decrypted device is removable and eject locks it,
  unmounting an encrypted volume in Tracker locks it, with no change to
  Tracker or mount_server. The same flag makes mount_server mount the volume
  when it's unlocked. `unmount` on the command line leaves it unlocked;
  `luks close` locks it.
- **Tracker's Mount menu** lists each locked LUKS2 partition as
  `Unlock "label" (size, encrypted)…`, which starts EncryptedVolume. Once
  it's unlocked, the volume inside is listed like any other.
- **DriveSetup:**
  - The Partition menu has "Unlock…", "Lock" (not for the volume the system
    booted from) and "Encrypt…" (for a partition that has no partition
    table and isn't in use), all through EncryptedVolume.
  - Its list shows "LUKS2 encrypted" under File system and the header's
    name under Volume name, also when only the secondary header is intact.
  - The Partition type column shows the type itself.
- **The "Linux LUKS" partition type:** GPT GUID
  `CA7D7CCB-63ED-4C53-861C-1742536059CC` (Linux's LUKS type: `sgdisk`
  code 8309, systemd's discoverable partitions) and MBR type 0xe8 are in
  Haiku's partition type lists. Encrypt… sets it, so no boot loader takes
  the volume for a file system of the type it had before.
- **Installer** needed no change. An unlocked volume is a valid target, it
  installs onto it, and its "Install EFI loader" puts the loader on the
  ESP. Encrypting the target first is DriveSetup's "Encrypt…" (Installer's
  "Set up partitions…" opens DriveSetup). The next boot asks for the
  passphrase.

Passphrases typed into EncryptedVolume pass through BTextView's buffer,
which isn't wiped; the copy the app works on is.

## Review

A second AI agent, given only the code, reviewed it after patch 0003. It
checked the code against the specification and the RFCs, and fuzzed the
header parsing and the keyslot writer under ASan/UBSan:
- 860,000 mutated headers (from cryptsetup and its own images);
- 24,000 add/remove rounds;
- 2 million raw JSON and base64 inputs.

It found no memory-safety bug, no leak and no crash. It confirmed:
- the crypto against the standards: SHA-2, HMAC/PBKDF2, BLAKE2b, Argon2,
  XTS with 4096-byte sectors, the AF split, the AES-NI key schedule;
- the constant-time comparisons;
- the driver's read-modify-write (80,000 random operations in a model);
- the SSE state handling of the x86_64 kernel.

What it found, and what changed (patch 0005; the driver deadlock in 0004):

| Finding | Fix |
|---|---|
| Keyslot areas and the data segment weren't checked against each other: `addkey` could write a keyslot over data, and `removekey` could wipe a shared area and lose the last keyslot | the keyslots area is at most 128 MiB; every area must be inside it and on the device, no two may overlap, and the data segment must be behind it; anything else is refused as damaged |
| Keyslot material was sized by header values (a 3 MiB image asked for 128 GiB) | 4000 AF stripes only (cryptsetup's only value), and the area is checked against the device first |
| Driver: the global lock was held while opening and closing the underlying device, against the device manager's lock: a deadlock between a failing open and a close | the slot is reserved, and the device opened and the key checked without the lock; the device name is copied before the node exists |
| The boot loader kept every locked volume's JSON tree (a 4 MiB crafted header: 200 MB) | the JSON is freed after reading (`luks2_release_metadata()`); at most 16,384 JSON nodes |
| Keys were left in RAM when the loader rebooted or returned to the firmware | wiped on "Reboot" and on the way out |
| Argon2 parameters were unbounded (16 million lanes, 4 TiB) | memory at most 4 GiB, 64 lanes, 2^20 passes; such keyslots aren't opened |
| A secondary header whose size isn't its offset was accepted | ignored, as cryptsetup does |
| Duplicate JSON member names: the first won here, the last in cryptsetup | refused |
| The digest length wasn't checked (a 1-byte digest lets 1 key in 256 through) | it must be the hash's size; at most 10 million digest iterations |
| The kernel matched the boot volume by UUID only | also by the partition's offset and size, which the loader passes |
| No fallback to the other header copy when the newer one's metadata is bad | the other copy is tried |
| A 4 KiB header buffer on kernel stacks; small AES temporaries and an AES-NI dead store not wiped | a heap buffer; wiped |

Not changed:
- The control device has no permission check of its own (Haiku has one
  user).
- AES-NI leaves round keys in xmm registers, which the kernel saves and
  restores per thread and doesn't hand to userland.

The review's reproducers are kept as tests (`craft_header.py`, and 8 cases
in `cryptsetup_test.sh`).

## Correctness review

Another AI reviewer checked the code after patch 0006. Its eight findings
were all real:
- five reproduced on the host;
- two confirmed by reading the code;
- one partly reproduced on the host, partly from the code.

Checking the report showed that finding 3 was worse than written: one
keyslot change from Haiku then wrote the old metadata into both copies.
It also turned up two more gaps. The fixes are in patch 0007:

| Finding | Fix |
|---|---|
| 1. Two `luks` commands could change one volume at once (both removals passed the last-key check, and neither passphrase opened it afterwards) | the tools take an exclusive `flock()` on the device before reading the header and hold it to the end; a second one is refused ("another program is changing …"). The library also reads both copies again before writing keyslot material and before each header write, and refuses with `LUKS2_ERROR_CHANGED` when they differ from what it read (cryptsetup takes no `flock()`) |
| 2. `removekey` counted unbound keyslots as a way into the data | it counts only keyslots bound to the segment's digest; `dump` marks unbound ones |
| 3. A newer header copy with an unsupported feature was replaced by the older copy, and the next change wrote that over both | only damaged metadata makes the reader try the other copy; "unsupported" is the answer. A copy whose metadata failed counts as damaged, and `dump` names the copy it read |
| 4. No flush between the two header copies; Haiku's `fsync()` on a device does nothing | `luks2_device_io` has a `flush` hook (`luks2_fd_flush()`: `fsync()` for files, `B_FLUSH_DRIVE_CACHE` for Haiku devices). The keyslot material is flushed before the header that points to it, and each header copy before the other is touched (a damaged or older copy first). The area of a removed keyslot is wiped after the header no longer points to it. Errors come back to the caller |
| 5. EncryptedVolume didn't check, right before formatting, whether the partition was mounted, had partitions or was an unlocked volume | `luks2_partition_in_use()` (mounted: by `fs_stat_dev()`; partitions: devfs entries next to it; open: the driver's list), when the window opens and again in the worker, under the lock; an alert instead of the window when it's in use |
| 6. `luks write` cut a file that didn't fit and reported success | the whole file must fit and be whole sectors, or nothing is written; the final flush is checked |
| 7. The GUI cut passphrases at 512 bytes | longer ones disable Encrypt/Unlock, with a note in the window, and a script request is refused |
| 8. The boot loader needed the primary header's magic, and the kernel read the UUID from the primary copy | `luks2_probe()` finds the secondary copy too; the kernel asks the driver's boot module (`volume_uuid()`), which reads the header like the driver |
| (found in the check) `luks format` didn't check whether the device was in use either | the same check as EncryptedVolume on Haiku; on Linux, an exclusive open of a block device |
| (found in the check) Tracker and DriveSetup didn't see a volume whose primary magic is damaged | `luks2_partition_label()` falls back to the secondary copy's binary header |

Not changed:
- Between the in-use check and the first write there's still a window of
  microseconds in which a mount could start. The first thing
  `luks2_format()` writes is zeros over the metadata area, after which no
  file system is found there.
- A drive's write cache is flushed, but flash may keep old keyslot data in
  blocks it has remapped; wiping can't reach those.

EncryptedVolume's error alert no longer blocks the application (it quits
when the alert is closed), so a script can read and close it.

## Real hardware

The first try on real hardware was the anyboot image on a USB stick, in a
laptop whose NVMe disk held an installed Haiku. That disk was one whole BFS
volume (r1~beta6, BIOS-booted) with about 155 old package states. The
stick's EFI loader ran, but it booted the NVMe's system, and the boot
menu's volume list didn't have the stick at all. There were two loader
problems, neither in the LUKS code; patch 0008 fixes both:

| Problem | Cause | Fix |
|---|---|---|
| The stick's loader boots the internal disk | `platform_add_boot_device()` adds the firmware's disks in the firmware's order, and `get_boot_file_system()` boots the first Haiku volume it finds; the laptop's firmware lists its NVMe first | the EFI loader asks the firmware which device it was loaded from (`EFI_LOADED_IMAGE_PROTOCOL`'s `DeviceHandle`), and moves the disk whose device path starts that device's path to the front, behind a network disk. That's the disk itself, a partition on it, or the El Torito image OVMF boots a hybrid stick from. A disk without a system on it falls through to the next one, as before. Each disk is logged: `efi: disk 1: 808 MiB, 512-byte blocks, the boot loader's` |
| The boot menu doesn't list the stick | the menu mounts the system package of each volume it lists, and the loader's heap is 1.5 MiB. About 770 KB of it was in use when the menu started (mostly the boot volume's package), the 152 old states took about 50 KB more, and the stick's package didn't fit in the rest (`malloc(): Out of memory`, `packagefs.cpp:859`) | the EFI loader's heap gets another 1.5 MiB region whenever it runs out, up to 8 (`BOOT_SUPPORT_HEAP_GROWTH`, `malloc(): heap region 2 added`). Other platforms keep their single region; a bigger fixed heap would have stayed allocated after boot on every machine |

OVMF boots the anyboot image through its El Torito EFI image, not through
the ESP partition (other EDK2-based firmware probably does the same). Both
hold a copy of the loader; `qemu/anyboot_layout.py` finds them.

Not changed: an encrypted system on the loader's disk still loses to an
unencrypted one on another disk. The loader looks for encrypted volumes
only once the whole disk scan found nothing to boot
(`luks_probe_partition()` runs when file systems are mounted), so it asks
for the passphrase only then. The boot menu's "Unlock encrypted volume"
item works either way.

### The first encrypted install

The stick then booted on the laptop, and an encrypted install went
through: in DriveSetup, a GPT disk with a FAT32 ESP and a partition of type
"Be File System"; then Encrypt…, Installer, and Install EFI loader.
Booting it showed "no boot path found, scan for all partitions..." and
nothing more.

Reproduced in QEMU, with screenshots of the display:
- with the encrypted partition typed BFS, the EFI loader crashes with a
  page fault right after that line;
- typed as Linux LUKS (as the tests had it) or Linux data, it reaches the
  prompt;
- the partition's size (119 GiB) doesn't matter.

The cause is older than the LUKS work. `get_boot_file_system()` tries to
mount each BFS-typed partition of a boot device, and deleted one that
failed. But `platform_get_boot_partitions()` had put that partition into
`bootPartitions` with the node's only list link while `gPartitions` still
pointed to it, so `mount_file_systems()` walked into freed memory. Patch
0010 keeps the partition, and `mount_file_systems()` then finds the LUKS2
volume in it. `run-boot-test.sh` boot 7 retypes the partition as BFS and
must reach the prompt.

On a disk installed with an older loader, the workaround is to change the
encrypted partition's type in DriveSetup (Change parameters…, "Linux
LUKS" or "Linux data"). The kernel finds the volume by its UUID, not by
its type.

After that retype, DriveSetup listed the encrypted partition as "Unknown
(Linux data)". Its list names only file systems the disk device manager
recognises, and an older quirk put the same text into the "Partition type"
column. Patch 0011 adds the labels and the Linux LUKS type described under
"Haiku integration".

The encrypted install then ran on the laptop, and a KVM virtual machine
(OVMF) installed from the image has been in use since. A backup of
that VM's disk was checked on Linux:
- libcryptsetup reads the header as valid LUKS2 and opens it;
- the stored data segment tests as random (byte distribution, no repeated
  16-byte blocks);
- what's visible without the passphrase is what LUKS always shows: the
  header's label and settings, and which sectors were ever written.

## Two fixes outside the LUKS code

Both turned up on that VM's EFI system partition, and both are in stock
Haiku.

- **FAT's hidden sectors (patch 0012).** The BPB's hidden sectors field
  counts the sectors before the volume on its disk. `mkdos.cpp` wrote the
  partition's size there (in 512-byte units), in both the FAT32 and the
  FAT12/16 boot sector. It now writes the partition's offset in the
  volume's sectors, or 0 when that's unknown or doesn't fit the 32-bit
  field. Haiku's driver doesn't read the field; boot code and other
  systems' tools may.
- **Unmounting at shutdown (patch 0013).** Shutdown only synced, so a
  volume mounted at the time was never unmounted. FAT records a clean
  state only at unmount, so an ESP or a USB stick mounted at shutdown
  stayed marked "not cleanly unmounted" (FAT[1] 0x7fff), which Windows
  takes for a crash and mtools refuses. `system_shutdown()` now unmounts:
  - the writable volumes on a `/dev` device other than the boot volume,
    newest first;
  - nothing on network or virtual file systems;
  - nothing by force: `fs_unmount()` can then wait forever, so a volume
    still in use stays mounted and is synced as before.

  When rebooting, the teams were just killed and release their files as
  they die, so a busy volume is tried again for up to 2 s. Each volume is
  logged as unmounted or staying mounted.

## Changes for current Haiku master

The work was done on a Haiku `master` of late October 2026. Two upstream
changes since then needed something of their own:

- **The EFI loader's place (patch 0014).** Upstream moved the EFI loader
  from `/boot/system/data/platform_loaders/` to `/boot/system/` (commit
  d1231c1ac9), and Installer takes it from there. The UI test compares the
  loader Installer put on the ESP with that file.
- **DriveSetup's window lock (patch 0015).** Since commit 08afac4cda,
  DriveSetup scans the disks in `MainWindow::Run()`, after the window's
  thread has started, without the window's lock: adding the rows of the
  partition list dropped into the debugger ("Looper must be locked.") in
  the UI test, which has four disks. The scan now locks the window. That
  bug is in stock Haiku.

## Tested

All in QEMU (x86_64, UEFI/OVMF, NVMe) and on the build host, unless it says
otherwise. The scripts are in `src/tests/system/luks2` (its `README.md`
describes each).

The table's results are from the work's original base (Haiku `master` of
late October 2026). On `master` a8694efb7c (2026-11-12), on 2026-11-13:
- **Same results:** the host tests (both builds), the driver test and the
  loader test.
- **Patches 0001-0015:** each of the boot test's checks passed in at least
  one of three runs. The UI test's live system passed its 25 checks, and
  the host's checks passed. The installed disk's boot passed once. The
  boots that failed hit one of two problems of stock Haiku, described at
  the end of `BUILDING.md`, section 4:
  - a deadlock between net_server and launch_daemon;
  - the kernel switching its serial output off when the UART stays busy
    too long. The guest carries on, but the test can't see its result.
    Patch 0016 puts the tests' serial port on a file instead of a socket,
    which avoids it.
- **All 16 patches:** the boot test passed 19 of 19, and the loader test
  8 of 8. The UI test's live system passed its 25 checks, and so did the
  host's checks of Installer's EFI loader and the partition type. The
  installed disk's first boot hung after package_daemon's "is consistent"
  lines, where the deadlock hangs; that hang wasn't captured to confirm
  it. The same disk then booted 4 more times and passed its checks each
  time.

| What | How | Result |
|---|---|---|
| Crypto primitives | `crypto_test vectors`: FIPS-197, IEEE 1619, FIPS 180-4, RFC 7914, RFC 7693, RFC 9106 | all pass, AES portable and AES-NI |
| Same, random inputs | `crypto_crosscheck.py` against OpenSSL (XTS, PBKDF2, Argon2i/id, odd parameters) | about 220 checks per run, 0 failures |
| LUKS2 against cryptsetup 2.8.4 | `cryptsetup_test.sh`: both directions, 8 hostile headers, the correctness review's cases; optimised and with ASan/UBSan | 69 of 69, both builds |
| Power loss while changing keyslots | `crash_test`: power lost after every write and flush, every fate of the unflushed writes | 368 states, all open with the old passphrase; without the flushes 1,404 of 5,546 don't, so the model would have caught the old code |
| Driver and tool in Haiku | `run-driver-test.sh` | 43 checks in Haiku, 6 on the host |
| Encrypted system volume | `run-boot-test.sh`: 7 boots of a GPT disk with an ESP and a LUKS2 partition, the RAM searched after two | 19 of 19 |
| EncryptedVolume, Tracker, Installer | `run-ui-test.sh`, the live system driven over ssh with `hey`, then the installed disk booted alone | 4 on the host, 25 in the live system, 5 in the installed one |
| The loader on a machine with Haiku on its internal disk | `run-loader-disk-test.sh` | 8 of 8; the loader before patch 0008 fails all 8 |
| arm64 | `arm64_compile_check.sh`: every LUKS2 source and the changed loader files, with an arm64 cross compiler, `-Werror` | 36 of 36 compile |

- **Data, byte for byte, both ways:**
  - What cryptsetup encrypted (512- and 4096-byte sectors, AES-128- and
    AES-256-XTS) decrypts identically with the host tool, and in Haiku
    through the driver.
  - What Haiku and the host tool wrote decrypts identically with
    cryptsetup's volume key and OpenSSL, a partial sector
    (read-modify-write in the driver) included.
  - A BFS volume made on Haiku inside a LUKS2 volume reads back on Linux
    (host tool + bfs_shell).
- **Keyslots:**
  - argon2id, argon2i, pbkdf2-sha256 and -sha512, several slots, priority
    "ignore";
  - slots killed by cryptsetup, and slots added and removed on Haiku and
    accepted by cryptsetup, and the reverse;
  - a damaged primary header (the secondary is used), both damaged, a
    damaged keyslot area; LUKS1 and an interrupted reencryption are
    refused.
- **Boot:**
  - The loader asks for the passphrase (typed through OVMF's serial
    console), refuses a wrong one and asks again, unlocks through the boot
    menu too, and boots.
  - The kernel opens the same volume with the loader's key
    (`Mounted boot partition: /dev/disk/virtual/luks/0/raw`).
  - Files written at one boot are there at the next, and read back on the
    build host. Closing the boot volume is refused.
  - It works with and without AES-NI (`-cpu max,-aes`: the portable AES in
    the loader and the kernel).
  - A keyslot cryptsetup added with argon2id, 512 MiB and 4 lanes opens in
    the loader (four 128 MiB EFI page allocations).
  - With the primary header's UUID damaged, and then its magic too, the
    loader and the kernel use the secondary copy.
- **Wiping:** the guest's 2 GiB of RAM, saved after boot, hold no copy of
  the 64-byte volume key and none of the passphrase.
  - With AES-NI each 32-byte half appears once: the first two round keys of
    the driver's key schedule are the key itself (dm-crypt is the same).
  - With the portable AES, nothing.
  - The first run found one whole copy: the loader had added the key to the
    boot volume message before other fields, the message grew, and the old
    buffer was freed with the key in it. Adding the key last fixed it.
- **Speed** on the build host (a laptop CPU, one thread):
  - AES-256-XTS 1256 MB/s with AES-NI, 52 MB/s portable;
  - PBKDF2-SHA256 2.46 million iterations/s;
  - Argon2 1.2 GiB per pass and second.

  In QEMU's software emulation (AES-NI emulated) the driver read 73 MB/s.
  A boot in QEMU takes 55 s, or 90 s with the 512 MiB keyslot.

**Not tested**

- More hardware than one laptop and one VM: other firmware, keyboards and
  their layouts at the prompt.
- A cryptsetup-default keyslot (up to 1 GiB, benchmarked for 2 s with 4
  threads) in the loader. By the host numbers that's about 3-4 s on a
  laptop CPU, with 4 lanes one after another.
- The BIOS loader, which has no LUKS support.
- Long runs, a real power cut (the crash test is a model), a full disk,
  large volumes, many open volumes at once, low memory.
- The driver's speed under load. It has no `io` hook, so devfs falls back
  to synchronous `read()`/`write()` through 64 KiB bounce buffers (4 per
  device).
- DriveSetup's and Tracker's menu items themselves: `hey` can't open those
  menus. The messages they send are tested, and DriveSetup's labels were
  checked on screenshots.

## Known risks and limits

- **No person has reviewed the crypto and LUKS2 code.** It passes the
  vectors and the cryptsetup tests, and the two AI reviews found and got
  fixed the problems listed above. Keep backups of anything on an
  encrypted volume.
- **The ESP isn't protected.** The EFI system partition and the loader are
  neither encrypted nor verified (there's no Secure Boot here), so someone
  with access to the machine can replace the loader and catch the
  passphrase ("evil maid").
- **The key stays in memory.** The volume key is in kernel memory while
  the volume is open (the AES key schedules), readable from KDL or by a
  cold-boot attack, as with dm-crypt.
- **No access control:** the control device has no permission check beyond
  devfs's, so any local user can open and close volumes with a key; Haiku
  has one user today.
- **Not supported:** discards are never passed down. There's no header
  backup/restore, reencryption, key files at boot, tokens, LUKS1, detached
  headers or integrity.
- **Keyboard layout:** passphrases are UTF-8 as typed in the firmware's
  keyboard layout. Characters the firmware can't type can't be used at
  boot.
- **Not for Haiku:** the Haiku project takes no AI-written code, so these
  changes are not for submission upstream.

## What arm64 still needs

- **An accelerator** in the `arch/arm64` slot (AESE/AESD/AESMC/AESIMC,
  detected through `ID_AA64ISAR0_EL1`). The portable AES does 52 MB/s on a
  laptop CPU, perhaps a fifth of that on a Cortex-A53, slower than a
  typical eMMC.
  - Before the kernel driver uses NEON, check that the arm64 kernel saves
    the FP/SIMD state on every kernel entry. The arm64 kernel isn't built
    with `-mgeneral-regs-only`; x86_64's saves it.
- **The prompt and Argon2 under U-Boot's EFI:** its console input (serial,
  USB keyboard), its `AllocatePages()` with 2 GiB of RAM (use keyslots of
  at most 256 MiB), and its watchdog (which the loader now switches off).
- **Images for boards** that boot from an SD card or eMMC. The host `luks`
  can make an encrypted one: `luks format`, then `luks write` of the BFS
  image. Growing it at first boot would grow the partition, then the BFS
  volume inside (a "dynamic" segment follows the partition).
- **Tests on a board.**

## Later

- Remembering passphrases of data volumes in the key store, and unlocking
  them at login.
- Argon2 keyslots with multi-lane threads in userland.
- `luks reencrypt` (encrypting an existing volume in place), header
  backup/restore, key files, and TRIM pass-through as an option.
