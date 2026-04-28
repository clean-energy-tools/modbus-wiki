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

MBAP stands for **MODBUS Application Protocol header**. It's a 7-byte package of information attached to the front of every MODBUS message when using TCP/IP networks. Think of it like a shipping label on a package - it helps the message get to the right place and lets you track it (source: [messagingimplementationguide.md](/raw/MODBUS/messagingimplementationguide.md)).

## Why MBAP is Needed

MODBUS started as a protocol for serial cables (like RS-485 and RS-232). Those cables work very differently from modern Ethernet networks. When engineers adapted MODBUS to work over TCP/IP networks, they needed to solve three problems:

### Problem 1: Knowing When a Message is Complete

**With serial cables:** When data stops arriving for a brief moment (3.5 character transmission times), you know the message is finished. The silence marks the boundary.

**With TCP/IP:** Data flows continuously like a stream of water. There are no natural breaks between messages. A single message might arrive split across multiple network packets, or several messages might arrive bunched together in one packet.

**MBAP's solution:** The header includes a Length field that states exactly how many bytes belong to this message. This lets the receiver know when it has received a complete message (source: [messagingimplementationguide.md](/raw/MODBUS/messagingimplementationguide.md)).

### Problem 2: Matching Responses to Requests

**With serial cables:** Only one conversation happens at a time. The master sends a request, waits for the response, then sends the next request. There's no confusion about which response belongs to which request.

**With TCP/IP:** You can send multiple requests without waiting for responses. This is faster but creates a matching problem - when responses arrive (possibly in a different order), how do you know which request each response answers?

**MBAP's solution:** Each request gets a unique Transaction ID number. The server copies this number into its response. When the response arrives, the client checks the Transaction ID to find the matching request (source: [messagingimplementationguide.md](/raw/MODBUS/messagingimplementationguide.md)).

### Problem 3: Routing Through Gateways

**With serial cables:** Messages include a slave address that identifies which device on the cable should respond.

**With TCP/IP:** Organizations often use gateway devices that connect TCP/IP networks to older serial MODBUS devices. A single gateway might connect to multiple serial devices.

**MBAP's solution:** The header includes a Unit ID field that tells the gateway which serial device should receive the message. When connecting directly to a TCP device (no gateway), this field is typically set to 255 (source: [messagingimplementationguide.md](/raw/MODBUS/messagingimplementationguide.md)).

## What's Inside the MBAP Header

The MBAP header contains four pieces of information packed into 7 bytes (source: [MBAP Header](/wiki/concepts/mbap-header.md)):

| Position | Field | Size | Value | What It Does |
|--------|-------|------|-------|-------------|
| Bytes 0-1 | Transaction ID | 2 bytes | Varies | Matches requests with responses |
| Bytes 2-3 | Protocol ID | 2 bytes | 0x0000 | Always 0 for MODBUS |
| Bytes 4-5 | Length | 2 bytes | N+1 | How many bytes follow this field |
| Byte 6 | Unit ID | 1 byte | 0x01-0xFF | Which device to talk to |

All multi-byte numbers use **big-endian** format, which means the most significant byte comes first. This is also called network byte order (source: [MBAP Header](/wiki/concepts/mbap-header.md)).

### Transaction ID (Bytes 0-1)

**What it does:** Acts like a claim ticket at a coat check - it pairs each request with its matching response.

**How it works:**
1. The client assigns a unique number to each request it sends
2. The server copies this same number into its response
3. When the response arrives, the client checks the number to find which request it answers

**Example:**
```
Client sends Request A with Transaction ID = 1
Client sends Request B with Transaction ID = 2
Server finishes B first and sends response with Transaction ID = 2
Server finishes A second and sends response with Transaction ID = 1
Client uses the numbers to correctly match each response to its request
```

Most programs use a simple counter that increases by one for each new request (source: [MBAP Header](/wiki/concepts/mbap-header.md)).

### Protocol ID (Bytes 2-3)

**What it does:** Identifies which protocol is being used. This field was designed to allow multiple communication protocols to share the same network port.

**Value:** Always **0x0000** for MODBUS.

**How it's used:**
- The client sets it to 0x0000
- The server copies the same value back in its response
- The field is reserved for possible future use

In practice, MODBUS uses its own dedicated port (port 502), so this field always contains zero (source: [MBAP Header](/wiki/concepts/mbap-header.md)).

### Length (Bytes 4-5)

**What it does:** Tells the receiver how many bytes of data follow this field.

**How to calculate it:**
```
Length = 1 (for the Unit ID byte) + size of the MODBUS command and data
```

**Example:**
```
For a "Read Holding Registers" request:
  Unit ID: 1 byte
  Function Code: 1 byte
  Start Address: 2 bytes
  Quantity: 2 bytes
  Command and data = 5 bytes
  Length = 1 + 5 = 6
```

**Why this matters:**
- TCP sends data as a continuous stream without natural breaks between messages
- The Length field tells the receiver when it has received a complete message
- Messages might arrive split across multiple network packets
- The receiver can wait until all bytes arrive before processing the message

(source: [MBAP Header](/wiki/concepts/mbap-header.md))

### Unit ID (Byte 6)

**What it does:** Identifies which device should handle the message. This is mainly useful when routing messages through gateway devices.

**Common values:**
- **255 (0xFF)**: Talking directly to a TCP device - the most common case
- **1-247 (0x01-0xF7)**: Routing through a gateway to a specific serial device
- **0 (0x00)**: Reserved, not used
- **248-254 (0xF8-0xFE)**: Reserved for future use

**Example - Direct connection:**
```
MODBUS TCP Client → MODBUS TCP Server (Unit ID = 255)
```

**Example - Through a gateway:**
```
MODBUS TCP Client → Gateway → RS-485 Cable → Device #5
                   (Unit ID = 5)
```

When using a gateway, the gateway reads the Unit ID, then forwards the message to the serial device that has that address (source: [MBAP Header](/wiki/concepts/mbap-header.md)).

## A Complete MODBUS TCP Message

A complete MODBUS TCP message (called an Application Data Unit or ADU) has two parts:

```
[MBAP Header (7 bytes)] + [MODBUS Command and Data]
```

**Size limits:** 260 bytes total
- MBAP header: 7 bytes
- Command and data: 253 bytes maximum

(source: [MBAP Header](/wiki/concepts/mbap-header.md))

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

## How MBAP Enables Multiple Requests at Once

With serial cables, MODBUS handles one request at a time. MODBUS TCP can handle multiple requests simultaneously on the same connection:

```
Client:    Send Request A (ID=1) → Send Request B (ID=2) → Send Request C (ID=3)
           Wait for responses...
           ← Receive Response B (ID=2)
           ← Receive Response A (ID=1)
           ← Receive Response C (ID=3)
```

The Transaction ID numbers let the client correctly match each response to its request, even when responses arrive in a different order than the requests were sent (source: [messagingimplementationguide.md](/raw/MODBUS/messagingimplementationguide.md)).

## How MBAP Handles Message Boundaries

TCP sends data as a continuous stream. The Length field solves the boundary problem:

**Scenario 1 - Message split across network packets:**
```
Network Packet 1: [MBAP Header (7 bytes)] [Function Code (1 byte)]
Network Packet 2: [Start Address (2 bytes)] [Quantity (2 bytes)]

The receiver reads Length=6, knows it needs 6 more bytes after the header,
and waits for the second packet before processing the complete message.
```

**Scenario 2 - Multiple messages in one network packet:**
```
Network Packet: [Complete Message 1] [Complete Message 2]

The receiver uses the Length field to find where the first message ends
and the second message begins.
```

(source: [messagingimplementationguide.md](/raw/MODBUS/messagingimplementationguide.md))

## How MBAP Enables Gateway Routing

The Unit ID field lets gateways connect TCP networks to serial MODBUS devices:

```
[TCP Client] → [Gateway at 192.168.1.100] → [RS-485 Cable]
                                                  ↓
                                        [Device 1, Device 2, Device 3...]

Client sends to 192.168.1.100 with Unit ID = 2
Gateway forwards the message to Device #2 on the serial cable
Gateway converts between MODBUS TCP and MODBUS RTU formats
```

(source: [messagingimplementationguide.md](/raw/MODBUS/messagingimplementationguide.md))

## How MBAP Differs from Serial MODBUS

| Feature | MODBUS Serial | MODBUS TCP (with MBAP) |
|--------|---------------|-------------------|
| **Device addressing** | 1-byte slave address | 1-byte Unit ID (mainly for gateways) |
| **Multiple requests** | One request at a time | Can send multiple requests without waiting |
| **Finding message boundaries** | Silence on the line (3.5 character times) | Length field in header |
| **Error detection** | CRC checksum (RTU) or LRC checksum (ASCII) | TCP's built-in checksum |
| **Message framing** | Based on timing | Based on Length field |

(source: [messagingimplementationguide.md](/raw/MODBUS/messagingimplementationguide.md))

## How to Build and Process MBAP Messages

### Building a Request

When creating a MODBUS TCP request:

1. **Assign a Transaction ID:** Use a counter or pick a unique number for this request
2. **Set Protocol ID:** Always use 0x0000
3. **Calculate Length:** Add 1 (for Unit ID) plus the size of your command and data
4. **Set Unit ID:** Use 255 for direct connections, or the device number for gateways
5. **Add your command:** Include the function code and request data
6. **Send the complete message:** Send the header and data together in one operation

### Processing a Response

When receiving a MODBUS TCP response:

1. **Read the MBAP header:** Always read 7 bytes first
2. **Check Protocol ID:** It must be 0x0000
3. **Read the Transaction ID:** Use it to find which request this response answers
4. **Read the rest:** Use the Length field to know how many more bytes to read
5. **Process the data:** Once you have the complete message

### Managing Transaction IDs

**Simple counter approach:**
```c
uint16_t next_transaction_id = 1;

uint16_t allocate_transaction_id() {
    uint16_t id = next_transaction_id++;
    if (id == 0) id = next_transaction_id++;  // Skip 0
    return id;
}
```

**Tracking multiple requests:**
```c
struct pending_transaction {
    uint16_t transaction_id;
    timestamp_t sent_time;
    callback_fn response_handler;
};

// Store these in a lookup table (hash map or array)
// Find the matching request when a response arrives
```

(source: [messagingimplementationguide.md](/raw/MODBUS/messagingimplementationguide.md))

## Common Mistakes to Avoid

### Mistake 1: Calculating Length Wrong

**Wrong approach:**
```
Length = size of command and data  // Forgot to include the Unit ID byte
```

**Correct approach:**
```
Length = 1 + size of command and data  // Must add 1 for the Unit ID
```

### Mistake 2: Reusing Transaction IDs Too Quickly

**Wrong approach:**
```
Send request with ID=1
Send another request with ID=1  // Problem! The first request hasn't been answered yet
```

**Correct approach:**
```
Send request with ID=1
Send next request with ID=2  // Use a different ID while the first is still waiting
```

### Mistake 3: Sending Multiple Messages Together

**Wrong approach:**
```
tcp_send([Message1][Message2][Message3])  // All at once
```

**Correct approach:**
```
tcp_send([Message1])
tcp_send([Message2])
tcp_send([Message3])
// Send each message separately
```

(source: [messagingimplementationguide.md](/raw/MODBUS/messagingimplementationguide.md))

### Mistake 4: Assuming Complete Messages Arrive at Once

**Wrong approach:**
```
read(socket, buffer, 12);  // Assumes the full message arrives in one read
```

**Correct approach:**
```
// Read the MBAP header first
read(socket, header, 7);
length = decode_length(header);

// Then read the remaining bytes
read(socket, command_and_data, length);
```

(source: [messagingimplementationguide.md](/raw/MODBUS/messagingimplementationguide.md))

## Security Considerations

MBAP provides **no security features**:
- No encryption - anyone can read the data as it travels across the network
- No authentication - any client can connect and send commands
- No tamper protection beyond TCP's basic checksum

Network analysis tools like Wireshark can capture and decode MODBUS TCP traffic because the data is sent in plain text (source: [messagingimplementationguide.md](/raw/MODBUS/messagingimplementationguide.md)).

For secure communication, use [MODBUS TCP Security](/wiki/concepts/modbus-tcp-security.md), which:
- Uses port 802 instead of port 502
- Wraps MODBUS messages in TLS encryption (version 1.2 or later)
- Adds authentication and authorization
- Uses the same MBAP structure, but inside an encrypted tunnel

## Related Pages

- [MODBUS TCP](/wiki/concepts/modbus-tcp.md) - Complete MODBUS TCP protocol
- [MBAP Header](/wiki/concepts/mbap-header.md) - Detailed MBAP header specification
- [TCP Connection Management](/wiki/concepts/tcp-connection-management.md) - Managing TCP connections
- [MODBUS TCP Security](/wiki/concepts/modbus-tcp-security.md) - Secure variant using TLS

## Backlinks

- [Connecting with MODBUS/TCP Security](/wiki/answers/connecting-with-modbus-tcp-security.md)
- [MODBUS TCP Message Format](/wiki/answers/modbus-tcp-message-format.md)
- [MBAP Header](/wiki/concepts/mbap-header.md)
- [MODBUS TCP](/wiki/concepts/modbus-tcp.md)
- [MODBUS TCP Security](/wiki/concepts/modbus-tcp-security.md)
- [TCP Connection Management](/wiki/concepts/tcp-connection-management.md)

------------

MODBUS is a trademark of the Modbus Organization, Inc.

All information on this page is derived by summarizing and analyzing solely the MODBUS specifications published by the Modbus Organization, Inc.  The specifications are copyright by the Modbus Organization.

By deriving this information solely from the specifications we hope to stay true to the specification.

No infringement is intended.
