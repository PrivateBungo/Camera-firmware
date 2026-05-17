# IP Camera Reverse Engineering Project Summary

## Overview

Objective:
- Investigate and potentially modify a low-cost PTZ IP camera for low-latency UGV teleoperation use.
- Explore custom firmware options (OpenIPC or similar).
- Reduce latency compared to stock H.265 firmware pipeline.
- Gain low-level access to bootloader and firmware environment.
- Understand the complete video pipeline and determine where latency originates.

The camera appears to belong to the XM/Xiongmai white-label ecosystem using a Novatek SoC platform.

The project has progressed from:
- basic UART probing
to:
- full bootloader compromise
- complete firmware extraction
- partition mapping
- recoverable firmware modification territory

At this point the device should be considered:
- fully researchable
- recoverable
- partially owned from a firmware perspective

---

# Hardware Identification

## Main SoC

Novatek NT98566

Observed directly from PCB photos.

Related internal platform identifiers seen in boot logs:

NA51089

This appears to be:
- Ethernet/platform subsystem naming
- internal Novatek platform reference
- not contradictory with NT98566

Architecture:
- ARM Cortex-A9 class Linux SoC
- Hardware H.264/H.265 encoding support
- Embedded Linux platform

---

# Bootloader / Firmware Details

## Bootloader

U-Boot 2019.04

Build timestamp:

Dec 26 2023 - 16:46:23 +0800

Firmware loader version:

LD_VER 03.00.06

DDR init string:

566_DRAM1_933_1024Mb 12/26/2023 16:43:00

CPU frequency detected:

960 MHz

RAM:

128 MiB

Toolchain:

arm-ca9-linux-uclibcgnueabihf-novatek-8.4.01-gcc.br_real

Buildroot:

Buildroot 2020.02.9-10-g744f210

---

# Storage

## SPI NOR Flash

Detected automatically during boot:

XM_XM25QH128C

Flash characteristics:

16 MiB SPI NOR

Boot log:

SPI NOR MID=00000020,TYPE=00000040,SIZE=00000018=>01000000

Meaning:
- 0x01000000 = 16 MiB total flash size

Flash interface:
- Accessible externally
- Likely compatible with SOIC8 clip + CH341A programmer

---

# UART Discovery

UART header discovered and confirmed functional.

PCB markings:

TXD
RXD
5V
GND

Another header also labeled:

GND
IO10
IO11

UART behavior:
- Clean boot logs observed
- Correct baud rate confirmed
- TX/RX verified operational

Likely serial settings:

115200 8N1

---

# Bootloader Access

## Autoboot Interrupt

Bootloader prompt:

Hit X to stop autoboot:

Interrupt key:
- Capital X

Behavior:
- Autoboot successfully interrupted
- Password prompt presented instead of direct shell

---

# Bootloader Password

## Successful Password

The following password successfully unlocked the U-Boot shell:

#Ux6@9V&4_Rz

This appears to be:
- OEM-generated
- Possibly manufacturing-derived
- Not a common/default password

After successful authentication:

nvt@na51089:

U-Boot shell access confirmed.

---

# U-Boot Environment Findings

## Board Information

board=nvt-na51055

soc=nvt-na51055

vendor=novatek

Potentially:
- reused BSP naming
- generic Novatek SDK board profile

---

# Flash Partition Layout

Recovered directly from U-Boot environment:

```text
mtdparts=spi_nor.0:
0x10000(loader),
0x30000(boot),
0x540000(romfs),
0x740000(usr),
0x180000(web),
0x80000(custom),
0x140000(mtd)
```

Expanded:

| Partition | Offset | Size | Purpose |
|---|---|---|---|
| loader | 0x000000 | 64 KB | first-stage loader |
| boot | 0x010000 | 192 KB | U-Boot / kernel bootstrap |
| romfs | 0x040000 | 5.25 MB | root filesystem |
| usr | 0x580000 | 7.25 MB | application binaries |
| web | 0xCC0000 | 1.5 MB | web interface |
| custom | 0xE40000 | 512 KB | settings/config |
| mtd | 0xEC0000 | 1.25 MB | persistent writable data |

---

# Firmware Update Mechanisms

## Existing Vendor Flashing Logic

U-Boot already contains flashing/update commands:

```bash
up=tftpboot 0x01000000 update.img;sf probe 0;flwrite
ua=tftpboot 0x01000000 upall_verify.img;sf probe 0;flwrite
```

Individual partition flashing commands also exist:

```bash
dr -> romfs
du -> usr
dw -> web
dc -> custom
da -> u-boot
```

Implications:
- vendor firmware update system exists
- TFTP flashing supported
- partition-level flashing supported
- recovery possible without hardware programmer

---

# Important Boot Observations

## SD Card Detection

Bootloader explicitly checks SD card during startup:

No card inserted

This occurs before SPI boot completes.

Implications:
- SD recovery/update path may exist
- Potential alternate boot mechanism
- Possibly manufacturing or recovery mode

However:
- U-Boot itself does NOT contain mmc/fatls commands
- SD support may exist only in lower-stage loader or Linux userspace

---

# MAC Address Warning

Observed during boot:

```text
Warning: eth_na51089 MAC addresses don't match:
Address in SROM is         00:80:48:ba:d1:30
Address in environment is  00:12:34:a6:56:a1
```

Potential causes:
- default environment values
- reflashed firmware
- manufacturing inconsistency

---

# PCB Observations

## KEY Label

PCB includes silkscreen label:

KEY

Potential meanings:
- factory reset
- recovery trigger
- GPIO boot strap
- manufacturing test mode

Not yet investigated fully.

---

# Networking

## U-Boot Network Configuration

Recovered from environment:

```bash
ipaddr=192.168.68.101
serverip=192.168.68.100
gatewayip=192.168.68.254
```

TFTP communication successfully established.

---

# Current Access Level

Confirmed:
- Physical UART access
- U-Boot access
- Password bypass completed
- Boot interruption working
- SPI flash identified
- TFTP networking operational
- Full firmware dump completed

Not yet confirmed:
- Linux shell access
- Root filesystem login
- Hidden telnet/SSH services
- SD recovery functionality

---

# Full Firmware Dump

## Dump Procedure

SPI flash dumped entirely from U-Boot:

```bash
sf read 0x02000000 0x0 0x1000000
```

Uploaded over TFTP:

```bash
tftpput 0x02000000 0x1000000 fullflash.bin
```

Transfer successful:
- 16 MiB extracted
- complete firmware image acquired

---

# Local Backup Locations

## Firmware Backup

Primary local backup:

```text
~/camera-backups/fullflash.bin
```

Temporary TFTP location:

```text
/srv/tftp/fullflash.bin
```

---

# Firmware Analysis Results

## Binwalk Findings

Main structures discovered:

| Offset | Content |
|---|---|
| 0x8008 | Flattened Device Tree |
| 0x10000 | LZMA compressed boot section |
| 0x40000 | SquashFS root filesystem |
| 0x580000 | SquashFS application partition |
| 0xCC0000 | SquashFS web partition |
| 0xEC0000+ | JFFS2 writable/config regions |

Compression:
- SquashFS + XZ
- JFFS2 writable storage

This matches the U-Boot partition map exactly.

---

# Reverse Engineering Hypotheses

The device likely belongs to:
- XM/Xiongmai OEM ecosystem
- White-label PTZ camera family
- Novatek-based Linux camera platform

Possible compatible ecosystems:
- OpenIPC
- XM firmware tooling
- Generic Novatek camera tooling

---

# Current Understanding Of The Problem

The original engineering goal is reducing end-to-end latency for UGV teleoperation.

Current hypothesis:
- latency is likely NOT hardware-limited
- NT98566 should be capable of reasonably low-latency H.264 streaming

More likely causes:
- aggressive H.265 buffering
- large GOP interval
- vendor encoder tuning
- RTSP buffering
- cloud relay architecture
- poor default stream settings
- multi-stage buffering pipeline

---

# Strategic Plan Going Forward

## Phase 1 — Firmware Archaeology

Goal:
- fully understand firmware architecture

Tasks:
- extract SquashFS filesystems
- inspect startup scripts
- inspect encoder configuration
- inspect RTSP stack
- inspect ONVIF implementation
- inspect web UI
- identify stream pipeline

Priority directories:
- /etc/init.d
- /usr/bin
- /etc
- CGI scripts
- RTSP configs
- ONVIF configs

Questions to answer:
- what binary controls encoding?
- where are bitrate/GOP settings?
- hidden low-latency settings?
- hidden MJPEG stream?
- hidden substream profiles?
- cloud relay architecture?
- internal buffering stages?

---

## Phase 2 — Access Escalation

Goal:
- gain Linux userspace access

Potential paths:
- hidden telnet
- hidden SSH
- web exploit
- CGI shell
- startup script modification
- custom firmware injection

---

## Phase 3 — Safe Modification Infrastructure

Goal:
- maintain recoverability while experimenting

Important:
- keep original firmware dump untouched
- avoid modifying bootloader initially
- prefer partition-level flashing
- use TFTP recovery path

Preferred targets:
- web partition
- usr partition
- startup scripts

Avoid:
- loader partition
- low-level boot code

---

## Phase 4 — Latency Optimization

Potential approaches:

### Option A — Existing Firmware Tuning
Best-case scenario.

Potential tweaks:
- lower GOP interval
- force H.264
- disable buffering
- lower encoder queues
- expose hidden stream profiles

### Option B — Binary/Config Patching
Possible:
- modify startup scripts
- patch encoder binaries
- modify RTSP stack

### Option C — Hybrid Firmware
Potential:
- retain vendor drivers
- replace userland services
- custom stream server

### Option D — Full OpenIPC Migration
Most ambitious option.

Benefits:
- total control
- modern stack
- low-latency tuning

Risks:
- driver compatibility
- Novatek support maturity
- higher brick risk

---

# Current Status

Current project stage:
- reverse engineering highly successful
- bootloader compromised
- firmware extracted
- partition layout understood
- network flashing functional
- recovery path available

The camera should now be considered:
- recoverable
- fully introspectable
- viable for advanced firmware experimentation

The project has transitioned from:
"hardware probing"

to:

"structural firmware analysis and controlled modification"

The remaining work is now primarily software archaeology and stream pipeline optimization rather than hardware access.
