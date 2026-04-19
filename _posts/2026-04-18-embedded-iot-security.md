# Zero-Allocation Embedded Security: Protecting IoT Devices with 256 KB of Flash

*How Craton Shield Embedded brings defense-in-depth to resource-constrained sensors and actuators*

---

The EU Cyber Resilience Act is coming. By 2027, every product with a digital element sold in the European Union must meet mandatory cybersecurity requirements — including your $3 sensor node, your industrial gateway, and your smart home thermostat. Non-compliance means fines up to EUR 15 million or 2.5% of global turnover.

For embedded developers shipping products on Cortex-M3 microcontrollers with 256 KB of flash and 64 KB of RAM, the question is urgent: how do you add meaningful security to a device that can barely fit its own application code?

We built Craton Shield Embedded to solve exactly this problem. It's a `#![no_std]`, zero-allocation Rust security stack that delivers intrusion detection, firewalling, encrypted OTA updates, and tamper-evident logging — all within the memory footprint of a typical embedded application.

## The Embedded Security Dilemma

Traditional security solutions were designed for servers and desktops. They assume gigabytes of RAM, fast CPUs, and operating system services like dynamic memory allocation, threading, and network stacks. None of these exist on a bare-metal Cortex-M microcontroller.

The result is that most embedded devices ship with minimal or no security:
- No firmware signature verification (OTA updates accepted from anyone)
- No network filtering (all inbound traffic processed)
- No intrusion detection (anomalous behavior goes unnoticed)
- No audit logging (no forensic evidence after an incident)

This isn't because embedded developers don't care about security. It's because the tools don't exist — or didn't, until now.

## What Fits in 256 KB

Craton Shield Embedded is composed of modular crates. You include only what you need:

| Crate | Flash Size | RAM Size | What It Does |
|-------|-----------|----------|-------------|
| vs-netfw (firewall) | ~12 KB | ~4 KB | Stateful packet filtering, default-deny |
| vs-ota-validator | ~18 KB | ~8 KB | TUF-based firmware signature verification |
| vs-crypto (software) | ~15 KB | ~2 KB | SHA-256, HMAC, ECDSA verification |
| vs-event-logger | ~8 KB | ~6 KB | HMAC-chained tamper-evident audit log |
| vs-anomaly | ~6 KB | ~2 KB | EWMA-based behavioral anomaly detection |
| vs-integrity | ~4 KB | ~1 KB | Runtime memory region integrity checks |
| vs-policy-engine | ~10 KB | ~3 KB | XACML-lite access control rules |

**Minimal security profile (firewall + OTA + crypto + logging): ~53 KB flash, ~20 KB RAM**

That leaves over 200 KB of flash and 44 KB of RAM for your application on a typical Cortex-M3 device. On larger Cortex-M4/M7 devices, the overhead is negligible.

## Zero Heap Allocation: Why It Matters

Every crate in Craton Shield compiles with `#![no_std]` and uses zero dynamic memory allocation. No `Vec`, no `Box`, no `String`, no `alloc` crate. Every data structure is a fixed-size array or a stack-allocated struct.

This isn't just a design preference — it's a safety and reliability requirement:

**1. No fragmentation.** Heap fragmentation is the silent killer of long-running embedded systems. A device that works fine for days can suddenly fail to allocate memory after weeks of fragmentation. With zero heap allocation, memory usage is constant and predictable from the first second to the millionth hour.

**2. No out-of-memory panics.** If your security layer can fail due to OOM, it will fail at exactly the wrong time — during an attack, when alert volume spikes. Fixed-size buffers mean the security layer either works at initialization or it doesn't compile. There is no runtime failure mode.

**3. Deterministic WCET.** Worst-Case Execution Time analysis requires bounded execution paths. Dynamic allocation introduces unbounded allocation and deallocation times (depending on heap state). Zero allocation means every code path has a deterministic, analyzable execution time.

**4. Certifiability.** Safety standards like IEC 61508 (industrial), IEC 62304 (medical), and DO-178C (avionics) heavily scrutinize dynamic memory allocation. Many certification bodies require proof that OOM conditions are handled or — preferably — that they cannot occur. Zero allocation satisfies the strictest interpretation.

## Protocol Support for IoT

Craton Shield Embedded monitors the protocols your IoT devices actually use:

**MQTT Monitoring:**
- Topic allowlist enforcement (prevent subscription to unauthorized topics)
- Payload size validation (detect oversized messages that could be buffer overflow attempts)
- Client ID verification (detect impersonation)
- QoS downgrade detection

**CoAP Monitoring:**
- Request method filtering (only allow expected GET/PUT/POST operations)
- Resource path allowlist
- Block transfer validation
- Observe notification rate limiting

**BLE Monitoring:**
- GATT characteristic access control
- Pairing mode enforcement (prevent downgrade attacks)
- Connection parameter validation
- Advertising data integrity checks

**Generic TCP/UDP Firewall:**
- Source/destination IP and port filtering
- Connection tracking for stateful inspection
- Rate limiting per source
- Default-deny with explicit allow rules

## Securing OTA Updates on Constrained Devices

Over-the-air firmware updates are the most critical attack surface for embedded devices. A compromised update mechanism means the attacker owns the device permanently.

Craton Shield's OTA validator (`vs-ota-validator`) implements the TUF (The Update Framework) specification adapted for constrained environments:

```
Firmware Update Flow:

  1. Receive update manifest ──> Verify threshold-of-N
     (signed metadata)          ECDSA signatures
                                    │
  2. Check version counter ──> Reject if <= current
     (rollback protection)      (monotonic counter in
                                persistent storage)
                                    │
  3. Download firmware image ──> Verify SHA-256 hash
     (chunked transfer)         matches manifest
                                    │
  4. Verify image signature ──> Reject if signature
     (ECDSA P-256)              invalid
                                    │
  5. Write to secondary slot ──> Hardware-specific
     (A/B partition scheme)     flash write
                                    │
  6. Set boot flag ──────────> Reboot into new firmware
                                with automatic rollback
                                if boot fails
```

**Key properties:**
- **Threshold signatures:** Require M-of-N valid signatures (e.g., 2-of-3). A single compromised signing key cannot push a malicious update.
- **Rollback protection:** Monotonic version counter prevents downgrade attacks. An attacker cannot revert the device to an older, vulnerable firmware version.
- **Fail-safe boot:** If the new firmware fails to boot, the device automatically reverts to the previous working image. The update process cannot brick the device.

All of this runs in ~18 KB of flash and ~8 KB of RAM.

## Tamper-Evident Logging for Forensics

When a security incident occurs, you need to know what happened. The `vs-event-logger` provides a tamper-evident audit log using HMAC chaining:

```
Entry 0: [timestamp | event_type | data | HMAC(key, entry_0)]
Entry 1: [timestamp | event_type | data | HMAC(key, entry_0_mac | entry_1)]
Entry 2: [timestamp | event_type | data | HMAC(key, entry_1_mac | entry_2)]
    ...
```

Each entry's HMAC covers the previous entry's HMAC, creating a hash chain. If an attacker modifies any historical entry, the chain breaks and the tampering is immediately detectable during the next verification pass.

**Performance:** Appending a log entry takes 108 nanoseconds, including HMAC computation. At this speed, you can log every network packet, every sensor reading, every actuator command — without impacting your application's timing budget.

**Storage:** The log uses a configurable ring buffer in flash or EEPROM. When the buffer is full, the oldest entries are overwritten (with the chain's integrity preserved). On devices with as little as 4 KB of dedicated log storage, you can retain thousands of events.

## Real-World Deployment: Smart Agriculture Sensor

Consider a typical IoT deployment: a soil moisture sensor network for precision agriculture. Each node has:
- STM32L4 microcontroller (Cortex-M4, 256 KB flash, 64 KB RAM)
- LoRaWAN or NB-IoT radio for uplink
- MQTT-SN for message transport
- OTA update capability via MQTT

**Without Craton Shield:**
- Firmware updates accepted without signature verification
- Any MQTT client can publish to any topic
- No detection of anomalous sensor readings (compromised node injection)
- No forensic evidence if a node is compromised

**With Craton Shield Embedded (53 KB flash overhead):**
- Firmware updates require 2-of-3 ECDSA signatures with rollback protection
- MQTT topic allowlist restricts each node to its own publish/subscribe topics
- EWMA anomaly detection flags statistically unusual sensor readings
- HMAC-chained log preserves forensic evidence across power cycles
- Default-deny firewall blocks all traffic except authorized MQTT broker

The security stack runs in the main loop alongside the application, adding less than 1 microsecond to each iteration. The sensor's 10-second measurement cycle is completely unaffected.

## Getting Started

Add Craton Shield Embedded to your project:

```toml
# Cargo.toml
[dependencies]
vs-netfw = "1.0"
vs-ota-validator = "1.0"
vs-crypto = "1.0"
vs-event-logger = "1.0"
vs-anomaly = "1.0"
```

```rust
#![no_std]
#![no_main]

use vs_netfw::Firewall;
use vs_ota_validator::OtaValidator;
use vs_event_logger::EventLogger;

// Configure firewall: allow only MQTT broker on port 1883
let mut fw = Firewall::new();
fw.add_rule(Rule::allow(Protocol::Tcp, broker_ip, 1883));
// Everything else is denied by default

// Configure OTA: require 2-of-3 signatures
let ota = OtaValidator::new(&[key_1, key_2, key_3], threshold: 2);

// Configure logging
let mut logger = EventLogger::new(&hmac_key, &mut flash_storage);

// In your network receive handler:
fn on_packet(packet: &[u8]) {
    if !fw.evaluate(packet) {
        logger.append(Event::Blocked(packet));
        return; // dropped in ~9 ns
    }
    // Process allowed packet...
}
```

## EU Cyber Resilience Act Readiness

The CRA requires manufacturers to:

| CRA Requirement | Craton Shield Feature |
|----------------|----------------------|
| Deliver products without known exploitable vulnerabilities | Memory-safe Rust eliminates buffer overflows, use-after-free, data races |
| Implement access control | vs-policy-engine (XACML-lite rules) + vs-netfw (default-deny firewall) |
| Protect stored and transmitted data | vs-crypto (SHA-256, HMAC, ECDSA) + Enterprise HSM support |
| Minimize attack surface | Default-deny firewall + protocol allowlists |
| Enable security updates | vs-ota-validator (TUF/Uptane with rollback protection) |
| Log security-relevant events | vs-event-logger (HMAC-chained tamper-evident audit log) |
| Report vulnerabilities and incidents | vs-report-iec62443 (compliance gap analysis reports) |

Craton Shield doesn't guarantee CRA compliance — that requires organizational processes beyond any single tool. But it provides the technical foundation that makes compliance achievable on resource-constrained devices.

## The Rust Advantage for Embedded Security

We chose Rust not because it's trendy, but because it solves the fundamental contradiction of embedded security: you need a security layer that is itself secure.

A CAN bus IDS written in C can detect injection attacks — but it can also be exploited via buffer overflow in its own packet parser. A firewall written in C++ can block unauthorized traffic — but a use-after-free in its connection tracker can be used to bypass it.

Rust eliminates these attack vectors at the language level. The compiler enforces memory safety, thread safety, and null safety. The resulting binary is as fast as C (often faster, due to aliasing guarantees that enable better optimization), but it cannot be exploited via the memory corruption vulnerabilities that account for 70% of CVEs in systems code.

For embedded security specifically, this means:
- Your IDS cannot be bypassed via buffer overflow
- Your firewall cannot be crashed via malformed input
- Your OTA validator cannot be tricked via memory corruption
- Your audit log cannot be corrupted via concurrent access bugs

The security layer is as reliable as the hardware it runs on.

---

*Craton Shield Embedded is open source under Apache-2.0. Enterprise features (HSM-backed crypto, fleet monitoring, SIEM integration) are available under BSL-1.1 via Shield Enterprise.*

*Join the community: GitHub | Discord | craton.com.ar*
