---
title: Summary of MODBUS Protocol Specification for AI Implementation
Summary: Comprehensive technical reference for MODBUS protocol optimized for AI implementations, covering protocol structure, function codes, and transport variants.
Sources:
  - raw/MODBUS/MODBUS.md
Categories:
  - specifications
  - reference
  - ai-implementation
type: summary
date-created: 2026-04-18T12:00:00+03:00
last-updated: 2026-04-18T14:43:52+03:00
---

This document provides a comprehensive technical reference for the MODBUS protocol, specifically optimized for AI agents generating MODBUS client/server implementations. It emphasizes precision, byte-level specifications, and concrete examples.

## Protocol Overview

MODBUS is an application-layer messaging protocol (OSI Layer 7) providing client/server communication between devices. Originally developed by Modicon in 1979 for programmable logic controllers, it remains the de facto standard for industrial device communication (source: [MODBUS.md](/raw/MODBUS/MODBUS.md)).

### Key Characteristics

| Property | Value |
|----------|-------|
| Architecture | Client/Server (Master/Slave) |
| Data encoding | Big-endian |
| Max PDU size | 253 bytes |
| TCP port | 502 (standard), 802 (secure/TLS) |
| Byte order | Most significant byte first |

## Data Model

MODBUS defines four primary data tables. Devices may implement these as separate memory areas or as overlapping views of the same memory:

| Table | Object Type | Size | Access | Typical Use |
|-------|-------------|------|--------|-------------|
| [Coils](/wiki/concepts/coils.md) | Bit | 1 bit | Read/Write | Digital outputs, relays |
| [Discrete Inputs](/wiki/concepts/discrete-inputs.md) | Bit | 1 bit | Read-only | Digital inputs, switches |
| [Holding Registers](/wiki/concepts/holding-registers.md) | Word | 16 bits | Read/Write | Configuration, setpoints |
| [Input Registers](/wiki/concepts/input-registers.md) | Word | 16 bits | Read-only | Measurements, status |

### Addressing

**CRITICAL: PDU addresses are 0-based. Documentation often uses 1-based numbering.**

| Data Model Reference | PDU Address | Wire Value |
|---------------------|-------------|------------|
| Register 1 | 0 | 0x0000 |
| Register 100 | 99 | 0x0063 |
| Register 40001 | 0 | 0x0000 (Holding Register context) |

The "40001" notation is a legacy convention where:
- 0xxxx = Coils
- 1xxxx = Discrete Inputs
- 3xxxx = Input Registers
- 4xxxx = Holding Registers

For implementation: Always use 0-based addresses in PDUs.

## Transport Variants

MODBUS operates over multiple transport layers:

| Variant | Medium | Error Check | Framing |
|---------|--------|-------------|---------|
| [MODBUS TCP](/wiki/concepts/modbus-tcp.md) | Ethernet | TCP/IP | MBAP header (7 bytes) |
| [MODBUS RTU](/wiki/concepts/modbus-rtu.md) | Serial (RS-485/232) | CRC-16 | Silent intervals |
| [MODBUS ASCII](/wiki/concepts/modbus-ascii.md) | Serial | LRC | Start ':' / End CR-LF |
| [MODBUS TCP Security](/wiki/concepts/modbus-tcp-security.md) | Ethernet + TLS | TCP/IP + TLS | MBAP over TLS |

## Protocol Data Unit (PDU)

The PDU is transport-independent and has a maximum size of 253 bytes:

**Request PDU:** `[Function Code: 1 byte][Request Data: 0-252 bytes]`

**Response PDU:** `[Function Code: 1 byte][Response Data: 0-252 bytes]`

**Exception Response PDU:** `[Function Code + 0x80: 1 byte][Exception Code: 1 byte]`

When an error occurs, the server returns the function code with bit 7 set (value + 0x80).

## Function Codes

The document details all major function codes including:
- Function 0x01: Read Coils (up to 2000 bits)
- Function 0x02: Read Discrete Inputs (up to 2000 bits)
- Function 0x03: Read Holding Registers (up to 125 registers)
- Function 0x04: Read Input Registers (up to 125 registers)
- Function 0x05: Write Single Coil
- Function 0x06: Write Single Register
- Function 0x0F: Write Multiple Coils (up to 1968 bits)
- Function 0x10: Write Multiple Registers (up to 123 registers)
- Function 0x16: Mask Write Register
- Function 0x17: Read/Write Multiple Registers
- Function 0x2B: Encapsulated Interface Transport

See [Function Codes](/wiki/concepts/function-codes.md) for detailed specifications.

## Exception Handling

MODBUS defines standard exception codes:

| Code | Hex | Name | Description |
|------|-----|------|-------------|
| 01 | 0x01 | ILLEGAL FUNCTION | Function code not supported by device |
| 02 | 0x02 | ILLEGAL DATA ADDRESS | Address not valid (out of range) |
| 03 | 0x03 | ILLEGAL DATA VALUE | Value not valid for request |
| 04 | 0x04 | SERVER DEVICE FAILURE | Unrecoverable error during execution |
| 05 | 0x05 | ACKNOWLEDGE | Request accepted, long processing time |
| 06 | 0x06 | SERVER DEVICE BUSY | Device busy; client should retry |
| 08 | 0x08 | MEMORY PARITY ERROR | Extended file area parity check failed |
| 10 | 0x0A | GATEWAY PATH UNAVAILABLE | Gateway misconfigured or overloaded |
| 11 | 0x0B | GATEWAY TARGET NO RESPONSE | Target device failed to respond |

## Implementation Notes

### Data Encoding

All multi-byte values use **big-endian** (network byte order):

```
Value: 0x1234
Transmitted: [0x12] [0x34]
             MSB    LSB
```

For 32-bit values across two registers, verify device documentation for word order.

### Common Pitfalls

| Pitfall | Solution |
|---------|----------|
| Using 1-based addresses | Always use 0-based in PDU; convert from documentation |
| Wrong byte order | Use big-endian for all multi-byte fields |
| CRC byte order (RTU) | CRC is little-endian (LSB first), unlike all other fields |
| Incorrect Length field | Length = bytes after Length field (Unit ID + PDU) |
| Batch requests per TCP frame | Send one ADU per TCP send; don't concatenate |
| Ignoring Transaction ID | Must match response to request in async scenarios |
| Coil value encoding | Only 0xFF00 (ON) and 0x0000 (OFF) are valid for 0x05 |

## Related pages

- [MODBUS TCP](/wiki/concepts/modbus-tcp.md)
- [MODBUS RTU](/wiki/concepts/modbus-rtu.md)
- [MODBUS ASCII](/wiki/concepts/modbus-ascii.md)
- [MODBUS TCP Security](/wiki/concepts/modbus-tcp-security.md)
- [Function Codes](/wiki/concepts/function-codes.md)
- [Coils](/wiki/concepts/coils.md)
- [Discrete Inputs](/wiki/concepts/discrete-inputs.md)
- [Holding Registers](/wiki/concepts/holding-registers.md)
- [Input Registers](/wiki/concepts/input-registers.md)

## Backlinks

------------

MODBUS is a trademark of the Modbus Organization, Inc.

All information on this page is derived by summarizing and analyzing solely the MODBUS specifications published by the Modbus Organization, Inc.  The specifications are copyright by the Modbus Organization.

By deriving this information solely from the specifications we hope to stay true to the specification.

No infringement is intended.

- [MODBUS Protocol](/wiki/concepts/modbus.md)
