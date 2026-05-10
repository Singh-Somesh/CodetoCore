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

Secure boot is one of those features that's easy to claim and hard to
verify. A device that signs and verifies images appears to have it.
But under a realistic threat model, the difference between "we
implement secure boot" and "an attacker can't replace our boot chain"
can be larger than expected.

This post walks through eight common pitfalls in secure boot
implementations — patterns that can quietly undermine the security
property even when the basic mechanics work. None are exotic. All are
worth thinking through before shipping a product that depends on
boot-time trust.

If you're working on a secure boot design, this is a useful checklist
to walk through. If you're reviewing one, these are the questions to
ask.

## What "secure boot" actually has to guarantee

Before the pitfalls, the bar. A working secure boot implementation
should guarantee one thing:

> The code running after reset is the code the manufacturer
> authorized, and an attacker — even one with physical access —
> cannot replace it with their own code.

That's the property. Every component of secure boot exists to make
that statement hold up. If any link in the chain is breakable, the
property doesn't hold, and the system has a feature that signs and
verifies images without delivering the underlying guarantee.

The chain typically has three parts:

1. **Root of trust** — an immutable hardware-anchored element that
   holds or hashes the public key used to verify boot images
2. **Boot ROM verification** — silicon-resident code that checks the
   first-stage bootloader against the root key before executing it
3. **Chain of trust** — each subsequent stage cryptographically
   verifies the next stage before transferring control

Each pitfall below relates to one of these three.

## Pitfall 1: Production fuses never get burned

This is probably the most common gap, and the one that's easiest to
overlook in development.

Most modern microcontrollers have secure boot capability that relies
on **one-time-programmable (OTP) fuses** to lock the device into
secure mode. The flow goes:

1. Generate signing keys
2. Compute the public key hash
3. Write the hash into OTP fuses (irreversible)
4. Burn a "production mode" fuse that disables debug, JTAG, and
   unsigned boot paths

In development, steps 3 and 4 are skipped. The device runs in "test
mode" — secure boot logic *appears* to work because images are signed
and the bootloader checks signatures, but **without the fuses set,
those checks can typically be bypassed**. Test mode often allows:

- Loading unsigned firmware via JTAG
- Disabling secure boot at runtime
- Reading firmware regardless of read-protection settings

It's easy to ship in test mode without realizing it, especially if
the manufacturing flow doesn't include explicit fuse-burning steps.
The result is a device where signature verification runs but doesn't
actually enforce anything.

**What works:** Burn production fuses on every unit before it leaves
the factory. Make this a hard gate in the manufacturing flow. Verify
it on every unit during final test.

**The honest tradeoff:** Burning fuses is irreversible. If firmware
is buggy and bricks devices, recovery isn't possible. This is a real
cost. The choice between "recoverable insecure devices" and
"non-recoverable secure devices" is a real business decision — worth
documenting explicitly when made.

## Pitfall 2: Debug interfaces left enabled

Closely related: even with fuses burned, JTAG, SWD, and ETM
interfaces are often left active. The usual reasoning: "we need it
for field debugging" or "production test still needs it."

A live debug interface on a device claiming secure boot weakens the
security property significantly. With JTAG access, an attacker can:

- Halt the CPU at any point and inject code
- Read memory contents (including keys derived at runtime)
- Modify program flow to skip signature checks
- Set breakpoints inside the verification routine itself

A secure bootloader that can be paused and patched mid-execution
doesn't provide the protection the design intends.

**What works:** Disable debug interfaces in production via OTP fuses.
Use authenticated debug protocols (such as ARM Authenticated Debug
Access Port, ADAC) when field debug is a real requirement. Avoid
"JTAG locking" via firmware-controlled GPIOs — those can be defeated
by glitching or physical bypass.

## Pitfall 3: Signing keys stored insecurely

Secure boot rests on the assumption that the private signing key is,
in fact, private. There are several common ways this assumption can
break:

- Keys committed to git history (often "for testing" then never
  removed)
- Keys stored on a developer's laptop
- Keys in CI/CD environment variables that broad teams can read
- Keys in shared password managers
- Keys printed in build logs that get archived

If any of these happens, the root of trust is compromised — anyone
with access can produce firmware the device will accept.

**What works:** Production signing keys should live in a Hardware
Security Module (HSM). The signing operation happens on the HSM; the
key never leaves it. Build pipelines submit images to a signing
service that fronts the HSM, with audit logs and access controls.

This is operationally heavier than keeping a `signing_key.pem` on the
build server. That weight is the cost of an actual root of trust.

## Pitfall 4: The "verify" step doesn't verify what you'd expect

Verification logic can fail in non-obvious ways. Common patterns to
watch for:

- Computing a CRC32 (not a cryptographic hash) and treating it as
  verification
- Computing SHA-256 but never checking the signature, just the hash
- Verifying the signature against a key embedded in the image being
  verified
- Verifying the signature, then *discarding the result* and
  continuing regardless

These are easy to miss in practice — here's how they typically slip
through:

- The original implementation has `verify_image()` returning success
  by default, and the "fail" branch never fully gets implemented
- A refactor moves the verify call before the key is loaded
- A debug `#define` that bypasses verification gets left in
  production builds

**What works:** Test the failure path explicitly. Build firmware with
an intentionally invalid signature, attempt to boot it, and confirm
the device refuses. This single test catches more secure boot bugs
than code review alone. Run it on every release as a CI gate.

## Pitfall 5: Time-of-Check to Time-of-Use (TOCTOU) bugs

The bootloader verifies the image at address X, then jumps to the
image at address X. Sounds fine. But on systems where flash is mapped
to memory and DMA is active, an attacker with hardware access can:

1. Wait for the bootloader to start verification
2. Use a glitch attack or bus interposer to swap memory contents
   during verification, or between verification and jump
3. The bootloader verifies the original image, then executes a
   swapped image

This is the TOCTOU class of attacks. It's rare in opportunistic
attacks but relevant for high-value targets — set-top boxes, game
consoles, and payment terminals have all been broken this way.

**What works:** Copy the verified image to a memory region the
attacker can't modify (typically internal SRAM with appropriate
access controls), then verify and execute from there. Or use silicon
features like execute-only memory and integrity protection. This
costs RAM, which is why it's often skipped — and why threat models
matter.

## Pitfall 6: No anti-rollback protection

Firmware v1.0 has a critical vulnerability. The vendor ships v1.1 to
patch it. An attacker downgrades the device to v1.0 and exploits the
known bug.

This is rollback, and without explicit protection, secure boot
implementations are vulnerable to it.

The fix is conceptually simple: maintain a monotonic counter (in OTP
fuses, replay-protected memory, or a secure element) of the minimum
acceptable firmware version. The bootloader rejects any image with a
lower version number than the counter.

The implementation is harder:

- The counter has to be incremented atomically with firmware
  installation, or units can get bricked
- The counter has to be in storage that survives flash erasure but
  can't be easily forged
- Old firmware images with valid signatures and lower version numbers
  remain in the wild and have to be invalidated

This is hard to retrofit. Ideally it's designed in from day one.

**What works:** Build rollback protection into the bootloader from
the start. Use OTP-based version counters where available. Test
downgrade attacks explicitly — flash a v2 firmware, then try to
flash signed v1, and confirm the device rejects it.

## Pitfall 7: Glitch attacks aren't considered

Voltage glitching, clock glitching, and electromagnetic fault
injection let an attacker skip arbitrary instructions during boot —
including the "if signature invalid, halt" instruction. With a few
hundred dollars of equipment and patience, fairly hardened
bootloaders can be defeated this way.

Many secure boot implementations don't consider glitch attacks
because:

- The threat model assumes "firmware-only" attackers
- Hardware-level attacks are harder to test for than software ones
- Glitch resistance requires specific code patterns and isn't free

**What works:** This depends on the threat model. For a smart bulb,
glitch resistance might not be needed. For an automotive ECU or a
payment terminal, it usually is. Design choices that help:

- Verify signatures multiple times with different code paths
- Use redundant flag variables (e.g., `authenticated == 0xA5A55A5A`,
  `unauthenticated == 0x00000000`) so a single bit-flip doesn't change
  the outcome
- Use silicon glitch detection where available
- Include explicit delays and integrity checks in critical paths

The important step is stating the threat model honestly. If glitch
attacks are out of scope, that's a valid design decision — as long
as it's documented and the security claim is framed accordingly.

## Pitfall 8: The chain stops at the first stage

Some products implement "secure boot" by having the boot ROM verify
the first-stage bootloader. The first-stage bootloader then loads the
kernel or application without verification.

This leaves the chain half-finished. The attacker simply modifies
whatever the first-stage bootloader loads next.

A complete chain of trust has every stage verifying the next:

- **Boot ROM** verifies → **First-stage bootloader**
- **First-stage bootloader** verifies → **Second-stage bootloader**
- **Second-stage bootloader** verifies → **Secure monitor (TF-A)**
- **Secure monitor** verifies → **TEE (OP-TEE)**
- **TEE** verifies → **Non-secure bootloader**
- **Non-secure bootloader** verifies → **Kernel**

Every step is a signature verification. Skip any one, and the chain
is broken at that point and below.

**What works:** Audit the boot chain. Document every stage, what
verifies the next, with what key, against what policy. Where
verification is missing, add it. A chain with gaps doesn't provide
end-to-end protection.

## What to do about all this

Three practical steps for verifying secure boot rather than assuming:

**1. Threat-model the boot flow.** Write down every stage, every key,
every assumption. Then ask: what does an attacker with physical
access, firmware-only access, and remote access need to do at each
stage to break the chain? Gaps tend to surface quickly.

**2. Test the failure paths.** Build deliberately broken firmware
(invalid signature, modified payload, downgraded version) and confirm
the device refuses every variation. Make these CI tests.

**3. Document residual risk honestly.** Some pitfalls are expensive
to fix. Documenting the ones not addressed — and why — is more
useful than blanket claims. "We don't protect against glitch attacks
because our threat model excludes attackers with hardware access" is
a valid statement. "We have secure boot" without that qualifier is
incomplete.

## The principle worth remembering

Secure boot isn't a feature that gets turned on. It's a property of
the system that has to be designed, implemented, tested, and verified
at every layer — silicon, bootloader, build pipeline, manufacturing,
field operations.

When secure boot doesn't deliver on its promise, it's usually because
the obvious 60% got implemented and the remaining 40% — the failure
modes, the operational discipline, the threat model honesty — was
treated as out of scope or "we'll get to it later."

Treating secure boot as a checkbox produces something that looks like
secure boot. Treating it as a property to be continuously verified
produces the actual thing.

---
