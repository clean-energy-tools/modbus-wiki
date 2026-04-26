---
title: MODBUS TCP
Summary: MODBUS over TCP/IP protocol using MBAP header for client/server communication over Ethernet networks.
Sources:
  - raw/MODBUS/MODBUS.md
  - raw/MODBUS/modbusprotocolspecification.md
  - raw/MODBUS/messagingimplementationguide.md
Categories:
  - tcp-ip
  - networking
  - client-server
type: concept
date-created: 2026-04-18T12:00:00+03:00
last-updated: 2026-04-18T14:43:24+03:00
---

MODBUS TCP is MODBUS protocol implementation over TCP/IP networks, using a 7-byte MBAP (MODBUS Application Protocol) header to encapsulate MODBUS requests and responses (source: [MODBUS.md](/raw/MODBUS/MODBUS.md)).

## Protocol Overview

MODBUS TCP provides a client/server communication model over Ethernet TCP/IP networks, removing the physical constraints of serial communication while maintaining the same MODBUS application protocol (source: [MODBUS.md](/raw/MODBUS/MODBUS.md)).

### Key Characteristics

| Property | Value |
|----------|-------|
| Transport | TCP/IP |
| Port | 502 (standard), 802 (secure) |
| Maximum ADU size | 260 bytes (7 header + 253 PDU) |
| Byte order | Big-endian (network byte order) |
| Error checking | TCP/IP checksum (inherent) |
| Framing | MBAP header provides message boundaries |

## MBAP Header

The MBAP header identifies and manages MODBUS messages on TCP/IP networks (source: [messagingimplementationguide.md](/raw/MODBUS/messagingimplementationguide.md)):

| Offset | Field | Size | Description |
|--------|-------|------|-------------|
| 0 | Transaction ID | 2 bytes | Request/response matching (client-assigned) |
| 2 | Protocol ID | 2 bytes | 0x0000 for MODBUS |
| 4 | Length | 2 bytes | Bytes following (Unit ID + PDU length) |
| 6 | Unit ID | 1 byte | Slave address (0xFF for direct, 1-247 for gateway) |

**Complete TCP ADU:**
```
[Transaction ID: 2][Protocol ID: 2][Length: 2][Unit ID: 1][Function: 1][Data: N]
|<---------------- MBAP Header (7 bytes) --------------->|<----- PDU ------->|
```

### MBAP Header Fields

**Transaction Identifier:**
- Used for request/response pairing
- Client assigns unique value
- Server echoes back in response
- Allows multiple concurrent transactions on same connection
- Must be unique at any time on each TCP connection

**Protocol Identifier:**
- Value of 0x0000 indicates MODBUS protocol
- Allows multiplexing with other protocols on same port
- Client sets it, server echoes back

**Length:**
- Byte count of following bytes (Unit ID + PDU)
- Client sets in request
- Server sets in response
- Allows recipient to detect message boundaries across TCP packets

**Unit Identifier:**
- Used for routing through gateways and bridges
- 0xFF for direct connection (most common)
- 1-247 for addressing MODBUS serial slaves through gateway
- Set by client, echoed by server

## Connection Management

### Connection Establishment

**Client:**
1. Create TCP socket
2. Connect to server IP:502
3. Set socket options (TCP_NODELAY, SO_KEEPALIVE)
4. Use connection for multiple MODBUS transactions

**Server:**
1. Create TCP socket
2. Bind to port 502
3. Listen for connections
4. Accept multiple clients (connection pool)
5. Track connections for cleanup

### Connection Lifecycle

**Best Practices (source: [messagingimplementationguide.md](/raw/MODBUS/messagingimplementationguide.md)):**
- Keep connections open - do not open/close per transaction
- Use one connection per application to same server
- Multiple transactions can be active on same connection
- Use unique Transaction IDs for concurrent transactions
- Send one ADU per TCP frame (don't batch requests)
- Enable TCP_NODELAY for real-time performance
- Enable SO_KEEPALIVE to detect dead connections

**Connection Timeout:**
- Response timeout: 1-5 seconds (application-specific)
- Use application-level timers for MODBUS responses
- TCP handles retransmission and reliability

## Request/Response Example

### Read Holding Registers (Function 0x03)

**Scenario:** Read 2 registers starting at address 0 from Unit ID 1

**Request (12 bytes):**
```
00 01 00 00 00 06 01 03 00 00 00 02
|     |     |     |  |  |     |
|     |     |     |  |  |     +-- Quantity: 2 registers
|     |     |     |  |  +-------- Start Address: 0
|     |     |     |  +----------- Function: 0x03
|     |     |     +-------------- Unit ID: 1
|     |     +-------------------- Length: 6 bytes follow
|     +-------------------------- Protocol ID: 0
+-------------------------------- Transaction ID: 1
```

**Response (13 bytes - Success):**
```
00 01 00 00 00 07 01 03 04 00 0A 00 14
|     |     |     |  |  |  |     |
|     |     |     |  |  |  |     +-- Register 1: 0x0014 (20)
|     |     |     |  |  |  +-------- Register 0: 0x000A (10)
|     |     |     |  |  +----------- Byte count: 4 bytes
|     |     |     |  +-------------- Function: 0x03
|     |     |     +----------------- Unit ID: 1
|     |     +----------------------- Length: 7 bytes follow
|     +----------------------------- Protocol ID: 0
+----------------------------------- Transaction ID: 1
```

**Response (9 bytes - Error):**
```
00 01 00 00 00 03 01 83 02
|     |     |     |  |  |
|     |     |     |  |  +-- Exception Code: 0x02
|     |     |     |  +----- Function: 0x83 (0x03 + 0x80)
|     |     |     +-------- Unit ID: 1
|     |     +-------------- Length: 3 bytes follow
|     +-------------------- Protocol ID: 0
+-------------------------- Transaction ID: 1
```

## Differences from MODBUS Serial

| Characteristic | MODBUS Serial | MODBUS TCP |
|---------------|---------------|-------------|
| Address field | 1 byte slave address | 1 byte Unit ID (for gateways) |
| Error checking | CRC-16 (RTU) or LRC (ASCII) | TCP/IP checksum |
| Maximum size | 256 bytes (RTU), 513 chars (ASCII) | 260 bytes |
| Framing | Silent intervals / delimiters | MBAP Length field |
| Connection | Point-to-point master/slave | Client/server with network |
| Broadcast | Supported (address 0) | Not applicable (Unit ID 0xFF) |

## TCP/IP Stack Integration

### Socket Options (source: [messagingimplementationguide.md](/raw/MODBUS/messagingimplementationguide.md))

**TCP_NODELAY:**
- Disables Nagle algorithm
- Sends small packets immediately
- Recommended for real-time MODBUS

**SO_KEEPALIVE:**
- Enables TCP keep-alive mechanism
- Detects crashed/absent endpoints
- Default idle time: 2 hours
- Probe interval: 75 seconds
- Maximum probes: 8

**SO-REUSEADDR:**
- Allows immediate port reuse
- Bypasses TIME-WAIT state
- Recommended for server restarts

**SO-RCVBUF, SO-SNDBUF:**
- Adjust socket buffer sizes
- Typical: 900 bytes (3 frames)
- Balances performance vs resource usage

## Gateway Support

The Unit Identifier field enables routing through gateways that convert between MODBUS TCP/IP and MODBUS Serial Line (source: [messagingimplementationguide.md](/raw/MODBUS/messagingimplementationguide.md)):

**Gateway Scenario:**
```
MODBUS TCP Client ──┐
                     ├── Gateway (IP: 192.168.1.100) ──┐
MODBUS TCP Client ──┘                              ├── RS-485 Bus
                                                   │
                                                  ┌──┴──┐
                                                  │     │
                                                Slave 1 Slave 2
                                                (ID:1) (ID:2)
```

**Unit ID Mapping:**
- Direct connection: Unit ID = 0xFF
- Via gateway to slave 1: Unit ID = 1
- Via gateway to slave 2: Unit ID = 2

## Security

MODBUS TCP itself has no built-in security. For secure communication, use [MODBUS TCP Security](/wiki/concepts/modbus-tcp-security.md) (port 802) which adds TLS encryption and mutual authentication.

## Related pages

- [MODBUS ASCII](/wiki/concepts/modbus-ascii.md)
- [MODBUS TCP Security](/wiki/concepts/modbus-tcp-security.md)
- [MBAP Header](/wiki/concepts/mbap-header.md)
- [TCP Connection Management](/wiki/concepts/tcp-connection-management.md)
- [Function Codes](/wiki/concepts/function-codes.md)

## Backlinks

No backlinks (yet)

------------

MODBUS is a trademark of the Modbus Organization, Inc.

All information on this page is derived by summarizing and analyzing solely the MODBUS specifications published by the Modbus Organization, Inc.  The specifications are copyright by the Modbus Organization.

By deriving this information solely from the specifications we hope to stay true to the specification.

No infringement is intended.

- [MODBUS Broadcast](/wiki/answers/modbus-broadcast.md) - Broadcast limitations on MODBUS TCP
