# Who can replace the code the machine runs

Security note for the OS-free Silica port on the HP EliteDesk 705 G5 SFF. Raised by Lee on
2026-11-21. **Nothing here is decided**; this records the question and the options.
Companions: `IDEATION_2026-11-21.md`, `HARDWARE.md`, `PRE_DEVELOPMENT_STEPS.md`.

## The question

If Secure Boot is turned off so the firmware will accept an unsigned Silica image from the network,
what stops just anybody from replacing the code the machine runs?

## The answer as things are planned

Nothing does. With Secure Boot off and network boot on, the firmware runs whatever the first boot
server to answer hands it. Network boot has no authentication of its own: neither the boot request
nor the file transfer checks who is answering or what the file is.

So anyone who can do both of these chooses the code:

1. answer the machine's boot request on that network, and
2. get the machine to reset (or wait for it to).

Code that is already running cannot be swapped from outside unless the image itself accepts
commands. The exposure is at the next boot.

This is the same position as the ESP32-S3 on the desk: anyone holding the USB cable can reflash it.

## Ways in

| Route | What it takes |
| --- | --- |
| Rogue boot server | Any device on the same network that answers the boot request before, or instead of, the Mac |
| Triggering a reset | A reset command the image accepts without authentication; a power cut; the power button |
| Physical access | A USB stick with a boot image; changing firmware settings |
| Remote management | The Realtek Ethernet chip's management (DASH) function, a path to the box that is separate from anything Silica does. A `DashControl` firmware setting exists |
| The image's own network surface | Any command or input the running image accepts over the network |

## Options

### During development: make the Mac the only possible boot server

- **Isolate the link.** A direct cable between the Mac and the HP, or a network segment with
  nothing else on it.
- **Set a firmware setup password** and disable USB boot, so physical access does not trivially
  change the boot path.
- **Turn off the Realtek chip's remote management (DASH).**

This does not authenticate anything; it removes the other parties.

### The proper fix: Secure Boot on, with Lee's own key

Secure Boot does not have to mean the factory (Microsoft) keys. The factory keys are replaced with
a key generated on the Mac, and every Silica image is signed with it as a build step. The firmware
then refuses any image not signed by that key, whether it arrives over the network or from disk.

- It needs the same one-time keyboard-and-screen session as turning Secure Boot off, so it could be
  done in that session instead.
- **Not verified:** that this HP firmware (R06 02.20.00) allows the keys to be replaced. Most
  business desktops do.
- The signing key on the Mac becomes the thing to protect.
- The image has to be a well-formed EFI (PE/COFF) file for the signature to be accepted, which
  depends on the image pipeline in step 4 of the pre-development steps.

A possible order: Secure Boot off for the first hand-written test images, then own-key Secure Boot
once the image pipeline produces well-formed files.

### A deployed machine

Network loading is a development convenience. A machine in service would have:

- the image on the NVMe SSD,
- network boot disabled,
- own-key Secure Boot on,
- the firmware setup password set.

## What Secure Boot does not cover

- **The running image's own network surface.** Secure Boot checks the image only at start. If the
  image accepts a "reset" or "load" command over the network, the image itself has to authenticate
  it. An unauthenticated reset command plus an open boot network is the whole attack.
- **The console.** UDP console output is readable by anyone on the network.
- **Code underneath Silica.** The firmware (including its system-management code) and the AMD
  security processor keep running regardless of what Silica does.
- **Detection.** The machine has a TPM 2.0, which could record what was booted so tampering can be
  noticed afterwards. Not explored.

## Open

- Off first and own key later, or own key from the start?
- Does this firmware allow replacing the Secure Boot keys? (Check in the keyboard-and-screen
  session.)
- Direct cable or an isolated network segment for development?
- Will the Silica image ever accept commands over the network, and if so how are they
  authenticated?
- Where the signing key is kept.

## Effect on other documents

`PRE_DEVELOPMENT_STEPS.md` step 5 now begins with "decide the Secure Boot route" (added
2026-11-22), so that the one keyboard-and-screen session covers whichever route is chosen. If the
own-key route is chosen, that step becomes "replace the Secure Boot keys", and step 4 gains a
signing step.
