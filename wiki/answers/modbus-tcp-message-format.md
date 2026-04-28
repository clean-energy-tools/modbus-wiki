---
title: MODBUS TCP Message Format
Summary: Complete description of MODBUS TCP message format including MBAP header structure, frame layout, field details, differences from MODBUS RTU, TCP error checking rationale, and connection management.
Sources:
  - /raw/MODBUS/MODBUS.md
  - /raw/MODBUS/modbusprotocolspecification.md
  - /raw/MODBUS/messagingimplementationguide.md
  - /raw/MODBUS/modbusoverserial.md
Categories:
  - tcp-ip
  - message-format
  - protocol-structure
type: answer
date-created: 2026-04-23T13:00:00+03:00
last-updated: 2026-04-23T13:00:00+03:00
---

This document describes the MODBUS TCP message format, including the MBAP header structure, complete message layout, what each field does, how it differs from MODBUS RTU, and why TCP error checking works the way it does.

## MODBUS TCP Message Structure Overview

A complete MODBUS TCP message (called an ADU - Application Data Unit) has two parts:

```
[MBAP Header: 7 bytes] + [MODBUS Command and Data: up to 253 bytes]
```

**Maximum MODBUS TCP message size:** 260 bytes total

Source: [modbus-tcp.md](/wiki/concepts/modbus-tcp.md:29)

## MBAP Header Structure

The MBAP (MODBUS Application Protocol) header is a 7-byte package of information that's unique to MODBUS TCP. It takes the place of the address and CRC checksum fields used in MODBUS serial protocols.

### Complete MBAP Header Layout

| Position | Field | Size | Possible Values | What It Does |
|--------|-------|------|-------------|-------------|
| Bytes 0-1 | Transaction ID | 2 bytes | 0x0000-0xFFFF | Pairs requests with responses |
| Bytes 2-3 | Protocol ID | 2 bytes | 0x0000 | Always 0 for MODBUS |
| Bytes 4-5 | Length | 2 bytes | 0x0002-0x00FE | How many bytes follow this field |
| Byte 6 | Unit ID | 1 byte | 0x01-0xFF | Which device (0xFF for direct connection) |

**Total MBAP header size:** 7 bytes

**Byte order:** All numbers use big-endian format (most significant byte first, also called network byte order)

Source: [mbap-header.md](/wiki/concepts/mbap-header.md:31)

## Complete MODBUS TCP Frame

### Visual Representation

```
┌─────────────────────────── MBAP Header (7 bytes) ──────────────────────────┬───── PDU (variable) ─────┐
│                                                                             │                          │
│ Transaction ID │ Protocol ID │   Length    │ Unit ID │ Function │         Data                        │
│    (2 bytes)   │  (2 bytes)  │  (2 bytes)  │(1 byte) │ (1 byte) │    (0-252 bytes)                    │
│                │             │             │         │          │                                     │
│  0x00 0x01     │  0x00 0x00  │  0x00 0x06  │  0x01   │   0x03   │   0x00 0x00 0x00 0x02               │
└────────────────┴─────────────┴─────────────┴─────────┴──────────┴─────────────────────────────────────┘
  Client assigns   Always 0000   = 1 + PDU    Target    Read Regs    Start=0    Qty=2
```

### Breakdown by Section

**MBAP Header (7 bytes):**
- Provides transaction management, routing, and framing
- Unique to MODBUS TCP (not present in serial protocols)

**MODBUS PDU (Protocol Data Unit):**
- Function Code (1 byte): Specifies operation (e.g., 0x03 = Read Holding Registers)
- Data (0-252 bytes): Request parameters or response values
- Maximum PDU size: 253 bytes

**Complete ADU:**
- MBAP Header (7 bytes) + PDU (up to 253 bytes) = Maximum 260 bytes

Source: [mbap-header.md](/wiki/concepts/mbap-header.md:166)

## Detailed Field Descriptions

### Field 1: Transaction Identifier (Bytes 0-1)

**What it does:** Pairs requests with responses and allows multiple requests at once

**Size:** 2 bytes (most significant byte first)

**Possible values:** 0x0000 to 0xFFFF (0 to 65,535)

**How it works:**
1. The client assigns a unique number (Transaction ID) to each request
2. The server copies the same Transaction ID into its response
3. The client uses the Transaction ID to figure out which request this response answers

**Important feature:** Lets you send multiple requests without waiting for responses

**Example:**
```
Client sends:    Request A (Transaction ID = 1)
Client sends:    Request B (Transaction ID = 2)
Server responds: Response B (Transaction ID = 2)
Server responds: Response A (Transaction ID = 1)
Client matches:  Transaction ID 2 → That's for Request B
Client matches:  Transaction ID 1 → That's for Request A
```

**How to implement:**
```c
// Simple counter approach
uint16_t next_transaction_id = 0;

uint16_t allocate_transaction_id() {
    return next_transaction_id++;
}
```

**Important rule:** On any TCP connection, you can't have two pending requests with the same Transaction ID.

Source: [mbap-header.md](/wiki/concepts/mbap-header.md:44)

### Field 2: Protocol Identifier (Bytes 2-3)

**Purpose:** Identifies the protocol type and allows multiplexing

**Size:** 2 bytes (big-endian)

**Value:** Always **0x0000** for MODBUS protocol

**How it works:**
1. Client sets to 0x0000 in request
2. Server validates it equals 0x0000
3. Server echoes 0x0000 back in response

**Why it exists:** Theoretically allows other protocols to share port 502, though in practice MODBUS TCP uses a dedicated port.

**Reserved for future use:** Other values are reserved for potential future protocol extensions.

Source: [mbap-header.md](/wiki/concepts/mbap-header.md:74)

### Field 3: Length (Bytes 4-5)

**Purpose:** Specifies the number of bytes that follow this field

**Size:** 2 bytes (big-endian)

**Value Range:** 0x0002 to 0x00FE (2 to 254 decimal)

**Calculation:**
```
Length = 1 (Unit ID) + PDU_length

Where PDU_length = 1 (Function Code) + Data_length
```

**Examples:**

**Read Holding Registers request:**
```
Unit ID:       1 byte
Function Code: 1 byte
Start Address: 2 bytes
Quantity:      2 bytes
─────────────────────
PDU =          5 bytes
Length =       1 + 5 = 6 bytes
```

**Read Holding Registers response (2 registers):**
```
Unit ID:       1 byte
Function Code: 1 byte
Byte Count:    1 byte
Register Data: 4 bytes (2 registers × 2 bytes)
─────────────────────
PDU =          6 bytes
Length =       1 + 6 = 7 bytes
```

**Why it's critical:**

TCP is a **stream-oriented** protocol with no inherent message boundaries. Data can be:
- Split across multiple TCP packets
- Combined into a single TCP packet
- Arrive with arbitrary byte boundaries

The Length field tells the receiver exactly how many bytes comprise the complete MODBUS message, enabling proper frame boundaries even when TCP packets don't align with MODBUS messages.

**Example - Split message:**
```
TCP Packet 1: [MBAP Header: 7 bytes] [Function Code: 1 byte]
TCP Packet 2: [Address: 2 bytes] [Quantity: 2 bytes]

Receiver reads Length = 6, knows to wait for 6 more bytes after Unit ID
before processing the complete message.
```

Source: [mbap-header.md](/wiki/concepts/mbap-header.md:93)

### Field 4: Unit Identifier (Byte 6)

**Purpose:** Routes messages through gateways to serial MODBUS devices

**Size:** 1 byte

**Value Range:**
- **0xFF (255)**: Direct connection to MODBUS TCP device (most common)
- **0x01-0xF7 (1-247)**: Gateway routing to MODBUS serial slave with this address
- **0x00**: Reserved, not used
- **0xF8-0xFE**: Reserved

**Direct Connection (Unit ID = 0xFF):**
```
MODBUS TCP Client ────→ MODBUS TCP Server
                        (Unit ID = 0xFF)
```

The server is directly connected via TCP/IP. This is the most common case.

**Gateway Routing (Unit ID = 1-247):**
```
MODBUS TCP Client ────→ Gateway (192.168.1.100) ────→ RS-485 Bus
                        Unit ID = 5                      │
                                                        │
                                                   ┌────┴────┬─────┬─────┐
                                                   │         │     │     │
                                                Slave 1   Slave 5  ...  Slave N
```

The gateway receives the MODBUS TCP message, extracts the Unit ID (5), and forwards the request to the serial slave device with address 5 on the RS-485 bus.

**How it works:**
1. Client sets Unit ID to target device address
2. Gateway receives TCP message
3. Gateway converts MODBUS TCP → MODBUS RTU
4. Gateway sends to serial slave with address = Unit ID
5. Slave responds on serial bus
6. Gateway converts MODBUS RTU → MODBUS TCP
7. Gateway sends TCP response back to client

Source: [mbap-header.md](/wiki/concepts/mbap-header.md:129)

## Complete Request/Response Examples

### Example 1: Read Holding Registers (Success)

**Scenario:** Read 2 holding registers starting at address 0 from Unit ID 1

#### Request (12 bytes total)

**Hex dump:**
```
00 01 00 00 00 06 01 03 00 00 00 02
```

**Breakdown:**
```
Byte  │ Field              │ Hex Value │ Decimal │ Description
──────┼────────────────────┼───────────┼─────────┼──────────────────────────
0-1   │ Transaction ID     │ 00 01     │ 1       │ Request identifier
2-3   │ Protocol ID        │ 00 00     │ 0       │ MODBUS protocol
4-5   │ Length             │ 00 06     │ 6       │ 6 bytes follow
6     │ Unit ID            │ 01        │ 1       │ Target device
7     │ Function Code      │ 03        │ 3       │ Read Holding Registers
8-9   │ Start Address      │ 00 00     │ 0       │ Start at register 0
10-11 │ Quantity           │ 00 02     │ 2       │ Read 2 registers
```

#### Response (13 bytes total - Success)

**Hex dump:**
```
00 01 00 00 00 07 01 03 04 00 0A 00 14
```

**Breakdown:**
```
Byte  │ Field              │ Hex Value │ Decimal │ Description
──────┼────────────────────┼───────────┼─────────┼──────────────────────────
0-1   │ Transaction ID     │ 00 01     │ 1       │ Echoed from request
2-3   │ Protocol ID        │ 00 00     │ 0       │ Echoed from request
4-5   │ Length             │ 00 07     │ 7       │ 7 bytes follow
6     │ Unit ID            │ 01        │ 1       │ Echoed from request
7     │ Function Code      │ 03        │ 3       │ Echoed from request
8     │ Byte Count         │ 04        │ 4       │ 4 bytes of register data
9-10  │ Register 0         │ 00 0A     │ 10      │ Value = 10
11-12 │ Register 1         │ 00 14     │ 20      │ Value = 20
```

**Note:** Server echoes Transaction ID, Protocol ID, and Unit ID from the request.

### Example 2: Read Holding Registers (Error)

**Scenario:** Same request, but address is invalid

#### Response (9 bytes total - Exception)

**Hex dump:**
```
00 01 00 00 00 03 01 83 02
```

**Breakdown:**
```
Byte  │ Field              │ Hex Value │ Decimal │ Description
──────┼────────────────────┼───────────┼─────────┼──────────────────────────
0-1   │ Transaction ID     │ 00 01     │ 1       │ Echoed from request
2-3   │ Protocol ID        │ 00 00     │ 0       │ Echoed from request
4-5   │ Length             │ 00 03     │ 3       │ 3 bytes follow
6     │ Unit ID            │ 01        │ 1       │ Echoed from request
7     │ Function Code      │ 83        │ 131     │ Exception (0x03 + 0x80)
8     │ Exception Code     │ 02        │ 2       │ ILLEGAL DATA ADDRESS
```

**Exception response indicators:**
- Function Code has MSB set: 0x83 = 0x03 + 0x80
- Exception Code provides error reason (0x02 = address out of range)

Source: [modbus-tcp.md](/wiki/concepts/modbus-tcp.md:110)

## Differences Between MODBUS TCP and MODBUS RTU

### Side-by-Side Comparison

| Aspect | MODBUS RTU | MODBUS TCP |
|--------|------------|------------|
| **Transport** | Serial (RS-485/RS-232) | TCP/IP (Ethernet) |
| **Port** | N/A (serial interface) | 502 (standard), 802 (secure) |
| **Framing** | Silent intervals (3.5 char times) | MBAP Length field |
| **Address field** | 1-byte slave address | 1-byte Unit ID (for gateways) |
| **Transaction tracking** | N/A (one at a time) | 2-byte Transaction ID |
| **Error checking** | CRC-16 (2 bytes) | TCP checksum (inherent) |
| **Maximum frame size** | 256 bytes | 260 bytes |
| **Concurrent requests** | Not supported | Supported (via Transaction ID) |
| **Byte order** | Big-endian (except CRC) | Big-endian (all fields) |
| **Connection model** | Master/Slave (point-to-point) | Client/Server (networked) |
| **Broadcast** | Supported (address 0) | Not applicable |

### Frame Structure Comparison

**MODBUS RTU Frame:**
```
[Silent 3.5 char] [Address: 1][Function: 1][Data: 0-252][CRC-16: 2] [Silent 3.5 char]
                  |<-------------- Frame (256 bytes max) ------------>|
```

**MODBUS TCP Frame:**
```
[Transaction ID: 2][Protocol ID: 2][Length: 2][Unit ID: 1][Function: 1][Data: 0-252]
|<----------------- MBAP Header (7 bytes) --------------->|<------- PDU ------->|
|<----------------------- Frame (260 bytes max) ------------------------->|
```

### Key Structural Differences

#### 1. Framing Method

**MODBUS RTU:**
- Uses **timing** for frame boundaries
- Silent interval of 3.5 character times marks frame start/end
- Receiver detects silence to identify message boundaries
- Requires precise timing implementation

**MODBUS TCP:**
- Uses **Length field** for frame boundaries
- Length field explicitly states how many bytes follow
- No timing requirements
- Works over stream-oriented TCP with arbitrary packet boundaries

#### 2. Error Checking

**MODBUS RTU:**
- **CRC-16** error checking (2 bytes)
- Calculated over Address, Function Code, and Data
- Polynomial: 0xA001
- Transmitted LSB first (little-endian) - **only field not big-endian!**
- Detects bit errors, corruption, noise on serial line

**MODBUS TCP:**
- **No CRC** in MODBUS frame
- Relies on TCP/IP checksum (16-bit ones' complement)
- TCP provides error detection at transport layer
- Ethernet provides error detection at link layer (CRC-32)
- **Why no CRC:** See dedicated section below

#### 3. Transaction Management

**MODBUS RTU:**
- Master sends one request at a time
- Must wait for response before next request
- No way to track multiple outstanding requests
- Single-threaded communication

**MODBUS TCP:**
- Client can send multiple requests without waiting
- Transaction ID pairs each response to its request
- Responses can arrive out of order
- Multi-threaded/concurrent communication supported

#### 4. Addressing

**MODBUS RTU:**
- **Slave Address** (1 byte)
- Physical slave device on serial bus
- Address 0 = broadcast (all slaves execute, none respond)
- Addresses 1-247 valid, 248-255 reserved

**MODBUS TCP:**
- **Unit ID** (1 byte)
- Primarily for gateway routing to serial slaves
- 0xFF (255) = direct TCP connection (most common)
- 1-247 = route through gateway to serial slave

Source: [modbus-tcp.md](/wiki/concepts/modbus-tcp.md:155) and [modbus-rtu.md](/wiki/concepts/modbus-rtu.md:48)

## Why No CRC in MODBUS TCP?

### The Question

MODBUS RTU includes a mandatory CRC-16 for error detection. Why doesn't MODBUS TCP include CRC?

### The Answer

MODBUS TCP relies on the **existing error detection mechanisms in the TCP/IP protocol stack**. Adding another CRC would be redundant and wasteful.

### TCP/IP Error Detection Layers

**1. Ethernet Layer (Layer 2):**
- Uses **CRC-32** (32-bit cyclic redundancy check)
- Detects bit errors in Ethernet frames
- Corrupted frames are silently dropped
- Detection capability: Very high (polynomial: 0x04C11DB7)

**2. IP Layer (Layer 3):**
- Uses **16-bit header checksum**
- Verifies IP header integrity
- Protects routing information
- Note: Does NOT cover IP payload (only header)

**3. TCP Layer (Layer 4):**
- Uses **16-bit TCP checksum** (ones' complement)
- Covers TCP header AND entire payload
- Mandatory in TCP
- Corrupted segments are silently dropped and retransmitted

**Combined Protection:**
```
┌─────────────────────────────────────────────────┐
│  MODBUS TCP Data (MBAP + PDU)                   │
├─────────────────────────────────────────────────┤
│  TCP Segment (includes 16-bit checksum)         │ ← TCP checksum covers MODBUS data
├─────────────────────────────────────────────────┤
│  IP Packet (includes 16-bit header checksum)    │ ← IP checksum covers IP header
├─────────────────────────────────────────────────┤
│  Ethernet Frame (includes 32-bit CRC)           │ ← CRC-32 covers entire frame
└─────────────────────────────────────────────────┘
```

### Reliability Comparison

| Aspect | MODBUS RTU (Serial) | MODBUS TCP (Ethernet) |
|--------|---------------------|----------------------|
| **Physical medium** | RS-485/RS-232 (noisy) | Twisted pair/fiber (clean) |
| **Error detection** | CRC-16 (MODBUS layer) | CRC-32 (Ethernet) + TCP checksum |
| **Error correction** | None (discard bad frames) | TCP automatic retransmission |
| **Bit error rate** | ~10^-5 to 10^-7 | ~10^-12 to 10^-15 |
| **Noise immunity** | Low (long cables, EMI) | High (switched network, shielded) |

### Why TCP/IP Error Detection is Sufficient

**1. Stronger Error Detection:**
- CRC-32 (Ethernet) is stronger than CRC-16 (MODBUS RTU)
- Detects all single-bit errors, all double-bit errors, all odd number of bit errors
- Detects all burst errors ≤ 32 bits

**2. Automatic Retransmission:**
- TCP automatically retransmits corrupted segments
- Application doesn't need to handle retransmission
- Provides reliable, in-order delivery

**3. Cleaner Physical Medium:**
- Ethernet physical layer (twisted pair, fiber) has much lower bit error rate than RS-485
- Switched networks eliminate collisions and crosstalk
- Shielded cables reduce electromagnetic interference

**4. Performance:**
- Calculating and checking CRC adds processing overhead
- With TCP/IP already providing error detection, additional CRC is wasteful
- MODBUS TCP is designed for higher throughput than RTU

### What If TCP Checksum Fails to Detect an Error?

While theoretically possible, this is **extremely rare**:

**Probability:** TCP's 16-bit checksum has approximately 1 in 65,536 chance of not detecting an error if corruption occurs.

**Multiple layers:** Even if TCP checksum misses an error:
- Ethernet CRC-32 likely caught it (frame already discarded)
- IP checksum may have caught header corruption
- MODBUS application layer may detect logical errors (e.g., invalid register addresses)

**In practice:** TCP/IP has been proven reliable for decades across billions of devices. The probability of undetected corruption reaching the MODBUS application is negligible.

### Could You Add CRC to MODBUS TCP?

**Technically yes**, but it's not part of the standard and would provide minimal benefit:

**Why not standardized:**
- Redundant with existing TCP/IP error detection
- Adds unnecessary processing overhead
- Increases message size (2 extra bytes)
- Incompatible with standard MODBUS TCP implementations

**When you might want extra protection:**
- Use **MODBUS/TCP Security** (port 802) which adds:
  - TLS encryption
  - TLS MAC (Message Authentication Code) - cryptographic integrity check
  - Much stronger than CRC-16

Source: [modbus-tcp.md](/wiki/concepts/modbus-tcp.md:31)

## Connection Management in MODBUS TCP

### Connection Model

MODBUS TCP uses a **client/server** model over TCP:

**Client (formerly "Master"):**
- Initiates connections to servers
- Sends MODBUS requests
- Receives MODBUS responses
- Can connect to multiple servers

**Server (formerly "Slave"):**
- Listens on port 502
- Accepts connections from multiple clients
- Processes requests
- Sends responses

### Connection Establishment

**Client Process:**
1. Create TCP socket
2. Connect to server IP address, port 502
3. Configure socket options (TCP_NODELAY, SO_KEEPALIVE)
4. Connection ready for MODBUS transactions

**Server Process:**
1. Create listening socket
2. Bind to port 502
3. Listen for incoming connections
4. Accept client connections
5. Spawn handler for each connection

**Example (BSD sockets):**
```c
// Client
int sockfd = socket(AF_INET, SOCK_STREAM, 0);
struct sockaddr_in server_addr;
server_addr.sin_family = AF_INET;
server_addr.sin_port = htons(502);  // Port 502
inet_pton(AF_INET, "192.168.1.100", &server_addr.sin_addr);
connect(sockfd, (struct sockaddr*)&server_addr, sizeof(server_addr));

// Configure socket options
int opt = 1;
setsockopt(sockfd, IPPROTO_TCP, TCP_NODELAY, &opt, sizeof(opt));
setsockopt(sockfd, SOL_SOCKET, SO_KEEPALIVE, &opt, sizeof(opt));
```

Source: [tcp-connection-management.md](/wiki/concepts/tcp-connection-management.md:74)

### Critical Socket Options

#### TCP_NODELAY (Disable Nagle's Algorithm)

**Purpose:** Send small packets immediately without waiting

**Why needed:** 
- Nagle's algorithm delays small packets to batch them (reduces overhead)
- MODBUS messages are often small (12-20 bytes)
- Delays are unacceptable for real-time control

**Usage:**
```c
int opt = 1;
setsockopt(sockfd, IPPROTO_TCP, TCP_NODELAY, &opt, sizeof(opt));
```

**Result:** MODBUS requests/responses sent immediately, no artificial delay

**Recommendation:** **Always enable** for MODBUS TCP

#### SO_KEEPALIVE (TCP Keep-Alive)

**Purpose:** Detect dead connections (crashed remote, network failure)

**How it works:**
- After idle period (default: 2 hours), send probe packet
- If no response, send additional probes (default: 8 probes at 75-second intervals)
- If all probes fail, declare connection dead

**Usage:**
```c
int opt = 1;
setsockopt(sockfd, SOL_SOCKET, SO_KEEPALIVE, &opt, sizeof(opt));
```

**Why needed:**
- Detects crashed servers/clients
- Prevents hung connections
- Automatic connection cleanup

**Recommendation:** **Always enable** for MODBUS TCP

#### SO_REUSEADDR (Port Reuse)

**Purpose:** Allow immediate port reuse after close (bypass TIME-WAIT)

**Why needed:**
- Server restart can fail if port still in TIME-WAIT state
- TIME-WAIT can last 1-4 minutes
- SO_REUSEADDR allows immediate reuse

**Usage:**
```c
int opt = 1;
setsockopt(sockfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
```

**Recommendation:** **Enable on server sockets**

Source: [tcp-connection-management.md](/wiki/concepts/tcp-connection-management.md:172)

### Connection Lifecycle Best Practices

#### DO: Keep Connections Open

**Good:**
```
Client → Server: Connect to port 502
Client → Server: Request 1
Client ← Server: Response 1
Client → Server: Request 2
Client ← Server: Response 2
... (many transactions on same connection)
Client → Server: Close connection
```

**Why:** Connection establishment has overhead (TCP 3-way handshake). Reusing connections is much more efficient.

#### DON'T: Open/Close Per Transaction

**Bad:**
```
Client → Server: Connect to port 502
Client → Server: Request 1
Client ← Server: Response 1
Client → Server: Close connection

Client → Server: Connect to port 502  (unnecessary overhead!)
Client → Server: Request 2
Client ← Server: Response 2
Client → Server: Close connection
```

**Why bad:** Wastes resources, adds latency, limits throughput.

#### DO: Support Concurrent Transactions

**Good:**
```
Client → Server: Request A (Transaction ID = 1)
Client → Server: Request B (Transaction ID = 2)
Client ← Server: Response B (Transaction ID = 2)
Client ← Server: Response A (Transaction ID = 1)
```

**Why:** Increases throughput, reduces latency for multiple operations.

**Requirement:** Each outstanding request must have unique Transaction ID on that connection.

#### DON'T: Batch Multiple ADUs in One TCP Send

**Bad:**
```c
uint8_t buffer[1000];
memcpy(buffer, request1, 12);
memcpy(buffer + 12, request2, 12);
send(sockfd, buffer, 24, 0);  // Two ADUs in one send
```

**Good:**
```c
send(sockfd, request1, 12, 0);  // One ADU per send
send(sockfd, request2, 12, 0);  // One ADU per send
```

**Why:** MODBUS specification requires one ADU per TCP write operation.

Source: [tcp-connection-management.md](/wiki/concepts/tcp-connection-management.md:367)

### Connection Timeout Management

#### Application Response Timeout

**Purpose:** Detect missing/delayed responses

**Typical value:** 1-5 seconds

**Implementation:** Application-level timer started when request sent

**Action on timeout:**
- Mark transaction as failed
- Retry request (optional)
- Report error to user

**Example:**
```c
// Send request
send(sockfd, request, request_len, 0);
start_timer(transaction_id, 5000);  // 5-second timeout

// Wait for response
if (wait_for_response(sockfd, &response, 5000) < 0) {
    // Timeout occurred
    handle_timeout(transaction_id);
}
```

#### TCP Keep-Alive Timeout

**Purpose:** Detect dead connections (crashed peer, network failure)

**Typical values:**
- Idle time before first probe: 2 hours (often reduced to 5-30 minutes)
- Probe interval: 75 seconds
- Maximum probes: 8

**Action:** TCP stack automatically closes connection if all probes fail

#### Connection Establishment Timeout

**Purpose:** Limit time spent trying to connect

**Typical value:** 10-75 seconds (platform-dependent)

**Action on timeout:** Connection attempt fails, application handles error

Source: [tcp-connection-management.md](/wiki/concepts/tcp-connection-management.md:391)

### Connection Pool Management

For applications with many MODBUS servers, implement connection pooling:

**Benefits:**
- Reuse existing connections
- Limit total connections (resource management)
- Prioritize critical devices
- Better performance

**Strategy:**
1. Maintain pool of open connections
2. Reuse connection if available
3. Create new connection if pool has capacity
4. Close oldest idle connection if pool full

**Two-tier pooling:**
- **Priority pool:** For critical devices (never closed on local initiative)
- **Non-priority pool:** For regular devices (close oldest when full)

Source: [tcp-connection-management.md](/wiki/concepts/tcp-connection-management.md:312)

### Connection Error Handling

**Connection failure:**
- Unable to establish TCP connection
- Check server availability, network connectivity
- Retry with exponential backoff

**Connection reset:**
- Remote closed connection unexpectedly
- Re-establish if needed
- Check server logs for reason

**Response timeout:**
- No response within timeout period
- Retry transaction
- Consider connection dead if multiple timeouts

**Exception response:**
- Server returned MODBUS exception (function code with MSB set)
- Parse exception code
- Handle according to error type (illegal address, illegal function, etc.)

Source: [tcp-connection-management.md](/wiki/concepts/tcp-connection-management.md:409)

## Summary Comparison Table

| Feature | MODBUS RTU | MODBUS TCP |
|---------|------------|------------|
| **Header** | 1-byte Address | 7-byte MBAP Header |
| **Framing** | Silent intervals (timing-based) | Length field (size-based) |
| **Error Detection** | CRC-16 (2 bytes) | TCP/IP checksum (no CRC) |
| **Transaction ID** | None (one at a time) | 2-byte Transaction ID |
| **Concurrent Requests** | No | Yes (via Transaction ID) |
| **Transport** | Serial (RS-485/RS-232) | TCP/IP (Ethernet) |
| **Port** | N/A (serial interface) | 502 (standard), 802 (secure) |
| **Connection Model** | Master/Slave | Client/Server |
| **Max Frame Size** | 256 bytes | 260 bytes |
| **Byte Order** | Big-endian (except CRC LSB first) | Big-endian (all fields) |
| **Broadcast** | Supported (address 0) | Not applicable |
| **Gateway Support** | N/A | Unit ID for serial gateway routing |

## Related Pages

- [MODBUS TCP](/wiki/concepts/modbus-tcp.md) - Complete MODBUS TCP protocol
- [MBAP Header](/wiki/concepts/mbap-header.md) - Detailed MBAP header specification
- [MODBUS RTU](/wiki/concepts/modbus-rtu.md) - MODBUS RTU protocol (serial)
- [TCP Connection Management](/wiki/concepts/tcp-connection-management.md) - TCP connection best practices
- [What is MBAP?](/wiki/answers/what-is-mbap.md) - MBAP header explained
- [CRC-16](/wiki/concepts/crc-16.md) - CRC-16 algorithm details

## Backlinks

- [How MODBUS TCP-to-RTU Gateways Work](/wiki/answers/modbus-tcp-to-rtu-gateway.md)
- [MODBUS Broadcast](/wiki/answers/modbus-broadcast.md)
- [MODBUS Errors and Exception Responses](/wiki/answers/modbus-errors-and-exceptions.md)
- [Using Wireshark for MODBUS TCP Analysis](/wiki/answers/wireshark-modbus-analysis.md)
- [CRC-16](/wiki/concepts/crc-16.md)
- [MBAP Header](/wiki/concepts/mbap-header.md)
- [MODBUS RTU](/wiki/concepts/modbus-rtu.md)
- [MODBUS TCP](/wiki/concepts/modbus-tcp.md)
- [TCP Connection Management](/wiki/concepts/tcp-connection-management.md)

------------

MODBUS is a trademark of the Modbus Organization, Inc.

All information on this page is derived by summarizing and analyzing solely the MODBUS specifications published by the Modbus Organization, Inc.  The specifications are copyright by the Modbus Organization.

By deriving this information solely from the specifications we hope to stay true to the specification.

No infringement is intended.
