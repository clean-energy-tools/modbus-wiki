# MODBUS Protocol Wiki

A comprehensive knowledge base for the MODBUS protocol, for developers building MODBUS services or applications.  All information on this site is drawn directly from the MODBUS specifications.  The wiki content covers summaries of each MODBUS specification document, MODBUS concepts, and answers to many questions about MODBUS.

## Overview

MODBUS is an application-layer messaging protocol (OSI Layer 7) providing client/server communication between devices. Originally developed by Modicon in 1979 for programmable logic controllers, it remains the de facto standard for industrial device communication.

This repository transforms official MODBUS specification documents into an easily navigable wiki format, making it simple for software developers to:
- Understand MODBUS protocol fundamentals
- Learn transport variants (TCP, RTU, ASCII, Security)
- Access function code specifications
- Implement correct data models
- Follow best practices for industrial communication

## Repository Structure

```
modbus-ai-wiki/
├── README.md                          # This file
├── AGENTS.md                          # Ingestion workflow documentation
├── raw/                               # Source documents (immutable)
│   └── MODBUS/
│       ├── MODBUS.md                    # Comprehensive AI-optimized reference
│       ├── modbusprotocolspecification.md    # Official application protocol spec
│       ├── modbusoverserial.md              # Serial line specification
│       ├── messagingimplementationguide.md     # TCP/IP implementation guide
│       └── modbussecurityprotocol.md       # Security protocol spec
└── wiki/                              # Maintained wiki pages
    ├── index.md                      # Table of contents
    ├── log.md                        # Change log
    ├── summaries/                    # Document summaries
    │   └── MODBUS/
    ├── answers/                      # Answers to questions about MODBUS
    └── concepts/                     # Concept pages
```

## Quick Start

### For Software Developers

1. **Start with the Index:** Begin at [wiki/index.md](wiki/index.md) for an overview of all content
2. **Choose Your Transport:**
   - [MODBUS TCP](wiki/concepts/modbus-tcp.md) for Ethernet networks (most common)
   - [MODBUS RTU](wiki/concepts/modbus-rtu.md) for RS-485/RS-232 serial
   - [MODBUS ASCII](wiki/concepts/modbus-ascii.md) for human-readable serial communication
   - [MODBUS TCP Security](wiki/concepts/modbus-tcp-security.md) for secure TLS connections
3. **Understand Data Models:** Learn about [Coils](wiki/concepts/coils.md), [Discrete Inputs](wiki/concepts/discrete-inputs.md), [Holding Registers](wiki/concepts/holding-registers.md), and [Input Registers](wiki/concepts/input-registers.md)
4. **Reference Function Codes:** See [Function Codes](wiki/concepts/function-codes.md) for read/write operations
5. **Implement:** Use protocol details to build your MODBUS client or server

### For Protocol Researchers

1. **Read Source Summaries:** Review [wiki/summaries/MODBUS/](wiki/summaries/MODBUS/) for comprehensive document summaries
2. **Explore Concepts:** Browse [wiki/concepts/](wiki/concepts/) for detailed protocol concepts
3. **Check Sources:** All wiki pages cite their source documents for verification

## MODBUS Protocol Fundamentals

### Transport Variants

| Variant | Medium | Error Check | Typical Use |
|---------|--------|-------------|-------------|
| **MODBUS TCP** | Ethernet | TCP/IP | Networked industrial systems |
| **MODBUS RTU** | Serial (RS-485/232) | CRC-16 | Legacy industrial equipment |
| **MODBUS ASCII** | Serial | LRC | Debugging, human-readable |
| **MODBUS TCP Security** | Ethernet + TLS | TLS 1.2+ | Secure control networks |

### Data Model

MODBUS defines four primary data tables:

| Table | Object Type | Size | Access | Typical Use |
|-------|-------------|------|--------|-------------|
| **Coils** | Bit | 1 bit | Read/Write | Digital outputs, relays |
| **Discrete Inputs** | Bit | 1 bit | Read-only | Digital inputs, switches |
| **Holding Registers** | Word | 16 bits | Read/Write | Configuration, setpoints |
| **Input Registers** | Word | 16 bits | Read-only | Measurements, status |

### Key Characteristics

- **Architecture:** Client/Server (Master/Slave)
- **Data Encoding:** Big-endian (network byte order)
- **Max PDU Size:** 253 bytes
- **Default Ports:** 502 (standard), 802 (secure/TLS)
- **Addressing:** 0-based in PDU (despite 1-based documentation)

## Key Topics Covered

### Core Concepts

- **Transport Layers:** TCP/IP, Serial Line (RTU/ASCII), TLS Security
- **Data Model:** Coils, Discrete Inputs, Holding Registers, Input Registers
- **Function Codes:** 0x01-0x24 public codes for read/write operations
- **Error Handling:** Exception codes and response formats
- **Framing:** MBAP header, CRC-16, LRC, timing requirements

### Implementation Guidance

- **TCP Connection Management:** Establishment, pooling, socket options
- **Serial Line Configuration:** RS-485 physical layer, termination, biasing
- **Security:** TLS configuration, mutual authentication, role-based authorization
- **Error Detection:** CRC-16 algorithms, LRC calculation
- **Best Practices:** Connection handling, timeout management, error recovery

### Code Examples

Each concept page includes practical implementation examples:
- C code snippets for CRC-16 calculation
- Socket configuration for TCP connections
- Data encoding/decoding examples
- Error handling patterns

## Usage Scenarios

### Scenario 1: Building a MODBUS TCP Client

1. Read [MODBUS TCP](wiki/concepts/modbus-tcp.md) overview
2. Study [MBAP Header](wiki/concepts/mbap-header.md) structure
3. Review [TCP Connection Management](wiki/concepts/tcp-connection-management.md)
4. Reference [Function Codes](wiki/concepts/function-codes.md) for operations
5. Implement using provided code examples

### Scenario 2: Implementing MODBUS RTU over RS-485

1. Study [MODBUS RTU](wiki/concepts/modbus-rtu.md) frame structure
2. Learn [CRC-16](wiki/concepts/crc-16.md) calculation
3. Review RS-485 physical layer requirements in serial spec
4. Understand timing (t1.5, t3.5) and framing
5. Implement state machine for master/slave operation

### Scenario 3: Adding Security to Existing MODBUS/TCP

1. Read [MODBUS TCP Security](wiki/concepts/modbus-tcp-security.md) specification
2. Understand TLS requirements and cipher suites
3. Implement mutual authentication with x.509v3 certificates
4. Configure role-based authorization
5. Migrate from port 502 to port 802

## Source Documents

This wiki is based on official MODBUS Organization specifications:

1. **MODBUS Protocol Specification for AI Implementation**
   - Comprehensive technical reference optimized for AI implementation
   - Precision, byte-level specifications, and concrete examples

2. **MODBUS Application Protocol Specification**
   - Official specification defining communication model and data model
   - Function codes, exception handling, and protocol variants

3. **MODBUS Over Serial Line Specification**
   - Master-slave protocol for serial communication
   - RTU and ASCII transmission modes, RS-485 physical layer

4. **MODBUS Messaging on TCP/IP Implementation Guide**
   - TCP/IP implementation guidance
   - Connection management, BSD socket interface, client/server architecture

5. **MODBUS TCP Security Protocol Specification**
   - Secure MODBUS over TLS
   - Mutual authentication, role-based authorization, certificate management

All source documents are located in [`raw/MODBUS/`](raw/MODBUS/) and remain unmodified.

## Contributing

This wiki is generated from source specifications using an AI-assisted ingestion workflow. When adding new MODBUS specifications:

1. Place source documents in `raw/MODBUS/`
2. Follow the ingestion workflow defined in [`AGENTS.md`](AGENTS.md)
3. Create summary pages in `wiki/summaries/MODBUS/`
4. Create or update concept pages in `wiki/concepts/`
5. Update `wiki/index.md` with new pages
6. Append entries to `wiki/log.md`

## Resources

### Official MODBUS Resources

- [MODBUS Organization](https://www.modbus.org/) - Official MODBUS website
- [MODBUS Specifications](https://www.modbus.org/modbus-specifications) - Official specifications
- [MODBUS FAQ](https://www.modbus.org/faq) - Frequently asked questions

### Related Standards

- IEC 61158: Industrial communication networks - Fieldbus specifications
- IEEE 802.3: Ethernet standard
- TIA/EIA-485: RS-485 electrical standard

### Community

- Stack Overflow: MODBUS protocol questions
- Industrial automation forums
- GitHub: MODBUS implementations and libraries

## License

This wiki summarizes public MODBUS specifications. The MODBUS protocol is maintained by the MODBUS Organization. Refer to official specifications for licensing and usage rights.

## Version History

See [wiki/log.md](wiki/log.md) for detailed change history.

**Current Version:** 1.0.0 (2026-04-18)
- Initial ingestion of 5 MODBUS specification documents
- 19 wiki pages created (5 summaries, 12 concepts, 2 navigation)
- ~30,000 words of technical content

## Support

For questions or issues:
1. Check relevant wiki pages first
2. Refer to official MODBUS specifications
3. Search community forums
4. Consult device vendor documentation

## Acknowledgments

This wiki is based on specifications maintained by the MODBUS Organization. Content is extracted and organized for software developer convenience.

---

*Last Updated: 2026-04-18*
*Purpose: MODBUS protocol reference for system control software developers*
