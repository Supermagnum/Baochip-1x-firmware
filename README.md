# Baochip-1x Firmware

Firmware for **USB hardware security tokens** and protected cryptographic storage on the **Baochip-1x** SoC (Dabao evaluation board): OpenPGP smartcard-class behavior and vault semantics in the same product category as **Nitrokey-class** devices, with the platform and boot security model documented here. This README covers **platform selection**, **hardware capabilities**, **boot and zeroisation**, **requirement mapping**, and **standalone cryptographic capabilities** (GnuPG/OpenPGP-class algorithms and curves, the **extended on-device profile** below, and **Shamir secret sharing**).

---

## Platform selection: Baochip-1x

The Baochip-1x SoC, as exercised on the **Dabao** evaluation board, has been evaluated against the token-platform requirements referenced in this README. It satisfies the **hardware and boot-architecture** requirements called out here; remaining gaps are **implementation tasks** in firmware and host software, not fundamental limitations of the silicon or boot model.

---

## Hardware specification (Dabao evaluation board)

| Area | Specification |
|------|----------------|
| **Application CPU** | 350 MHz VexRiscv RV32-IMAC with MMU and **Zkn** scalar cryptography extensions (AES instructions in the CPU pipeline) |
| **I/O coprocessors** | 4 x 700 MHz PicoRV32 **BIO** cores with custom register extensions |
| **On-chip memory** | 4 MiB **RRAM** (non-volatile, XIP up to ~1200 MB/s); 2 MiB SRAM + 256 KiB I/O buffers |
| **Swap** | Swap memory supported |
| **Crypto accelerators** (175 MHz) | **PKE**: RSA, ECC, ECDSA, X25519, GCD; **ComboHash**: HMAC, SHA-256, SHA-512, SHA3, RIPEMD, Blake2, Blake3; **AES**; **ALU**; **TRNG**; **SDMA** |
| **PKE flexibility** | Algorithm-agnostic engine: 256-bit field prime supplied at runtime via **N0Dat**; Montgomery parameter computed on-the-fly. **BrainpoolP256r1** confirmed viable without firmware changes to the PKE RTL (see verification notes for `PkeCore.sv`) |
| **Physical hardening** | Glitch sensors, security mesh, PV sensor, ECC-protected RAM |
| **Always-on domain** | AORAM (8 KiB), 256-bit backup register, one-way counters, WDT, RTC (state survives power loss) |
| **Boot and secrets** | Signed boot, key store, hardware one-way counters |
| **USB** | USB High Speed via USB-C |
| **Openness** | Fully open RTL (CERN-OHL-W-2.0), reproducible bootloader, Rust-based [Xous](https://github.com/betrusted-io/xous-core) OS, open schematics; **IRIS-inspectable** silicon |

### Optional external memory

An optional **PSRAM** chip may be present for bulk encrypted storage presentation to the host (see [Host-visible behavior](#host-visible-behavior)).

---

## Boot chain security model

1. **Immutable boot0** is programmed at **wafer probe**. **JTAG is fused out** after packaging.
2. Boot0 runs **before** USB enumeration, application load, and any host interaction.
3. Boot0 verifies **itself** and **boot1** using up to **four Ed25519** public keys burned into the device.
4. On verification **failure**: volatile state is **actively zeroised**; the CPU is then halted.
5. **One-way counters** (always-on domain) drive key revocation, boot mode, board type coding, and related policy. They are **hardware-enforced**, monotonic across power cycles.

This satisfies an **active zeroisation** requirement at a layer **below** replaceable application software: the zeroisation controller runs in the immutable boot path with access to sensitive regions.

---

## Requirement mapping

| Requirement | Status | Notes |
|-------------|--------|--------|
| Ephemeral ECDH on-device | Implementable | Xous application; MMU process isolation |
| BrainpoolP256r1 HW acceleration | Confirmed | PKE runtime-configurable; aligned with RTL verification |
| ChaCha20 / Poly1305 HW acceleration | Unconfirmed | Software on 350 MHz VexRiscv; acceptable for token-class throughput |
| HKDF / HMAC-SHA256/512 HW acceleration | Confirmed | ComboHash block; HKDF composition is a firmware responsibility |
| TRNG | Confirmed | Dedicated block; ring-oscillator sourced |
| Programmable open application layer | Confirmed | Open RTL, reproducible bootloader, open Xous/Rust stack |
| Measured boot / integrity-bound key vault | Confirmed | Signed boot, immutable boot0, Ed25519 verification |
| Full open stack | Confirmed | RTL, bootloader, OS, schematics open |
| Active zeroisation | Confirmed | Boot0 path on signature failure; PIN-threshold extension is firmware work |
| Power-loss resume during zeroisation | Architecturally supported | AORAM + always-on domain + boot0 pre-enumeration checks |
| Monotonic PIN counter, tamper-evident | Confirmed | Hardware one-way counters in always-on domain |
| Physical attack hardening | Confirmed | Glitch sensors, mesh, PV sensor, ECC RAM |
| Algorithm-agnostic vault / HKDF domain separation | Implementable | Application design and test vectors |
| USB form factor (production) | Pending | Dabao is an evaluation board; **custom carrier** needed for product shape |

---

## Implementation notes

### Extending active zeroisation to PIN exhaustion

Boot0 already implements a **reference zeroisation path** on signature verification failure. To tie **PIN attempt exhaustion** to the same guarantees:

1. Increment a **one-way counter** in the always-on domain on each failed PIN attempt.
2. During boot0 **pre-USB-enumeration** sequence, compare the counter to a **policy threshold**.
3. On threshold breach, trigger the same **TRNG-sourced multi-pass overwrite** across sensitive regions: key vault, AORAM, SRAM holding derived key material, and other policy-defined banks.

**Power-loss during zeroisation**: Because the device does not enumerate until boot0 completes, an interrupted wipe **resumes** on next power-up within the boot0 flow.

### Key storage capacity (illustrative)

Rough order of magnitude: on-chip **4 MiB RRAM** can hold on the order of **~1024 BrainpoolP256r1 key pairs** (actual count depends on metadata, padding, and vault layout).

### Developer mode and SKU policy

Transitioning a part to **developer mode** must **permanently erase** the secret key area with **no recovery path**. **Production tokens** should use standard SKU silicon, not engineering samples (e.g. ES suffix / engineering BGA variants) intended for bring-up only.

---

## Host-visible behavior

### Uninformed host or adversary

The device presents as a **standard USB mass storage** volume. Optional PSRAM-backed content appears as a **normal (encrypted) filesystem**. There is no requirement for special drivers for this personality.

### Informed host (custom protocol or driver)

With a **vendor or integrator-supplied** driver or userspace component that speaks the token’s protocol, the host can:

1. Recognise the device class or protocol extension.
2. Issue a challenge for a **minimum 5-character alphanumeric passphrase** (policy may enforce longer secrets).
3. On **correct** response, unlock **real key material** from **RRAM** and mount the **protected volume**.
4. On **wrong** response or **absence of host support**, the device **falls back** to mass-storage behavior **without advertising** the enhanced path.

Integrators remain responsible for key management, export and import controls, and applicable local laws for cryptographic hardware and storage. This README does not grant legal authorization for any specific deployment or jurisdiction.

---

## Cryptographic capabilities (standalone firmware)

The firmware is **standalone**: it does not depend on any particular host-side application or driver stack. Cryptographic coverage is defined by **GnuPG / OpenPGP** interoperability targets, the **extended on-device profile** in the tables below, and **Shamir secret sharing**.

1. **GnuPG / OpenPGP** — algorithms and curves that appear in contemporary **GnuPG** (`gpg`) and **OpenPGP** usage (key types, ciphers, digests, and key-wrapping conventions the token is designed to support).
2. **Extended on-device profile** — Brainpool ECC (P256r1, P384r1, P512r1), AEAD suites (AES-GCM, ChaCha20-Poly1305), HKDF, HMAC, PBKDF2, ECIES-style and multi-recipient constructions, authenticated encryption profiles, and digests available from **ComboHash** (SHA-2, SHA-3, RIPEMD, Blake2, Blake3), as reflected in the following subsections.
3. **Shamir secret sharing** — **K-of-N** threshold split and recovery **in firmware** (generation of shares, secure handling of shares at rest, and reconstruction when at least **K** valid shares are supplied).

Implementation uses the **Baochip-1x accelerators** where applicable and **software** (VexRiscv, Xous) elsewhere. The following summarizes scope; exact API surfaces and test vectors belong in this repository’s code and release documentation.

### Symmetric and AEAD (GnuPG and extended profile)

| Area | Notes |
|------|--------|
| **AES** | 128 / 192 / 256-bit keys; modes including CBC, CTR, GCM, ECB as required by supported profiles (**HW AES** and **Zkn** where used) |
| **ChaCha20-Poly1305** | RFC 8439 AEAD; **software** on VexRiscv if no dedicated HW path (see requirement mapping) |
| **Other OpenPGP ciphers** | Per product profile (e.g. Camellia, legacy 3DES) where GnuPG interoperability is required |

### Public-key: curves and algorithms (GnuPG and extended profile)

| Area | Notes |
|------|--------|
| **RSA** | Encryption, signing, verification sizes supported by **PKE** and policy |
| **NIST elliptic curves** | ECDH / ECDSA as used in OpenPGP (e.g. P-256, P-384, P-521) subject to PKE configuration and validation |
| **Brainpool** | **brainpoolP256r1**, **brainpoolP384r1**, **brainpoolP512r1** — ECDH, ECDSA, ECIES-style encrypt/decrypt and related flows (**P256r1** confirmed on PKE with runtime prime); multi-recipient patterns as implemented in firmware |
| **Curve25519 / Ed25519** | **X25519** / **Ed25519** (and OpenPGP Cv25519 / Ed25519 usage) via **PKE** and software glue as required |
| **ECDH / ECDSA** | Ephemeral and static key agreement and signatures per supported curves |

### Digests, MAC, and key derivation

| Mechanism | Notes |
|-----------|--------|
| **SHA-2** | SHA-224, SHA-256, SHA-384, SHA-512 (OpenPGP and modern profiles) |
| **SHA-3** | Where required by policy or interchange |
| **RIPEMD-160** | OpenPGP legacy interoperability where enabled |
| **Blake2 / Blake3** | Via **ComboHash** and extended profile |
| **HMAC** | HMAC-SHA256, HMAC-SHA512 and variants aligned with **ComboHash** |
| **HKDF** | RFC 5869; composed in firmware with explicit **info** / salt for domain separation |
| **PBKDF2** | OpenPGP S2K and standalone KDF use cases |
| **OpenPGP S2K** | String-to-key modes required for GnuPG-class interoperability |

### Shamir secret sharing (firmware)

- **Threshold K-of-N**: split secrets into **N** shares with threshold **K**; recover the secret only when **K** (or more) valid shares are combined.
- Uses finite-field arithmetic consistent with chosen security parameters (e.g. prime or curve constraints for embedded secrets, as specified in implementation docs).
- Intended for **vault policies**, **backup/recovery**, and **quorum-controlled** key material — independent of host OS or application stack.

### Authenticated ephemeral ECDH (recommended session model)

Use **long-term** OpenPGP / GnuPG keys **only to authenticate** an additional **ephemeral ECDH** exchange, not as the direct input to bulk session keys.

1. At **session start**, both parties generate a **fresh ephemeral key pair**.
2. They **exchange ephemeral public keys**, each binding **signed with the long-term key** so the peer cannot be impersonated.
3. They **derive the session key** from the **ephemeral shared secret** (for example with HKDF over the ECDH output, with explicit domain separation).
4. They **immediately destroy** the **ephemeral private key** when the session ends or when the derived keys are installed.

**Security property:** compromise of a **long-term** signing key lets an attacker **impersonate** the owner in **future** sessions (forge authenticated ephemeral offers) but does **not** by itself allow **decryption of past** sessions, because those used secrets that depended on ephemeral private values that should no longer exist.

This firmware’s **Ephemeral ECDH on-device** row in [Requirement mapping](#requirement-mapping) is aimed at implementing that pattern with MMU-isolated Xous components and secure erasure of ephemeral material.

### Ephemeral ECDH and Shamir: gap in commercial tokens (creators’ knowledge)

To the **creators’ knowledge**, as of **25 March 2026**, **no hardware security device** they are aware of supports **ephemeral key pairs** (in the authenticated ephemeral ECDH sense above) **or** **Shamir’s secret** sharing as first-class on-token features at the time of writing. This project is intended to close that gap on **Baochip-1x** through open firmware. Other products may add such capabilities later; integrators should verify current vendor documentation.

### Optional / profile-gated features

Algorithms that are **optional in GnuPG builds** or **experimental in OpenPGP** may be enabled per **firmware profile**. Document what is compiled in for each release.

---

## Cryptographic algorithm background

The following is **general background** on how some algorithms relate to standards bodies and historical influence. It is not specific legal or security advice; it may help when choosing among options this firmware can offer (for example preferring **Brainpool**, **ChaCha20**, or **RSA** where policy calls for diversity from NSA-influenced defaults).

### Cryptographic ciphers influenced by the NSA

The National Security Agency (NSA) has been involved in various cryptographic standards and algorithms. Here are some ciphers likely influenced by the NSA:

| Cipher | Description |
|--------|-------------|
| **AES** (Advanced Encryption Standard) | Endorsed by the NSA for federal applications, widely used for secure data encryption. |
| **DSA** (Digital Signature Algorithm) | Developed under NSA auspices, commonly used for digital signatures. |
| **SHA** (Secure Hash Algorithm) | NSA has influenced multiple versions, with SHA-1 and SHA-2 being widely used and critiqued for certain vulnerabilities. |
| **Skipjack** | Created by the NSA for the Clipper chip, aimed at secure voice communications. |
| **KASUMI** | A block cipher influenced by NSA standards, utilized in 3G cellular networks. |

### Cryptographic ciphers not influenced by the NSA

Several algorithms developed independently of the NSA are widely used:

| Cipher | Description |
|--------|-------------|
| **RSA** (Rivest–Shamir–Adleman) | An academic standard widely used for secure key exchange, not influenced by NSA. |
| **Elliptic Curve Cryptography (ECC)** | Developed independently, focusing on secure and efficient cryptographic solutions. |
| **ChaCha20** | Designed by Daniel Bernstein for speed and security, with no NSA involvement. |
| **Twofish** | An AES finalist created by Bruce Schneier, independently developed. |
| **Serpent** | Another AES finalist, also created without direct NSA influence. |
| **Brainpool** | A suite of elliptic curves (e.g. Brainpool P-256) developed without NSA influence, though it is implemented in many cryptographic systems. |

**Summary:** While several ciphers have ties to the NSA, such as AES and SHA, there are many robust alternatives like RSA, ChaCha20, and Brainpool, developed independently. Understanding these distinctions helps in choosing secure cryptographic solutions.

### Known scandals involving the NSA and cryptography

Several scandals and controversies have surrounded the NSA’s involvement in cryptography, revealing concerns about security, privacy, and possible manipulation of standards. Here are some key incidents:

| Incident | Description |
|----------|-------------|
| **NSA’s involvement in Dual_EC_DRBG** | This random number generator was adopted by NIST but later revealed to be potentially compromised by the NSA, raising suspicions of backdoors. |
| **PRISM** | Exposed by Edward Snowden in 2013, revealing that the NSA collects data from major tech companies, including communications encrypted using NSA-influenced standards. |
| **Clapper’s misleading testimony** | Then-Director James Clapper’s testimony before Congress in 2013 was scrutinized after revelations about extensive surveillance practices came to light. |
| **Clipper Chip** | Launched in the early 1990s, it aimed to provide secure phone communication but faced backlash due to mandatory key escrow, which many viewed as a significant privacy infringement. |
| **SHA-1 deprecation** | The SHA-1 hashing algorithm, once endorsed by the NSA, was later found vulnerable, leading to its deprecation and questions about the NSA’s early assessments of its security. |

**Summary:** These incidents highlight significant concerns regarding the NSA’s influence in cryptography and the potential implications for security and privacy. The revelations have fostered a mistrust of cryptographic standards and increased the demand for independent auditing and verification of cryptographic algorithms.

---

## Repository role

This repository holds **firmware** (boot stages beyond public RTL if applicable, Xous services, USB personalities, vault layout, and test harnesses). It does **not** replace open RTL, bootloader sources, or the Xous tree; those remain upstream with reproducible build instructions published alongside releases.

---

## References

- **GnuPG / OpenPGP:** [GnuPG](https://gnupg.org/) and the OpenPGP specifications as implemented by `gpg` for interoperability targets.
- Baochip-1x / Dabao hardware documentation and RTL (CERN-OHL-W-2.0) as published by the silicon and board maintainers.
- Xous operating system: [xous-core](https://github.com/betrusted-io/xous-core).

---

## Disclaimer

**Legal and compliance** (cryptographic export and import controls, data-protection and key-storage rules in your jurisdiction, corporate policy, etc.) is the responsibility of **integrators, vendors, and end users**. This README describes **technical capability** only; it is not legal advice and does not authorize any specific deployment.
