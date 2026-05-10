---
title: "Why Your Secure Boot Probably Isn't Actually Secure"
date: 2025-05-10
draft: false
tags:
  - secure-boot
  - embedded-security
  - hardware-root-of-trust
  - threat-modeling
categories:
  - Security
description: "Most secure boot implementations look fine on paper but fail under realistic threat models. Eight common pitfalls — and what fixing them requires."
summary: "Most secure boot implementations look fine on paper but fail under realistic threat models. Eight common pitfalls — and what fixing them requires."
ShowToc: true
TocOpen: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowCodeCopyButtons: true
---

Secure boot is one of the most claimed and least verified properties
in embedded products today. Many teams ship devices marketed as having
secure boot when, under a realistic threat model, the implementation
falls apart.

This post walks through eight common pitfalls — failure modes that
appear repeatedly across vendors, architectures, and product
categories. None are exotic; all are overlooked because secure boot
gets treated as a checkbox feature rather than a system property that
must be designed, implemented, and continuously verified.

If you're shipping a product with secure boot, this is a forcing
function: read each pitfall and ask whether your implementation is
actually safe from it.

## What "secure boot" actually has to guarantee

Before the pitfalls, the bar. A working secure boot implementation has
to guarantee one thing:

> The code running after reset is the code the manufacturer authorized,
> and an attacker — even one with physical access — cannot replace it
> with their own code.

That's it. Every component of secure boot exists to make that statement
true. If any link in the chain is breakable, the property doesn't hold,
and the system has a feature that looks like secure boot rather than
secure boot itself.

The chain typically has three parts:

1. **Root of trust** — an immutable hardware-anchored element that
   holds or hashes the public key used to verify boot images
2. **Boot ROM verification** — silicon-resident code that checks the
   first-stage bootloader against the root key before executing it
3. **Chain of trust** — each subsequent stage cryptographically verifies
   the next stage before transferring control

Every pitfall below breaks one of these three.

## Pitfall 1: Production fuses never get burned

This is one of the most common failure modes, and the most embarrassing
when discovered.

Most modern microcontrollers ship with secure boot capability that
relies on **one-time-programmable (OTP) fuses** to lock the device into
secure mode. The flow is something like:

1. Generate signing keys
2. Compute the public key hash
3. Write the hash into OTP fuses (irreversible)
4. Burn a "production mode" fuse that disables debug, JTAG, and
   unsigned boot paths

In development, steps 3 and 4 are skipped. The device runs in "test
mode" — secure boot logic *appears* to work because images are signed
and the bootloader checks signatures, but **without the fuses set,
those checks can be bypassed**. Test mode typically allows:

- Loading unsigned firmware via JTAG
- Disabling secure boot at runtime
- Reading firmware regardless of read-protection settings

Many products ship in test mode. Sometimes nobody on the team realized
the fuses needed to be burned. Sometimes the fuses were skipped because
"we want to be able to recover bricked units in the field." Either way,
the entire secure boot story is fiction.

**The fix:** Burn production fuses on every unit before it leaves the
factory. Make this a hard gate in the manufacturing flow. Verify it on
every unit during final test.

**The honest tradeoff:** Burning fuses is irreversible. If firmware is
buggy and bricks devices, recovery isn't possible. This is a real cost.
But "we'd rather have recoverable insecure devices" is a business
decision, not a security strategy. Document it explicitly when chosen.

## Pitfall 2: Debug interfaces left enabled

Closely related: even with fuses burned, JTAG, SWD, and ETM interfaces
are often left active. The reasoning is always the same — "we need it
for field debugging" or "production test needs it."

A live debug interface on a device claiming secure boot is a
contradiction. With JTAG access, an attacker can:

- Halt the CPU at any point and inject code
- Read memory contents (including keys derived at runtime)
- Modify program flow to skip signature checks
- Set breakpoints inside the verification routine itself

A secure bootloader that can be paused and patched mid-execution is
not secure.

**The fix:** Disable debug interfaces in production via OTP fuses. Use
authenticated debug protocols (such as ARM Authenticated Debug Access
Port, ADAC) when field debug is a real requirement. Don't use "JTAG
locking" via firmware-controlled GPIOs — those can be defeated by
glitching or physical bypass.

## Pitfall 3: Signing keys live in plaintext somewhere

Secure boot rests on the assumption that the private signing key is,
in fact, private. In practice, this assumption breaks in predictable
ways:

- Keys committed to git history (often "for testing" then never
  removed)
- Keys stored on a developer's laptop, sometimes for years
- Keys in CI/CD environment variables that 30 engineers can read
- Keys in shared password managers
- Keys printed in build logs that get archived to S3

If any of these happen, the root of trust is compromised the moment
someone with access decides to (or is compelled to) leak the key.

**The fix:** Production signing keys must live in a Hardware Security
Module (HSM). The signing operation happens on the HSM; the key never
leaves it. Build pipelines submit images to a signing service that
fronts the HSM, with audit logs and access controls.

This is operationally heavier than keeping `signing_key.pem` on the
build server. That weight is the cost of having an actual root of
trust.

## Pitfall 4: The "verify" step doesn't verify what you think

Bootloaders ship in production with verification logic that:

- Computes a CRC32 (not a cryptographic hash)
- Computes SHA-256 but never checks the signature, just the hash
- Verifies the signature against a key embedded in the image being
  verified
- Verifies the signature, then *discards the result* and continues
  regardless

These aren't beginner mistakes — they appear in commercial code shipped
to thousands of customers. They happen because:

- The original developer wrote `verify_image()` and it returned
  successfully because nobody implemented the "fail" branch
- A refactor moved the verify call before the key was loaded
- A debug `#define` that bypassed verification got left on in
  production

**The fix:** Test the failure path explicitly. Build firmware with an
intentionally invalid signature, attempt to boot it, and confirm the
device refuses. This single test catches more secure boot bugs than
any amount of code review. Run it on every release as a CI gate.

## Pitfall 5: Time-of-Check to Time-of-Use (TOCTOU) bugs

The bootloader verifies the image at address X, then jumps to the
image at address X. Sounds fine. But on systems where flash is mapped
to memory and DMA is active, an attacker with hardware access can:

1. Wait for the bootloader to start verification
2. Use a glitch attack or bus interposer to swap memory contents
   during verification or between verification and jump
3. The bootloader verifies the original image, then executes a swapped
   image

This is the TOCTOU class of attacks. It's rare in opportunistic
attacks but real for high-value targets — set-top boxes, game
consoles, and payment terminals have all been broken this way.

**The fix:** Copy the verified image to a memory region the attacker
can't modify (typically internal SRAM with appropriate access
controls), then verify and execute from there. Or use silicon features
like execute-only memory and integrity protection. This costs RAM,
which is why it's often skipped.

## Pitfall 6: No anti-rollback protection

Firmware v1.0 has a critical vulnerability. The vendor ships v1.1 to
patch it. An attacker downgrades the device to v1.0 and exploits the
known bug.

This is rollback, and without explicit protection, every secure boot
implementation is vulnerable.

The fix is conceptually simple: maintain a monotonic counter (in OTP
fuses, replay-protected memory, or a secure element) of the minimum
acceptable firmware version. The bootloader rejects any image with a
lower version number than the counter.

But:

- The counter has to be incremented atomically with firmware
  installation, or units get bricked
- The counter has to be in storage that survives flash erasure but
  can't be easily forged
- Old firmware images with valid signatures and lower version numbers
  are still in the wild and have to be invalidated

This is hard to retrofit. It needs to be designed in from day one.

**The fix:** Build rollback protection into the bootloader from the
start. Use OTP-based version counters where available. Test downgrade
attacks explicitly — flash a v2 firmware, then try to flash signed v1,
confirm the device rejects it.

## Pitfall 7: Glitch attacks aren't considered

Voltage glitching, clock glitching, and electromagnetic fault
injection let an attacker skip arbitrary instructions during boot —
including the "if signature invalid, halt" instruction. With a few
hundred dollars of equipment and patience, even a fairly hardened
bootloader can be defeated this way.

Most secure boot implementations don't consider glitch attacks
because:

- The threat model assumes "firmware-only" attackers
- Hardware-level attacks are harder to test for than software ones
- Glitch resistance requires specific code patterns and isn't free

**The fix:** This depends on the threat model. For a smart bulb,
glitch resistance might not be needed. For an automotive ECU or a
payment terminal, it absolutely is. Design choices that help:

- Verify signatures multiple times with different code paths
- Use redundant flag variables (e.g., `authenticated == 0xA5A55A5A`,
  `unauthenticated == 0x00000000`) so a single bit-flip doesn't change
  the outcome
- Use silicon glitch detection where available
- Include explicit delays and integrity checks in critical paths

State the threat model honestly. If glitch attacks are out of scope,
say so in the security documentation. Don't claim resistance that
isn't there.

## Pitfall 8: The chain stops at the first stage

Many products implement "secure boot" by having the boot ROM verify
the first-stage bootloader. The first-stage bootloader then loads the
kernel or application without verification.

This is a half-finished chain. The attacker simply modifies the
kernel.

A real chain of trust has every stage verifying the next:

A real chain of trust has every stage verifying the next:

- **Boot ROM** verifies → **MB1**
- **MB1** verifies → **MB2**
- **MB2** verifies → **TF-A**
- **TF-A** verifies → **OP-TEE**
- **OP-TEE** verifies → **CBoot**
- **CBoot** verifies → **Kernel**

Each arrow represents a signature verification. Skip any one and the chain is broken from that point onward.

Every arrow is a signature verification. Skip any one and the chain
is broken at that point and below.

**The fix:** Audit the boot chain. Document every stage, what
verifies the next, with what key, against what policy. Where
verification is missing, add it. The bootloader on most platforms is
non-trivial to modify, but a chain with gaps isn't a chain.

## What to do about all this

Three practical steps for any team that wants to verify rather than
assume:

**1. Threat-model the boot flow.** Write down every stage, every key,
every assumption. Then ask: what does an attacker with physical
access, firmware-only access, and remote access need to do at each
stage to break the chain? Gaps appear quickly.

**2. Test the failure paths.** Build deliberately broken firmware
(invalid signature, modified payload, downgraded version) and confirm
the device refuses every variation. Make these CI tests.

**3. Be honest about residual risk.** Some pitfalls are expensive to
fix. Document the ones not being addressed and why. "We don't protect
against glitch attacks because our threat model excludes attackers
with hardware access" is a valid statement. "We have secure boot"
without that qualifier is misleading.

## The principle that ties them together

Secure boot isn't a feature that gets turned on. It's a property of
the system that has to be designed, implemented, tested, and verified
at every layer — silicon, bootloader, build pipeline, manufacturing,
field operations.

The reason most "secure boot" isn't actually secure is the same
reason most "encrypted communications" aren't actually encrypted and
most "authenticated APIs" aren't actually authenticated: the obvious
60% gets implemented and called done. The remaining 40% — the failure
modes, the operational discipline, the threat model honesty — is
where the security actually lives.

A team that treats secure boot as a checkbox doesn't have secure boot.
A team that treats it as a property to be continuously verified
might.

---
