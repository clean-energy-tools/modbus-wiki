---
title: LRC (Longitudinal Redundancy Check)
Summary: Error detection algorithm used in MODBUS ASCII mode for frame integrity verification through simple checksum calculation.
Sources:
  - raw/MODBUS/modbusoverserial.md
  - raw/MODBUS/MODBUS.md
Categories:
  - error-detection
  - modbus-ascii
  - protocol-details
type: concept
date-created: 2026-04-25T15:00:00+03:00
last-updated: 2026-04-25T15:00:00+03:00
---

LRC (Longitudinal Redundancy Check) is a simple error detection algorithm used in [MODBUS ASCII](/wiki/concepts/modbus-ascii.md) mode to verify frame integrity. It provides basic error detection through an 8-bit checksum calculation (source: [modbusoverserial.md](/raw/MODBUS/modbusoverserial.md)).

## Overview

LRC is an error-checking field that verifies the contents of MODBUS ASCII messages, exclusive of the beginning colon `:` and ending CR-LF pair (source: [modbusoverserial.md](/raw/MODBUS/modbusoverserial.md)). While less robust than [CRC-16](/wiki/concepts/crc-16.md) used in MODBUS RTU, LRC provides adequate error detection for the slower MODBUS ASCII transmission mode.

## LRC Field Characteristics

| Property | Value |
|----------|-------|
| Size | 1 byte (8-bit binary value) |
| Transmission | 2 ASCII characters (hex representation) |
| Position | Before CR-LF terminator |
| Calculation | Sum of bytes + two's complement |
| Coverage | Address through last data byte |

## Calculation Algorithm

The LRC is calculated by adding together successive 8-bit bytes in the message, discarding any carries, and then two's complementing the result (source: [modbusoverserial.md](/raw/MODBUS/modbusoverserial.md)).

### LRC Generation Steps

1. **Add all bytes**: Sum all message bytes (excluding colon and CR-LF) into an 8-bit field
2. **Discard carries**: Any value over 255 automatically rolls over (no 9th bit)
3. **One's complement**: Subtract final value from 0xFF
4. **Two's complement**: Add 1 to produce final LRC value

### Mathematical Formula

```
LRC = (-(sum of all bytes)) & 0xFF
```

Or equivalently:
```
LRC = ((0xFF - (sum of all bytes)) + 1) & 0xFF
```

## Implementation

### C Implementation

```c
/**
 * Calculate LRC for MODBUS ASCII
 * @param auchMsg Pointer to message buffer (binary data)
 * @param usDataLen Number of bytes in message
 * @return LRC value (8-bit)
 */
static unsigned char LRC(unsigned char *auchMsg, unsigned short usDataLen)
{
    unsigned char uchLRC = 0;
    
    while (usDataLen--)
        uchLRC += *auchMsg++;
    
    return ((unsigned char)(-((char)uchLRC)));
}
```

(source: [modbusoverserial.md](/raw/MODBUS/modbusoverserial.md), [MODBUS.md](/raw/MODBUS/MODBUS.md))

### Python Implementation

```python
def calculate_lrc(data: bytes) -> int:
    """
    Calculate LRC for MODBUS ASCII frame
    
    Args:
        data: Message bytes (excluding colon and CRLF)
    
    Returns:
        LRC value (0-255)
    """
    lrc = sum(data) & 0xFF
    return (-lrc) & 0xFF
```

### Rust Implementation

```rust
/// Calculate LRC for MODBUS ASCII frame
fn calculate_lrc(data: &[u8]) -> u8 {
    let sum: u8 = data.iter().fold(0u8, |acc, &b| acc.wrapping_add(b));
    (!sum).wrapping_add(1)
}
```

## Calculation Example

**Message (binary):** Address=0x01, Function=0x03, Data=0x00, 0x00, 0x00, 0x02

**Step 1: Sum all bytes**
```
0x01 + 0x03 + 0x00 + 0x00 + 0x00 + 0x02 = 0x06
```

**Step 2: Two's complement**
```
LRC = -0x06 = 0xFA
```

**Transmitted as ASCII:** "FA" (0x46 0x41)

## LRC in MODBUS ASCII Frame

The LRC field is transmitted as two ASCII hex characters before the CR-LF terminator:

```
[':'][Address: 2 chars][Function: 2 chars][Data: 2N chars][LRC: 2 chars][CR][LF]
```

**Example frame:**
```
:01 03 00 00 00 02 FA CR LF
 |  |  |        |  +-- LRC value (2 ASCII chars)
 |  |  +----------- Data bytes
 |  +-------------- Function code
 +----------------- Address
```

## LRC Transmission Order

When the 8-bit LRC is transmitted, it is encoded as two ASCII characters with the high-order nibble transmitted first (source: [modbusoverserial.md](/raw/MODBUS/modbusoverserial.md)).

**Example:** LRC value = 0x61
- High nibble: 6 → ASCII '6' (0x36)
- Low nibble: 1 → ASCII '1' (0x31)
- Transmitted: "61"

## Error Detection Process

### Transmitter Side

1. Calculate LRC over address, function code, and data bytes
2. Append LRC to message as two ASCII hex characters
3. Append CR-LF terminator
4. Transmit complete frame

### Receiver Side

1. Receive frame and extract all bytes between ':' and CR-LF
2. Convert ASCII hex pairs to binary bytes
3. Calculate LRC over received address, function, and data bytes
4. Compare calculated LRC with received LRC field
5. If values match: accept frame; if different: reject frame

## Error Detection Capability

LRC provides basic error detection but is **weaker than CRC-16** used in MODBUS RTU:

| Aspect | LRC (ASCII) | CRC-16 (RTU) |
|--------|-------------|--------------|
| Algorithm | Simple checksum | Polynomial division |
| Size | 8 bits | 16 bits |
| Burst error detection | Limited | Excellent |
| Single bit error detection | Yes | Yes |
| Multi-bit error detection | Partial | Strong |
| Collision probability | ~1/256 | ~1/65536 |

**Why LRC is acceptable for MODBUS ASCII:**
- ASCII mode is slower (half the throughput of RTU)
- ASCII encoding itself provides some error detection (invalid hex characters)
- Typically used on cleaner communication channels
- Trade-off: simplicity vs. robustness

## Common Implementation Issues

### Issue 1: Including Colon or CR-LF in Calculation

**Incorrect:**
```c
// DON'T include ':' or CR-LF in LRC calculation
uchLRC += ':';  // WRONG
```

**Correct:**
```c
// Calculate LRC only over address, function, and data
for (int i = 0; i < data_len; i++)
    uchLRC += message[i];
```

### Issue 2: Incorrect Two's Complement

**Incorrect:**
```c
return 0xFF - uchLRC;  // This is one's complement only
```

**Correct:**
```c
return ((unsigned char)(-((char)uchLRC)));  // Two's complement
```

Or:
```c
return (0xFF - uchLRC) + 1;  // Two's complement formula
```

### Issue 3: ASCII Encoding Errors

**Remember:** The LRC value is binary, but transmitted as two ASCII hex characters:
- Binary LRC: 0xFA
- Transmitted: "FA" → 0x46 ('F') 0x41 ('A')

## Comparison with CRC-16

LRC is simpler but less robust than [CRC-16](/wiki/concepts/crc-16.md):

| Feature | LRC | CRC-16 |
|---------|-----|--------|
| Used in | MODBUS ASCII | MODBUS RTU |
| Calculation | Addition + two's complement | Polynomial division |
| Complexity | Very simple | More complex |
| Overhead | 1 byte | 2 bytes |
| Error detection | Basic | Strong |
| Implementation | ~5 lines of code | ~20-30 lines or lookup table |
| Performance | Very fast | Fast (with lookup table) |

**When to use LRC:**
- MODBUS ASCII mode (required by specification)
- Simple implementations with limited resources
- Clean communication channels

**When to use CRC-16:**
- MODBUS RTU mode (required by specification)
- Higher data rates
- Noisy industrial environments

## Best Practices

1. **Always validate LRC** on received frames before processing
2. **Use correct two's complement** calculation (not just one's complement)
3. **Exclude framing characters** (colon, CR, LF) from calculation
4. **Test with known values** during implementation
5. **Handle ASCII encoding correctly** when transmitting/receiving
6. **Consider using CRC-16** if your application allows MODBUS RTU instead of ASCII

## Testing LRC Implementation

### Test Vector 1: Simple Frame

**Input (binary):** `01 03 00 00 00 02`
**Expected LRC:** `0xFA`

### Test Vector 2: All Zeros

**Input (binary):** `00 00 00 00`
**Expected LRC:** `0x00`

### Test Vector 3: Maximum Values

**Input (binary):** `FF FF FF FF`
**Expected LRC:** `0x04`

## Related pages

- [MODBUS ASCII](/wiki/concepts/modbus-ascii.md) - Uses LRC for error detection
- [CRC-16](/wiki/concepts/crc-16.md) - More robust error detection used in MODBUS RTU
- [MODBUS RTU](/wiki/concepts/modbus-rtu.md) - Uses CRC-16 instead of LRC

## Backlinks

- [MODBUS ASCII](/wiki/concepts/modbus-ascii.md)
- [CRC-16](/wiki/concepts/crc-16.md)
- [Summary of MODBUS over serial line specification and implementation guide](/wiki/summaries/MODBUS/modbusoverserial.md)
