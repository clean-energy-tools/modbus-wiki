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

This page summarizes the MODBUS specification document. It provides detailed information for building MODBUS software, with precise technical details and examples (source: [MODBUS.md](/raw/MODBUS/MODBUS.md)).

## What MODBUS Is

MODBUS is a messaging system that lets industrial devices talk to each other. Created by Modicon in 1979 for factory controllers, it's still the most common way industrial devices communicate today (source: [MODBUS.md](/raw/MODBUS/MODBUS.md)).

### Main Features

| Feature | Details |
|----------|-------|
| How it works | One device asks (client), others answer (server) |
| Number format | Big-endian (high byte first) |
| Max message size | 253 bytes for the core message |
| Port numbers | 502 (normal), 802 (encrypted) |
| Byte order | Most important byte sent first |

## MODBUS Data Types

MODBUS organizes data into four types. Devices can store these separately or use the same memory for multiple types:

| Data Type | Size | Can You Change It? | What It's For |
|-------|-------------|------|--------|
| [Coils](/wiki/concepts/coils.md) | 1 bit (ON/OFF) | Yes (Read/Write) | Controlling relays and outputs |
| [Discrete Inputs](/wiki/concepts/discrete-inputs.md) | 1 bit (ON/OFF) | No (Read-only) | Reading switches and inputs |
| [Holding Registers](/wiki/concepts/holding-registers.md) | 16 bits (number) | Yes (Read/Write) | Settings and setpoints |
| [Input Registers](/wiki/concepts/input-registers.md) | 16 bits (number) | No (Read-only) | Measurements and status |

### How Addressing Works

**IMPORTANT: Addresses in messages start at 0, but documentation often counts from 1.**

| How Documentation Labels It | Actual Address in Message | Sent as Hex |
|---------------------|-------------|------------|
| Register 1 | 0 | 0x0000 |
| Register 100 | 99 | 0x0063 |
| Register 40001 | 0 | 0x0000 (in Holding Register context) |

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

- [Coils](/wiki/concepts/coils.md)
- [Discrete Inputs](/wiki/concepts/discrete-inputs.md)
- [Function Codes](/wiki/concepts/function-codes.md)
- [Holding Registers](/wiki/concepts/holding-registers.md)
- [Input Registers](/wiki/concepts/input-registers.md)
- [MODBUS](/wiki/concepts/modbus.md)
- [MODBUS ASCII](/wiki/concepts/modbus-ascii.md)
- [MODBUS Data Type Mapping](/wiki/concepts/modbus-data-type-mapping.md)
- [MODBUS RTU](/wiki/concepts/modbus-rtu.md)
- [MODBUS TCP](/wiki/concepts/modbus-tcp.md)
- [MODBUS TCP Security](/wiki/concepts/modbus-tcp-security.md)
- [MODBUS Usage and Applications](/wiki/concepts/modbus-usage-and-applications.md)

------------

MODBUS is a trademark of the Modbus Organization, Inc.

All information on this page is derived by summarizing and analyzing solely the MODBUS specifications published by the Modbus Organization, Inc.  The specifications are copyright by the Modbus Organization.

By deriving this information solely from the specifications we hope to stay true to the specification.

No infringement is intended.
