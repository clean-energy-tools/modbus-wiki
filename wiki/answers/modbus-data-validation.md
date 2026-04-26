---
title: MODBUS Data Validation
Summary: Comprehensive guide to validating MODBUS register values when reading from and writing to devices, including protocol-level validation, data type checking, and best practices.
Sources:
  - raw/MODBUS/modbusprotocolspecification.md
  - raw/MODBUS/MODBUS.md
  - raw/MODBUS/messagingimplementationguide.md
Categories:
  - validation
  - data-mapping
  - implementation
  - error-handling
type: answer
date-created: 2026-04-19T12:00:00+03:00
last-updated: 2026-04-19T12:00:00+03:00
---

MODBUS data validation is critical for reliable communication with industrial devices. Validation occurs at three levels: protocol-level (enforced by the device), data type validation (when mapping registers to program variables), and output validation (when sending values to devices).

## Protocol-Level Validation

MODBUS devices enforce specific validation rules and will reject invalid requests with exception codes.

### Exception Code 0x03: ILLEGAL DATA VALUE

The device returns exception code 0x03 when:
- Invalid quantity or value in request (source: [function-codes](/wiki/concepts/function-codes.md:280))
- Coil values are not exactly 0xFF00 (ON) or 0x0000 (OFF) (source: [coils](/wiki/concepts/coils.md:106))
- Quantity exceeds protocol limits

### Quantity Limits

MODBUS defines strict limits on data transfer sizes (source: [function-codes](/wiki/concepts/function-codes.md:260-269)):

| Operation | Function Code | Maximum Quantity |
|-----------|---------------|------------------|
| Read Coils | 0x01 | 2000 bits |
| Read Discrete Inputs | 0x02 | 2000 bits |
| Read Holding Registers | 0x03 | 125 registers |
| Read Input Registers | 0x04 | 125 registers |
| Write Multiple Coils | 0x0F | 1968 bits |
| Write Multiple Registers | 0x10 | 123 registers |
| Read/Write Multiple Registers | 0x17 | Read: 125, Write: 121 |

### Address Range Validation

Exception code 0x02 (ILLEGAL DATA ADDRESS) is returned when:
- Address is outside the device's valid range (source: [function-codes](/wiki/concepts/function-codes.md:279))
- Address does not exist in device's memory map
- Address is not accessible (e.g., read-only address for write operation)

## Data Type Validation: Registers to Program Variables

When reading MODBUS registers and converting them to program variables, validate the following:

### Range Validation

**16-bit unsigned (u16):**
- Valid range: 0-65535 (source: [holding-registers](/wiki/concepts/holding-registers.md:35))
- All values are technically valid, but device-specific limits may apply

**16-bit signed (i16):**
- Valid range: -32768 to 32767 (source: [holding-registers](/wiki/concepts/holding-registers.md:35))
- Two's complement representation

### Register Count Validation

Ensure sufficient registers before parsing multi-register values (source: [implementation](/wiki/concepts/implementation.md:262)):

```rust
pub fn validate_register_count(registers: &[u16], expected: usize) -> Result<(), ValidationError> {
    if registers.len() < expected {
        Err(ValidationError::NotEnoughRegisters {
            expected,
            actual: registers.len(),
        })
    } else {
        Ok(())
    }
}
```

**Multi-register type requirements:**
- 32-bit types (u32, i32, f32): 2 registers
- 64-bit types (u64, i64, f64): 4 registers

### Scale Factor Validation

Validate scale factors before converting scaled values (source: [implementation](/wiki/concepts/implementation.md:185)):

```rust
pub fn validate_scale_factor(scale: f32) -> Result<(), ValidationError> {
    if scale <= 0.0 {
        return Err(ValidationError::InvalidScaleFactor);
    }
    Ok(())
}

pub fn validate_scaled_range(engineering_value: f32, scale: f32) -> Result<u16, ValidationError> {
    validate_scale_factor(scale)?;
    let raw = (engineering_value * scale).round();
    if raw < 0.0 || raw > 65535.0 {
        return Err(ValidationError::ValueOutOfRange { value: engineering_value, min: 0.0, max: 65535.0 / scale });
    }
    Ok(raw as u16)
}
```

### String Encoding Validation

Validate string data before conversion (source: [implementation](/wiki/concepts/implementation.md:186)):

```rust
pub fn validate_string_bytes(bytes: &[u8], encoding: &str) -> Result<(), ValidationError> {
    match encoding {
        "ASCII" => {
            // All bytes must be 0-127
            if bytes.iter().any(|&b| b > 127) {
                return Err(ValidationError::InvalidStringEncoding);
            }
        }
        "UTF-8" => {
            // Validate UTF-8 sequence
            if std::str::from_utf8(bytes).is_err() {
                return Err(ValidationError::InvalidStringEncoding);
            }
        }
        _ => return Err(ValidationError::UnknownEncoding(encoding.to_string())),
    }
    Ok(())
}
```

### Floating Point Validation

Validate IEEE 754 floating point values:

```rust
pub fn validate_f32(value: f32) -> Result<(), ValidationError> {
    if value.is_nan() {
        return Err(ValidationError::NaN);
    }
    if value.is_infinite() {
        return Err(ValidationError::Infinity);
    }
    Ok(())
}
```

### Bit Field Validation

Validate bit field structures:

```rust
pub fn validate_bit_field(value: u16, mask: u16) -> Result<(), ValidationError> {
    // Check that reserved bits are not set
    if (value & mask) != 0 {
        return Err(ValidationError::ReservedBitsSet);
    }
    Ok(())
}

// Example: Validate device status with reserved bits
pub fn validate_device_status(status: u16) -> Result<(), ValidationError> {
    // Bits 5-15 are reserved (11 bits)
    validate_bit_field(status, 0xFFE0)?;
    Ok(())
}
```

## Data Type Validation: Program Variables to Registers

When converting program variables to MODBUS register values, validate and clamp as needed.

### Range Enforcement

Clamp values to valid ranges before conversion (source: [implementation](/wiki/concepts/implementation.md:190)):

```rust
pub fn clamp_u16(value: i32) -> u16 {
    value.max(0).min(65535) as u16
}

pub fn clamp_i16(value: i32) -> i16 {
    value.max(-32768).min(32767) as i16
}
```

### Scaled Value Validation

Validate and convert engineering units to raw register values:

```rust
pub struct ScaledValue {
    pub raw: u16,
    pub scale_factor: f32,
}

impl ScaledValue {
    pub fn from_engineering(eng: f32, scale: f32) -> Result<Self, ValidationError> {
        validate_scale_factor(scale)?;
        let raw = validate_scaled_range(eng, scale)?;
        Ok(Self { raw, scale_factor: scale })
    }

    pub fn engineering_value(&self) -> f32 {
        self.raw as f32 / self.scale_factor
    }
}

// Usage
let temp_setpoint = ScaledValue::from_engineering(25.5, 100.0)?;  // 2550
let too_high = ScaledValue::from_engineering(1000.0, 100.0);  // Error: out of range
```

### Coil Value Validation

Validate coil values before writing (source: [coils](/wiki/concepts/coils.md:102-106)):

```rust
pub fn validate_coil_value(value: u16) -> Result<(), ValidationError> {
    match value {
        0x0000 | 0xFF00 => Ok(()),
        _ => Err(ValidationError::InvalidCoilValue(value)),
    }
}

pub fn bool_to_coil(on: bool) -> u16 {
    if on { 0xFF00 } else { 0x0000 }
}
```

### Multi-Register Value Validation

Validate multi-register values before conversion:

```rust
pub fn validate_u32(value: u32) -> Result<(), ValidationError> {
    // u32 always fits in 2 registers, but check for device-specific limits
    Ok(())
}

pub fn validate_u64(value: u64) -> Result<(), ValidationError> {
    // u64 always fits in 4 registers
    Ok(())
}

// Example with device-specific limit
pub fn validate_speed_rpm(speed: u32) -> Result<(), ValidationError> {
    const MAX_SPEED: u32 = 10000;
    if speed > MAX_SPEED {
        return Err(ValidationError::ValueOutOfRange {
            value: speed as f32,
            min: 0.0,
            max: MAX_SPEED as f32,
        });
    }
    Ok(())
}
```

## Common Validation Patterns

### Read-Modify-Write with Validation

```rust
pub fn update_register_with_validation<F>(
    client: &mut ModbusClient,
    address: u16,
    modifier: F,
) -> Result<(), ModbusError>
where
    F: FnOnce(u16) -> Result<u16, ValidationError>,
{
    // Read current value
    let current = client.read_holding_registers(address, 1)?[0];

    // Apply modification with validation
    let new_value = modifier(current)?;

    // Write new value
    client.write_single_register(address, new_value)?;

    Ok(())
}

// Usage: Clamp value to valid range
update_register_with_validation(&mut client, 100, |current| {
    Ok(clamp_u16(current as i32 + 100))
})?;
```

### Batch Validation

Validate entire batch before sending:

```rust
pub struct RegisterWrite {
    pub address: u16,
    pub value: u16,
}

pub fn validate_batch_writes(writes: &[RegisterWrite]) -> Result<(), ValidationError> {
    // Check for overlapping writes
    let mut addresses: Vec<(u16, u16)> = writes.iter()
        .map(|w| (w.address, w.address))
        .collect();
    addresses.sort_by_key(|a| a.0);

    for window in addresses.windows(2) {
        if window[0].1 >= window[1].0 {
            return Err(ValidationError::OverlappingWrites);
        }
    }

    // Check quantity limits
    if writes.len() > 123 {
        return Err(ValidationError::ExceedsLimit(123));
    }

    Ok(())
}
```

### Type-Safe Register Access

```rust
pub struct ValidatedRegister<T> {
    pub address: u16,
    _phantom: std::marker::PhantomData<T>,
}

impl ValidatedRegister<u16> {
    pub const MODE: Self = Self { address: 0, _phantom: std::marker::PhantomData };
    pub const SPEED: Self = Self { address: 2, _phantom: std::marker::PhantomData };
}

impl ValidatedRegister<f32> {
    pub const TEMPERATURE: Self = Self { address: 1, _phantom: std::marker::PhantomData };
}

pub struct DeviceClient {
    word_order: WordOrder,
}

impl DeviceClient {
    pub fn read_validated<T: ModbusMapping<T>>(
        &self,
        reg: ValidatedRegister<T>,
    ) -> Result<T, ModbusError> {
        let count = T::register_count();
        let registers = self.read_holding_registers(reg.address, count)?;
        Ok(T::from_registers(&registers, self.word_order))
    }

    pub fn write_validated<T: ModbusMapping<T>>(
        &mut self,
        reg: ValidatedRegister<T>,
        value: &T,
    ) -> Result<(), ModbusError> {
        let registers = T::to_registers(value, self.word_order);
        self.write_multiple_registers(reg.address, &registers)?;
        Ok(())
    }
}
```

## Error Handling Best Practices

### Define Comprehensive Error Types

```rust
#[derive(Debug)]
pub enum ValidationError {
    NotEnoughRegisters { expected: usize, actual: usize },
    InvalidScaleFactor,
    ValueOutOfRange { value: f32, min: f32, max: f32 },
    InvalidStringEncoding,
    InvalidCoilValue(u16),
    ReservedBitsSet,
    NaN,
    Infinity,
    UnknownEncoding(String),
    OverlappingWrites,
    ExceedsLimit(usize),
}
```

### Handle Protocol Exceptions

```rust
pub fn handle_modbus_exception(code: u8) -> Result<(), ModbusError> {
    match code {
        0x01 => Err(ModbusError::IllegalFunction),
        0x02 => Err(ModbusError::IllegalDataAddress),
        0x03 => Err(ModbusError::IllegalDataValue),
        0x04 => Err(ModbusError::DeviceFailure),
        0x05 => Err(ModbusError::Acknowledge),
        0x06 => Err(ModbusError::DeviceBusy),
        _ => Err(ModbusError::UnknownException(code)),
    }
}
```

### Validation Before Network Operations

Always validate locally before sending to network:

```rust
pub fn safe_write_register(
    client: &mut ModbusClient,
    address: u16,
    value: u16,
    max_value: u16,
) -> Result<(), ModbusError> {
    // Local validation first
    if value > max_value {
        return Err(ModbusError::Validation(ValidationError::ValueOutOfRange {
            value: value as f32,
            min: 0.0,
            max: max_value as f32,
        }));
    }

    // Then network operation
    client.write_single_register(address, value)
}
```

## Testing Validation Logic

### Unit Tests for Validation

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_scale_factor_validation() {
        assert!(validate_scale_factor(1.0).is_ok());
        assert!(validate_scale_factor(0.0).is_err());
        assert!(validate_scale_factor(-1.0).is_err());
    }

    #[test]
    fn test_scaled_value_clamping() {
        // Valid value
        assert!(ScaledValue::from_engineering(25.5, 100.0).is_ok());

        // Out of range (too high)
        assert!(ScaledValue::from_engineering(1000.0, 100.0).is_err());

        // Out of range (negative)
        assert!(ScaledValue::from_engineering(-10.0, 100.0).is_err());
    }

    #[test]
    fn test_coil_value_validation() {
        assert!(validate_coil_value(0x0000).is_ok());
        assert!(validate_coil_value(0xFF00).is_ok());
        assert!(validate_coil_value(0x0001).is_err());
        assert!(validate_coil_value(0x0100).is_err());
    }

    #[test]
    fn test_register_count_validation() {
        assert!(validate_register_count(&[1, 2, 3], 2).is_ok());
        assert!(validate_register_count(&[1, 2, 3], 4).is_err());
    }

    #[test]
    fn test_f32_validation() {
        assert!(validate_f32(1.0).is_ok());
        assert!(validate_f32(f32::NAN).is_err());
        assert!(validate_f32(f32::INFINITY).is_err());
        assert!(validate_f32(f32::NEG_INFINITY).is_err());
    }
}
```

### Integration Testing with Real Devices

Test validation against actual device behavior:

```rust
#[test]
fn test_device_enforces_limits() {
    let mut client = connect_to_device();

    // Try to write value that exceeds device limit
    let result = client.write_single_register(0, 99999);
    assert!(matches!(result, Err(ModbusError::IllegalDataValue)));

    // Try to read from invalid address
    let result = client.read_holding_registers(9999, 1);
    assert!(matches!(result, Err(ModbusError::IllegalDataAddress)));
}
```

## Summary Checklist

When implementing MODBUS data validation, ensure you:

### Before Reading
- [ ] Validate address is within device range
- [ ] Validate quantity is within protocol limits
- [ ] Check register count matches expected type requirements
- [ ] Validate scale factors are correct
- [ ] Handle device-specific validation rules

### After Reading
- [ ] Validate register count was sufficient
- [ ] Check for NaN/infinity in floating point values
- [ ] Validate string encoding
- [ ] Check bit field reserved bits
- [ ] Verify scaled values are within expected engineering range

### Before Writing
- [ ] Clamp values to valid ranges
- [ ] Validate coil values are 0x0000 or 0xFF00
- [ ] Check scaled values fit in 16-bit range
- [ ] Validate quantity is within protocol limits
- [ ] Check for overlapping register writes
- [ ] Apply device-specific validation rules

### Error Handling
- [ ] Define comprehensive error types
- [ ] Handle all exception codes properly
- [ ] Implement retry logic for transient errors
- [ ] Log validation failures for debugging
- [ ] Provide user-friendly error messages

## Related pages

- [MODBUS Data Type Mapping](/wiki/concepts/modbus-data-type-mapping.md) - Data type conversion patterns
- [Function Codes](/wiki/concepts/function-codes.md) - Protocol function codes and limits
- [Holding Registers](/wiki/concepts/holding-registers.md) - Register properties and usage
- [Coils](/wiki/concepts/coils.md) - Coil operations and validation
- [Implementation](/wiki/concepts/implementation.md) - Implementation best practices

## Backlinks

No backlinks (yet)

------------

MODBUS is a trademark of the Modbus Organization, Inc.

All information on this page is derived by summarizing and analyzing solely the MODBUS specifications published by the Modbus Organization, Inc.  The specifications are copyright by the Modbus Organization.

By deriving this information solely from the specifications we hope to stay true to the specification.

No infringement is intended.

