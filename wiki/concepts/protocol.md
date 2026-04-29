---
title: Protocol
Summary: General protocol concepts including OSI model, request-response patterns, and communication principles used in MODBUS.
Sources:
  - raw/MODBUS/modbusprotocolspecification.md
  - raw/MODBUS/MODBUS.md
Categories:
  - overview
  - communication-model
  - architecture
type: concept
date-created: 2026-04-18T12:00:00+03:00
last-updated: 2026-04-18T16:00:00+03:00
---

This page explains the core ideas behind how MODBUS communication works (source: [modbusprotocolspecification.md](/raw/MODBUS/modbusprotocolspecification.md)). Understanding these concepts helps you build and troubleshoot MODBUS systems.

## MODBUS Works on Top of Other Communication Systems

MODBUS sits on top of whatever communication system you're using - whether that's Ethernet, serial cables, or something else (source: [modbusprotocolspecification.md](/raw/MODBUS/modbusprotocolspecification.md)). This is like how email works the same way whether you're on Wi-Fi, cellular, or a wired connection.

### Why This Separation Helps

- **Connection Flexibility:** Use MODBUS on Ethernet, RS-485 serial, RS-232 serial, and more
- **Reusable Messages:** The same MODBUS messages work on any connection type
- **Simpler Code:** You can handle connection details separately from MODBUS logic
- **Future-Proof:** New connection types can be added without changing MODBUS messages

### What Each Layer Does

The connection layer (Ethernet or serial) handles:
- **Making Connections:** Setting up and keeping communication channels open
- **Checking for Errors:** Making sure messages arrive correctly (CRC for serial, checksums for Ethernet)
- **Marking Message Boundaries:** Showing where one message ends and another begins
- **Controlling Flow:** Making sure messages don't arrive too fast

The MODBUS layer handles:
- **Operations:** What to do (read a value, write a value, etc.)
- **Data Organization:** How data is stored (coils, registers, etc.)
- **Error Messages:** Reporting when something goes wrong

## How MODBUS Devices Talk to Each Other

### The Question-and-Answer Pattern

MODBUS uses a question-and-answer pattern (called "client/server" or "master/slave") (source: [modbusprotocolspecification.md](/raw/MODBUS/modbusprotocolspecification.md)):

| Aspect | Client (Master) | Server (Slave) |
|--------|------------------|------------------|
| What it does | Asks questions | Answers questions |
| Who's in charge | Controls when to ask | Waits to be asked |
| Connections | Can talk to many servers | Waits for clients to talk to it |
| Addresses | One address | Unique address (1-247) |

This pattern provides:
- **Predictable Timing:** The client decides when things happen
- **No Conflicts:** Only one device asks at a time
- **Isolated Errors:** If one server has a problem, it doesn't break the client

### How a Conversation Works

Every MODBUS conversation follows these steps (source: [modbusprotocolspecification.md](/raw/MODBUS/modbusprotocolspecification.md)):

1. **Ask:** Client sends a message saying what it wants
2. **Process:** Server does the requested work
3. **Answer:** Server sends back the result or an error message
4. **Next:** Client can ask another question

**Important Rules:**
- **One at a Time:** On serial connections, wait for an answer before asking the next question
- **Independent Questions:** Each question stands alone (though Ethernet can handle multiple at once)
- **Always Answer:** The server always sends something back - data or an error
- **Clear Errors:** Problems are reported with specific error codes, not silence

## MODBUS Message Parts

### The Core Message (PDU)

The PDU is the actual message content that works on any connection type (source: [modbusprotocolspecification.md](/raw/MODBUS/modbusprotocolspecification.md)):

**What's in a PDU:**
```
[What to do: 1 byte][Additional details: 0-252 bytes]
```

**Types of Operations:**
- **ON/OFF Operations (0x01-0x06):** Read/write individual switches and relays
- **Number Operations (0x03-0x06):** Read/write measurements and settings
- **Device Info (0x07-0x11):** Get information about the device
- **Special (0x2B):** Custom operations specific to certain devices

See [Function Codes](/wiki/concepts/function-codes.md) for all available operations.

### The Complete Message (ADU)

The ADU adds extra information around the core message for the specific connection type:

**For Ethernet (MODBUS TCP):**
```
[Header info: 7 bytes][Core message: 1-253 bytes]
Total: 8-260 bytes
```

**For Serial Cables (RTU - binary):**
```
[Device address: 1 byte][Core message: 1-253 bytes][Error check: 2 bytes]
Plus: pause between messages (3.5 character times)
```

**For Serial Cables (ASCII - text):**
```
[Start ':'][Device address: 1 byte][Core message: 1-253 bytes][Error check: 1 byte][End: CR+LF]
```

See [MBAP Header](/wiki/concepts/mbap-header.md) for Ethernet header details and [CRC-16](/wiki/concepts/crc-16.md) for error checking.

## Communication Principles

### Deterministic Timing

MODBUS serial variants require precise timing for frame detection (source: [modbusprotocolspecification.md](/raw/MODBUS/modbusprotocolspecification.md)):

**Silent Interval (RTU):**
- Minimum: 3.5 character times at specified baud rate
- Purpose: Allow receivers to detect end of frame
- Requirement: Both transmitter and receiver must maintain timing

**Character Timeout (ASCII):**
- Maximum: 1 second between characters
- Purpose: Detect incomplete frames or communication errors

**Response Timeout:**
- Typical: 1-5 seconds for MODBUS responses
- Application-specific based on network conditions

### Addressing Schemes

MODBUS supports different addressing schemes depending on transport:

**TCP/IP Addressing:**
- **Unit ID:** 1 byte in MBAP header
- **Direct Connection:** 0xFF for TCP without gateways
- **Gateway Routing:** 1-247 for routing to serial devices

**Serial Addressing:**
- **Slave Address:** 1 byte in frame header
- **Broadcast:** Address 0 (no response expected)
- **Unicast:** Addresses 1-247 (single response expected)

### Broadcast Communication

MODBUS supports broadcast addressing (address 0) on serial networks (source: [modbusprotocolspecification.md](/raw/MODBUS/modbusprotocolspecification.md)):

**Characteristics:**
- **One-to-Many:** Single request reaches all devices
- **No Response:** Devices don't respond to broadcast
- **Use Cases:** Synchronization, configuration updates, status queries
- **Limitations:** No confirmation of receipt, potential for collisions

## Handling Problems

### Error Messages

MODBUS always sends an error message when something goes wrong - it never stays silent (source: [modbusprotocolspecification.md](/raw/MODBUS/modbusprotocolspecification.md)):

**What an error message looks like:**
```
[Operation code + error flag: 1 byte][What went wrong: 1 byte]
```

**Types of Errors:**
- **Don't Understand (0x01):** "I don't know how to do that operation"
- **Wrong Address (0x02):** "That address doesn't exist"
- **Bad Value (0x03):** "That value doesn't make sense"
- **Device Problem (0x04):** "Something's broken inside me"
- **Busy (0x06):** "I'm busy, try again later"

### Catching Corrupted Messages

Different MODBUS types check for corrupted messages in different ways (source: [modbusprotocolspecification.md](/raw/MODBUS/modbusprotocolspecification.md)):

| Connection Type | How It Checks | How Good | How It Works |
|---------|-----------------|--------------|----------------|
| MODBUS TCP | TCP checksum | Very good | Built into Ethernet |
| MODBUS RTU | CRC-16 | Very good | Complex math formula |
| MODBUS ASCII | LRC | Good enough | Simple addition check |

See [CRC-16](/wiki/concepts/crc-16.md) for how the CRC-16 calculation works.

## Performance Considerations

### Bandwidth Efficiency

MODBUS is designed for efficient use of limited bandwidth in industrial environments (source: [modbusprotocolspecification.md](/raw/MODBUS/modbusprotocolspecification.md)):

**Efficiency Features:**
- **Binary Encoding (RTU):** Compact representation, minimal overhead
- **Packed Data:** Multiple values in single request
- **Batch Operations:** Read/write multiple registers in one transaction
- **Minimal Headers:** Only 2-7 bytes of framing overhead

### Scalability Limits

MODBUS defines practical limits for scalability (source: [modbusprotocolspecification.md](/raw/MODBUS/modbusprotocolspecification.md)):

| Limit | Value | Impact |
|-------|--------|--------|
| Device Addresses | 1-247 | Maximum devices on network |
| Coils per Request | 2000 bits | Maximum read/write per request |
| Registers per Request | 125 registers | Maximum read/write per request |
| PDU Size | 253 bytes | Maximum application data |
| TCP ADU Size | 260 bytes | Maximum TCP message |

### Throughput Optimization

**Best Practices for Performance:**
- **Batch Reads:** Read multiple registers in single request when possible
- **Optimal Polling:** Balance between data freshness and network load
- **Connection Reuse:** Keep TCP connections open (avoid handshakes)
- **Appropriate Timeout:** Set timeouts based on network conditions
- **Selective Polling:** Only poll data that has changed or is critical

## Security Considerations

MODBUS implementations have varying security capabilities:

| Variant | Native Security | Recommendations |
|---------|----------------|-------------------|
| MODBUS TCP | None (clear text) | Use [MODBUS TCP Security](/wiki/concepts/modbus-tcp-security.md) for encryption |
| MODBUS RTU/ASCII | None | Physical security (isolated networks) |
| MODBUS/TCP Security | TLS + Authentication | Recommended for new deployments |

**Security Risks:**
- **Clear Text:** All data transmitted unencrypted
- **No Authentication:** Any device can communicate
- **Replay Attacks:** No protection against message replay
- **Sniffing:** Data can be intercepted on shared networks

## Related Information

- [MODBUS Protocol](/wiki/concepts/modbus.md) - Comprehensive MODBUS protocol overview
- [Function Codes](/wiki/concepts/function-codes.md) - Complete function code reference
- [MODBUS TCP](/wiki/concepts/modbus-tcp.md) - TCP/IP implementation details
- [MODBUS RTU](/wiki/concepts/modbus-rtu.md) - Serial RTU specifications
- [MODBUS ASCII](/wiki/concepts/modbus-ascii.md) - Serial ASCII specifications
- [MBAP Header](/wiki/concepts/mbap-header.md) - TCP/IP framing details

See also: [protocol-architecture](/wiki/concepts/protocol-architecture.md) for advanced protocol design concepts.

## Related pages

## Backlinks

- [CRC-16](/wiki/concepts/crc-16.md)
- [Function Codes](/wiki/concepts/function-codes.md)
- [MBAP Header](/wiki/concepts/mbap-header.md)
- [Master-Slave Architecture](/wiki/concepts/master-slave.md)
- [MODBUS](/wiki/concepts/modbus.md)
- [MODBUS ASCII](/wiki/concepts/modbus-ascii.md)
- [MODBUS Data Type Mapping](/wiki/concepts/modbus-data-type-mapping.md)
- [MODBUS RTU](/wiki/concepts/modbus-rtu.md)
- [MODBUS TCP](/wiki/concepts/modbus-tcp.md)
- [MODBUS TCP Security](/wiki/concepts/modbus-tcp-security.md)
- [MODBUS Usage and Applications](/wiki/concepts/modbus-usage-and-applications.md)

------------

MODBUS is a trademark of the Modbus Organization, Inc.

All information on this page is derived by summarizing and analyzing solely the MODBUS specifications published by the Modbus Organization, Inc.  The specifications are copyright by the Modbus Organization.

By deriving this information solely from the specifications we hope to stay true to the specification.

No infringement is intended.
