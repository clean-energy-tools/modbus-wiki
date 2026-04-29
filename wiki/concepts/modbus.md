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
type: concept
date-created: 2026-04-18T12:00:00+03:00
last-updated: 2026-04-18T16:00:00+03:00
---

MODBUS is a messaging system that lets industrial devices talk to each other (source: [modbusprotocolspecification.md](/raw/MODBUS/modbusprotocolspecification.md)). Think of it like a common language that factory equipment, building controllers, and power systems use to share information. Modicon created it in 1979 for programmable logic controllers (PLCs), and it's still the most common way industrial devices communicate today (source: [MODBUS.md](/raw/MODBUS/MODBUS.md)).

## What MODBUS Does

MODBUS works the same way whether you're using network cables, serial cables, or wireless connections (source: [modbusprotocolspecification.md](/raw/MODBUS/modbusprotocolspecification.md)). It uses a simple request-and-response pattern: one device (the client or "master") asks questions, and other devices (the servers or "slaves") answer with data or error messages.

## What Makes MODBUS Special

| Feature | What It Means |
|-----------|-------------|
| How it works | One device asks, others answer (Client/Server) |
| Where it fits | Top layer - the actual messages devices send |
| Easy to use | Simple question-and-answer pattern |
| Free to use | Public documentation, no license fees |
| Works anywhere | TCP/IP networks, RS-485 serial, RS-232 serial |
| Scalability | Up to 247 devices on one network |
| Battle-tested | Over 45 years in factories worldwide (since 1979) |

## Where MODBUS Came From

In 1979, a company called Modicon (now part of Schneider Electric) created MODBUS to let their programmable logic controllers talk to sensors, switches, and other equipment (source: [MODBUS.md](/raw/MODBUS/MODBUS.md)).

Because Modicon published the specifications publicly and kept the design simple, other manufacturers started using MODBUS too. Today it's one of the most widely used industrial communication systems in the world.

## How MODBUS Communication Works

### The Client/Server Pattern

MODBUS uses a question-and-answer pattern (the original specifications call this "master/slave"):

- **Client (Master):** The device that asks questions and sends commands
- **Server (Slave):** Devices that answer questions and provide data
- **One-to-One:** A client talks to one server at a time
- **Broadcast:** A client can send the same message to all servers, but they won't answer back

The client is always in control - it decides when to ask questions and which server to talk to (source: [modbusprotocolspecification.md](/raw/MODBUS/modbusprotocolspecification.md)).

### Works on Different Connection Types

MODBUS messages work the same way whether you're using Ethernet cables, old-style serial cables, or other connection methods (source: [modbusprotocolspecification.md](/raw/MODBUS/modbusprotocolspecification.md)):

**Common Ways to Send MODBUS:**
- [MODBUS TCP](/wiki/concepts/modbus-tcp.md) - Over modern Ethernet/IP networks
- [MODBUS RTU](/wiki/concepts/modbus-rtu.md) - Over serial cables (binary format)
- [MODBUS ASCII](/wiki/concepts/modbus-ascii.md) - Over serial cables (text format)
- [MODBUS TCP Security](/wiki/concepts/modbus-tcp-security.md) - Encrypted version for security

This flexibility means you can use MODBUS in old factories with serial wiring and modern facilities with Ethernet networks using the same basic message format.

## Types of Data MODBUS Can Handle

MODBUS organizes data into four types, like four different filing cabinets (source: [modbusprotocolspecification.md](/raw/MODBUS/modbusprotocolspecification.md)):

| Data Type | Size | Can You Change It? | What It's For |
|-------|-------------|--------|-------------|
| [Coils](/wiki/concepts/coils.md) | 1 bit (ON/OFF) | Yes (Read/Write) | Controlling things like relays and outputs |
| [Discrete Inputs](/wiki/concepts/discrete-inputs.md) | 1 bit (ON/OFF) | No (Read-only) | Reading switches and digital sensors |
| [Input Registers](/wiki/concepts/input-registers.md) | 16 bits (number) | No (Read-only) | Reading measurements like temperature |
| [Holding Registers](/wiki/concepts/holding-registers.md) | 16 bits (number) | Yes (Read/Write) | Storing settings and setpoints |

This simple organization keeps things straightforward while still being flexible enough for most industrial uses.

## MODBUS Message Structure

MODBUS messages have two main parts (source: [modbusprotocolspecification.md](/raw/MODBUS/modbusprotocolspecification.md)):

### The Core Message (PDU)

The PDU is the actual message content that works the same way on any connection type:
- **Function Code:** 1 byte - tells what operation to do (like "read temperature")
- **Request Data:** 0-252 bytes - additional details for the request
- **Response Data:** 0-252 bytes - the answer to the request
- **Maximum Size:** 253 bytes total (1 + 252)

### The Complete Message (ADU)

The ADU wraps the core message with extra information needed for the specific connection type:

**For Ethernet (MODBUS TCP):**
- Header: 7 bytes (message ID, which device, message length)
- Core message: up to 253 bytes
- Total maximum: 260 bytes

**For Serial Cables (RTU/ASCII):**
- Device address: 1 byte (which device to talk to)
- Error check: 2 bytes (makes sure message isn't corrupted)
- Core message: variable length
- Timing gaps: pauses between messages to mark boundaries

See [MBAP Header](/wiki/concepts/mbap-header.md) for details about the Ethernet header.

## How a MODBUS Conversation Works

### The Request/Response Pattern

1. **Client Asks:** The client sends a request saying what it wants (like "read register 100")
2. **Server Works:** The server does the requested operation
3. **Server Answers:** The server sends back the data or an error message
4. **Client Uses Answer:** The client processes the answer or handles any error

This simple back-and-forth pattern makes MODBUS reliable and easy to troubleshoot.

### When Things Go Wrong

If a server can't do what the client asks, it sends back an error message (exception) instead of data (source: [modbusprotocolspecification.md](/raw/MODBUS/modbusprotocolspecification.md)).

Common error messages:
- 0x01: ILLEGAL FUNCTION - "I don't know how to do that"
- 0x02: ILLEGAL DATA ADDRESS - "That address doesn't exist here"
- 0x03: ILLEGAL DATA VALUE - "That value doesn't make sense"
- 0x04: SERVER DEVICE FAILURE - "Something broke inside me"

## How MODBUS Finds the Right Device

MODBUS uses different ways to identify devices depending on how they're connected:

**On Ethernet Networks:**
- Uses a Unit ID number in the message header
- 0xFF (255) means "device directly connected"
- 1-247 for reaching devices through a gateway

**On Serial Networks:**
- Uses a 1-byte address at the start of each message
- 0 means "everyone listen" (broadcast - no one answers back)
- 1-247 for talking to specific devices
- 248-255 are reserved and shouldn't be used

## What MODBUS Does Well and What It Doesn't

### Strengths

- **Simple:** Easy to learn, implement, and troubleshoot
- **Free:** Public documentation, no license costs
- **Widely Supported:** Thousands of devices from hundreds of manufacturers
- **Flexible Connections:** Works on Ethernet, serial cables, and more
- **Scalable:** Can handle networks with many devices
- **Proven:** Used successfully in factories for over 45 years

### Limitations

- **Speed:** Not fast enough for high-speed control (like catching a falling object)
- **Security:** Standard MODBUS has no encryption or passwords (see [MODBUS TCP Security](/wiki/concepts/modbus-tcp-security.md) for secure version)
- **Message Size:** Messages can't be bigger than 253 bytes
- **Timing:** Ethernet version doesn't guarantee exact timing (serial versions do)
- **Broadcasting:** Limited ability to send to multiple devices at once

## Where You'll Find MODBUS

MODBUS is used in many industries:
- [MODBUS Usage and Applications](/wiki/concepts/modbus-usage-and-applications.md) - Factories and manufacturing
- Building controls (heating, cooling, lighting)
- Solar panels, wind turbines, and power systems
- Water treatment plants and environmental monitoring
- Traffic signals and infrastructure
- Oil and gas pipelines

## Learn More

- [Function Codes](/wiki/concepts/function-codes.md) - All the operations MODBUS can do
- [MODBUS TCP](/wiki/concepts/modbus-tcp.md) - How MODBUS works on Ethernet
- [MODBUS RTU](/wiki/concepts/modbus-rtu.md) - How MODBUS works on serial cables (binary)
- [MODBUS ASCII](/wiki/concepts/modbus-ascii.md) - How MODBUS works on serial cables (text)
- [Summary of MODBUS Protocol Specification for AI Implementation](/wiki/summaries/MODBUS/MODBUS.md) - Complete technical reference

See also: [protocol-architecture](/wiki/concepts/protocol-architecture.md) for technical protocol concepts.

## Related pages

## Backlinks

No backlinks (yet)

## Backlinks

- [Coils](/wiki/concepts/coils.md)
- [Discrete Inputs](/wiki/concepts/discrete-inputs.md)
- [Function Codes](/wiki/concepts/function-codes.md)
- [Holding Registers](/wiki/concepts/holding-registers.md)
- [Input Registers](/wiki/concepts/input-registers.md)
- [MBAP Header](/wiki/concepts/mbap-header.md)
- [MODBUS ASCII](/wiki/concepts/modbus-ascii.md)
- [MODBUS Data Type Mapping](/wiki/concepts/modbus-data-type-mapping.md)
- [MODBUS RTU](/wiki/concepts/modbus-rtu.md)
- [MODBUS TCP](/wiki/concepts/modbus-tcp.md)
- [MODBUS TCP Security](/wiki/concepts/modbus-tcp-security.md)
- [MODBUS Usage and Applications](/wiki/concepts/modbus-usage-and-applications.md)
- [Implementation](/wiki/concepts/implementation.md)
- [Protocol](/wiki/concepts/protocol.md)
- [Summary of MODBUS](/wiki/summaries/MODBUS/MODBUS.md)

------------

MODBUS is a trademark of the Modbus Organization, Inc.

All information on this page is derived by summarizing and analyzing solely the MODBUS specifications published by the Modbus Organization, Inc.  The specifications are copyright by the Modbus Organization.

By deriving this information solely from the specifications we hope to stay true to the specification.

No infringement is intended.
