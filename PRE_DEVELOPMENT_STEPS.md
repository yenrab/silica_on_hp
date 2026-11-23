# Pre-development steps

What to do before the first attempt to run Silica raw on the HP EliteDesk 705 G5 SFF.
Written 2026-11-21, revised 2026-11-22 for the development ladder. Companion to
`IDEATION_2026-11-21.md`, `HARDWARE.md` and `DEVELOPMENT_LADDER.md`.

These steps end where rung 1 of the development ladder begins. The ladder's shape sets what has to
be proven here: rung 1 runs Silica as a guest of the firmware using the firmware's network driver
as its console, and rung 2 leaves the firmware with nothing but a hand-written transmit-only
Realtek routine (the bootstrap console) to see by, while it brings up its own page tables and
fault handler. Everything rung 2 depends on is rehearsed here in hand-written assembly first.

Unlike those two documents, this one is written from the real state on 2026-11-21: **Ubuntu is still
installed and the box still answers as `nix.local`.** The wipe is step 7 here, and the order matters,
because most of the earlier steps become hard or impossible once Linux is gone.

Who does what: steps marked **(Lee)** need root on the box, physical access, or are destructive.
Nothing here has been done yet.

Facts checked on 2026-11-21 that drive this list:

- `/Volumes/2T/silica/binaries` has **no compiler that runs on the Mac and emits x86-64**. Every
  `linux-x86_64` binary there, including the one behind the `silica-compiler-linux_x86_64` link, is
  a Linux executable. Today x86-64 code can only be emitted on the HP itself.
- `~/silica` on the HP is **not a git checkout** (it is a copied tree), and the home directory holds
  other things: `project-bees`, `build`, `opt`, and several scripts and logs. 24 GB of the root
  volume is in use.
- The Mac has Apple clang (which assembles x86-64 COFF objects; tested), Homebrew `llvm` with
  `llvm-mc` and `llvm-objcopy`, and macOS's built-in TFTP server. It has **no linker that can produce
  an EFI image** (`lld-link` / `ld.lld` absent) and no `dnsmasq`.
- Mac and HP are on the same network, `192.168.88.0/24`, gateway `192.168.88.1`; the HP was at
  `192.168.88.211`.
- Secure Boot is enabled. Wake-on-LAN is disabled in the firmware's wake table.
- **There is no serial port and none will be added** (Lee, 2026-11-21: not realistic, since the
  image has to be loaded over the network anyway). The Ethernet cable is the only channel for
  loading, output and input.

---

## 1. Gather equipment

- [ ] **(Lee)** Borrow a USB keyboard and any HDMI or DisplayPort screen (a TV works) for one
      session of firmware settings (step 5).
- [ ] **(Lee)** Make an Ubuntu installer / live USB stick and keep it. It is the way back if the
      experiment needs Linux on the box again, and the only way to inspect the box later.
- [ ] **(Lee)** Optional: a switched or smart outlet, as the reset of last resort for a hung image.
- [ ] **(Lee)** Optional: open the case once and note: is a speaker fitted, which M.2 and SATA
      positions exist, the power-supply label. (The serial-port header no longer matters.) Record in `HARDWARE.md`
      section 11.

## 2. Save everything worth keeping from the HP

- [ ] **(Lee)** Copy the home directory to the Mac or another disk: `~/silica` (not in git on that
      machine), `~/project-bees`, `~/build`, `~/opt`, the loose scripts and logs.
- [ ] Confirm the Mac tree already has every `linux_x86_64` golden and published binary that exists
      on the HP, so the last Linux x86-64 fixed point is reproducible from the Mac's copy.
- [ ] **(Lee)** Capture the root-only hardware details into `hardware_raw/`:
      ```
      sudo dmidecode > dmidecode.txt
      sudo acpidump > acpi.dat            # package acpica-tools
      sudo dmesg > dmesg.txt
      sudo cat /proc/iomem /proc/ioports > maps.txt
      sudo lspci -vvv > lspci_vvv.txt
      sudo efibootmgr -v > efibootmgr.txt
      ```
      Then fold the answers into `HARDWARE.md` (memory modules, real memory map, MADT / HPET / MCFG
      contents, MSI-X layouts).
- [ ] **(Lee)** Capture what a Silica Realtek driver will need, since the network is the only
      console once firmware services are given up:
      ```
      sudo dmesg | grep -i r8169 > r8169_dmesg.txt     # exact chip variant (the "XID" line)
      sudo ethtool -d enp5s0f0 > r8169_registers.txt   # register dump of the working chip
      sudo ethtool enp5s0f0 > r8169_link.txt           # negotiated link settings
      ```
- [ ] **(Lee)** Optional but cheap: `sudo modprobe sp5100_tco` and `sudo modprobe pcspkr`, to learn
      whether the chipset watchdog and the speaker actually work on this unit before depending on
      them.

## 3. Decide what replaces the Linux x86-64 build host

Once the HP is wiped there is no native Linux x86-64 machine.

- [ ] Decide: another x86-64 Linux machine takes over `linux_x86_64`, or that platform goes
      unbuilt and untested for the duration.
- [ ] Remove or repoint the `linux_x86_64` row in `/Volumes/2T/silica/.silica_build_hosts`, so the
      multi-platform scripts stop trying to reach `nix`.

## 4. Get an x86-64 toolchain working on the Mac

Do this **while the HP still runs Linux**, so the Mac's output can be checked against the native
compiler.

- [ ] Build a compiler that runs on the Mac and emits x86-64 (`make TARGET=linux_x86_64` in
      `compiler/src`, the route the Linux port originally used), and install it as a cross
      compiler. None exists in `binaries/` today.
- [ ] Verify it: for a set of units, the assembly it emits on the Mac must equal what the native
      compiler on the HP emits, byte for byte.
Decided 2026-11-22: the cross compiler emits for the EFI object format directly (COFF objects,
linked by `lld-link`), not ELF converted afterwards.

- [ ] Install Homebrew `lld` for `lld-link`. Apple clang already assembles x86-64 COFF objects.
- [ ] List every line in the emitter's output that the COFF assembler rejects. Known so far: the
      ELF-style section lines (`.section .data.rel.ro,"aw",@progbits`; `.note.GNU-stack`). Find
      the rest by fixing those in a copy of a golden file and assembling again until it passes.
      That list is the emitter change for `x86_64_uefi_raw`.
- [ ] Prove the pipeline end to end with a hand-written assembly EFI application that does nothing
      but return: clang to COFF, `lld-link` to an EFI application. It should be accepted as a
      valid EFI image by the firmware in step 6.
- [ ] Prove load-time address fix-ups work: a second hand-written image with a table of absolute
      addresses in its data, which reports whether they were corrected for where the firmware
      loaded it.

## 5. Set the firmware up (one session with keyboard and screen)

Before the session, so it only has to happen once:

- [ ] **Decide the Secure Boot route** (`SECURITY_IMAGE_AUTHENTICATION.md`): off for good, or
      Lee's own key now or later. If an own key is ever wanted, prepare it on the Mac first and
      enrol it in this same session, and check in the session whether this firmware allows the
      keys to be replaced. Deciding afterwards means borrowing the keyboard and screen again and
      adding a signing step to an image pipeline built without one.

**(Lee)**, in firmware setup (F10 at power-on):

- [ ] Turn **Secure Boot off** (or, per the decision above, replace its keys). The firmware asks
      for a displayed code to be typed back.
- [ ] Make sure the network (PXE) boot option is enabled and comes before anything that could hang
      waiting; leave the IPv4 entry active. Note whether HTTP boot is offered.
- [ ] Set **power-on after AC power loss** to "power on" if a switched outlet will be used.
- [ ] Enable Wake-on-LAN from power-off if it is to be used (it is disabled now).
- [ ] Check there is no setting that stops the boot for a missing keyboard or display.
- [ ] Do not update the firmware. Record the version shown (expected R06 02.20.00) and write down
      every setting changed, in `HARDWARE.md` section 2.
- [ ] Unplug the keyboard and screen and confirm the machine still boots Ubuntu and answers ssh
      with nothing attached.

## 6. Build and rehearse the load / run / reset loop, with Ubuntu as the safety net

- [ ] Choose how the HP learns where its boot file is: have the router at `192.168.88.1` hand out
      the boot server and file name for the HP's MAC (`e8:d8:d1:c5:92:fd`), or run a boot-request
      responder on the Mac (for example `dnsmasq` in proxy mode). Serve the file with the Mac's
      TFTP server.
- [ ] Write the Mac-side runner, the counterpart of `run_on_board.py` for the ESP32: put the image
      where the server serves it, wait for the fetch, collect console packets, report the exit
      marker.
- [ ] Build the runner as the engine for the ladder's gates (`DEVELOPMENT_LADDER.md`, "Gates"): a
      test is an image, an expected-output file, a time limit and optional Mac-side actions; a
      gate is a list of tests and exits pass or fail; gate N runs the earlier gates first.
- [ ] **Gate 0**: the rehearsal images below, each with its expected report, run as the first
      gate. Passing gate 0 is what "pre-development is done" means for the loop.
- [ ] Rehearse with Linux still installed: **(Lee)** `sudo efibootmgr --bootnext 0001` then
      reboot. The HP network-boots once, runs the test image, and when the image resets the box
      returns to Ubuntu.
- [ ] Test images for the rehearsal, hand-written assembly, no Silica yet, in the spirit of the
      ESP32 `asm_00`..`asm_07` apps:
      1. Fetched and resets (seen in the TFTP log; the box comes back to Ubuntu).
      2. Sends a "hello" packet to the Mac through the firmware's network driver.
      3. Sends the UEFI memory map and the addresses of the ACPI and SMBIOS tables.
      4. Reports whether the firmware offers a framebuffer with no display attached.
      5. Hangs on purpose with the firmware watchdog armed, to prove automatic recovery.
      6. Calls `ExitBootServices`, then resets through the runtime reset service.
      7. **After `ExitBootServices`, sends a packet to the Mac with its own minimal Realtek
         code**: reset the chip, one transmit ring, polled, transmit only, no interrupts. With no
         serial port this is the only way any later image can be seen, so it is a required part
         of the rehearsal, not an extra.
      8. Writes a message to a fixed RAM buffer and resets; the next boot looks for it and sends
         it through the firmware's network driver. This shows whether RAM survives a reset on
         this machine, which decides whether a RAM log can back up the network console.
      9. Optional: beeps the speaker; arms the chipset watchdog.

      Rehearsals for ladder rung 2 (leaving the firmware), all reporting through image 7's routine:

      10. Builds its own page tables after `ExitBootServices`, switches to them, then transmits.
      11. Installs its own interrupt table, faults on purpose, and the handler reports the fault
          address. This is the shape of the status-70 fault report.
      12. Touches an unmapped page; the page-fault handler maps a page there and the code carries
          on. This is the mechanism growable actor stacks rest on.
      13. Switches on the extended processor state and executes an AVX-encoded float16
          instruction (`vcvtph2ps`), before and after leaving the firmware. The emitter uses
          these, so start-up must enable them.
      14. Tells a crash-reset from a deliberate reset across a reboot (a boot counter), so a
          silent triple fault can be recognised.
      14a. Calibrates the processor's counter against a known time source and prints at fixed
          intervals; the Mac checks the spacing. Rung 2 now includes the polled clock.

      Rehearsals for the ladder's interrupt rungs (4 and 8), optional before rung 1 but cheap
      once image 11 exists:

      17. Switches on the processor's local interrupt controller and starts its timer. The
          handler only sets a pending flag; the main loop checks the flag and halts in one step,
          counts wake-ups and reports the count.
      18. Sets up the Ethernet chip's message-signalled interrupt. The handler masks the vector
          in the device's interrupt table and sets a pending flag; the main loop reports it,
          clears the chip's status and unmasks. Shows that generic masking works on this chip.

      Rehearsals for ladder rung 1 (firmware guest):

      15. Asks the firmware for a large block of memory and reports what it was given.
      16. Calls the firmware from code using Silica's calling convention through the call shim,
          many times, and returns intact.
- [ ] Turn image 2's send routine and image 7's transmit routine into two small reusable
      pieces with the same interface, "write these bytes to the console": the **firmware
      console** for rung 1 and the **bootstrap console** for rungs 2 to 8.
- [ ] Measure the loop time from "runner started" to "output received", and how long a reset takes.
- [ ] Decide the hang recovery for images that have given up firmware services, based on what
      image 9 and step 2's module tests showed.
- [ ] Decide how a fault inside the transmit path itself gets reported, based on image 8 (RAM log
      replayed on the next boot, or a status code in a UEFI variable).

## 7. Wipe Linux

Only after steps 2 to 6 are done.

- [ ] **(Lee)** Erase the NVMe SSD.
- [ ] **(Lee)** Remove the stale Ubuntu boot entry (`Boot0005`) if the firmware keeps it.
- [ ] Power on with nothing attached but power and Ethernet, and confirm the box network-boots
      test image 2 on its own, with no key press.
- [ ] Confirm reset and power-cycle both lead back to a network boot.

## 8. Settle the design questions that block rungs 1 and 2

Paper only.

- [x] Emit target: decided 2026-11-22. General target `x86_64_uefi_raw` with the board pack
      `board/hp_elitedesk_705_g5/`, begun as a copy of `emitter/linux_x86_64`.
- [x] How the EFI image is produced: decided 2026-11-22. The emitter writes COFF-clean assembly,
      clang assembles it, `lld-link` links the EFI application.
- [ ] The calling-convention boundary: Silica code uses the SysV convention, firmware calls use the
      Microsoft x64 one (rehearsed by image 16).
- [x] C library dependence: decided 2026-11-22. The copied emitter's C library calls are all
      rewritten up front to the raw runtime's own entry points; no stand-in C library.
- [ ] The call-site inventory: every place in `emitter/linux_x86_64` that writes out a C library
      call, by function and by emitter file, and what each call is for. (Checked so far only
      from one binary's import list; the reasons for calls such as `dladdr` and `sysinfo` have
      not been traced.) Re-check against what the first app uses.
- [x] Level and origin of the contract: decided 2026-11-22. Low level (the embedded assembly
      runtime in the copied emitter is kept, and each C library call in it becomes a primitive),
      and designed fresh for this target, not taken from the ESP32-S3 runtime.
- [ ] The runtime contract, on paper: one low-level primitive per need found in the inventory,
      with arguments and results; which needs are refused at compile time (files, standard
      input, environment variables); which rung fills in each primitive.
- [ ] Read the embedded actor runtime's thread structure (carrier threads, the timer thread,
      `dangerous` actor threads; what each waits on) before fixing the thread primitives. Seen
      so far only by searching for the calls.
- [ ] Multi-core is designed in from the beginning (decided 2026-11-22; rules in
      `DEVELOPMENT_LADDER.md`). Settle before rung 1: the per-core block and the register that
      reaches it; that the contract includes "which core", "how many cores" and "place this
      thread on a core"; start-up split into "first core" and "every core".
- [ ] Memory in two modes with one allocator above them: a block borrowed from the firmware
      (rung 1), and every usable range of the firmware memory map (rung 2 onward).
- [ ] The console, network only, in its three forms (firmware console, bootstrap console, Silica
      UDP console): one interface the runtime writes to, so rung 1's programs run unchanged on
      rung 2.
- [ ] The console packet format and the Mac-side listener, including how the exit marker and the
      fault and abort reports (status 70 and 71) appear in it.
- [ ] Which pieces of `ESP32-S3_raw/board/runtime` carry over in shape. Its stack growth does not:
      it uses compiler-emitted limit checks over chained chunks, and this target uses the
      fault-driven design of the specification.

These can wait until the rung that needs them, but are listed so they are not forgotten (the table
at the end of `DEVELOPMENT_LADDER.md` says which rung each blocks): the restricted "runtime Silica"
for a scheduler written in Silica (rung 3); device discovery at run time and port I/O in the
language (rung 5); fixed-address memory for devices (rung 6). The interrupt model is decided
(see `DEVELOPMENT_LADDER.md`): the scheduler never masks, handlers set a pending flag and mask
their vector generically, and the scheduler enqueues.
- [ ] How this target relates to the trials (the ESP32 port started with board apps, not trials).

## Done when

- A hand-written EFI image, built on the Mac alone, loads over the network onto a machine with no
  OS, no keyboard and no display, prints to the Mac, and resets, repeatedly and unattended.
- A hand-written image prints to the Mac over Ethernet **after** giving up firmware services,
  using its own Realtek code. There is no serial port to fall back on.
- Hand-written images have run on their own page tables, reported a deliberate fault, and
  recovered from a page fault by mapping a page (images 10 to 12).
- The firmware console and the bootstrap console exist as reusable routines with one interface.
- The design questions that block rungs 1 and 2 are answered.
- The Mac can emit and link x86-64 code with no help from the HP.
- Everything from the old Linux install that matters is saved, and the hardware reference has no
  root-only gaps left.
