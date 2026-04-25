---
title: Summary of MODBUS Messaging on TCP/IP Implementation Guide
Summary: Implementation guide for MODBUS messaging over TCP/IP, including TCP connection management, BSD socket interface usage, and client/server architecture.
Sources:
  - raw/MODBUS/messagingimplementationguide.md
Categories:
  - implementation
  - tcp-ip
  - networking
type: summary
date-created: 2026-04-18T12:00:00+03:00
last-updated: 2026-04-18T14:43:52+03:00
---

This document provides detailed implementation guidance for MODBUS messaging over TCP/IP networks, covering TCP connection management, BSD socket interface usage, client/server architecture, and component design patterns (source: [messagingimplementationguide.md](/raw/MODBUS/messagingimplementationguide.md)).

## Protocol Overview

MODBUS over TCP/IP encapsulates the MODBUS Application Protocol within TCP/IP using a 7-byte MBAP (MODBUS Application Protocol) header. The protocol maintains the client/server model from the original MODBUS specification (source: [messagingimplementationguide.md](/raw/MODBUS/messagingimplementationguide.md)).

### Communication Architecture

MODBUS TCP/IP supports several device configurations:
- MODBUS TCP/IP Client and Server devices on TCP/IP network
- Interconnection devices (bridges, routers, gateways) for TCP/IP to serial line conversion
- MODBUS Serial Line Client and Server end devices via gateways

## MBAP Header

The MBAP header identifies and manages MODBUS messages on TCP/IP networks (source: [messagingimplementationguide.md](/raw/MODBUS/messagingimplementationguide.md)):

| Field | Size | Description | Direction |
|-------|------|-------------|-----------|
| Transaction Identifier | 2 bytes | Request/response pairing | Client → Server (echoed) |
| Protocol Identifier | 2 bytes | 0 = MODBUS protocol | Client → Server (echoed) |
| Length | 2 bytes | Number of following bytes (Unit ID + PDU) | Client (request) / Server (response) |
| Unit Identifier | 1 byte | Remote slave address for gateways | Client → Server (echoed) |

**Key Differences from Serial:**
- Slave address field replaced by Unit Identifier in MBAP header
- Unit Identifier used for routing through gateways (single IP address supporting multiple MODBUS units)
- Length field allows message boundary detection across TCP packet boundaries
- All fields encoded in big-endian

**Port:** All MODBUS/TCP ADUs sent via TCP to registered port 502

## TCP Connection Management

### Connection Models

Two approaches to TCP connection management:

#### Explicit Connection Management
- User application manages all TCP connections
- Provides maximum flexibility
- Requires TCP/IP expertise from application programmer
- Uses BSD Socket interface directly

#### Automatic Connection Management
- Connection management transparent to user application
- Module establishes connections as needed
- Less flexible but simpler for applications

### Implementation Rules

1. **Automatic management recommended** without explicit user requirement
2. **Keep connections open** - do not open/close per transaction
3. **Minimum connections** - one connection per application to same server is good practice
4. **Concurrent transactions** - multiple MODBUS transactions can be active on same TCP connection (requires unique Transaction IDs)
5. **Separate connections** for bi-directional communication (each device acts as both client and server)
6. **One ADU per TCP frame** - send only one MODBUS request/response per TCP PDU

### Connection Lifecycle

**Establishment:**
- Client connects to server port 502
- Server uses listening socket on port 502
- Client uses local port > 1024, different for each connection

**Data Transfer:**
- Use existing TCP connection for multiple transactions
- Send one ADU per TCP send operation
- Multiple requests can be outstanding (pending responses)

**Closing:**
- Client initiates connection close when communications complete
- Server must accept close requests from clients
- Connections can be reopened when required

### Connection Pool Management

Implement two connection pools:

**Priority Connection Pool:**
- Connections for "marked" devices (specific IP addresses)
- Never closed on local initiative
- Configurable maximum connections per device

**Non-priority Connection Pool:**
- Connections for non-marked devices
- Close oldest unused connection when pool full
- Configurable maximum connections

**Pool Strategy:**
- When pool full and new request from non-marked device → close oldest unused connection
- Access control can authorize/forbid specific IP addresses

### Connection Failure Handling

**Communication break between operational endpoints:**
- No packets sent: Not detected until Keep-Alive timer expires
- Packets sent: TCP retransmission algorithms active, may trigger connection reset

**Server crash and reboot:**
- Connection becomes "half-open" on client
- No packets: Client sees connection open until Keep-Alive expires
- Packets sent: Server sends RST to close connection

**Client crash and reboot:**
- Connection becomes "half-open" on server
- No packets: Server sees connection open until Keep-Alive expires
- Client reconnects with same characteristics: Connection establishment timeout (75s)
- Client reconnects with different characteristics: New connection accepted

## BSD Socket Interface

### Socket Operations

**Socket Creation:**
```
fd = socket()        // Create socket
bind(fd, port)      // Bind to port (server)
connect(fd, ip, port)  // Connect to remote (client)
listen(fd)          // Listen for connections (server)
accept(fd)           // Accept connection (server)
```

**Data Transfer:**
```
send(fd, data)       // Send data
recv(fd, buffer)     // Receive data
```

**Socket Options:**
```
setsockopt(fd, option)  // Set socket options
shutdown(fd)         // Disable send/receive
close(fd)            // Close socket
```

**Event Monitoring:**
```
select()             // Test events on multiple sockets
```

## TCP Parameterization

### Per-Connection Parameters

**SO-RCVBUF, SO-SNDBUF:**
- Set high water mark for send/receive sockets
- Adjust for flow control management
- Typical size: 900 bytes (3 frames of 300 bytes each)

**TCP-NODELAY:**
- Disables Nagle algorithm
- Recommended for real-time behavior
- Sends small packets immediately without waiting

**SO-REUSEADDR:**
- Allows immediate port reuse after connection close
- Bypasses TIME-WAIT state restriction
- Recommended for both client and server

**SO-KEEPALIVE:**
- Enables TCP keep-alive mechanism
- Detects crashed/absent endpoints
- Polls other end to verify connection

### TCP Layer Parameters

**Connection Establishment Timeout:**
- Default: 75 seconds on Berkeley-derived systems
- Adapt to application real-time requirements

**Keep-Alive Parameters:**
- Idle time before first probe: 2 hours (default)
- Probe interval: 75 seconds
- Maximum probes: 8
- Action after failure: Signal error to application

**Retransmission Parameters:**
- **RTO (Retransmission Time-Out):** Dynamically estimated
- **RTT (Round-Trip Time):** Measured after each packet
- **Default RTT:** 3 seconds (if estimate unavailable within 3 seconds)
- **Exponential BackOff:** Double retransmission timeout each time (max 64 seconds)
- **Fast Retransmission:** After 3 duplicate ACKs (recommended for LAN)

## MODBUS Client Design

### Client Activity Diagram

**Events:**
1. Request from user application
2. Response from TCP management
3. Timeout expiration

**Request Processing:**
1. Instantiate MODBUS transaction (for request/response pairing)
2. Encode MODBUS request (PDU + MBAP header)
3. Send request to TCP management module
4. Set waiting response timer
5. Wait for response or timeout
6. Process confirmation or retry

**Transaction Identifier Usage:**
- Simple counter incremented at each request
- Smart index/pointer to transaction context
- Must be unique on each TCP connection at any time
- Allows multiple concurrent transactions

### Request Building

**Transaction Instantiation:**
- Store all information for response binding
- Track current remote server
- Store pending MODBUS request

**Request Encoding:**
- Build PDU per MODBUS Application Protocol Specification
- Fill all MBAP header fields
- Prefix PDU with MBAP header to form ADU

**Request Transmission:**
- Pass ADU to TCP management
- Pass destination IP address
- TCP management selects appropriate socket

### Example Request Encoding

**Read register #5 from remote server:**

| Field | Size | Example Value |
|-------|------|---------------|
| Transaction ID Hi | 1 byte | 0x15 |
| Transaction ID Lo | 1 byte | 0x01 |
| Protocol Identifier | 2 bytes | 0x0000 |
| Length | 2 bytes | 0x0006 |
| Unit Identifier | 1 byte | 0xFF |
| Function Code | 1 byte | 0x03 (Read Holding Registers) |
| Starting Address | 2 bytes | 0x0004 |
| Quantity of Registers | 2 bytes | 0x0001 |

**Total: 12 bytes**

## MODBUS Server Design

### Server Components

**MODBUS Server Module:**
- Wait for MODBUS requests on port 502
- Process requests
- Build responses based on device context

**MODBUS Backend Interface:**
- Interface to user application objects
- Four areas: input discrete, output discrete (coils), input registers, output registers
- Pre-mapping required between interface and application data

**Communication Application Layer:**
- Client and/or server MODBUS interfaces
- Backend interface for application access

### Request Processing

1. Read request from TCP socket
2. Validate MBAP header
3. Extract function code
4. Process request (read/write/other actions)
5. Build response (echo MBAP header)
6. Send response via TCP

## Related pages

- [MODBUS TCP](/wiki/concepts/modbus-tcp.md)
- [MBAP Header](/wiki/concepts/mbap-header.md)
- [TCP Connection Management](/wiki/concepts/tcp-connection-management.md)
- [Function Codes](/wiki/concepts/function-codes.md)
- [Summary of MODBUS Application Protocol Specification](/wiki/summaries/MODBUS/modbusprotocolspecification.md)

## Backlinks

------------

MODBUS is a trademark of the Modbus Organization, Inc.

All information on this page is derived by summarizing and analyzing solely the MODBUS specifications published by the Modbus Organization, Inc.  The specifications are copyright by the Modbus Organization.

By deriving this information solely from the specifications we hope to stay true to the specification.

No infringement is intended.
