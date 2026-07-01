---
title: What is MODBUS RTU-over-TCP
Summary: Explanation of the non-standard "RTU-over-TCP" variant that wraps a complete MODBUS RTU serial frame (including its CRC-16) inside a TCP connection without the standard MBAP header, and how it differs from official MODBUS TCP and MODBUS RTU.
Sources:
  - raw/MODBUS/MODBUS.md
  - raw/MODBUS/messagingimplementationguide.md
  - raw/MODBUS/modbusoverserial.md
  - raw/MODBUS/modbusprotocolspecification.md
Categories:
  - protocol-variants
  - tcp
  - rtu
  - interoperability
type: answer
date-created: 2026-07-01T09:13:34+03:00
last-updated: 2026-07-01T09:13:34+03:00
---

## Important: This Is Not an Official MODBUS Variant

"MODBUS RTU-over-TCP" (also called "MODBUS RTU/TCP" or "encapsulated RTU") is **not defined in any of the official MODBUS Organization specifications**. The official specifications only describe:

- **MODBUS TCP** — uses a 7-byte MBAP header, no CRC (source: [messagingimplementationguide.md](/raw/MODBUS/messagingimplementationguide.md))
- **MODBUS RTU** — serial line, 1-byte address, CRC-16 error check (source: [modbusoverserial.md](/raw/MODBUS/modbusoverserial.md))
- **MODBUS ASCII** — serial line, ASCII hex encoding, LRC error check (source: [modbusoverserial.md](/raw/MODBUS/modbusoverserial.md))
- **MODBUS/TCP Security** — MBAP encapsulated within TLS (source: [MODBUS.md](/raw/MODBUS/MODBUS.md))

Because RTU-over-TCP is a vendor-introduced convention rather than a specification, the descriptions below explain how it works in practice and contrast it with the official variants documented in the specs. Any statement not traceable to the specifications is marked as *(convention, not in specs)*.

## What It Is

RTU-over-TCP takes a **complete MODBUS RTU serial frame** — including its trailing **CRC-16** — and sends it as the payload of a TCP/IP connection **without** adding the standard MBAP header used by real MODBUS TCP *(convention, not in specs)*.

**Official MODBUS RTU frame** (source: [modbus-rtu.md](/wiki/concepts/modbus-rtu.md)):

```
[Address: 1][Function: 1][Data: N][CRC-16: 2]
```

**Official MODBUS TCP ADU** (source: [modbus-tcp.md](/wiki/concepts/modbus-tcp.md)):

```
[Transaction ID: 2][Protocol ID: 2][Length: 2][Unit ID: 1][Function: 1][Data: N]
|<---------------- MBAP Header (7 bytes) --------------->|<----- PDU ------->|
```

**RTU-over-TCP** *(convention, not in specs)* — the RTU frame is sent verbatim over TCP:

```
[Address: 1][Function: 1][Data: N][CRC-16: 2]
|<------------------ raw RTU frame over TCP ------------->|
```

Note there is **no MBAP header** and the **CRC-16 is retained**, even though TCP already provides its own error checking.

## How It Differs From the Official Variants

| Aspect | MODBUS TCP (official) | RTU-over-TCP (convention) | MODBUS RTU (official) |
|--------|----------------------|---------------------------|-----------------------|
| Transport | TCP/IP | TCP/IP | Serial RS-485/RS-232 |
| Typical port | 502 | Often 502 (vendor-chosen) | N/A |
| Header | 7-byte [MBAP Header](/wiki/concepts/mbap-header.md) | None (bare RTU frame) | 1-byte address |
| Error check | TCP checksum only | [CRC-16](/wiki/concepts/crc-16.md) present *and* TCP checksum | CRC-16 |
| Framing | MBAP Length field | Relies on CRC + timing/idle gaps | Silent intervals (t3.5) |
| Transaction ID | Yes (2 bytes) | No | No |
| Device addressing | Unit ID field in MBAP | Address byte at start of RTU frame | Address byte |

Sources: [MODBUS TCP Message Format](/wiki/answers/modbus-tcp-message-format.md), [modbus-rtu.md](/wiki/concepts/modbus-rtu.md), [mbap-header.md](/wiki/concepts/mbap-header.md).

## Why It Exists (Factual)

RTU-over-TCP is typically produced by simple **serial-to-Ethernet device servers** (sometimes called "serial tunnels" or "virtual serial port" adapters). These devices wrap the raw serial byte stream into a TCP connection **without understanding the MODBUS protocol**. Because they treat the serial data as opaque bytes, the original RTU frame — CRC and all — passes through unchanged *(convention, not in specs)*.

This contrasts with a true [MODBUS TCP-to-RTU gateway](/wiki/answers/modbus-tcp-to-rtu-gateway.md), which actively parses the MBAP header, strips it, computes a fresh CRC-16, and enforces RTU serial timing (source: [modbus-tcp-to-rtu-gateway.md](/wiki/answers/modbus-tcp-to-rtu-gateway.md)).

## Interoperability Consequences

Because RTU-over-TCP omits the MBAP header and keeps the CRC:

- A **standard MODBUS TCP client** expects a 7-byte MBAP header and **no** CRC. It cannot parse a bare RTU frame, so it will not interoperate with an RTU-over-TCP server.
- A **standard MODBUS TCP server** likewise will not understand incoming RTU-over-TCP frames.
- Both endpoints must be **explicitly configured for RTU-over-TCP mode**. Many MODBUS client libraries offer this as a separate transport option, distinct from "MODBUS TCP."

There is also **no Transaction ID**, so — like serial RTU — an RTU-over-TCP link generally handles **one outstanding request at a time** rather than the pipelined, transaction-matched requests that the MBAP Transaction ID enables on real MODBUS TCP (source: [mbap-header.md](/wiki/concepts/mbap-header.md)).

## How To Recognize It

When capturing traffic (for example with [Wireshark](/wiki/answers/wireshark-modbus-analysis.md)):

- **MODBUS TCP:** The first bytes are the MBAP header. The Protocol ID field is `0x0000`, and there is no CRC at the end.
- **RTU-over-TCP:** The TCP payload begins directly with the slave **address byte** and ends with a **2-byte CRC-16**. There is no MBAP header, and the bytes just after the address will not match an MBAP `[Protocol ID = 0x0000][Length]` pattern.

## Practical Guidance

- If you control both ends and can use standard MODBUS TCP, prefer it — it is specified, interoperable, and does not carry a redundant CRC.
- Use RTU-over-TCP only when a device or serial tunnel **requires** it, typically because a legacy RTU device is bridged onto Ethernet by a protocol-unaware serial server.
- Ensure your client library's transport setting matches: "MODBUS TCP" and "MODBUS RTU-over-TCP" are **not interchangeable**.

## Related Pages

- [MODBUS TCP](/wiki/concepts/modbus-tcp.md) - The official MBAP-based TCP variant
- [MODBUS RTU](/wiki/concepts/modbus-rtu.md) - The serial frame (with CRC-16) that RTU-over-TCP transports
- [MBAP Header](/wiki/concepts/mbap-header.md) - The header that RTU-over-TCP omits
- [CRC-16](/wiki/concepts/crc-16.md) - The error check that RTU-over-TCP retains
- [How MODBUS TCP-to-RTU Gateways Work](/wiki/answers/modbus-tcp-to-rtu-gateway.md) - The standard-compliant alternative
- [MODBUS TCP Message Format](/wiki/answers/modbus-tcp-message-format.md) - The official TCP frame layout
- [Using Wireshark for MODBUS TCP Analysis](/wiki/answers/wireshark-modbus-analysis.md) - Distinguishing the two on the wire

## Backlinks

- [How is MODBUS Handled Over UDP](/wiki/answers/modbus-over-udp.md)
- [MODBUS TCP Message Format](/wiki/answers/modbus-tcp-message-format.md)
- [MODBUS TCP](/wiki/concepts/modbus-tcp.md)
- [MODBUS RTU](/wiki/concepts/modbus-rtu.md)
- [MBAP Header](/wiki/concepts/mbap-header.md)
- [CRC-16](/wiki/concepts/crc-16.md)

------------

MODBUS is a trademark of the Modbus Organization, Inc.

All information on this page is derived by summarizing and analyzing solely the MODBUS specifications published by the Modbus Organization, Inc.  The specifications are copyright by the Modbus Organization.

By deriving this information solely from the specifications we hope to stay true to the specification.

No infringement is intended.
