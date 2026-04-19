---
title: MODBUS Wiki Index
Summary: Table of contents for the MODBUS protocol wiki, containing links to all summaries and concept pages.
Sources:
  - raw/MODBUS/MODBUS.md
  - raw/MODBUS/modbusprotocolspecification.md
  - raw/MODBUS/modbusoverserial.md
  - raw/MODBUS/messagingimplementationguide.md
  - raw/MODBUS/modbussecurityprotocol.md
date-created: 2026-04-18T12:00:00+03:00
last-updated: 2026-04-18T14:43:24+03:00
---

Welcome to the MODBUS Protocol Wiki. This wiki contains organized information about the MODBUS protocol specifications, extracted from official MODBUS documentation.

## Document Summaries

Summaries of source documents in `raw/MODBUS/`:

| Document                                                                                                  | Description                                                                                                                      |
| --------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| [MODBUS Protocol Specification for AI Implementation](/wiki/summaries/MODBUS/MODBUS.md)                   | Comprehensive technical reference for MODBUS protocol optimized for AI implementations                                           |
| [MODBUS Application Protocol Specification](/wiki/summaries/MODBUS/modbusprotocolspecification.md)        | Official MODBUS Application Protocol specification describing communication model, data model, and function codes                |
| [MODBUS Over Serial Line Specification](/wiki/summaries/MODBUS/modbusoverserial.md)                       | MODBUS serial line specification covering master-slave protocol, RTU and ASCII modes, and RS485 physical layer                   |
| [MODBUS Messaging on TCP/IP Implementation Guide](/wiki/summaries/MODBUS/messagingimplementationguide.md) | Implementation guide for MODBUS messaging over TCP/IP including TCP connection management and BSD socket interface usage         |
| [MODBUS TCP Security Protocol Specification](/wiki/summaries/MODBUS/modbussecurityprotocol.md)            | MODBUS/TCP Security specification defining secure communication over TLS with mutual authentication and role-based authorization |

## Core Concepts

### Protocol Variants

| Concept | Description |
|---------|-------------|
| [MODBUS TCP](/wiki/concepts/modbus-tcp.md) | MODBUS over TCP/IP protocol using MBAP header for client/server communication over Ethernet networks |
| [MODBUS RTU](/wiki/concepts/modbus-rtu.md) | MODBUS Remote Terminal Unit mode for serial line communication using binary encoding and CRC-16 error checking |
| [MODBUS ASCII](/wiki/concepts/modbus-ascii.md) | MODBUS ASCII transmission mode for serial line communication using ASCII hex encoding and LRC error checking |
| [MODBUS TCP Security](/wiki/concepts/modbus-tcp-security.md) | MODBUS/TCP Security (MBAPS) protocol adding TLS encryption, mutual authentication, and role-based authorization to MODBUS/TCP |

### Protocol Architecture

| Concept | Description |
|---------|-------------|
| [MODBUS Protocol](/wiki/concepts/modbus.md) | Comprehensive overview of MODBUS protocol including history, architecture, key characteristics, and role in industrial automation |
| [Protocol](/wiki/concepts/protocol.md) | General protocol concepts including OSI model, request-response patterns, and communication principles |
| [Master-Slave Architecture](/wiki/concepts/master-slave.md) | MODBUS master-slave communication model including roles, addressing, timing, and operational characteristics |
| [Implementation](/wiki/concepts/implementation.md) | Best practices, patterns, and guidelines for implementing MODBUS client and server functionality in software systems |
| [Modular Agent Architecture](/wiki/concepts/modular-architecture.md) | Modular agent system enabling specialized, focused agents with minimal context loading and single responsibility principle |

### Protocol Variants

### Data Model

| Concept | Description |
|---------|-------------|
| [Coils](/wiki/concepts/coils.md) | Single-bit read-write data objects used for digital outputs and control relays |
| [Discrete Inputs](/wiki/concepts/discrete-inputs.md) | Single-bit read-only data objects used for digital inputs and limit switches |
| [Input Registers](/wiki/concepts/input-registers.md) | 16-bit read-only data objects used for analog measurements and sensor readings |
| [Holding Registers](/wiki/concepts/holding-registers.md) | 16-bit read-write data objects used for configuration, setpoints, and control values |

### Protocol Components

| Concept | Description |
|---------|-------------|
| [Function Codes](/wiki/concepts/function-codes.md) | MODBUS function codes defining operations for reading and writing data including coils, discrete inputs, and registers |
| [MBAP Header](/wiki/concepts/mbap-header.md) | MODBUS Application Protocol header for TCP/IP encapsulation providing transaction management and routing information |
| [TCP Connection Management](/wiki/concepts/tcp-connection-management.md) | TCP connection lifecycle management for MODBUS/TCP including establishment, maintenance, pooling, and error handling |
| [CRC-16](/wiki/concepts/crc-16.md) | Cyclic Redundancy Check 16-bit error detection algorithm used in MODBUS RTU mode for frame integrity |

### Applications and Usage

| Concept | Description |
|---------|-------------|
| [MODBUS Usage and Applications](/wiki/concepts/modbus-usage-and-applications.md) | Industries and applications where MODBUS protocol is commonly deployed including manufacturing, building automation, energy, and process control |

### Implementation and Integration

| Concept | Description |
|---------|-------------|
| [MODBUS Data Type Mapping](/wiki/concepts/modbus-data-type-mapping.md) | Techniques and patterns for mapping MODBUS register values to/from modern programming language types including Rust and Python implementations |

## Quick Reference

### MODBUS Transport Variants

| Variant | Medium | Error Check | Framing | Port |
|---------|--------|-------------|---------|------|
| MODBUS TCP | Ethernet | TCP/IP | MBAP header (7 bytes) | 502 |
| MODBUS RTU | Serial (RS-485/232) | CRC-16 | Silent intervals | N/A |
| MODBUS ASCII | Serial | LRC | Start ':' / End CR-LF | N/A |
| MODBUS TCP Security | Ethernet + TLS | TCP/IP + TLS | MBAP over TLS | 802 |

### MODBUS Data Model

| Table | Object Type | Size | Access | Typical Use |
|-------|-------------|------|--------|-------------|
| Coils | Bit | 1 bit | Read/Write | Digital outputs, relays |
| Discrete Inputs | Bit | 1 bit | Read-only | Digital inputs, switches |
| Holding Registers | Word | 16 bits | Read/Write | Configuration, setpoints |
| Input Registers | Word | 16 bits | Read-only | Measurements, status |

### Common Function Codes

| Code | Hex | Name | Description |
|------|-----|------|-------------|
| 01 | 0x01 | Read Coils | Read status of coils (up to 2000 bits) |
| 02 | 0x02 | Read Discrete Inputs | Read status of discrete inputs (up to 2000 bits) |
| 03 | 0x03 | Read Holding Registers | Read holding register values (up to 125 registers) |
| 04 | 0x04 | Read Input Registers | Read input register values (up to 125 registers) |
| 05 | 0x05 | Write Single Coil | Write single coil to ON or OFF |
| 06 | 0x06 | Write Single Register | Write single register value |
| 15 | 0x0F | Write Multiple Coils | Write multiple coils (up to 1968 bits) |
| 16 | 0x10 | Write Multiple Registers | Write multiple registers (up to 123 registers) |

## Navigation

- Browse by [Document Summaries](#document-summaries)
- Browse by [Core Concepts](#core-concepts)
- View [Change Log](/wiki/log.md)

## Related Information

- [MODBUS Organization](https://www.modbus.org/) - Official MODBUS website
- [MODBUS Specifications](https://www.modbus.org/modbus-specifications) - Official specifications
- [Source Documents](/raw/MODBUS/) - Raw source documents (immutable)
