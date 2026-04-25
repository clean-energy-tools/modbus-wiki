---
title: What is MBAP?
Summary: Explanation of MBAP (MODBUS Application Protocol) header, its structure, purpose, and how it enables MODBUS communication over TCP/IP networks.
Sources:
  - /raw/MODBUS/MODBUS.md
  - /raw/MODBUS/messagingimplementationguide.md
Categories:
  - protocol-components
  - tcp-ip
  - fundamentals
type: answer
date-created: 2026-04-23T12:00:00+03:00
last-updated: 2026-04-23T12:00:00+03:00
---

MBAP stands for **MODBUS Application Protocol header**. It is a 7-byte header that identifies and manages MODBUS messages on TCP/IP networks, replacing the address field from MODBUS serial (source: [messagingimplementationguide.md](/raw/MODBUS/messagingimplementationguide.md)).

## Why MBAP Exists

MODBUS was originally designed for serial communication (RS-485/RS-232), which has very different characteristics than TCP/IP networks. The MBAP header solves several critical problems when adapting MODBUS to TCP/IP:

### Problem 1: Message Boundaries

**Serial MODBUS:** Uses silent intervals (3.5 character times) to mark message boundaries. When the line goes quiet, you know the message is complete.

**TCP/IP:** Stream-oriented protocol with no inherent message boundaries. Data can be split across multiple packets or combined into one packet.

**MBAP Solution:** The Length field tells you exactly how many bytes follow, allowing you to detect complete messages even if they span multiple TCP packets.

### Problem 2: Request/Response Matching

**Serial MODBUS:** Only one transaction at a time (master-slave model). No ambiguity about which response matches which request.

**TCP/IP:** Multiple requests can be outstanding on the same connection simultaneously.

**MBAP Solution:** The Transaction ID field allows matching responses to requests, enabling concurrent transactions on a single TCP connection.

### Problem 3: Gateway Routing

**Serial MODBUS:** Direct point-to-point communication using slave address.

**TCP/IP:** Often deployed with gateways that bridge between MODBUS TCP and serial MODBUS devices.

**MBAP Solution:** The Unit ID field allows routing through gateways to address specific serial devices behind the gateway.

## MBAP Header Structure

The MBAP header consists of four fields totaling 7 bytes (source: [MBAP Header](/wiki/concepts/mbap-header.md)):

| Offset | Field | Size | Value | Description |
|--------|-------|------|-------|-------------|
| 0 | Transaction ID | 2 bytes | Varies | Request/response pairing |
| 2 | Protocol ID | 2 bytes | 0x0000 | Always 0 for MODBUS |
| 4 | Length | 2 bytes | N+1 | Bytes following (Unit ID + PDU) |
| 6 | Unit ID | 1 byte | 0x01-0xFF | Device identifier |

All fields use **big-endian** (network byte order) encoding.

### Transaction ID (Bytes 0-1)

**Purpose:** Pairs requests with responses and enables concurrent transactions.

**How it works:**
1. Client assigns a unique Transaction ID to each request
2. Server echoes the same Transaction ID back in the response
3. Client matches the Transaction ID to find the pending request

**Example:**
```
Client sends Request A with Transaction ID = 0x0001
Client sends Request B with Transaction ID = 0x0002
Server responds to B first with Transaction ID = 0x0002
Server responds to A second with Transaction ID = 0x0001
Client correctly matches each response to its request
```

**Implementation:** Most implementations use a simple counter that increments for each request.

### Protocol ID (Bytes 2-3)

**Purpose:** Identifies the protocol type, allowing multiplexing of different protocols on the same port.

**Value:** Always **0x0000** for MODBUS protocol.

**Usage:**
- Client sets to 0x0000
- Server echoes it back
- Reserved for future protocol extensions

**Note:** While this field theoretically allows other protocols to share port 502, in practice MODBUS/TCP uses a dedicated port.

### Length (Bytes 4-5)

**Purpose:** Specifies the number of bytes following this field.

**Calculation:**
```
Length = 1 (Unit ID) + PDU_length
```

**Example:**
```
Read Holding Registers request:
  Unit ID: 1 byte
  Function Code: 1 byte
  Start Address: 2 bytes
  Quantity: 2 bytes
  PDU = 5 bytes
  Length = 1 + 5 = 6
```

**Why it's critical:**
- TCP is stream-oriented with no message boundaries
- Length field lets receiver know when complete message has arrived
- Messages can be split across multiple TCP packets
- Receiver can buffer partial messages until complete

### Unit ID (Byte 6)

**Purpose:** Routes messages through gateways to serial MODBUS devices.

**Values:**
- **0xFF** (255): Direct connection to TCP device (most common)
- **0x01-0xF7** (1-247): Gateway routing to serial slave with this address
- **0x00**: Reserved, not used
- **0xF8-0xFF**: Reserved

**Direct Connection Example:**
```
MODBUS TCP Client → MODBUS TCP Server (Unit ID = 0xFF)
```

**Gateway Example:**
```
MODBUS TCP Client → Gateway → RS-485 Bus → Slave #5
                   (Unit ID = 5)
```

The gateway receives the TCP message, extracts the Unit ID, and forwards the request to the serial slave with that address.

## Complete MODBUS TCP Message (ADU)

A complete MODBUS TCP Application Data Unit (ADU) consists of:

```
[MBAP Header (7 bytes)] + [MODBUS PDU (Function Code + Data)]
```

**Maximum size:** 260 bytes total
- MBAP header: 7 bytes
- PDU: 253 bytes maximum

**Example - Read Holding Registers:**

Request (12 bytes):
```
Offset  Field              Hex Value
------  -----              ---------
0-1     Transaction ID     00 01
2-3     Protocol ID        00 00
4-5     Length             00 06
6       Unit ID            01
7       Function Code      03
8-9     Start Address      00 00
10-11   Quantity           00 02
```

Response (13 bytes):
```
Offset  Field              Hex Value
------  -----              ---------
0-1     Transaction ID     00 01     (echoed)
2-3     Protocol ID        00 00     (echoed)
4-5     Length             00 07
6       Unit ID            01        (echoed)
7       Function Code      03        (echoed)
8       Byte Count         04
9-10    Register 0         00 0A     (value = 10)
11-12   Register 1         00 14     (value = 20)
```

## How MBAP Enables TCP Features

### Concurrent Transactions

Unlike serial MODBUS which processes one request at a time, MODBUS TCP can handle multiple outstanding requests on the same connection:

```
Client:    Send Req A (TID=1) → Send Req B (TID=2) → Send Req C (TID=3)
           Wait for responses...
           ← Recv Resp B (TID=2)
           ← Recv Resp A (TID=1)
           ← Recv Resp C (TID=3)
```

The Transaction ID allows correct pairing even when responses arrive out of order.

### Message Framing Over TCP

TCP is a byte stream without message boundaries. The Length field solves this:

**Scenario 1 - Message split across packets:**
```
TCP Packet 1: [MBAP Header (7 bytes)] [Function Code (1 byte)]
TCP Packet 2: [Start Address (2 bytes)] [Quantity (2 bytes)]

Receiver uses Length=6 to know it needs 6 bytes after the header,
waits for second packet before processing complete message.
```

**Scenario 2 - Multiple messages in one packet:**
```
TCP Packet: [Message 1 (complete)] [Message 2 (complete)]

Receiver uses Length field to know where first message ends
and second begins.
```

### Gateway Routing

The Unit ID enables protocol conversion:

```
[TCP Client] → [Gateway with IP 192.168.1.100] → [RS-485 Bus]
                                                      ↓
                                              [Slave 1, 2, 3...]

Client sends to 192.168.1.100 with Unit ID = 2
Gateway forwards to Slave #2 on serial bus
Gateway translates MODBUS TCP ↔ MODBUS RTU
```

## Differences from MODBUS Serial

| Aspect | MODBUS Serial | MODBUS TCP (MBAP) |
|--------|---------------|-------------------|
| **Addressing** | 1-byte slave address | 1-byte Unit ID (for gateway) |
| **Transaction tracking** | One at a time | Transaction ID enables concurrent requests |
| **Message boundaries** | Silent intervals (3.5 char) | Length field |
| **Error checking** | CRC-16 (RTU) or LRC (ASCII) | TCP/IP checksum (inherent) |
| **Framing** | Timing-based | Length-based |
| **Concurrent requests** | Not supported | Supported |

## Implementation Guidelines

### Building a Request

1. **Assign Transaction ID:** Use counter or random value (must be unique per connection)
2. **Set Protocol ID:** Always 0x0000
3. **Calculate Length:** Count Unit ID + PDU bytes
4. **Set Unit ID:** 0xFF for direct, or slave address for gateway
5. **Append PDU:** Function code + request data
6. **Send as single TCP write:** Don't split MBAP header and PDU across separate sends

### Processing a Response

1. **Read MBAP header first:** Always read 7 bytes
2. **Validate Protocol ID:** Must be 0x0000
3. **Extract Transaction ID:** Use to find matching request
4. **Read remaining bytes:** Use Length field to determine how many bytes follow
5. **Process PDU:** Once complete message received

### Transaction ID Management

**Simple counter approach:**
```c
uint16_t next_transaction_id = 1;

uint16_t allocate_transaction_id() {
    uint16_t id = next_transaction_id++;
    if (id == 0) id = next_transaction_id++;  // Skip 0
    return id;
}
```

**Concurrent transaction tracking:**
```c
struct pending_transaction {
    uint16_t transaction_id;
    timestamp_t sent_time;
    callback_fn response_handler;
};

// Store in hash map or array
// Look up by transaction_id when response arrives
```

## Common Mistakes

### Mistake 1: Wrong Length Calculation

**Wrong:**
```
Length = PDU_length  // Missing Unit ID byte
```

**Correct:**
```
Length = 1 + PDU_length  // Must include Unit ID
```

### Mistake 2: Reusing Transaction IDs Too Soon

**Wrong:**
```
Send request with TID=1
Send request with TID=1  // COLLISION - previous request still pending!
```

**Correct:**
```
Send request with TID=1
Send request with TID=2  // Different ID while TID=1 is outstanding
```

### Mistake 3: Batching Multiple ADUs in One TCP Write

**Wrong:**
```
tcp_send([ADU1][ADU2][ADU3])  // Multiple ADUs in one send
```

**Correct:**
```
tcp_send([ADU1])
tcp_send([ADU2])
tcp_send([ADU3])
// One ADU per TCP write operation
```

Source: [messagingimplementationguide.md](/raw/MODBUS/messagingimplementationguide.md)

### Mistake 4: Not Handling Partial Reads

**Wrong:**
```
read(socket, buffer, 12);  // Assume full message arrives
```

**Correct:**
```
// Read MBAP header first
read(socket, header, 7);
length = decode_length(header);

// Read remaining bytes
read(socket, pdu, length);
```

## Security Considerations

MBAP itself provides **no security**:
- No encryption - all data visible in plaintext
- No authentication - any client can connect
- No integrity protection beyond TCP checksum

**Wireshark can easily capture and decode MODBUS TCP traffic** because it's unencrypted.

For secure communication, use [MODBUS TCP Security](/wiki/concepts/modbus-tcp-security.md) which:
- Runs on port 802 (not 502)
- Wraps MODBUS TCP in TLS 1.2+
- Provides encryption, authentication, and authorization
- Uses the same MBAP structure inside TLS tunnel

## Related Pages

- [MODBUS TCP](/wiki/concepts/modbus-tcp.md) - Complete MODBUS TCP protocol
- [MBAP Header](/wiki/concepts/mbap-header.md) - Detailed MBAP header specification
- [TCP Connection Management](/wiki/concepts/tcp-connection-management.md) - Managing TCP connections
- [MODBUS TCP Security](/wiki/concepts/modbus-tcp-security.md) - Secure variant using TLS

## Backlinks

None yet.
