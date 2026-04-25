---
title: CRC-16
Summary: Cyclic Redundancy Check 16-bit error detection algorithm used in MODBUS RTU mode for frame integrity.
Sources:
  - raw/MODBUS/MODBUS.md
  - raw/MODBUS/modbusoverserial.md
Categories:
  - protocol-components
  - error-detection
  - rtu-mode
type: concept
date-created: 2026-04-18T12:00:00+03:00
last-updated: 2026-04-18T14:43:24+03:00
---

CRC-16 (Cyclic Redundancy Check) is a 16-bit error detection algorithm used in MODBUS RTU mode to ensure frame integrity over serial communication (source: [modbusoverserial.md](/raw/MODBUS/modbusoverserial.md)).

## Overview

CRC-16 provides robust error detection for MODBUS RTU frames by calculating a checksum based on the entire frame contents (excluding the CRC bytes themselves) (source: [modbusoverserial.md](/raw/MODBUS/modbusoverserial.md)).

### Key Characteristics

| Property | Value |
|----------|-------|
| CRC size | 16 bits (2 bytes) |
| Polynomial | 0x8005 (bit-reversed: 0xA001) |
| Initial value | 0xFFFF |
| Input reflection | Yes (LSB first) |
| Output reflection | Yes |
| Final XOR | 0x0000 |
| Byte order | LSB first (little-endian) |

**Critical:** CRC bytes are transmitted LSB first, unlike all other MODBUS fields which are big-endian (source: [MODBUS.md](/raw/MODBUS/MODBUS.md)).

## CRC-16 Parameters

| Parameter | Value | Description |
|-----------|-------|-------------|
| Polynomial | 0x8005 | Generator polynomial |
| Reversed polynomial | 0xA001 | Bit-reversed for LSB-first implementation |
| Initial value | 0xFFFF | Starting CRC value |
| Input reflection | Yes | Process bits LSB first |
| Output reflection | Yes | Output bits LSB first |
| Final XOR | 0x0000 | XOR result with 0x0000 |

## CRC-16 Calculation Algorithm

### Bit-by-Bit Algorithm

The fundamental CRC-16 algorithm processes each bit sequentially (source: [modbusoverserial.md](/raw/MODBUS/modbusoverserial.md)):

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
    return crc
```

### C Implementation (Bit-by-Bit)

```c
/**
 * Calculate CRC-16 for MODBUS RTU (bit-by-bit)
 * @param data Pointer to message buffer
 * @param length Number of bytes in message
 * @return CRC value
 */
uint16_t crc16_modbus_bitwise(const uint8_t *data, int length) {
    uint16_t crc = 0xFFFF;

    for (int i = 0; i < length; i++) {
        crc ^= data[i];
        for (int j = 0; j < 8; j++) {
            if (crc & 0x0001) {
                crc = (crc >> 1) ^ 0xA001;
            } else {
                crc >>= 1;
            }
        }
    }

    return crc;
}
```

## Lookup Table Implementation

Lookup table implementation provides significantly better performance by processing one byte at a time instead of one bit (source: [modbusoverserial.md](/raw/MODBUS/modbusoverserial.md)).

### CRC-16 Lookup Tables

**High-order byte table (auchCRCHi):**
```c
static const uint8_t auchCRCHi[] = {
    0x00, 0xC1, 0x81, 0x40, 0x01, 0xC0, 0x80, 0x41, 0x01, 0xC0, 0x80, 0x41,
    0x00, 0xC1, 0x81, 0x40, 0x01, 0xC0, 0x80, 0x41, 0x00, 0xC1, 0x81, 0x40,
    0x00, 0xC1, 0x81, 0x40, 0x01, 0xC0, 0x80, 0x41, 0x01, 0xC0, 0x80, 0x41,
    0x00, 0xC1, 0x81, 0x40, 0x00, 0xC1, 0x81, 0x40, 0x01, 0xC0, 0x80, 0x41,
    0x01, 0xC0, 0x80, 0x41, 0x00, 0xC1, 0x81, 0x40, 0x00, 0xC1, 0x81, 0x40,
    0x01, 0xC0, 0x80, 0x41, 0x01, 0xC0, 0x80, 0x41, 0x00, 0xC1, 0x81, 0x40,
    0x01, 0xC0, 0x80, 0x41, 0x00, 0xC1, 0x81, 0x40, 0x00, 0xC1, 0x81, 0x40,
    0x01, 0xC0, 0x80, 0x41, 0x01, 0xC0, 0x80, 0x41, 0x00, 0xC1, 0x81, 0x40,
    0x00, 0xC1, 0x81, 0x40, 0x01, 0xC0, 0x80, 0x41, 0x01, 0xC0, 0x80, 0x41,
    0x00, 0xC1, 0x81, 0x40, 0x00, 0xC1, 0x81, 0x40, 0x01, 0xC0, 0x80, 0x41,
    0x00, 0xC1, 0x81, 0x40, 0x01, 0xC0, 0x80, 0x41, 0x01, 0xC0, 0x80, 0x41,
    0x00, 0xC1, 0x81, 0x40, 0x00, 0xC1, 0x81, 0x40, 0x01, 0xC0, 0x80, 0x41,
    0x01, 0xC0, 0x80, 0x41, 0x00, 0xC1, 0x81, 0x40, 0x00, 0xC1, 0x81, 0x40,
    0x01, 0xC0, 0x80, 0x41, 0x01, 0xC0, 0x80, 0x41, 0x00, 0xC1, 0x81, 0x40,
    0x01, 0xC0, 0x80, 0x41, 0x00, 0xC1, 0x81, 0x40, 0x00, 0xC1, 0x81, 0x40,
    0x01, 0xC0, 0x80, 0x41, 0x00, 0xC1, 0x81, 0x40, 0x01, 0xC0, 0x80, 0x41,
    0x01, 0xC0, 0x80, 0x41, 0x00, 0xC1, 0x81, 0x40, 0x00, 0xC1, 0x81, 0x40,
    0x01, 0xC0, 0x80, 0x41, 0x00, 0xC1, 0x81, 0x40, 0x01, 0xC0, 0x80, 0x41,
    0x01, 0xC0, 0x80, 0x41, 0x00, 0xC1, 0x81, 0x40, 0x00, 0xC1, 0x81, 0x40,
    0x01, 0xC0, 0x80, 0x41, 0x01, 0xC0, 0x80, 0x41, 0x00, 0xC1, 0x81, 0x40
};
```

**Low-order byte table (auchCRCLo):**
```c
static const uint8_t auchCRCLo[] = {
    0x00, 0xC0, 0xC1, 0x01, 0xC3, 0x03, 0x02, 0xC2, 0xC6, 0x06, 0x07, 0xC7,
    0x05, 0xC5, 0xC4, 0x04, 0xCC, 0x0C, 0x0D, 0xCD, 0x0F, 0xCF, 0xCE, 0x0E,
    0x0A, 0xCA, 0xCB, 0x0B, 0xC9, 0x09, 0x08, 0xC8, 0xD8, 0x18, 0x19, 0xD9,
    0x1B, 0xDB, 0xDA, 0x1A, 0x1E, 0xDE, 0xDF, 0x1F, 0xDD, 0x1D, 0x1C, 0xDC,
    0x14, 0xD4, 0xD5, 0x15, 0xD7, 0x17, 0x16, 0xD6, 0xD2, 0x12, 0x13, 0xD3,
    0x11, 0xD1, 0xD0, 0x10, 0xF0, 0x30, 0x31, 0xF1, 0x33, 0xF3, 0xF2, 0x32,
    0x36, 0xF6, 0xF7, 0x37, 0xF5, 0x35, 0x34, 0xF4, 0x3C, 0xFC, 0xFD, 0x3D,
    0xFF, 0x3F, 0x3E, 0xFE, 0xFA, 0x3A, 0x3B, 0xFB, 0x39, 0xF9, 0xF8, 0x38,
    0x28, 0xE8, 0xE9, 0x29, 0xEB, 0x2B, 0x2A, 0xEA, 0xEE, 0x2E, 0x2F, 0xEF,
    0x2D, 0xED, 0xEC, 0x2C, 0xE4, 0x24, 0x25, 0xE5, 0x27, 0xE7, 0xE6, 0x26,
    0x22, 0xE2, 0xE3, 0x23, 0xE1, 0x21, 0x20, 0xE0, 0xA0, 0x60, 0x61, 0xA1,
    0x63, 0xA3, 0xA2, 0x62, 0x66, 0xA6, 0xA7, 0x67, 0xA5, 0x65, 0x64, 0xA4,
    0x6C, 0xAC, 0xAD, 0x6D, 0xAF, 0x6F, 0x6E, 0xAE, 0xAA, 0x6A, 0x6B, 0xAB,
    0x69, 0xA9, 0xA8, 0x68, 0x78, 0xB8, 0xB9, 0x79, 0xBB, 0x7B, 0x7A, 0xBA,
    0xBE, 0x7E, 0x7F, 0xBF, 0x7D, 0xBD, 0xBC, 0x7C, 0xB4, 0x74, 0x75, 0xB5,
    0x77, 0xB7, 0xB6, 0x76, 0x72, 0xB2, 0xB3, 0x73, 0xB1, 0x71, 0x70, 0xB0,
    0x50, 0x90, 0x91, 0x51, 0x93, 0x53, 0x52, 0x92, 0x96, 0x56, 0x57, 0x97,
    0x55, 0x95, 0x94, 0x54, 0x9C, 0x5C, 0x5D, 0x9D, 0x5F, 0x9F, 0x9E, 0x5E,
    0x5A, 0x9A, 0x9B, 0x5B, 0x99, 0x59, 0x58, 0x98, 0x88, 0x48, 0x49, 0x89,
    0x4B, 0x8B, 0x8A, 0x4A, 0x4E, 0x8E, 0x8F, 0x4F, 0x8D, 0x4D, 0x4C, 0x8C,
    0x44, 0x84, 0x85, 0x45, 0x87, 0x47, 0x46, 0x86, 0x82, 0x42, 0x43, 0x83,
    0x41, 0x81, 0x80, 0x40
};
```

### Lookup Table Algorithm

```c
/**
 * Calculate CRC-16 for MODBUS RTU (lookup table)
 * @param puchMsg Pointer to message buffer
 * @param usDataLen Number of bytes in message
 * @return CRC value (already in correct byte order for transmission)
 */
unsigned short CRC16(unsigned char *puchMsg, unsigned short usDataLen) {
    unsigned char uchCRCHi = 0xFF;  /* high byte of CRC initialized */
    unsigned char uchCRCLo = 0xFF;  /* low byte of CRC initialized */
    unsigned int uIndex;

    while (usDataLen--) {
        uIndex = uchCRCLo ^ *puchMsg++;
        uchCRCLo = uchCRCHi ^ auchCRCHi[uIndex];
        uchCRCHi = auchCRCLo[uIndex];
    }

    return (uchCRCHi << 8 | uchCRCLo);
}
```

## CRC-16 Examples

### Example 1: Read Holding Registers

**Frame data:** Read 10 registers from slave 1
```
Address:    0x01
Function:   0x03
Start Addr:  0x00 0x00
Quantity:   0x00 0x0A
```

**CRC Calculation:**
```
CRC16([01][03][00][00][00][0A]) = 0xC5CD
```

**Transmitted frame:**
```
01 03 00 00 00 0A CD C5
|                 |  |
|                 |  +-- CRC High Byte (0xC5)
|                 +----- CRC Low Byte (0xCD)
+---------------------- Frame data
```

**Important:** CRC is transmitted LSB first (CD C5), unlike all other MODBUS fields.

### Example 2: Write Single Register

**Frame data:** Write 0x0003 to register 1
```
Address:    0x01
Function:   0x06
Register:   0x00 0x01
Value:      0x00 0x03
```

**CRC Calculation:**
```
CRC16([01][06][00][01][00][03]) = 0x083C
```

**Transmitted frame:**
```
01 06 00 01 00 03 3C 08
|              |  |  |
|              |  |  +-- CRC High Byte (0x08)
|              |  +----- CRC Low Byte (0x3C)
|              +-------- Frame data
+------------------------- Address
```

## CRC-16 Validation

### Sender (Transmitter)

**Process:**
1. Build frame without CRC
2. Calculate CRC-16 over frame data
3. Append CRC to frame (LSB first)
4. Transmit complete frame

**Code example:**
```c
void send_modbus_rtu_frame(int fd, uint8_t *data, int len) {
    uint16_t crc = CRC16(data, len);

    // Append CRC (LSB first)
    data[len] = crc & 0xFF;       // CRC Low Byte
    data[len + 1] = (crc >> 8) & 0xFF;  // CRC High Byte

    // Transmit frame
    write(fd, data, len + 2);
}
```

### Receiver (Receiver)

**Process:**
1. Receive complete frame
2. Extract CRC from end of frame
3. Calculate CRC-16 over frame data (excluding CRC)
4. Compare calculated CRC with received CRC
5. Process frame if CRC matches, discard if CRC mismatch

**Code example:**
```c
int validate_modbus_rtu_frame(uint8_t *frame, int len) {
    if (len < 3) return 0;  // Too short for CRC

    uint16_t received_crc = (frame[len - 1] << 8) | frame[len - 2];
    uint16_t calculated_crc = CRC16(frame, len - 2);

    return (received_crc == calculated_crc);
}
```

## CRC-16 Properties

### Error Detection Capabilities

**Strengths:**
- Detects all single-bit errors
- Detects all double-bit errors
- Detects all odd-number of bit errors
- Detects all burst errors up to 16 bits
- Detects 99.998% of burst errors longer than 16 bits

**Limitations:**
- Cannot detect all errors (mathematically impossible)
- Small probability of undetected error
- Collision rate: 1 in 65536 for random errors

### Performance Comparison

| Algorithm | Speed | Memory | Typical Usage |
|-----------|-------|--------|---------------|
| Bit-by-bit | Slow | Small | Simple implementations, debugging |
| Lookup table | Fast | 512 bytes table | Production code, high performance |
| Byte-wise (half table) | Medium | 256 bytes table | Memory-constrained systems |

## Implementation Best Practices

### Byte Order Handling

**Critical:** Always transmit CRC bytes LSB first (little-endian):

```c
// Correct transmission order
uint16_t crc = calculate_crc(data, len);
transmit(crc & 0xFF);        // LSB first
transmit((crc >> 8) & 0xFF); // MSB second
```

### Frame Validation

**Always validate CRC before processing:**
```c
if (!validate_crc(frame, len)) {
    // Discard frame with invalid CRC
    return;
}
```

### Error Handling

**Handle CRC errors appropriately:**
- Discard frames with invalid CRC
- Do not respond to invalid frames (silent discard)
- Optionally log CRC errors for diagnostics

## Comparison with Other Error Checks

| Check | Size | Strength | Complexity |
|-------|------|----------|------------|
| CRC-16 | 16 bits | Very strong | Moderate |
| LRC | 8 bits | Weak | Simple |
| Checksum | 8/16 bits | Weak-Moderate | Simple |

## Related pages

- [MODBUS RTU](/wiki/concepts/modbus-rtu.md)
- [MODBUS ASCII](/wiki/concepts/modbus-ascii.md)
- [LRC](/wiki/concepts/lrc.md)

## Backlinks

- [MODBUS RTU vs ASCII Comparison](/wiki/answers/modbus-rtu-vs-ascii.md) - CRC-16 calculation and comparison with LRC
