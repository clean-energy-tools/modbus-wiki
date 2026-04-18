---
title: MODBUS ASCII
Summary: MODBUS ASCII transmission mode for serial line communication using ASCII hex encoding and LRC error checking.
Sources:
	- raw/MODBUS/MODBUS.md
	- raw/MODBUS/modbusoverserial.md
Last updated: 2026-04-18


MODBUS ASCII is a transmission mode for MODBUS over serial lines that transmits each byte as two ASCII hexadecimal characters, providing human-readable communication at the cost of efficiency (source: [modbusoverserial.md](raw/MODBUS/modbusoverserial.md)).

## Protocol Overview

ASCII mode transmits MODBUS frames in human-readable ASCII format, making debugging easier but requiring twice the transmission time compared to RTU mode (source: [modbusoverserial.md](raw/MODBUS/modbusoverserial.md)).

### Key Characteristics

| Property | Value |
|----------|-------|
| Mode | ASCII hex transmission |
| Error checking | LRC (Longitudinal Redundancy Check) |
| Frame delimiter | Start ':', End CR-LF |
| Character time | 10 bits (with parity), 9 bits (no parity) |
| Maximum frame size | 513 characters |
| Character format | 7 data bits (ASCII) |

## Character Format

ASCII mode uses a specific character format optimized for ASCII transmission (source: [modbusoverserial.md](raw/MODBUS/modbusoverserial.md)):

| Field | Bits | Description |
|-------|------|-------------|
| Start bit | 1 | Always 0 |
| Data bits | 7 | ASCII character, LSB first |
| Parity bit | 1 | Even parity (default) |
| Stop bit(s) | 1-2 | 1 with parity, 2 without |
| **Total** | **10** (with parity) or **9** (without parity) | Bits per character |

**Default configuration:** 7 data bits, even parity, 1 stop bit (7E1)

## Frame Structure

### ASCII ADU Structure

```
[':'][Address: 2 chars][Function Code: 2 chars][Data: 2N chars][LRC: 2 chars][CR][LF]
```

| Field | Value | Description |
|-------|-------|-------------|
| Start delimiter | ':' (0x3A) | Marks start of ASCII frame |
| Address | 2 ASCII hex chars | Slave address (00-FF) |
| Function Code | 2 ASCII hex chars | Function code (01-7F) |
| Data | 2N ASCII hex chars | Data bytes, each as 2 hex chars |
| LRC | 2 ASCII hex chars | LRC checksum |
| End delimiter | CR (0x0D), LF (0x0A) | Marks end of ASCII frame |

**Maximum ASCII ADU: 513 characters**

### Data Encoding

Each binary byte is transmitted as two ASCII hexadecimal characters:

| Binary | ASCII | Chars |
|--------|-------|-------|
| 0x00 | '00' | 2 |
| 0x0A | '0A' | 2 |
| 0xFF | 'FF' | 2 |
| 0x12 | '12' | 2 |

**ASCII hex digits:** '0'-'9' for 0-9, 'A'-'F' for 10-15

### Frame Delimiters

**Start delimiter:**
- Single character ':' (0x3A)
- Marks beginning of new frame
- Any partial frame is discarded on new start

**End delimiter:**
- Two characters: CR (0x0D) followed by LF (0x0A)
- Marks end of frame
- Frame complete only after LF received

## LRC Calculation

### LRC Algorithm

LRC (Longitudinal Redundancy Check) is the two's complement of the 8-bit sum of all bytes from address through last data byte.

**Calculation steps:**
1. Sum all bytes (address through last data byte)
2. Discard carries (keep only lower 8 bits)
3. Take two's complement (negate)
4. Result is 8-bit LRC

**Formula:**
```
LRC = (-(sum of bytes) AND 0xFF)
```

### LRC C Implementation

```c
/**
 * Calculate LRC for MODBUS ASCII
 * @param data Pointer to binary message buffer (not ASCII-encoded)
 * @param length Number of bytes in message
 * @return LRC value (8-bit)
 */
uint8_t calculate_lrc(uint8_t *data, int length) {
    uint8_t lrc = 0;

    for (int i = 0; i < length; i++) {
        lrc += data[i];
    }

    return (uint8_t)(-(int8_t)lrc);
}
```

### LRC Example

**Binary frame:** Read 10 registers from slave 1
```
Address:      0x01
Function:     0x03
Start Addr:    0x00 0x00
Quantity:     0x00 0x0A
```

**Sum:** 0x01 + 0x03 + 0x00 + 0x00 + 0x00 + 0x0A = 0x0E

**LRC:** -0x0E = 0xF2 (two's complement)

**ASCII frame:**
```
:01030000000AF2<CR><LF>
| |  |    | |
| |  |    +-- LRC "F2"
| |  +------- Data as "00000A"
| +---------- Function "03"
+------------ Address "01"
```

## Framing and Timing

### Inter-Character Timeout

- **Timeout:** Up to 1 second between characters (configurable)
- **Purpose:** Detect incomplete frames
- **Action:** Discard partial frame on timeout

### Frame Delimiters

- **Start:** Receiving ':' starts new frame (any partial frame discarded)
- **End:** CR-LF marks end of frame
- **Validation:** LRC must be valid after CR-LF received

### Frame Reception Rules

1. Wait for start delimiter ':'
2. Discard any partial frame on new start
3. Accumulate characters until CR-LF
4. Validate LRC
5. Convert ASCII hex to binary
6. Process frame

## Example: Read Holding Registers

**Binary Request:** Read 10 registers from slave 1, address 0
```
Address:   0x01
Function:  0x03
Start:     0x00 0x00
Quantity:  0x00 0x0A
```

**LRC Calculation:**
```
Sum: 0x01 + 0x03 + 0x00 + 0x00 + 0x00 + 0x0A = 0x0E
LRC: -0x0E = 0xF2
```

**ASCII Request:**
```
:01030000000AF2<CR><LF>
```

**ASCII Binary Response (3 registers with values 0x0000, 0x0064, 0x00C8):**
```
Binary: [01][03][06][00][00][00][64][00][C8]
LRC:    0x01 + 0x03 + 0x06 + 0x00 + 0x00 + 0x00 + 0x64 + 0x00 + 0xC8 = 0x16C
       = 0x6C (lower 8 bits)
       LRC = -0x6C = 0x94

ASCII: :0103060000006400C894<CR><LF>
```

## Comparison with RTU Mode

| Characteristic | RTU | ASCII |
|---------------|-----|-------|
| Efficiency | High (binary) | Low (ASCII hex, 2× bandwidth) |
| Error checking | CRC-16 (strong) | LRC (weak) |
| Timing | Strict (t1.5, t3.5) | Relaxed (1s timeout) |
| Character time | 11 bits | 10 bits |
| Max frame size | 256 bytes | 513 characters |
| Debugging | Binary analyzer needed | Human-readable |
| Error detection | Very strong | Weak (simple sum) |
| Implementation complexity | Moderate | Simple |
| Bandwidth usage | Efficient | Inefficient (2×) |

## Advantages of ASCII Mode

**Human-Readable:**
- Frames can be read and typed manually
- Easier debugging with terminal programs
- Useful for testing and diagnostics

**Relaxed Timing:**
- No strict inter-character timing requirements
- More tolerant of baud rate variations
- Works with longer cable runs

**Simple Implementation:**
- Easy to implement on systems with ASCII terminals
- No complex timing state machine needed

## Disadvantages of ASCII Mode

**Low Efficiency:**
- Requires 2× bandwidth compared to RTU
- Transmission time doubled for same data
- Not suitable for high-speed applications

**Weak Error Checking:**
- LRC only provides 8-bit checksum
- Higher probability of undetected errors
- CRC-16 provides much better protection

**Larger Overhead:**
- Start and end delimiters add overhead
- ASCII encoding doubles data size

## Typical Applications

ASCII mode is typically used for:
- Testing and debugging with terminal programs
- Manual data entry and verification
- Systems with simple ASCII-only terminals
- Long cable runs where timing is critical
- Prototyping and development
- Educational purposes
- Diagnostic tools

## Implementation Considerations

### ASCII to Binary Conversion

```c
// Convert 2 ASCII hex chars to byte
uint8_t ascii_to_byte(char high, char low) {
    uint8_t result = 0;

    // High nibble
    if (high >= '0' && high <= '9') {
        result = (high - '0') << 4;
    } else if (high >= 'A' && high <= 'F') {
        result = (high - 'A' + 10) << 4;
    }

    // Low nibble
    if (low >= '0' && low <= '9') {
        result |= (low - '0');
    } else if (low >= 'A' && low <= 'F') {
        result |= (low - 'A' + 10);
    }

    return result;
}
```

### Binary to ASCII Conversion

```c
// Convert byte to 2 ASCII hex chars
void byte_to_ascii(uint8_t byte, char *output) {
    static const char hex_chars[] = "0123456789ABCDEF";

    output[0] = hex_chars[(byte >> 4) & 0x0F];
    output[1] = hex_chars[byte & 0x0F];
}
```

## Related pages

- [[wiki/concepts/modbus-rtu]]
- [[wiki/concepts/modbus-tcp]]
- [[wiki/concepts/lrc]]
- [[wiki/concepts/crc-16]]
- [[wiki/concepts/function-codes]]
