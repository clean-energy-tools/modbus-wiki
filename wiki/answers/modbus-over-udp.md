---
title: How is MODBUS Handled Over UDP
Summary: Explains that UDP is not part of any official MODBUS specification, how vendor implementations carry the MBAP header and PDU inside UDP datagrams, and why UDP's lack of reliability, ordering, and error checking matters for a protocol that assumes TCP guarantees.
Sources:
  - raw/MODBUS/messagingimplementationguide.md
  - raw/MODBUS/MODBUS.md
  - raw/MODBUS/modbusprotocolspecification.md
Categories:
  - protocol-variants
  - tcp
  - udp
  - interoperability
type: answer
date-created: 2026-07-02T00:16:56+03:00
last-updated: 2026-07-02T00:16:56+03:00
---

## Important: UDP Is Not Part of the MODBUS Standard

UDP is **not defined in any of the official MODBUS Organization specifications**. A search of all source specifications in `raw/MODBUS/` finds no mention of "UDP", "datagram", or "connectionless". The official IP transport specification is titled *MODBUS Messaging on **TCP/IP** Implementation Guide* and describes **TCP only** (source: [messagingimplementationguide.md](/raw/MODBUS/messagingimplementationguide.md)).

Key statements from the specification:

- "A MODBUS communication requires the establishment of a **TCP connection** between a client and a server" (source: [messagingimplementationguide.md](/raw/MODBUS/messagingimplementationguide.md:786)).
- "All MODBUS/TCP ADU are sent via **TCP** to registered port 502" (source: [messagingimplementationguide.md](/raw/MODBUS/messagingimplementationguide.md:442)).

The entire guide is built around TCP-specific mechanisms — connection establishment, connection pools, keep-alive, and TCP retransmission (source: [messagingimplementationguide.md](/raw/MODBUS/messagingimplementationguide.md:776)).

Because UDP is not in the specifications, everything below about "MODBUS over UDP" describes a **vendor/implementation convention**, not a standard. Non-spec statements are marked *(convention, not in specs)*.

## A Note on Port 502

Registered port 502 is assigned by IANA for both TCP and UDP. However, the MODBUS specification only ever describes port 502 as a **TCP listening port** (source: [messagingimplementationguide.md](/raw/MODBUS/messagingimplementationguide.md:731)). The existence of a UDP port assignment does not make UDP a defined MODBUS transport.

## How UDP Is Handled in Practice (Convention)

Where vendors offer "MODBUS over UDP", it typically works as follows *(convention, not in specs)*:

- The **same ADU as MODBUS TCP** — a 7-byte [MBAP Header](/wiki/concepts/mbap-header.md) followed by the MODBUS PDU — is placed inside a **single UDP datagram**, usually targeting port 502.
- There is **no connection** to establish, keep alive, or pool, because UDP is connectionless. Each request and each response is an independent datagram.
- The **MBAP Transaction ID** becomes essential for pairing a response with its request, since UDP provides no ordering or delivery guarantees (source: [mbap-header.md](/wiki/concepts/mbap-header.md)).

**Datagram payload (same bytes as MODBUS TCP):**

```
[Transaction ID: 2][Protocol ID: 2][Length: 2][Unit ID: 1][Function: 1][Data: N]
|<---------------- MBAP Header (7 bytes) --------------->|<----- PDU ------->|
```

## Why This Matters: MODBUS Relies on TCP Guarantees

MODBUS/TCP deliberately carries **no application-layer error check** (no CRC or LRC), because it assumes TCP guarantees reliable, ordered, error-checked delivery (source: [MODBUS TCP Message Format](/wiki/answers/modbus-tcp-message-format.md)). UDP provides none of these guarantees, so a UDP implementation must compensate in the application layer.

| Guarantee | TCP (standard) | UDP (convention) |
|-----------|----------------|------------------|
| Reliable delivery | Yes | No — datagrams can be lost |
| Ordering | Yes | No — datagrams can arrive out of order |
| Duplicate protection | Yes | No |
| Error detection | TCP checksum + retransmit | UDP checksum only, no retransmit |
| Connection state | Yes (established connection) | None (connectionless) |

**Consequences for a UDP implementation** *(convention, not in specs)*:

- It must add its own **timeout and retry** logic, because lost datagrams are never retransmitted by UDP.
- It must handle **duplicate and out-of-order** responses, using the MBAP Transaction ID to match and discard as needed.
- Because MODBUS itself adds no frame integrity check over IP, corruption beyond the UDP checksum's coverage is not detected at the application layer.

Note that the related **RTU-over-TCP** convention keeps the RTU CRC-16, but that runs over **TCP**, not UDP — see [What is MODBUS RTU-over-TCP](/wiki/answers/modbus-rtu-over-tcp.md).

## Interoperability

Because UDP is not standardized for MODBUS:

- A standard **MODBUS TCP** client or server will **not** interoperate with a UDP implementation. TCP endpoints expect a connection-oriented byte stream, not datagrams.
- Both endpoints must be **explicitly configured for the same UDP convention**. Many MODBUS libraries expose "MODBUS UDP" as a separate transport option, distinct from "MODBUS TCP".

## Practical Guidance

- Prefer standard **MODBUS TCP** whenever possible — it is specified, interoperable, and provides the delivery guarantees MODBUS depends on.
- Use MODBUS over UDP only when a specific device or application **requires** it, and confirm both ends use the same convention.
- If you must use UDP, implement application-level **timeouts, retries, and duplicate handling**, and rely on the MBAP Transaction ID for request/response matching.

## Related Pages

- [MODBUS TCP](/wiki/concepts/modbus-tcp.md) - The official, TCP-based IP transport
- [MBAP Header](/wiki/concepts/mbap-header.md) - The header a UDP datagram would carry
- [TCP Connection Management](/wiki/concepts/tcp-connection-management.md) - The connection model UDP lacks
- [MODBUS TCP Message Format](/wiki/answers/modbus-tcp-message-format.md) - Why MODBUS/TCP omits a CRC
- [What is MODBUS RTU-over-TCP](/wiki/answers/modbus-rtu-over-tcp.md) - Another non-standard IP transport convention
- [MODBUS Broadcast](/wiki/answers/modbus-broadcast.md) - Notes UDP multicast as non-standard

## Backlinks

- [MODBUS Broadcast](/wiki/answers/modbus-broadcast.md)
- [MODBUS TCP Message Format](/wiki/answers/modbus-tcp-message-format.md)
- [What is MODBUS RTU-over-TCP](/wiki/answers/modbus-rtu-over-tcp.md)
- [MODBUS TCP](/wiki/concepts/modbus-tcp.md)
- [MBAP Header](/wiki/concepts/mbap-header.md)
- [TCP Connection Management](/wiki/concepts/tcp-connection-management.md)

------------

MODBUS is a trademark of the Modbus Organization, Inc.

All information on this page is derived by summarizing and analyzing solely the MODBUS specifications published by the Modbus Organization, Inc.  The specifications are copyright by the Modbus Organization.

By deriving this information solely from the specifications we hope to stay true to the specification.

No infringement is intended.
