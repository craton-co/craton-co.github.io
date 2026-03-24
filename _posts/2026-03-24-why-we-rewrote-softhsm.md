---
layout: post
title: "Why We Rewrote SoftHSMv2 in Rust"
date: 2026-03-24
author: "Victor Bobrovskiy"
description: "Craton HSM is a memory-safe, post-quantum-ready PKCS#11 software HSM in Rust — a modern replacement for the unmaintained SoftHSMv2."
tags: [rust, security, cryptography, pkcs11, hsm, post-quantum]
---

*By Victor Bobrovskiy, Craton Software Company — March 24, 2026*

SoftHSMv2 is one of those projects that quietly holds up a lot of the internet. If you've ever set up DNSSEC, tested a PKCS#11 integration, or needed a software-based crypto token for development, you've probably used it. It's the default HSM for OpenDNSSEC. Java's SunPKCS11 provider works with it out of the box. OpenSSL can load it as an engine.

It's also written in C++, hasn't had a release since 2020, and handles your private keys in manually-managed memory.

We built Craton HSM to fix that.

## The problem with C++ in HSM software

An HSM's entire purpose is to protect cryptographic keys. It's the one piece of software that is expected to never leak key material, never corrupt key state, and never expose secrets through side channels.

C++ makes all of these things possible through ordinary bugs:

**Buffer overflows** are the classic. A miscalculated length in a PKCS#11 `C_Sign` call writes past the end of an output buffer. In an HSM, that buffer might sit next to private key material in memory.

**Use-after-free** on key objects is worse. A session closes, the key object is freed, but a dangling pointer still references the memory. The next allocation reuses it. Now a new session can read fragments of the old session's private key through a completely unrelated object.

**Missing zeroization** is the most insidious. C++ destructors don't zero memory by default. When a key object goes out of scope, the bytes stay in the freed heap until something else overwrites them. If the process forks, the child gets a copy. If the page gets swapped to disk, the key hits persistent storage.

None of these require a sophisticated attacker. They're the normal failure modes of manual memory management applied to security-critical data.

## What Rust gives us

Rust doesn't just reduce these risks — it eliminates entire categories by construction:

**Ownership prevents use-after-free.** When a `Session` drops, it owns its key handles. The compiler won't let another session hold a reference to freed memory. This isn't a runtime check — it's a compile-time guarantee.

**`ZeroizeOnDrop` automates secret cleanup.** Every type that holds key material in Craton HSM implements `ZeroizeOnDrop`. When the value goes out of scope, the memory is zeroed before deallocation. The compiler inserts this automatically through the `Drop` trait. You can't forget it.

**`mlock` is wired into the lifecycle.** Our `RawKeyMaterial` type calls `mlock` on construction and `munlock` on drop. Key bytes are locked in physical memory for their entire lifetime. On Windows, we use `VirtualLock`. The compiler enforces that you can't create raw key material without locking it.

**Concurrency is safe by default.** SoftHSMv2 uses pthread mutexes to protect shared state. Miss a lock acquisition, and you have a data race on cryptographic state. Craton HSM uses `DashMap` for session storage — the compiler prevents data races at compile time.

**Bounds checking is automatic.** Every array access, every slice operation, every buffer copy is bounds-checked. A malformed PKCS#11 call that passes an incorrect buffer length gets a controlled error, not a heap corruption.

## The PKCS#11 boundary

PKCS#11 is a C interface specification. There's no way around it — the exported functions must use C types, C calling conventions, and raw pointers. This is where `unsafe` lives in Craton HSM.

We made a deliberate architectural choice: all `unsafe` code is confined to the `src/pkcs11_abi/` module. Every exported function wraps its body in `catch_unwind` to prevent Rust panics from crossing the FFI boundary (which is undefined behavior). Five module roots — audit, session, config, store, and token — carry `#![forbid(unsafe_code)]`, making it a compiler error to introduce `unsafe` anywhere in those modules.

The result is a clear boundary: the outer C ABI layer is `unsafe` (it must be), and everything behind it is safe Rust enforced by the compiler.

## Post-quantum cryptography through a 30-year-old interface

PKCS#11 was designed in the 1990s. It predates elliptic curve cryptography, let alone lattice-based algorithms. But it has a mechanism extension model that works surprisingly well for PQC.

Craton HSM exposes ML-KEM-768 (CRYSTALS-Kyber), ML-DSA (CRYSTALS-Dilithium), and SLH-DSA (SPHINCS+) through standard PKCS#11 `C_GenerateKeyPair`, `C_Sign`, `C_Verify`, and `C_DeriveKey` calls. Applications that already use PKCS#11 for key management can start doing PQC key generation and signing without code changes — they just request a different mechanism.

This matters because NIST has set a 2030 deadline for migrating federal systems to post-quantum cryptography. The cryptographic infrastructure stack — HSMs, key stores, PKCS#11 libraries — needs to support PQC before applications can adopt it. We're one of the first PKCS#11 implementations to ship this.

## What we didn't do

We didn't write a PKCS#11 wrapper around OpenSSL or aws-lc-rs. Craton HSM is a ground-up implementation of HSM logic in Rust. Session management, object storage, PIN handling, key lifecycle, audit logging — all written in Rust, all benefiting from the ownership model.

We also didn't pursue FIPS 140-3 certification. The codebase implements the technical requirements — 17 power-on self-tests, HMAC_DRBG with prediction resistance, SP 800-57 key lifecycle, pairwise consistency tests — but CMVP validation is a $50K-$200K process that takes 6-12 months. For a FIPS-validated backend, we offer an aws-lc-rs integration in a separate repository.

We didn't try to replace hardware HSMs. Craton HSM is a software token. It's for the same use cases as SoftHSMv2: development, testing, DNSSEC, software-only deployments, and environments where hardware HSMs are too expensive or impractical.

## Current state

Craton HSM v0.9.1 ships with:

- 70+ PKCS#11 C ABI functions (full v3.0 coverage)
- 617+ tests across unit, integration, interop, and conformance suites
- Verified interoperability with Java SunPKCS11 and OpenSSL 3.x
- 5 cargo-fuzz targets covering C ABI, crypto operations, sessions, attributes, and buffer handling
- Multi-platform CI: Linux, macOS, Windows
- Encrypted persistent storage (redb + AES-256-GCM)
- gRPC daemon with mutual TLS
- Admin CLI for token management and diagnostics
- Docker multi-stage build + Helm chart

It's Apache-2.0 licensed and available at [github.com/craton-co/craton-hsm-core](https://github.com/craton-co/craton-hsm-core).

## What's next

We're working on:
- Perfomance optimization to reach SoftHSMv2 levels
- Stabilizing PQC crate dependencies as they reach 1.0
- A SoftHSMv2 migration tool for importing existing key databases
- Distribution packages (deb, rpm, apk, Homebrew)
- A comprehensive third-party security audit

If you use SoftHSMv2 or work with PKCS#11, we'd love your feedback. File issues, try it with your stack, tell us what breaks.

---

*Craton HSM is built by [Craton Software Company](https://github.com/craton-co) — open-source cryptographic infrastructure in Rust.*

*[GitHub](https://github.com/craton-co/craton-hsm-core) | [Sponsor](https://github.com/sponsors/craton-co)*
