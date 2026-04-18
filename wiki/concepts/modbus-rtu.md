---
title: MODBUS RTU
Summary: MODBUS Remote Terminal Unit mode for serial line communication using binary encoding and CRC-16 error checking.
Sources:
  - raw/MODBUS/MODBUS.md
  - raw/MODBUS/modbusoverserial.md
Categories:
  - protocol-variants
  - serial
  - rs-485
date-created: 2026-04-18T12:00:00+03:00
last-updated: 2026-04-18T14:43:24+03:00
---

MODBUS RTU (Remote Terminal Unit) is a binary transmission mode for MODBUS over serial lines, providing efficient communication with CRC-16 error checking (source: [modbusoverserial.md](raw/MODBUS/modbusoverserial.md)).

## Protocol Overview

RTU mode is the most commonly used MODBUS serial mode, transmitting data in binary format with efficient error checking through CRC-16 (source: [modbusoverserial.md](raw/MODBUS/modbusoverserial.md)).

### Key Characteristics

| Property | Value |
|----------|-------|
| Mode | Binary transmission |
| Error checking | CRC-16 |
| Frame delimiter | Silent intervals (t1.5, t3.5) |
| Character time | 11 bits (with parity), 10 bits (no parity) |
| Maximum frame size | 256 bytes |
| Character format | 8 data bits, LSB first |

## Character Format

RTU mode uses a specific character format for serial transmission (source: [modbusoverserial.md](raw/MODBUS/modbusoverserial.md)):

| Field | Bits | Description |
|-------|------|-------------|
| Start bit | 1 | Always 0 |
| Data bits | 8 | LSB first |
| Parity bit | 1 | Even parity (default), odd, or none |
| Stop bit(s) | 1-2 | 1 with parity, 2 without |
| **Total** | **11** (with parity) or **10** (without parity) | Bits per character |

**Default configuration:** 8 data bits, even parity, 1 stop bit (8E1)
**Alternative (no parity):** 8 data bits, no parity, 2 stop bits (8N2)

## Frame Structure

### RTU ADU Structure

```
[Address: 1][Function Code: 1][Data: 0-252][CRC-16: 2]
|<- Slave ->|<-------- PDU ---------->|<- CRC ->|
```

**Maximum RTU ADU: 256 bytes** (1 address + 253 PDU + 2 CRC)

### Frame Components

**Address Field (1 byte):**
- Slave device address (1-247)
- Address 0 reserved for broadcast (no response)
- All slaves check if address matches

**Function Code (1 byte):**
- Specifies operation to perform
- 0x01-0x7F for normal operations
- 0x80-0xFF for exception responses

**Data Field (0-252 bytes):**
- Request parameters or response data
- Content depends on function code

**CRC-16 (2 bytes):**
- Error checking for entire frame
- Calculated over Address, Function Code, and Data
- Transmitted LSB first (little-endian) - **critical difference**

## Framing

### Silent Intervals

RTU uses silent intervals (no character transmission) to delimit frames (source: [modbusoverserial.md](raw/MODBUS/modbusoverserial.md)):

| Timer | Duration | Purpose |
|-------|----------|---------|
| t1.5 | 1.5 character times | Max gap between characters within frame |
| t3.5 | 3.5 character times | Min gap between frames |

### Timing Calculation

**Character Time Formula:**
```
Character time (seconds) = 11 / baud_rate  (with parity)
                         = 10 / baud_rate  (no parity)
```

**Timing Examples (11 bits/char):**

| Baud Rate | Char Time | t1.5 | t3.5 |
|-----------|-----------|------|------|
| 9600 | 1.146 ms | 1.72 ms | 4.01 ms |
| 19200 | 0.573 ms | 0.86 ms | 2.01 ms |
| 38400 | 0.286 ms | 0.43 ms | 1.00 ms |
| 115200 | 0.095 ms | 0.14 ms | 0.33 ms |

### High Baud Rate Rule

For baud rates > 19200 bps, use **fixed timing values**:
- t1.5 = 750 µs (inter-character timeout)
- t3.5 = 1.75 ms (inter-frame delay)

This simplifies implementation at high baud rates where precise timing becomes impractical.

### Frame Reception Rules

1. Frame starts after t3.5 silence followed by first character
2. Characters must arrive within t1.5 of each other
3. If t1.5 exceeded between characters → frame error, discard frame
4. Frame ends when t3.5 silence detected after last character
5. Validate CRC-16 before processing frame

## CRC-16 Calculation

### CRC-16 Parameters

| Parameter | Value |
|-----------|-------|
| Polynomial | 0x8005 (bit-reversed: 0xA001) |
| Initial value | 0xFFFF |
| Input reflection | Yes (LSB first) |
| Output reflection | Yes |
| Final XOR | 0x0000 |
| Byte order | LSB first (little-endian) |

**Critical:** CRC bytes are transmitted LSB first, unlike all other MODBUS fields which are big-endian (source: [MODBUS.md](raw/MODBUS/MODBUS.md)).

### CRC-16 Algorithm

**Pseudocode:**
```
function crc16_modbus(data: bytes) -> uint16:
    crc = 0xFFFF
    for byte in data:
        crc = crc XOR byte
        for i in 0..7:
            if (crc AND 0x0001) != 0:
                crc = (crc >> 1) XOR 0xA001
            else:
                crc = crc >> 1
    return crc  // Return as-is; transmit LSB first
```

### CRC-16 C Implementation (Lookup Table)

```c
/* High-order byte table */
static const unsigned char auchCRCHi[] = {
    0x00, 0xC1, 0x81, 0x40, 0x01, 0xC0, 0x80, 0x41, 0x01, 0xC0, 0x80, 0x41, 0x00, 0xC1, 0x81, 0x40,
    0x01, 0xC0, 0x80, 0x41, 0x00, 0xC1, 0x81, 0x40, 0x00, 0xC1, 0x81, 0x40, 0x01, 0xC0, 0x80, 0x41,
    0x01, 0xC0, 0x80, 0x41, 0x00, 0xC1, 0x81, 0x40, 0x00, 0xC1, 0x81, 0x40, 0x01, 0xC0, 0x80, 0x41,
    0x00, 0xC1, 0x81, 0x40, 0x01, 0xC0, 0x80, 0x41, 0x01, 0xC0, 0x80, 0x41, 0x00, 0xC1, 0x81, 0x40,
    0x01, 0xC0, 0x80, 0x41, 0x00, 0xC1, 0x81, 0x40, 0x00, 0xC1, 0x81, 0x40, 0x01, 0xC0, 0x80, 0x41,
    0x01, 0xC0, 0x80, 0x41, 0x00, 0xC1, 0x81, 0x40, 0x01, 0xC0, 0x80, 0x41, 0x01, 0xC0, 0x80, 0x41,
    0x00, 0xC1, 0x81, 0x40, 0x00, 0xC1, 0x81, 0x40, 0x01, 0xC0, 0x80, 0x41, 0x01, 0xC0, 0x80, 0x41,
    0x00, 0xC1, 0x81, 0x40, 0x01, 0xC0, 0x80, 0x41, 0x00, 0xC1, 0x81, 0x40, 0x00, 0xC1, 0x81, 0x40,
    0x01, 0xC0, 0x80, 0x41, 0x01, 0xC0, 0x80, 0x41, 0x00, 0xC1, 0x81, 0x40, 0x00, 0xC1, 0x81, 0x40,
    0x01, 0xC0, 0x80, 0x41, 0x00, 0xC1, 0x81, 0x40, 0x01, 0xC0, 0x80, 0x41, 0x01, 0xC0, 0x80, 0x41,
    0x00, 0xC1, 0x81, 0x40, 0x00, 0xC1, 0x81, 0x40, 0x01, 0xC0, 0x80, 0x41, 0x01, 0xC0, 0x80, 0x41,
    0x00, 0xC1, 0x81, 0x40, 0x01, 0xC0, 0x80, 0x41, 0x00, 0xC1, 0x81, 0x40, 0x00, 0xC1, 0x81, 0x40,
    0x01, 0xC0, 0x80, 0x41, 0x01, 0xC0, 0x80, 0x41, 0x00, 0xC1, 0x81, 0x40, 0x01, 0xC0, 0x80, 0x41,
    0x00, 0xC1, 0x81, 0x40, 0x00, 0xC1, 0x81, 0x40, 0x01, 0xC0, 0x80, 0x41, 0x01, 0xC0, 0x80, 0x41,
    0x00, 0xC1, 0x81, 0x40, 0x01, 0xC0, 0x80, 0x41, 0x00, 0xC1, 0x81, 0x40, 0x00, 0xC1, 0x81, 0x40,
    0x01, 0xC0, 0x80, 0x41, 0x00, 0xC1, 0x81, 0x40, 0x01, 0xC0, 0x80, 0x41, 0x00, 0xC1, 0x81, 0x40
};

/* Low-order byte table */
static const unsigned char auchCRCLo[] = {
    0x00, 0xC0, 0xC1, 0x01, 0xC3, 0x03, 0x02, 0xC2, 0xC6, 0x06, 0x07, 0xC7, 0x05, 0xC5, 0xC4, 0x04,
    0xCC, 0x0C, 0x0D, 0xCD, 0x0F, 0xCF, 0xCE, 0x0E, 0x0A, 0xCA, 0xCB, 0x0B, 0xC9, 0x09, 0x08, 0xC8,
    0xD8, 0x18, 0x19, 0xD9, 0x1B, 0xDB, 0xDA, 0x1A, 0x1E, 0xDE, 0xDF, 0x1F, 0xDD, 0x1D, 0x1C, 0xDC,
    0x14, 0xD4, 0xD5, 0x15, 0xD7, 0x17, 0x16, 0xD6, 0xD2, 0x12, 0x13, 0xD3, 0x11, 0xD1, 0xD0, 0x10,
    0xF0, 0x30, 0x31, 0xF1, 0x33, 0xF3, 0xF2, 0x32, 0x36, 0xF6, 0xF7, 0x37, 0xF5, 0x35, 0x34, 0xF4,
    0x3C, 0xFC, 0xFD, 0x3D, 0xFF, 0x3F, 0x3E, 0xFE, 0xFA, 0x3A, 0x3B, 0xFB, 0x39, 0xF9, 0xF8, 0x38,
    0x28, 0xE8, 0xE9, 0x29, 0xEB, 0x2B, 0x2A, 0xEA, 0xEE, 0x2E, 0x2F, 0xEF, 0x2D, 0xED, 0xEC, 0x2C,
    0xE4, 0x24, 0x25, 0xE5, 0x27, 0xE7, 0xE6, 0x26, 0x22, 0xE2, 0xE3, 0x23, 0xE1, 0x21, 0x20, 0xE0,
    0xA0, 0x60, 0x61, 0xA1, 0x63, 0xA3, 0xA2, 0x62, 0x66, 0xA6, 0xA7, 0x67, 0xA5, 0x65, 0x64, 0xA4,
    0x6C, 0xAC, 0xAD, 0x6D, 0xAF, 0x6F, 0x6E, 0xAE, 0xAA, 0x6A, 0x6B, 0xAB, 0x69, 0xA9, 0xA8, 0x68,
    0x78, 0xB8, 0xB9, 0x79, 0xBB, 0x7B, 0x7A, 0xBA, 0xBE, 0x7E, 0x7F, 0xBF, 0x7D, 0xBD, 0xBC, 0x7C,
    0xB4, 0x74, 0x75, 0xB5, 0x77, 0xB7, 0xB6, 0x76, 0x72, 0xB2, 0xB3, 0x73, 0xB1, 0x71, 0x70, 0xB0,
    0x50, 0x90, 0x91, 0x51, 0x93, 0x53, 0x52, 0x92, 0x96, 0x56, 0x57, 0x97, 0x55, 0x95, 0x94, 0x54,
    0x9C, 0x5C, 0x5D, 0x9D, 0x5F, 0x9F, 0x9E, 0x5E, 0x5A, 0x9A, 0x9B, 0x5B, 0x99, 0x59, 0x58, 0x98,
    0x88, 0x48, 0x49, 0x89, 0x4B, 0x8B, 0x8A, 0x4A, 0x4E, 0x8E, 0x8F, 0x4F, 0x8D, 0x4D, 0x4C, 0x8C,
    0x44, 0x84, 0x85, 0x45, 0x87, 0x47, 0x46, 0x86, 0x82, 0x42, 0x43, 0x83, 0x41, 0x81, 0x80, 0x40
};

unsigned short CRC16(unsigned char *puchMsg, unsigned short usDataLen)
{
    unsigned char uchCRCHi = 0xFF;
    unsigned char uchCRCLo = 0xFF;
    unsigned uIndex;

    while (usDataLen--) {
        uIndex = uchCRCLo ^ *puchMsg++;
        uchCRCLo = uchCRCHi ^ auchCRCHi[uIndex];
        uchCRCHi = auchCRCLo[uIndex];
    }

    return (uchCRCHi << 8 | uchCRCLo);
}
```

### CRC Example

**Request: Read 10 registers from slave 1**
```
Frame data: [01] [03] [00] [00] [00] [0A]
CRC-16:     0xC5CD
Transmitted: [01] [03] [00] [00] [00] [0A] [CD] [C5]
                                            LSB  MSB
```

## Master/Slave Protocol

### Master Operations

**Transmission:**
1. Wait for t3.5 silence before sending
2. Build frame: [Address][Function][Data][CRC-Lo][CRC-Hi]
3. Transmit all bytes as continuous stream
4. Start response timeout after transmission

**Reception:**
1. Detect frame start after t3.5 silence
2. Accumulate bytes, verify t1.5 inter-character timing
3. Detect frame end on t3.5 silence
4. Validate CRC-16
5. Verify response address matches request

### Slave Operations

**Reception:**
1. Detect frame start after t3.5 silence
2. Validate t1.5 inter-character timing
3. Check CRC-16
4. Verify address (own address or broadcast 0)
5. Discard frames with errors silently (no response)

**Processing:**
1. Parse function code and data
2. Execute requested operation or generate exception
3. For broadcast: execute but don't respond

**Transmission:**
1. Build response: [Address][Function][Data][CRC-Lo][CRC-Hi]
2. Wait for turnaround time if needed
3. Transmit all bytes as continuous stream

## Comparison with Other Modes

| Characteristic | RTU | ASCII | TCP |
|---------------|-----|-------|-----|
| Efficiency | High (binary) | Low (ASCII hex) | High (binary) |
| Error checking | CRC-16 | LRC | TCP checksum |
| Timing | Strict (t1.5, t3.5) | Relaxed (1s timeout) | N/A (framing by TCP) |
| Character time | 11 bits | 10 bits | N/A |
| Max frame size | 256 bytes | 513 chars | 260 bytes |
| Debugging | Binary analyzer | Human-readable | Binary analyzer |
| Medium | Serial (RS485/232) | Serial (RS485/232) | Ethernet |

## Physical Layer

RTU mode typically operates over RS-485 (2-wire or 4-wire) or RS232 physical layers. See [[wiki/concepts/rs485]] for physical layer specifications.

## Related pages

- [[wiki/concepts/modbus-ascii]]
- [[wiki/concepts/modbus-tcp]]
- [[wiki/concepts/crc-16]]
- [[wiki/concepts/rs485]]
- [[wiki/concepts/function-codes]]
