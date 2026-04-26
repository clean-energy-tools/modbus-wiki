---
title: Master-Slave Architecture
Summary: MODBUS master-slave communication model including roles, addressing, timing, and operational characteristics.
Sources:
  - raw/MODBUS/modbusprotocolspecification.md
  - raw/MODBUS/modbusoverserial.md
Categories:
  - architecture
  - communication-model
  - serial-communication
type: concept
date-created: 2026-04-18T12:00:00+03:00
last-updated: 2026-04-18T16:00:00+03:00
---

The master-slave architecture defines the fundamental communication model used in MODBUS serial networks, where a single master device controls communication with one or more slave devices (source: [modbusprotocolspecification.md](/raw/MODBUS/modbusprotocolspecification.md)). This architecture ensures deterministic communication and prevents bus contention.

## Architecture Overview

### Master (Client) Role

The master device initiates all communication on the MODBUS network and controls the timing and sequence of operations (source: [modbusprotocolspecification.md](/raw/MODBUS/modbusprotocolspecification.md)):

**Master Responsibilities:**
- **Initiate Requests:** Start all communication transactions
- **Control Timing:** Determine when to send requests
- **Polling:** Regularly query slave devices for data
- **Broadcast Commands:** Send commands to multiple slaves simultaneously
- **Process Responses:** Handle responses from multiple slaves
- **Error Handling:** Detect and manage communication failures
- **Coordinate Operations:** Ensure no conflicting operations

**Master Characteristics:**
- **Active Controller:** Always initiates communication
- **Address Aware:** Knows addresses of all connected slaves
- **Timing Control:** Controls request/response timing
- **Priority Management:** Can prioritize certain devices or operations
- **Network Supervisor:** Monitors network health and device status

### Slave (Server) Role

Slave devices respond to requests from the master and perform the requested operations (source: [modbusprotocolspecification.md](/raw/MODBUS/modbusprotocolspecification.md)):

**Slave Responsibilities:**
- **Listen for Requests:** Wait for master's queries
- **Process Commands:** Execute requested function codes
- **Access Data:** Read/write internal data as requested
- **Generate Responses:** Return data or exception codes
- **Maintain State:** Keep device state consistent
- **Report Status:** Provide operational status and diagnostics

**Slave Characteristics:**
- **Passive Responder:** Never initiates communication
- **Address Identification:** Has unique address (1-247)
- **Request Processing:** Processes one request at a time
- **Response Time:** Must respond within specified timeout
- **Data Consistency:** Maintain valid device state

## Communication Flow

### Request-Response Cycle

MODBUS master-slave communication follows a strict request-response pattern:

```
Master Request → Slave Processing → Slave Response → Master Processing
     ↓                ↓                  ↓               ↓
  [PDU]          [Execute]           [PDU]          [Next Request]
```

**Timing Requirements:**
- **Turnaround Time:** Time for slave to process and respond
- **Inter-frame Delay:** Minimum time between consecutive requests
- **Response Timeout:** Maximum time to wait for response
- **Bus Idle Time:** Quiet period before next request

**Synchronous Operation:**
- **One Request at a Time:** Master sends only one request
- **Wait for Response:** Master waits before sending next request
- **Deterministic:** Predictable timing for real-time control
- **No Overlap:** New request only after previous completes

### Polling Patterns

Master devices typically poll slaves in one of these patterns:

**Round-Robin Polling:**
- Cyclic polling of all devices
- Equal time allocation per device
- Suitable for similar priority devices
- Easy to implement and understand

**Priority Polling:**
- Critical devices polled more frequently
- Less critical devices polled less often
- Based on device importance or update rate
- Optimizes for system needs

**Event-Driven Polling:**
- Poll only when device signals data ready
- Reduces unnecessary network traffic
- Requires interrupt capability or event notifications
- More efficient for slowly changing data

## Addressing

### Slave Address Range

MODBUS defines specific address ranges for slave devices (source: [modbusprotocolspecification.md](/raw/MODBUS/modbusprotocolspecification.md)):

| Address Type | Range | Description |
|--------------|--------|-------------|
| Individual Slaves | 1-247 | Standard slave addresses |
| Broadcast Address | 0 | All devices, no response |
| Reserved | 248-255 | Reserved for future use |

**Address Assignment:**
- **Unique per Device:** Each slave has unique address
- **Configuration Required:** Addresses must be configured
- **Network Planning:** Address allocation affects network topology
- **Address Conflicts:** Must be avoided in network design

### Address Resolution

**Direct Addressing:**
- **Single Request:** Master addresses slave directly
- **Expected Response:** Single slave responds
- **Fast Communication:** Minimal addressing overhead

**Broadcast Addressing:**
- **All Slaves:** Master sends to address 0
- **No Response:** Slaves don't respond to broadcast
- **Use Cases:** Configuration updates, synchronization, status queries
- **Reliability:** No confirmation of receipt

## Frame Timing

### RTU Mode Timing

MODBUS RTU (Remote Terminal Unit) mode requires precise timing for frame detection (source: [modbusoverserial.md](/raw/MODBUS/modbusoverserial.md)):

**Silent Interval (T3.5):**
- **Purpose:** Allow receivers to detect end of frame
- **Minimum:** 3.5 character times at specified baud rate
- **Example:** At 9600 baud, minimum = 3.5ms
- **Implementation:** Both master and slaves must maintain timing

**Frame Processing Time:**
- **Slave Response Time:** Time for slave to process request
- **Master Turnaround Time:** Time to prepare next request
- **Total Cycle Time:** Request + Processing + Response + Turnaround

### Timing Calculation

**Baud Rate Examples:**

| Baud Rate | Character Time | Silent Interval | Example Cycle |
|------------|----------------|-----------------|---------------|
| 9600 | 1.04ms | 3.5ms | ~10ms (typical read) |
| 19200 | 0.52ms | 1.75ms | ~5ms (typical read) |
| 38400 | 0.26ms | 0.91ms | ~3ms (typical read) |

## Error Handling

### Error Detection and Prevention

**Frame-Level Errors:**
- **CRC-16 Errors:** Detected by CRC check (RTU)
- **LRC Errors:** Detected by LRC check (ASCII)
- **Timeout Errors:** No response within timeout period
- **Framing Errors:** Incorrect frame structure or timing

**Device-Level Errors:**
- **Exception Responses:** Error codes from slave devices
- **Busy Conditions:** Slave not ready (exception 0x06)
- **Address Errors:** Invalid or no device at address
- **Data Errors:** Invalid data values or quantities

### Error Recovery

**Master Error Recovery:**
- **Retry Logic:** Implement retry with exponential backoff
- **Device Bypass:** Skip failed device and continue
- **Network Reset:** Reset communication if multiple errors
- **Logging:** Record errors for analysis and troubleshooting

**Slave Error Handling:**
- **Exception Codes:** Return appropriate exception codes
- **Error States:** Set device error flags if applicable
- **Diagnostic Support:** Provide diagnostic information
- **Recovery Actions:** Implement automatic recovery when possible

## Network Topology

### Bus Topology

**Multi-Drop Configuration:**
- **Single Trunk:** All devices on same RS-485 bus
- **Termination:** Proper termination at both ends of bus
- **Stub Length:** Short stubs for branch connections
- **Device Spacing:** Proper spacing between device taps

**Star Topology:**
- **Central Hub:** Multiple point-to-point connections
- **Isolation:** Each device on separate pair
- **No Collision:** No bus contention
- **Higher Cost:** More cabling and hardware

### Daisy-Chain Topology:**
- **Sequential Connection:** Devices connected in series
- **Signal Degradation:** Signal attenuates across chain
- **Limited Devices:** Typically 32 devices maximum
- **Reliability Issues:** Single point failure affects all downstream

## Performance Optimization

### Polling Optimization

**Efficient Polling Strategies:**
- **Differential Polling:** Only read changed values
- **Batch Operations:** Read multiple registers per request
- **Optimal Frequency:** Balance between responsiveness and load
- **Selective Polling:** Skip devices that don't need updates

**Response Time Optimization:**
- **Fast Slave Response:** Minimize processing time
- **Efficient Master Processing:** Quick response handling
- **Parallel Processing:** Handle multiple slave responses
- **Timeout Tuning:** Optimize timeout for network conditions

### Throughput Calculation

**Maximum Theoretical Throughput:**
```
Throughput = (Request Size + Response Size) / Poll Cycle Time
```

**Example Calculation:**
- Request: 8 bytes (read 4 registers)
- Response: 11 bytes (function + byte count + 4 registers)
- Total: 19 bytes
- Poll Cycle: 10ms (typical)
- Throughput: 19 bytes / 10ms = 1.9 KB/s per device

**Real-World Considerations:**
- **Network Overhead:** Includes CRC, framing, and turnaround time
- **Contention:** Reduced throughput with multiple devices
- **Error Recovery:** Retries reduce effective throughput
- **Protocol Overhead:** Address, function code, byte count

## Implementation Considerations

### Master Implementation

**Configuration:**
- **Slave List:** Maintain list of connected slave devices
- **Address Mapping:** Map logical addresses to physical addresses
- **Polling Schedule:** Configure polling intervals per device
- **Timeout Values:** Set appropriate timeouts for different operations

**Operation:**
- **Polling Logic:** Implement round-robin or priority-based polling
- **Broadcast Support:** Handle broadcast operations correctly
- **Response Management:** Track and match responses to requests
- **Error Handling:** Comprehensive error detection and recovery

### Slave Implementation

**Configuration:**
- **Address Setting:** Configure unique slave address
- **Function Support:** Implement supported function codes
- **Data Model:** Define accessible data objects
- **Response Timing:** Optimize response processing

**Operation:**
- **Request Processing:** Efficiently handle incoming requests
- **Data Access:** Provide access to internal data structures
- **Exception Handling:** Return appropriate exception codes
- **State Management:** Maintain consistent device state

## Troubleshooting

### Common Issues

**Communication Failures:**
- **No Response:** Check addressing, connections, and timeouts
- **CRC Errors:** Verify baud rate and electrical connections
- **Timeout Errors:** Adjust timeout values and check slave processing time
- **Wrong Response:** Verify function codes and data formats

**Addressing Problems:**
- **Duplicate Addresses:** Ensure unique addresses on network
- **Address Conflicts:** Review network configuration
- **Broadcast Issues:** Verify broadcast usage and expectations
- **Address Range:** Verify addresses within 1-247 range

**Timing Issues:**
- **Frame Collisions:** Reduce polling frequency or improve timing
- **Timeout Too Short:** Increase timeout for slow devices
- **Silent Interval Violation:** Check baud rate and timing
- **Response Delay:** Investigate slave processing time

## Related Information

- [MODBUS RTU](/wiki/concepts/modbus-rtu.md) - Serial RTU mode specifications
- [MODBUS ASCII](/wiki/concepts/modbus-ascii.md) - Serial ASCII mode specifications
- [Protocol](/wiki/concepts/protocol.md) - General protocol concepts
- [Implementation](/wiki/concepts/implementation.md) - Implementation best practices

See also: [protocol-architecture](/wiki/concepts/protocol-architecture.md) for advanced architectural patterns.

## Related pages

## Backlinks

No backlinks (yet)

------------

MODBUS is a trademark of the Modbus Organization, Inc.

All information on this page is derived by summarizing and analyzing solely the MODBUS specifications published by the Modbus Organization, Inc.  The specifications are copyright by the Modbus Organization.

By deriving this information solely from the specifications we hope to stay true to the specification.

No infringement is intended.

- [MODBUS Broadcast](/wiki/answers/modbus-broadcast.md) - Comprehensive guide to broadcast functionality