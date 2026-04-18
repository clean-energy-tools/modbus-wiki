---
title: MODBUS Data Type Mapping
Summary: Techniques and patterns for mapping MODBUS register values to/from modern programming language types, including endianness handling, scaling, and multi-register values.
Sources:
  - raw/MODBUS/MODBUS.md
  - raw/MODBUS/modbusprotocolspecification.md
Categories:
  - implementation
  - data-mapping
  - rust
  - programming
date-created: 2026-04-18T12:00:00+03:00
last-updated: 2026-04-18T14:43:24+03:00
---

MODBUS uses a flat array of 16-bit registers, while modern programming languages like Rust have rich type systems with complex data structures. This page describes techniques and patterns for mapping between these two models (source: [modbusprotocolspecification.md](raw/MODBUS/modbusprotocolspecification.md)).

## The Fundamental Mismatch

| Aspect | MODBUS Registers | Modern Languages (Rust) |
|--------|------------------|-------------------------|
| Data model | Flat array of u16 | Rich types: structs, enums, floats, strings |
| Type info | Lost in protocol | Compile-time type safety |
| Alignment | 16-bit boundaries | Various alignments |
| Endianness | Big-endian bytes, device-specific word order | Native endianness |
| Complex types | Device-specific conventions | Language-defined semantics |

**Key challenges:**
- No built-in type information in the protocol
- Multi-register values for 32-bit, 64-bit types
- Device-specific word order conventions
- Scaling factors for engineering units
- String encoding and padding
- Bit fields and packed data

## Common Data Type Representations

### Single-Register Types

| MODBUS Storage | Rust Type | Description | Range |
|-----------------|-----------|-------------|-------|
| 1 register | `u16` | Unsigned 16-bit integer | 0-65535 |
| 1 register | `i16` | Signed 16-bit (two's complement) | -32768 to 32767 |
| 1 register | `f32` (scaled) | Scaled 32-bit float | Device-specific |

**Example: u16/i16**
```rust
// Read single register
let raw_registers = read_holding_registers(100, 1)?;  // Read register 100
let value_u16: u16 = raw_registers[0];              // Direct mapping
let value_i16: i16 = raw_registers[0] as i16;       // Signed interpretation

// Write single register
let write_value: u16 = 1234;
write_single_register(100, write_value)?;
```

### Multi-Register Types

| MODBUS Storage | Rust Type | Register Count | Word Order |
|-----------------|-----------|----------------|------------|
| 2 registers | `u32` | 2 | Device-specific |
| 2 registers | `i32` | 2 | Device-specific |
| 2 registers | `f32` | 2 | Device-specific (IEEE 754) |
| 4 registers | `u64` | 4 | Device-specific |
| 4 registers | `i64` | 4 | Device-specific |
| 4 registers | `f64` | 4 | Device-specific (IEEE 754) |

**Critical:** Word order varies by manufacturer. Always check device documentation.

## Endianness Considerations

MODBUS has two levels of endianness:

### Byte Order (Within Each Register)
- **Always big-endian** (network byte order)
- High byte first, low byte second
- Consistent across all devices

**Example: Register value 0x1234**
```
Wire: 0x12 0x34
       MSB  LSB
```

### Word Order (For Multi-Register Values)
- **Device-specific**
- Can be big-endian or little-endian
- Must be determined from device documentation

**Example: 32-bit value 0x12345678**

**Big-endian word order (most common):**
```
Register N:   0x1234 (MSW)
Register N+1: 0x5678 (LSW)
```

**Little-endian word order (some devices):**
```
Register N:   0x5678 (LSW)
Register N+1: 0x1234 (MSW)
```

## Rust Implementation Patterns

### Define Word Order Enum

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum WordOrder {
    BigEndian,      // MSW first (most common)
    LittleEndian,   // LSW first
}
```

### Define Mapping Trait

```rust
pub trait ModbusMapping<T> {
    /// Convert from MODBUS registers to Rust type
    fn from_registers(registers: &[u16], word_order: WordOrder) -> T;
    
    /// Convert from Rust type to MODBUS registers
    fn to_registers(value: &T, word_order: WordOrder) -> Vec<u16>;
    
    /// Number of registers required for this type
    fn register_count() -> usize;
}
```

### Implement for Basic Types

#### u16 (Trivial Mapping)

```rust
impl ModbusMapping<u16> for u16 {
    fn from_registers(registers: &[u16], _word_order: WordOrder) -> u16 {
        registers[0]
    }
    
    fn to_registers(value: &u16, _word_order: WordOrder) -> Vec<u16> {
        vec![*value]
    }
    
    fn register_count() -> usize {
        1
    }
}
```

#### i16 (Signed 16-bit)

```rust
impl ModbusMapping<i16> for i16 {
    fn from_registers(registers: &[u16], _word_order: WordOrder) -> i16 {
        registers[0] as i16
    }
    
    fn to_registers(value: &i16, _word_order: WordOrder) -> Vec<u16> {
        vec![*value as u16]
    }
    
    fn register_count() -> usize {
        1
    }
}
```

#### u32 (32-bit Unsigned)

```rust
impl ModbusMapping<u32> for u32 {
    fn from_registers(registers: &[u16], word_order: WordOrder) -> u32 {
        let high = registers[0] as u32;
        let low = registers[1] as u32;
        
        match word_order {
            WordOrder::BigEndian => (high << 16) | low,
            WordOrder::LittleEndian => (low << 16) | high,
        }
    }
    
    fn to_registers(value: &u32, word_order: WordOrder) -> Vec<u16> {
        let high = ((value >> 16) & 0xFFFF) as u16;
        let low = (*value & 0xFFFF) as u16;
        
        match word_order {
            WordOrder::BigEndian => vec![high, low],
            WordOrder::LittleEndian => vec![low, high],
        }
    }
    
    fn register_count() -> usize {
        2
    }
}
```

#### i32 (32-bit Signed)

```rust
impl ModbusMapping<i32> for i32 {
    fn from_registers(registers: &[u16], word_order: WordOrder) -> i32 {
        let u32_value = u32::from_registers(registers, word_order);
        u32_value as i32
    }
    
    fn to_registers(value: &i32, word_order: WordOrder) -> Vec<u16> {
        let u32_value = *value as u32;
        u32::to_registers(&u32_value, word_order)
    }
    
    fn register_count() -> usize {
        2
    }
}
```

#### f32 (32-bit Floating Point)

```rust
impl ModbusMapping<f32> for f32 {
    fn from_registers(registers: &[u16], word_order: WordOrder) -> f32 {
        let u32_value = u32::from_registers(registers, word_order);
        f32::from_bits(u32_value)
    }
    
    fn to_registers(value: &f32, word_order: WordOrder) -> Vec<u16> {
        let u32_value = value.to_bits();
        u32::to_registers(&u32_value, word_order)
    }
    
    fn register_count() -> usize {
        2
    }
}
```

#### u64 (64-bit Unsigned)

```rust
impl ModbusMapping<u64> for u64 {
    fn from_registers(registers: &[u16], word_order: WordOrder) -> u64 {
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
    
    fn to_registers(value: &u64, word_order: WordOrder) -> Vec<u16> {
        let w0 = ((value >> 48) & 0xFFFF) as u16;
        let w1 = ((value >> 32) & 0xFFFF) as u16;
        let w2 = ((value >> 16) & 0xFFFF) as u16;
        let w3 = (*value & 0xFFFF) as u16;
        
        match word_order {
            WordOrder::BigEndian => vec![w0, w1, w2, w3],
            WordOrder::LittleEndian => vec![w3, w2, w1, w0],
        }
    }
    
    fn register_count() -> usize {
        4
    }
}
```

### Scaled Values

Many MODBUS devices use integer registers to represent floating-point values with a scale factor.

```rust
#[derive(Debug, Clone, Copy)]
pub struct ScaledValue {
    pub raw: u16,
    pub scale_factor: f32,
}

impl ScaledValue {
    pub fn from_raw(raw: u16, scale_factor: f32) -> Self {
        Self { raw, scale_factor }
    }
    
    pub fn from_engineering(eng: f32, scale_factor: f32) -> Self {
        Self {
            raw: (eng * scale_factor).round() as u16,
            scale_factor,
        }
    }
    
    pub fn engineering_value(&self) -> f32 {
        self.raw as f32 / self.scale_factor
    }
}

// Implement ModbusMapping for ScaledValue
impl ModbusMapping<ScaledValue> for ScaledValue {
    fn from_registers(registers: &[u16], _word_order: WordOrder) -> ScaledValue {
        ScaledValue::from_raw(registers[0], 1.0)
    }
    
    fn to_registers(value: &ScaledValue, _word_order: WordOrder) -> Vec<u16> {
        vec![value.raw]
    }
    
    fn register_count() -> usize {
        1
    }
}
```

**Example: Temperature setpoint**
```rust
// Device uses scale factor of 100 (0-10000 = 0-100.0°C)
let temp_setpoint = ScaledValue::from_engineering(25.5, 100.0);
// temp_setpoint.raw = 2550

let raw_reg = temp_setpoint.to_registers(&temp_setpoint, WordOrder::BigEndian)[0];
write_single_register(10, raw_reg)?;

// Read back
let raw_read = read_holding_registers(10, 1)?[0];
let temp = ScaledValue::from_raw(raw_read, 100.0);
println!("Temperature: {:.1}°C", temp.engineering_value());
```

### Strings

String encoding varies by device. Common approaches:

#### Fixed-Length ASCII Strings

```rust
pub fn decode_fixed_ascii(registers: &[u16], max_chars: usize) -> String {
    let mut bytes = Vec::new();
    for reg in registers {
        bytes.push((reg >> 8) as u8);
        bytes.push((reg & 0xFF) as u8);
    }
    
    // Find null terminator or trim to max_chars
    let len = bytes.iter().position(|&b| b == 0).unwrap_or(bytes.len());
    let len = len.min(max_chars);
    
    String::from_utf8_lossy(&bytes[..len]).to_string()
}

pub fn encode_fixed_ascii(s: &str, num_registers: usize) -> Vec<u16> {
    let mut result = Vec::with_capacity(num_registers);
    let bytes = s.as_bytes();
    
    for i in 0..num_registers {
        let high = if i * 2 < bytes.len() {
            bytes[i * 2]
        } else {
            0 // Padding
        };
        let low = if i * 2 + 1 < bytes.len() {
            bytes[i * 2 + 1]
        } else {
            0 // Padding
        };
        result.push(((high as u16) << 8) | (low as u16));
    }
    
    result
}

// Example
let device_name = encode_fixed_ascii("Temperature Controller", 8);
// 8 registers = 16 bytes max

let read_name = decode_fixed_ascii(&device_name, 16);
// "Temperature Controller"
```

### Bit Fields

```rust
#[derive(Debug, Clone, Copy)]
pub struct DeviceStatus {
    pub running: bool,
    pub fault: bool,
    pub warning: bool,
    pub manual_mode: bool,
    pub remote_mode: bool,
    pub _reserved: u11, // 11 reserved bits
}

impl ModbusMapping<DeviceStatus> for DeviceStatus {
    fn from_registers(registers: &[u16], _word_order: WordOrder) -> DeviceStatus {
        let bits = registers[0];
        DeviceStatus {
            running: (bits & 0x0001) != 0,
            fault: (bits & 0x0002) != 0,
            warning: (bits & 0x0004) != 0,
            manual_mode: (bits & 0x0008) != 0,
            remote_mode: (bits & 0x0010) != 0,
            _reserved: (bits >> 5) & 0x07FF,
        }
    }
    
    fn to_registers(value: &DeviceStatus, _word_order: WordOrder) -> Vec<u16> {
        let mut bits: u16 = 0;
        if value.running { bits |= 0x0001; }
        if value.fault { bits |= 0x0002; }
        if value.warning { bits |= 0x0004; }
        if value.manual_mode { bits |= 0x0008; }
        if value.remote_mode { bits |= 0x0010; }
        bits |= (value._reserved & 0x07FF) << 5;
        vec![bits]
    }
    
    fn register_count() -> usize {
        1
    }
}
```

## Complex Device Mapping

### Define Device Configuration Struct

```rust
#[derive(Debug, Clone)]
pub struct DeviceConfig {
    // Register 0: Operation mode (u16)
    pub mode: u16,
    
    // Register 1: Temperature setpoint (scaled)
    pub temperature_setpoint: ScaledValue,
    
    // Registers 2-3: Motor speed (u32, RPM)
    pub motor_speed: u32,
    
    // Registers 4-5: Motor current (f32, Amps)
    pub motor_current: f32,
    
    // Register 6: Status bits (DeviceStatus)
    pub status: DeviceStatus,
    
    // Registers 7-10: Device name (ASCII, 8 bytes)
    pub device_name: String,
}
```

### Create Device Mapping Handler

```rust
pub struct DeviceConfigMapping {
    word_order: WordOrder,
}

impl DeviceConfigMapping {
    pub fn new(word_order: WordOrder) -> Self {
        Self { word_order }
    }
    
    pub fn from_registers(&self, registers: &[u16]) -> Result<DeviceConfig, &'static str> {
        if registers.len() < 11 {
            return Err("Not enough registers (need 11)");
        }
        
        Ok(DeviceConfig {
            mode: u16::from_registers(&registers[0..1], self.word_order),
            temperature_setpoint: ScaledValue::from_raw(registers[1], 100.0),
            motor_speed: u32::from_registers(&registers[2..4], self.word_order),
            motor_current: f32::from_registers(&registers[4..6], self.word_order),
            status: DeviceStatus::from_registers(&registers[6..7], self.word_order),
            device_name: decode_fixed_ascii(&registers[7..11], 16),
        })
    }
    
    pub fn to_registers(&self, config: &DeviceConfig) -> Vec<u16> {
        let mut result = vec![];
        
        result.extend(u16::to_registers(&config.mode, self.word_order));
        result.extend(ScaledValue::to_registers(&config.temperature_setpoint, self.word_order));
        result.extend(u32::to_registers(&config.motor_speed, self.word_order));
        result.extend(f32::to_registers(&config.motor_current, self.word_order));
        result.extend(DeviceStatus::to_registers(&config.status, self.word_order));
        result.extend(encode_fixed_ascii(&config.device_name, 4));
        
        result
    }
    
    pub fn register_count() -> usize {
        11
    }
}
```

### Usage Example

```rust
// Create mapper with device's word order
let mapper = DeviceConfigMapping::new(WordOrder::BigEndian);

// Read all registers for device config
let registers = read_holding_registers(0, DeviceConfigMapping::register_count())?;

// Parse into Rust struct
let config = mapper.from_registers(&registers)?;
println!("Device: {} (Speed: {} RPM, Current: {:.2} A)", 
         config.device_name, config.motor_speed, config.motor_current);

// Modify configuration
let mut new_config = config.clone();
new_config.motor_speed = 1500;
new_config.temperature_setpoint = ScaledValue::from_engineering(30.0, 100.0);

// Write back to device
let write_regs = mapper.to_registers(&new_config);
write_multiple_registers(0, &write_regs)?;
```

## Python Implementation (For Comparison)

```python
from dataclasses import dataclass
from enum import Enum

class WordOrder(Enum):
    BIG_ENDIAN = "big"
    LITTLE_ENDIAN = "little"

def u16_from_registers(registers, word_order):
    return registers[0]

def u32_from_registers(registers, word_order):
    if word_order == WordOrder.BIG_ENDIAN:
        return (registers[0] << 16) | registers[1]
    else:
        return (registers[1] << 16) | registers[0]

def f32_from_registers(registers, word_order):
    import struct
    u32_value = u32_from_registers(registers, word_order)
    return struct.unpack('!f', struct.pack('!I', u32_value))[0]

@dataclass
class DeviceConfig:
    mode: int
    temperature_setpoint: float  # stored as scaled, exposed as float
    motor_speed: int
    motor_current: float
    status: int

class DeviceConfigMapping:
    def __init__(self, word_order):
        self.word_order = word_order
    
    def from_registers(self, registers):
        if len(registers) < 6:
            raise ValueError("Not enough registers")
        
        return DeviceConfig(
            mode=u16_from_registers(registers[0:1], self.word_order),
            temperature_setpoint=registers[1] / 100.0,  # Scale factor
            motor_speed=u32_from_registers(registers[2:4], self.word_order),
            motor_current=f32_from_registers(registers[4:6], self.word_order),
            status=registers[6]
        )

# Usage
mapper = DeviceConfigMapping(WordOrder.BIG_ENDIAN)
registers = [1, 2550, 0, 1000, 0x423A, 0x8000, 1]  # Example data
config = mapper.from_registers(registers)
print(f"Temperature: {config.temperature_setpoint:.1f}°C")
```

## Common Pitfalls

### 1. Wrong Word Order

**Problem:** Assuming big-endian word order when device uses little-endian.

```rust
// WRONG
let value = u32::from_registers(&regs, WordOrder::BigEndian);  // Always big-endian?

// CORRECT
let value = u32::from_registers(&regs, device_word_order);  // Device-specific
```

**Solution:** Always check device documentation. Store word order per device type.

### 2. Ignoring Scale Factors

**Problem:** Treating raw register values directly as engineering units.

```rust
// WRONG
let temperature: f32 = raw_value as f32;  // 2550 -> 2550.0°C? Wrong!

// CORRECT
let temperature: f32 = ScaledValue::from_raw(raw_value, 100.0).engineering_value();  // 2550 -> 25.5°C
```

**Solution:** Document scale factors and use ScaledValue wrapper.

### 3. Incorrect Byte Order

**Problem:** Treating MODBUS registers as little-endian bytes.

```rust
// WRONG
let value = registers[0] + (registers[1] << 16);  // Wrong byte order!

// CORRECT (but still need to check word order)
let high = registers[0];
let low = registers[1];
let value = (high << 16) | low;  // Correct byte order, check word order
```

**Solution:** MODBUS is always big-endian bytes. Only word order varies.

### 4. Not Validating Register Count

**Problem:** Buffer overflows when reading insufficient registers.

```rust
// WRONG
let value = u32::from_registers(&registers, word_order);  // What if len < 2?

// CORRECT
if registers.len() < 2 {
    return Err("Not enough registers");
}
let value = u32::from_registers(&registers, word_order);
```

**Solution:** Always validate input register counts.

### 5. String Encoding Issues

**Problem:** Assuming UTF-8 when device uses ASCII, or vice versa.

```rust
// WRONG
let s = String::from_utf8_lossy(&bytes).to_string();  // UTF-8?

// CORRECT
let s = decode_fixed_ascii(&registers, max_chars);  // ASCII
```

**Solution:** Check device documentation for string encoding.

## Best Practices

### 1. Document Device Mappings

Create device-specific documentation files:

```toml
# motor_controller.toml
[device]
name = "Motor Controller V2"
model = "MC-200"
word_order = "BigEndian"

[registers.mode]
address = 0
type = "u16"
description = "Operation mode"

[registers.temperature_setpoint]
address = 1
type = "ScaledValue"
scale_factor = 100.0
description = "Temperature setpoint (°C)"

[registers.motor_speed]
address = 2
type = "u32"
word_order = "BigEndian"
description = "Motor speed (RPM)"
```

### 2. Use Type-Safe Wrappers

```rust
// Define type-safe register addresses
#[derive(Debug, Clone, Copy)]
pub struct RegisterAddress(u16);

impl RegisterAddress {
    pub const MODE: Self = Self(0);
    pub const TEMP_SETPOINT: Self = Self(1);
    pub const MOTOR_SPEED: Self = Self(2);
}

pub struct DeviceClient {
    word_order: WordOrder,
}

impl DeviceClient {
    pub fn get_mode(&self) -> Result<u16, Error> {
        self.read_u16(RegisterAddress::MODE)
    }
    
    pub fn set_temperature(&self, temp: f32) -> Result<(), Error> {
        let scaled = ScaledValue::from_engineering(temp, 100.0);
        self.write_u16(RegisterAddress::TEMP_SETPOINT, scaled.raw)
    }
}
```

### 3. Implement Error Handling

```rust
#[derive(Debug)]
pub enum MappingError {
    NotEnoughRegisters { expected: usize, actual: usize },
    InvalidWordOrder,
    InvalidScaleFactor,
    InvalidStringEncoding,
}

impl ModbusMapping<u32> for u32 {
    fn from_registers(registers: &[u16], word_order: WordOrder) -> u32 {
        if registers.len() < 2 {
            panic!("Not enough registers");  // Or use Result
        }
        // ... implementation
    }
}
```

### 4. Test with Real Data

```rust
#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn test_u32_big_endian() {
        let registers = vec![0x1234, 0x5678];
        let value = u32::from_registers(&registers, WordOrder::BigEndian);
        assert_eq!(value, 0x12345678);
    }
    
    #[test]
    fn test_u32_little_endian() {
        let registers = vec![0x5678, 0x1234];
        let value = u32::from_registers(&registers, WordOrder::LittleEndian);
        assert_eq!(value, 0x12345678);
    }
    
    #[test]
    fn test_f32_conversion() {
        let registers = vec![0x423A, 0x8000];  // 47.0 in IEEE 754
        let value = f32::from_registers(&registers, WordOrder::BigEndian);
        assert!((value - 47.0).abs() < 0.01);
    }
}
```

### 5. Use Serde for Complex Types

```rust
use serde::{Serialize, Deserialize};

#[derive(Debug, Serialize, Deserialize)]
pub struct DeviceConfig {
    pub mode: u16,
    pub temperature_setpoint: f32,
    pub motor_speed: u32,
    pub motor_current: f32,
}

// Implement ModbusMapping
impl ModbusMapping<DeviceConfig> for DeviceConfig {
    fn from_registers(registers: &[u16], word_order: WordOrder) -> DeviceConfig {
        DeviceConfig {
            mode: u16::from_registers(&registers[0..1], word_order),
            temperature_setpoint: ScaledValue::from_raw(registers[1], 100.0).engineering_value(),
            motor_speed: u32::from_registers(&registers[2..4], word_order),
            motor_current: f32::from_registers(&registers[4..6], word_order),
        }
    }
    
    fn to_registers(value: &DeviceConfig, word_order: WordOrder) -> Vec<u16> {
        let mut result = vec![];
        result.extend(u16::to_registers(&value.mode, word_order));
        result.push((value.temperature_setpoint * 100.0) as u16);
        result.extend(u32::to_registers(&value.motor_speed, word_order));
        result.extend(f32::to_registers(&value.motor_current, word_order));
        result
    }
    
    fn register_count() -> usize {
        6
    }
}
```

## Related pages

- [[/wiki/concepts/holding-registers]]]
- [[/wiki/concepts/input-registers]]]
- [[/wiki/concepts/modbus-tcp]]]
- [[/wiki/concepts/function-codes]]]

## See Also

### Source Documentation

- [[/wiki/summaries/MODBUS/MODBUS.md]] - Protocol specification for AI implementation
- [[/wiki/summaries/MODBUS/modbusprotocolspecification.md]] - Official MODBUS application protocol specification

### Related Concepts

- [[/wiki/concepts/modbus]] - Comprehensive MODBUS protocol overview
- [[/wiki/concepts/protocol]] - Protocol concepts and communication principles
- [[/wiki/concepts/implementation]] - Implementation best practices and patterns
- [[/wiki/concepts/master-slave]] - Master-slave architecture details