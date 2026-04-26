---
title: How MODBUS TCP-to-RTU Gateways Work
Summary: Comprehensive explanation of MODBUS TCP-to-RTU gateway operation including protocol conversion, Unit ID mapping, frame translation, timing management, error handling, and practical implementation considerations.
Sources:
  - /raw/MODBUS/MODBUS.md
  - /raw/MODBUS/messagingimplementationguide.md
  - /raw/MODBUS/modbusoverserial.md
Categories:
  - gateways
  - protocol-conversion
  - tcp-to-serial
  - integration
type: answer
date-created: 2026-04-23T14:00:00+03:00
last-updated: 2026-04-23T14:00:00+03:00
---

You're absolutely right that there's much more to MODBUS TCP-to-RTU gateways than simple connection translation. This document explains how these gateways work in detail, covering protocol conversion, addressing, timing, error handling, and practical implementation considerations.

## Quick Overview

A MODBUS TCP-to-RTU gateway is a protocol converter that bridges the gap between:
- **TCP side:** Ethernet network with MODBUS TCP clients
- **RTU side:** RS-485/RS-232 serial bus with MODBUS RTU slave devices

**Key function:** Translate between two fundamentally different protocols while maintaining semantic equivalence.

## Gateway Architecture

### Physical Topology

```
┌─────────────────────────────────────────────────────────────────┐
│                     Ethernet Network (TCP/IP)                    │
│                                                                   │
│  ┌──────────┐      ┌──────────┐       ┌──────────┐             │
│  │ Client 1 │      │ Client 2 │       │ Client N │             │
│  │192.168.1.│      │192.168.1.│       │192.168.1.│             │
│  │   .50    │      │   .51    │       │   .5N    │             │
│  └────┬─────┘      └────┬─────┘       └────┬─────┘             │
│       │                 │                   │                    │
│       └─────────────────┴───────────────────┘                    │
│                         │                                        │
│                         │ Port 502                               │
│              ┌──────────▼──────────┐                             │
│              │   MODBUS Gateway    │                             │
│              │  IP: 192.168.1.100  │                             │
│              │                     │                             │
│              │  TCP Side │ RTU Side│                             │
│              └───────────┬─────────┘                             │
└──────────────────────────┼───────────────────────────────────────┘
                           │
                           │ RS-485 Bus
              ┌────────────┴────────────┬──────────────┬──────────┐
              │                         │              │          │
         ┌────▼────┐              ┌────▼────┐    ┌────▼────┐   ...
         │ Slave 1 │              │ Slave 2 │    │ Slave N │
         │ Addr: 1 │              │ Addr: 2 │    │ Addr: N │
         └─────────┘              └─────────┘    └─────────┘
```

### Gateway Components

**1. TCP/IP Network Interface:**
- Ethernet port (RJ-45)
- TCP/IP stack
- Listens on port 502 for MODBUS TCP connections
- Manages multiple concurrent TCP client connections

**2. Serial Interface:**
- RS-485 or RS-232 port (typically RS-485)
- UART (Universal Asynchronous Receiver/Transmitter)
- Handles serial timing and character transmission
- Acts as MODBUS RTU master on serial side

**3. Protocol Converter:**
- Core gateway logic
- Translates MODBUS TCP ↔ MODBUS RTU
- Maps Unit ID to slave addresses
- Manages request queuing and response routing

**4. Configuration:**
- Serial port settings (baud rate, parity, stop bits)
- Timeout values
- Optional: slave address mapping tables
- Optional: transaction filtering/security

Source: [mbap-header.md](/wiki/concepts/mbap-header.md:129)

## How Unit ID Mapping Works

### The Unit ID Field

The MBAP header includes a **Unit ID** field (byte 6) specifically designed for gateway routing:

| Unit ID Value | Meaning |
|---------------|---------|
| **0xFF (255)** | Direct TCP connection (no gateway) |
| **0x01-0xF7 (1-247)** | Route to serial slave with this address |
| **0x00** | Reserved (not used) |
| **0xF8-0xFE** | Reserved |

**Key principle:** When a MODBUS TCP client wants to communicate with a serial slave through a gateway, it sets the Unit ID to the slave's serial address.

### Example: Routing to Slave #5

**Client request:**
```
MODBUS TCP Frame:
┌─────────────────── MBAP Header ───────────────────┬──── PDU ────┐
│ Trans ID │ Proto ID │ Length │ Unit ID │ Function │ Start │ Qty │
│  0x0001  │  0x0000  │ 0x0006 │  0x05   │   0x03   │ 0x00  │ 0x02│
└──────────┴──────────┴────────┴─────────┴──────────┴───────┴─────┘
                                   └─> Target: Slave 5
```

**Gateway action:**
1. Receives TCP frame on port 502
2. Extracts Unit ID = 5
3. Knows this means "route to serial slave address 5"
4. Converts to MODBUS RTU frame
5. Sends to slave 5 on RS-485 bus

Source: [modbus-tcp.md](/wiki/concepts/modbus-tcp.md:192)

## Protocol Conversion: TCP to RTU

### Step-by-Step Conversion Process

#### Request Flow: TCP → RTU

**1. Receive MODBUS TCP request from client**

TCP frame (12 bytes):
```
00 01 00 00 00 06 05 03 00 00 00 02
│  MBAP Header │UD│    PDU      │
└──────────────┴──┴─────────────┘
```

**2. Parse MBAP header**

```c
Transaction ID: 0x0001 (save for response routing)
Protocol ID:    0x0000 (verify = MODBUS)
Length:         0x0006 (6 bytes follow)
Unit ID:        0x05   (target slave address)
```

**3. Extract PDU**

```
Function Code: 0x03 (Read Holding Registers)
Start Address: 0x0000 (register 0)
Quantity:      0x0002 (2 registers)
```

**4. Build MODBUS RTU frame**

RTU frame structure:
```
[Address][Function][Data][CRC-16]
```

Convert:
```
Address:  0x05 (from Unit ID)
Function: 0x03 (from PDU)
Data:     00 00 00 02 (from PDU)
CRC-16:   Calculate over [Address][Function][Data]
```

**5. Calculate CRC-16**

```
Input data: 05 03 00 00 00 02
CRC-16:     0xC40B (example)
Transmitted as: 0B C4 (LSB first!)
```

**6. Build complete RTU frame (8 bytes)**

```
05 03 00 00 00 02 0B C4
│  │  └─ PDU ──┘ └CRC┘
│  └─ Function
└─ Slave Address
```

**7. Transmit on serial bus**

- Wait for serial bus idle (3.5 character times)
- Transmit frame as continuous stream
- Start response timeout timer

#### Response Flow: RTU → TCP

**1. Receive MODBUS RTU response from slave**

RTU frame (9 bytes):
```
05 03 04 00 0A 00 14 XX XX
│  │  │  └─ Data ─┘ └CRC┘
│  │  └─ Byte Count
│  └─ Function
└─ Slave Address
```

**2. Validate RTU frame**

- Check CRC-16 (recalculate and compare)
- Verify slave address matches request (0x05)
- Verify function code matches request (0x03)

**3. Extract PDU from RTU frame**

```
Function Code: 0x03
Byte Count:    0x04
Data:          00 0A 00 14
```

**4. Look up pending transaction**

- Find TCP client waiting for response from slave 5
- Retrieve saved Transaction ID (0x0001)

**5. Build MBAP header**

```
Transaction ID: 0x0001 (from saved state)
Protocol ID:    0x0000 (always for MODBUS)
Length:         0x0007 (1 byte Unit ID + 6 bytes PDU)
Unit ID:        0x05   (echo from request)
```

**6. Assemble MODBUS TCP response (13 bytes)**

```
00 01 00 00 00 07 05 03 04 00 0A 00 14
│  MBAP Header │UD│      PDU           │
└──────────────┴──┴────────────────────┘
```

**7. Send TCP response to client**

- Locate TCP connection for original client
- Send response over TCP socket
- No CRC needed (TCP provides error checking)

Source: [MODBUS TCP Message Format](/wiki/answers/modbus-tcp-message-format.md)

## Critical Differences Between TCP and RTU

### Frame Format Differences

| Aspect | MODBUS TCP | MODBUS RTU |
|--------|------------|------------|
| **Header** | MBAP (7 bytes) | Slave Address (1 byte) |
| **Error Check** | None (TCP checksum) | CRC-16 (2 bytes) |
| **Framing** | Length field | Silent intervals (3.5 char) |
| **Byte Order** | Big-endian (all) | Big-endian (except CRC LSB first) |
| **Max Frame** | 260 bytes | 256 bytes |
| **Transaction ID** | Yes (2 bytes) | No |

### Gateway Translation Table

| Field | MODBUS TCP | → | MODBUS RTU |
|-------|------------|---|------------|
| Transaction ID | 0x0001 | → | *(saved for response)* |
| Protocol ID | 0x0000 | → | *(verified, discarded)* |
| Length | 0x0006 | → | *(used to extract PDU)* |
| Unit ID | 0x05 | → | **Slave Address: 0x05** |
| PDU | [Function][Data] | → | **[Function][Data]** |
| CRC | *(none)* | → | **Calculate CRC-16** |

**Key conversions:**
1. **Unit ID → Slave Address:** Direct 1:1 mapping
2. **PDU preserved:** Function code and data unchanged
3. **MBAP removed:** Not sent to serial slaves
4. **CRC added:** Calculated over [Address][PDU]

Source: [modbus-rtu.md](/wiki/concepts/modbus-rtu.md:48)

## Timing Management

### Critical Timing Challenges

**Problem:** TCP and RTU have incompatible timing models:
- **TCP:** Asynchronous, no timing constraints
- **RTU:** Strict timing requirements (silent intervals)

### RTU Timing Requirements

**Inter-character timing (t1.5):**
- Maximum gap between characters in a frame: 1.5 character times
- If exceeded → frame error

**Inter-frame timing (t3.5):**
- Minimum gap between frames: 3.5 character times
- Marks frame boundaries

**Character time calculation:**
```
Character time = 11 bits / baud_rate  (with parity)

Example at 9600 baud:
t1.5 = 1.5 × (11 / 9600) = 1.72 ms
t3.5 = 3.5 × (11 / 9600) = 4.01 ms
```

Source: [modbus-rtu.md](/wiki/concepts/modbus-rtu.md:80)

### Gateway Timing Management

**Serial bus arbitration:**

**Gateway acts as master on RTU side:**
1. Wait for bus idle (t3.5 silence)
2. Transmit complete frame as continuous stream
3. Maintain t1.5 timing between characters
4. Wait for slave response (timeout: typically 100ms - 1s)
5. Wait t3.5 before next transmission

**Turnaround time:**
- Time between receiving request and sending response
- Slave needs time to process request
- Gateway must wait for this delay

**Queue management:**

If multiple TCP clients send requests simultaneously:
```
TCP Client 1 → Request for Slave 5 ──┐
TCP Client 2 → Request for Slave 2 ──┼→ Gateway Queue
TCP Client 3 → Request for Slave 5 ──┘
```

Gateway must serialize:
1. Send request to Slave 5 (from Client 1)
2. Wait for response
3. Wait t3.5
4. Send request to Slave 2 (from Client 2)
5. Wait for response
6. Wait t3.5
7. Send request to Slave 5 (from Client 3)
8. And so on...

**Critical:** Only one serial transaction at a time. TCP clients may need to wait.

## Transaction Management

### Request Queuing

**Gateway maintains request queue:**

```
┌──────────────────────────────────────────┐
│           Gateway Request Queue          │
├────┬──────────┬─────────┬─────────┬──────┤
│ ID │ Trans ID │ Client  │ Slave   │ PDU  │
├────┼──────────┼─────────┼─────────┼──────┤
│ 1  │ 0x0001   │ 192...50│    5    │ ...  │
│ 2  │ 0x0045   │ 192...51│    2    │ ...  │
│ 3  │ 0x0002   │ 192...50│    5    │ ...  │
└────┴──────────┴─────────┴─────────┴──────┘
```

**For each queued request, gateway tracks:**
- Transaction ID (from MODBUS TCP request)
- Source TCP client (IP address, socket)
- Target slave address (from Unit ID)
- PDU (function code + data)
- Timestamp (for timeout detection)

### Response Routing

**Challenge:** Multiple TCP clients, same slave address

**Example scenario:**
- Client A requests data from Slave 5 (Transaction ID = 0x0001)
- Client B requests data from Slave 5 (Transaction ID = 0x0045)

**Gateway must:**
1. Send first request to Slave 5
2. Receive response from Slave 5
3. Route response to **correct client** (Client A)
4. Send second request to Slave 5
5. Receive response from Slave 5
6. Route response to **correct client** (Client B)

**Routing mechanism:**
1. Gateway sends request, saves (Transaction ID, Client Socket) mapping
2. Slave responds (knows nothing about Transaction ID)
3. Gateway looks up pending transaction for that slave
4. Retrieves Transaction ID and Client Socket
5. Builds TCP response with saved Transaction ID
6. Sends to correct client socket

### Concurrent TCP Connections

**Gateway supports multiple TCP clients simultaneously:**

```
Client A ──┐
Client B ──┼→ Gateway → Serial Bus (one request at a time)
Client C ──┘
```

**TCP side:** Concurrent (multiple clients connected)
**Serial side:** Sequential (one transaction at a time)

**Gateway must:**
- Accept multiple TCP connections
- Queue requests from all clients
- Serialize to serial bus
- Route responses back to correct client

## Error Handling

### Error Types and Gateway Response

#### 1. TCP-Side Errors

**TCP connection failure:**
- Client disconnects while request pending
- Gateway action: Discard pending request, log error

**Malformed MODBUS TCP frame:**
- Protocol ID ≠ 0x0000
- Length field inconsistent
- Gateway action: Discard frame, optionally close connection

#### 2. Serial-Side Errors

**No response from slave (timeout):**
- Gateway waits (typically 1 second)
- No response received
- Gateway action: Generate MODBUS exception response, send to TCP client
  ```
  Exception Code: 0x0B (Gateway Target Device Failed to Respond)
  ```

**CRC error:**
- Received RTU frame has invalid CRC
- Gateway action: Discard frame, wait for timeout, generate exception to TCP client

**Invalid slave response:**
- Slave returns exception (function code ≥ 0x80)
- Gateway action: Convert RTU exception to TCP exception, forward to client

#### 3. Serial Bus Errors

**Bus collision (multiple masters):**
- Should not happen if gateway is only master
- If detected: Retransmit after delay

**Character timing error (t1.5 violated):**
- Indicates noisy serial line or baud rate mismatch
- Gateway action: Discard frame, wait for timeout

### Exception Response Translation

**RTU exception from slave:**
```
RTU Response: 05 83 02 XX XX
              │  │  │  └CRC┘
              │  │  └─ Exception Code: 0x02 (ILLEGAL DATA ADDRESS)
              │  └─ Exception Function: 0x83 (0x03 + 0x80)
              └─ Slave Address: 5
```

**Gateway converts to TCP exception:**
```
TCP Response: 00 01 00 00 00 03 05 83 02
              │  MBAP Header │UD│ PDU│
              └──────────────┴──┴────┘
              
Transaction ID: 0x0001 (from saved state)
Protocol ID:    0x0000
Length:         0x0003 (3 bytes: Unit ID + exception PDU)
Unit ID:        0x05
Function:       0x83 (exception)
Exception Code: 0x02
```

**Client receives exception, knows address was invalid.**

## Configuration Considerations

### Serial Port Settings

**Must match all slaves on bus:**
- **Baud rate:** 9600, 19200, 38400, 115200 (common values)
- **Data bits:** 8 (standard)
- **Parity:** Even (default), Odd, None
- **Stop bits:** 1 (with parity), 2 (without parity)

**Example:** 9600 baud, 8 data bits, even parity, 1 stop bit (9600-8E1)

**All devices on RS-485 bus must use identical settings.**

### Timeout Values

**Serial response timeout:**
- How long to wait for slave response
- Typical: 100ms - 1000ms (1 second common)
- Too short: False timeouts on slow slaves
- Too long: Poor TCP client experience

**TCP response timeout:**
- How long TCP client should wait for gateway response
- Should be > (Serial timeout + queue delay)
- Typical: 5 seconds

**Inter-frame delay:**
- Enforced t3.5 silent interval
- Calculated from baud rate
- Fixed at 1.75ms for baud > 19200

### Slave Address Mapping

**Simple mode (default):**
- Unit ID directly maps to slave address
- Unit ID 1 → Slave 1, Unit ID 2 → Slave 2, etc.

**Advanced mapping (optional):**
- Gateway can remap addresses
- Example: Unit ID 10 → Slave 1, Unit ID 11 → Slave 2
- Useful for addressing conflicts or security

**Configuration example:**
```
Unit ID → Slave Address
   1    →     1
   2    →     2
   5    →     5
  10    →     1  (remapped)
  11    →     2  (remapped)
```

### Security and Filtering

**Optional gateway features:**

**IP filtering:**
- Allow only specific TCP clients
- Example: Only 192.168.1.50-59 can connect

**Function code filtering:**
- Block dangerous functions (e.g., Write Multiple Registers)
- Read-only mode: Only allow function codes 0x01-0x04

**Slave protection:**
- Rate limiting (max requests per second)
- Prevent serial bus flooding

## Performance Considerations

### Throughput Limitations

**Serial bus is the bottleneck:**

**Example calculation at 9600 baud:**
- Character time: 11 / 9600 = 1.146 ms
- Minimum frame (6 bytes): 6 × 1.146 = 6.88 ms
- Inter-frame delay: 4.01 ms
- Slave processing: ~10 ms (varies)
- **Total:** ~21 ms per transaction
- **Maximum throughput:** ~47 transactions/second

**Higher baud rates improve performance:**
- 115200 baud: ~330 transactions/second theoretical

**TCP side is much faster:**
- Ethernet: 100 Mbps or 1 Gbps
- MODBUS TCP over LAN: <1 ms latency
- **Bottleneck is always the serial side**

### Queue Depth

**Gateway queue size affects responsiveness:**

**Scenario:** 10 TCP clients, each sends request simultaneously

**With queue depth = 10:**
- All requests queued
- First client: Response in ~21 ms
- Last client: Response in ~210 ms (10 × 21 ms)

**With queue depth = 3:**
- First 3 requests queued
- Remaining 7 rejected or delayed
- Better for latency-sensitive applications

**Trade-off:** Queue depth vs. fairness

### Optimization Strategies

**Read coalescing:**
- Multiple TCP clients request same register from same slave
- Gateway combines into single RTU request
- Distributes response to all waiting clients
- Significantly improves throughput

**Polling optimization:**
- If TCP clients poll regularly, cache recent slave responses
- Respond from cache if fresh enough
- Reduces serial bus traffic

**Priority queuing:**
- Critical clients/slaves get priority
- Less important requests queued with lower priority

## Practical Example: Complete Transaction

### Scenario

**TCP Client** (192.168.1.50) wants to read 2 holding registers starting at address 100 from **Slave 5** connected to gateway's RS-485 bus.

### Complete Flow

**Step 1: Client sends MODBUS TCP request**

```
TCP Socket: 192.168.1.50:52341 → 192.168.1.100:502

MODBUS TCP Frame (12 bytes):
00 01 00 00 00 06 05 03 00 64 00 02
│Trans│Proto│Length│UD│FC│Addr │Qty│
```

Decoded:
- Transaction ID: 0x0001
- Protocol ID: 0x0000
- Length: 6
- Unit ID: 5 (target slave)
- Function: 0x03 (Read Holding Registers)
- Start Address: 0x0064 (100 decimal)
- Quantity: 2 registers

**Step 2: Gateway receives and parses**

- Extracts Unit ID = 5
- Saves transaction state: (Trans ID=1, Client=192.168.1.50, Slave=5)
- Extracts PDU: 03 00 64 00 02

**Step 3: Gateway converts to RTU**

```
RTU Frame Construction:
Address:    05 (from Unit ID)
Function:   03 (from PDU)
Data:       00 64 00 02 (from PDU)
CRC:        Calculate over [05 03 00 64 00 02]
            Result: 0xC5B1
            Transmitted: B1 C5 (LSB first)

Complete RTU Frame (8 bytes):
05 03 00 64 00 02 B1 C5
```

**Step 4: Gateway transmits on RS-485**

- Waits for bus idle (t3.5 = 4 ms at 9600 baud)
- Transmits frame as continuous stream
- Starts response timer (1 second timeout)

**Step 5: Slave 5 processes request**

- Receives frame, validates CRC
- Sees slave address = 5 (matches)
- Reads registers 100-101
- Values: Register 100 = 1234, Register 101 = 5678

**Step 6: Slave 5 responds**

```
RTU Response Frame (9 bytes):
05 03 04 04 D2 16 2E XX XX
│  │  │  │Reg100│Reg101│CRC│

Breakdown:
Address:     05
Function:    03
Byte Count:  04 (4 bytes of data)
Register 100: 04 D2 (0x04D2 = 1234 decimal)
Register 101: 16 2E (0x162E = 5678 decimal)
CRC-16:      (calculated by slave)
```

**Step 7: Gateway receives RTU response**

- Validates CRC
- Verifies slave address = 5
- Looks up pending transaction for Slave 5
- Finds: Trans ID=1, Client=192.168.1.50

**Step 8: Gateway converts to TCP response**

```
TCP Response Frame (13 bytes):
00 01 00 00 00 07 05 03 04 04 D2 16 2E
│Trans│Proto│Length│UD│FC│BC│ Data  │

Breakdown:
Transaction ID: 0x0001 (from saved state)
Protocol ID:    0x0000
Length:         7 (1 + 6)
Unit ID:        0x05
PDU:            03 04 04 D2 16 2E
```

**Step 9: Gateway sends to TCP client**

```
TCP Socket: 192.168.1.100:502 → 192.168.1.50:52341

Sends 13-byte response
```

**Step 10: Client receives and decodes**

- Matches Transaction ID = 1 to original request
- Extracts register values:
  - Register 100 = 0x04D2 = 1234
  - Register 101 = 0x162E = 5678
- Application uses the data

**Total time:** ~25ms (TCP: <1ms + RTU: ~20ms + Processing: ~5ms)

## Common Gateway Issues and Troubleshooting

### Issue 1: Timeout Errors

**Symptom:** TCP clients receive "Gateway Target Device Failed to Respond" exceptions

**Possible causes:**
- Serial timeout too short (slave needs more time)
- Wrong baud rate (slave not receiving correctly)
- Slave is offline or malfunctioning
- RS-485 wiring issues (termination, polarity)

**Troubleshooting:**
1. Verify serial port settings match slaves
2. Increase timeout value
3. Test slave directly with serial tool
4. Check RS-485 bus wiring and termination

### Issue 2: CRC Errors

**Symptom:** Gateway logs CRC errors, slaves not responding

**Possible causes:**
- Baud rate mismatch
- Electrical noise on RS-485 bus
- Poor grounding
- Cable too long (>1200m for RS-485)

**Troubleshooting:**
1. Verify baud rate configuration
2. Check RS-485 cable quality and length
3. Add/check termination resistors (120Ω)
4. Improve grounding
5. Use shielded cable

### Issue 3: Slow Performance

**Symptom:** Responses take seconds instead of milliseconds

**Possible causes:**
- Low baud rate (9600 vs 115200)
- Queue depth too large
- Many TCP clients competing
- Slaves have slow processing

**Troubleshooting:**
1. Increase baud rate (if all slaves support)
2. Reduce queue depth
3. Implement priority queuing
4. Use read coalescing
5. Cache frequent reads

### Issue 4: Wrong Slave Responding

**Symptom:** Slave 1 responds to requests for Slave 2

**Possible causes:**
- Duplicate slave addresses on bus
- Slave misconfigured

**Troubleshooting:**
1. Verify each slave has unique address
2. Reconfigure slaves with unique addresses
3. Use gateway address mapping as workaround

### Issue 5: Gateway Not Accessible

**Symptom:** TCP clients can't connect to gateway

**Possible causes:**
- IP address conflict
- Firewall blocking port 502
- Gateway network settings incorrect

**Troubleshooting:**
1. Ping gateway IP address
2. Verify port 502 open (telnet 192.168.1.100 502)
3. Check firewall rules
4. Verify gateway IP configuration

## Advanced Features

### Multi-Master Support (Rare)

**Problem:** Multiple gateways on same RS-485 bus

**Solution:**
- Bus arbitration protocol
- Each gateway monitors bus before transmitting
- Detects collisions and retransmits
- **Not recommended:** Single gateway per bus is standard

### Protocol Translation

**Some gateways support additional protocols:**
- MODBUS TCP ↔ MODBUS RTU (standard)
- MODBUS TCP ↔ MODBUS ASCII
- MODBUS TCP ↔ Other protocols (proprietary)

### Data Mapping and Scaling

**Advanced gateways may provide:**
- Register address translation (virtual addressing)
- Data scaling (convert raw values to engineering units)
- Data type conversion (32-bit float from 2 registers)

**Example:**
```
TCP Request: Read register 1000
Gateway maps: 1000 → Slave 5, Register 100
Gateway reads: Slave 5, Register 100-101 (2 registers)
Gateway converts: Combine as 32-bit float
Gateway returns: Float value to TCP client
```

### Logging and Diagnostics

**Enterprise gateways provide:**
- Transaction logging (all requests/responses)
- Error logging (timeouts, CRC errors, exceptions)
- Performance metrics (transactions/second, latency)
- SNMP/syslog integration for monitoring

## Summary: Gateway Operation

| Aspect | Details |
|--------|---------|
| **Primary Function** | Translate MODBUS TCP ↔ MODBUS RTU |
| **TCP Side** | Server on port 502, supports multiple clients |
| **Serial Side** | Master on RS-485/RS-232 bus |
| **Addressing** | Unit ID (TCP) → Slave Address (RTU) |
| **Frame Conversion** | Add/remove MBAP header, add/remove CRC-16 |
| **Timing** | Enforce RTU t3.5 intervals, serialize requests |
| **Error Handling** | Convert RTU exceptions to TCP, timeout management |
| **Performance** | Limited by serial baud rate (bottleneck) |
| **Configuration** | Baud rate, parity, timeouts, address mapping |

## Related Pages

- [MODBUS TCP](/wiki/concepts/modbus-tcp.md) - MODBUS TCP protocol
- [MODBUS RTU](/wiki/concepts/modbus-rtu.md) - MODBUS RTU protocol
- [MBAP Header](/wiki/concepts/mbap-header.md) - Unit ID field for gateway routing
- [MODBUS TCP Message Format](/wiki/answers/modbus-tcp-message-format.md) - TCP frame structure
- [Using Wireshark for MODBUS TCP Analysis](/wiki/answers/wireshark-modbus-analysis.md) - Debugging gateway traffic

## Backlinks

- [MODBUS Broadcast](/wiki/answers/modbus-broadcast.md)
- [MODBUS Commissioning Checklist](/wiki/answers/modbus-commissioning-checklist.md)
- [MODBUS Errors and Exception Responses](/wiki/answers/modbus-errors-and-exceptions.md)
- [MBAP Header](/wiki/concepts/mbap-header.md)
- [MODBUS RTU](/wiki/concepts/modbus-rtu.md)
- [MODBUS TCP](/wiki/concepts/modbus-tcp.md)

------------

MODBUS is a trademark of the Modbus Organization, Inc.

All information on this page is derived by summarizing and analyzing solely the MODBUS specifications published by the Modbus Organization, Inc.  The specifications are copyright by the Modbus Organization.

By deriving this information solely from the specifications we hope to stay true to the specification.

No infringement is intended.
