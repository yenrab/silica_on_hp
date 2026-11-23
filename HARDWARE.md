# HP EliteDesk 705 G5 SFF: hardware reference

Target machine for the OS-free Silica port.

**State of the machine.** It has **no operating system**. The Ubuntu installation that made it the
`nix.local` Linux x86-64 build host has been wiped as part of the experiment; the NVMe SSD is blank.
There is no keyboard and no monitor attached. Nothing can be run on the box to inspect it any more,
so this document and the two raw captures are the record.

**Sources.**

- *Measured*: read over ssh without root on 2026-11-21, while Ubuntu 24.04 (kernel 6.8.0-88) was
  still installed. Raw captures: `hardware_raw/nix_hardware_2026-11-21.txt` and
  `hardware_raw/nix_chipset_2026-11-21.txt`.
- *Photo*: Lee's photos of the case and its labels.
- *Unverified*: general knowledge of this product line, not confirmed against HP's QuickSpecs (the
  copies found online refused to be fetched). Every such statement is marked **(unverified)**.
- *Not read*: needed root or a look inside the case, and was never captured. See the last section.

**Addresses are a cross-check, not constants.** Every bus number, memory address, I/O port and
interrupt number below was assigned by firmware R06 02.20.00 for one boot under Linux. A raw image
must discover them (PCI enumeration, ACPI tables, UEFI memory map).

---

## 1. Make and model

| Item | Value | Source |
| --- | --- | --- |
| Maker | HP Inc. | photo, measured |
| Product line | EliteDesk: HP's upper business-desktop line. The 705 is the AMD-processor member; "G5" is the fifth generation | measured (family `103C_53307F HP EliteDesk`) |
| Model | HP EliteDesk 705 G5 SFF (Small Form Factor), a horizontal desktop case that also stands on its side | photo, measured |
| Model year | Sold from 2019; this unit is labelled "2020" by the reseller and the firmware date is 2024-04-11 | photo, measured |
| Product number / SKU | 6JH69AV. The `AV` suffix is HP's configure-to-order base unit; the label reads "BU RCTO ELITEDESK 705 G5 SFF". The label also shows `6JW41AV` beside the warranty | photo, measured |
| Serial number | MXL0263RWF | photo |
| Regulatory model | TPC-P068-SF | photo |
| Mainboard | HP board `8617`. Most on-board devices carry PCI subsystem ID `103c:8617` | measured |
| DMI product version | `SBKPF` | measured |
| Embedded controller | "KBC Version 08.09.20", EC firmware release 8.9 | measured |
| Build ID | `19WWP7CT6**#SABA#DABA` | photo |
| Feature byte | `3E6J 6S6b 7B7H 7M7Q 7T7W 7maB apaq asau bhcA dUdp dqfP hKhZ jhk8 .hR` (HP's coded list of factory options) | photo |
| Country of manufacture | Mexico | photo |
| Original OS | Windows 10; the firmware still carries the licence table (ACPI `MSDM`) | photo, measured |
| Warranty class | 3 years parts / labour / on-site (`3y/3y/3y`) | photo |
| Power input | 100–240 V AC, 50–60 Hz, internal power supply | photo |
| Power supply rating | 180 W class internal supply **(unverified)** | |
| Case size | Roughly 27 × 30 × 9.5 cm **(unverified)** | |
| Local asset tag | 179302; described by the reseller as "16RAM, AMD", condition "fair" | photo |

Platform in one sentence: a single-socket AMD AM4 desktop built around a Ryzen PRO 3000-series
processor with integrated graphics ("Picasso"), paired with an AMD 400-series chipset, with a
Realtek gigabit Ethernet chip that includes remote-management functions.

### Case and connectors (photos)

Front, left to right: HP badge, a blank bay cover (slim optical bay position), one USB Type-C, four
USB Type-A marked SuperSpeed (the right-most with a charging mark), one headset jack, a drive
activity light, the power button with its light.

Rear: two audio jacks (line in, line out), two DisplayPort outputs, one HDMI output (HP's optional
video port position, fitted here), one RJ-45 Ethernet jack, six USB Type-A (a mix of USB 2 and
SuperSpeed), a blanked knockout marked for a serial port ("IOIOI A"), further blanked knockouts
along the top edge, two low-profile expansion slot openings (both covered), a lock slot, a padlock
loop, and the AC inlet with the power-supply fan grille.

Firmware inventory counts (measured, DMI): 34 port-connector entries, 5 system-slot entries, **4
memory-device entries (four memory slots)**.

---

## 2. Firmware

| Item | Value |
| --- | --- |
| Type | UEFI, 64-bit |
| Vendor / version | HP, family `R06`, version 02.20.00, dated 2024-04-11, BIOS release 20.0 |
| Secure Boot | Was **enabled** when read on 2026-11-21. An unsigned EFI image is refused until this is turned off in firmware setup |
| TPM | TPM 2.0, Infineon (ACPI device `IFX0785`, table `TPM2`) |
| Runtime services | Present |
| Setup / boot menu keys | F10 / F9 on HP business desktops **(unverified for this unit)** |
| HP settings seen as firmware variables | `DashControl`, `AfterPowerLoss`, `HP_BootOptions`, `FastBootStatus`, `HP_AudioAlert`, `ExUsbPortConfig`, `HP_DriveLock`, `HP_BiosUpdatePolicy`, `ExtendedDHCPTimeout`, `BiosPowerOn`, `ExpansionSlotConfig` |

Boot entries as last read (before the wipe):

| Entry | What | Active |
| --- | --- | --- |
| `Boot0005` | Ubuntu shim on the NVMe. **Stale: the file no longer exists** | was first |
| `Boot0004` | Kingston DataTraveler 3.0 USB stick (not attached) | yes |
| `Boot0003` | SATA CD | no |
| `Boot0001` | **IPv4 network boot**, Realtek NIC, MAC `e8:d8:d1:c5:92:fd` | yes |
| `Boot0002` | IPv6 network boot, same NIC | yes |
| `Boot0006` | Western Digital external HDD (not attached) | no |

Order was `0005, 0004, 0003, 0001, 0002, 0006`, menu timeout 0. With the disk blank and no USB
stick attached, the first entry that can succeed is the IPv4 network boot.

ACPI tables present (name, bytes): `APIC` 536 (the MADT), `FACP` 268, `FACS` 64, `HPET` 56,
`MCFG` 60, `DSDT` 59675, `SSDT1`..`SSDT10`, `IVRS` 208 (IOMMU description), `CRAT` 2064, `CDIT` 41,
`VFCT` 54404 (video BIOS image), `TPM2` 52, `MSDM` 85, `FPDT` 68, `ASF!` 110, `UEFI` 66, `WSMT` 40,
`OEML` 40. Contents not read.

ACPI sleep states offered: suspend-to-idle and S3 ("deep"). ACPI declares 32 processor objects
(`C000`..`C01F`); only 8 exist.

---

## 3. Processor

| Item | Value |
| --- | --- |
| Model | AMD Ryzen 5 PRO 3400G with Radeon Vega Graphics ("Picasso", Zen+ cores, socket AM4) |
| CPUID | Family 23 (17h), model 24 (18h), stepping 1; microcode `0x8108109` |
| Cores / threads | 4 cores × 2 threads = 8 logical processors |
| Frequency | 1.4–3.7 GHz range reported, boost enabled |
| Address sizes | 43 bits physical, 48 bits virtual (4-level paging) |
| Page sizes | 4 KB, 2 MB, 1 GB |

Topology: logical CPUs 0-1 are core 0, 2-3 core 1, 4-5 core 2, 6-7 core 3. APIC IDs are 0–7 in the
same order (even/odd pair = two threads of one core).

Caches: L1 data 32 KB and L1 instruction 64 KB per core, L2 512 KB per core, L3 4 MB shared.
Line size 64 bytes.

Features relevant to a raw runtime:

- **Instruction baseline:** x86-64-v3 (SSE–SSE4.2, SSE4a, AVX, AVX2, FMA, F16C, BMI1, BMI2,
  POPCNT/ABM, MOVBE, ADX). No AVX-512.
- **Time:** invariant TSC (`constant_tsc`, `nonstop_tsc`), `rdtscp`; local APIC timer that keeps
  running in idle (`arat`). Linux chose the TSC as clock source; HPET and ACPI PM timer also exist.
- **Interrupt controller:** memory-mapped local APIC with extended APIC IDs. No `x2apic` flag.
- **Paging and protection:** NX, PAT, MTRR, global pages, SMEP, SMAP, `fsgsbase`.
- **State save:** FXSR, XSAVE, XSAVEOPT, XSAVEC.
- **Other:** AES-NI, SHA, PCLMULQDQ, RDRAND, RDSEED, CLFLUSHOPT, CLZERO, MONITOR/MWAIT, MWAITX,
  `cmpxchg16b`, `syscall`.
- **Per-core watchdog timer** feature (`wdt`).
- **Virtualisation:** AMD-V with nested paging; SEV and SEV-ES reported.

---

## 4. Memory

- Four memory slots (DMI). Module sizes, speeds and which slots are filled: not read.
- Sold as 16 GB. Linux saw 14.2 GB (`MemTotal` 14,202,128 kB); the rest is consistent with memory
  set aside for the integrated graphics and firmware.
- DDR4 unbuffered modules, up to 64 GB **(unverified)**.
- RAM as Linux laid it out: `0x0000_0000`–`0x5FFF_FFFF` (1.5 GB) and
  `0x1_0000_0000`–`0x4_1FFF_FFFF` (12.5 GB).
- Device memory window below 4 GB: `0xE000_0000`–`0xF08F_FFFF`.

---

## 5. Chipset

"Chipset" on this machine is two things. The Ryzen processor is a system-on-chip that contains its
own controller hub (AMD calls it the FCH), and the mainboard adds a separate AMD 400-series chipset
chip that hangs off the processor by four PCI Express lanes. Section 5.1 covers the hub inside the
processor, 5.2 the separate chip.

### 5.1 Controller hub inside the processor (AMD FCH, "Raven/Picasso")

| Function | Detail (measured unless marked) |
| --- | --- |
| Root complex | `00:00.0`, `1022:15d0` |
| PCI Express root ports | `00:01.2`: 4 lanes at 8 GT/s (PCIe 3.0), running at full speed, leads to the 400-series chipset. `00:01.6`: 4 lanes at 8 GT/s, running at full speed, leads to the NVMe SSD. ACPI also names a graphics-slot port (`PEG0`) and ports `GPP2`..`GPP6` that are not populated |
| Internal bus | `00:08.1`, 16 lanes at 8 GT/s, leads to bus 0b: graphics, audio, security processor, two USB controllers |
| PCI configuration access | Memory-mapped (ECAM) for buses 00–3f, described by ACPI `MCFG`; legacy ports `0xCF8`/`0xCFC` also work |
| IOMMU | AMD-Vi at `00:00.2` (`1022:15d1`), described by ACPI `IVRS`. Capability word `0x190b640f`, feature word `0x4f77ef22294ada`. Interrupt remapping was active. Linux formed only 4 isolation groups: everything behind the two external root ports (chipset, Ethernet, NVMe) shares one group, and all internal devices share another |
| Interrupt controllers | Local APIC at `0xFEE0_0000`. **Two I/O APICs**, at `0xFEC0_0000` and `0xFEC2_0000`. Legacy dual 8259 PICs present. PCI interrupt-routing registers at ports `0xC00`/`0xC01` |
| Message-signalled interrupts | Every device that Linux drove used MSI or MSI-X (see 5.3), so a raw runtime can avoid I/O APIC routing for PCI devices |
| Timers | HPET (ACPI `HPET`, device `PNP0103`), 8254 interval timer, ACPI power-management timer, real-time clock |
| Real-time clock | CMOS RTC at ports `0x70`/`0x71`, interrupt 8; extended CMOS at `0x72`/`0x73` |
| ACPI power-management block | Ports `0x400`–`0x4CF`; SCI on interrupt 9 |
| FCH power-management registers | Index/data ports in `0xCD0`–`0xCDB` (AMD's PM register pair at `0xCD6`/`0xCD7` falls in this range) |
| SMBus / I2C | `00:14.0`, `1022:790b` rev 61. Linux `i2c_piix4` matches it |
| **Hardware watchdog** | The same device is matched by Linux's `sp5100_tco` watchdog driver. The module was not loaded, so the watchdog is known to be present by ID but is untested |
| LPC / ISA bridge | `00:14.3`, `1022:790e` rev 51. The embedded controller, TPM and legacy devices sit behind it |
| GPIO controller | ACPI device `AMDI0030` (`\_SB.GPIO`), interrupt 7, driven by Linux `pinctrl_amd`. A second GPIO block for the 400-series chipset is declared as `AMDIF030` (`\_SB.PTIO`) |
| USB | Two xHCI controllers: `0b:00.3` (`1022:15e0`) with 4 USB 2 + 4 USB 3.1 root ports, three of each wired to connectors; `0b:00.4` (`1022:15e1`) with 1 + 1 root port. Both report USB 3.1 (10 Gb/s-capable root hubs) and use MSI-X with 8 vectors |
| USB Type-C | A Type-C policy controller is exposed through ACPI (`USBC000`, UCSI); a firmware variable is named `CCG5` (Cypress CCG5 family) |
| Audio | HD Audio controller `0b:00.6` (`1022:15e3`) with a **Conexant CX20632** codec (jacks: front headphone, mic, line in, line out). Separate HDMI/DisplayPort audio controller `0b:00.1` (`1002:15de`) |
| Security processor | AMD Platform Security Processor `0b:00.2` (`1022:15df`), also the cryptographic coprocessor; 2 MSI-X vectors |
| Thermal | Processor temperature through function `00:18.3`; an ACPI thermal zone exists |
| Legacy keyboard controller | Present: ports `0x60`/`0x64`, interrupt 1 (keyboard) and 12 (mouse), ACPI devices `PS2K`/`PS2M`. The controller exists even though no PS/2 sockets are fitted on this unit's rear panel |
| PC speaker | Firmware declares the speaker device (`PNP0800`); Linux registered `pcspkr`. Whether a speaker is physically fitted: not read |
| Embedded controller | ACPI `PNP0C09` at ports `0x62`/`0x66` |
| Other reserved ports | `0x10`–`0x1F`, `0x80` (POST code), `0x92` (fast A20/reset), `0xB0`/`0xB1` (SMI command), `0x4D0`/`0x4D1` (interrupt edge/level), `0x680`–`0x6FF`, `0x200`–`0x23F` with memory `0xFEDB_1000` |
| Legacy serial | Standard addresses `0x3F8`/`0x2F8`/`0x3E8`/`0x2E8` were probed by Linux and **no UART answered** |

### 5.2 Separate chipset chip (AMD 400-series)

Identity: Linux names the functions "AMD 400 Series Chipset". Two of its ports carry the subsystem
ID `1b21:3306`, which belongs to ASMedia, the company that designs this chipset family for AMD. HP
markets the chipset on this model as "AMD PRO 565", which is understood to be the business-desktop
name for the same silicon as the consumer B450/B550A **(unverified)**.

It is one PCI Express device with three functions, connected to the processor by 4 lanes at
8 GT/s (running at full speed):

**Function 0, USB controller** (`01:00.0`, `1022:43d2` rev 01)

- xHCI, USB 3.1; root hub reports 10 Gb/s capability.
- 14 USB 2 root ports and 8 USB 3 root ports.
- Wired on this board: USB 2 ports 1, 3, 4, 5, 6, 11, 12 are external connectors and port 9 is
  hard-wired to an internal device position; USB 3 ports 1, 3, 4, 5, 6 are external and port 7 is
  hard-wired. The rest are marked unused by the firmware.
- MSI-X, 8 vectors. Register block 32 KB at `0xF082_0000`.

**Function 1, SATA controller** (`01:00.1`, `1022:43c8` rev 01)

- AHCI version 1.3.1, AHCI-only mode (no legacy IDE emulation).
- Capability register `0xEF36FF27`, decoded:
  - 8 ports implemented in the controller (ACPI names 6: `PRT0`..`PRT5`);
  - 6 Gb/s (SATA III) interface speed;
  - 32 command slots per port, native command queuing;
  - 64-bit addressing;
  - port multiplier support, external SATA support;
  - staggered spin-up, aggressive link power management, partial and slumber power states;
  - activity LED support, command list override, SNotification register;
  - no enclosure management, no command-completion coalescing, no FIS-based switching.
- Extended capability `0x3C`: device sleep, aggressive device-sleep management, automatic
  partial-to-slumber transitions. No BIOS/OS handoff, no NVMHCI.
- No drive was attached to any port.
- MSI, 1 vector. Register block 128 KB at `0xF080_0000`; option ROM at `0xF088_0000`.

**Function 2, PCI Express switch** (`01:00.2`, `1022:43c6`, downstream ports `1022:43c7`)

| Port | Capability | State when read | Leads to |
| --- | --- | --- | --- |
| `02:00.0` | 5 GT/s (PCIe 2.0), 1 lane | no link | bus 03, empty |
| `02:01.0` | 5 GT/s, 1 lane | link at 2.5 GT/s, 1 lane | bus 04, no device listed |
| `02:02.0` | 5 GT/s, 1 lane | link at 2.5 GT/s, 1 lane | bus 05: Realtek Ethernet chip |
| `02:03.0` | 5 GT/s, 1 lane | link at 2.5 GT/s, 1 lane | bus 06, no device listed |
| `02:04.0` | 5 GT/s, 4 lanes | no link | bus 07, empty (likely an expansion slot) |
| `02:08.0` | 8 GT/s (PCIe 3.0), 2 lanes | no link | bus 08, empty |
| `02:09.0` | 8 GT/s, 2 lanes | no link | bus 09, empty |

Which downstream port feeds which physical connector (the two low-profile slots, the second M.2
position, the Wi-Fi M.2 position) is not confirmed. HP's published layout is one M.2 storage slot
with 4 lanes, one with 2 lanes, and an M.2 2230 slot for a wireless card **(unverified)**; the
4-lane storage slot is the one the processor drives directly (`00:01.6`), where the NVMe SSD sits.

### 5.3 Interrupt delivery as Linux configured it

| Device | Mechanism | Vectors |
| --- | --- | --- |
| Timer, keyboard controller, RTC, ACPI SCI, GPIO | I/O APIC pins 2, 1 and 12, 8, 9, 7 | |
| IOMMU, root ports, chipset switch ports | MSI | 1 each |
| Chipset xHCI, both processor xHCI | MSI-X | 8 each |
| Chipset SATA | MSI | 1 |
| Realtek Ethernet | MSI-X | 1 |
| Realtek management EHCI | MSI | 1 |
| NVMe SSD | MSI-X | 33 (1 admin + 32 I/O queues) |
| Graphics | MSI-X | 1 |
| Security processor | MSI-X | 2 |
| Both audio controllers | MSI | 1 each |

### 5.4 Wake and power control

ACPI wake table: the two processor USB controllers and the PS/2 keyboard could wake the machine
from S3; the chipset bridges from S4. **Wake from the Ethernet chip was disabled** (`NIC`, S5,
disabled), so Wake-on-LAN would need a firmware setting changed before it can be relied on. The
firmware has an `AfterPowerLoss` setting (power on when AC returns).

---

## 6. PCI device list

| Address | Device | IDs | Resources (that boot) |
| --- | --- | --- | --- |
| `00:00.0` | Root complex | `1022:15d0` | |
| `00:00.2` | IOMMU | `1022:15d1` | |
| `00:01.2`, `00:01.6` | External PCIe root ports | `1022:15d3` | |
| `00:08.1` | Internal PCIe bridge | `1022:15db` | |
| `00:14.0` | SMBus (and watchdog) | `1022:790b` rev 61 | |
| `00:14.3` | LPC bridge | `1022:790e` rev 51 | |
| `00:18.0`–`.7` | Processor configuration | `1022:15e8`–`15ef` | |
| `01:00.0` | Chipset xHCI | `1022:43d2` | mem `F082_0000` 32 KB |
| `01:00.1` | Chipset SATA | `1022:43c8` | mem `F080_0000` 128 KB |
| `01:00.2`, `02:xx.0` | Chipset PCIe switch | `1022:43c6`, `43c7` | |
| `05:00.0` | **Ethernet, Realtek RTL8111/8168 family** | `10ec:8168` rev 0e | I/O `3300` (256); mem `F071_4000` 4 KB; mem `F070_0000` 16 KB |
| `05:00.1` | Realtek management UART #1 | `10ec:816a` | I/O `3200`; mem `F071_5000`, `F070_4000` |
| `05:00.2` | Realtek management UART #2 | `10ec:816b` | I/O `3100`; mem `F071_6000`, `F070_8000` |
| `05:00.3` | Realtek IPMI interface | `10ec:816c` | I/O `3000`; mem `F071_7000`, `F070_C000` |
| `05:00.4` | Realtek EHCI USB (1 port) | `10ec:816d` | mem `F071_8000`, `F071_0000` |
| `0a:00.0` | **NVMe SSD** | `1179:011a` | mem `F060_0000` 16 KB |
| `0b:00.0` | **Graphics, Radeon Vega** | `1002:15d8` rev d8 | mem `E000_0000` 256 MB; mem `F000_0000` 2 MB; I/O `2000`; mem `F050_0000` 512 KB |
| `0b:00.1` | HDMI / DisplayPort audio | `1002:15de` | mem `F058_8000` |
| `0b:00.2` | Platform Security Processor | `1022:15df` | mem `F040_0000`, `F058_C000` |
| `0b:00.3` | Processor xHCI | `1022:15e0` | mem `F020_0000` 1 MB |
| `0b:00.4` | Processor xHCI | `1022:15e1` | mem `F030_0000` 1 MB |
| `0b:00.6` | HD audio | `1022:15e3` | mem `F058_0000` |

---

## 7. Network

- Realtek RTL8111/8168 family gigabit Ethernet, `10ec:8168` revision `0e`, **MAC
  `e8:d8:d1:c5:92:fd`**. One lane at 2.5 GT/s behind the chipset switch. Linked at 1000 Mb/s full
  duplex when read. Linux driver was `r8169`, using one MSI-X vector.
- It is the variant with remote-management (DASH) functions, which is why the same chip exposes two
  UARTs, an IPMI interface and a one-port EHCI controller. Those are management channels, not
  sockets on the case. A `DashControl` firmware setting exists.
- The exact variant name Linux prints at driver start was not read.
- No Wi-Fi or Bluetooth device was present.

## 8. Storage

- KIOXIA (Toshiba) XG6 NVMe SSD, model `KXG60ZNV512G`, serial `30QF72EDFGNL`, firmware `AGHA4101`,
  512 GB: 1,000,215,216 sectors of 512 bytes (logical and physical).
- 4 lanes at 8 GT/s directly on the processor. Under Linux: 33 queues (1 admin, 32 I/O), queue
  depth 1024, MSI-X.
- **Now blank.** No partition table, no EFI system partition, nothing bootable.
- SATA: 8-port AHCI controller in the chipset, nothing attached.

## 9. Graphics and display

- Integrated Radeon Vega (Picasso). Outputs: two DisplayPort and one HDMI on the rear. Linux listed
  three DisplayPort connectors and one HDMI; all disconnected.
- Nothing is attached, so whether the firmware offers a framebuffer to a booting image with no
  display connected is unknown.
- Video BIOS image is supplied through ACPI (`VFCT`).

## 10. Serial and console options

- No legacy serial port answers at the standard addresses.
- The rear knockout marked for serial is blanked. No serial port will be added (decided
  2026-11-21: the image has to be loaded over the network anyway).
- The PC speaker device is declared by firmware; physical speaker not confirmed.
- Ethernet is the only channel to the outside: loading, output and input all go through it.

---

## 11. Never read, and how it can still be obtained

With no OS on the machine, the Linux commands that would have answered these are gone. The routes
left are: the raw Silica image reading the information itself, a look inside the case, or booting a
live Linux USB stick temporarily (which installs nothing).

| Wanted | Route now |
| --- | --- |
| Memory modules, slots, speed | SMBIOS tables, reachable from a UEFI image through the configuration table; or open the case |
| Firmware memory map | UEFI `GetMemoryMap` from the raw image |
| ACPI table contents (MADT, FADT, HPET, MCFG, IVRS) | Read from the raw image through the UEFI configuration table |
| PCI capability lists (MSI/MSI-X layout, link details) | Read configuration space from the raw image |
| Realtek chip variant | Read the chip's version register from the raw image |
| Whether a framebuffer exists with no display | Query the UEFI graphics protocol from the raw image |
| Speaker, free M.2 / SATA positions, which slot is which, PSU label | Open the case |
| HP firmware settings (Secure Boot, boot order, power-on after AC loss, Wake-on-LAN) | Firmware setup with a borrowed keyboard and display |
