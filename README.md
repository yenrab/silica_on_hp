# Silica on an HP desktop with no operating system

Planning documents for running a [Silica](#silica) application directly on an HP EliteDesk 705 G5
SFF, with no operating system on the machine, in the way Silica applications already run on an
ESP32-S3 board.

**Status: planning only.** Nothing here has been built or run. The repository holds documents, not
code.

## The goal

The first application is small on purpose: the Mac sends a message over TCP and the machine replies
`42`. Getting there needs a new Silica emit target, a runtime with no C library underneath it, an
Ethernet driver, and a TCP stack, with the driver, the network code and the threading written in
Silica.

## The approach in brief

- The image is a UEFI application, loaded by the machine's firmware over the network from the Mac.
  There is no third-party bootloader.
- Development is on the real hardware from the start, not in an emulator.
- The machine has no keyboard, no monitor and no serial port. Ethernet is the only channel, for
  loading the image and for the console.
- A run claims all of the machine's memory. One application runs at a time.
- Actor stacks stay growable, as the language specifies.
- Multi-core is designed in from the beginning, though the first application uses one core.

## The documents

| File | What it holds |
| --- | --- |
| [`IDEATION_2026-11-21.md`](IDEATION_2026-11-21.md) | The record of decisions and the discussion behind them, with the questions still open |
| [`HARDWARE.md`](HARDWARE.md) | The machine in detail: firmware, processor, chipset, PCI devices, network, storage |
| [`PRE_DEVELOPMENT_STEPS.md`](PRE_DEVELOPMENT_STEPS.md) | What has to be done before the first attempt to run Silica on the machine |
| [`DEVELOPMENT_LADDER.md`](DEVELOPMENT_LADDER.md) | Ten rungs from "runs as a firmware guest" to "replies 42 over TCP", each ending in a gate that can be run and checked |
| [`SECURITY_IMAGE_AUTHENTICATION.md`](SECURITY_IMAGE_AUTHENTICATION.md) | Who can replace the code the machine runs, and the options for preventing it |
| [`hardware_raw/`](hardware_raw/) | The raw captures the hardware reference was written from |

Start with the ideation record for the reasoning, or the development ladder for the plan.

## The ladder

| Rung | What it reaches |
| --- | --- |
| 1 | The language test programs run as a guest of the firmware |
| 2 | The image leaves the firmware: own page tables, fault handler, console and clock |
| 3 | Actors with growable stacks |
| 4 | The timer interrupt and idle |
| 5 | Device access from Silica |
| 6 | An Ethernet driver in Silica |
| 7 | ARP and ping |
| 8 | Interrupt-driven receive |
| 9 | UDP and a console written in Silica |
| 10 | TCP: reply `42` |

## Silica

Silica is a memory-safe functional systems language built on actors and regions, with no garbage
collector. Its compiler is self-hosted and already emits for macOS, Linux and three microcontroller
boards with no operating system. This project would add a general x86-64 UEFI target with a board
pack for this machine.
