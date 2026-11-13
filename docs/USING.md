# Using encrypted volumes

How to use what the patches add, on a Haiku image built with them (see
[`BUILDING.md`](BUILDING.md)).

> **No person has reviewed this encryption code.** Don't keep the only copy
> of anything important on an encrypted volume.

## What's in the image

- **EncryptedVolume** (Applications): unlocks, encrypts and locks volumes.
  Tracker and DriveSetup start it for you.
- **Tracker's Mount menu** (right-click the desktop → Mount): each locked
  encrypted partition is listed as `Unlock "label" (size, encrypted)…`.
- **DriveSetup's Partition menu:** Unlock…, Lock and Encrypt… . Its list
  shows encrypted partitions as "LUKS2 encrypted", with their names.
- **The boot loader** (EFI only): asks for the passphrase of an encrypted
  system volume.
- **`luks`,** a command line tool (`luks --help`).

The volumes are standard LUKS2, as Linux's cryptsetup makes them
(AES-XTS, Argon2id). Linux opens Haiku's, and Haiku opens Linux's when
they use the supported subset ([`DESIGN.md`](DESIGN.md)).

## Encrypting a data partition

This erases the partition.

1. In DriveSetup, select a partition that isn't mounted. If it's mounted,
   use Partition → Unmount first.
2. Choose Partition → Encrypt….
3. Give the volume a name, type the passphrase twice, click Encrypt, and
   confirm.

EncryptedVolume formats it as LUKS2 (measuring Argon2id for about 2 s on
this machine), puts a BFS volume inside, and mounts it. On a GPT or MBR
disk it also changes the partition's type to "Linux LUKS".

From Terminal, `EncryptedVolume --encrypt /dev/disk/…` does the same
through the same window.

## Unlocking and locking

- **Unlock:** Tracker's Mount menu, or DriveSetup's Partition → Unlock….
  EncryptedVolume asks for the passphrase (with a Read-only option), then
  mounts the volume inside and opens it in Tracker. A wrong passphrase is
  reported in the window, and you can try again.
- **Lock:** unmount the volume in Tracker. mount_server ejects the
  decrypted device, which locks it. DriveSetup's Partition → Lock and
  `EncryptedVolume --lock /dev/disk/…` do the same.
- **The terminal's `unmount`** leaves the volume unlocked. `luks close`
  locks it.

## An encrypted system volume

The encrypted system starts only through the EFI boot loader: the
machine must boot in UEFI mode, with Secure Boot off. The BIOS loader
can't unlock it.

**Before you start:**
- This erases the disk you install onto. Back it up first.
- The boot loader reads keys through the firmware, which normally uses a
  US layout. Pick a passphrase you can type correctly on that layout.

**1. Start the image.** Write the anyboot image to a USB stick and boot it
in UEFI mode. At the welcome window, click "Try Haiku".

**2. Partition the disk** with DriveSetup:
1. Select the disk. If one of its volumes is mounted, select it and use
   Partition → Unmount.
2. Choose Disk → Initialize → GUID Partition Map, and confirm.
3. Select the empty space and choose Partition → Create…. Make it at least
   128 MiB, of type "EFI system data". Then use Partition → Format → FAT32
   File System, and name it "ESP".
4. Select the rest of the empty space and choose Partition → Create…,
   using all of it. The type can be "Linux LUKS" (or "Be File System";
   Encrypt… changes it). Don't format it.

**3. Encrypt the big partition.** Select it and choose Partition →
Encrypt…. Give it a name (for example "Haiku") and the passphrase twice,
then click Encrypt. After a few seconds it's unlocked and mounted, with BFS
inside.

**4. Install** with Installer:
1. For "Onto", pick the encrypted volume. It's listed with
   `/dev/disk/virtual/luks/0/raw`, not with the disk's own partition.
2. For "Install from", pick the stick's Haiku, then click Begin.
3. When it finishes, choose Tools → Install EFI loader → the ESP, and
   confirm.

**5. First boot.**
1. Shut down, and **pull the stick out first**. The loader asks for a
   passphrase only when it finds nothing else to boot. While the stick is
   in, it boots the stick's unencrypted system instead.
2. Power on. If the machine doesn't boot the disk by itself, choose it in
   the firmware's boot menu.
3. The loader shows the volume's name and
   `Passphrase (Esc for the boot menu):`. Type the passphrase (a `*` per
   character) and press Enter. Haiku boots without a second prompt.

**At the prompt:**
- A wrong passphrase says so and asks again.
- Esc opens the boot menu. Its "Select boot volume" also has an "Unlock
  encrypted volume …" item. Use it to boot the encrypted system while
  another disk with Haiku is attached.
- If the keyslot needs more memory than the firmware can give, the loader
  says so. A keyslot made on a much larger machine may need
  `luks addkey --memory 262144`.

**If the boot goes wrong:**
- Press Space at power-on for the boot menu, then Select debug options →
  Display current boot loader log.
- The loader logs each disk (`efi: disk 1: … the boot loader's`) and each
  LUKS2 volume it finds (`luks: locked LUKS2 volume …`).

The system volume's swap file (`/var/swap`) is inside the encrypted volume,
so it's encrypted too.

### A disk installed with an older loader

A loader without patch 0010 crashes after "no boot path found" when the
encrypted partition's GPT type is "Be File System". Boot the stick, and in
DriveSetup select the encrypted partition, choose Change parameters…, and
set the type to "Linux LUKS". Or install the new loader onto the ESP with
Installer's Tools → Install EFI loader.

## The `luks` command

```
luks format <device>           create a LUKS2 volume (destroys data)
luks dump <device>             show the header and keyslots
luks test <device>             check a passphrase
luks addkey <device>           add a passphrase
luks removekey <device> [slot] remove the passphrase's keyslot, or the given one
luks open <device>             unlock; prints the decrypted device
luks close <id|device>         lock an open volume again
luks list                      show the open volumes
luks read <device> <file>      decrypt the data area into a file
luks write <device> <file>     encrypt a file into the data area
```

- **Options:**
  - `-k file` reads the passphrase from a file (`-` for stdin).
  - `--new-key-file` gives addkey's new passphrase.
  - `-L` sets the label, `--sector-size` the sector size (512-4096),
    `--key-size` the key size (512 or 256 bits).
  - `--pbkdf argon2id|argon2i|pbkdf2`, `--iter-time ms`, `--memory KiB`,
    `--parallel n`, `--time n` and `--iterations n` set the KDF.
  - `-r` opens read-only. `--force` allows removing the last keyslot.
    `-y` doesn't ask before formatting.
- **Locking:** a command that changes a volume takes an exclusive lock on
  the device. A second command on the same volume is refused instead of
  racing it.
- **A second passphrase** (`luks addkey`) is the only spare key there is:
  header backups aren't supported yet.
- **A raw device:** after `luks open`, put a file system on the device it
  prints (`mkfs -t bfs /dev/disk/virtual/luks/0/raw Name`) and mount it as
  usual.

`luks` also builds on Linux, without `open`, `close` and `list`
(`src/tests/system/luks2/build_host.sh`). With `luks format` followed by
`luks write` of a BFS image, it makes encrypted disk images without root.

## With Linux

- **Haiku's volumes on Linux:** `cryptsetup open /dev/sdX2 haiku` opens
  them. Linux can only read BFS (the `befs` driver), so the files inside
  are read-only there.
- **Linux's volumes on Haiku:** they open when they use `aes-xts-plain64`,
  one data segment, and argon2id, argon2i or pbkdf2 keyslots. Haiku then
  mounts what's inside if it has a driver for it; for ext4, that's
  read-only. Volumes with integrity, a detached header, an unfinished
  reencryption, or LUKS1 are refused.
- **Partition type:** "Linux LUKS" is the same type Linux tools use for
  LUKS partitions (GPT `CA7D7CCB-63ED-4C53-861C-1742536059CC`, `sgdisk`
  code 8309; MBR 0xe8).

## What isn't protected

- **The ESP and the loader** are neither encrypted nor verified. Someone
  with access to the machine could replace the loader and catch the
  passphrase.
- **Memory:** while a volume is unlocked, its key is in kernel memory.
- **What's visible without the passphrase:** the LUKS2 header (the label,
  the cipher and KDF settings), and which sectors were ever written.
- **Flash storage** may keep old keyslot data in blocks it has remapped
  after `luks removekey`.
