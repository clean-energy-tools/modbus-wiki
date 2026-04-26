---
title: Converting MODBUS Registers to Program Variables
Summary: Comprehensive guide to converting between MODBUS register data and modern programming language variables, handling type system mismatches, endianness, and floating-point representation, with complete Rust implementation examples.
Sources:
  - raw/MODBUS/modbusprotocolspecification.md
  - raw/MODBUS/MODBUS.md
Categories:
  - data-mapping
  - implementation
  - rust
  - type-conversion
type: answer
date-created: 2026-04-24T12:30:00+03:00
last-updated: 2026-04-24T12:30:00+03:00
---

Converting between MODBUS registers and modern programming language variables is challenging because MODBUS uses a simple flat array of 16-bit registers while modern languages have rich type systems with integers, floats, strings, structs, and enums. This guide shows how to bridge this gap with practical Rust implementations.

## The Core Problem

### Type System Mismatch

| Aspect | MODBUS | Modern Languages (Rust) |
|--------|--------|-------------------------|
| Data model | Flat `u16` array | Rich types: i32, f32, String, structs |
| Type safety | None (all registers are u16) | Compile-time type checking |
| Metadata | No type information | Full type information |
| Endianness | Big-endian bytes, device-specific words | Native endianness |
| Floats | IEEE 754 in registers (if supported) | Native IEEE 754 |

**The challenge:** You must manually track which registers hold which types, handle endianness conversion, and deal with multi-register values correctly.

## Design Principles

### 1. Type-Safe Abstractions

Don't work with raw `Vec<u16>` everywhere. Create abstractions that encapsulate the conversion logic.

**Bad approach:**
```rust
// Manual bit manipulation everywhere - error-prone
let value = ((registers[0] as u32) << 16) | (registers[1] as u32);
```

**Good approach:**
```rust
// Type-safe conversion with error handling
let value = u32::from_modbus_registers(&registers, WordOrder::BigEndian)?;
```

### 2. Explicit Word Order

Always make word order explicit - never assume big-endian or little-endian.

**Bad approach:**
```rust
// Implicit assumption - will break with some devices
let value = u32_from_registers(&regs);
```

**Good approach:**
```rust
// Explicit word order from device configuration
let value = u32_from_registers(&regs, device.word_order);
```

### 3. Error Handling

MODBUS conversions can fail (wrong register count, invalid values). Use `Result` types.

**Bad approach:**
```rust
// Panic on error
fn from_registers(regs: &[u16]) -> u32 {
    assert!(regs.len() >= 2);  // Will crash
    // ...
}
```

**Good approach:**
```rust
// Proper error handling
fn from_registers(regs: &[u16]) -> Result<u32, ConversionError> {
    if regs.len() < 2 {
        return Err(ConversionError::NotEnoughRegisters { 
            expected: 2, 
            actual: regs.len() 
        });
    }
    // ...
}
```

## Complete Rust Implementation

### Step 1: Define Core Types

```rust
/// Word order for multi-register values
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum WordOrder {
    /// Most significant word first (most common)
    BigEndian,
    /// Least significant word first (some devices)
    LittleEndian,
}

/// Errors that can occur during MODBUS data conversion
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum ConversionError {
    /// Not enough registers for the requested type
    NotEnoughRegisters { expected: usize, actual: usize },
    /// Invalid scale factor (must be > 0)
    InvalidScaleFactor,
    /// Invalid string encoding
    InvalidStringEncoding,
    /// Value out of valid range
    ValueOutOfRange { value: String, valid_range: String },
}

impl std::fmt::Display for ConversionError {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        match self {
            ConversionError::NotEnoughRegisters { expected, actual } => {
                write!(f, "Not enough registers: expected {}, got {}", expected, actual)
            }
            ConversionError::InvalidScaleFactor => {
                write!(f, "Scale factor must be greater than 0")
            }
            ConversionError::InvalidStringEncoding => {
                write!(f, "Invalid string encoding in registers")
            }
            ConversionError::ValueOutOfRange { value, valid_range } => {
                write!(f, "Value {} out of valid range {}", value, valid_range)
            }
        }
    }
}

impl std::error::Error for ConversionError {}

/// Result type for MODBUS conversions
pub type ConversionResult<T> = Result<T, ConversionError>;
```

### Step 2: Define Conversion Trait

```rust
/// Trait for types that can be converted to/from MODBUS registers
pub trait ModbusConversion: Sized {
    /// Convert from MODBUS registers to Rust type
    fn from_registers(registers: &[u16], word_order: WordOrder) -> ConversionResult<Self>;
    
    /// Convert from Rust type to MODBUS registers
    fn to_registers(&self, word_order: WordOrder) -> Vec<u16>;
    
    /// Number of registers required for this type
    fn register_count() -> usize;
}
```

### Step 3: Implement for Basic Types

#### u16 (Single Register, No Endianness Issues)

```rust
impl ModbusConversion for u16 {
    fn from_registers(registers: &[u16], _word_order: WordOrder) -> ConversionResult<Self> {
        if registers.is_empty() {
            return Err(ConversionError::NotEnoughRegisters {
                expected: 1,
                actual: 0,
            });
        }
        Ok(registers[0])
    }
    
    fn to_registers(&self, _word_order: WordOrder) -> Vec<u16> {
        vec![*self]
    }
    
    fn register_count() -> usize {
        1
    }
}
```

#### i16 (Signed 16-bit)

```rust
impl ModbusConversion for i16 {
    fn from_registers(registers: &[u16], _word_order: WordOrder) -> ConversionResult<Self> {
        if registers.is_empty() {
            return Err(ConversionError::NotEnoughRegisters {
                expected: 1,
                actual: 0,
            });
        }
        // Reinterpret u16 as i16 (two's complement)
        Ok(registers[0] as i16)
    }
    
    fn to_registers(&self, _word_order: WordOrder) -> Vec<u16> {
        vec![*self as u16]
    }
    
    fn register_count() -> usize {
        1
    }
}
```

#### u32 (32-bit Unsigned, Word Order Matters)

```rust
impl ModbusConversion for u32 {
    fn from_registers(registers: &[u16], word_order: WordOrder) -> ConversionResult<Self> {
        if registers.len() < 2 {
            return Err(ConversionError::NotEnoughRegisters {
                expected: 2,
                actual: registers.len(),
            });
        }
        
        let high = registers[0] as u32;
        let low = registers[1] as u32;
        
        let value = match word_order {
            WordOrder::BigEndian => {
                // MSW first: [0x1234, 0x5678] -> 0x12345678
                (high << 16) | low
            }
            WordOrder::LittleEndian => {
                // LSW first: [0x5678, 0x1234] -> 0x12345678
                (low << 16) | high
            }
        };
        
        Ok(value)
    }
    
    fn to_registers(&self, word_order: WordOrder) -> Vec<u16> {
        let high = ((self >> 16) & 0xFFFF) as u16;
        let low = (self & 0xFFFF) as u16;
        
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
impl ModbusConversion for i32 {
    fn from_registers(registers: &[u16], word_order: WordOrder) -> ConversionResult<Self> {
        // Convert as u32, then reinterpret as i32
        let u32_value = u32::from_registers(registers, word_order)?;
        Ok(u32_value as i32)
    }
    
    fn to_registers(&self, word_order: WordOrder) -> Vec<u16> {
        let u32_value = *self as u32;
        u32_value.to_registers(word_order)
    }
    
    fn register_count() -> usize {
        2
    }
}
```

#### f32 (32-bit Float, IEEE 754)

```rust
impl ModbusConversion for f32 {
    fn from_registers(registers: &[u16], word_order: WordOrder) -> ConversionResult<Self> {
        // Get u32 bit pattern, then convert to f32
        let u32_value = u32::from_registers(registers, word_order)?;
        Ok(f32::from_bits(u32_value))
    }
    
    fn to_registers(&self, word_order: WordOrder) -> Vec<u16> {
        // Get bit pattern as u32, then convert to registers
        let u32_value = self.to_bits();
        u32_value.to_registers(word_order)
    }
    
    fn register_count() -> usize {
        2
    }
}
```

#### u64 (64-bit Unsigned)

```rust
impl ModbusConversion for u64 {
    fn from_registers(registers: &[u16], word_order: WordOrder) -> ConversionResult<Self> {
        if registers.len() < 4 {
            return Err(ConversionError::NotEnoughRegisters {
                expected: 4,
                actual: registers.len(),
            });
        }
        
        let value = match word_order {
            WordOrder::BigEndian => {
                // MSW first: [r0, r1, r2, r3]
                let r0 = registers[0] as u64;
                let r1 = registers[1] as u64;
                let r2 = registers[2] as u64;
                let r3 = registers[3] as u64;
                (r0 << 48) | (r1 << 32) | (r2 << 16) | r3
            }
            WordOrder::LittleEndian => {
                // LSW first: [r3, r2, r1, r0]
                let r0 = registers[0] as u64;
                let r1 = registers[1] as u64;
                let r2 = registers[2] as u64;
                let r3 = registers[3] as u64;
                (r3 << 48) | (r2 << 32) | (r1 << 16) | r0
            }
        };
        
        Ok(value)
    }
    
    fn to_registers(&self, word_order: WordOrder) -> Vec<u16> {
        let w0 = ((self >> 48) & 0xFFFF) as u16;
        let w1 = ((self >> 32) & 0xFFFF) as u16;
        let w2 = ((self >> 16) & 0xFFFF) as u16;
        let w3 = (self & 0xFFFF) as u16;
        
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

#### i64 and f64 (64-bit Signed and Float)

```rust
impl ModbusConversion for i64 {
    fn from_registers(registers: &[u16], word_order: WordOrder) -> ConversionResult<Self> {
        let u64_value = u64::from_registers(registers, word_order)?;
        Ok(u64_value as i64)
    }
    
    fn to_registers(&self, word_order: WordOrder) -> Vec<u16> {
        (*self as u64).to_registers(word_order)
    }
    
    fn register_count() -> usize {
        4
    }
}

impl ModbusConversion for f64 {
    fn from_registers(registers: &[u16], word_order: WordOrder) -> ConversionResult<Self> {
        let u64_value = u64::from_registers(registers, word_order)?;
        Ok(f64::from_bits(u64_value))
    }
    
    fn to_registers(&self, word_order: WordOrder) -> Vec<u16> {
        self.to_bits().to_registers(word_order)
    }
    
    fn register_count() -> usize {
        4
    }
}
```

### Step 4: Scaled Values

Many MODBUS devices use scaled integers instead of floats (source: [holding-registers](/wiki/concepts/holding-registers.md:207-218)).

```rust
/// Scaled value representation (integer with scale factor)
#[derive(Debug, Clone, Copy, PartialEq)]
pub struct ScaledValue {
    /// Raw register value
    pub raw: u16,
    /// Scale factor (engineering_value = raw / scale_factor)
    pub scale_factor: f32,
}

impl ScaledValue {
    /// Create from raw register value
    pub fn from_raw(raw: u16, scale_factor: f32) -> ConversionResult<Self> {
        if scale_factor <= 0.0 {
            return Err(ConversionError::InvalidScaleFactor);
        }
        Ok(Self { raw, scale_factor })
    }
    
    /// Create from engineering value
    pub fn from_engineering(value: f32, scale_factor: f32) -> ConversionResult<Self> {
        if scale_factor <= 0.0 {
            return Err(ConversionError::InvalidScaleFactor);
        }
        
        let raw_float = value * scale_factor;
        
        // Check if value fits in u16
        if raw_float < 0.0 || raw_float > 65535.0 {
            return Err(ConversionError::ValueOutOfRange {
                value: value.to_string(),
                valid_range: format!("0.0 to {}", 65535.0 / scale_factor),
            });
        }
        
        Ok(Self {
            raw: raw_float.round() as u16,
            scale_factor,
        })
    }
    
    /// Get engineering value
    pub fn engineering_value(&self) -> f32 {
        self.raw as f32 / self.scale_factor
    }
}

impl ModbusConversion for ScaledValue {
    fn from_registers(registers: &[u16], _word_order: WordOrder) -> ConversionResult<Self> {
        if registers.is_empty() {
            return Err(ConversionError::NotEnoughRegisters {
                expected: 1,
                actual: 0,
            });
        }
        // Default scale factor of 1.0 (caller should set correct scale)
        Ok(ScaledValue {
            raw: registers[0],
            scale_factor: 1.0,
        })
    }
    
    fn to_registers(&self, _word_order: WordOrder) -> Vec<u16> {
        vec![self.raw]
    }
    
    fn register_count() -> usize {
        1
    }
}
```

**Usage example:**
```rust
// Temperature sensor with scale factor 100 (0-10000 = 0-100.0°C)
let registers = vec![2550];  // Raw value from device
let temp = ScaledValue::from_raw(registers[0], 100.0)?;
println!("Temperature: {:.1}°C", temp.engineering_value());  // 25.5°C

// Set temperature setpoint
let setpoint = ScaledValue::from_engineering(30.5, 100.0)?;
let write_regs = setpoint.to_registers(WordOrder::BigEndian);
// write_regs = [3050]
```

### Step 5: Strings

```rust
/// Convert ASCII string from MODBUS registers
pub fn string_from_registers(
    registers: &[u16],
    max_chars: usize,
) -> ConversionResult<String> {
    let mut bytes = Vec::with_capacity(registers.len() * 2);
    
    // Each register holds 2 bytes (big-endian)
    for &reg in registers {
        bytes.push((reg >> 8) as u8);   // High byte
        bytes.push((reg & 0xFF) as u8); // Low byte
    }
    
    // Find null terminator or trim to max length
    let end = bytes
        .iter()
        .position(|&b| b == 0)
        .unwrap_or(bytes.len())
        .min(max_chars);
    
    // Convert to string (handle invalid UTF-8 gracefully)
    String::from_utf8(bytes[..end].to_vec())
        .or_else(|_| {
            // Fall back to lossy conversion if not valid UTF-8
            Ok(String::from_utf8_lossy(&bytes[..end]).to_string())
        })
}

/// Convert string to MODBUS registers (ASCII, big-endian)
pub fn string_to_registers(s: &str, num_registers: usize) -> Vec<u16> {
    let mut result = Vec::with_capacity(num_registers);
    let bytes = s.as_bytes();
    
    for i in 0..num_registers {
        let high = if i * 2 < bytes.len() {
            bytes[i * 2]
        } else {
            0 // Padding with null bytes
        };
        
        let low = if i * 2 + 1 < bytes.len() {
            bytes[i * 2 + 1]
        } else {
            0 // Padding with null bytes
        };
        
        result.push(((high as u16) << 8) | (low as u16));
    }
    
    result
}
```

**Usage example:**
```rust
// Read device name (8 registers = 16 bytes max)
let registers = vec![0x5465, 0x6D70, 0x2043, 0x6F6E, 0x7472, 0x6F6C, 0x6C65, 0x7200];
let name = string_from_registers(&registers, 16)?;
// name = "Temp Controller"

// Write device name
let name = "PLC-001";
let write_regs = string_to_registers(name, 4);  // 4 registers = 8 bytes
// write_regs = [0x504C, 0x432D, 0x3030, 0x3100]
```

### Step 6: Boolean Bit Fields

```rust
/// Extract boolean from specific bit in register
pub fn bool_from_bit(register: u16, bit_index: u8) -> bool {
    if bit_index >= 16 {
        return false; // Invalid bit index
    }
    (register & (1 << bit_index)) != 0
}

/// Set boolean in specific bit of register
pub fn bool_to_bit(register: u16, bit_index: u8, value: bool) -> u16 {
    if bit_index >= 16 {
        return register; // Invalid bit index, return unchanged
    }
    
    if value {
        register | (1 << bit_index)  // Set bit
    } else {
        register & !(1 << bit_index) // Clear bit
    }
}

/// Bit field structure example
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct StatusFlags {
    pub running: bool,
    pub fault: bool,
    pub warning: bool,
    pub manual_mode: bool,
    pub remote_enabled: bool,
}

impl ModbusConversion for StatusFlags {
    fn from_registers(registers: &[u16], _word_order: WordOrder) -> ConversionResult<Self> {
        if registers.is_empty() {
            return Err(ConversionError::NotEnoughRegisters {
                expected: 1,
                actual: 0,
            });
        }
        
        let bits = registers[0];
        Ok(StatusFlags {
            running: (bits & 0x0001) != 0,      // Bit 0
            fault: (bits & 0x0002) != 0,        // Bit 1
            warning: (bits & 0x0004) != 0,      // Bit 2
            manual_mode: (bits & 0x0008) != 0,  // Bit 3
            remote_enabled: (bits & 0x0010) != 0, // Bit 4
        })
    }
    
    fn to_registers(&self, _word_order: WordOrder) -> Vec<u16> {
        let mut bits: u16 = 0;
        if self.running { bits |= 0x0001; }
        if self.fault { bits |= 0x0002; }
        if self.warning { bits |= 0x0004; }
        if self.manual_mode { bits |= 0x0008; }
        if self.remote_enabled { bits |= 0x0010; }
        vec![bits]
    }
    
    fn register_count() -> usize {
        1
    }
}
```

**Usage example:**
```rust
// Read status flags
let registers = vec![0x0005];  // Binary: 0000 0000 0000 0101
let status = StatusFlags::from_registers(&registers, WordOrder::BigEndian)?;
assert!(status.running);   // Bit 0 set
assert!(!status.fault);    // Bit 1 clear
assert!(status.warning);   // Bit 2 set

// Write status flags
let status = StatusFlags {
    running: true,
    fault: false,
    warning: true,
    manual_mode: false,
    remote_enabled: true,
};
let write_regs = status.to_registers(WordOrder::BigEndian);
// write_regs = [0x0015]  // Binary: 0000 0000 0001 0101
```

### Step 7: Complex Device Structures

```rust
/// Complete device configuration structure
#[derive(Debug, Clone, PartialEq)]
pub struct DeviceConfig {
    /// Register 0: Operation mode
    pub mode: u16,
    
    /// Register 1: Temperature setpoint (scaled, °C)
    pub temperature_setpoint: f32,
    
    /// Registers 2-3: Motor speed (RPM)
    pub motor_speed: u32,
    
    /// Registers 4-5: Current draw (Amps)
    pub current: f32,
    
    /// Register 6: Status flags
    pub status: StatusFlags,
    
    /// Registers 7-10: Device name (8 bytes)
    pub device_name: String,
}

impl DeviceConfig {
    /// Convert from MODBUS registers
    pub fn from_registers(
        registers: &[u16],
        word_order: WordOrder,
        temp_scale: f32,
    ) -> ConversionResult<Self> {
        const REQUIRED_REGS: usize = 11;
        
        if registers.len() < REQUIRED_REGS {
            return Err(ConversionError::NotEnoughRegisters {
                expected: REQUIRED_REGS,
                actual: registers.len(),
            });
        }
        
        Ok(DeviceConfig {
            // Register 0: mode
            mode: u16::from_registers(&registers[0..1], word_order)?,
            
            // Register 1: temperature setpoint (scaled)
            temperature_setpoint: {
                let scaled = ScaledValue::from_raw(registers[1], temp_scale)?;
                scaled.engineering_value()
            },
            
            // Registers 2-3: motor speed
            motor_speed: u32::from_registers(&registers[2..4], word_order)?,
            
            // Registers 4-5: current (IEEE 754 float)
            current: f32::from_registers(&registers[4..6], word_order)?,
            
            // Register 6: status flags
            status: StatusFlags::from_registers(&registers[6..7], word_order)?,
            
            // Registers 7-10: device name
            device_name: string_from_registers(&registers[7..11], 8)?,
        })
    }
    
    /// Convert to MODBUS registers
    pub fn to_registers(
        &self,
        word_order: WordOrder,
        temp_scale: f32,
    ) -> ConversionResult<Vec<u16>> {
        let mut result = Vec::with_capacity(11);
        
        // Register 0: mode
        result.extend(self.mode.to_registers(word_order));
        
        // Register 1: temperature setpoint (scaled)
        let temp_scaled = ScaledValue::from_engineering(self.temperature_setpoint, temp_scale)?;
        result.extend(temp_scaled.to_registers(word_order));
        
        // Registers 2-3: motor speed
        result.extend(self.motor_speed.to_registers(word_order));
        
        // Registers 4-5: current
        result.extend(self.current.to_registers(word_order));
        
        // Register 6: status flags
        result.extend(self.status.to_registers(word_order));
        
        // Registers 7-10: device name
        result.extend(string_to_registers(&self.device_name, 4));
        
        Ok(result)
    }
}
```

**Usage example:**
```rust
// Read complete device configuration
let registers = read_holding_registers(0, 11)?;  // Read 11 registers starting at 0
let config = DeviceConfig::from_registers(
    &registers,
    WordOrder::BigEndian,
    100.0,  // Temperature scale factor
)?;

println!("Device: {}", config.device_name);
println!("Mode: {}", config.mode);
println!("Temperature setpoint: {:.1}°C", config.temperature_setpoint);
println!("Motor speed: {} RPM", config.motor_speed);
println!("Current: {:.2} A", config.current);
println!("Status: {:?}", config.status);

// Modify and write back
let mut new_config = config.clone();
new_config.temperature_setpoint = 25.5;
new_config.motor_speed = 1800;

let write_regs = new_config.to_registers(WordOrder::BigEndian, 100.0)?;
write_multiple_registers(0, &write_regs)?;
```

## Handling Endianness Properly

### Byte Order vs Word Order

**MODBUS has two levels of endianness** (source: [MODBUS Register Data Representation](/wiki/answers/modbus-register-data-representation.md)):

1. **Byte order within each register**: Always big-endian (high byte first)
2. **Word order for multi-register values**: Device-specific

**Example: 32-bit value 0x12345678**

```
Byte order (always big-endian):
  Register 0: [0x12] [0x34]  (not [0x34] [0x12])
              MSB    LSB

Word order (device-specific):
  Big-endian:    Register 0 = 0x1234, Register 1 = 0x5678
  Little-endian: Register 0 = 0x5678, Register 1 = 0x1234
```

### Testing Endianness

```rust
#[cfg(test)]
mod tests {
    use super::*;
    
    #[test]
    fn test_u32_big_endian() {
        let registers = vec![0x1234, 0x5678];
        let value = u32::from_registers(&registers, WordOrder::BigEndian).unwrap();
        assert_eq!(value, 0x12345678);
        
        let back = value.to_registers(WordOrder::BigEndian);
        assert_eq!(back, registers);
    }
    
    #[test]
    fn test_u32_little_endian() {
        let registers = vec![0x5678, 0x1234];
        let value = u32::from_registers(&registers, WordOrder::LittleEndian).unwrap();
        assert_eq!(value, 0x12345678);
        
        let back = value.to_registers(WordOrder::LittleEndian);
        assert_eq!(back, registers);
    }
    
    #[test]
    fn test_f32_ieee754() {
        // IEEE 754: 46.5 = 0x423A0000
        let registers = vec![0x423A, 0x0000];
        let value = f32::from_registers(&registers, WordOrder::BigEndian).unwrap();
        assert!((value - 46.5).abs() < 0.001);
    }
    
    #[test]
    fn test_scaled_value() {
        let temp = ScaledValue::from_engineering(25.5, 100.0).unwrap();
        assert_eq!(temp.raw, 2550);
        assert!((temp.engineering_value() - 25.5).abs() < 0.01);
    }
    
    #[test]
    fn test_string_conversion() {
        let s = "TEMP";
        let regs = string_to_registers(s, 2);
        assert_eq!(regs, vec![0x5445, 0x4D50]);
        
        let back = string_from_registers(&regs, 4).unwrap();
        assert_eq!(back, "TEMP");
    }
    
    #[test]
    fn test_status_flags() {
        let status = StatusFlags {
            running: true,
            fault: false,
            warning: true,
            manual_mode: false,
            remote_enabled: true,
        };
        
        let regs = status.to_registers(WordOrder::BigEndian);
        assert_eq!(regs[0], 0x0015);  // Bits 0, 2, 4 set
        
        let back = StatusFlags::from_registers(&regs, WordOrder::BigEndian).unwrap();
        assert_eq!(back, status);
    }
}
```

## Handling IEEE 754 Floats

### The Challenge

MODBUS devices that support floats use IEEE 754 format, but the word order may vary.

### Float Conversion Detail

```rust
// f32 is 32 bits = 4 bytes = 2 registers

// IEEE 754 single precision format:
// [Sign: 1 bit][Exponent: 8 bits][Mantissa: 23 bits]

// Example: 46.5 decimal
// Binary: 1.0111010 × 2^5
// IEEE 754: 0 10000100 01110100000000000000000
// Hex: 0x423A0000

// In MODBUS registers (big-endian word order):
// Register 0: 0x423A
// Register 1: 0x0000

fn demonstrate_float_conversion() {
    let value: f32 = 46.5;
    
    // Get IEEE 754 bit pattern
    let bits: u32 = value.to_bits();
    println!("Float: {}", value);
    println!("Bits: 0x{:08X}", bits);
    // Output: Bits: 0x423A0000
    
    // Convert to MODBUS registers (big-endian)
    let high = ((bits >> 16) & 0xFFFF) as u16;
    let low = (bits & 0xFFFF) as u16;
    println!("Registers: [0x{:04X}, 0x{:04X}]", high, low);
    // Output: Registers: [0x423A, 0x0000]
    
    // Convert back
    let reconstructed = ((high as u32) << 16) | (low as u32);
    let float_back = f32::from_bits(reconstructed);
    println!("Reconstructed: {}", float_back);
    // Output: Reconstructed: 46.5
}
```

### Handling Special Float Values

```rust
/// Check for special IEEE 754 values
pub fn classify_f32(value: f32) -> &'static str {
    if value.is_nan() {
        "NaN"
    } else if value.is_infinite() {
        if value.is_sign_positive() {
            "+Infinity"
        } else {
            "-Infinity"
        }
    } else if value == 0.0 {
        if value.is_sign_positive() {
            "+Zero"
        } else {
            "-Zero"
        }
    } else {
        "Normal"
    }
}

// Example: Handle NaN from device
let registers = vec![0x7FC0, 0x0000];  // NaN pattern
let value = f32::from_registers(&registers, WordOrder::BigEndian)?;
if value.is_nan() {
    println!("Device reported invalid/missing data (NaN)");
    // Handle appropriately
}
```

## Best Practices

### 1. Validate Register Count

Always check you have enough registers before conversion:

```rust
fn safe_conversion_example(registers: &[u16]) -> ConversionResult<(u32, f32)> {
    // Need 4 registers: 2 for u32, 2 for f32
    if registers.len() < 4 {
        return Err(ConversionError::NotEnoughRegisters {
            expected: 4,
            actual: registers.len(),
        });
    }
    
    let speed = u32::from_registers(&registers[0..2], WordOrder::BigEndian)?;
    let current = f32::from_registers(&registers[2..4], WordOrder::BigEndian)?;
    
    Ok((speed, current))
}
```

### 2. Document Device-Specific Details

Create configuration structures for each device type:

```rust
/// Device-specific configuration
#[derive(Debug, Clone)]
pub struct DeviceProfile {
    pub model: String,
    pub word_order: WordOrder,
    pub temp_scale_factor: f32,
    pub register_map: RegisterMap,
}

#[derive(Debug, Clone)]
pub struct RegisterMap {
    pub mode_addr: u16,
    pub temp_setpoint_addr: u16,
    pub motor_speed_addr: u16,
    pub current_addr: u16,
    pub status_addr: u16,
}

// Example profile for specific device
const DEVICE_ABC123: DeviceProfile = DeviceProfile {
    model: "ABC-123".to_string(),
    word_order: WordOrder::BigEndian,
    temp_scale_factor: 100.0,
    register_map: RegisterMap {
        mode_addr: 0,
        temp_setpoint_addr: 1,
        motor_speed_addr: 2,
        current_addr: 4,
        status_addr: 6,
    },
};
```

### 3. Use Type-Safe Register Addresses

```rust
/// Type-safe register address
#[derive(Debug, Clone, Copy, PartialEq, Eq, PartialOrd, Ord)]
pub struct RegisterAddress(u16);

impl RegisterAddress {
    pub const MODE: Self = Self(0);
    pub const TEMP_SETPOINT: Self = Self(1);
    pub const MOTOR_SPEED: Self = Self(2);  // 2 registers
    pub const CURRENT: Self = Self(4);      // 2 registers
    pub const STATUS: Self = Self(6);
    
    pub fn value(&self) -> u16 {
        self.0
    }
}

// Usage
let mode_reg = read_holding_registers(RegisterAddress::MODE.value(), 1)?;
let mode = u16::from_registers(&mode_reg, WordOrder::BigEndian)?;
```

### 4. Handle Errors Gracefully

```rust
/// High-level device client with error handling
pub struct DeviceClient {
    word_order: WordOrder,
    temp_scale: f32,
}

impl DeviceClient {
    pub fn read_temperature(&self) -> Result<f32, Box<dyn std::error::Error>> {
        let registers = read_holding_registers(1, 1)
            .map_err(|e| format!("Failed to read register: {}", e))?;
        
        let scaled = ScaledValue::from_raw(registers[0], self.temp_scale)?;
        let temp = scaled.engineering_value();
        
        // Sanity check
        if temp < -50.0 || temp > 150.0 {
            return Err(format!("Temperature out of valid range: {}", temp).into());
        }
        
        Ok(temp)
    }
    
    pub fn set_temperature(&self, temp: f32) -> Result<(), Box<dyn std::error::Error>> {
        // Validate input
        if temp < -50.0 || temp > 150.0 {
            return Err(format!("Temperature {} outside valid range [-50, 150]", temp).into());
        }
        
        let scaled = ScaledValue::from_engineering(temp, self.temp_scale)?;
        let registers = scaled.to_registers(WordOrder::BigEndian);
        
        write_single_register(1, registers[0])
            .map_err(|e| format!("Failed to write register: {}", e).into())
    }
}
```

### 5. Test with Real Device Data

```rust
#[cfg(test)]
mod integration_tests {
    use super::*;
    
    #[test]
    fn test_with_real_device_data() {
        // Captured from actual device
        let real_registers = vec![
            0x0001,  // Mode
            0x09C4,  // Temperature (25.0°C with scale 100)
            0x0000, 0x0708,  // Motor speed (1800 RPM, big-endian)
            0x4148, 0x0000,  // Current (12.5A, IEEE 754)
            0x0011,  // Status (running + remote)
            0x4142, 0x432D, 0x3132, 0x3300,  // "ABC-123" + null
        ];
        
        let config = DeviceConfig::from_registers(
            &real_registers,
            WordOrder::BigEndian,
            100.0,
        ).unwrap();
        
        assert_eq!(config.mode, 1);
        assert!((config.temperature_setpoint - 25.0).abs() < 0.1);
        assert_eq!(config.motor_speed, 1800);
        assert!((config.current - 12.5).abs() < 0.1);
        assert_eq!(config.device_name, "ABC-123");
    }
}
```

## Common Pitfalls and Solutions

### Pitfall 1: Wrong Word Order Assumption

**Problem:**
```rust
// Assumes all devices are big-endian
let value = u32::from_registers(&regs, WordOrder::BigEndian);
```

**Solution:**
```rust
// Use device-specific configuration
let value = u32::from_registers(&regs, device_profile.word_order);
```

### Pitfall 2: Ignoring Scale Factors

**Problem:**
```rust
// Treats raw value as engineering value
let temp = registers[0] as f32;  // 2550 → 2550.0°C (WRONG!)
```

**Solution:**
```rust
// Apply scale factor
let scaled = ScaledValue::from_raw(registers[0], 100.0)?;
let temp = scaled.engineering_value();  // 2550 → 25.5°C (CORRECT)
```

### Pitfall 3: Not Validating Input

**Problem:**
```rust
// No validation - crashes on invalid data
let value = u32::from_registers(&regs, word_order);
```

**Solution:**
```rust
// Validate and handle errors
let value = u32::from_registers(&regs, word_order)
    .map_err(|e| format!("Conversion failed: {}", e))?;
```

### Pitfall 4: Incorrect Float Handling

**Problem:**
```rust
// Manual float conversion (error-prone)
let bytes = [...];
let float = unsafe { std::mem::transmute::<[u8; 4], f32>(bytes) };
```

**Solution:**
```rust
// Use from_bits for safe conversion
let u32_value = u32::from_registers(&regs, word_order)?;
let float = f32::from_bits(u32_value);
```

### Pitfall 5: String Encoding Issues

**Problem:**
```rust
// Assumes UTF-8, panics on invalid data
let s = String::from_utf8(bytes).unwrap();
```

**Solution:**
```rust
// Handle invalid UTF-8 gracefully
let s = String::from_utf8(bytes)
    .unwrap_or_else(|_| String::from_utf8_lossy(&bytes).to_string());
```

## Performance Considerations

### Batch Conversions

When reading many values, batch the register reads:

```rust
// BAD: Multiple individual reads
let mode = read_holding_registers(0, 1)?;
let temp = read_holding_registers(1, 1)?;
let speed = read_holding_registers(2, 2)?;

// GOOD: Single read for all registers
let all_regs = read_holding_registers(0, 4)?;
let mode = u16::from_registers(&all_regs[0..1], word_order)?;
let temp_scaled = ScaledValue::from_raw(all_regs[1], 100.0)?;
let speed = u32::from_registers(&all_regs[2..4], word_order)?;
```

### Zero-Copy When Possible

```rust
// If you need the same registers multiple times, avoid re-reading
let registers = read_holding_registers(0, 11)?;

// Parse different fields from same buffer
let mode = u16::from_registers(&registers[0..1], word_order)?;
let temp = ScaledValue::from_raw(registers[1], 100.0)?;
let speed = u32::from_registers(&registers[2..4], word_order)?;
// ... etc
```

## Summary

Converting between MODBUS registers and program variables requires:

1. **Type-safe abstractions** - Define traits/functions for each type
2. **Explicit word order** - Never assume, always specify
3. **Error handling** - Use Result types, validate inputs
4. **Endianness awareness** - Bytes always big-endian, words device-specific
5. **Scale factor handling** - Many devices use scaled integers
6. **String encoding** - Handle ASCII/UTF-8 properly
7. **Float conversion** - Use from_bits/to_bits for IEEE 754
8. **Testing** - Test with real device data
9. **Documentation** - Document device-specific mappings

The complete Rust implementation provided in this guide handles all these concerns with type safety and proper error handling.

## Related Pages

- [MODBUS Register Data Representation](/wiki/answers/modbus-register-data-representation.md) - Register size, endianness, and data types
- [MODBUS Data Type Mapping](/wiki/concepts/modbus-data-type-mapping.md) - Detailed mapping patterns and examples
- [Holding Registers](/wiki/concepts/holding-registers.md) - 16-bit read/write registers
- [Input Registers](/wiki/concepts/input-registers.md) - 16-bit read-only registers
- [MODBUS Data Validation](/wiki/answers/modbus-data-validation.md) - Validating register values

## Backlinks

No backlinks (yet)

------------

MODBUS is a trademark of the Modbus Organization, Inc.

All information on this page is derived by summarizing and analyzing solely the MODBUS specifications published by the Modbus Organization, Inc.  The specifications are copyright by the Modbus Organization.

By deriving this information solely from the specifications we hope to stay true to the specification.

No infringement is intended.

None yet.
