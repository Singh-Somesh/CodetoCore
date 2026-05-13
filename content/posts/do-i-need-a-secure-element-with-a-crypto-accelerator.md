---
title: "Do I Need a Secure Element If My MCU Has a Crypto Accelerator?"
date: 2026-03-15
draft: false
tags: ["Quick Take", "Cryptography", "Secure Element", "ECC", "HRoT", "Embedded Security"]
categories: ["Embedded Security"]
description: "Short answer: they solve different problems. A crypto accelerator makes crypto operations fast. A secure element makes sure your keys never leave the chip. You usually want both, for different reasons."
summary: "Why a hardware accelerator and a secure element aren't substitutes for each other, when one is enough, and when you actually want both."
ShowToc: false
ShowReadingTime: true
---


**What a crypto accelerator gives you:**
Speed. AES, SHA, ECC, RSA in hardware instead of software, often 10–100× faster. The CPU triggers the operation, the hardware engine does the math, the result comes back. That's it.

The key still lives in regular MCU memory — RAM, flash, wherever you put it. The CPU can read it. Any code running on the MCU can read it. The accelerator just does the math faster.

**What a secure element gives you:**
A hardware boundary around the key. Parts like the NXP SE050, Infineon OPTIGA Trust M, ST STSAFE-A110 — small separate chips, usually I²C or SPI. They have their own CPU, their own memory, their own crypto engine, and physical anti-tamper protections.

You don't read the key out. The MCU asks the secure element to *sign* or *decrypt* — sends in data, gets a result back. The private key never crosses the bus.

Anti-tamper matters too. Secure elements are built to defeat physical attacks — voltage glitches, side channels, probing the silicon. A general-purpose MCU isn't.

**When a crypto accelerator alone is enough:**

- Bulk encryption with short-lived keys (TLS session keys derived per connection).
- Performance-critical paths where the key is short-lived anyway.

**When you need a secure element:**

- Long-lived identity keys — device certificates, attestation keys.
- Anti-counterfeit. If someone clones your device, you want the identity key uncloneable.
- Compliance (FIPS, Common Criteria) requires hardware key storage.
- The device is in the field, in someone's hands. Physical access is the threat.

**When you want both (most production designs):**
The secure element holds the long-lived identity keys. The MCU's accelerator handles bulk crypto — once a TLS session is up, the session keys are short-lived, and the accelerator's the right tool for them.

The mental model worth keeping: an accelerator is a *math engine*. A secure element is a *key vault*. They're not interchangeable, and pricing them against each other ("the accelerator is free on my MCU, why pay for a secure element?") misses the point.

*This is personal exploration on publicly available reference platforms; it does not represent the views or work of my employer.*