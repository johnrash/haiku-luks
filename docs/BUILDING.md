# Building Haiku with the LUKS2 patches

The patches in [`../patches`](../patches) are `git format-patch` files for
Haiku's `master` branch. They were made on top of commit
`a8694efb7c64c84072a83d405531bed559680bb8` ("PCI: Cleanup ReadConfig and
WriteConfig methods.", 2026-11-12). That tree, with all 16 patches, builds
an x86_64 image with no compiler warnings in the files they change. Its
results in the tests below are in [`DESIGN.md`](DESIGN.md) ("Tested"). The
QEMU tests also run into a deadlock of stock Haiku now and then (end of
section 4).

You need a Linux (or other) machine that can build Haiku: see Haiku's
["Building Haiku"](https://www.haiku-os.org/guides/building/) guides for
the packages it needs. Only x86_64 has been built and tested. arm64 passes
a compile check of the changed files only, and the encrypted boot needs an
EFI boot loader.

## 1. Get the sources

Follow Haiku's guide to get `haiku` and `buildtools`
(<https://www.haiku-os.org/guides/building/get-source-git>), and put them
side by side, with this repository next to them:

```
work/
  buildtools/
  haiku/
  haiku-luks/        (this repository)
```

Haiku's build names the image after the `hrev` tag of the commit it's
built from. A clone without those tags (GitHub's mirror has none) stops
with "you are using a Haiku clone without tags". Clone from Haiku's own
server as the guide says, or give the revision yourself (step 3).

## 2. Apply the patches

```bash
cd haiku
git checkout -b luks a8694efb7c64c84072a83d405531bed559680bb8
git am ../haiku-luks/patches/*.patch
```

On a newer `master`, try `git checkout -b luks origin/master` and
`git am -3 ../haiku-luks/patches/*.patch`. The patches touch few shared
files (the build's package and image lists, a few Jamfiles, the boot
loader, `vfs_boot.cpp`, DriveSetup, Tracker's Mount menu, the FAT
initializer, the kernel's shutdown). When `git am` stops, fix the files it
names, `git add` them, and run `git am --continue`.

## 3. Build an image

From the `haiku` directory:

```bash
mkdir generated.x86_64
cd generated.x86_64
../configure --cross-tools-source ../../buildtools --build-cross-tools x86_64 -j8
jam -q -j8 @release-anyboot
```

- **The image** is `haiku-release-anyboot.iso`. Despite its name, it's a
  disk image for a USB stick as well as for an optical disc.
- **Other profiles:** the tests used `@release-anyboot`. EncryptedVolume
  and `luks` are in the `minimum` image definition, which the other
  profiles include too, and the driver is in the `haiku` package.
- **A clone without tags:** before running jam, put a line like
  `HAIKU_REVISION = hrev60450 ;` into `generated.x86_64/UserBuildConfig`
  (the hrev of the commit you built from, if you know it). The revision
  only names the build.
- **Memory:** cross tools and an image need a few GiB of disk space. Lower
  `-j` if the machine runs short of RAM.

To install it, write the image to a USB stick (`dd` on Linux) and boot it
in UEFI mode, then see [`USING.md`](USING.md).

### An existing system

On a Haiku x86_64 system of about the same revision, the image's packages
can replace the system's own. That's only for testing: back up first. The
packages are in `generated.x86_64/objects/haiku/x86_64/packaging/packages/`:
`haiku.hpkg`, `haiku_loader.hpkg` and `haiku_devel.hpkg`, plus
`haiku_datatranslators.hpkg` if your system has it. To install them:
1. Copy them to the system.
2. Run `pkgman install -y haiku*.hpkg` there.
3. Reboot.

For an encrypted system disk, the new EFI loader must also be on the ESP:
use Installer's Tools → Install EFI loader.

## 4. Run the tests

The tests are in the patched tree, in `src/tests/system/luks2`; its
`README.md` describes each. They need:
- **host tests:** gcc, Python 3 with `cryptography`, `openssl`, and
  Linux's `cryptsetup` 2.x. cryptsetup works on image files without root.
  If it isn't installed, Debian's and Ubuntu's `cryptsetup-bin` package
  can be unpacked anywhere with `apt-get download cryptsetup-bin` and
  `dpkg-deb -x`; it uses the system's `libcryptsetup`.
- **QEMU tests:** `qemu-system-x86_64`, OVMF (`/usr/share/OVMF`), `sgdisk`,
  `mtools`, `ssh` and `scp`, plus three things from the build: the
  anyboot image, `haiku_loader.efi` and `bfs_shell`.

```bash
cd haiku
T=src/tests/system/luks2
G=generated.x86_64
CS=/usr/sbin/cryptsetup    # or the unpacked one

# Host: crypto vectors, OpenSSL comparisons, cryptsetup interoperability,
# the crash model; optimised and with ASan/UBSan
sh $T/run_tests.sh $CS /tmp/luks-host

# What the QEMU tests need from the build
(cd $G && jam -q '<build>bfs_shell')
BFS=$PWD/$G/objects/linux/x86_64/release/tools/bfs_shell/bfs_shell
LOADER=$PWD/$G/objects/haiku/x86_64/release/system/boot/efi/haiku_loader.efi
IMAGE=$PWD/$G/haiku-release-anyboot.iso
LUKS=/tmp/luks-host/o2/luks        # the host luks that run_tests.sh built

# QEMU: the driver and the tool, an encrypted system volume, the GUI and
# Installer, the loader on a machine with Haiku on its internal disk
sh $T/qemu/run-driver-test.sh $IMAGE /tmp/luks-driver $CS $LUKS $BFS
sh $T/qemu/run-boot-test.sh $IMAGE $LOADER /tmp/luks-boot $LUKS $BFS $CS
sh $T/qemu/run-ui-test.sh $IMAGE /tmp/luks-ui $BFS
sh $T/qemu/run-loader-disk-test.sh $IMAGE $LOADER /tmp/luks-loader $BFS
```

Each test prints `ok` and `FAIL` lines and a summary, and keeps its logs in
its work directory. QEMU runs in software emulation (`-cpu max` emulates
AES-NI), so a boot takes about a minute. The work directories need a few
GiB, and the QEMU tests' socket paths must be short: use short work
directory names.

**A hang in stock Haiku.** Now and then a QEMU boot stops shortly after
package_daemon's "is consistent" lines, with every CPU idle, and the test
waits until its timeout (an hour per boot by default; the boot test's last
argument sets it, in seconds). That's a deadlock in Haiku itself, not in
these patches:
1. With no network card, net_server closes its last socket, and the network
   stack module is unloaded.
2. The uninit of `network/notifications` calls `unregister_generic_syscall()`,
   which waits for calls in progress, while `put_module()` holds the module
   lock.
3. launch_daemon's `start_watching_network()` call is in progress, and
   waits in `get_module()` for that lock.

It hit 8 of 125 boots here. If a boot hangs, run the test again.

**Serial output that stops.** Since early 2025 (commit b190b70845),
Haiku's x86 kernel turns its serial debug output off for the rest of the
boot the first time the UART stays busy for 256K polls. Under QEMU that
happens when whatever reads a socket-backed serial port falls behind by
about 50 ms. The tests used to read the serial port through a socket, and
in 4 of those 125 boots the serial log ended about 13 s into the boot
(during the kernel's device scan): the guest finished its test and powered
off as usual, but the test saw no result. Since patch 0016, QEMU writes
the serial output to a file and reads the keystrokes from a FIFO
(`-chardev file,...,input-path=<fifo>`), which never makes the UART wait.
Keep it that way if you change the scripts or run QEMU yourself.

`sh $T/arm64_compile_check.sh generated.arm64 /tmp/luks-arm64` compiles
every LUKS2 source and the changed loader files for arm64 with `-Werror`.
It needs an arm64 build directory (`configure ... --build-cross-tools
arm64`) that has built the kernel once: it reads that directory's cross
compiler, its unpacked `gcc_syslibs_devel` package and the kernel's
generated headers, and builds nothing there.
