---
title: Using Wireshark for MODBUS TCP Analysis
Summary: Comprehensive guide to using Wireshark for MODBUS TCP communication analysis including byte-level inspection, packet recognition, protocol decoding, filtering, and analysis of both standard and secure MODBUS traffic.
Sources:
  - /raw/MODBUS/MODBUS.md
  - /raw/MODBUS/messagingimplementationguide.md
  - /raw/MODBUS/modbussecurityprotocol.md
Categories:
  - network-analysis
  - troubleshooting
  - wireshark
  - security
type: answer
date-created: 2026-04-23T13:30:00+03:00
last-updated: 2026-04-23T13:30:00+03:00
---

Wireshark is a powerful network protocol analyzer that can capture and decode MODBUS TCP traffic. This guide covers how to use Wireshark for byte-level MODBUS TCP analysis, recognize packet boundaries, leverage Wireshark's built-in MODBUS decoder, and analyze both standard and secure MODBUS traffic.

## Quick Answer Summary

**Can Wireshark analyze MODBUS TCP?** Yes, completely.

**Does Wireshark decode MODBUS?** Yes, Wireshark has a built-in MODBUS dissector that automatically decodes MODBUS TCP frames.

**Can Wireshark analyze MODBUS Security traffic?** Partially - it can see TLS handshake and encrypted records, but cannot decode the encrypted MODBUS data without TLS keys.

**How to recognize MODBUS packets?** Look for TCP port 502 (standard) or 802 (secure), and the 7-byte MBAP header pattern.

## Overview: Wireshark and MODBUS TCP

### What Wireshark Can Do

**Standard MODBUS TCP (Port 502):**
- ✅ Capture all MODBUS TCP traffic
- ✅ Automatically decode MBAP header fields
- ✅ Automatically decode MODBUS PDU (function codes, data)
- ✅ Display register values, coil states, addresses
- ✅ Show request/response pairing (Transaction ID matching)
- ✅ Identify errors and exceptions
- ✅ Calculate timing and performance metrics

**MODBUS/TCP Security (Port 802):**
- ✅ Capture TLS-encrypted traffic
- ✅ Decode TLS handshake (certificate exchange, cipher negotiation)
- ✅ Show TLS record layer structure
- ❌ Cannot decode encrypted MODBUS data (without TLS session keys)
- ⚠️ Can decrypt if you provide TLS session keys (advanced)

Source: [modbus-tcp-security.md](/wiki/concepts/modbus-tcp-security.md:233)

## Setting Up Wireshark for MODBUS Capture

### Step 1: Select Capture Interface

**Identify the network interface:**
- Ethernet adapter connected to MODBUS network
- Wireless adapter if MODBUS devices are on WiFi (less common)
- Loopback (lo/127.0.0.1) if capturing local MODBUS communication

**In Wireshark:**
1. Open Wireshark
2. View available interfaces (Capture → Options or double-click interface)
3. Select the interface connected to MODBUS network
4. Click "Start" to begin capture

### Step 2: Apply Capture Filter (Optional)

**Purpose:** Reduce captured traffic to only MODBUS-related packets

**Capture filter syntax (BPF - Berkeley Packet Filter):**
```
tcp port 502
```

**For both standard and secure MODBUS:**
```
tcp port 502 or tcp port 802
```

**For specific device:**
```
tcp port 502 and host 192.168.1.100
```

**Note:** Capture filters are applied during capture and cannot be changed after. Use display filters for post-capture analysis.

### Step 3: Start Capture

Click "Start" or press `Ctrl+E` to begin capturing packets.

## Recognizing MODBUS TCP Packets

### Method 1: By Port Number

MODBUS TCP uses well-known ports:
- **Port 502:** Standard MODBUS TCP
- **Port 802:** MODBUS/TCP Security (TLS)

**In Wireshark packet list:**
- Look at "Info" column: `502 → xxxxx` or `xxxxx → 502`
- Source/Destination port columns will show 502 or 802

**Display filter:**
```
tcp.port == 502
```

### Method 2: By Protocol Column

If Wireshark recognizes MODBUS:
- **Protocol column** shows "MODBUS/TCP" or "Modbus"
- Automatically applied when Wireshark detects port 502 traffic

### Method 3: By MBAP Header Pattern

**MODBUS TCP packets have distinctive MBAP header:**

```
Bytes 0-1:   Transaction ID (varies)
Bytes 2-3:   Protocol ID = 0x00 0x00 (always)
Bytes 4-5:   Length (0x0002 to 0x00FE)
Byte 6:      Unit ID (often 0x01 or 0xFF)
```

**Key indicator:** Bytes 2-3 are **always 0x00 0x00** in MODBUS TCP.

**Byte-level inspection:**
1. Select packet in Wireshark
2. Look at "Packet Bytes" pane (bottom)
3. Look for `00 00` at bytes 2-3 of TCP payload

## Recognizing Packet Boundaries

### Understanding TCP Stream vs MODBUS Messages

**TCP is stream-oriented:**
- One TCP packet may contain multiple MODBUS messages
- One MODBUS message may span multiple TCP packets
- No inherent message boundaries in TCP

**MODBUS TCP uses Length field for boundaries:**
- Byte 4-5 of MBAP header = Length
- Length = number of bytes following Length field
- Length includes Unit ID (1 byte) + PDU (variable)

### Finding MODBUS Message Boundaries

#### Scenario 1: One MODBUS Message Per TCP Packet (Common)

**Example packet:**
```
TCP Payload (12 bytes total):
00 01 00 00 00 06 01 03 00 00 00 02
│     │     │     │  └─ PDU (5 bytes)
│     │     │     └─ Unit ID (1 byte)
│     │     └─ Length = 6 (Unit ID + PDU)
│     └─ Protocol ID = 0
└─ Transaction ID = 1
```

**Boundary recognition:**
- Length field = 0x0006 (6 bytes)
- 6 bytes follow the Length field → message ends at byte 12
- TCP payload = MBAP (7 bytes) + 6 bytes = 12 bytes total

#### Scenario 2: Multiple MODBUS Messages in One TCP Packet

**Example packet:**
```
TCP Payload (24 bytes total):

Message 1 (bytes 0-11):
00 01 00 00 00 06 01 03 00 00 00 02
          └─ Length = 6

Message 2 (bytes 12-23):
00 02 00 00 00 06 01 03 00 0A 00 02
          └─ Length = 6
```

**Boundary recognition:**
1. First message: Length = 6, ends at byte 11 (7 header + 6 data = 13 bytes, 0-indexed = byte 11)
2. Second message starts at byte 12
3. Second message: Length = 6, ends at byte 23

**Wireshark handling:**
- Wireshark's MODBUS dissector automatically handles this
- Shows as separate MODBUS frames in same TCP packet
- Look for "MODBUS/TCP" protocol entries with same TCP frame number

#### Scenario 3: MODBUS Message Split Across TCP Packets

**Example:**

**TCP Packet 1 (8 bytes):**
```
00 01 00 00 00 06 01 03
│     │     │     │  └─ Function Code (start of PDU)
│     │     │     └─ Unit ID
│     │     └─ Length = 6 (need 6 more bytes)
│     └─ Protocol ID = 0
└─ Transaction ID = 1
```

**TCP Packet 2 (4 bytes):**
```
00 00 00 02
└─ Rest of PDU (Start Address + Quantity)
```

**Boundary recognition:**
- Length field in Packet 1 says 6 bytes follow
- Packet 1 only has 1 byte after Unit ID (Function Code)
- Need 5 more bytes → found in Packet 2
- Complete MODBUS message = Packet 1 (8 bytes) + Packet 2 (4 bytes) = 12 bytes

**Wireshark handling:**
- Wireshark TCP reassembly combines packets
- MODBUS dissector shows complete message
- May see `[TCP segment of a reassembled PDU]` in Info column

### Visual Boundary Recognition in Wireshark

**In the "Packet Bytes" pane:**
1. Look for MBAP header signature: `xx xx 00 00 xx xx yy`
   - Bytes 2-3 are always `00 00` (Protocol ID)
2. Read Length field (bytes 4-5) as big-endian
3. Count forward that many bytes from byte 6
4. Next message (if any) starts after

**Example packet bytes view:**
```
Offset  Hex                                                 ASCII
0000   00 01 00 00 00 06 01 03 00 00 00 02                 ............
       └─┬─┘ └─0000─┘ └06┘ └─┬┘ └──┬─────┘
       TID   Proto   Len   UID   PDU
```

## Using Wireshark's MODBUS Dissector

### Automatic Protocol Detection

Wireshark automatically detects MODBUS TCP when:
- TCP destination port = 502 (standard)
- TCP destination port = 802 (secure, pre-TLS handshake)
- Pattern matches MODBUS/TCP structure

**If not auto-detected:**
- Right-click packet → Decode As → Select "MODBUS/TCP"

### MODBUS Protocol Tree View

When Wireshark decodes MODBUS, the **Packet Details** pane shows:

```
▼ Modbus/TCP
  ▼ MBAP Header
      Transaction Identifier: 1
      Protocol Identifier: 0
      Length: 6
      Unit Identifier: 1
  ▼ Modbus
      Function Code: Read Holding Registers (3)
      Starting Address: 0
      Quantity: 2 registers
```

**Expandable sections:**
- **MBAP Header:** Transaction ID, Protocol ID, Length, Unit ID
- **Modbus (PDU):** Function code, addresses, quantities, data values

### Decoded Information

**For Read Requests:**
- Function Code name (e.g., "Read Holding Registers")
- Starting Address
- Quantity (number of registers/coils)

**For Read Responses:**
- Function Code (echoed)
- Byte Count
- **Register Values** (decoded as decimal and hex)

**For Write Requests:**
- Function Code name (e.g., "Write Multiple Registers")
- Starting Address
- Quantity
- **Data Values** (registers to write)

**For Write Responses:**
- Function Code (echoed)
- Starting Address
- Quantity (confirmation)

**For Exceptions:**
- Exception Function Code (original + 0x80)
- Exception Code with description:
  - 0x01: ILLEGAL FUNCTION
  - 0x02: ILLEGAL DATA ADDRESS
  - 0x03: ILLEGAL DATA VALUE
  - 0x04: SERVER DEVICE FAILURE

**Example decoded response:**
```
▼ Modbus
    Function Code: Read Holding Registers (3)
    Byte Count: 4
  ▼ Data
      Register 0 (UINT16): 10 (0x000a)
      Register 1 (UINT16): 20 (0x0014)
```

## Display Filters for MODBUS Analysis

### Basic Filters

**Show all MODBUS TCP:**
```
modbus
```

**Show only requests:**
```
modbus.func_code <= 127
```

**Show only responses:**
```
modbus
```
(Then manually inspect - Wireshark doesn't distinguish request/response in filter)

**Show only exceptions:**
```
modbus.func_code >= 128
```
or
```
modbus.except_code
```

### Function Code Filters

**Show specific function code:**
```
modbus.func_code == 3
```
(Read Holding Registers)

**Common function codes:**
```
modbus.func_code == 1    # Read Coils
modbus.func_code == 2    # Read Discrete Inputs
modbus.func_code == 3    # Read Holding Registers
modbus.func_code == 4    # Read Input Registers
modbus.func_code == 5    # Write Single Coil
modbus.func_code == 6    # Write Single Register
modbus.func_code == 15   # Write Multiple Coils
modbus.func_code == 16   # Write Multiple Registers
```

### Transaction ID Filters

**Show specific transaction:**
```
modbus.trans_id == 1
```

**Show request and response pair:**
```
modbus.trans_id == 1
```
(Both request and response have same Transaction ID)

### Address Range Filters

**Show operations on specific register:**
```
modbus.reference_num == 100
```

**Show operations in address range:**
```
modbus.reference_num >= 100 && modbus.reference_num <= 200
```

### Device Filters (IP Address)

**Show traffic to/from specific device:**
```
ip.addr == 192.168.1.100 and modbus
```

**Show traffic between two devices:**
```
ip.src == 192.168.1.50 and ip.dst == 192.168.1.100 and modbus
```

### Combined Filters

**Show Read Holding Registers from specific device:**
```
modbus.func_code == 3 and ip.addr == 192.168.1.100
```

**Show all exceptions:**
```
modbus.except_code
```

**Show write operations only:**
```
modbus.func_code == 5 or modbus.func_code == 6 or modbus.func_code == 15 or modbus.func_code == 16
```

## Byte-Level Analysis in Wireshark

### Accessing Byte-Level View

**Packet Bytes pane (bottom of Wireshark):**
- Shows raw packet data in hex and ASCII
- Left: Byte offset
- Middle: Hexadecimal values
- Right: ASCII representation

**Selecting bytes:**
- Click on field in Packet Details pane
- Corresponding bytes highlight in Packet Bytes pane

### Manual Byte-Level Inspection

**Example MODBUS TCP request packet:**

```
Frame 10: 66 bytes on wire (528 bits), 66 bytes captured
Ethernet II
Internet Protocol Version 4
Transmission Control Protocol, Src Port: 52341, Dst Port: 502
  [TCP Payload]: 12 bytes
    
Packet Bytes:
Offset  Hex                                                 ASCII
0000   00 01 00 00 00 06 01 03 00 00 00 02                 ............
```

**Manual decoding:**

| Offset | Bytes | Field | Value | Interpretation |
|--------|-------|-------|-------|----------------|
| 0-1 | `00 01` | Transaction ID | 0x0001 | Transaction 1 |
| 2-3 | `00 00` | Protocol ID | 0x0000 | MODBUS protocol |
| 4-5 | `00 06` | Length | 6 | 6 bytes follow |
| 6 | `01` | Unit ID | 1 | Device 1 |
| 7 | `03` | Function Code | 3 | Read Holding Registers |
| 8-9 | `00 00` | Start Address | 0 | Register 0 |
| 10-11 | `00 02` | Quantity | 2 | 2 registers |

**Verification:**
- Protocol ID = `00 00` ✓ (confirms MODBUS)
- Length = 6 = 1 (Unit ID) + 5 (PDU) ✓
- Function Code = 3 ✓ (valid)

### Analyzing Response Data Values

**Example MODBUS TCP response:**

```
Packet Bytes:
Offset  Hex                                                 ASCII
0000   00 01 00 00 00 07 01 03 04 00 0a 00 14             .............
```

**Manual decoding:**

| Offset | Bytes | Field | Value | Interpretation |
|--------|-------|-------|-------|----------------|
| 0-1 | `00 01` | Transaction ID | 0x0001 | Response to Transaction 1 |
| 2-3 | `00 00` | Protocol ID | 0x0000 | MODBUS protocol |
| 4-5 | `00 07` | Length | 7 | 7 bytes follow |
| 6 | `01` | Unit ID | 1 | From Device 1 |
| 7 | `03` | Function Code | 3 | Read Holding Registers |
| 8 | `04` | Byte Count | 4 | 4 bytes of data |
| 9-10 | `00 0a` | Register 0 | 0x000a | Decimal: 10 |
| 11-12 | `00 14` | Register 1 | 0x0014 | Decimal: 20 |

**Data value extraction:**
- Register values are 16-bit (2 bytes)
- Big-endian byte order
- Register 0: `00 0a` = (0x00 << 8) | 0x0a = 10
- Register 1: `00 14` = (0x00 << 8) | 0x14 = 20

### Analyzing Exception Responses

**Example exception response:**

```
Packet Bytes:
Offset  Hex                                                 ASCII
0000   00 01 00 00 00 03 01 83 02                          .........
```

**Manual decoding:**

| Offset | Bytes | Field | Value | Interpretation |
|--------|-------|-------|-------|----------------|
| 0-1 | `00 01` | Transaction ID | 0x0001 | Response to Transaction 1 |
| 2-3 | `00 00` | Protocol ID | 0x0000 | MODBUS protocol |
| 4-5 | `00 03` | Length | 3 | 3 bytes follow |
| 6 | `01` | Unit ID | 1 | From Device 1 |
| 7 | `83` | Function Code | 0x83 | Exception: 0x03 + 0x80 |
| 8 | `02` | Exception Code | 2 | ILLEGAL DATA ADDRESS |

**Exception identification:**
- Function Code MSB = 1 (0x83 = 0b10000011)
- Original function = 0x83 - 0x80 = 0x03 (Read Holding Registers)
- Exception Code 0x02 = address out of range

## Following MODBUS Conversations

### TCP Stream Following

**Purpose:** See complete conversation between client and server

**Steps:**
1. Select any MODBUS packet from desired conversation
2. Right-click → Follow → TCP Stream
3. Wireshark shows complete TCP stream

**Result:**
- All data exchanged between client and server
- Color-coded (red = client→server, blue = server→client)
- Shows multiple MODBUS transactions in sequence

**Limitations:**
- Shows raw bytes (not decoded MODBUS)
- Better for seeing overall pattern

### Request/Response Pairing

**Using Transaction ID:**
1. Note Transaction ID from request (e.g., 0x0001)
2. Apply filter: `modbus.trans_id == 1`
3. See both request and response with same Transaction ID

**Using Wireshark's "Time" column:**
- Sort by time to see request→response sequence
- Calculate response time (difference between request and response timestamps)

### Analyzing Performance

**Response time calculation:**
1. Note timestamp of request packet
2. Note timestamp of response packet
3. Difference = response time

**Wireshark can calculate automatically:**
- Statistics → Flow Graph
- Shows timing between packets

**Common metrics:**
- Request→Response time (server processing + network)
- Requests per second
- Error rate (exceptions / total requests)

## Analyzing MODBUS/TCP Security (Port 802)

### What Wireshark Shows for Encrypted Traffic

**TLS Handshake (Visible):**
```
▼ TLSv1.2 Record Layer: Handshake Protocol: Client Hello
    Content Type: Handshake (22)
    Version: TLS 1.2 (0x0303)
    Length: ...
  ▼ Handshake Protocol: Client Hello
      Handshake Type: Client Hello (1)
      Cipher Suites:
        Cipher Suite: TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
```

**What you can see:**
- TLS version (should be 1.2 or 1.3)
- Cipher suite negotiation
- Certificate exchange (X.509 certificates)
- Client/Server Hello messages

**TLS Application Data (Encrypted):**
```
▼ TLSv1.2 Record Layer: Application Data Protocol
    Content Type: Application Data (23)
    Version: TLS 1.2 (0x0303)
    Length: 29
    Encrypted Application Data: [29 bytes]
```

**What you CANNOT see:**
- MBAP header contents (encrypted)
- MODBUS function codes (encrypted)
- Register addresses (encrypted)
- Data values (encrypted)

Source: [modbus-tcp-security.md](/wiki/concepts/modbus-tcp-security.md:34)

### Recognizing MODBUS/TCP Security Traffic

**Port number:**
- Destination port = 802 (not 502)

**Protocol detection:**
- Wireshark shows "TLSv1.2" or "TLSv1.3" in Protocol column
- Not "MODBUS/TCP" (encrypted payload)

**Certificate analysis:**
1. Find TLS handshake packets
2. Expand "Handshake Protocol: Certificate"
3. Examine X.509 certificate details:
   - Subject/Issuer
   - Validity period
   - **Look for MODBUS role extension** (OID 1.3.6.1.4.1.50316.802.1)

**Cipher suite verification:**
- Look for: `TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256`
- Required for MODBUS/TCP Security

### Display Filters for MODBUS/TCP Security

**Show all TLS traffic on port 802:**
```
tcp.port == 802 and tls
```

**Show TLS handshakes only:**
```
tls.handshake
```

**Show TLS application data (encrypted MODBUS):**
```
tls.record.content_type == 23
```

**Show certificate exchange:**
```
tls.handshake.type == 11
```

**Show specific cipher suite:**
```
tls.handshake.ciphersuite == 0xc02f
```
(0xc02f = TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256)

### Decrypting MODBUS/TCP Security Traffic (Advanced)

**Wireshark CAN decrypt TLS if you provide session keys.**

**Method 1: Pre-Master Secret (requires client/server cooperation)**

**Requirements:**
- Client or server logs TLS session keys to file
- Set environment variable before starting application:
  ```bash
  export SSLKEYLOGFILE=/path/to/sslkeys.log
  ```

**Configure Wireshark:**
1. Edit → Preferences → Protocols → TLS
2. "(Pre)-Master-Secret log filename" → Browse to sslkeys.log
3. Click OK

**Result:**
- Wireshark decrypts TLS application data
- MODBUS dissector can now decode MODBUS frames
- See MBAP header, function codes, data values

**Method 2: RSA Private Key (limited use)**

**Requirements:**
- Server's RSA private key
- Cipher suite must NOT use ECDHE (no perfect forward secrecy)

**Note:** MODBUS/TCP Security requires ECDHE cipher, so this method won't work for compliant implementations.

**Why decryption is hard:**
- ECDHE provides perfect forward secrecy
- Session keys derived from ephemeral Diffie-Hellman exchange
- Cannot decrypt without live session key capture

## Common MODBUS Analysis Tasks in Wireshark

### Task 1: Verify Communication is Happening

**Filter:**
```
modbus
```

**Check:**
- Packets appearing in capture?
- Both requests and responses?
- Transaction IDs incrementing?

**If no packets:**
- Verify port 502 traffic: `tcp.port == 502`
- Check if traffic exists at all
- Verify capture interface correct

### Task 2: Identify Which Registers Are Being Read/Written

**Filter for reads:**
```
modbus.func_code == 3 or modbus.func_code == 4
```

**Look at Packet Details:**
- Starting Address (modbus.reference_num)
- Quantity (modbus.word_cnt)

**Calculate range:**
- Range = [Starting Address, Starting Address + Quantity - 1]

**Filter for writes:**
```
modbus.func_code == 6 or modbus.func_code == 16
```

### Task 3: Extract Register Values

**Method 1: Use Wireshark dissector**
- Expand "Modbus" → "Data"
- See decoded register values

**Method 2: Manual extraction**
- Look at Packet Bytes pane
- Find data section (after Byte Count field)
- Extract 2-byte pairs (big-endian)

**Example:**
```
Byte Count: 04
Data: 00 0a 00 14

Register 0 = 0x000a = 10
Register 1 = 0x0014 = 20
```

### Task 4: Measure Response Times

**Method 1: Manual calculation**
1. Note timestamp of request
2. Note timestamp of response
3. Subtract: Response Time = Response Timestamp - Request Timestamp

**Method 2: Statistics → Flow Graph**
- Visualizes packet timing
- Shows request→response flow

**Typical MODBUS TCP response times:**
- LAN: 1-10 ms
- WAN: 10-100 ms
- Slow device: 100-500 ms

### Task 5: Identify Communication Errors

**Look for:**

**TCP-level errors:**
```
tcp.analysis.flags
```
- Retransmissions
- Out-of-order packets
- Zero window (flow control issues)

**MODBUS exceptions:**
```
modbus.except_code
```
- Function code ≥ 128
- Exception codes: 0x01, 0x02, 0x03, 0x04

**No response (timeout):**
- Request with no matching response (same Transaction ID)
- Use filter: `modbus.trans_id == X` to check if response exists

### Task 6: Debug Gateway Routing

**Look at Unit ID field:**
```
modbus.unit_id
```

**Values:**
- 0xFF (255): Direct TCP connection
- 0x01-0xF7 (1-247): Routing through gateway to serial slave

**Verify routing:**
1. Check request Unit ID
2. Check response Unit ID (should match)
3. If Unit ID < 0xFF, communication going through gateway

**Gateway scenarios:**
```
Client → Gateway (IP: 192.168.1.100, Unit ID: 5) → RS-485 Bus → Slave #5
```

Filter for specific slave:
```
modbus.unit_id == 5 and ip.addr == 192.168.1.100
```

## Troubleshooting with Wireshark

### Problem: No MODBUS Packets Captured

**Checks:**
1. Verify port 502 traffic exists: `tcp.port == 502`
2. Check IP addresses: `ip.addr == 192.168.1.100`
3. Verify capture interface is correct
4. Check for promiscuous mode (may be needed on switched networks)
5. Consider using port mirroring/SPAN on managed switch

### Problem: Wireshark Not Decoding as MODBUS

**Solutions:**
1. Verify port 502 is used
2. Right-click packet → Decode As → Select "MODBUS/TCP"
3. Check Protocol ID field = 0x0000
4. Update Wireshark to latest version

### Problem: Incomplete MODBUS Messages

**Symptom:** `[TCP segment of a reassembled PDU]`

**Cause:** MODBUS message split across multiple TCP packets

**Solution:**
- Enable TCP reassembly: Edit → Preferences → Protocols → TCP → "Allow subdissector to reassemble TCP streams"
- Wireshark will automatically reassemble and show complete MODBUS message

### Problem: Cannot See Response

**Checks:**
1. Filter by Transaction ID: `modbus.trans_id == X`
2. Check if response exists at all
3. Look for TCP RST (connection reset)
4. Look for MODBUS exception response
5. Check server is actually responding (timeout?)

### Problem: Encrypted Traffic (Port 802) Not Decrypting

**Reality check:**
- Without TLS session keys, you CANNOT decrypt
- This is intentional security feature

**Options:**
1. Capture TLS session keys (SSLKEYLOGFILE)
2. Analyze TLS handshake (certificates, cipher suite) instead
3. Use standard MODBUS TCP (port 502) for debugging, secure (port 802) for production

## Export and Analysis

### Export Packet Dissections

**Export as plain text:**
- File → Export Packet Dissections → As Plain Text
- Choose "Packet Details" or "Packet Bytes"

**Export as CSV:**
- File → Export Packet Dissections → As CSV
- Columns: frame number, time, source, destination, protocol, info

### Extract Specific Values

**Using tshark (command-line Wireshark):**

**Extract all register values:**
```bash
tshark -r capture.pcap -Y "modbus.func_code == 3" -T fields -e modbus.data
```

**Extract Transaction IDs:**
```bash
tshark -r capture.pcap -Y "modbus" -T fields -e modbus.trans_id
```

**Count MODBUS requests:**
```bash
tshark -r capture.pcap -Y "modbus.func_code <= 127" | wc -l
```

**Calculate average response time (requires scripting):**
- Export timestamps and Transaction IDs
- Match requests with responses
- Calculate time differences

## Summary: Wireshark Capabilities

| Capability | Standard MODBUS TCP | MODBUS/TCP Security |
|------------|---------------------|---------------------|
| **Capture traffic** | ✅ Yes | ✅ Yes |
| **Decode MBAP header** | ✅ Yes (automatic) | ❌ No (encrypted) |
| **Decode function codes** | ✅ Yes (automatic) | ❌ No (encrypted) |
| **See register values** | ✅ Yes | ❌ No (encrypted) |
| **See TLS handshake** | N/A | ✅ Yes |
| **See certificates** | N/A | ✅ Yes |
| **Decrypt with session keys** | N/A | ✅ Yes (advanced) |
| **Decrypt without keys** | N/A | ❌ No (by design) |
| **Filter by function code** | ✅ Yes | ❌ No (encrypted) |
| **Measure response time** | ✅ Yes | ✅ Yes (TCP-level timing) |
| **Identify exceptions** | ✅ Yes | ❌ No (encrypted) |

## Related Pages

- [MODBUS TCP Message Format](/wiki/answers/modbus-tcp-message-format.md) - Understanding MODBUS TCP structure
- [MODBUS TCP](/wiki/concepts/modbus-tcp.md) - MODBUS TCP protocol overview
- [MBAP Header](/wiki/concepts/mbap-header.md) - MBAP header specification
- [MODBUS TCP Security](/wiki/concepts/modbus-tcp-security.md) - Secure MODBUS protocol
- [Connecting with MODBUS/TCP Security](/wiki/answers/connecting-with-modbus-tcp-security.md) - Security setup

## Backlinks

- [How MODBUS TCP-to-RTU Gateways Work](/wiki/answers/modbus-tcp-to-rtu-gateway.md)
- [MODBUS Commissioning Checklist](/wiki/answers/modbus-commissioning-checklist.md)
- [MODBUS Errors and Exception Responses](/wiki/answers/modbus-errors-and-exceptions.md)
- [Reading MODBUS Register Maps](/wiki/answers/reading-modbus-register-maps.md)
- [MBAP Header](/wiki/concepts/mbap-header.md)
- [MODBUS TCP](/wiki/concepts/modbus-tcp.md)
- [MODBUS TCP Security](/wiki/concepts/modbus-tcp-security.md)

------------

MODBUS is a trademark of the Modbus Organization, Inc.

All information on this page is derived by summarizing and analyzing solely the MODBUS specifications published by the Modbus Organization, Inc.  The specifications are copyright by the Modbus Organization.

By deriving this information solely from the specifications we hope to stay true to the specification.

No infringement is intended.
