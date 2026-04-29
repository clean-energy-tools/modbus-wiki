---
title: Summary of MODBUS Application Protocol Specification
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

This page summarizes the official MODBUS Application Protocol specification. It explains how MODBUS communication works, what data types exist, available operations, and how errors are handled (source: [modbusprotocolspecification.md](/raw/MODBUS/modbusprotocolspecification.md)).

## What This Specification Covers

MODBUS is a messaging system that lets devices talk to each other over different connection types. The core messages (PDU) work the same way whether you're using Ethernet, serial cables, or other connections (source: [modbusprotocolspecification.md](/raw/MODBUS/modbusprotocolspecification.md)).

### How Communication Works

MODBUS uses a question-and-answer pattern:
- The client asks questions and sends commands
- The server processes requests and sends back answers
- Messages go to one specific device (unicast) or all devices (broadcast)

Messages are wrapped differently depending on the connection:
- Ethernet (TCP/IP): Adds an MBAP header with routing info
- Serial cables: Adds device address and error checking

## MODBUS Data Types

MODBUS organizes data into four standard types:

| Data Type | Size | Can You Change It? | What It's For |
|-------|-------------|---------------|-------------|
| [Coils](/wiki/concepts/coils.md) | 1 bit (ON/OFF) | Yes (Read-Write) | Controlling outputs and relays |
| [Discrete Inputs](/wiki/concepts/discrete-inputs.md) | 1 bit (ON/OFF) | No (Read-Only) | Reading switches and sensors |
| [Input Registers](/wiki/concepts/input-registers.md) | 16 bits (number) | No (Read-Only) | Reading measurements |
| [Holding Registers](/wiki/concepts/holding-registers.md) | 16 bits (number) | Yes (Read-Write) | Storing settings and setpoints |

### How Devices Store This Data

Devices can organize this data in two ways:
1. **Separate areas** - Four different memory sections
2. **Single area** - All data in one place, accessed differently

Either way works - the messages sent and received look the same.

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

See [Function Codes](/wiki/concepts/function-codes.md) for detailed specifications.

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
- [MODBUS TCP](/wiki/concepts/modbus-tcp.md) - MODBUS over TCP/IP
- [MODBUS RTU](/wiki/concepts/modbus-rtu.md) - MODBUS over serial line in RTU mode
- [MODBUS ASCII](/wiki/concepts/modbus-ascii.md) - MODBUS over serial line in ASCII mode
- [MODBUS TCP Security](/wiki/concepts/modbus-tcp-security.md) - Secure MODBUS over TLS

## Related pages

- [Function Codes](/wiki/concepts/function-codes.md)
- [Coils](/wiki/concepts/coils.md)
- [Discrete Inputs](/wiki/concepts/discrete-inputs.md)
- [Holding Registers](/wiki/concepts/holding-registers.md)
- [Input Registers](/wiki/concepts/input-registers.md)
- [MODBUS TCP](/wiki/concepts/modbus-tcp.md)
- [MODBUS RTU](/wiki/concepts/modbus-rtu.md)
- [MODBUS ASCII](/wiki/concepts/modbus-ascii.md)

## Backlinks

- [Coils](/wiki/concepts/coils.md)
- [Discrete Inputs](/wiki/concepts/discrete-inputs.md)
- [Function Codes](/wiki/concepts/function-codes.md)
- [Holding Registers](/wiki/concepts/holding-registers.md)
- [Input Registers](/wiki/concepts/input-registers.md)
- [MODBUS ASCII](/wiki/concepts/modbus-ascii.md)
- [MODBUS Data Type Mapping](/wiki/concepts/modbus-data-type-mapping.md)
- [MODBUS RTU](/wiki/concepts/modbus-rtu.md)
- [MODBUS TCP](/wiki/concepts/modbus-tcp.md)
- [MODBUS TCP Security](/wiki/concepts/modbus-tcp-security.md)
- [MODBUS Usage and Applications](/wiki/concepts/modbus-usage-and-applications.md)
- [Summary of messagingimplementationguide](/wiki/summaries/MODBUS/messagingimplementationguide.md)

------------

MODBUS is a trademark of the Modbus Organization, Inc.

All information on this page is derived by summarizing and analyzing solely the MODBUS specifications published by the Modbus Organization, Inc.  The specifications are copyright by the Modbus Organization.

By deriving this information solely from the specifications we hope to stay true to the specification.

No infringement is intended.
