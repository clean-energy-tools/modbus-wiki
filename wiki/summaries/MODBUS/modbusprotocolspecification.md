---
title: MODBUS Application Protocol Specification
Summary: Official MODBUS Application Protocol specification describing the general communication model, data model, function codes, and exception handling for MODBUS protocol.
Sources:
  - raw/MODBUS/modbusprotocolspecification.md
Categories:
  - specifications
  - protocol-reference
  - data-model
type: summary
date-created: 2026-04-18T12:00:00+03:00
last-updated: 2026-04-18T14:43:52+03:00
---

This document is the official MODBUS Application Protocol specification that describes the general communication model, data model, function codes, and exception handling mechanisms used in MODBUS protocol implementations (source: [modbusprotocolspecification.md](/raw/MODBUS/modbusprotocolspecification.md)).

## Protocol Overview

MODBUS is an application layer messaging protocol that provides client/server communication between devices connected on different types of buses or networks. The protocol defines a simple Protocol Data Unit (PDU) that is independent of the underlying communication layers (source: [modbusprotocolspecification.md](/raw/MODBUS/modbusprotocolspecification.md)).

### Communication Architecture

MODBUS operates in a client/server model where:
- The client initiates MODBUS transactions
- The server processes requests and sends responses
- Communication is unicast (directed to specific device) or broadcast (all devices on network)

The protocol maps to different underlying layers by adding additional fields to the Application Data Unit (ADU):
- TCP/IP: MBAP header added
- Serial line: Address and error checking added

## Data Model

MODBUS defines a standardized data model with four primary tables:

| Table | Object Type | Type of Access | Typical Use |
|-------|-------------|---------------|-------------|
| [coils](/wiki/concepts/coils.md) | Single bit | Read-Write | Digital outputs, control relays |
| [discrete-inputs](/wiki/concepts/discrete-inputs.md) | Single bit | Read-Only | Digital inputs, limit switches |
| [input-registers](/wiki/concepts/input-registers.md) | 16-bit word | Read-Only | Analog inputs, sensor readings |
| [holding-registers](/wiki/concepts/holding-registers.md) | 16-bit word | Read-Write | Setpoints, configuration values |

### Memory Organization

The specification allows two implementation approaches:
1. **Separate blocks** - Four distinct memory areas
2. **Single block** - All data in one memory area with different addressing

Devices may implement either approach, but the protocol interface remains consistent.

## Function Codes

The specification defines standard function codes organized into categories:

### Public Function Codes

These are standardized codes that must be implemented according to specification:

| Code | Name | Description |
|------|------|-------------|
| 01 | Read Coils | Read status of coils (read-write bits) |
| 02 | Read Discrete Inputs | Read status of discrete inputs (read-only bits) |
| 03 | Read Holding Registers | Read holding register values |
| 04 | Read Input Registers | Read input register values |
| 05 | Write Single Coil | Write single coil to ON or OFF |
| 06 | Write Single Register | Write single register value |
| 07 | Read Exception Status | Read internal exception status |
| 08 | Diagnostics | Diagnostics and communications testing |
| 11 | Get Comm Event Counter | Get event counter |
| 12 | Get Comm Event Log | Get event log |
| 15 | Write Multiple Coils | Write multiple coils |
| 16 | Write Multiple Registers | Write multiple registers |
| 17 | Report Slave ID | Report identification of remote slave |
| 20 | Read File Record | Read file record |
| 21 | Write File Record | Write file record |
| 22 | Mask Write Register | Mask write register |
| 23 | Read/Write Multiple Registers | Read/write multiple registers |
| 24 | Read FIFO Queue | Read FIFO queue |
| 43 | Encapsulated Interface Transport | MEI type transport |

See [function-codes](/wiki/concepts/function-codes.md) for detailed specifications.

## Protocol Data Unit (PDU)

### PDU Structure

The MODBUS PDU consists of:
- **Function Code** (1 byte) - Specifies the operation to perform
- **Data** (0-252 bytes) - Parameters or data for the function

### ADU Mapping

Different transport layers encapsulate the PDU:

**MODBUS Serial Line ADU:**
```
[Address: 1][Function Code: 1][Data: N][Error Check: 1-2]
```

**MODBUS TCP ADU:**
```
[MBAP Header: 7][Function Code: 1][Data: N]
```

## Exception Handling

### Exception Response Format

When a server cannot process a request, it returns an exception response:

```
[Function Code + 0x80: 1 byte][Exception Code: 1 byte]
```

The function code in the response has the most significant bit set (function code | 0x80).

### Exception Codes

| Code | Name | Description |
|------|------|-------------|
| 01 | Illegal Function | Function code received is not supported |
| 02 | Illegal Data Address | Data address in request is not valid |
| 03 | Illegal Data Value | Value in data field is not valid |
| 04 | Server Device Failure | Unrecoverable error while processing |
| 05 | Acknowledge | Request accepted but will take time to process |
| 06 | Server Device Busy | Server busy processing another command |
| 08 | Memory Parity Error | Parity error in memory file area |
| 10 | Gateway Path Unavailable | Gateway unable to process request |
| 11 | Gateway Target Device Failed | Target device failed to respond |

## Modbus Variants

The protocol specification serves as the foundation for multiple MODBUS variants:
- [modbus-tcp](/wiki/concepts/modbus-tcp.md) - MODBUS over TCP/IP
- [modbus-rtu](/wiki/concepts/modbus-rtu.md) - MODBUS over serial line in RTU mode
- [modbus-ascii](/wiki/concepts/modbus-ascii.md) - MODBUS over serial line in ASCII mode
- [modbus-tcp-security](/wiki/concepts/modbus-tcp-security.md) - Secure MODBUS over TLS

## Related pages

- [function-codes](/wiki/concepts/function-codes.md)
- [coils](/wiki/concepts/coils.md)
- [discrete-inputs](/wiki/concepts/discrete-inputs.md)
- [holding-registers](/wiki/concepts/holding-registers.md)
- [input-registers](/wiki/concepts/input-registers.md)
- [modbus-tcp](/wiki/concepts/modbus-tcp.md)
- [modbus-rtu](/wiki/concepts/modbus-rtu.md)
- [modbus-ascii](/wiki/concepts/modbus-ascii.md)
