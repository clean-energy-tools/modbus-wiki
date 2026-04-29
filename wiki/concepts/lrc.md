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

LRC (Longitudinal Redundancy Check) is a simple way to catch errors in [MODBUS ASCII](/wiki/concepts/modbus-ascii.md) messages (source: [modbusoverserial.md](/raw/MODBUS/modbusoverserial.md)). It adds up all the numbers in the message and includes a checksum so the receiver can verify nothing got corrupted.

## What LRC Does

LRC checks everything in a MODBUS ASCII message except the starting colon ':' and the ending Enter (CR-LF) (source: [modbusoverserial.md](/raw/MODBUS/modbusoverserial.md)). It's simpler and less powerful than [CRC-16](/wiki/concepts/crc-16.md) used in MODBUS RTU, but it's good enough for the slower ASCII mode.

## How LRC Works

| Feature | Details |
|----------|-------|
| Size | 1 byte (8 bits) |
| Sent as | 2 text characters (like "A5") |
| Where it goes | Just before the Enter at the end |
| How it's made | Add all bytes, then negate the result |
| What it covers | Everything from device address to last data byte |

## How to Calculate LRC

LRC is calculated by adding all the bytes together and then negating the result (source: [modbusoverserial.md](/raw/MODBUS/modbusoverserial.md)).

### Steps to Make an LRC

1. **Add everything**: Add all message bytes together (skip the starting ':' and ending Enter)
2. **Keep only 8 bits**: If the sum goes over 255, it wraps around automatically
3. **Flip the bits**: Subtract the sum from 0xFF (255)
4. **Add 1**: Add 1 to get the final LRC value

### The Math

In simple terms:
```
LRC = (negative of sum) with only bottom 8 bits
```

More precisely:
```
LRC = ((255 - sum) + 1) keeping only bottom 8 bits
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

- [CRC-16](/wiki/concepts/crc-16.md)
- [MODBUS ASCII](/wiki/concepts/modbus-ascii.md)
- [MODBUS RTU](/wiki/concepts/modbus-rtu.md)
- [Summary of MODBUS over serial line specification and implementation guide](/wiki/summaries/MODBUS/modbusoverserial.md)

------------

MODBUS is a trademark of the Modbus Organization, Inc.

All information on this page is derived by summarizing and analyzing solely the MODBUS specifications published by the Modbus Organization, Inc.  The specifications are copyright by the Modbus Organization.

By deriving this information solely from the specifications we hope to stay true to the specification.

No infringement is intended.
