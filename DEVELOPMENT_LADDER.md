# Development ladder

The order of work for running Silica raw on the HP EliteDesk 705 G5 SFF, from the first Silica image
to the first app. Written 2026-11-22; interrupt rungs added, the order revised and gates added the same day (ten
rungs). Nothing here has been built.

Companions: `IDEATION_2026-11-21.md` (decisions and discussion), `HARDWARE.md`,
`PRE_DEVELOPMENT_STEPS.md` (what must be true before rung 1), `SECURITY_IMAGE_AUTHENTICATION.md`.

## The first app

The Mac sends a message over TCP; the machine replies with the number 42.
Seen from the Mac as: `echo hi | nc <hp-address> 4242` prints `42`.

## Fixed points this ladder is built on (Lee's decisions)

- The image is a UEFI application, loaded by network boot from the Mac. No bootloader, no OS on the
  machine, no keyboard, no display, no serial port. Ethernet is the only channel.
- Work is done on the hardware, not in an emulator.
- The TCP stack, its supporting network code and threading are written in Silica.
- The run claims all the usable RAM; one app at a time.
- Actor stacks are growable as the language specifies (specification section 15.1.2.2): a contiguous
  reservation at spawn, memory committed on fault, the five release algorithms, a stack policy on
  every spawn.

- Emit target (decided 2026-11-22): a general target, **`x86_64_uefi_raw`**, for any 64-bit UEFI
  PC, with a board pack under it for this machine, `board/hp_elitedesk_705_g5/`. The pack holds
  what is specific to this HP: the PCI ownership table, the Realtek device description, the
  bootstrap console. The target directory **starts as a copy of `emitter/linux_x86_64`** and
  diverges; the Linux target is left untouched. Fixes to x86-64 instruction selection have to be
  made in both from then on.
- Image production (decided 2026-11-22): the cross compiler **emits for the EFI object format
  directly**. The `x86_64_uefi_raw` emitter writes COFF-clean section directives; clang on the Mac
  assembles to COFF; `lld-link` links an EFI application. The linker writes the load-time
  address fix-up table and the firmware applies it, so the start-up stub does no relocation.
  Measured 2026-11-22: a `linux_x86_64` golden assembles to ELF on the Mac unchanged and fails for
  COFF on the ELF-style `.section` lines (first error seen; others may follow).
- C library dependence and the runtime contract (decided 2026-11-22):
  - Every C library call the copied emitter writes out is replaced up front by a call to the raw
    runtime's own entry points. No C-named function remains; no stand-in C library is built.
  - **The contract is low level.** The copied emitter keeps its embedded assembly runtime (on
    the Linux target the runtime is assembly text inside the emitter's source files, about 18,600
    lines for the actor parts alone). Each C library call in it is replaced by a primitive:
    allocate, reserve and commit pages, start a thread, register a fault handler, wait, read the
    clock, write to the console. The actor runtime's own logic stays as it is.
  - **The contract is designed fresh for this target.** It does not reuse the ESP32-S3 runtime's
    entry points or names.
  - What stands behind each primitive is filled in rung by rung; a primitive whose rung has not
    arrived ends the run with a clear "not yet on this target" abort.
  - **"Threading in Silica" is read as the thread layer under the actor runtime** (my reading,
    to be confirmed by Lee): the embedded actor runtime creates carrier threads, a timer thread
    and a thread for each `dangerous` actor, and the primitive that provides those threads on
    this machine is what is written in Silica.
- **More than one core is designed in from the beginning** (decided 2026-11-22). The first app
  runs on one core, and the other cores are not started on this ladder, but nothing may be built
  that assumes one core. See "Multi-core rules" below.
- Interrupt model (decided 2026-11-22):
  - **The scheduler never masks interrupts.** Handlers touch nothing the scheduler owns. A handler
    writes only to its own pending flag, and the scheduler collects pending flags at dispatch
    boundaries. (With the low-level contract, "the scheduler" here is two things: the embedded
    actor runtime kept from the Linux copy, and the Silica thread layer beneath it. Where exactly
    the flags are collected is an open question below.) The one race, checking for pending work and then halting, is closed by making
    those a single step.
  - **A handler sets a pending flag; the scheduler enqueues.** At the next dispatch boundary the
    scheduler turns each flag into one message for the device actor that owns that interrupt.
    Repeated interrupts collapse into one pending wake-up. The timer has no device actor: its
    handler only tells the scheduler that time has passed.
  - **A handler acknowledges generically**, by masking the interrupt's vector in the device's
    standard interrupt table. The Silica driver actor does the chip-specific work and unmasks.
    One assembly handler serves every PCI device.
  - **Two assembly paths run with interrupts masked:** the page-fault path (the processor masks
    on entry) and the context switch (masked for its few instructions). "Never mask" therefore
    means the Silica thread layer needs no masked-section construct. Runtime code called
    from the fault path, such as the page allocator, runs masked without knowing it.
  - Handlers do not nest, and the page-fault handler runs on its own dedicated stack (required:
    the faulting actor's stack is the thing that is missing).
  - A flag-only handler uses no floating-point or vector registers, so it does not save the
    extended processor state.
  - **The message a device actor receives is a bare `:interrupt`.** The driver reads the chip to
    learn what happened.
  - **Ownership is checked twice.** The board pack's table, keyed by PCI vendor and device ID,
    says which chips may be claimed and by which device actor; the device actor claims the actual
    device's interrupt when it sets the device up, and the runtime refuses a second claim.
  - Unchanged from the device actor specification (section 9): the handler is not a Silica
    behaviour, and each interrupt belongs to exactly one device actor, named by the board pack.

## Why the ladder has this shape

- The x86-64 emitter grows stacks by page fault. So actors cannot exist until the image has its own
  page tables and fault handler, and that is only safe after the image leaves the firmware.
  (The ESP32-S3 grows stacks by compiler-emitted limit checks over chained heap chunks; the x86-64
  emitter has none of that, and adding it for a temporary stage would be thrown away.)
- `main` is not an actor and its stack never grows, so Silica code without actors can run while the
  firmware is still in charge, using the firmware's own network driver as the console.
- A Silica network driver is a device actor, so it cannot exist before actors do. Between leaving
  the firmware and the Silica network stack working, the console is a small hand-written
  transmit-only Realtek routine (the **bootstrap console**), proven in pre-development.

- Interrupts arrive in two steps, each placed where there is something to deliver them to and
  something to test them with. The timer interrupt comes on rung 4, once a scheduler
  exists to wake. The Ethernet chip's interrupt comes after ping works (rung 8), so the driver's
  rings are proven by polling first and ping supplies a steady test load. Rungs 2 and 3 stay free
  of interrupts: rung 2 has nothing to deliver them to, and rung 3 is where fault-driven stack
  growth is debugged.

- The order is chosen so that nothing is built twice:
  - **The clock comes before actors.** The embedded actor runtime already waits on time (it
    sleeps, waits with timeouts, and runs its own timer thread), so the polled clock is built on
    rung 2 and rung 3 gets the real thing, not a stand-in.
  - **The Ethernet driver is written once.** From rung 6 it is "when woken, drain everything
    that arrived". On rung 6 the wake-up is a periodic timer message; on rung 8 only the source
    of the wake-up changes, to the chip's interrupt.
  - Two things are thrown away on purpose: firmware-guest mode (rung 1) and the bootstrap
    console (retired on rung 9). Each is the only way to see what is happening at its stage.

## Multi-core rules

These apply from rung 1, though only one core runs until after the first app.

- **Per-core data is separate from the start.** Each core has its own block (current thread,
  run queue, pending interrupt flags, fault stack, interrupt stack, clock calibration), reached
  through a per-core register, never through a global variable.
- **Every primitive that touches shared runtime data is written to be safe with other cores
  running**: the heap, the physical-page allocator, the page tables, the thread layer's queues.
  Locks or per-core structures, not "there is only one core".
- **The contract carries core identity.** The primitives include "which core am I", "how many
  cores", and placing a thread on a core, because the embedded runtime already asks for these
  (it pins threads to cores and reads the core count). On this ladder they answer for one core.
- **Start-up is written as "first core" and "every core".** Switching on processor state, loading
  the interrupt table, the page tables and the per-core block is one routine every core will
  run; only the first core also talks to the firmware and sets up memory.
- **Interrupt flags and ownership are per core.** A pending flag belongs to the core its
  interrupt is delivered to; a device's interrupt names a target core.
- **Page-table changes assume other cores hold stale mappings.** Releasing stack pages must be
  written so a later "tell the other cores" step can be added in one place.
- **Checks are run so one-core assumptions show.** Where a check can be written to be
  core-count independent, it is.

## Gates

Every rung ends in a gate: one command on the Mac that loads that rung's test images onto the
machine, checks what comes back, and exits pass or fail. A rung is finished when its gate passes.

- **Form.** `gate <rung>`. It network-boots each of the rung's test images in turn, collects the
  console output, compares it with expected output kept beside the test, and performs any checks
  made from the Mac side (ping, a TCP connection). The exit status is the result.
- **Cumulative.** Gate N runs gates 0 to N-1 first, so a later rung cannot quietly break an
  earlier one. A gate can also be run alone while working on its rung.
- **Engine.** The Mac-side runner built in pre-development step 6. It is designed for gates from
  the start: a test is an image, an expected-output file, a time limit and optional Mac-side
  actions.
- **Same pattern as the ESP32-S3 port**: board apps with expected output and a runner script.
- **A hung image fails its gate by time limit.** Getting the machine back then depends on the
  chipset watchdog (present, untested) or the power button, so from rung 2 on gates are not
  fully unattended until that is settled.
- **No privileged capture on the Mac.** Where a gate must see a raw frame, the test frame is
  built as a UDP broadcast, which an ordinary program can receive.
- **Timing checks have stated tolerances**, because they compare against the Mac's clock across
  a network.
- **A gate shows that the listed cases work, not that nothing is broken.**
- **Run time is unknown** until the loop is measured: each image costs a full power-on and
  network boot. If the cumulative run becomes too slow, several test programs are batched into
  one image, accepting that one failure can hide the rest of that batch.

| Gate | Passes when |
| --- | --- |
| 0 | The pre-development rehearsal images each report as expected |
| 1 | The language ladder passes as a firmware guest; the image has no undefined or C-named symbols |
| 2 | The same programs pass in raw mode; a deliberate fault is reported; a timed printer keeps time |
| 3 | Actors, stack growth and release, containment and timeouts pass |
| 4 | Gate 3's tests pass with interrupts on; an idle image halts |
| 5 | The PCI list and the MAC address match the hardware reference |
| 6 | A frame from Silica reaches the Mac and a frame from the Mac is echoed back |
| 7 | The Mac's ping gets every reply |
| 8 | Ping gets every reply, including a burst, woken only by the chip's interrupt |
| 9 | Earlier gates pass with the console carried over UDP |
| 10 | The Mac sends a message over TCP and gets `42`, repeatedly |

## The three consoles

| Console | Available | Written in | Used on rungs |
| --- | --- | --- | --- |
| Firmware console: packets sent through the firmware's network driver | Only while firmware boot services are alive | Assembly shim calling the firmware | 1 |
| Bootstrap console: transmit-only, polled Realtek routine | After leaving the firmware | Assembly (from pre-development image 7) | 2 to 8 |
| Silica console: UDP log over the Silica network stack | Once rung 9 works | Silica | 9 onward |

A RAM log replayed on the next boot may back these up, if pre-development shows RAM survives a reset.

The bootstrap console stays polled and never depends on interrupts, so it keeps working inside
interrupt handlers and fault reports.

---

## Rung 1. Language ladder as a firmware guest

**Goal.** Silica code compiled for this machine runs and is seen on the Mac. `main` only, no actors.

**Built.**
- The emit target `x86_64_uefi_raw`, begun as a copy of `emitter/linux_x86_64`, with the board
  pack `board/hp_elitedesk_705_g5/`.
- **The runtime contract**: the complete list of low-level primitives emitted code may call,
  with each one's arguments and results, one for every need the copied emitter meets today by
  calling the C library. Designed fresh for this target.
- **The emitter rewrite**: every C library call the copied emitter writes out, in ordinary
  emitted code and inside its embedded assembly runtime, replaced by a contract call. The
  embedded runtime's logic is otherwise left alone. Things with no meaning on this target
  (files, standard input, environment variables) are refused at compile time. A check that a
  built image has no undefined or C-named symbols.
- The emitter's section directives changed from ELF style to COFF style (and any other
  ELF-only lines the assembler rejects).
- The image pipeline on the Mac: clang assembles to COFF, `lld-link` links the EFI application.
- Start-up stub: firmware entry, processor state switched on (SSE and AVX; the emitter uses
  AVX-encoded float16 instructions), `main`'s fixed stack.
- The firmware-call shim (firmware and Silica use different calling conventions).
- Memory, guest mode: a block borrowed from the firmware; heap, regions and frames on top.
- Behind the contract for this rung: byte copy and fill, the heap, the language support
  routines (strings, lists, ordering, checked-arithmetic aborts, floats), console and exit.
  Primitives for later rungs (pages, threads, faults, waiting, time) exist and abort.
- Console output and the exit marker over the firmware console; reset at the end.
- The Mac-side runner compares output with the expected output.

**Seen from the Mac.** The equivalents of the ESP32 apps `silica_00_return42` through
`silica_15_list_index_abort` print the expected lines and exit status.

**Done when.** The language ladder passes unattended, run after run.

**Not yet.** Faults are whatever the firmware does with them. No actors, no time, no devices.

**Size.** The rewrite touches many emitter files before the first Silica program can run, but
each change is a call swapped for a primitive, not logic rewritten. Later rungs fill in what is
behind the primitives.

**Gate 1.**
- Runs: the language-ladder programs (the equivalents of `silica_00_return42` through
  `silica_15_list_index_abort`), each as a firmware-guest image.
- Checks: each program's console output and exit status equal its expected file; on the Mac, the
  linked image has no undefined symbols and no C library names; the machine resets and is ready
  for the next image every time.
- Also runs gate 0.

## Rung 2. Leave the firmware

**Goal.** The same `main`-only programs run with the firmware's boot services gone.

**Built.**
- Read the firmware memory map, call `ExitBootServices`.
- The runtime's own page tables, built before any firmware memory is reused.
- Memory, raw mode: every usable range from the memory map handed to the allocator (several
  separate ranges); the same heap, region and frame code as rung 1 on top.
- Interrupt table and fault entry stubs; the fatal fault report (status 70) and runtime abort
  (status 71).
- The bootstrap console wired in as the runtime's output.
- A way to tell a crash-reset from a deliberate reset (for example a boot counter).
- **The polled clock**: the processor's invariant counter calibrated to real time; behind the
  time primitives, "what time is it" and "sleep this long" (a busy wait for now). No interrupt
  is involved. Moved here from rung 4 so that rung 3 has a real clock.
- The per-core block for the first core, and start-up split into "first core" and "every core".
- Reset through the firmware's runtime service.

**Seen from the Mac.** The rung 1 programs produce the same output over the bootstrap console. A
program that faults on purpose (the `silica_08_stack_guard` equivalent) prints the fault banner and
status 70 instead of silently rebooting.

**Done when.** Rung 1's whole set passes in raw mode, deliberate faults are reported, and a
program that prints at fixed intervals keeps time with the Mac's clock.

**Risk.** This is the hardest rung: between `ExitBootServices` and the first bootstrap-console
packet nothing can be seen. Pre-development images 7, 10, 11 and 12 exist to take that risk out
before any Silica is involved.

**Gate 2.**
- Runs: every gate 1 program built in raw mode; a program that faults on purpose (the
  `silica_08_stack_guard` equivalent); a program that aborts on purpose (the
  `silica_15_list_index_abort` equivalent); a program that prints a line at fixed intervals; a
  program that allocates and frees until it has touched memory in every usable range.
- Checks: output equals gate 1's expected files, now over the bootstrap console; the fault prints
  the fault banner with status 70 and the abort status 71; the boot counter shows deliberate
  resets, not crash resets; the interval lines arrive with the stated spacing within tolerance;
  the allocator reports the amount of memory it claimed, close to the usable total in the
  hardware reference.

## Rung 3. Actors with growable stacks

**Goal.** Actors exchange messages; stacks grow and release as specified. One core runs; the
design is for many.

**Built.**
- A physical-page allocator in 4 KB pages, safe for use from several cores.
- A dedicated page-fault stack per core.
- Context switch (assembly), including saving the extended processor state.
- **The thread layer, written in Silica**, behind the "start a thread" primitive and its
  companions (per-thread storage, locks, wait and wake, place on a core). The embedded actor
  runtime creates several threads (carriers, a timer thread, one per `dangerous` actor), so the
  layer runs several threads per core. Its queues are per core from the start. It needs the
  restricted "runtime Silica" settled first: code at this level cannot itself use actors or
  receive.
- The page primitives behind the embedded runtime's stack code: reserve at spawn, commit on
  page fault in doubling chunks up to 1 MB, release at the message boundary. The five release
  algorithms, the stack policy and the `:stack_reserve_failed` / `:stack_exhausted` reasons are
  already in the embedded runtime and should come across unchanged.
- The actor runtime itself (scheduler, mailboxes, spawn / cast / call / receive, registries,
  failure reports, supervisors) is the embedded assembly kept from the Linux copy, now running
  on the primitives. Its sleeps and timed waits use the polled clock from rung 2.
- **Written for interrupts before any arrive.** The thread layer has no masked sections. It is
  built with the per-core pending-flag collection point and the idle path that checks for
  pending work and halts in one step, even though no flag is set until rung 4. Until then the
  idle path spins.

**Seen from the Mac.** Two actors ping-pong and print. A deep recursion inside an actor grows its
stack and completes. An actor that crashes is contained and reported; the others carry on. A
timed receive fires when expected, and `:release_when_idle` releases after its period.

**Done when.** The actor, timeout and stack-policy checks pass.

**Not yet.** No interrupts. Faults (including the page faults that grow stacks) are the only
things that enter the interrupt table. An idle image spins.

**Gate 3.**
- Runs: two actors exchanging messages; a deep recursion inside an actor; an actor that crashes
  beside ones that carry on; a supervised actor that is restarted; a timed receive; one test per
  release algorithm; an actor with a lowered reservation that recurses past it; a spawn count
  large enough to show that idle actors hold no stack.
- Checks: expected output for each; the crash is reported and the other actors' output is
  complete; the timed receive fires within tolerance; memory use reported by the image
  (`get_actor_memory_usage`) rises during growth and falls as each release algorithm specifies;
  the lowered reservation ends that actor with `:stack_exhausted` and no other.

## Rung 4. The timer interrupt and idle

**Goal.** An idle core sleeps until there is work; waiting no longer burns the processor.

**Built.**
- The processor's local interrupt controller switched on (found at its standard address or through
  the processor's own register; no ACPI needed) and its timer calibrated against the rung 2 clock.
  Part of the "every core" start-up.
- The first interrupt handler: entry and exit stubs that save and restore only the registers the
  handler uses (no extended processor state), on top of the interrupt table from rung 2, running
  on a per-core interrupt stack.
- The timer handler sets its core's pending flag and nothing else; at the next collection point
  the thread layer sees that time has passed and makes threads whose waits are due ready.
- "Sleep this long" and timed waits changed behind the same primitives from busy waiting to
  halting until the timer fires. The primitives' callers do not change.
- Idle: with nothing ready, the core halts and the next interrupt wakes it.
- The context switch and the page-fault path made safe by running masked.

**Seen from the Mac.** The rung 3 checks pass again with interrupts arriving throughout. A long
run with actors mostly idle keeps correct time.

**Done when.** Timeouts are as accurate as on rung 3; the rung 3 actor and stack-growth checks
still pass with the timer interrupt running; an idle image no longer spins.

**Risk.** This is the first asynchronous event in the system. The page-fault path and the context
switch run masked, so neither can be interrupted, but everything else can. The rung 3 checks are
re-run here for that reason.

**Gate 4.**
- Runs: every gate 3 test again with the timer interrupt on; a long run in which actors are
  mostly idle; a stress run that grows stacks and switches between actors continuously while
  the timer fires.
- Checks: gate 3's results are unchanged; a counter kept by the image shows the core halted for
  most of the idle run and the timer interrupt firing at the set rate; timeouts are as accurate
  as on gate 3; the stress run completes with no fault report.

## Rung 5. Device access from Silica

**Goal.** Silica code finds and reads a real device.

**Built.**
- Compiler: device access allowed on this target; register reads and writes at 8, 16 and 32 bits;
  port I/O; the primitive set for anything privileged the driver needs.
- PCI enumeration through the port-based configuration method (no ACPI needed).
- `map_device` checked against device windows discovered at run time.
- A device description for the Realtek chip.

**Seen from the Mac.** Silica prints the PCI device list, and the Ethernet chip's MAC address read
from its registers: `e8:d8:d1:c5:92:fd`.

**Done when.** The printed list matches `HARDWARE.md` section 6 and the MAC matches.

**Gate 5.**
- Runs: a Silica program that walks the PCI bus and prints each device's address, vendor and
  device ID; a program that maps the Ethernet chip and prints its MAC address; a program that
  tries to map an address outside every discovered device window.
- Checks: the list equals section 6 of `HARDWARE.md`; the MAC is `e8:d8:d1:c5:92:fd`; the
  illegal map is refused with the runtime's abort and status 71.

## Rung 6. Silica Ethernet driver

**Goal.** Frames go out and come in under Silica's control.

**Built.**
- Memory the chip can be pointed at: buffers and descriptor rings at fixed physical addresses.
- The driver as a device actor: reset, receive and transmit rings, link check, send and receive.
- **Written once, in its final shape: "when woken, drain everything that arrived, then wait."**
  On this rung the wake-up is a periodic timer message, so the chip's own interrupt stays off
  and the rings and registers are proven with nothing asynchronous from the device. Rung 8
  changes only where the wake-up comes from.

**Seen from the Mac.** A packet capture on the Mac shows a frame sent by Silica; a frame sent by the
Mac is printed by Silica (over the bootstrap console, which still exists beside the driver).

**Done when.** Frames pass both ways repeatedly without loss at low rate.

**Note.** The bootstrap console and the Silica driver both touch the same chip from this rung until
rung 9. How they share it has to be decided (for example, the driver takes over transmission and the
bootstrap routine is called only on a fatal fault).

**Gate 6.**
- Runs: a driver image that transmits a numbered series of frames built as UDP broadcasts, then
  echoes back whatever the Mac sends it.
- Checks: from an ordinary program on the Mac, every numbered frame arrives, in order, and every
  frame the Mac sends is echoed with the same contents; the image reports link up and no
  receive or transmit errors on the bootstrap console.

## Rung 7. ARP and ping

**Goal.** The machine is reachable by address.

**Built.** Ethernet framing, ARP (answer and resolve), IPv4 without fragmentation, checksums,
byte-order helpers, ICMP echo reply. An IP address: static, or the one the firmware obtained when
it network-booted.

**Seen from the Mac.** `ping <hp-address>` gets replies.

**Done when.** A long ping run shows no loss. From here on, ping is the first diagnostic for
everything.

**Gate 7.**
- Runs: an image that brings the stack up to ICMP and stays up.
- Checks: from the Mac, the address resolves (ARP) and `ping` with a fixed count gets every
  reply; a second run with a larger packet size also gets every reply; the image keeps
  answering for a stated length of time.

## Rung 8. Interrupt-driven receive

**Goal.** The Ethernet chip tells the driver when a frame has arrived, instead of being asked.

**Built.**
- Message-signalled interrupt set-up for the chip: read its capability from PCI configuration,
  point its single vector at the processor's local interrupt controller, unmask receive (and link
  change) in the chip. No I/O interrupt controllers and no ACPI are needed.
- The generic device interrupt handler (assembly, not chip-specific): mask the vector in the
  device's interrupt table and set that interrupt's pending flag.
- The scheduler turns the flag into one bare `:interrupt` message to the driver actor at the next
  dispatch boundary.
- Ownership: the board pack's table (PCI vendor and device ID to device actor) and the run-time
  claim made when the driver sets the device up; a second claim is refused.
- The driver actor's wake-up changed from the periodic timer message to `:interrupt`, and one
  step added after draining: read and clear the chip's status, then unmask the vector. The drain
  logic from rung 6 is unchanged.
- Transmit can stay polled; the first app does not need transmit-complete interrupts.

**Seen from the Mac.** `ping <hp-address>` behaves as on rung 7, and a counter printed by the image
shows frames received by interrupt with the periodic wake-up gone. With no traffic the core is halted.

**Done when.** A long ping run shows no loss and no stuck receive (a missed interrupt shows up as
ping stalling until the next frame), including a burst (`ping -f` or many pings at once), and the
earlier checks still pass.

**Why here.** Ping is steady traffic that can run for hours and allows a direct comparison with
the timer-woken behaviour of rung 7. UDP and TCP are then built on the final wake-up model.

**Gate 8.**
- Runs: the gate 7 image with the chip's interrupt on.
- Checks: gate 7's ping checks pass; a burst of pings gets every reply; after a quiet interval
  the first ping is answered (a missed interrupt shows as a stall); counters printed by the
  image show wake-ups from `:interrupt` and none from the periodic timer message, and the core
  halted while there was no traffic; a second device actor's attempt to claim the same
  interrupt is refused.

## Rung 9. UDP and the Silica console

**Goal.** The Silica stack carries the console; the bootstrap routine is retired from normal use.
Built on interrupt-driven receive from rung 8.

**Built.** UDP send and receive; the log console and exit marker over UDP; fault and abort reports
over it where the stack is still usable.

**Seen from the Mac.** The runner collects console output over UDP; earlier rungs' programs still
pass with the new console.

**Done when.** All earlier checks pass using the Silica console. What reports a fault inside the
network stack itself is decided here (bootstrap routine kept for fatal faults only, or RAM log).

**Gate 9.**
- Runs: earlier gates' programs with the console carried over UDP; an image that faults on
  purpose outside the network stack.
- Checks: gates 1 to 8 pass with output arriving over UDP; the exit marker arrives; the fault
  report arrives by the route decided on this rung.

## Rung 10. TCP: reply 42

**Goal.** The first app.

**Built.** TCP: listen, the three-way handshake, sequence and acknowledgement handling, receive
window, retransmission timers (on the timer interrupt from rung 4), close and reset handling, checksum with pseudo-header. A random
initial sequence number (needs a compiler primitive for the processor's random-number instruction).
The app: accept a connection, read the message, write `42`, close.

**Seen from the Mac.** `echo hi | nc <hp-address> 4242` prints `42`.

**Done when.** It answers correctly many times in a row, including after a deliberately dropped
packet, and the machine keeps answering ping throughout.

---

**Gate 10.**
- Runs: the first app.
- Checks: from the Mac, a TCP connection to port 4242 with a message sent gets `42` and a clean
  close; this succeeds a stated number of times in a row; it succeeds with a packet deliberately
  dropped on the Mac side, showing retransmission; several connections one after another all
  succeed; ping is answered throughout.

## Not on this ladder

Not needed for the first app, and deliberately left out: starting the other cores (the design
allows for them from rung 1; switching them on is the natural step after the first app); the I/O interrupt controllers
and interrupts from any device other than the Ethernet chip; transmit-complete interrupts;
pre-emption of a running actor by the timer (scheduling stays cooperative); ACPI parsing; the disk; files; USB;
display; the IOMMU; foreign-function calls; Secure Boot with an own key; running the trial tree on
this target.

## Open design questions that block specific rungs

| Question | Blocks |
| --- | --- |
| The primitive list itself, with arguments and results, from a call-site inventory of the copied emitter | 1 |
| What becomes of the four embedded foreign-function runtime files (foreign calls are not on this ladder, but the files call the C library) | 1 |
| Several threads per core: is the Silica thread layer cooperative or driven by the timer interrupt, and does the embedded runtime's separate timer thread survive or is it replaced by the timer interrupt | 3, 4 |
| The per-core block: its contents, which register reaches it, and how the embedded runtime's per-thread storage maps onto it | 2, 3 |
| Which time calls the embedded actor runtime makes (from the call-site inventory), to confirm the rung 2 clock covers them | 2, 3 |
| Where pending interrupt flags are collected and turned into messages, now that the actor runtime is embedded assembly and the thread layer is Silica | 3, 4, 8 |
| Device actors: the Linux emitter refuses device access, so its embedded runtime may have no `spawn_device` / `cast_device` support; where that comes from | 5, 8 |
| How the runtime reaches the bootstrap console, and how it shares the chip with the Silica driver later | 2, 6, 8 |
| "Runtime Silica": the restricted language the thread layer is written in, and how the primitives reach it | 3 |
| Device discovery at run time and port I/O in the language | 5 |
| Fixed-address memory for devices as a region kind | 6 |
| Console packet format, exit marker and fault reports on the wire | 1, 2, 9 |
| Static address or the firmware's network-boot address | 7 |
