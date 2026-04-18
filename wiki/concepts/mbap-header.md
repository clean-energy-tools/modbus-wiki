---
title: MBAP Header
Summary: MODBUS Application Protocol header for TCP/IP encapsulation, providing transaction management and routing information.
Sources:
  - raw/MODBUS/MODBUS.md
  - raw/MODBUS/messagingimplementationguide.md
Categories:
  - protocol-components
  - tcp-ip
  - framing
date-created: 2026-04-18T12:00:00+03:00
last-updated: 2026-04-18T14:43:24+03:00
---

The MBAP (MODBUS Application Protocol) header is a 7-byte header that identifies and manages MODBUS messages on TCP/IP networks, replacing the address field from MODBUS serial (source: [messagingimplementationguide.md](raw/MODBUS/messagingimplementationguide.md)).

## Overview

The MBAP header provides message boundaries, transaction pairing, and routing capabilities for MODBUS over TCP/IP, enabling reliable communication over connection-oriented networks (source: [messagingimplementationguide.md](raw/MODBUS/messagingimplementationguide.md)).

### Key Characteristics

| Property | Value |
|----------|-------|
| Size | 7 bytes |
| Byte order | Big-endian (network byte order) |
| Purpose | Transaction pairing, message boundaries, routing |
| Location | Prefix to MODBUS PDU in TCP ADU |

## Header Structure

The MBAP header consists of four fields (source: [messagingimplementationguide.md](raw/MODBUS/messagingimplementationguide.md)):

| Offset | Field | Size | Description | Direction |
|--------|-------|------|-------------|-----------|
| 0 | Transaction ID | 2 bytes | Request/response matching | Client → Server (echoed) |
| 2 | Protocol ID | 2 bytes | 0x0000 for MODBUS | Client → Server (echoed) |
| 4 | Length | 2 bytes | Number of following bytes (Unit ID + PDU) | Client (request) / Server (response) |
| 6 | Unit ID | 1 byte | Remote slave address for gateways | Client → Server (echoed) |

## Header Fields

### Transaction Identifier

**Size:** 2 bytes (big-endian)

**Purpose:**
- Request/response pairing
- Matching responses to requests
- Supporting concurrent transactions on same connection

**Usage:**
- Client assigns unique value to each request
- Server echoes same value back in response
- Client uses to match response to pending request

**Implementation strategies:**
- Simple counter (incremented for each request)
- Random value (with collision detection)
- Smart index/pointer to transaction context

**Example:**
```
Request:  Transaction ID = 0x0001
Response: Transaction ID = 0x0001 (echoed)
```

**Critical for concurrent transactions:**
- Multiple requests can be outstanding on same TCP connection
- Transaction ID must be unique at any time on each connection
- Allows parallel processing of MODBUS transactions

### Protocol Identifier

**Size:** 2 bytes (big-endian)

**Purpose:**
- Protocol multiplexing
- Identifying MODBUS protocol
- Allowing future protocol extensions

**Value:**
- 0x0000 for MODBUS protocol
- Reserved for future use

**Usage:**
- Client sets to 0x0000
- Server echoes same value back

**Note:** This field allows multiplexing other protocols on the same port, though MODBUS/TCP typically uses dedicated port 502.

### Length

**Size:** 2 bytes (big-endian)

**Purpose:**
- Byte count of following fields
- Message boundary detection
- Framing across TCP packet boundaries

**Value:**
- Number of bytes following Length field
- Includes Unit ID (1 byte) + PDU length

**Request:**
```
Length = 1 (Unit ID) + PDU_length
```

**Response:**
```
Length = 1 (Unit ID) + PDU_length
```

**Example:**
```
Request PDU length: 6 bytes (Read Holding Registers)
Unit ID: 1 byte
Length field = 7
```

**Critical for TCP:**
- TCP is stream-oriented (no message boundaries)
- Length field allows detection of message boundaries
- Messages can be split across multiple TCP packets
- Recipient can determine complete message size

### Unit Identifier

**Size:** 1 byte

**Purpose:**
- Routing through gateways
- Addressing MODBUS serial slaves via gateways
- Intra-system routing

**Values:**
- 0xFF: Direct connection (most common)
- 0x00: Reserved (not used)
- 0x01-0xF7: Gateway to MODBUS serial slave
- 0xF8-0xFF: Reserved

**Direct connection (0xFF):**
- MODBUS TCP device directly connected
- No gateway involved
- Most common usage

**Gateway routing:**
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

**Unit ID mapping:**
- Direct connection: Unit ID = 0xFF
- Via gateway to slave 1: Unit ID = 1
- Via gateway to slave 2: Unit ID = 2

## Complete MODBUS TCP ADU

**Structure:**
```
[Transaction ID: 2][Protocol ID: 2][Length: 2][Unit ID: 1][Function: 1][Data: N]
|<---------------- MBAP Header (7 bytes) --------------->|<----- PDU ------->|
```

**Total maximum size:** 260 bytes (7 header + 253 PDU)

**Byte order:** All fields in big-endian (network byte order)

## Message Flow

### Request Flow

1. **Client generates Transaction ID**
   - Assign unique value (e.g., increment counter)

2. **Client builds MBAP header**
   - Set Transaction ID to unique value
   - Set Protocol ID = 0x0000
   - Calculate Length = 1 + PDU_length
   - Set Unit ID (0xFF for direct, or slave address for gateway)

3. **Client builds PDU**
   - Function code
   - Request data

4. **Client sends complete ADU**
   - MBAP header + PDU
   - Send as single TCP write

### Response Flow

1. **Server receives ADU**
   - Read MBAP header first (7 bytes)
   - Validate Protocol ID = 0x0000

2. **Server extracts information**
   - Transaction ID: for response matching
   - Unit ID: for routing
   - Length: determines PDU size

3. **Server processes request**
   - Execute MODBUS operation
   - Generate response PDU

4. **Server builds response MBAP header**
   - Copy Transaction ID from request
   - Copy Protocol ID from request (0x0000)
   - Calculate Length = 1 + response_PDU_length
   - Copy Unit ID from request

5. **Server sends response**
   - MBAP header + response PDU
   - Send as single TCP write

### Client Response Matching

1. **Client receives response**
   - Read MBAP header first (7 bytes)
   - Extract Transaction ID

2. **Client finds pending transaction**
   - Match Transaction ID to outstanding request
   - Process response for matching transaction

3. **Client handles concurrent transactions**
   - Multiple requests with different Transaction IDs
   - Responses can arrive out of order
   - Transaction ID enables correct pairing

## Example: Read Holding Registers

**Scenario:** Read 2 registers starting at address 0 from Unit ID 1

### Request ADU (12 bytes)

| Offset | Field | Size | Value | Description |
|--------|-------|------|-------|-------------|
| 0 | Transaction ID Hi | 1 | 0x00 | Transaction ID high byte |
| 1 | Transaction ID Lo | 1 | 0x01 | Transaction ID low byte |
| 2 | Protocol ID Hi | 1 | 0x00 | Protocol ID high byte |
| 3 | Protocol ID Lo | 1 | 0x00 | Protocol ID low byte |
| 4 | Length Hi | 1 | 0x00 | Length high byte |
| 5 | Length Lo | 1 | 0x06 | Length = 6 bytes |
| 6 | Unit ID | 1 | 0x01 | Unit Identifier |
| 7 | Function Code | 1 | 0x03 | Read Holding Registers |
| 8 | Start Address Hi | 1 | 0x00 | Start address high byte |
| 9 | Start Address Lo | 1 | 0x00 | Start address low byte |
| 10 | Quantity Hi | 1 | 0x00 | Quantity high byte |
| 11 | Quantity Lo | 1 | 0x02 | Quantity = 2 registers |

**Hex dump:** `00 01 00 00 00 06 01 03 00 00 00 02`

### Response ADU (13 bytes - Success)

| Offset | Field | Size | Value | Description |
|--------|-------|------|-------|-------------|
| 0 | Transaction ID Hi | 1 | 0x00 | Echoed |
| 1 | Transaction ID Lo | 1 | 0x01 | Echoed |
| 2 | Protocol ID Hi | 1 | 0x00 | Echoed |
| 3 | Protocol ID Lo | 1 | 0x00 | Echoed |
| 4 | Length Hi | 1 | 0x00 | Length = 7 bytes |
| 5 | Length Lo | 1 | 0x07 | Echoed |
| 6 | Unit ID | 1 | 0x01 | Echoed |
| 7 | Function Code | 1 | 0x03 | Echoed |
| 8 | Byte Count | 1 | 0x04 | 4 bytes of data |
| 9 | Register 0 Hi | 1 | 0x00 | Register 0 value high |
| 10 | Register 0 Lo | 1 | 0x0A | Register 0 value = 10 |
| 11 | Register 1 Hi | 1 | 0x00 | Register 1 value high |
| 12 | Register 1 Lo | 1 | 0x14 | Register 1 value = 20 |

**Hex dump:** `00 01 00 00 00 07 01 03 04 00 0A 00 14`

### Response ADU (9 bytes - Error)

| Offset | Field | Size | Value | Description |
|--------|-------|------|-------|-------------|
| 0 | Transaction ID Hi | 1 | 0x00 | Echoed |
| 1 | Transaction ID Lo | 1 | 0x01 | Echoed |
| 2 | Protocol ID Hi | 1 | 0x00 | Echoed |
| 3 | Protocol ID Lo | 1 | 0x00 | Echoed |
| 4 | Length Hi | 1 | 0x00 | Length = 3 bytes |
| 5 | Length Lo | 1 | 0x03 | Echoed |
| 6 | Unit ID | 1 | 0x01 | Echoed |
| 7 | Function Code | 1 | 0x83 | Exception (0x03 + 0x80) |
| 8 | Exception Code | 1 | 0x02 | ILLEGAL DATA ADDRESS |

**Hex dump:** `00 01 00 00 00 03 01 83 02`

## Key Differences from MODBUS Serial

| Characteristic | MODBUS Serial | MODBUS TCP (MBAP) |
|---------------|---------------|-------------------|
| Address field | 1 byte slave address | 1 byte Unit ID (for gateways) |
| Transaction tracking | N/A (one transaction at a time) | Transaction ID (2 bytes) |
| Message boundaries | Silent intervals / delimiters | Length field |
| Framing | Character timing | TCP stream + Length field |
| Concurrent requests | Not supported | Supported (via Transaction ID) |
| Maximum size | 256 bytes (RTU), 513 chars (ASCII) | 260 bytes |

## Implementation Considerations

### Transaction ID Management

**Simple counter:**
```c
uint16_t next_transaction_id = 0;

uint16_t get_next_transaction_id() {
    return next_transaction_id++;
}
```

**With wrap-around handling:**
```c
uint16_t get_next_transaction_id() {
    uint16_t id = next_transaction_id++;
    if (id == 0xFFFF) next_transaction_id = 1;  // Skip 0
    return id;
}
```

**Concurrent transaction tracking:**
```c
typedef struct {
    uint16_t transaction_id;
    time_t timestamp;
    void *context;  // Application context
} pending_transaction;

pending_transaction *find_transaction(uint16_t id) {
    // Search pending transactions list
    // Return matching transaction or NULL
}
```

### MBAP Header Encoding

```c
void encode_mbap_header(uint8_t *header, uint16_t trans_id,
                       uint16_t length, uint8_t unit_id) {
    header[0] = (trans_id >> 8) & 0xFF;
    header[1] = trans_id & 0xFF;
    header[2] = 0x00;  // Protocol ID high
    header[3] = 0x00;  // Protocol ID low
    header[4] = (length >> 8) & 0xFF;
    header[5] = length & 0xFF;
    header[6] = unit_id;
}
```

### MBAP Header Decoding

```c
typedef struct {
    uint16_t transaction_id;
    uint16_t protocol_id;
    uint16_t length;
    uint8_t unit_id;
} mbap_header;

mbap_header decode_mbap_header(uint8_t *header) {
    mbap_header h;
    h.transaction_id = (header[0] << 8) | header[1];
    h.protocol_id = (header[2] << 8) | header[3];
    h.length = (header[4] << 8) | header[5];
    h.unit_id = header[6];
    return h;
}
```

## Related pages

- [[/wiki/concepts/modbus-tcp]]]
- [[/wiki/concepts/tcp-connection-management]]]
- [[/wiki/concepts/function-codes]]]
