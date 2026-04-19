---
title: Holding Registers
Summary: 16-bit read-write data objects in MODBUS used for configuration, setpoints, and control values.
Sources:
  - raw/MODBUS/MODBUS.md
  - raw/MODBUS/modbusprotocolspecification.md
Categories:
  - data-model
  - word-access
  - configuration
type: concept
date-created: 2026-04-18T12:00:00+03:00
last-updated: 2026-04-18T14:43:24+03:00
---

Holding Registers are 16-bit read-write data objects in the MODBUS data model, typically used for configuration parameters, setpoints, and control values (source: [modbusprotocolspecification.md](/raw/MODBUS/modbusprotocolspecification.md)).

## Overview

Holding Registers represent 16-bit word data that can be both read and written by MODBUS clients. They are one of four primary data tables defined by the MODBUS protocol (source: [modbusprotocolspecification.md](/raw/MODBUS/modbusprotocolspecification.md)):

| Table | Object Type | Size | Access | Typical Use |
|-------|-------------|------|--------|-------------|
| Holding Registers | Word | 16 bits | Read/Write | Configuration, setpoints |

## Holding Register Properties

| Property | Value |
|----------|-------|
| Data size | 16 bits (2 bytes) |
| Access type | Read/Write |
| Byte order | Big-endian (MSB first) |
| Typical usage | Configuration, setpoints, control values |
| Address range | 0-65535 (PDU address) |
| Legacy notation | 4xxxx (e.g., holding register 1 = 40001) |
| Value range | 0-65535 (unsigned) or -32768 to 32767 (signed) |

## Addressing

**PDU addresses are 0-based**, while documentation often uses 1-based numbering:

| Documentation | PDU Address | Wire Value |
|---------------|---------------|------------|
| Holding Register 1 | 0 | 0x0000 |
| Holding Register 100 | 99 | 0x0063 |
| Holding Register 40001 | 0 | 0x0000 (Holding Register context) |

**Legacy notation:** Holding Registers are often referenced using the 4xxxx convention:
- 40001 = Holding Register 1
- 40100 = Holding Register 100
- 40001 = PDU address 0

For implementation, always use 0-based addresses in PDUs.

## Function Codes for Holding Registers

### Function 0x03: Read Holding Registers

Read one or more holding register values from a remote device.

**Request:**
```
[Function Code: 0x03][Starting Address: 2][Quantity of Registers: 2]
```

**Maximum quantity:** 125 registers

**Response:**
```
[Function Code: 0x03][Byte Count: 1][Register Values: 2N bytes]
```

**Byte Count:** 2 × Quantity of Registers

**Data encoding:** Each register is 2 bytes, big-endian (MSB first)

**Example:** Read 2 holding registers starting at address 0

**Request:**
```
03 00 00 00 02
|  |     |
|  |     +-- Quantity: 2 registers
|  +-------- Start address: 0
+----------- Function code: 0x03
```

**Response (values: 10, 20):**
```
03 04 00 0A 00 14
|  |  |     |
|  |  |     +-- Register 1: 0x0014 (20)
|  |  +-------- Register 0: 0x000A (10)
|  +----------------- Byte count: 4 bytes
+-------------------- Function code: 0x03
```

### Function 0x06: Write Single Register

Write a single holding register value.

**Request:**
```
[Function Code: 0x06][Register Address: 2][Register Value: 2]
```

**Response (5 bytes):** Echo of request (confirms write).

**Example:** Write value 0x0003 to register 1
```
Request:  06 00 01 00 03
Response: 06 00 01 00 03
```

### Function 0x10: Write Multiple Registers

Write multiple holding register values.

**Request (6 + 2N bytes):**
```
[Function Code: 0x10][Starting Address: 2][Quantity of Registers: 2][Byte Count: 1][Register Values: 2N]
```

**Maximum quantity:** 123 registers

**Response (5 bytes):**
```
[Function Code: 0x10][Starting Address: 2][Quantity of Registers: 2]
```

**Example:** Write 2 registers (values 0x000A, 0x0102)
```
Request:  10 00 01 00 02 04 00 0A 01 02
Response: 10 00 01 00 02
```

### Function 0x16: Mask Write Register

Modify specific bits within a register using AND/OR masks.

**Request (7 bytes):**
```
[Function Code: 0x16][Reference Address: 2][AND_Mask: 2][OR_Mask: 2]
```

**Operation:**
```
New value = (Current value AND AND_Mask) OR (OR_Mask AND ~AND_Mask)
```

**Response (7 bytes):** Echo of request.

**Example:** Set bit 3 to 1, clear bit 2
```
Current value:  0b0000 0000
AND_Mask:       0b1111 1011  (keep all except bit 2)
OR_Mask:        0b0000 1000  (set bit 3)

Operation: (0x00 AND 0xFB) OR (0x08 AND ~0xFB)
         = 0x00 OR 0x08
         = 0x08 (binary: 0000 1000)
```

### Function 0x17: Read/Write Multiple Registers

Perform a combined read and write operation.

**Request (10 + 2W bytes):**
```
[Function Code: 0x17][Read Starting Address: 2][Quantity to Read: 2][Write Starting Address: 2][Quantity to Write: 2][Write Byte Count: 1][Write Register Values: 2W]
```

**Max quantities:** Read: 125, Write: 121

**Response (3 + 2R bytes):**
```
[Function Code: 0x17][Byte Count: 1][Read Register Values: 2R]
```

## Typical Applications

Holding Registers are commonly used for:
- Configuration parameters
- Setpoints (temperature setpoint, pressure setpoint)
- Control values (speed reference, position reference)
- Calibration values
- Mode selection
- Timer values
- Threshold values (alarm limits)
- PID parameters (Kp, Ki, Kd)
- User settings
- Device configuration
- Command registers

## Data Representation

### Unsigned Integer (0-65535)
Most common representation for configuration values.

Example: 0x000A = 10

### Signed Integer (-32768 to 32767)
Two's complement representation for values that can be negative.

Example: 0xF000 = -4096 (signed), 61440 (unsigned)

### Scaled Values
Often scaled to represent engineering units:
- 0-10000 represents 0-100.0°C (scale factor 100)
- 0-1000 represents 0-100.0% (scale factor 10)

**Scaling formula:**
```
engineering_value = raw_value / scale_factor
raw_value = engineering_value × scale_factor
```

Example: Setpoint 25.5°C, scale factor 100 → 2550

### Multi-Register Values

Some values require multiple registers:
- **32-bit integers:** 2 registers
- **32-bit floating point:** 2 registers (IEEE 754)
- **64-bit integers:** 4 registers

**32-bit value register order:**
- **Big-endian (default):** High word first → Register N = MSW, Register N+1 = LSW
- **Little-endian (some devices):** Low word first → Register N = LSW, Register N+1 = MSW

**Always verify device documentation for multi-register ordering.**

## Implementation Considerations

### Reading Holding Registers

**Address conversion:**
```
Documentation holding register 100 → PDU address 99
Documentation holding register 1 → PDU address 0
```

**Decoding single register:**
```c
// Decode 16-bit big-endian value
uint16_t decode_register(uint8_t *data, int offset) {
    return (data[offset] << 8) | data[offset + 1];
}
```

**Reading multiple registers:**
```c
// Read array of holding registers
void read_holding_registers(uint8_t *response, int count, uint16_t *values) {
    for (int i = 0; i < count; i++) {
        int byte_offset = 2 + (i * 2);  // Skip function code and byte count
        values[i] = (response[byte_offset] << 8) | response[byte_offset + 1];
    }
}
```

### Writing Holding Registers

**Encoding single register:**
```c
// Encode 16-bit big-endian value
void encode_register(uint8_t *data, int offset, uint16_t value) {
    data[offset] = (value >> 8) & 0xFF;
    data[offset + 1] = value & 0xFF;
}
```

**Writing multiple registers:**
```c
// Encode array of holding registers
void write_registers(uint8_t *request, int start_addr, int count, uint16_t *values) {
    request[0] = 0x10;  // Function code
    request[1] = (start_addr >> 8) & 0xFF;
    request[2] = start_addr & 0xFF;
    request[3] = (count >> 8) & 0xFF;
    request[4] = count & 0xFF;
    request[5] = count * 2;  // Byte count

    for (int i = 0; i < count; i++) {
        int byte_offset = 6 + (i * 2);
        request[byte_offset] = (values[i] >> 8) & 0xFF;
        request[byte_offset + 1] = values[i] & 0xFF;
    }
}
```

**Mask Write Register:**
```c
// Build mask write request
void build_mask_write(uint8_t *request, uint16_t addr,
                      uint16_t and_mask, uint16_t or_mask) {
    request[0] = 0x16;  // Function code
    request[1] = (addr >> 8) & 0xFF;
    request[2] = addr & 0xFF;
    request[3] = (and_mask >> 8) & 0xFF;
    request[4] = and_mask & 0xFF;
    request[5] = (or_mask >> 8) & 0xFF;
    request[6] = or_mask & 0xFF;
}
```

### Scaling to Engineering Units

```c
// Scale engineering units to raw value
uint16_t scale_from_units(float engineering_value, float scale_factor) {
    return (uint16_t)(engineering_value * scale_factor);
}

// Scale raw value to engineering units
float scale_to_units(uint16_t raw_value, float scale_factor) {
    return (float)raw_value / scale_factor;
}

// Example: Set temperature setpoint to 25.5°C
float setpoint = 25.5f;
uint16_t raw_value = scale_from_units(setpoint, 100.0);  // = 2550
```

## Comparison with Input Registers

| Characteristic | Holding Registers | Input Registers |
|---------------|------------------|-----------------|
| Access type | Read/Write | Read-only |
| Typical use | Configuration, setpoints | Measurements, sensor values |
| Direction of data | Bidirectional | Device → Client |
| Function codes | 0x03, 0x06, 0x10, 0x16, 0x17 | 0x04 |
| Legacy notation | 4xxxx | 3xxxx |
| Example | 40001 = Holding Register 1 | 30001 = Input Register 1 |

## Error Handling

Common exception codes for holding register operations:
- **0x01:** ILLEGAL FUNCTION - Device doesn't support function
- **0x02:** ILLEGAL DATA ADDRESS - Register address out of range
- **0x03:** ILLEGAL DATA VALUE - Invalid value or quantity

## Best Practices

### Writing Values
- Read-Modify-Write pattern for partial changes
- Use Mask Write (0x16) for bit-level modifications
- Validate write values before sending
- Handle write failures gracefully

### Configuration Management
- Document all register addresses and meanings
- Use descriptive variable names for registers
- Implement configuration validation
- Store current configuration for recovery

### Read/Write Strategies
- Group related registers in single operation
- Use Read/Write Multiple (0x17) for atomic read-then-write
- Minimize write operations to reduce wear (if applicable)
- Use appropriate polling rates

## Related Data Objects

Compare with other MODBUS data objects:

| Object | Size | Access | Typical Use |
|--------|------|--------|-------------|
| Holding Registers | 16 bits | Read/Write | Configuration, setpoints |
| Input Registers | 16 bits | Read-only | Measurements, status |
| Coils | 1 bit | Read/Write | Digital outputs, relays |
| Discrete Inputs | 1 bit | Read-only | Digital inputs, switches |

## Related pages

- [input-registers](/wiki/concepts/input-registers.md)
- [coils](/wiki/concepts/coils.md)
- [discrete-inputs](/wiki/concepts/discrete-inputs.md)
- [function-codes](/wiki/concepts/function-codes.md)
