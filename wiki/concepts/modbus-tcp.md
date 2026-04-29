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

MODBUS TCP is the version of MODBUS that works on modern Ethernet networks (source: [MODBUS.md](/raw/MODBUS/MODBUS.md)). It adds a 7-byte header (called an MBAP header) to wrap MODBUS messages for sending over TCP/IP.

## What MODBUS TCP Does

MODBUS TCP lets devices talk to each other over Ethernet networks, just like they browse the internet (source: [MODBUS.md](/raw/MODBUS/MODBUS.md)). The MODBUS messages stay the same, but they travel over network cables instead of old-style serial cables.

### What Makes MODBUS TCP Different

| Feature | Details |
|----------|-------|
| Connection type | Ethernet (TCP/IP) |
| Port number | 502 (normal), 802 (encrypted) |
| Maximum message size | 260 bytes (7 for header + 253 for data) |
| Number format | Big-endian (most significant byte first) |
| Error checking | Built into TCP/IP automatically |
| Message boundaries | MBAP header shows where messages start and end |

## The MBAP Header

The MBAP header is extra information added to identify and track MODBUS messages on Ethernet (source: [messagingimplementationguide.md](/raw/MODBUS/messagingimplementationguide.md)):

| Position | What It Contains | Size | Purpose |
|--------|-------|------|-------------|
| 0 | Message ID | 2 bytes | Matches requests with responses |
| 2 | Protocol marker | 2 bytes | 0x0000 means "this is MODBUS" |
| 4 | Message length | 2 bytes | How many bytes follow this field |
| 6 | Device ID | 1 byte | Which device (0xFF = directly connected, 1-247 = through gateway) |

**A complete MODBUS TCP message:**
```
[Message ID: 2][Protocol: 2][Length: 2][Device: 1][Operation: 1][Data: varies]
|<---------------- MBAP Header (7 bytes) ------------->|<-- MODBUS Message -->|
```

### What Each Header Field Does

**Message ID (Transaction Identifier):**
- Helps match answers to questions
- The client picks a unique number for each question
- The server copies this number into its answer
- Lets you ask multiple questions at once without mixing up the answers
- Must be different for each active conversation

**Protocol Marker:**
- The value 0x0000 means "this is a MODBUS message"
- Lets different types of messages share the same network port
- Client sets it, server copies it back

**Message Length:**
- Tells how many bytes come after this field
- Client sets it when asking
- Server sets it when answering
- Helps figure out where one message ends and the next begins

**Device ID (Unit Identifier):**
- Used to route messages through gateways
- 0xFF (255) means "device directly connected" (most common case)
- 1-247 for reaching serial devices through a gateway
- Client sets it, server copies it back

## Managing Connections

### Setting Up a Connection

**Client (the one asking questions):**
1. Open a network connection
2. Connect to server's IP address on port 502
3. Configure connection options for best performance
4. Keep using this same connection for many messages

**Server (the one answering):**
1. Open a network connection
2. Listen on port 502
3. Wait for connections
4. Accept connections from multiple clients at once
5. Keep track of all connections

### How to Use Connections Well

**Good Practices (source: [messagingimplementationguide.md](/raw/MODBUS/messagingimplementationguide.md)):**
- Keep connections open - don't open and close for each message
- Use one connection per server you're talking to
- You can send multiple messages on the same connection
- Use different Message IDs if you send messages before getting answers
- Send one message at a time (don't batch them together)
- Turn on TCP_NODELAY for faster responses
- Turn on SO_KEEPALIVE to notice if the connection dies

**Timeouts:**
- Wait 1-5 seconds for an answer (depends on your application)
- Use timers in your program to watch for MODBUS answers
- TCP automatically resends lost messages

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

- [Connecting with MODBUS/TCP Security](/wiki/answers/connecting-with-modbus-tcp-security.md)
- [How MODBUS TCP-to-RTU Gateways Work](/wiki/answers/modbus-tcp-to-rtu-gateway.md)
- [MODBUS Broadcast](/wiki/answers/modbus-broadcast.md)
- [MODBUS Commissioning Checklist](/wiki/answers/modbus-commissioning-checklist.md)
- [MODBUS over RS-232](/wiki/answers/modbus-over-rs232.md)
- [MODBUS TCP Message Format](/wiki/answers/modbus-tcp-message-format.md)
- [TLS and MODBUS Security](/wiki/answers/tls-and-modbus-security.md)
- [Using Wireshark for MODBUS TCP Analysis](/wiki/answers/wireshark-modbus-analysis.md)
- [What is MBAP?](/wiki/answers/what-is-mbap.md)
- [Implementation](/wiki/concepts/implementation.md)
- [MBAP Header](/wiki/concepts/mbap-header.md)
- [MODBUS ASCII](/wiki/concepts/modbus-ascii.md)
- [MODBUS Data Type Mapping](/wiki/concepts/modbus-data-type-mapping.md)
- [MODBUS Protocol](/wiki/concepts/modbus.md)
- [MODBUS RTU](/wiki/concepts/modbus-rtu.md)
- [MODBUS TCP Security](/wiki/concepts/modbus-tcp-security.md)
- [MODBUS Usage and Applications](/wiki/concepts/modbus-usage-and-applications.md)
- [Protocol](/wiki/concepts/protocol.md)
- [TCP Connection Management](/wiki/concepts/tcp-connection-management.md)
- [Summary of MODBUS Application Protocol Specification](/wiki/summaries/MODBUS/MODBUS.md)
- [Summary of MODBUS Messaging on TCP/IP Implementation Guide](/wiki/summaries/MODBUS/messagingimplementationguide.md)
- [Summary of MODBUS over serial line specification and implementation guide](/wiki/summaries/MODBUS/modbusoverserial.md)
- [Summary of MODBUS Protocol Specification for AI Implementation](/wiki/summaries/MODBUS/modbusprotocolspecification.md)
- [Summary of MODBUS/TCP Security](/wiki/summaries/MODBUS/modbussecurityprotocol.md)

------------

MODBUS is a trademark of the Modbus Organization, Inc.

All information on this page is derived by summarizing and analyzing solely the MODBUS specifications published by the Modbus Organization, Inc.  The specifications are copyright by the Modbus Organization.

By deriving this information solely from the specifications we hope to stay true to the specification.

No infringement is intended.

- [MODBUS Broadcast](/wiki/answers/modbus-broadcast.md) - Broadcast limitations on MODBUS TCP
