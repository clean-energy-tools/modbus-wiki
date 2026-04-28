---
title: MODBUS RTU vs ASCII Comparison
Summary: Comprehensive comparison of MODBUS RTU and ASCII transmission modes including protocol frames, encoding differences, error checking methods, CRC calculation, and when to use each mode.
Sources:
  - raw/MODBUS/modbusoverserial.md
  - raw/MODBUS/MODBUS.md
Categories:
  - serial-communication
  - protocol-comparison
  - rtu-mode
  - ascii-mode
type: answer
date-created: 2026-04-24T13:30:00+03:00
last-updated: 2026-04-24T13:30:00+03:00
---

MODBUS RTU and MODBUS ASCII are two different ways to send MODBUS messages over serial cables. Both use the same MODBUS commands and data, but they package and send that information very differently. Think of it like sending the same letter by regular mail versus certified mail - the content is the same, but the envelope and handling process are different.

## Quick Comparison

| Feature | MODBUS RTU | MODBUS ASCII |
|--------|------------|--------------|
| **How data is sent** | Binary numbers | Text characters (hex digits) |
| **Error detection** | CRC-16 (very strong, 16-bit) | LRC (weaker, 8-bit) |
| **Message size** | Compact | Double the size |
| **Can humans read it?** | No | Yes |
| **How messages are separated** | Brief silence on the line | Special start/end characters |
| **Bits per character** | 8 data bits | 7 data bits |
| **Serial port settings** | Usually 8E1 or 8N2 | Usually 7E1 |
| **Maximum message size** | 256 bytes | 513 characters |
| **Where it's used** | Production systems (95%+) | Testing and debugging |
| **Speed** | Fast | About half the speed |

## Frame Structure Differences

### MODBUS RTU Frame

**Binary transmission** - each byte sent as-is (source: [MODBUS RTU](/wiki/concepts/modbus-rtu.md)):

```
[Address: 1][Function: 1][Data: 0-252][CRC-16: 2]
|<------------- binary bytes ------------->|
```

**Example: Read 10 holding registers from slave 1**
```
Hex:  01 03 00 00 00 0A CD C5
Bytes: [Address][Func][Start H][Start L][Qty H][Qty L][CRC L][CRC H]
Size: 8 bytes
```

### MODBUS ASCII Frame

**ASCII hex encoding** - each byte becomes two ASCII characters (source: [MODBUS ASCII](/wiki/concepts/modbus-ascii.md)):

```
[':'][Addr: 2][Func: 2][Data: 2N][LRC: 2][CR][LF]
|<------- ASCII hex characters -------->|
```

**Same example: Read 10 holding registers from slave 1**
```
ASCII: :01030000000AF2<CR><LF>
Chars:  : 0 1 0 3 0 0 0 0 0 0 0 A F 2 \r \n
Size: 17 characters
```

**Encoding comparison:**
- RTU: 8 bytes (64 bits)
- ASCII: 17 characters (170 bits with 10-bit char format)
- **ASCII is 2.1x larger than RTU**

## Do They Use the Same Commands?

**Answer:** Yes and No.

### Same MODBUS Commands

Both RTU and ASCII use the **same MODBUS commands and data** (source: [modbusoverserial.md](/raw/MODBUS/modbusoverserial.md)):
- Same function codes (numbered 0x01-0x7F)
- Same types of data (coils, discrete inputs, holding registers, input registers)
- Same command structure
- Same error codes and error responses
- Same addressing rules

**Example command (identical for both):**
```
Function: 0x03 (Read Holding Registers)
Starting Address: 0x0000
Quantity: 0x000A (10 registers)

Command bytes: 03 00 00 00 0A
```

### Different Ways of Sending

The command gets packaged and sent differently in RTU vs ASCII:

**RTU transmission (binary):**
```
Binary bytes on wire:
01 03 00 00 00 0A CD C5
^^ Address
   ^^ PDU starts here
                  ^^^^ CRC-16
```

**ASCII transmission (hex characters):**
```
ASCII characters on wire:
:01030000000AF2<CR><LF>
 ^^Address (as '0' '1')
   ^^PDU starts here (as '0' '3')
             ^^LRC (as 'F' '2')
```

### Frame Format Comparison

| Component | RTU | ASCII |
|-----------|-----|-------|
| **Start delimiter** | Silent interval (t3.5) | ':' character (0x3A) |
| **Address** | 1 byte binary | 2 ASCII hex chars |
| **Function code** | 1 byte binary | 2 ASCII hex chars |
| **Data** | N bytes binary | 2N ASCII hex chars |
| **Error check** | CRC-16 (2 bytes) | LRC (2 ASCII hex chars) |
| **End delimiter** | Silent interval (t3.5) | CR + LF (0x0D 0x0A) |

## How Data Is Encoded

### RTU: Sending Binary Numbers

**Sends numbers directly:**
- Each byte value (0x00-0xFF) is sent as raw binary data
- No extra conversion needed
- **Example:** The number 5 (0x05) → sent as a single byte 0x05

**Character format (8E1):**
```
| Start | D0 D1 D2 D3 D4 D5 D6 D7 | Parity | Stop |
|   0   |    8 data bits         |   E    |  1   |
|       |<-- LSB first ---------->|        |      |
```

**Total:** 11 bits per byte (with parity)

### ASCII: Converting to Text

**Converts each byte to two text characters:**
- Each byte value becomes two readable hex characters ('0'-'9', 'A'-'F')
- This doubles the size
- **Example:** The number 5 (0x05) → sent as two characters: '0' and '5' (which are themselves bytes 0x30 and 0x35)

**Character format (7E1):**
```
| Start | D0 D1 D2 D3 D4 D5 D6 | Parity | Stop |
|   0   |   7 data bits       |   E    |  1   |
|       |<-- LSB first ------>|        |      |
```

**Total:** 10 bits per text character

**Encoding table:**

| Binary | RTU Transmission | ASCII Transmission |
|--------|------------------|-------------------|
| 0x00 | 0x00 (1 byte) | '0' '0' (0x30 0x30 = 2 bytes) |
| 0x0A | 0x0A (1 byte) | '0' 'A' (0x30 0x41 = 2 bytes) |
| 0xFF | 0xFF (1 byte) | 'F' 'F' (0x46 0x46 = 2 bytes) |
| 0x5A | 0x5A (1 byte) | '5' 'A' (0x35 0x41 = 2 bytes) |

## How Messages Are Separated

### RTU: Using Silence

**Detects message boundaries by timing** (source: [MODBUS RTU](/wiki/concepts/modbus-rtu.md)):

```
<-silence->|<----- Message ----->|<-silence->|<----- Message ----->|
  (quiet)  | Addr Func Data CRC  |  (quiet)  | Addr Func Data CRC  |
           ^                     ^
       Message starts       Message ends
```

**Key timing rules:**
- **t1.5:** Maximum gap between bytes within a message (1.5 character transmission times)
- **t3.5:** Silence between messages (3.5 character transmission times)

**Example timing at 9600 baud (11 bits per character):**
- One character takes: 1.146 milliseconds
- t1.5 = 1.72 ms (bytes must arrive faster than this)
- t3.5 = 4.01 ms (silence must be at least this long)

**How it works:**
1. A message starts after the line is silent for t3.5
2. All bytes in the message must arrive within t1.5 of each other
3. If there's a gap longer than t1.5 → error, discard the message
4. The message ends when the line goes silent for t3.5

**Advantage:** No special characters needed for start/end markers
**Disadvantage:** Requires precise timing, which can be challenging to implement

### ASCII: Using Special Characters

**Detects message boundaries using markers** (source: [MODBUS ASCII](/wiki/concepts/modbus-ascii.md)):

```
:<Addr><Func><Data><LRC><CR><LF>:<Addr><Func><Data><LRC><CR><LF>
^                          ^  ^                              ^
Start                   End     Start                     End
```

**Special characters:**
- **Start marker:** ':' (colon character, value 0x3A)
- **End markers:** CR-LF (carriage return 0x0D + line feed 0x0A, like pressing Enter)

**How it works:**
1. A message starts when you receive a ':' character
2. Any incomplete previous message is discarded
3. The message ends when you receive CR-LF (Enter key sequence)
4. No timing requirements - characters can arrive with any delay between them

**Advantage:** No precise timing needed, can pause between characters
**Disadvantage:** Can't use ':' or CR-LF as data (but the encoding method prevents this anyway)

## How Errors Are Detected

### RTU: CRC-16 Checksum

**Strength:** Very strong error detection (source: [CRC-16](/wiki/concepts/crc-16.md))

**Parameters:**
```
Polynomial:    0x8005 (reversed: 0xA001)
Initial value: 0xFFFF
Check size:    16 bits (2 bytes)
```

**Algorithm:**
```rust
fn crc16_modbus(data: &[u8]) -> u16 {
    let mut crc: u16 = 0xFFFF;
    
    for &byte in data {
        crc ^= byte as u16;
        for _ in 0..8 {
            if crc & 0x0001 != 0 {
                crc = (crc >> 1) ^ 0xA001;
            } else {
                crc >>= 1;
            }
        }
    }
    
    crc
}
```

**Detection capabilities:**
- ✅ All single-bit errors
- ✅ All double-bit errors
- ✅ All odd numbers of bit errors
- ✅ All burst errors ≤ 16 bits
- ✅ 99.998% of burst errors > 16 bits

**Example calculation:**
```
Data: [01][03][00][00][00][0A]
CRC = crc16_modbus(data) = 0xC5CD

Transmitted: 01 03 00 00 00 0A CD C5
                              ^^ ^^
                              LSB MSB (transmitted LSB first!)
```

**Important detail:** CRC bytes are sent **LSB first** (least significant byte first), unlike all other MODBUS numbers which send the most significant byte first.

### ASCII: LRC Checksum

**Strength:** Weaker error detection (source: [MODBUS ASCII](/wiki/concepts/modbus-ascii.md))

**How it's calculated:** Two's complement of the sum of all bytes
```
LRC = (-sum_of_bytes) & 0xFF
```

**Implementation:**
```rust
fn calculate_lrc(data: &[u8]) -> u8 {
    let sum: u8 = data.iter().fold(0u8, |acc, &b| acc.wrapping_add(b));
    (!sum).wrapping_add(1)  // Two's complement
}
```

**Detection capabilities:**
- ✅ Some single-bit errors
- ❌ Many multi-bit errors go undetected
- ❌ Weak against burst errors
- ❌ Collisions common (1 in 256)

**Example calculation:**
```
Data: [01][03][00][00][00][0A]
Sum = 0x01 + 0x03 + 0x00 + 0x00 + 0x00 + 0x0A = 0x0E
LRC = -0x0E = 0xF2

Transmitted: :01030000000AF2<CR><LF>
                         ^^
                         LRC as 'F' '2'
```

### Error Detection Comparison

| Feature | CRC-16 (RTU) | LRC (ASCII) |
|---------|--------------|-------------|
| Check size | 16 bits | 8 bits |
| Strength | Very strong | Weak |
| Single-bit detection | 100% | ~99% |
| Multi-bit detection | Very high | Low |
| Burst error detection | Excellent | Poor |
| Collision rate | 1 in 65,536 | 1 in 256 |
| Calculation complexity | Moderate | Simple |
| Typical use | Production | Debugging only |

**Bottom line:** CRC-16 is approximately **256 times better** than LRC at detecting errors.

## CRC-16 Calculation Details

### Bit-by-Bit Algorithm

**Step-by-step process:**

```rust
fn crc16_modbus_explained(data: &[u8]) -> u16 {
    let mut crc: u16 = 0xFFFF;  // Start with all 1s
    
    for &byte in data {
        // XOR byte into lower 8 bits of CRC
        crc ^= byte as u16;
        
        // Process each bit (LSB first)
        for _ in 0..8 {
            // Check if LSB is 1
            if crc & 0x0001 != 0 {
                // Shift right and XOR with polynomial
                crc = (crc >> 1) ^ 0xA001;
            } else {
                // Just shift right
                crc >>= 1;
            }
        }
    }
    
    crc
}
```

**Example walkthrough:**

Frame: `01 03 00 00 00 0A`

```
Initial: CRC = 0xFFFF

Byte 1: 0x01
  CRC = 0xFFFF ^ 0x01 = 0xFFFE
  Bit 0: 0xFFFE & 1 = 0 → CRC = 0x7FFF
  Bit 1: 0x7FFF & 1 = 1 → CRC = (0x3FFF) ^ 0xA001 = 0x9FFE
  ... (6 more bits)
  Result: CRC = 0xBF7F

Byte 2: 0x03
  CRC = 0xBF7F ^ 0x03 = 0xBF7C
  ... (process 8 bits)
  Result: CRC = 0x409E

... (continue for remaining bytes)

Final: CRC = 0xC5CD
```

### Lookup Table Algorithm (Optimized)

**Much faster implementation** using pre-computed tables (source: [CRC-16](/wiki/concepts/crc-16.md)):

```c
// Pre-computed lookup tables (256 bytes each)
static const uint8_t crc_hi_table[256] = { /* ... */ };
static const uint8_t crc_lo_table[256] = { /* ... */ };

uint16_t crc16_modbus_fast(const uint8_t *data, size_t len) {
    uint8_t crc_hi = 0xFF;
    uint8_t crc_lo = 0xFF;
    
    while (len--) {
        uint8_t index = crc_lo ^ (*data++);
        crc_lo = crc_hi ^ crc_hi_table[index];
        crc_hi = crc_lo_table[index];
    }
    
    return (crc_hi << 8) | crc_lo;
}
```

**Performance:** Lookup table is typically 8-10x faster than bit-by-bit.

### CRC Transmission Order

**Critical difference from other MODBUS fields:**

All MODBUS fields are **big-endian** (MSB first) EXCEPT CRC-16.

**CRC is transmitted LSB first:**

```rust
// Calculate CRC
let crc = crc16_modbus(&frame_data);  // Returns 0xC5CD

// Transmit frame
transmit_bytes(&frame_data);
transmit_byte((crc & 0xFF) as u8);        // Send 0xCD (LSB)
transmit_byte(((crc >> 8) & 0xFF) as u8); // Send 0xC5 (MSB)

// On wire:
// ... [data] CD C5
//            ^^ ^^ 
//           LSB MSB
```

**Why LSB first?** Historical compatibility with early MODBUS implementations.

### CRC Validation

**Receiver side:**

```rust
fn validate_rtu_frame(frame: &[u8]) -> bool {
    if frame.len() < 3 {
        return false;  // Too short
    }
    
    let data_len = frame.len() - 2;
    let received_crc_lo = frame[data_len];
    let received_crc_hi = frame[data_len + 1];
    let received_crc = (received_crc_hi as u16) << 8 | received_crc_lo as u16;
    
    let calculated_crc = crc16_modbus(&frame[..data_len]);
    
    received_crc == calculated_crc
}
```

## When to Use Each Mode

### Use MODBUS RTU For:

**✅ Production systems** (the vast majority of real-world use)
- You need maximum speed and efficiency
- Data transmission speed matters
- Strong error detection is important
- This is what the industry expects

**✅ Applications with lots of data**
- Frequent data requests
- Large amounts of data
- Multiple devices on the same cable
- Fast communication speeds (38400 baud and higher)

**✅ Embedded controllers**
- Limited computing power (lookup tables help)
- Standard software libraries available
- Needs to work with other standard devices

**Real-world examples:**
- Factory automation controllers (PLCs)
- Building automation systems
- Electricity meters
- Motor controllers
- Industrial sensors
- SCADA monitoring systems

**Market share:** About 95% or more of MODBUS serial installations

### Use MODBUS ASCII For:

**✅ Testing and learning**
- Humans can read the messages
- Easy to type test messages manually
- Can see what's happening without tools
- Good for learning how MODBUS works

**✅ Old equipment**
- Existing devices that only understand ASCII
- Legacy software systems
- Historical equipment that can't be changed

**✅ Noisy electrical environments (questionable benefit)**
- Some people believe ASCII works better with electrical noise (this is debatable)
- The character markers help recover from errors
- But CRC-16 (used by RTU) is much better at detecting errors anyway

**✅ Simple terminal testing**
- Can send commands using any terminal program
- Don't need special software tools
- Good for quick manual tests

**Real-world examples:**
- Learning and development
- Educational settings
- Simple testing tools
- Connecting to very old equipment
- Low-priority monitoring where accuracy is less critical

**Market share:** Less than 5% of MODBUS serial installations

### Quick Comparison

| Feature | RTU | ASCII |
|-----------|-----|-------|
| **Speed** | ⭐⭐⭐⭐⭐ Faster | ⭐⭐ Slower |
| **Data efficiency** | ⭐⭐⭐⭐⭐ Compact | ⭐⭐ Double the size |
| **Error detection** | ⭐⭐⭐⭐⭐ Very strong | ⭐⭐ Weaker |
| **Can humans read it?** | ⭐ No | ⭐⭐⭐⭐⭐ Yes |
| **Easy to test?** | ⭐⭐ Need tools | ⭐⭐⭐⭐⭐ Use any terminal |
| **Difficulty to implement** | ⭐⭐⭐ Moderate | ⭐⭐⭐⭐ Simpler |
| **Industry use** | ⭐⭐⭐⭐⭐ Standard | ⭐ Rare |
| **Timing requirements** | ⭐⭐ Precise timing needed | ⭐⭐⭐⭐⭐ No timing needed |

## Detailed Example: Same Command in Both Modes

### Command: Read 10 Holding Registers from Slave 1

**MODBUS PDU (same for both):**
```
Function: 0x03 (Read Holding Registers)
Start Address: 0x0000
Quantity: 0x000A (10 registers)
```

### RTU Encoding

**Binary frame:**
```
Byte 0:    0x01    Address (slave 1)
Byte 1:    0x03    Function (Read Holding Registers)
Byte 2:    0x00    Starting Address High
Byte 3:    0x00    Starting Address Low
Byte 4:    0x00    Quantity High
Byte 5:    0x0A    Quantity Low
Byte 6:    0xCD    CRC Low Byte
Byte 7:    0xC5    CRC High Byte
```

**Wire transmission (hex):**
```
01 03 00 00 00 0A CD C5
```

**Size:** 8 bytes = 88 bits (with 11-bit char format)

**Timing at 9600 baud:**
- Transmission time: 88 bits / 9600 bps = 9.17 ms
- Plus inter-frame delays: ~13 ms total

### ASCII Encoding

**ASCII hex frame:**
```
Start:     ':'     0x3A
Addr:      '0'     0x30
           '1'     0x31
Function:  '0'     0x30
           '3'     0x33
Start H:   '0'     0x30
           '0'     0x30
Start L:   '0'     0x30
           '0'     0x30
Qty H:     '0'     0x30
           '0'     0x30
Qty L:     '0'     0x30
           'A'     0x41
LRC:       'F'     0x46
           '2'     0x32
End:       CR      0x0D
           LF      0x0A
```

**Wire transmission (characters):**
```
:01030000000AF2<CR><LF>
```

**Size:** 17 characters = 170 bits (with 10-bit char format)

**Timing at 9600 baud:**
- Transmission time: 170 bits / 9600 bps = 17.7 ms
- **Nearly 2x slower than RTU**

### Side-by-Side Wire Comparison

**RTU (binary hex view):**
```
01 03 00 00 00 0A CD C5
```

**ASCII (character view):**
```
: 0 1 0 3 0 0 0 0 0 0 0 A F 2 CR LF
```

**ASCII (binary hex view):**
```
3A 30 31 30 33 30 30 30 30 30 30 30 41 46 32 0D 0A
```

## Implementation Considerations

### RTU Implementation Challenges

**1. Timing precision required:**
```rust
// Must detect silent intervals accurately
const CHAR_TIME_US: u32 = 1146;  // At 9600 baud, 11 bits
const T1_5_US: u32 = CHAR_TIME_US * 3 / 2;  // 1719 us
const T3_5_US: u32 = CHAR_TIME_US * 7 / 2;  // 4011 us

// Frame reception with timing
loop {
    if let Some(byte) = uart.read_byte_timeout(T1_5_US) {
        frame_buffer.push(byte);
    } else {
        if frame_buffer.len() > 0 {
            // T1.5 timeout - frame error or end
            if time_since_last_byte() >= T3_5_US {
                // Frame complete
                process_frame(&frame_buffer);
            } else {
                // Frame error (gap too large within frame)
                discard_frame();
            }
        }
    }
}
```

**2. CRC calculation and byte order:**
```rust
// Calculate CRC
let crc = crc16_modbus(&frame_data);

// CRITICAL: Transmit LSB first
frame.push((crc & 0xFF) as u8);        // Low byte
frame.push(((crc >> 8) & 0xFF) as u8); // High byte
```

**3. High baud rate handling:**
```rust
// For baud > 19200, use fixed timing
const T1_5_FIXED_US: u32 = 750;   // 750 microseconds
const T3_5_FIXED_US: u32 = 1750;  // 1750 microseconds
```

### ASCII Implementation Challenges

**1. Character encoding/decoding:**
```rust
fn hex_to_nibble(c: u8) -> Option<u8> {
    match c {
        b'0'..=b'9' => Some(c - b'0'),
        b'A'..=b'F' => Some(c - b'A' + 10),
        b'a'..=b'f' => Some(c - b'a' + 10),
        _ => None,
    }
}

fn decode_ascii_byte(high: u8, low: u8) -> Option<u8> {
    let h = hex_to_nibble(high)?;
    let l = hex_to_nibble(low)?;
    Some((h << 4) | l)
}

fn encode_byte_to_ascii(byte: u8) -> (u8, u8) {
    const HEX: &[u8] = b"0123456789ABCDEF";
    let high = HEX[(byte >> 4) as usize];
    let low = HEX[(byte & 0x0F) as usize];
    (high, low)
}
```

**2. Frame parsing:**
```rust
fn parse_ascii_frame(input: &[u8]) -> Option<Vec<u8>> {
    // Find start ':'
    let start = input.iter().position(|&c| c == b':')?;
    
    // Find end CR-LF
    let mut end = start + 1;
    while end < input.len() - 1 {
        if input[end] == 0x0D && input[end + 1] == 0x0A {
            break;
        }
        end += 1;
    }
    
    // Extract hex characters (skip ':' and CR-LF)
    let hex_chars = &input[start + 1..end];
    
    // Decode pairs of hex characters
    let mut decoded = Vec::new();
    for i in (0..hex_chars.len()).step_by(2) {
        if i + 1 >= hex_chars.len() {
            break;
        }
        let byte = decode_ascii_byte(hex_chars[i], hex_chars[i + 1])?;
        decoded.push(byte);
    }
    
    Some(decoded)
}
```

**3. LRC calculation:**
```rust
fn calculate_lrc(data: &[u8]) -> u8 {
    let sum = data.iter().fold(0u8, |acc, &b| acc.wrapping_add(b));
    (!sum).wrapping_add(1)  // Two's complement
}

fn validate_ascii_frame(frame: &[u8]) -> bool {
    if frame.len() < 2 {
        return false;
    }
    
    let data_len = frame.len() - 1;
    let received_lrc = frame[data_len];
    let calculated_lrc = calculate_lrc(&frame[..data_len]);
    
    received_lrc == calculated_lrc
}
```

## Can RTU and ASCII Work Together?

### On the Same Cable?

**On the same cable: NO**

A MODBUS serial cable must use **either** RTU **or** ASCII, not both at the same time:
- They use different ways to mark message boundaries
- They use different character formats (8-bit vs 7-bit)
- They have different timing requirements
- Devices are configured for one mode only

**Using separate cables: YES**

A master device can have:
- Cable 1 using RTU at 9600 baud
- Cable 2 using ASCII at 9600 baud
- Each cable operates independently

### Automatic Detection

**Can a device automatically detect which mode is being used?**

In theory, yes - by looking at the incoming data:
- ASCII always starts with the ':' character (value 0x3A)
- ASCII only uses specific characters: '0'-'9', 'A'-'F', ':', CR, LF
- RTU can use any binary values

However, **most devices don't have automatic detection** - you must configure the mode manually.

### Protocol Conversion

**RTU-to-ASCII converter:**

```rust
fn convert_rtu_to_ascii(rtu_frame: &[u8]) -> Vec<u8> {
    if rtu_frame.len() < 3 {
        return vec![];  // Invalid
    }
    
    // Extract data (exclude CRC)
    let data = &rtu_frame[..rtu_frame.len() - 2];
    
    // Calculate LRC
    let lrc = calculate_lrc(data);
    
    // Build ASCII frame
    let mut ascii = vec![b':'];
    
    for &byte in data {
        let (high, low) = encode_byte_to_ascii(byte);
        ascii.push(high);
        ascii.push(low);
    }
    
    let (lrc_high, lrc_low) = encode_byte_to_ascii(lrc);
    ascii.push(lrc_high);
    ascii.push(lrc_low);
    ascii.push(0x0D);  // CR
    ascii.push(0x0A);  // LF
    
    ascii
}
```

## Testing and Debugging

### RTU Testing Tools

**1. Protocol analyzers:**
- Display binary hex
- Calculate and verify CRC
- Detect timing violations
- Example: Modbus Poll, Modbus Slave simulators

**2. Oscilloscope/logic analyzer:**
- View actual electrical signals
- Measure timing precisely
- Detect electrical issues

**3. Software simulators:**
```rust
// Send RTU request
let request = vec![0x01, 0x03, 0x00, 0x00, 0x00, 0x0A];
let crc = crc16_modbus(&request);
let mut frame = request;
frame.push((crc & 0xFF) as u8);
frame.push((crc >> 8) as u8);

println!("RTU frame (hex): {}", hex::encode(&frame));
// Output: 01030000000acdc5
```

### ASCII Testing Tools

**1. Terminal programs:**
- HyperTerminal, PuTTY, minicom
- Type ASCII frames manually
- View responses directly

**2. Manual frame construction:**
```
Type into terminal:
:01030000000AF2<ENTER>

Device responds:
:0103140001000200030004000500060007000800090000F8<ENTER>
```

**3. Easy debugging:**
```rust
// ASCII is human-readable
let ascii_frame = b":01030000000AF2\r\n";
println!("Sending: {}", String::from_utf8_lossy(ascii_frame));

// Can visually verify:
// - Address: 01
// - Function: 03  
// - Start: 0000
// - Quantity: 000A
// - LRC: F2
```

## Performance Analysis

### Throughput Comparison

**Scenario:** Read 100 holding registers repeatedly at 9600 baud

**RTU:**
```
Request:  8 bytes
Response: 5 + 200 + 2 = 207 bytes
Total:    215 bytes per transaction

Bits: 215 × 11 = 2365 bits
Time: 2365 / 9600 = 246 ms per transaction
Throughput: 4.06 transactions/second
```

**ASCII:**
```
Request:  17 characters
Response: 10 + 400 + 4 = 414 characters
Total:    431 characters per transaction

Bits: 431 × 10 = 4310 bits
Time: 4310 / 9600 = 449 ms per transaction
Throughput: 2.23 transactions/second
```

**RTU is 1.82x faster than ASCII** in this scenario.

### CPU Load Comparison

**CRC-16 calculation (RTU):**
- Bit-by-bit: ~500 cycles per byte (slow)
- Lookup table: ~20 cycles per byte (fast)

**LRC calculation (ASCII):**
- Simple addition: ~10 cycles per byte (fastest)

**But encoding overhead in ASCII:**
- Hex encoding/decoding: ~30 cycles per byte
- Overall: ASCII requires more CPU for encoding

**Conclusion:** RTU with lookup tables is most efficient overall.

## Summary

### Main Differences

| Feature | RTU | ASCII |
|--------|-----|-------|
| How data is sent | Binary numbers | Text characters (doubles the size) |
| Message boundaries | Brief silence (timing-based) | ':' character at start, CR-LF at end |
| Error checking | CRC-16 (very strong) | LRC (weaker) |
| Speed | Fast (100%) | Slower (about 55% as fast) |
| Can humans read it? | No (binary data) | Yes (readable text) |
| Testing | Requires special tools | Can use any terminal program |
| Industry use | 95% or more | Less than 5% |
| Implementation | Moderate difficulty (timing is critical) | Simpler (no timing requirements) |

### Which Should You Use?

**For real-world systems:** Use RTU
- Much better error detection
- More efficient use of bandwidth
- Industry standard - what everyone expects
- Faster data throughput

**For learning and testing:** Use ASCII
- Humans can read the messages
- Easy to test manually
- Good for understanding how MODBUS works
- Can use simple terminal programs

**For new projects:** Always choose RTU unless you have specific reasons to use ASCII.

## Related Pages

- [MODBUS RTU](/wiki/concepts/modbus-rtu.md) - Detailed RTU mode specification
- [MODBUS ASCII](/wiki/concepts/modbus-ascii.md) - Detailed ASCII mode specification
- [CRC-16](/wiki/concepts/crc-16.md) - CRC-16 calculation details
- [Master-Slave Architecture](/wiki/concepts/master-slave.md) - MODBUS communication model

## Backlinks

- [MODBUS Over RS232](/wiki/answers/modbus-over-rs232.md)
- [RS485 Wiring for MODBUS RTU](/wiki/answers/rs485-wiring-for-modbus-rtu.md)
- [CRC-16](/wiki/concepts/crc-16.md)
- [Master-Slave Architecture](/wiki/concepts/master-slave.md)
- [MODBUS ASCII](/wiki/concepts/modbus-ascii.md)
- [MODBUS RTU](/wiki/concepts/modbus-rtu.md)

------------

MODBUS is a trademark of the Modbus Organization, Inc.

All information on this page is derived by summarizing and analyzing solely the MODBUS specifications published by the Modbus Organization, Inc.  The specifications are copyright by the Modbus Organization.

By deriving this information solely from the specifications we hope to stay true to the specification.

No infringement is intended.
