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
last-updated: 2026-04-25T10:00:00+03:00
---

Welcome to the MODBUS Protocol Wiki. This wiki contains organized information about the MODBUS protocol specifications, extracted from official MODBUS documentation.

## Document Summaries

Summaries of source documents in `raw/MODBUS/`:

| Document                                                                                                  | Description                                                                                                                      |
| --------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| [Summary of MODBUS Protocol Specification for AI Implementation](/wiki/summaries/MODBUS/MODBUS.md)                   | Comprehensive technical reference for MODBUS protocol optimized for AI implementations                                           |
| [Summary of MODBUS Application Protocol Specification](/wiki/summaries/MODBUS/modbusprotocolspecification.md)        | Official MODBUS Application Protocol specification describing communication model, data model, and function codes                |
| [Summary of MODBUS over serial line specification and implementation guide](/wiki/summaries/MODBUS/modbusoverserial.md)                       | MODBUS serial line specification covering master-slave protocol, RTU and ASCII modes, and RS485 physical layer                   |
| [Summary of MODBUS Messaging on TCP/IP Implementation Guide](/wiki/summaries/MODBUS/messagingimplementationguide.md) | Implementation guide for MODBUS messaging over TCP/IP including TCP connection management and BSD socket interface usage         |
| [Summary of MODBUS/TCP Security](/wiki/summaries/MODBUS/modbussecurityprotocol.md)            | MODBUS/TCP Security specification defining secure communication over TLS with mutual authentication and role-based authorization |

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
| [LRC](/wiki/concepts/lrc.md) | Longitudinal Redundancy Check error detection algorithm used in MODBUS ASCII mode for frame integrity |

### Applications and Usage

| Concept | Description |
|---------|-------------|
| [MODBUS Usage and Applications](/wiki/concepts/modbus-usage-and-applications.md) | Industries and applications where MODBUS protocol is commonly deployed including manufacturing, building automation, energy, and process control |

### Implementation and Integration

| Concept | Description |
|---------|-------------|
| [MODBUS Data Type Mapping](/wiki/concepts/modbus-data-type-mapping.md) | Techniques and patterns for mapping MODBUS register values to/from modern programming language types including Rust and Python implementations |

## Answers

Answers to specific questions about MODBUS implementation and usage:

| Answer | Description |
|--------|-------------|
| [MODBUS Function Codes Complete Guide](/wiki/answers/modbus-function-codes-guide.md) | Comprehensive guide to MODBUS function codes including what they do, how they are grouped, when to use each one, efficiency comparisons between single and multiple operations, and practical decision criteria |
| [Reading MODBUS Register Maps](/wiki/answers/reading-modbus-register-maps.md) | Comprehensive guide to reading and interpreting MODBUS register maps including addressing conventions, register types, function codes, data types, scaling factors, and handling vendor-specific documentation inconsistencies |
| [MODBUS Commissioning Checklist](/wiki/answers/modbus-commissioning-checklist.md) | Comprehensive checklist for commissioning MODBUS communications on site including pre-commissioning preparation, required physical tools, software tools, installation procedures, testing steps, and troubleshooting guidelines |
| [MODBUS Errors and Exception Responses](/wiki/answers/modbus-errors-and-exceptions.md) | Comprehensive guide to MODBUS error types, exception responses, exception codes, diagnostic procedures, and troubleshooting addressing errors, unsupported functions, and device limitations |
| [How MODBUS TCP-to-RTU Gateways Work](/wiki/answers/modbus-tcp-to-rtu-gateway.md) | Comprehensive explanation of MODBUS TCP-to-RTU gateway operation including protocol conversion, Unit ID mapping, frame translation, timing management, error handling, and practical implementation considerations |
| [Using Wireshark for MODBUS TCP Analysis](/wiki/answers/wireshark-modbus-analysis.md) | Comprehensive guide to using Wireshark for MODBUS TCP communication analysis including byte-level inspection, packet recognition, protocol decoding, filtering, and analysis of both standard and secure MODBUS traffic |
| [MODBUS TCP Message Format](/wiki/answers/modbus-tcp-message-format.md) | Complete description of MODBUS TCP message format including MBAP header structure, frame layout, field details, differences from MODBUS RTU, TCP error checking rationale, and connection management |
| [Connecting with MODBUS/TCP Security](/wiki/answers/connecting-with-modbus-tcp-security.md) | Essential considerations and requirements for establishing secure MODBUS/TCP connections including certificates, TLS configuration, role-based authorization, and connection establishment |
| [What is MBAP?](/wiki/answers/what-is-mbap.md) | Explanation of MBAP (MODBUS Application Protocol) header, its structure, purpose, and how it enables MODBUS communication over TCP/IP networks |
| [MODBUS Data Validation](/wiki/answers/modbus-data-validation.md) | Comprehensive guide to validating MODBUS register values when reading from and writing to devices, including protocol-level validation, data type checking, and best practices |
| [MODBUS Register Data Representation](/wiki/answers/modbus-register-data-representation.md) | Comprehensive guide to how MODBUS registers store different data types including booleans, integers, floats, and strings, covering register size, byte order, word order, and determining device endianness |
| [Converting MODBUS Registers to Program Variables](/wiki/answers/converting-modbus-registers-to-program-variables.md) | Comprehensive guide to converting between MODBUS register data and modern programming language variables, handling type system mismatches, endianness, and floating-point representation, with complete Rust implementation examples |
| [MODBUS Broadcast](/wiki/answers/modbus-broadcast.md) | Comprehensive guide to MODBUS broadcast functionality including broadcast addresses, operation on serial vs TCP networks, effects and limitations, and troubleshooting broadcast issues |
| [MODBUS RTU vs ASCII Comparison](/wiki/answers/modbus-rtu-vs-ascii.md) | Comprehensive comparison of MODBUS RTU and ASCII transmission modes including protocol frames, encoding differences, error checking methods, CRC calculation, and when to use each mode |
| [RS-485 Wiring for MODBUS RTU](/wiki/answers/rs485-wiring-for-modbus-rtu.md) | Comprehensive guide to RS-485 physical wiring for MODBUS RTU including topology, voltage levels, 2-wire vs 4-wire configurations, shielding, termination resistors, polarization, device limits, and practical installation guidelines |
| [MODBUS over RS-232](/wiki/answers/modbus-over-rs232.md) | Comprehensive guide to running MODBUS over RS-232 including setup, limitations, comparison with RS-485, point-to-point operation, wiring, and when to use RS-232 vs RS-485 for MODBUS RTU |
| [TLS and MODBUS Security](/wiki/answers/tls-and-modbus-security.md) | Comprehensive explanation of TLS, its role in MODBUS Security, cyber-attacks prevented by TLS, and how TLS prevents unauthorized access to MODBUS devices through mutual authentication and role-based authorization |

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
- View [MODBUS Wiki Change Log](/wiki/log.md)

## Related Information

- [MODBUS Organization](https://www.modbus.org/) - Official MODBUS website
- [MODBUS Specifications](https://www.modbus.org/modbus-specifications) - Official specifications
- [Source Documents](/raw/MODBUS/) - Raw source documents (immutable)
