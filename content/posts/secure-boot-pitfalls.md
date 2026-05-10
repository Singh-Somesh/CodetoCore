---
title: "Why Your Secure Boot Probably Isn't Actually Secure"
date: 2025-05-10
draft: false
tags:
  - secure-boot
  - embedded-security
  - hardware-root-of-trust
  - threat-modeling
  - opinion
categories:
  - Security
description: "Most secure boot implementations look fine on paper but fail under realistic threat models. Here are the eight most common ways teams get it wrong — and what fixing them actually requires."
summary: "Most secure boot implementations look fine on paper but fail under realistic threat models. Here are the eight most common ways teams get it wrong — and what fixing them actually requires."
ShowToc: true
TocOpen: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowCodeCopyButtons: true
---

Secure boot is one of those things that everyone in embedded says they
have, but very few actually do correctly. Walk into ten product teams
shipping connected devices and ask "do you have secure boot?" — eight
will say yes. Ask the follow-up — "what does an attacker have to do to
defeat it?" — and the answers fall apart.

This isn't a tutorial. It's a critique. After a decade of building and
reviewing embedded security on Cortex-M and Cortex-A platforms, I've
watched the same mistakes repeat across teams, vendors, and product
categories. They're not exotic. They're not subtle. They're just
overlooked because secure boot is treated as a checkbox feature rather
than a system property.

If you're shipping a product with secure boot, this post is a forcing
function: read each pitfall and ask whether your implementation is
actually safe from it.

## What "secure boot" actually has to do

Before the pitfalls, the bar. A secure boot implementation has to
guarantee one thing:

> The code running after reset is the code the manufacturer authorized,
> and an attacker — even one with physical access — cannot replace it
> with their own code.

That's it. Every component of secure boot exists to make that statement
true. If any link in the chain is breakable, the property doesn't hold,
and you don't have secure boot — you have a feature that looks like one.

The chain typically has three parts:

1. **Root of trust** — an immutable hardware-anchored element that holds
   or hashes the public key used to verify boot images
2. **Boot ROM verification** — silicon-resident code that checks the
   first-stage bootloader against the root key before executing it
3. **Chain of trust** — each subsequent stage cryptographically verifies
   the next stage before transferring control

Every pitfall below breaks one of these three.

## Pitfall 1: Production fuses never get burned

This is the most common failure I see, and it's the most embarrassing.

Most modern microcontrollers ship with secure boot capability that
relies on **one-time-programmable (OTP) fuses** to lock the device into
secure mode. The flow is something like:

1. Generate signing keys
2. Compute the public key hash
3. Write the hash into OTP fuses (irreversible)
4. Burn a "production mode" fuse that disables debug, JTAG, and unsigned
   boot paths

In development, teams skip steps 3 and 4. The device runs in "test
mode" — secure boot logic *appears* to work because images are signed
and the bootloader checks signatures, but **without the fuses set, those
checks can be bypassed**. Test mode typically allows:

- Loading unsigned firmware via JTAG
- Disabling secure boot at runtime
- Reading back firmware regardless of read-protection settings

Many products ship in test mode. Sometimes nobody on the team realized
the fuses needed to be burned. Sometimes the fuses were skipped because
"we want to be able to recover bricked units in the field." Either way,
the entire secure boot story is fiction.

**The fix:** Burn production fuses on every unit before it leaves the
factory. Make this a hard gate in your manufacturing flow. Verify it on
every unit during final test.

**The honest acknowledgment:** Burning fuses is irreversible. If your
firmware is buggy and bricks devices, you can't recover them. This is a
real cost. But "we'd rather have recoverable insecure devices" is a
business decision, not a security strategy. State it explicitly.

## Pitfall 2: Debug interfaces left enabled

Closely related: even with fuses burned, JTAG/SWD/ETM interfaces are
often left active. The reasoning is always the same — "we need it for
field debugging" or "production test needs it."

A live debug interface on a device with secure boot is a contradiction.
With JTAG access, an attacker can:

- Halt the CPU at any point and inject code
- Read memory contents (including keys derived at runtime)
- Modify program flow to skip signature checks
- Set breakpoints inside the verification routine itself

A secure bootloader that can be paused and patched mid-execution is not
secure. It's theater.

**The fix:** Disable debug interfaces in production via OTP fuses. Use
authenticated debug protocols (like ARM Authenticated Debug Access
Port — ADAC) if field debug is a real requirement. Don't use "JTAG
locking" via firmware-controlled GPIOs — those can be defeated by
glitching or physical bypass.

## Pitfall 3: Signing keys live in plaintext somewhere

Secure boot rests on the assumption that the private signing key is, in
fact, private. In practice:

- Keys committed to git history (often "for testing" then never removed)
- Keys stored on a developer's laptop, sometimes years
- Keys in CI/CD environment variables that 30 engineers can read
- Keys in shared password managers
- Keys printed in build logs that get archived to S3

If any of these happen, your root of trust is compromised the moment
someone with access decides to (or is compelled to) leak the key.

**The fix:** Production signing keys must live in a Hardware Security
Module (HSM). The signing operation happens on the HSM; the key never
leaves it. Build pipelines submit images to a signing service that
fronts the HSM, with audit logs and access controls.

This is operationally heavier than just keeping a `signing_key.pem` on
the build server. That weight is the cost of having an actual root of
trust.

## Pitfall 4: The "verify" step doesn't verify what you think

I've reviewed bootloaders that "verified" images by:

- Computing a CRC32 (not a cryptographic hash)
- Computing SHA-256 but never checking the signature, just the hash
- Verifying the signature against a key embedded in the image being
  verified (yes, this happens)
- Verifying signature, then *discarding the result* and continuing
  regardless

These aren't beginner mistakes — they show up in commercial code
shipped to thousands of customers. They happen because:

- The original developer wrote `verify_image()` and it returned
  successfully because nobody implemented the "fail" branch
- A refactor moved the verify call before the key was loaded
- A debug `#define` that bypassed verification got left on in production

**The fix:** Test the failure path explicitly. Build firmware with an
intentionally invalid signature, attempt to boot it, and confirm the
device refuses. This single test catches more secure boot bugs than any
amount of code review. Run it on every release as a CI gate.

## Pitfall 5: Time-of-Check to Time-of-Use (TOCTOU) bugs

The bootloader verifies image at address X, then jumps to image at
address X. Sounds fine. But on systems where flash is mapped to memory
and DMA is active, an attacker with hardware access can:

1. Wait for the bootloader to start verification
2. Use a glitch attack or bus interposer to swap memory contents during
   verification or between verification and jump
3. The bootloader verifies the original image, then executes a swapped
   image

This is the TOCTOU class of attacks. It's rare but real, and high-value
products (set-top boxes, game consoles) have been broken this way.

**The fix:** Copy the verified image to a memory region the attacker
can't modify (typically internal SRAM, with appropriate access
controls), then verify and execute from there. Or use silicon features
like execute-only memory and integrity protection. This costs RAM,
which is why it's often skipped.

## Pitfall 6: No anti-rollback protection

Your firmware v1.0 has a critical vulnerability. You ship v1.1 that
patches it. An attacker downgrades the device to v1.0 and exploits the
known bug.

This is rollback, and without explicit protection, every secure boot
implementation is vulnerable.

The fix is conceptually simple: maintain a monotonic counter (in OTP
fuses, replay-protected memory, or a secure element) of the minimum
acceptable firmware version. The bootloader rejects any image with a
lower version number than the counter.

But:

- The counter has to be incremented atomically with the firmware
  installation, or you brick devices
- The counter has to be in storage that survives flash erasure but
  can't be easily forged
- Old firmware images with valid signatures and lower version numbers
  are still in the wild and have to be invalidated

This is hard to retrofit. It needs to be designed in from day one.

**The fix:** Build rollback protection into the bootloader from the
start. Use OTP-based version counters where possible. Test downgrade
attacks explicitly — burn a v2 firmware, then try to flash signed v1,
confirm the device rejects it.

## Pitfall 7: Glitch attacks aren't considered

Voltage glitching, clock glitching, and electromagnetic fault injection
let an attacker skip arbitrary instructions during boot — including the
"if signature invalid, halt" instruction. With a few hundred dollars
of equipment and patience, even a fairly hardened bootloader can be
defeated this way.

Most secure boot implementations don't consider glitch attacks because:

- The threat model assumes "firmware-only" attackers
- Hardware is harder to test than software
- Glitch resistance requires specific code patterns and isn't free

**The fix:** This depends on your threat model. For a smart bulb,
maybe glitch resistance isn't needed. For an automotive ECU or a
payment terminal, it absolutely is. Design choices that help:

- Verify signatures multiple times with different code paths
- Use redundant flag variables (e.g., authenticated == 0xA5A55A5A,
  unauthenticated == 0x00000000) so a single bit-flip doesn't change
  the outcome
- Use silicon glitch detection where available
- Include explicit delays and integrity checks in critical paths

State your threat model honestly. If glitch attacks are out of scope,
say so in the security documentation. Don't claim resistance you don't
have.

## Pitfall 8: The chain stops at the first stage

Many products implement "secure boot" by having the boot ROM verify
the first-stage bootloader. The first-stage bootloader then loads the
kernel/application without verification.

This is a half-finished chain. The attacker just modifies the kernel.

A real chain of trust has every stage verifying the next: