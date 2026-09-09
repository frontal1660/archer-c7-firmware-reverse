# This Archer Firmware Really Has Daddy Issues


  <br/>

## Preamble

This repository documents the reverse engineering of a TP-Link Archer C7 v1.1 router with one initial objective: obtain a root shell on the device.

What looked like a relatively straightforward target quickly became less trivial.

Two main obstacles shaped the investigation:

```text
1. Recover the complete flash contents without desoldering the SPI flash chip.
2. Bypass the mechanisms preventing the available root credentials from providing
   usable root access on the running router.
```

Rather than removing the flash chip or replacing the firmware outright, the investigation stays entirely within the interfaces and capabilities already exposed by the device.

The first challenge is solved by turning the UART/U-Boot console into a read-only flash acquisition interface. The complete 8 MiB NOR flash is dumped through UART, reconstructed on the host, independently acquired twice, and validated against checksums calculated directly by U-Boot.

The second challenge requires going considerably deeper into the boot chain.

After the extracted root filesystem fails to explain the live authentication behavior, the investigation moves into the original Linux kernel itself: extracting and decompressing it, locating its embedded command line and startup logic, applying minimal binary patches, recompressing it, understanding the TP-Link-specific kernel image format, rebuilding a bootable image, and finally loading the modified kernel through TFTP to boot it entirely from RAM.

The resulting path is therefore:

```text
UART
  |
  +--> U-Boot
  |      |
  |      +--> complete flash dump without desoldering
  |
  +--> firmware extraction
         |
         +--> rootfs analysis
         |
         +--> kernel deep dive
                |
                +--> binary patching
                +--> LZMA recompression
                +--> TP-Link image reconstruction
                +--> TFTP
                +--> RAM boot
                       |
                       v
                  early root shell
```

None of the modified kernels are written back to flash.

The original firmware remains untouched throughout the experiment, and every modified-kernel test is disposable: a power cycle always returns the router to the original kernel stored in NOR flash.

The working rules are:

```text
No desoldering
No flash erase
No flash write
Keep the validated 8 MiB flash dump immutable
Perform modified-kernel tests from RAM through U-Boot/TFTP
A power cycle must always return to the untouched kernel stored in flash
```

The approach throughout the reverse is deliberately empirical:

observe first, conclude second.

The interesting part of this project is therefore not only obtaining root access, but reaching it while preserving the original hardware and firmware, using the router's own UART, bootloader and kernel against the limitations encountered along the way.


  <br/>

## Action Plan

- [Identify the UART Pads on the Board](#link-identify-the-uart-pads-on-the-board)
- [Connect the USB-to-UART Adapter](#link-connect-the-usb-to-uart-adapter)
- [Capture the Complete Boot Sequence](#link-capture-the-complete-boot-sequence)
- [Interrupt Autoboot and Access U-Boot](#link-interrupt-autoboot-and-access-u-boot)
- [Identify the Real Flash Layout from the Linux Boot Log](#link-identify-the-real-flash-layout-from-the-linux-boot-log)
- [Acquire the Complete 8 MiB Flash Image](#link-acquire-the-complete-8-mib-flash-image)
- [Validate the Flash Dump](#link-validate-the-flash-dump)
- [Split the Flash Dump](#link-split-the-flash-dump)
- [Identify the Firmware Components](#link-identify-the-firmware-components)
- [Extract the SquashFS Root Filesystem](#link-extract-the-squashfs-root-filesystem)
- [Analyze the Runtime Boot Sequence and Authentication Files](#link-analyze-the-runtime-boot-sequence-and-authentication-files)
- [Identify the Root Password Hash and Test the Static Credentials](#link-identify-the-root-password-hash-and-test-the-static-credentials)
- [Determine Why the Static Root Password Does Not Authenticate](#link-determine-why-the-static-root-password-does-not-authenticate)
- [Override the Effective Kernel Startup With U-Boot bootargs](#link-override-the-effective-kernel-startup-with-u-boot-bootargs)
- [Define a New Strategy](#link-define-a-new-strategy)
- [Extract the LZMA Kernel Payload](#link-extract-the-lzma-kernel-payload)
- [Decompress the Kernel](#link-decompress-the-kernel)
- [Locate the Embedded Kernel Command Line](#link-locate-the-embedded-kernel-command-line)
- [First Patch: Change init=/sbin/init in the Embedded Command Line](#link-first-patch-change-init-sbin-init-in-the-embedded-command-line)
- [Recompress the First Patched Kernel](#link-recompress-the-first-patched-kernel)
- [Try a Standard U-Boot uImage](#link-try-a-standard-u-boot-uimage)
- [Analyze the First RAM-Boot Failure](#link-analyze-the-first-ram-boot-failure)
- [Compare the Standard uImage with the Original TP-Link Kernel Block](#link-compare-the-standard-uimage-with-the-original-tp-link-kernel-block)
- [Rebuild the First Patched TP-Link Kernel Block](#link-rebuild-the-first-patched-tp-link-kernel-block)
- [Boot the First Patched TP-Link Kernel from RAM](#link-boot-the-first-patched-tp-link-kernel-from-ram)
- [Locate the Kernel's Internal Init Fallback Strings](#link-locate-the-kernel-s-internal-init-fallback-strings)
- [Second Patch: Patch Both Init Paths](#link-second-patch-patch-both-init-paths)
- [Recompress and Rebuild the Second Patched Kernel](#link-recompress-and-rebuild-the-second-patched-kernel)
- [Configure U-Boot for the Second RAM Boot](#link-configure-u-boot-for-the-second-ram-boot)
- [Load the Second Patched TP-Link Kernel Block into RAM](#link-load-the-second-patched-tp-link-kernel-block-into-ram)
- [Boot the Second Patched Kernel from RAM](#link-boot-the-second-patched-kernel-from-ram)
- [Interpret the Early Shell](#link-interpret-the-early-shell)
- [Next Step: Recreate the Runtime Initialization](#link-next-step-recreate-the-runtime-initialization)


  <br/>

## <a name="link-identify-the-uart-pads-on-the-board">Identify the UART Pads on the Board</a>

The board exposes a small 1×4 header next to the larger debug header. The objective is to determine whether this 1×4 header is a UART and identify its signals without desoldering anything.

Only three signals are required for a serial console:

```text
GND
Router TX
Router RX
```


The final identification method is therefore:

```text
GND  -> confirmed by continuity
VCC  -> stable ~3.3 V reference
TX   -> serial activity during boot
RX   -> remaining data pad, confirmed once U-Boot accepts input
```

The next step is to connect only GND, TX and RX and verify that the boot console is readable.


  <br/>

## <a name="link-connect-the-usb-to-uart-adapter">Connect the USB-to-UART Adapter</a>

First, create the directories that will be used during this hack:

```console
$ mkdir 01_boot_log 02_extract_firmware 03_firmware_split 04_rootfs 05_kernel_hacking
```

The UART signals must be crossed:

```text
Archer GND  -> USB-UART GND
Archer TX   -> USB-UART RX
Archer RX   -> USB-UART TX
Archer VCC  -> NOT CONNECTED
```

The serial configuration is:

```text
115200 baud
8 data bits
No parity
1 stop bit
No flow control
```

Once the CP2102 is plugged, identify the serial device exposed by the adapter:

```console
$ python3 -m serial.tools.list_ports -v
/dev/ttyUSB0
    desc: CP2102 USB to UART Bridge Controller - CP2102 USB to UART Bridge Controller
    hwid: USB VID:PID=10C4:EA60 SER=0001 LOCATION=1-2.
```

Open the console with `miniterm`:

```console
$ python3 -m serial.tools.miniterm /dev/ttyUSB0 115200 --parity N
```

Power-cycle the Archer.

Readable boot messages appearing immediately on the terminal confirm all of the following:

```text
The selected header is a UART
The router TX line is correctly identified
GND is correct
The serial parameters are correct
```

At this point the connection is useful because the entire boot sequence can be observed.


  <br/>

## <a name="link-capture-the-complete-boot-sequence">Capture the Complete Boot Sequence</a>

Capture one complete normal boot from power on to the login prompt.

This first capture is important because it provides the router's actual runtime information before any U-Boot variable or boot path is modified.

Save the console output as:

```text
01_boot_log/boot.log
```

The complete log exposes several key pieces of information used later in the analysis:

```text
U-Boot version
Linux kernel version
Kernel command line
MTD partition creation
Root filesystem type
Serial console configuration
Normal userspace startup
Login prompt
```

Relevant versions observed on this Archer are:

```text
U-Boot 1.1.4 (May 11 2015 - 15:13:34)

Linux version 2.6.31--LSDK-9.2.0_U6.616
(root@liaozhiming) (gcc version 4.3.3 (GCC))
#1 Mon May 11 15:16:26 CST 2015
```

The normal boot eventually reaches:

```text
Archer C7 login:
```

Do not infer the flash layout from U-Boot variables yet.

The Linux boot log is more valuable because it shows the partition map that the running kernel actually uses.


  <br/>

## <a name="link-interrupt-autoboot-and-access-u-boot">Interrupt Autoboot and Access U-Boot</a>

Older TP-Link U-Boot builds are known to accept the hidden `tpl` sequence during autoboot to drop into the bootloader prompt.
The `tpl` must be sent at a precise moment : 

```text
Autobooting in 1 seconds
```

Reboot the Archer, wait for the moment and send:

```text
tpl
```

U-Boot stops the normal boot process and exposes its command prompt:

```console
ap135>
```

Run `help` command:

```console
ap135> help
?       - alias for 'help'
bootm   - boot application image from memory
cp      - memory copy
crc32   - checksum calculation
erase   - erase FLASH memory
flinfo  - print FLASH memory information
go      - start application at address 'addr'
help    - print online help
md      - memory display
mm      - memory modify (auto-incrementing)
mw      - memory write (fill)
nm      - memory modify (constant address)
printenv- print environment variables
progmac - Set ethernet MAC addresses
protect - enable or disable FLASH write protection
reset   - Perform RESET of the CPU
setenv  - set environment variables
tftpboot- boot image via network using TFTP protocol
version - print monitor version
```

Confirm the bootloader version.

```console
ap135> version

U-Boot 1.1.4 (May 11 2015 - 15:13:34)
```

The prompt provides direct read access to the memory-mapped flash and will later be used both for the flash acquisition and for the temporary RAM boot.

`printenv` can be inspected, but its partition information must not be treated as authoritative on this device.

```console
ap135> printenv
bootargs=console=ttyS0,115200 root=31:02 rootfstype=jffs2 init=/sbin/init mtdparts=ath-nor0:256k(u-boot),64k(u-boot-env),6336k(rootfs),1408k(uImage),64k(mib0),64k(ART)
bootcmd=bootm 0x9f020000
bootdelay=1
baudrate=115200
ethaddr=0xba:0xbe:0xfa:0xce:0x08:0x41
ipaddr=192.168.1.111
serverip=192.168.1.100
dir=
lu=tftp 0x80060000 ${dir}u-boot.bin&&erase 0x9f000000 +$filesize&&cp.b $fileaddr 0x9f000000 $filesize
lf=tftp 0x80060000 ${dir}ap135${bc}-jffs2&&erase 0x9f050000 +0x630000&&cp.b $fileaddr 0x9f050000 $filesize
lk=tftp 0x80060000 ${dir}vmlinux${bc}.lzma.uImage&&erase 0x9f680000 +$filesize&&cp.b $fileaddr 0x9f680000 $filesize
stdin=serial
stdout=serial
stderr=serial
ethact=eth0
```

This line does not match the Linux runtime observed on the router and is therefore stale or unused for the effective partition layout.

```text
bootargs=console=ttyS0,115200 root=31:02 rootfstype=jffs2 init=/sbin/init mtdparts=ath-nor0:256k(u-boot),64k(u-boot-env),6336k(rootfs),1408k(uImage),64k(mib0),64k(ART)
```

The next step is to return to the captured Linux boot log and extract the partition map that is actually used at runtime.


  <br/>

## <a name="link-identify-the-real-flash-layout-from-the-linux-boot-log">Identify the Real Flash Layout from the Linux Boot Log</a>

Search the normal boot log for the kernel command line and MTD initialization.

```console
$ grep -iE 'command line|mtd|partition|rootfs' 01_boot_log/boot.log
[    0.000000] Kernel command line: console=ttyS0,115200 root=31:02 rootfstype=squashfs init=/sbin/init mtdparts=ath-nor0:128k(u-boot),1024k(kernel),6848k(rootfs),128k(config),64k(ART) mem=128M
[    0.404000] 5 cmdlinepart partitions found on MTD device ath-nor0
[    0.412000] Creating 5 MTD partitions on "ath-nor0":
[    0.428000] 0x000000120000-0x0000007d0000 : "rootfs"
```

The important part is the `mtdparts=` parameter passed to the running kernel:

```text
mtdparts=ath-nor0:128k(u-boot),1024k(kernel),6848k(rootfs),128k(config),64k(ART)
```

Unlike the stale U-Boot environment, this is the partition layout actually consumed by Linux during the successful normal boot.

The flash layout below is therefore confirmed by the Linux boot log:

```text
0x000000  128 KiB   u-boot
0x020000 1024 KiB   kernel
0x120000 6848 KiB   rootfs
0x7d0000  128 KiB   config
0x7f0000   64 KiB   ART
```

The final boundary is:

```text
0x800000 = 8 MiB
```

The runtime kernel command line is:

```text
console=ttyS0,115200 root=31:02 rootfstype=squashfs init=/sbin/init mtdparts=ath-nor0:128k(u-boot),1024k(kernel),6848k(rootfs),128k(config),64k(ART) mem=128M
```

The kernel partition begins at flash offset:

```text
0x020000
```

We know from `printenv` that U-Boot boots that kernel from:

```text
0x9f020000
```

The kernel starts at flash offset `0x020000`, so the flash base address is:

```text
0x9f020000 - 0x020000 = 0x9f000000
```

The complete 8 MiB flash occupies:

```text
0x9f000000 -> 0x9f7fffff
```

This is the address range we will read through U-Boot to acquire the full flash image.


  <br/>

## <a name="link-acquire-the-complete-8-mib-flash-image">Acquire the Complete 8 MiB Flash Image</a>

Because this procedure excludes desoldering the flash chip, we need another way to extract its contents.

U-Boot can read the flash directly through its memory-mapped address space. The `md.b` command displays raw bytes without writing anything to flash.

A small read from the beginning of the flash confirms access:

```console
ap135> md.b 0x9f000000 0x100
9f000000: 10 00 00 ff 00 00 00 00 10 00 00 fd 00 00 00 00    ................
9f000010: 10 00 02 34 00 00 00 00 10 00 02 32 00 00 00 00    ...4.......2....
9f000020: 10 00 02 30 00 00 00 00 10 00 02 2e 00 00 00 00    ...0............
9f000030: 10 00 02 2c 00 00 00 00 10 00 02 2a 00 00 00 00    ...,.......*....
9f000040: 10 00 02 28 00 00 00 00 10 00 02 26 00 00 00 00    ...(.......&....
9f000050: 10 00 02 24 00 00 00 00 10 00 02 22 00 00 00 00    ...$......."....
9f000060: 10 00 02 20 00 00 00 00 10 00 02 1e 00 00 00 00    ... ............
9f000070: 10 00 02 1c 00 00 00 00 10 00 02 1a 00 00 00 00    ................
9f000080: 10 00 02 18 00 00 00 00 10 00 02 16 00 00 00 00    ................
9f000090: 10 00 02 14 00 00 00 00 10 00 02 12 00 00 00 00    ................
9f0000a0: 10 00 02 10 00 00 00 00 10 00 02 0e 00 00 00 00    ................
9f0000b0: 10 00 02 0c 00 00 00 00 10 00 02 0a 00 00 00 00    ................
9f0000c0: 10 00 02 08 00 00 00 00 10 00 02 06 00 00 00 00    ................
9f0000d0: 10 00 02 04 00 00 00 00 10 00 02 02 00 00 00 00    ................
9f0000e0: 10 00 02 00 00 00 00 00 10 00 01 fe 00 00 00 00    ................
9f0000f0: 10 00 01 fc 00 00 00 00 10 00 01 fa 00 00 00 00    ................
```

The complete acquisition can be performed by repeatedly issuing `md.b` over UART, reconstructing the hexadecimal byte output and verifying blocks with `crc32`.

Only read operations are used:

```text
md.b    read memory as bytes
crc32   calculate a checksum over memory
```

A well designed python script can do the job:

```console
$ python3 extract_flash_full_32k.py

>>>>>> ARCHER FLASH DUMP <<<<<<
Base       : 0x9f000000
Size       : 0x800000 (8388608 bytes)
Block      : 32768 bytes
NB Blocks  : 256
File       : archer-flash-dump1-8MiB.bin

[+] CRC32 flash before dump : 26ee9efa

[0001/256] 0x9f000000 CRC=d23db168 ACK
[0002/256] 0x9f008000 CRC=41de76ef ACK
...
[0256/256] 0x9f7f8000 CRC=22bee4ed ACK

[+] CRC32 : validated 
```

The first dump is performed:

```text
FLASH_BASE  = 0x9f000000
FLASH_SIZE  = 0x800000
BLOCK_SIZE  = 0x8000
BLOCK_COUNT = 256
OUTPUT      = archer-flash-dump1-8MiB.bin
```

A complete independent second acquisition is then performed :

```text
FLASH_BASE  = 0x9f000000
FLASH_SIZE  = 0x800000
BLOCK_SIZE  = 0x8000
BLOCK_COUNT = 256
OUTPUT      = archer-flash-dump2-8MiB.bin
```

This gives two independently reconstructed images of the same physical NOR flash.

An additional read-only checksum can be calculated directly by U-Boot across the entire flash:

```console
ap135> crc32 0x9f000000 0x800000
CRC32 for 9f000000 ... 9f7fffff ==> 26ee9efa
```

At no point is the flash erased, rewritten or desoldered.

The next step is to verify that both independently acquired 8 MiB images are byte-for-byte identical.


  <br/>

## <a name="link-validate-the-flash-dump">Validate the Flash Dump</a>

First verify that both files have the expected physical flash size.

```console
$ stat -c '%n %s' archer-flash-dump1-8MiB.bin archer-flash-dump2-8MiB.bin
archer-flash-dump1-8MiB.bin 8388608
archer-flash-dump2-8MiB.bin 8388608
```

Both files are exactly:

```text
8388608 bytes = 0x800000 bytes = 8 MiB
```

Calculate a SHA-256 digest for both dumps.

```console
$ sha256sum archer-flash-dump1-8MiB.bin archer-flash-dump2-8MiB.bin
365c2408075d2685a5385c83cf206160907994d1b6d16c69d141e793d3c57b32  archer-flash-dump1-8MiB.bin
365c2408075d2685a5385c83cf206160907994d1b6d16c69d141e793d3c57b32  archer-flash-dump2-8MiB.bin
```

Both files return the same SHA-256 digest.

Confirm the result with a direct byte-for-byte comparison.

```console
$ cmp archer-flash-dump1-8MiB.bin archer-flash-dump2-8MiB.bin
```

`cmp` produces no output when both files are identical.

The validated dump can now be used as the immutable source for all subsequent firmware splitting and offline analysis.


  <br/>

## <a name="link-split-the-flash-dump">Split the Flash Dump</a>

Work on a copy of the original flash dump.

```console
$ cp 02_extract_firmware/archer-flash-dump1-8MiB.bin 03_firmware_split/
$ cd 03_firmware_split
```

Split the 8 MiB dump according to the flash layout confirmed by the Linux boot log.

```console
$ dd if=archer-flash-dump1-8MiB.bin of=01-uboot.bin  bs=1 skip=$((0x000000)) count=$((0x020000)) status=progress
131072+0 records in
131072+0 records out
131072 bytes (131 kB, 128 KiB) copied, 0.149744 s, 875 kB/s

$ dd if=archer-flash-dump1-8MiB.bin of=02-kernel.bin bs=1 skip=$((0x020000)) count=$((0x100000)) status=progress
865828 bytes (866 kB, 846 KiB) copied, 1 s, 866 kB/s
1048576+0 records in
1048576+0 records out
1048576 bytes (1.0 MB, 1.0 MiB) copied, 1.21529 s, 863 kB/s

$ dd if=archer-flash-dump1-8MiB.bin of=03-rootfs.bin bs=1 skip=$((0x120000)) count=$((0x6b0000)) status=progress
6577242 bytes (6.6 MB, 6.3 MiB) copied, 8 s, 822 kB/s
7012352+0 records in
7012352+0 records out
7012352 bytes (7.0 MB, 6.7 MiB) copied, 8.51448 s, 824 kB/s

$ dd if=archer-flash-dump1-8MiB.bin of=04-config.bin bs=1 skip=$((0x7d0000)) count=$((0x020000)) status=progress
131072+0 records in
131072+0 records out
131072 bytes (131 kB, 128 KiB) copied, 0.157941 s, 830 kB/s

$ dd if=archer-flash-dump1-8MiB.bin of=05-art.bin bs=1 skip=$((0x7f0000)) count=$((0x010000)) status=progress
65536+0 records in
65536+0 records out
65536 bytes (66 kB, 64 KiB) copied, 0.0799275 s, 820 kB/s
```


  <br/>

## <a name="link-identify-the-firmware-components">Identify the Firmware Components</a>

Identify the five extracted regions.

```console
$ file 0*.bin
01-uboot.bin:  data
02-kernel.bin: firmware 7500 v1 TP-LINK Technologies ver. 1.0, version 3.15.3, 8060928 bytes or less, at 0x200 853771 bytes LZMA compressed data, non-streamed, size 2442812, at 0x100000 7012352 bytes
03-rootfs.bin: Squashfs filesystem, little endian, version 4.0, lzma compressed, 5959387 bytes, 704 inodes, blocksize: 131072 bytes, created: Mon May 11 07:31:23 2015
04-config.bin: BIOS (ia32) ROM Ext. (3*512) instruction 0x0f000000
05-art.bin:    data
```

The important results are:

- `01-uboot.bin` contains the bootloader.
- `02-kernel.bin` contains a TP-Link wrapper followed by an LZMA-compressed kernel at offset `0x200`.
- `03-rootfs.bin` is a SquashFS 4.0 filesystem using LZMA.
- `04-config.bin` and `05-art.bin` are not recognized by `file` (they must be analyzed separately).

Run `binwalk` for additional signatures.

```console
$ binwalk 0*.bin
Target File:   01-uboot.bin
MD5 Checksum:  95ea08dd8ba5ee1275fa5e5018ad443a
Signatures:    411

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
70688         0x11420         Certificate in DER format (x509 v3), header length: 4, sequence length: 64
92928         0x16B00         U-Boot version string, "U-Boot 1.1.4 (May 11 2015 - 15:13:34)"
93104         0x16BB0         CRC32 polynomial table, big endian

Target File:   02-kernel.bin
MD5 Checksum:  0ecf2a500162788b8c2c7dc96211fa35
Signatures:    411

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             TP-Link firmware header, firmware version: 0.0.3, image version: "", product ID: 0x0, product version: 1962934273, kernel load address: 0x0, kernel entry point: 0x80002000, kernel offset: 8060928, kernel length: 512, rootfs offset: 853771, rootfs length: 1048576, bootloader offset: 7012352, bootloader length: 0
512           0x200           LZMA compressed data, properties: 0x5D, dictionary size: 33554432 bytes, uncompressed size: 2442812 bytes

Target File:   03-rootfs.bin
MD5 Checksum:  e14ef2a74b3d2f6bdacefe5409029ddb
Signatures:    411

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             Squashfs filesystem, little endian, version 4.0, compression:lzma, size: 5959387 bytes, 704 inodes, blocksize: 131072 bytes, created: 2015-05-11 07:31:23

Target File:   04-config.bin
MD5 Checksum:  de7a973a3fcd5df9de124e6a0865ed59
Signatures:    411

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------

MD5 Checksum:  8ccaa95d0fcb8c6a2e53b8a281543d89
Signatures:    411

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
```

`binwalk` correctly identifies the LZMA stream at `0x200` and the SquashFS root filesystem.
Some TP-Link header field labels shown by `binwalk` are not reliable enough to use as authoritative structure documentation.
Later steps validate the fields that are actually needed.


  <br/>

## <a name="link-extract-the-squashfs-root-filesystem">Extract the SquashFS Root Filesystem</a>

Use a separate working directory for the root filesystem.

```console
$ cd ..
$ cp 03_firmware_split/03-rootfs.bin 04_rootfs
$ cd 04_rootfs
```

First inspect the SquashFS superblock without extracting anything.

```console
$ unsquashfs -s 03-rootfs.bin
Found a valid SQUASHFS 4:0 superblock on 03-rootfs.bin.
Creation or last append time Mon May 11 09:31:23 2015
Filesystem size 5959387 bytes (5819.71 Kbytes / 5.68 Mbytes)
Compression lzma
Block size 131072
Filesystem is exportable via NFS
Inodes are compressed
Data is compressed
Uids/Gids (Id table) are compressed
Fragments are compressed
Always-use-fragments option is specified
Xattrs are compressed
Duplicates are removed
Number of fragments 71
Number of inodes 704
Number of ids 1
Number of xattr ids 0
```

The superblock is valid and confirms SquashFS 4.0 with LZMA compression.

A normal `unsquashfs` extraction fails while decoding the filesystem:

```console
$ unsquashfs -d rootfs 03-rootfs.bin
lzma uncompress failed with error code 0
read_block: failed to read block @0x5ae81f
read_fragment_table: failed to read fragment table index
FATAL ERROR: File system corruption detected
```

This does not by itself prove that the flash dump is corrupted. 
Two independent flash dumps were already identical, and the SquashFS superblock is valid.
The next test is therefore to use `sasquatch`, which supports several vendor-modified SquashFS variants.

Build `sasquatch`.

```console
$ git clone https://github.com/devttys0/sasquatch
$ cd sasquatch
$ git pull origin pull/56/head
$ ./build.sh
$ cd ..
```

Extract the filesystem with `sasquatch`.

```console
$ sasquatch -d rootfs 03-rootfs.bin
SquashFS version [4.0] / inode count [704] suggests a SquashFS image of the same endianess
Parallel unsquashfs: Using 1 processor
Trying to decompress using default lzma decompressor...
Successfully decompressed with default lzma decompressor
661 inodes (773 blocks) to write


Trying to decompress with gzip...

Trying to decompress with lzma-adaptive...

Detected lzma-adaptive compression
[=========================================/] 773/773 100%

created 507 files
created 43 directories
created 85 symlinks
created 69 devices
created 0 fifo
```

The successful extraction confirms that the root filesystem is usable.


  <br/>

## <a name="link-analyze-the-runtime-boot-sequence-and-authentication-files">Analyze the Runtime Boot Sequence and Authentication Files</a>

The extracted SquashFS root filesystem contains only a very small static `/etc`, so the next objective is to determine how the router builds its runtime environment and which authentication files are actually used after boot.

Start from the extracted root filesystem.

```console
$ cd rootfs
```

Inspect the authentication-related files and links.

```console
$ ls -l etc/passwd etc/shadow etc/securetty etc/inittab
lrwxrwxrwx 1 root root 11 Aug 30 01:30 etc/passwd -> /tmp/passwd
-rwxr-xr-x 1 root root 59 May 11  2015 etc/shadow
-rwxr-xr-x 1 root root 124 May 11  2015 etc/securetty
-rwxr-xr-x 1 root root 85 May 11  2015 etc/inittab
```

The important observation is:

```text
/etc/passwd -> /tmp/passwd
```

The SquashFS therefore does not contain the runtime `passwd` database itself. It contains only a symbolic link to a file created under `/tmp`.

Inspect the init configuration.

```console
$ cat etc/inittab
::sysinit:/etc/rc.d/rcS
::respawn:/sbin/getty ttyS0 115200
::shutdown:/bin/umount -a
```

This gives the normal userspace startup sequence:

```text
kernel
  |
  v
/sbin/init
  |
  v
/etc/inittab
  |
  +-- /etc/rc.d/rcS
  |
  +-- /sbin/getty ttyS0 115200
```

The serial login prompt is therefore intentionally started on `ttyS0`.

Inspect the startup script.

```console
$ cat etc/rc.d/rcS
#!/bin/sh

# This script runs when init it run during the boot process.
# Mounts everything in the fstab

mount -a
#mount -o remount +w /

#
# Mount the RAM filesystem to /tmp
#

mount -t ramfs -n none /tmp
mount -t ramfs -n none /var

export PATH=$PATH:/etc/ath

#insmod /lib/modules/2.6.15/net/ag7100_mod.ko
#insmod /lib/modules/2.6.15/net/ag7240_mod.ko

#
# Set lo eth1 up
#
ifconfig lo 127.0.0.1 up
#ifconfig eth1 up

#
# insert netfilter/iptables modules
#

/etc/rc.d/rc.modules

#
# Start Our Router Program
#

/usr/bin/httpd &
/usr/bin/uclited &

# when processes uses page-cache more than 30% of system memory,
# lets force them to write
echo 20 > /proc/sys/vm/dirty_ratio

# when the dirty pages cross more than 5% of sys memory,
# kick in the pdflush
echo 5 > /proc/sys/vm/dirty_background_ratio

echo 524288 > /proc/sys/net/ipv4/ipfrag_high_thresh
```

The key runtime behavior is now clear:

```text
SquashFS /etc/passwd
        |
        v
symlink to /tmp/passwd
        |
        v
/tmp is replaced by a RAM filesystem during rcS
        |
        v
/tmp/passwd must be generated at runtime
```

The two vendor daemons started immediately afterward are:

```text
/usr/bin/httpd
/usr/bin/uclited
```

Search those binaries for references to the runtime password file.

```console
$ strings -a usr/bin/uclited | grep -niE -C5 'passwd|shadow|/tmp/passwd|root:'
7251:root:x:0:0:root:/root:/bin/sh
7252:Admin:x:0:0:root:/root:/bin/sh
...
7321:/tmp/passwd
7322-%s -p %s -a %s
7323:smbpasswd
...
7336:/tmp/vsftp/etc/passwd
7337-%s:%s:%d:%d;
7338:passwd: %s
```

The same code/data family is also present in `httpd`.

```console
$ strings -a usr/bin/httpd | grep -niE -C5 'passwd|shadow|/tmp/passwd|root:'
11062:/tmp/passwd
11063-%s -p %s -a %s
11064:smbpasswd
11065:/tmp/vsftp/etc/passwd
...
11147:root:x:0:0:root:/root:/bin/sh
11148:Admin:x:0:0:root:/root:/bin/sh
...
```

This does not yet prove the exact contents of the live `/tmp/passwd`, but it establishes that the vendor userspace contains the logic and templates required to build runtime account files.

The static SquashFS therefore provides:

```text
/etc/shadow          static password hash database
/etc/passwd          symlink only
/usr/bin/uclited     runtime vendor logic
/usr/bin/httpd       runtime vendor logic
```

while the normal boot creates runtime state under `/tmp`.

The next step is to identify the static root hash and determine whether its recovered password is accepted by the live serial login.


  <br/>

## <a name="link-identify-the-root-password-hash-and-test-the-static-credentials">Identify the Root Password Hash and Test the Static Credentials</a>

Inspect the password hash stored in the extracted SquashFS.

```console
$ awk -F':' '{print $2}' etc/shadow
$1$GTN.gpri$DlSyKvZKMR9A9Uj9e9wR3/
```

The `$1$` prefix identifies the format as Unix `md5crypt`.

The string is structured as:

```text
$1                      md5crypt
GTN.gpri                salt
DlSyKvZKMR9A9Uj9e9wR3/	md5crypt output
```

Initial obvious password guesses can be checked locally by recomputing md5crypt with the same salt.

```console
$ openssl passwd -1 -salt 'GTN.gpri' 'root'
$1$GTN.gpri$siejy7QaickUwgoLbW9H4/

$ openssl passwd -1 -salt 'GTN.gpri' 'admin'
$1$GTN.gpri$0z9rgEjFvvB1KaOcRN4Mt0

$ openssl passwd -1 -salt 'GTN.gpri' 'tpl'
$1$GTN.gpri$jwJ8H/B1.GPih5filHzJO.
```

None of these matches the hash stored in `/etc/shadow`.

Dictionary attacks with specific password lists succeded.

A candidate password, `sohoadmin`, is then verified locally against the exact salt.

```console
$ openssl passwd -1 -salt GTN.gpri sohoadmin
$1$GTN.gpri$DlSyKvZKMR9A9Uj9e9wR3/
```

This output is byte-for-byte identical to the hash extracted from `/etc/shadow`.

Therefore:

```text
static SquashFS root password = sohoadmin
```

This is not a guess: the md5crypt result matches exactly.

Test the credential on the live serial login.

```console
Archer C7 login: root
Password:
Login incorrect
```

The result is unexpected:

```text
Static /etc/shadow hash -> sohoadmin confirmed
Live serial login       -> rejects sohoadmin
```

The next step is therefore not to search for another plaintext corresponding to this hash. The hash is already solved.

The real question becomes:

```text
Why does the live login path reject a password that exactly matches
the static SquashFS /etc/shadow entry?
```


  <br/>

## <a name="link-determine-why-the-static-root-password-does-not-authenticate">Determine Why the Static Root Password Does Not Authenticate</a>

The first possibility to eliminate is a serial-console restriction on root logins.

Inspect `/etc/securetty`.

```console
$ cat etc/securetty
tty1
tty2
tty3
tty4
tty5
tty6
tty7
tty8
ttyS0
ttyS1
ttyS2
ttyS3
pts/0
pts/1
pts/2
pts/3
pts/4
pts/5
pts/6
pts/7
pts/8
pts/9
```

`ttyS0` is explicitly authorized.

The serial login path is also confirmed by `/etc/inittab`:

```console
$ cat etc/inittab
::sysinit:/etc/rc.d/rcS
::respawn:/sbin/getty ttyS0 115200
::shutdown:/bin/umount -a
```

Therefore the failure is not explained by `/etc/securetty`, and the login prompt is intentionally attached to the UART being used.

Next, inspect the actual login and init binaries.

```console
$ ls -l bin/sh bin/busybox sbin/init
-rwxr-xr-x 1 root root 301292 May 11  2015 bin/busybox
lrwxrwxrwx 1 root root      7 Aug 30 01:30 bin/sh -> busybox
lrwxrwxrwx 1 root root     14 Aug 30 01:30 sbin/init -> ../bin/busybox
```

Both shell and init resolve to the same BusyBox binary.

The more important runtime clue remains:

```text
/etc/passwd -> /tmp/passwd
```

and `/tmp/passwd` is not part of the static SquashFS image.

The vendor binaries contain both the runtime path and account templates:

```console
$ grep -aob '/tmp/passwd' usr/bin/uclited usr/bin/httpd
usr/bin/uclited:1095332:/tmp/passwd
usr/bin/httpd:1600152:/tmp/passwd
```

Combined with the earlier strings:

```text
root:x:0:0:root:/root:/bin/sh
Admin:x:0:0:root:/root:/bin/sh
/tmp/passwd
```

this establishes an important distinction:

```text
Static analysis:
    /etc/shadow exists in SquashFS
    hash resolves to sohoadmin

Runtime:
    /etc/passwd points to /tmp/passwd
    /tmp/passwd is generated after /tmp is mounted in RAM
```

At this stage, the exact live contents of `/tmp/passwd` are still unknown.

Possible explanations include:

```text
1. /tmp/passwd contains root:x:...
   -> authentication should normally delegate to /etc/shadow

2. /tmp/passwd contains a password hash directly
   -> the live password may differ from the static shadow password

3. the vendor BusyBox/login path applies additional behavior
```

The first two possibilities cannot be distinguished from the extracted SquashFS alone because the interesting file exists only after the normal runtime has been created.

The cleanest way forward is therefore to obtain a shell before the normal init sequence, then manually start the vendor runtime and inspect the generated `/tmp/passwd`.

Before anything else, we can try to ask U-Boot to boot the existing kernel with `init=/bin/sh`.


  <br/>

## <a name="link-override-the-effective-kernel-startup-with-u-boot-bootargs">Override the Effective Kernel Startup With U-Boot bootargs</a>

U-Boot already exposes a `bootargs` environment variable, so the least invasive attempt is to override only the init process for one boot.

No flash write is required.

At the U-Boot prompt, set a temporary command line using the real Linux runtime partition map and replace:

```text
init=/sbin/init
```

with:

```text
init=/bin/sh
```

```console
ap135> setenv bootargs console=ttyS0,115200 root=31:02 rootfstype=squashfs init=/bin/sh mtdparts=ath-nor0:128k(u-boot),1024k(kernel),6848k(rootfs),128k(config),64k(ART) mem=128M
```

Do not run `saveenv`.

Boot the original kernel directly from flash.

```console
ap135> bootm 0x9f020000
```

If the U-Boot `bootargs` override controlled the effective Linux startup, the kernel should launch `/bin/sh` instead of `/sbin/init`.

It does not.

The router follows its normal boot path and eventually returns to:

```text
Archer C7 login:
```

This proves that changing the U-Boot `bootargs` variable is not sufficient to alter the effective startup behavior of this TP-Link kernel.

The next question is therefore:

```text
Where does the effective kernel command line actually come from?
```

The answer will be obtained offline by extracting and decompressing the kernel itself.


  <br/>

## <a name="link-define-a-new-strategy">Define a New Strategy</a>

The next strategy is to keep the original TP-Link Linux 2.6.31 kernel and change only the minimum number of bytes needed to test the startup behavior.

No new kernel is compiled, and nothing is written back to flash. The modified copy will be loaded into RAM so every result can be observed before deciding the next step.

The test path we want to build is:

```text
original TP-Link kernel
        |
        | minimal binary patch
        v
patched kernel copy
        |
        | TFTP
        v
RAM
        |
        | U-Boot bootm
        v
observe the real boot behavior
```


  <br/>

## <a name="link-extract-the-lzma-kernel-payload">Extract the LZMA Kernel Payload</a>

Create the working directory and copy the 1 MiB kernel region into it.

```console
$ cp 03_firmware_split/02-kernel.bin 05_kernel_hacking/
$ cd 05_kernel_hacking
```

Inspect the kernel region again.

```console
$ binwalk 02-kernel.bin
DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             TP-Link firmware header, firmware version: 0.0.3, image version: "", product ID: 0x0, product version: 1962934273, kernel load address: 0x0, kernel entry point: 0x80002000, kernel offset: 8060928, kernel length: 512, rootfs offset: 853771, rootfs length: 1048576, bootloader offset: 7012352, bootloader length: 0
512           0x200           LZMA compressed data, properties: 0x5D, dictionary size: 33554432 bytes, uncompressed size: 2442812 bytes
```

The useful information here is:

```text
LZMA start offset      = 0x200
compressed LZMA size   = 853771 bytes
uncompressed size      = 2442812 bytes
dictionary size        = 32 MiB
```

The first `0x200` bytes are therefore not part of the compressed kernel payload.

The layout currently known is:

```text
02-kernel.bin
|
+-- 0x000000 - 0x0001ff : TP-Link wrapper/header, 512 bytes
|
+-- 0x000200 - ...      : LZMA-compressed Linux kernel
```

Extract exactly the LZMA payload reported by `binwalk`.

```console
$ dd if=02-kernel.bin of=kernel.lzma bs=1 skip=$((0x200)) count=853771 status=progress
853771+0 records in
853771+0 records out
853771 bytes (854 kB, 834 KiB) copied, 0.980174 s, 871 kB/s
```

Verify the extracted size.

```console
$ stat -c '%n %s' kernel.lzma
kernel.lzma 853771
```

Inspect the 13-byte LZMA "alone" header.

```console
$ xxd -g1 -l 13 kernel.lzma
00000000: 5d 00 00 00 02 3c 46 25 00 00 00 00 00  ]....<F%.....
```

The header can be read as:

```text
5d                         LZMA properties
00 00 00 02                dictionary size = 0x02000000 = 32 MiB
3c 46 25 00 00 00 00 00    uncompressed size, little-endian
```

The uncompressed size field is:

```text
0x25463c = 2442812 bytes
```

This matches the size reported by `binwalk`.

At this point we have established only the following:

```text
02-kernel.bin
|
+-- first 512 bytes
|   +-- TP-Link wrapper/header to be analyzed later
|
+-- offset 0x200
    +-- LZMA-compressed Linux kernel
        +-- compressed size   : 853771 bytes
        +-- uncompressed size : 2442812 bytes
```

The TP-Link wrapper itself is not interpreted yet.


  <br/>

## <a name="link-decompress-the-kernel">Decompress the Kernel</a>

Decompress the LZMA payload.

```console
$ xz --format=lzma -dc kernel.lzma > kernel.raw
```

Verify the resulting size.

```console
$ stat -c '%n %s' kernel.raw
kernel.raw 2442812
```

The result exactly matches the uncompressed size stored in the LZMA header.


  <br/>

## <a name="link-locate-the-embedded-kernel-command-line">Locate the Embedded Kernel Command Line</a>

Search for the runtime kernel command line.

```console
$ strings -a kernel.raw | grep 'console=ttyS0'
console=ttyS0,115200 root=31:02 rootfstype=squashfs init=/sbin/init mtdparts=ath-nor0:128k(u-boot),1024k(kernel),6848k(rootfs),128k(config),64k(ART) mem=128M
```

This is the same command line observed in the real Linux boot log.

Therefore the command line is physically embedded in the decompressed TP-Link kernel.

This also explains an earlier observation: changing U-Boot `bootargs` to request `init=/bin/sh` did not change the actual boot behavior. The TP-Link kernel contains its own command line and uses it.


  <br/>

## <a name="link-first-patch-change-init-sbin-init-in-the-embedded-command-line">First Patch: Change init=/sbin/init in the Embedded Command Line</a>

Create a copy of the raw kernel.

```console
$ cp kernel.raw kernel-patched.raw
```

Replace the 15-byte string `init=/sbin/init` with another 15-byte string, `init=/bin/sh` followed by three spaces.

The fixed length is important: no data in the kernel is shifted.

```console
$ python3 - <<'PY'
from pathlib import Path

p = Path("kernel-patched.raw")
d = p.read_bytes()

old = b"init=/sbin/init"
new = b"init=/bin/sh   "

print("old:", len(old), "bytes")
print("new:", len(new), "bytes")
print("occurrences:", d.count(old))

assert len(old) == len(new)
assert d.count(old) == 1

p.write_bytes(d.replace(old, new, 1))
PY
old: 15 bytes
new: 15 bytes
occurrences: 1
```

Verify the modified command line.

```console
$ strings -a kernel-patched.raw | grep 'console=ttyS0'
console=ttyS0,115200 root=31:02 rootfstype=squashfs init=/bin/sh    mtdparts=ath-nor0:128k(u-boot),1024k(kernel),6848k(rootfs),128k(config),64k(ART) mem=128M
```

The patch is present and the kernel size has not changed. Let's test whether this single patch is sufficient.


  <br/>

## <a name="link-recompress-the-first-patched-kernel">Recompress the First Patched Kernel</a>

The first binary patch exists only in `kernel-patched.raw`. U-Boot cannot boot that decompressed raw kernel directly through the same path as the original firmware, so the next step is to reconstruct the layers around it.

Start by recompressing it with the LZMA properties observed in the original TP-Link kernel.

```console
$ xz --format=lzma \
  --lzma1=lc=3,lp=0,pb=2,dict=32MiB \
  -c kernel-patched.raw > kernel-patched.lzma
```

The compressed stream is slightly different from the original because a few bytes in the raw kernel changed.

```console
$ stat -c '%n %s' kernel.lzma kernel-patched.lzma
kernel.lzma 853771
kernel-patched.lzma 853644
```

Inspect its 13-byte LZMA-alone header.

```console
$ xxd -g1 -l 13 kernel-patched.lzma
00000000: 5d 00 00 00 02 ff ff ff ff ff ff ff ff  ]............
```

At this point we do not yet change those bytes. The next question is simply: what object should be given to U-Boot?


  <br/>

## <a name="link-try-a-standard-u-boot-uimage">Try a Standard U-Boot uImage</a>

The obvious first attempt is a standard legacy U-Boot `uImage`.

A `uImage` is not another kernel compression format. It is a small U-Boot container placed around an existing payload:

```text
raw Linux kernel
      |
      | LZMA compression
      v
kernel-patched.lzma
      |
      | mkimage adds a U-Boot header
      v
kernel-patched.uImage
```

Install `mkimage` if necessary.

```console
$ apt install -y u-boot-tools
```

Wrap the LZMA stream as a MIPS Linux kernel image.

```console
$ mkimage \
  -A mips \
  -O linux \
  -T kernel \
  -C lzma \
  -a 0x80002000 \
  -e 0x80002000 \
  -n 'Archer C7 init-shell' \
  -d kernel-patched.lzma \
  kernel-patched.uImage
```

The resulting file contains a 64-byte legacy U-Boot header followed by the LZMA payload.

```text
0x000   64-byte uImage header
0x040   LZMA payload
```

The file size confirms the additional 64-byte header.

```console
$ stat -c '%n %s' kernel-patched.uImage
kernel-patched.uImage 853708
```

This is only an experiment. We have not yet established that the original TP-Link kernel region uses this format.


  <br/>

### Prepare TFTP for the first RAM test

The test network is:

```text
TFTP VM        192.168.1.111
Archer U-Boot  192.168.1.112
```

A yellow LAN port on the Archer is used for the Ethernet connection.

Install and configure `tftpd-hpa`.

```console
$ apt install -y tftpd-hpa
$ mkdir -p /srv/tftp
$ chmod 755 /srv/tftp
$ cp kernel-patched.uImage /srv/tftp/
$ chmod 644 /srv/tftp/kernel-patched.uImage
```

On this host IPv6 is disabled, so bind the daemon explicitly to the IPv4 address.

```console
$ cat >/etc/default/tftpd-hpa <<'EOF'
TFTP_USERNAME="tftp"
TFTP_DIRECTORY="/srv/tftp"
TFTP_ADDRESS="192.168.1.111:69"
TFTP_OPTIONS="--secure --ipv4"
EOF

$ dpkg --configure -a
$ systemctl restart tftpd-hpa
$ systemctl status tftpd-hpa --no-pager
$ ss -lunp | grep ':69'
```

Interrupt autoboot with `tpl`, then configure temporary network values in U-Boot.

```console
ap135> setenv ipaddr 192.168.1.112
ap135> setenv serverip 192.168.1.111
```

Do not run `saveenv`.

Load the standard `uImage` into RAM.

```console
ap135> tftp 0x81000000 kernel-patched.uImage
dup 1 speed 1000
Using eth1 device
TFTP from server 192.168.1.111; our IP address is 192.168.1.112
Filename 'kernel-patched.uImage'.
Load address: 0x81000000
...
Bytes transferred = 853708 (d06cc hex)
```

Boot it directly from RAM.

```console
ap135> bootm 0x81000000
## Booting image at 81000000 ...
   Uncompressing Kernel Image ... Too big uncompressed stream
LZMA ERROR 1 - must RESET board to recover
```

The kernel never starts. This is useful: U-Boot reached the LZMA decompression stage, so the first thing to investigate is the compressed stream itself rather than the `init=` patch.


  <br/>

## <a name="link-analyze-the-first-ram-boot-failure">Analyze the First RAM-Boot Failure</a>

Compare the beginning of the original LZMA stream with the newly generated one.

```console
$ xxd -g1 -l 13 kernel.lzma
00000000: 5d 00 00 00 02 3c 46 25 00 00 00 00 00  ]....<F%.....

$ xxd -g1 -l 13 kernel-patched.lzma
00000000: 5d 00 00 00 02 ff ff ff ff ff ff ff ff  ]............
```

The LZMA properties and dictionary are identical. The difference is the eight-byte uncompressed-size field.

The original TP-Link stream contains the real raw-kernel size:

```text
3c 46 25 00 00 00 00 00  -> 0x25463c -> 2442812 bytes
```

`xz` generated an LZMA-alone stream with that field set to all `0xff`, which means "unknown size". Given the `Too big uncompressed stream` failure and the explicit size stored by the original image, this field is the first concrete incompatibility to remove before testing anything else.

Restore the explicit uncompressed size without changing the compressed payload itself.

```console
$ python3 - <<'PY'
from pathlib import Path

raw = Path("kernel-patched.raw")
lz  = Path("kernel-patched.lzma")

d = bytearray(lz.read_bytes())
size = raw.stat().st_size

assert d[:5] == bytes.fromhex("5d00000002")

d[5:13] = size.to_bytes(8, "little")
lz.write_bytes(d)

print(d[:13].hex(" "))
PY
5d 00 00 00 02 3c 46 25 00 00 00 00 00
```

Validate the corrected stream by decompressing it and comparing it byte-for-byte with the patched raw kernel.

```console
$ xz --format=lzma -dc kernel-patched.lzma | cmp - kernel-patched.raw && echo "LZMA OK"
LZMA OK
```

Because the `uImage` contains the LZMA bytes and checksums over them, rebuild the `uImage` after correcting the stream.

```console
$ mkimage \
  -A mips \
  -O linux \
  -T kernel \
  -C lzma \
  -a 0x80002000 \
  -e 0x80002000 \
  -n 'Archer C7 init-shell' \
  -d kernel-patched.lzma \
  kernel-patched.uImage
```

The compression problem is now understood. Before attempting another boot, compare this standard U-Boot container with the known-good kernel block extracted from the Archer.


  <br/>

## <a name="link-compare-the-standard-uimage-with-the-original-tp-link-kernel-block">Compare the Standard uImage with the Original TP-Link Kernel Block</a>

The original `02-kernel.bin` already showed an LZMA stream beginning at offset `0x200`.

```console
$ xxd -g1 -s 0x200 -l 16 02-kernel.bin
00000200: 5d 00 00 00 02 3c 46 25 00 00 00 00 00 00 00 6f  ]....<F%.......o
```

The standard `uImage` places the same LZMA signature at `0x40` instead.

```console
$ xxd -g1 -s 0x040 -l 16 kernel-patched.uImage
00000040: 5d 00 00 00 02 3c 46 25 00 00 00 00 00 00 00 6f  ]....<F%.......o

$ xxd -g1 -s 0x200 -l 16 kernel-patched.uImage
00000200: a0 25 73 0b 43 02 bc e9 e0 5b 18 13 33 30 a9 0f  .%s.C....[..30..
```

So the two containers have different layouts:

```text
standard legacy uImage              original TP-Link kernel block
----------------------              -----------------------------
0x000  64-byte header               0x000  512-byte TP-Link wrapper
0x040  LZMA payload                 0x200  LZMA payload
```

This comparison does not require us to claim that a corrected standard `uImage` could never be accepted by this U-Boot build. 
It proves something more useful for this investigation: a standard `uImage` is not the structure of the known-good TP-Link kernel block.

The safest next step is therefore to stop changing the container format and preserve the original vendor wrapper.

In simple terms:

```text
kernel-patched.raw
      |
      | LZMA
      v
kernel-patched.lzma
      |
      | DO NOT replace the vendor container with a generic one
      | reuse the original TP-Link 512-byte wrapper
      v
kernel-patched.tplink
```

Only the fields that must change will be identified from the original image.


  <br/>

## <a name="link-rebuild-the-first-patched-tp-link-kernel-block">Rebuild the First Patched TP-Link Kernel Block</a>

The original compressed payload length is already known from the extraction step:

```text
853771 bytes = 0x000d070b
```

Search the first `0x200` bytes of the original kernel block for this value as a 32-bit big-endian integer.

```console
$ xxd -g4 -l 0x200 02-kernel.bin | grep '000d070b'
00000080: 00000200 000d070b 00100000 006b0000  .............k..
```

The value begins at offset `0x84`. Together with the observed LZMA start at `0x200`, the two fields needed for this reconstruction are now known from the original image:

```text
0x084   compressed-kernel length, 32-bit big-endian
0x200   compressed-kernel payload
```


Start from a copy of the untouched 1 MiB kernel region.

```console
$ cp 02-kernel.bin kernel-patched.tplink
```

Compare the original and patched compressed-kernel sizes:

```console
$ stat -c '%n %s' kernel.lzma kernel-patched.lzma
kernel.lzma 853771
kernel-patched.lzma 853644
```

The patched LZMA stream is therefore 127 bytes smaller than the original one.

Inject the corrected first-patch LZMA stream at `0x200`, overwrite the 127 bytes left from the original compressed stream with `0xff`, and update the compressed-length field at `0x84`.

```console
$ python3 - <<'PY'
from pathlib import Path

img_path = Path("kernel-patched.tplink")
lz_path  = Path("kernel-patched.lzma")

d = bytearray(img_path.read_bytes())
lz = lz_path.read_bytes()

assert len(d) == 0x100000

old_len = int.from_bytes(d[0x84:0x88], "big")
new_len = len(lz)

print("old kernel_len:", old_len)
print("new kernel_len:", new_len)

assert old_len == 853771
assert new_len == 853644
assert new_len <= old_len

d[0x200:0x200+new_len] = lz
d[0x200+new_len:0x200+old_len] = b"\xff" * (old_len-new_len)
d[0x84:0x88] = new_len.to_bytes(4, "big")

img_path.write_bytes(d)
PY
old kernel_len: 853771
new kernel_len: 853644
```

Verify that the rebuilt TP-Link kernel image still occupies the original 1 MiB kernel region:

```console
$ stat -c '%n %s' kernel-patched.tplink
kernel-patched.tplink 1048576
```

Verify that the LZMA payload starts at the expected offset `0x200`:

```console
$ xxd -g1 -s 0x200 -l 13 kernel-patched.tplink
00000200: 5d 00 00 00 02 3c 46 25 00 00 00 00 00  ]....<F%.....
```

Finally, verify that the compressed-kernel length stored in the TP-Link wrapper at `0x84` matches the actual size of `kernel-patched.lzma`:

```console
$ python3 - <<'PY'
from pathlib import Path

img = Path("kernel-patched.tplink").read_bytes()
lz  = Path("kernel-patched.lzma").read_bytes()

header_len = int.from_bytes(img[0x84:0x88], "big")

print("header kernel_len =", header_len)
print("actual LZMA len   =", len(lz))

assert header_len == len(lz)

print("WRAPPER OK")
PY
header kernel_len = 853644
actual LZMA len   = 853644
WRAPPER OK
```

The first patched boot image now has the same high-level layout as the original TP-Link kernel region:

```text
kernel-patched.tplink
|
+-- 0x000 - 0x1ff : original 512-byte TP-Link wrapper
|                    compressed-kernel length updated at 0x84
|
+-- 0x200 - ...     : kernel-patched.lzma
|
+-- remaining bytes : padding up to the original 1 MiB kernel-region size
```

Now the first patch can finally be tested without writing anything to flash.


  <br/>

## <a name="link-boot-the-first-patched-tp-link-kernel-from-ram">Boot the First Patched TP-Link Kernel from RAM</a>

Copy the reconstructed image to the already configured TFTP root.

```console
$ cp kernel-patched.tplink /srv/tftp/
$ chmod 644 /srv/tftp/kernel-patched.tplink
```

The failed LZMA test required a reset, and the earlier U-Boot network values were deliberately not saved. Interrupt autoboot again and restore them for this session.

```console
ap135> setenv ipaddr 192.168.1.112
ap135> setenv serverip 192.168.1.111
```

Load the complete 1 MiB TP-Link kernel block into RAM.

```console
ap135> tftp 0x81000000 kernel-patched.tplink
dup 1 speed 1000
Using eth1 device
TFTP from server 192.168.1.111; our IP address is 192.168.1.112
Filename 'kernel-patched.tplink'.
Load address: 0x81000000
Loading: #################################################################
         #################################################################
         #################################################################
         ##########
done
Bytes transferred = 1048576 (100000 hex)
```

Boot it from RAM.

```console
ap135> bootm 0x81000000
## Booting image at 81000000 ...
   Uncompressing Kernel Image ... OK

Starting kernel ...

Booting QCA955x
[    0.000000] Linux version 2.6.31--LSDK-9.2.0_U6.616 (root@liaozhiming) (gcc version 4.3.3 (GCC) ) #1 Mon May 11 15:16:26 CST 2015
[    0.000000] Ram size passed from bootloader =128M
```

This time Linux starts. More importantly, the boot log proves that the modified embedded command line is the one actually used by the running kernel.

```text
[    0.000000] Kernel command line: console=ttyS0,115200 root=31:02 rootfstype=squashfs init=/bin/sh    mtdparts=ath-nor0:128k(u-boot),1024k(kernel),6848k(rootfs),128k(config),64k(ART) mem=128M
```

So the first patch itself is active.

However, the router still completes its normal startup sequence and eventually returns to:

```text
Archer C7 login:
```

The result is now precise:

```text
The patched kernel boots successfully from RAM.
The embedded command line really contains init=/bin/sh.
The flash is still untouched.
But changing only that init= parameter is not sufficient to stop normal userspace startup.
```

This failed functional test gives the next question instead of the answer: is there another `/sbin/init` path inside the kernel that can still be used independently of the embedded command line?


  <br/>

## <a name="link-locate-the-kernel-s-internal-init-fallback-strings">Locate the Kernel's Internal Init Fallback Strings</a>

Search for init-related strings and their offsets.

```console
$ strings -a -t x kernel.raw | grep -E 'Failed to execute|/sbin/init|/etc/init|/bin/init|/bin/sh'
 1fc684 <4>Failed to execute %s
 1fc6b0 /sbin/init
 1fc6bc /etc/init
 1fc6c8 /bin/init
 1fc6d4 /bin/sh
 227cb0 console=ttyS0,115200 root=31:02 rootfstype=squashfs init=/sbin/init mtdparts=ath-nor0:128k(u-boot),1024k(kernel),6848k(rootfs),128k(config),64k(ART) mem=128M
```

```console
$ grep -aob '/sbin/init' kernel.raw
2082480:/sbin/init
2260201:/sbin/init
```

`strings -a -t x` reports the offset where each complete printable string begins, whereas `grep -aob` reports the exact offset of each searched byte sequence.

```console
$ printf '0x%x\n' 2082480 2260201
0x1fc6b0
0x227ce9
```

We now have the exact locations of both `/sbin/init` occurrences.

```
0x1fc6b0   standalone /sbin/init string
0x227ce9   /sbin/init inside the embedded kernel command line
```

The standalone occurrence at `0x1fc6b0` appears next to the following strings:

```
Failed to execute %s
/sbin/init
/etc/init
/bin/init
/bin/sh
```

This strongly suggests that this standalone `/sbin/init` belongs to the kernel's internal init fallback logic.
The occurrence at `0x227ce9` is definitively the `/sbin/init` contained inside the embedded kernel command line.

The first failed test patched only the second one.

The next build will patch both mechanisms.


  <br/>

## <a name="link-second-patch-patch-both-init-paths">Second Patch: Patch Both Init Paths</a>

The first RAM test established two facts before this second patch is attempted:

```text
1. The embedded init=/bin/sh command line is active.
2. The router still reaches its normal login prompt.
```

The nearby standalone `/sbin/init` string is therefore worth testing as a second mechanism, but it is still only a hypothesis until the next boot.

Always start again from the untouched `kernel.raw`.

```console
$ cp kernel.raw kernel-patched2.raw
```

Patch the standalone init path at `0x1fc6b0` and the embedded command-line parameter already modified during the first test.

The standalone C string occupies 10 characters plus its terminating NULL:

```text
/sbin/init\0
```

Replace the first 10 bytes with `/bin/sh` followed by three NULL bytes. The original terminating NULL remains in place.

```console
$ python3 - <<'PY'
from pathlib import Path

p = Path("kernel-patched2.raw")
d = bytearray(p.read_bytes())

# Standalone init/fallback string.
off = 0x1fc6b0
assert d[off:off+11] == b"/sbin/init\x00"
d[off:off+10] = b"/bin/sh\x00\x00\x00"

# Embedded kernel command line.
old = b"init=/sbin/init"
new = b"init=/bin/sh   "

assert len(old) == len(new) == 15
assert d.count(old) == 1

d = d.replace(old, new, 1)
p.write_bytes(d)
PY
```

Verify both changes.

```console
$ strings -a -t x kernel-patched2.raw | grep -E '/sbin/init|/bin/sh'
 1fc6b0 /bin/sh
 1fc6d4 /bin/sh
 227cb0 console=ttyS0,115200 root=31:02 rootfstype=squashfs init=/bin/sh    mtdparts=ath-nor0:128k(u-boot),1024k(kernel),6848k(rootfs),128k(config),64k(ART) mem=128M
```

Verify that the raw kernel size is unchanged.

```console
$ stat -c '%n %s' kernel.raw kernel-patched2.raw
kernel.raw 2442812
kernel-patched2.raw 2442812
```

Only a very small number of bytes differ from the original decompressed kernel.

```console
$ cmp -l kernel.raw kernel-patched2.raw | wc -l
18
```

This second raw kernel changes only 18 bytes. No kernel compilation is involved.


  <br/>

## <a name="link-recompress-and-rebuild-the-second-patched-kernel">Recompress and Rebuild the Second Patched Kernel</a>

The packaging rules are no longer unknown: the first test already established the LZMA-header requirement and the TP-Link wrapper layout. The second build therefore repeats the same validated process with `kernel-patched2.raw`.

Recompress it with the same LZMA properties.

```console
$ xz --format=lzma \
  --lzma1=lc=3,lp=0,pb=2,dict=32MiB \
  -c kernel-patched2.raw > kernel-patched2.lzma
```

Restore the explicit uncompressed-size field exactly as before.

```console
$ python3 - <<'PY'
from pathlib import Path

raw = Path("kernel-patched2.raw")
lz  = Path("kernel-patched2.lzma")

d = bytearray(lz.read_bytes())
assert d[:5] == bytes.fromhex("5d00000002")
d[5:13] = raw.stat().st_size.to_bytes(8, "little")
lz.write_bytes(d)
PY
```

Validate the compressed stream.

```console
$ xz --format=lzma -dc kernel-patched2.lzma | cmp - kernel-patched2.raw && echo "LZMA OK"
LZMA OK
```

Rebuild the 1 MiB TP-Link kernel block using the same two established locations:

```text
0x084   compressed-kernel length
0x200   LZMA payload
```

```console
$ cp 02-kernel.bin kernel-patched2.tplink

$ python3 - <<'PY'
from pathlib import Path

img_path = Path("kernel-patched2.tplink")
lz_path  = Path("kernel-patched2.lzma")

d = bytearray(img_path.read_bytes())
lz = lz_path.read_bytes()

assert len(d) == 0x100000

old_len = int.from_bytes(d[0x84:0x88], "big")
new_len = len(lz)

assert old_len == 853771
assert new_len <= old_len

d[0x200:0x200+new_len] = lz
d[0x200+new_len:0x200+old_len] = b"\xff" * (old_len-new_len)
d[0x84:0x88] = new_len.to_bytes(4, "big")

img_path.write_bytes(d)

print("new kernel_len:", new_len)
PY
new kernel_len: 853644
```

Verify the resulting image and wrapper field.

```console
$ stat -c '%n %s' kernel-patched2.tplink
kernel-patched2.tplink 1048576

$ python3 - <<'PY'
from pathlib import Path

img = Path("kernel-patched2.tplink").read_bytes()
lz  = Path("kernel-patched2.lzma").read_bytes()

assert int.from_bytes(img[0x84:0x88], "big") == len(lz)
assert img[0x200:0x205] == bytes.fromhex("5d00000002")
print("TP-Link kernel image ready")
PY
TP-Link kernel image ready
```

The TFTP server is already configured from the first-patch tests. Only copy the new image into its root.

```console
$ cp kernel-patched2.tplink /srv/tftp/
$ chmod 644 /srv/tftp/kernel-patched2.tplink
```


  <br/>

## <a name="link-configure-u-boot-for-the-second-ram-boot">Configure U-Boot for the Second RAM Boot</a>

Interrupt the normal boot and enter the U-Boot prompt with `tpl`. Because `saveenv` was never used, set the temporary IP addresses again for this U-Boot session.

```console
ap135> setenv ipaddr 192.168.1.112
ap135> setenv serverip 192.168.1.111
```

Do not run `saveenv`.

These values only need to exist for the current U-Boot session.


  <br/>

## <a name="link-load-the-second-patched-tp-link-kernel-block-into-ram">Load the Second Patched TP-Link Kernel Block into RAM</a>

Load the complete 1 MiB image at `0x81000000`.

```console
ap135> tftp 0x81000000 kernel-patched2.tplink
dup 1 speed 1000
Using eth1 device
TFTP from server 192.168.1.111; our IP address is 192.168.1.112
Filename 'kernel-patched2.tplink'.
Load address: 0x81000000
Loading: #################################################################
         #################################################################
         #################################################################
         ##########
done
Bytes transferred = 1048576 (100000 hex)
```

At this point:

```text
flash kernel   unchanged at 0x9f020000
patched kernel loaded only in RAM at 0x81000000
```

No flash write has occurred.


  <br/>

## <a name="link-boot-the-second-patched-kernel-from-ram">Boot the Second Patched Kernel from RAM</a>

Boot the image loaded at `0x81000000`.

```console
ap135> bootm 0x81000000
## Booting image at 81000000 ...
   Uncompressing Kernel Image ... OK

Starting kernel ...

Booting QCA955x
[    0.000000] Linux version 2.6.31--LSDK-9.2.0_U6.616 (root@liaozhiming) (gcc version 4.3.3 (GCC) ) #1 Mon May 11 15:16:26 CST 2015
[    0.000000] Ram size passed from bootloader =128M
```

The boot log now shows the patched command line.

```text
[    0.000000] Kernel command line: console=ttyS0,115200 root=31:02 rootfstype=squashfs init=/bin/sh    mtdparts=ath-nor0:128k(u-boot),1024k(kernel),6848k(rootfs),128k(config),64k(ART) mem=128M
```

The successful final build reaches a BusyBox shell instead of the normal login prompt.

```console
BusyBox v1.01 (2015.04.24-06:01+0000) Built-in shell (msh)
Enter 'help' for a list of built-in commands.

#
```

This proves that:

- the original TP-Link Linux 2.6.31 kernel is running;
- the original QCA955x-specific kernel and drivers are still used;
- the original SquashFS root filesystem is still mounted from flash;
- the booted kernel itself came from RAM;
- the normal `/sbin/init` startup path has been bypassed;
- the flash has not been modified.


  <br/>

## <a name="link-interpret-the-early-shell">Interpret the Early Shell</a>

This BusyBox build is very old and minimal. Some familiar commands are simply not present.

```console
# id
id: not found

# whoami
whoami: not found

# pwd
pwd: not found
```

This does not indicate a failed boot.

Listing the root filesystem confirms that the original embedded filesystem is available.

```console
# ls -la
drwxr-xr-x   15 0        0             200 May 11  2015 .
drwxr-xr-x   15 0        0             200 May 11  2015 ..
drwxr-xr-x    2 0        0             300 May 11  2015 bin
drwxr-xr-x    3 0        0            1042 May 11  2015 dev
drwxr-xr-x    8 0        0             407 May 11  2015 etc
drwxr-xr-x    5 0        0            1400 May 11  2015 lib
lrwxrwxrwx    1 0        0              11 May 11  2015 linuxrc -> bin/busybox
drwxr-xr-x    2 0        0               3 May 11  2015 mnt
drwxr-xr-x    2 0        0               3 May 11  2015 proc
drwxr-xr-x    2 0        0               3 May 11  2015 root
drwxr-xr-x    2 0        0             671 May 11  2015 sbin
drwxr-xr-x    2 0        0               3 May 11  2015 sys
drwxr-xr-x    2 0        0               3 May 11  2015 tmp
drwxr-xr-x    4 0        0              66 May 11  2015 usr
drwxr-xr-x    3 0        0              26 May 11  2015 var
drwxr-xr-x   10 0        0             141 May 11  2015 web
```

Immediately after entering this shell, `/tmp/passwd` does not exist.

```console
# cat /tmp/passwd
cat: /tmp/passwd: No such file or directory

# ls -la /tmp
drwxr-xr-x    2 0        0               3 May 11  2015 .
drwxr-xr-x   15 0        0             200 May 11  2015 ..
```

This is expected.

The normal firmware startup path is approximately:

```text
kernel
  |
  v
/sbin/init
  |
  v
/etc/inittab
  |
  v
/etc/rc.d/rcS
  |
  +-- mount ramfs on /tmp
  +-- mount ramfs on /var
  +-- load modules
  +-- start httpd
  +-- start uclited
```

The patched boot currently stops much earlier:

```text
kernel
  |
  v
/bin/sh
  |
  v
#
```

Therefore:

```text
/sbin/init     has not run
/etc/inittab   has not been processed
/etc/rc.d/rcS  has not run
uclited        has not started
/tmp           has not yet been replaced by the runtime ramfs
/tmp/passwd    has not yet been generated
```

This is exactly the desired position: a shell inside the original firmware environment before normal userspace initialization.


  <br/>

## <a name="link-next-step-recreate-the-runtime-initialization">Next Step: Recreate the Runtime Initialization</a>

The next stage of the reverse will be documented soon.
