---
title: MODBUS Register Data Representation
Summary: Comprehensive guide to how MODBUS registers store different data types including booleans, integers, floats, and strings, covering register size, byte order, word order, multi-register values, and determining device endianness.
Sources:
  - raw/MODBUS/modbusprotocolspecification.md
  - raw/MODBUS/MODBUS.md
Categories:
  - data-model
  - data-mapping
  - implementation
  - endianness
type: answer
date-created: 2026-04-24T12:00:00+03:00
last-updated: 2026-04-24T12:00:00+03:00
---

MODBUS registers are the fundamental data storage units in the MODBUS protocol. Understanding how different data types are represented in these 16-bit registers is essential for correctly reading from and writing to MODBUS devices.

## Register Size

MODBUS registers are **16 bits (2 bytes)** each (source: [Holding Registers](/wiki/concepts/holding-registers.md), [Input Registers](/wiki/concepts/input-registers.md)).

There are two types of word-based registers in MODBUS:
- **Holding Registers:** 16-bit read/write registers for configuration and setpoints
- **Input Registers:** 16-bit read-only registers for measurements and sensor values

Both types have identical data representation characteristics; the difference is only in access permissions.

## Storing Different Data Types

### Booleans

**Single-bit storage (preferred):**
- Use [Coils](/wiki/concepts/coils.md) (read/write) or [Discrete Inputs](/wiki/concepts/discrete-inputs.md) (read-only)
- Each boolean occupies exactly 1 bit
- Most efficient for digital I/O signals

**Register-based storage:**
- Can store booleans as bit fields within 16-bit registers
- Allows packing up to 16 boolean values in a single register
- Requires bit masking to extract individual values

Example bit field (source: [modbus-data-type-mapping](/wiki/concepts/modbus-data-type-mapping.md:393-432)):
```rust
// Single register holding multiple boolean flags
let bits = register_value;
let running = (bits & 0x0001) != 0;      // Bit 0
let fault = (bits & 0x0002) != 0;        // Bit 1
let warning = (bits & 0x0004) != 0;      // Bit 2
let manual_mode = (bits & 0x0008) != 0;  // Bit 3
```

### Integers

#### Single-Register Integers (16-bit)

**Unsigned 16-bit (u16):**
- Storage: 1 register
- Range: 0 to 65,535
- Direct mapping from register value

**Signed 16-bit (i16):**
- Storage: 1 register
- Range: -32,768 to 32,767
- Uses two's complement representation
- Example: 0xF000 = -4,096 (signed) or 61,440 (unsigned)

#### Multi-Register Integers

**Unsigned 32-bit (u32):**
- Storage: 2 registers
- Range: 0 to 4,294,967,295
- Word order is device-specific (see [Endianness](#endianness-and-byte-order))

**Signed 32-bit (i32):**
- Storage: 2 registers
- Range: -2,147,483,648 to 2,147,483,647
- Two's complement representation
- Word order is device-specific

**64-bit integers (u64/i64):**
- Storage: 4 registers
- Word order is device-specific

### Floating-Point Numbers

**32-bit float (f32):**
- Storage: 2 registers
- Format: IEEE 754 single-precision
- Word order is device-specific
- Example: Temperature, pressure, flow rate values

**64-bit float (f64):**
- Storage: 4 registers
- Format: IEEE 754 double-precision
- Word order is device-specific

**Scaled integer values (alternative to floats):**
Many devices use integer registers with scale factors instead of IEEE 754 floats (source: [holding-registers](/wiki/concepts/holding-registers.md:207-218)):

```
engineering_value = raw_value / scale_factor
raw_value = engineering_value × scale_factor
```

Examples:
- Temperature: 2550 with scale factor 100 = 25.50°C
- Percentage: 875 with scale factor 10 = 87.5%
- Current: 12345 with scale factor 1000 = 12.345 A

### Strings

Strings are stored in consecutive registers using ASCII encoding (source: [modbus-data-type-mapping](/wiki/concepts/modbus-data-type-mapping.md:342-389)).

**Storage format:**
- Each register holds 2 ASCII characters
- Byte order within register: big-endian (high byte first, low byte second)
- Strings may be null-terminated or fixed-length with padding

**Example: "TEMP" in 2 registers**
```
Register 0: 0x5445  ('T' = 0x54, 'E' = 0x45)
Register 1: 0x4D50  ('M' = 0x4D, 'P' = 0x50)
```

**Example: "Temperature Controller" in 12 registers (24 bytes)**
```
Register 0:  0x5465  "Te"
Register 1:  0x6D70  "mp"
Register 2:  0x6572  "er"
...
Register 11: 0x6572  "er"
```

Fixed-length strings shorter than the allocated space are typically padded with null bytes (0x00).

## Endianness and Byte Order

MODBUS has **two levels** of endianness (source: [modbus-data-type-mapping](/wiki/concepts/modbus-data-type-mapping.md:72-104)).

### Byte Order (Within Each Register)

**Always big-endian** (network byte order):
- High byte transmitted/stored first
- Low byte transmitted/stored second
- **Consistent across all MODBUS devices**

Example: Register value 0x1234
```
Wire/Memory: [0x12] [0x34]
              MSB    LSB
```

This is a protocol requirement and never varies by device.

### Word Order (For Multi-Register Values)

**Device-specific** - varies by manufacturer:
- Can be big-endian (most common) or little-endian
- **Must be determined from device documentation**

Example: 32-bit value 0x12345678

**Big-endian word order (most common):**
```
Register N:   0x1234  (MSW - Most Significant Word)
Register N+1: 0x5678  (LSW - Least Significant Word)
```

**Little-endian word order (some devices):**
```
Register N:   0x5678  (LSW - Least Significant Word)
Register N+1: 0x1234  (MSW - Most Significant Word)
```

**Four possible byte/word orderings for 32-bit values:**

| Name | Description | Bytes on wire | Notes |
|------|-------------|---------------|-------|
| Big-endian (ABCD) | MSW first, MSB first | 0x12 0x34 0x56 0x78 | Most common |
| Little-endian word swap (CDAB) | LSW first, MSB first | 0x56 0x78 0x12 0x34 | Some PLCs |
| Mid-big-endian (BADC) | MSW first, LSB first | 0x34 0x12 0x78 0x56 | Rare |
| Little-endian (DCBA) | LSW first, LSB first | 0x78 0x56 0x34 0x12 | Very rare |

Since MODBUS byte order within registers is always big-endian, only ABCD and CDAB are commonly encountered in practice.

## Multi-Register Value Storage

### 32-bit Values (2 Registers)

Implementation example for u32 (source: [modbus-data-type-mapping](/wiki/concepts/modbus-data-type-mapping.md:172-198)):

```rust
// Read from registers
fn u32_from_registers(registers: &[u16], word_order: WordOrder) -> u32 {
    let high = registers[0] as u32;
    let low = registers[1] as u32;
    
    match word_order {
        WordOrder::BigEndian => (high << 16) | low,
        WordOrder::LittleEndian => (low << 16) | high,
    }
}

// Write to registers
fn u32_to_registers(value: u32, word_order: WordOrder) -> Vec<u16> {
    let high = ((value >> 16) & 0xFFFF) as u16;
    let low = (value & 0xFFFF) as u16;
    
    match word_order {
        WordOrder::BigEndian => vec![high, low],
        WordOrder::LittleEndian => vec![low, high],
    }
}
```

### 32-bit IEEE 754 Floats (2 Registers)

Floats use the same word ordering as integers (source: [modbus-data-type-mapping](/wiki/concepts/modbus-data-type-mapping.md:222-238)):

```rust
fn f32_from_registers(registers: &[u16], word_order: WordOrder) -> f32 {
    let u32_value = u32_from_registers(registers, word_order);
    f32::from_bits(u32_value)
}

fn f32_to_registers(value: f32, word_order: WordOrder) -> Vec<u16> {
    let u32_value = value.to_bits();
    u32_to_registers(u32_value, word_order)
}
```

### 64-bit Values (4 Registers)

Similar pattern with 4 registers (source: [modbus-data-type-mapping](/wiki/concepts/modbus-data-type-mapping.md:242-279)):

```rust
fn u64_from_registers(registers: &[u16], word_order: WordOrder) -> u64 {
    match word_order {
        WordOrder::BigEndian => {
            let r0 = registers[0] as u64;
            let r1 = registers[1] as u64;
            let r2 = registers[2] as u64;
            let r3 = registers[3] as u64;
            (r0 << 48) | (r1 << 32) | (r2 << 16) | r3
        }
        WordOrder::LittleEndian => {
            let r0 = registers[0] as u64;
            let r1 = registers[1] as u64;
            let r2 = registers[2] as u64;
            let r3 = registers[3] as u64;
            (r3 << 48) | (r2 << 32) | (r1 << 16) | r0
        }
    }
}
```

## Not-a-Number (NaN) Representation

MODBUS registers **do not have a built-in NaN concept**, but there are several approaches:

### IEEE 754 Floats

If using 32-bit or 64-bit IEEE 754 floats in registers:
- NaN is supported as part of the IEEE 754 standard
- NaN values have specific bit patterns (exponent = all 1s, mantissa ≠ 0)
- Example: 0x7FC00000 is a common NaN representation for f32

### Integer Values

For integer registers, there is **no standard NaN representation**. Common device-specific approaches:

**Reserved sentinel values:**
- 0xFFFF (65,535) for u16
- 0x8000 (-32,768) for i16
- 0x7FFF (32,767) for i16
- 0x0000 (if zero is not a valid measurement)

**Example from device documentation:**
```
Temperature register value 0x8000 indicates "Sensor not connected"
Pressure register value 0xFFFF indicates "Out of range"
```

**Always check device documentation** to understand how invalid/missing data is represented for that specific device.

### Quality Bits Pattern

Some devices use a separate status register with quality bits:
```
Data Register: Actual value
Status Register bits:
  - Bit 0: Valid (1 = valid, 0 = invalid)
  - Bit 1: Out of range
  - Bit 2: Sensor fault
  - Bit 3: Stale data
```

## Determining Device Byte Order

Since word order for multi-register values is **device-specific**, you must determine it for each device type.

### Method 1: Check Device Documentation (Primary Method)

**Always consult the device manual or datasheet** (source: [modbus-data-type-mapping](/wiki/concepts/modbus-data-type-mapping.md:70, 90, 600)).

Look for:
- "Byte order" or "Word order" specifications
- "Endianness" settings
- "Register format" descriptions
- Data table examples showing multi-register values

Common terminology in documentation:
- "Big-endian" / "MSW first" → Big-endian word order
- "Little-endian" / "LSW first" → Little-endian word order
- "ABCD format" → Big-endian
- "CDAB format" → Little-endian word order
- "Byte swap" → Little-endian word order

### Method 2: Test with Known Values

If documentation is unclear or unavailable:

1. **Write a known value** (if device allows writes):
   ```
   Write 32-bit value: 0x12345678
   Read back 2 registers
   
   If registers are [0x1234, 0x5678] → Big-endian word order
   If registers are [0x5678, 0x1234] → Little-endian word order
   ```

2. **Read a known sensor value**:
   ```
   Use an external reference (thermometer, multimeter, etc.)
   Read the MODBUS register value
   Try both byte orders to see which matches the reference
   ```

3. **Look for obvious patterns**:
   ```
   If you read [0x0000, 0x03E8] for what should be "1000":
   - Big-endian: (0x0000 << 16) | 0x03E8 = 1,000 ✓
   - Little-endian: (0x03E8 << 16) | 0x0000 = 65,011,712 ✗
   ```

### Method 3: Common Device Patterns

While not definitive, these patterns are common:

**Typically big-endian word order:**
- Industrial PLCs (Siemens, Allen-Bradley)
- Process controllers
- Flow meters
- Energy meters
- Most industrial automation devices

**Sometimes little-endian word order:**
- PC-based devices
- Some microcontroller-based devices
- Embedded Linux devices
- Devices designed for compatibility with x86 systems

**Important:** These are tendencies only. Always verify for your specific device.

### Method 4: Configuration Switches

Some devices allow configuring byte order:
- DIP switches on the device
- Configuration registers in MODBUS
- Setup software/menu interface
- Web configuration interface

Check the device manual for configuration options.

## Common Pitfalls and Best Practices

### Pitfall 1: Assuming Word Order

**Problem:** Assuming all devices use big-endian word order.

```rust
// WRONG - hardcoded assumption
let value = u32_from_registers(&regs, WordOrder::BigEndian);

// CORRECT - device-specific configuration
let value = u32_from_registers(&regs, device.word_order);
```

**Solution:** Store word order configuration per device type.

### Pitfall 2: Confusing Byte Order and Word Order

**Problem:** Treating MODBUS bytes as little-endian.

MODBUS bytes are **always big-endian**. Only word order varies for multi-register values.

### Pitfall 3: Not Validating Register Count

**Problem:** Reading multi-register values without checking buffer size (source: [modbus-data-type-mapping](/wiki/concepts/modbus-data-type-mapping.md:632-646)).

```rust
// WRONG - no validation
let value = u32_from_registers(&registers, word_order);

// CORRECT - validate first
if registers.len() < 2 {
    return Err("Not enough registers for u32");
}
let value = u32_from_registers(&registers, word_order);
```

### Pitfall 4: Ignoring Scale Factors

**Problem:** Treating scaled integers directly as engineering values (source: [modbus-data-type-mapping](/wiki/concepts/modbus-data-type-mapping.md:603-613)).

```rust
// WRONG
let temperature = raw_value as f32;  // 2550 → 2550°C

// CORRECT
let temperature = (raw_value as f32) / 100.0;  // 2550 → 25.5°C
```

### Best Practice 1: Document Device Mappings

Create configuration files for each device type (source: [modbus-data-type-mapping](/wiki/concepts/modbus-data-type-mapping.md:665-692)):

```toml
[device]
name = "Temperature Controller"
model = "TC-2000"
word_order = "BigEndian"

[registers.temperature]
address = 100
type = "u16"
scale_factor = 100.0
units = "°C"
description = "Process temperature"

[registers.setpoint]
address = 200
type = "u16"
scale_factor = 100.0
units = "°C"
description = "Temperature setpoint"

[registers.power]
address = 300
type = "f32"
word_order = "BigEndian"
units = "kW"
description = "Power consumption"
```

### Best Practice 2: Use Type-Safe Abstractions

Implement type-safe wrappers that encapsulate byte order handling (source: [modbus-data-type-mapping](/wiki/concepts/modbus-data-type-mapping.md:119-131)):

```rust
pub trait ModbusMapping<T> {
    fn from_registers(registers: &[u16], word_order: WordOrder) -> T;
    fn to_registers(value: &T, word_order: WordOrder) -> Vec<u16>;
    fn register_count() -> usize;
}
```

### Best Practice 3: Test with Real Data

Always test data conversions with known values (source: [modbus-data-type-mapping](/wiki/concepts/modbus-data-type-mapping.md:746-772)):

```rust
#[cfg(test)]
mod tests {
    #[test]
    fn test_u32_big_endian() {
        let registers = vec![0x1234, 0x5678];
        let value = u32_from_registers(&registers, WordOrder::BigEndian);
        assert_eq!(value, 0x12345678);
    }
    
    #[test]
    fn test_f32_conversion() {
        let registers = vec![0x423A, 0x8000];  // 46.5 in IEEE 754
        let value = f32_from_registers(&registers, WordOrder::BigEndian);
        assert!((value - 46.5).abs() < 0.01);
    }
}
```

### Best Practice 4: Handle Invalid Data

Check for sentinel values that indicate invalid/missing data:

```rust
fn read_temperature(registers: &[u16]) -> Option<f32> {
    let raw = registers[0];
    
    // Check for "invalid" sentinel value
    if raw == 0x8000 {
        return None;  // Sensor disconnected
    }
    
    Some((raw as f32) / 100.0)
}
```

## Summary Table

| Data Type | Registers | Byte Order | Word Order | Range/Notes |
|-----------|-----------|------------|------------|-------------|
| Boolean (bit field) | 1 | N/A | N/A | 16 booleans per register |
| u16 | 1 | Big-endian | N/A | 0 to 65,535 |
| i16 | 1 | Big-endian | N/A | -32,768 to 32,767 |
| u32 | 2 | Big-endian | Device-specific | 0 to 4,294,967,295 |
| i32 | 2 | Big-endian | Device-specific | -2,147,483,648 to 2,147,483,647 |
| f32 (IEEE 754) | 2 | Big-endian | Device-specific | ±1.5×10⁻⁴⁵ to ±3.4×10³⁸ |
| u64 | 4 | Big-endian | Device-specific | 0 to 2⁶⁴-1 |
| i64 | 4 | Big-endian | Device-specific | -2⁶³ to 2⁶³-1 |
| f64 (IEEE 754) | 4 | Big-endian | Device-specific | ±5.0×10⁻³²⁴ to ±1.7×10³⁰⁸ |
| Scaled integer | 1 | Big-endian | N/A | Raw value ÷ scale factor |
| ASCII string | N/2 | Big-endian | N/A | 2 chars per register |

**Key takeaway:** Byte order within registers is always big-endian. Word order for multi-register values is device-specific and must be determined from documentation or testing.

## Related Pages

- [MODBUS Data Type Mapping](/wiki/concepts/modbus-data-type-mapping.md) - Detailed implementation patterns in Rust and Python
- [Holding Registers](/wiki/concepts/holding-registers.md) - 16-bit read/write registers
- [Input Registers](/wiki/concepts/input-registers.md) - 16-bit read-only registers
- [Coils](/wiki/concepts/coils.md) - Single-bit read/write objects
- [Discrete Inputs](/wiki/concepts/discrete-inputs.md) - Single-bit read-only objects
- [MODBUS Data Validation](/wiki/answers/modbus-data-validation.md) - Validating register values

## Backlinks

No backlinks (yet)

------------

MODBUS is a trademark of the Modbus Organization, Inc.

All information on this page is derived by summarizing and analyzing solely the MODBUS specifications published by the Modbus Organization, Inc.  The specifications are copyright by the Modbus Organization.

By deriving this information solely from the specifications we hope to stay true to the specification.

No infringement is intended.

- [Converting MODBUS Registers to Program Variables](/wiki/answers/converting-modbus-registers-to-program-variables.md) - Type-safe conversion with Rust examples
