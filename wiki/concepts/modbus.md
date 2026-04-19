---
title: MODBUS Protocol
Summary: Comprehensive overview of the MODBUS protocol, including its history, architecture, key characteristics, and role in industrial automation.
Sources:
  - raw/MODBUS/MODBUS.md
  - raw/MODBUS/modbusprotocolspecification.md
Categories:
  - overview
  - protocol-architecture
  - history
date-created: 2026-04-18T12:00:00+03:00
last-updated: 2026-04-18T16:00:00+03:00
---

MODBUS is an application-layer messaging protocol (OSI Layer 7) providing client/server communication between devices connected on different types of buses or networks (source: [modbusprotocolspecification.md](raw/MODBUS/modbusprotocolspecification.md)). Originally developed by Modicon in 1979 for programmable logic controllers (PLCs), it remains the de facto standard for industrial device communication (source: [MODBUS.md](raw/MODBUS/MODBUS.md)).

## Protocol Overview

MODBUS is a simple, robust, and openly published protocol that has become ubiquitous in industrial automation due to its simplicity, openness, and widespread vendor support (source: [modbusprotocolspecification.md](raw/MODBUS/modbusprotocolspecification.md)). The protocol defines a client/server (also known as master/slave) communication model where clients initiate requests and servers respond with data or exception codes.

## Key Characteristics

| Property | Description |
|-----------|-------------|
| Architecture | Client/Server (Master/Slave) |
| Protocol Layer | OSI Layer 7 (Application) |
| Simplicity | Simple request/response model |
| Openness | Public specification, no licensing fees |
| Portability | Multiple transport layers (TCP/IP, RS-485, RS-232) |
| Scalability | Supports up to 247 devices on same network (1-247) |
| Maturity | Over 45 years in production use (since 1979) |

## Historical Background

MODBUS was developed in 1979 by Modicon (now Schneider Electric) for use with their Modicon programmable logic controllers (source: [MODBUS.md](raw/MODBUS/MODBUS.md)). The protocol was designed to enable communication between Modicon PLCs and other devices such as sensors, actuators, and monitoring systems.

The protocol's openness and simplicity led to widespread adoption across manufacturers, making it one of the most common industrial protocols in use today.

## Protocol Architecture

### Client/Server Model

MODBUS uses a client/server communication model (also referred to as master/slave in the original specification):

- **Client (Master):** Initiates all requests to one or more server devices
- **Server (Slave):** Responds to requests from clients, processes commands, and provides data
- **Point-to-Point:** Single client communicates with single server
- **Broadcasting:** Client can send to all servers (address 0), but servers don't respond

This model ensures deterministic communication where the client controls the timing and sequence of operations (source: [modbusprotocolspecification.md](raw/MODBUS/modbusprotocolspecification.md)).

### Transport Independence

MODBUS is transport-independent at the application layer, meaning it can operate over various physical and data link layers (source: [modbusprotocolspecification.md](raw/MODBUS/modbusprotocolspecification.md)):

**Common Transport Variants:**
- [modbus-tcp](/wiki/concepts/modbus-tcp.md) - MODBUS over TCP/IP networks
- [modbus-rtu](/wiki/concepts/modbus-rtu.md) - MODBUS Remote Terminal Unit (binary serial)
- [modbus-ascii](/wiki/concepts/modbus-ascii.md) - MODBUS ASCII (text serial)
- [modbus-tcp-security](/wiki/concepts/modbus-tcp-security.md) - MODBUS/TCP with TLS security

This transport independence allows MODBUS to be deployed across diverse network topologies and physical media while maintaining the same application protocol.

## Protocol Data Model

MODBUS defines four primary data tables representing different types of data objects (source: [modbusprotocolspecification.md](raw/MODBUS/modbusprotocolspecification.md)):

| Table | Object Type | Access | Description |
|-------|-------------|--------|-------------|
| [coils](/wiki/concepts/coils.md) | Bit | Read/Write | Single-bit outputs for digital control |
| [discrete-inputs](/wiki/concepts/discrete-inputs.md) | Bit | Read-only | Single-bit inputs for digital sensing |
| [input-registers](/wiki/concepts/input-registers.md) | Word | Read-only | 16-bit sensor measurements and status |
| [holding-registers](/wiki/concepts/holding-registers.md) | Word | Read/Write | 16-bit configuration and setpoints |

This simple data model provides flexibility while maintaining protocol simplicity.

## Protocol Data Units

MODBUS exchanges data using two types of protocol data units (source: [modbusprotocolspecification.md](raw/MODBUS/modbusprotocolspecification.md)):

### Protocol Data Unit (PDU)

The PDU is the application layer message that is independent of the underlying transport layer. It contains:
- **Function Code:** 1 byte - specifies the operation to perform
- **Request Data:** 0-252 bytes - parameters for the requested operation
- **Response Data:** 0-252 bytes - result of the requested operation
- **Maximum Size:** 253 bytes total (1 + 252)

### Application Data Unit (ADU)

The ADU adds transport-specific framing to the PDU:

**MODBUS TCP ADU Structure:**
- MBAP Header: 7 bytes (Transaction ID, Protocol ID, Length, Unit ID)
- PDU: Variable length (up to 253 bytes)
- Total maximum: 260 bytes

**MODBUS Serial ADU Structure:**
- Address: 1 byte (slave address)
- CRC-16/LRC: 2 bytes (error detection)
- PDU: Variable length
- Silent intervals: Frame timing (3.5 character times)

See [mbap-header](/wiki/concepts/mbap-header.md) for MBAP header details.

## Operation Model

### Request/Response Cycle

1. **Client Request:** Client sends request with function code and parameters
2. **Server Processing:** Server executes requested operation
3. **Server Response:** Server sends response with data or exception code
4. **Client Processing:** Client processes response or handles exception

This simple request/response model provides reliability and allows for error handling through exception codes.

### Exception Handling

When a server cannot process a request, it returns an exception response with the function code's most significant bit set (function code + 0x80) and an exception code (source: [modbusprotocolspecification.md](raw/MODBUS/modbusprotocolspecification.md)).

Common exception codes include:
- 0x01: ILLEGAL FUNCTION - Function code not supported
- 0x02: ILLEGAL DATA ADDRESS - Address out of range
- 0x03: ILLEGAL DATA VALUE - Invalid parameter value
- 0x04: SERVER DEVICE FAILURE - Internal error

## Addressing

MODBUS uses different addressing models depending on the transport variant:

**TCP/IP Addressing:**
- Unit ID field in MBAP header
- 0xFF for direct connection
- 1-247 for gateway routing to serial devices

**Serial Addressing:**
- 1-byte slave address in frame
- 0 for broadcast (no response)
- 1-247 for individual devices
- 248-255 reserved

## Advantages and Limitations

### Advantages

- **Simplicity:** Easy to implement and debug
- **Openness:** Public specification, no licensing fees
- **Widespread Support:** Available from many vendors
- **Transport Flexibility:** Works over TCP/IP and multiple serial media
- **Scalability:** Supports large networks with many devices
- **Maturity:** Proven reliability over decades of use

### Limitations

- **Performance:** Not suitable for high-speed, real-time control loops
- **Security:** Basic implementations lack encryption and authentication (use [modbus-tcp-security](/wiki/concepts/modbus-tcp-security.md))
- **Data Size:** Limited to 253-byte PDU size
- **Determinism:** TCP is not deterministic (serial variants are)
- **Broadcasting:** Limited broadcast support, no responses

## Applications

MODBUS is used across many industries including:
- [modbus-usage-and-applications](/wiki/concepts/modbus-usage-and-applications.md) - Manufacturing and factory automation
- Building automation and HVAC control
- Energy management and power generation
- Water treatment and environmental monitoring
- Transportation and infrastructure systems
- Oil and gas pipeline monitoring

## Related Information

- [function-codes](/wiki/concepts/function-codes.md) - Detailed function code specifications
- [modbus-tcp](/wiki/concepts/modbus-tcp.md) - MODBUS/TCP implementation details
- [modbus-rtu](/wiki/concepts/modbus-rtu.md) - Serial RTU mode specifications
- [modbus-ascii](/wiki/concepts/modbus-ascii.md) - Serial ASCII mode specifications
- [MODBUS](/wiki/summaries/MODBUS/MODBUS.md) - AI implementation reference

See also: [protocol-architecture](/wiki/concepts/protocol-architecture.md) for detailed protocol design concepts.