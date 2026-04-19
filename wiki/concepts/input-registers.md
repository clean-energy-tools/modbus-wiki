---
title: Input Registers
Summary: 16-bit read-only data objects in MODBUS used for analog measurements and sensor readings.
Sources:
  - raw/MODBUS/MODBUS.md
  - raw/MODBUS/modbusprotocolspecification.md
Categories:
  - data-model
  - word-access
  - measurements
type: concept
date-created: 2026-04-18T12:00:00+03:00
last-updated: 2026-04-18T14:43:24+03:00
---

Input Registers are 16-bit read-only data objects in the MODBUS data model, typically used for analog measurements, sensor readings, and other analog input data (source: [modbusprotocolspecification.md](/raw/MODBUS/modbusprotocolspecification.md)).

## Overview

Input Registers represent 16-bit word data that can only be read (not written) by MODBUS clients. They are one of four primary data tables defined by the MODBUS protocol (source: [modbusprotocolspecification.md](/raw/MODBUS/modbusprotocolspecification.md)):

| Table | Object Type | Size | Access | Typical Use |
|-------|-------------|------|--------|-------------|
| Input Registers | Word | 16 bits | Read-only | Measurements, status |

## Input Register Properties

| Property | Value |
|----------|-------|
| Data size | 16 bits (2 bytes) |
| Access type | Read-only |
| Byte order | Big-endian (MSB first) |
| Typical usage | Analog inputs, sensor values |
| Address range | 0-65535 (PDU address) |
| Legacy notation | 3xxxx (e.g., input register 1 = 30001) |
| Value range | 0-65535 (unsigned) or -32768 to 32767 (signed) |

## Addressing

**PDU addresses are 0-based**, while documentation often uses 1-based numbering:

| Documentation | PDU Address | Wire Value |
|---------------|---------------|------------|
| Input Register 1 | 0 | 0x0000 |
| Input Register 100 | 99 | 0x0063 |
| Input Register 40001 | 0 | 0x0000 (Input Register context) |

**Legacy notation:** Input Registers are often referenced using the 3xxxx convention:
- 30001 = Input Register 1
- 30100 = Input Register 100
- 30001 = PDU address 0

For implementation, always use 0-based addresses in PDUs.

## Function Code for Input Registers

### Function 0x04: Read Input Registers

Read one or more input register values from a remote device.

**Request:**
```
[Function Code: 0x04][Starting Address: 2][Quantity of Registers: 2]
```

**Maximum quantity:** 125 registers

**Response:**
```
[Function Code: 0x04][Byte Count: 1][Register Values: 2N bytes]
```

**Byte Count:** 2 × Quantity of Registers

**Data encoding:** Each register is 2 bytes, big-endian (MSB first)

**Example:** Read 3 input registers starting at address 100

**Request:**
```
04 00 64 00 03
|  |     |
|  |     +-- Quantity: 3 registers
|  +-------- Start address: 100 (0x0064)
+----------- Function code: 0x04
```

**Response (values: 1000, 2000, 3000):**
```
04 06 03 E8 07 D0 0B B8
|  |  |     |     |
|  |  |     |     +-- Register 102: 0x0BB8 (3000)
|  |  |     +-------- Register 101: 0x07D0 (2000)
|  |  +-------------- Register 100: 0x03E8 (1000)
|  +----------------- Byte count: 6 bytes
+-------------------- Function code: 0x04
```

**Decoding:**
- Register 100: Bytes 0x03 0xE8 → 0x03E8 = 1000 (decimal)
- Register 101: Bytes 0x07 0xD0 → 0x07D0 = 2000 (decimal)
- Register 102: Bytes 0x0B 0xB8 → 0x0BB8 = 3000 (decimal)

## Typical Applications

Input Registers are commonly used for:
- Temperature measurements
- Pressure readings
- Flow rates
- Level measurements (tank levels)
- Voltage and current readings
- Frequency measurements
- Speed measurements (RPM)
- Energy readings (kWh)
- Power readings (kW, VA, VAR)
- Process variables (pH, conductivity)
- Position feedback (encoder counts)
- Humidity readings

## Data Representation

### Unsigned Integer (0-65535)
Most common representation for raw sensor values.

Example: 0x03E8 = 1000

### Signed Integer (-32768 to 32767)
Two's complement representation.

Example: 0xF000 = -4096 (signed), 61440 (unsigned)

### Scaled Values
Often scaled to represent engineering units:
- 0-10000 represents 0-100.0°C (scale factor 100)
- 0-1000 represents 0-100.0% (scale factor 10)

**Scaling formula:**
```
engineering_value = raw_value / scale_factor
```

Example: Raw value 2500, scale factor 100 → 25.0°C

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

### Reading Input Registers

**Address conversion:**
```
Documentation input register 100 → PDU address 99
Documentation input register 1 → PDU address 0
```

**Decoding single register:**
```c
// Decode 16-bit big-endian value
uint16_t decode_register(uint8_t *data, int offset) {
    return (data[offset] << 8) | data[offset + 1];
}

// Example: decode register at byte offset 2
uint16_t value = decode_register(response, 2);
```

**Decoding signed register:**
```c
// Decode 16-bit signed big-endian value
int16_t decode_signed_register(uint8_t *data, int offset) {
    return (int16_t)((data[offset] << 8) | data[offset + 1]);
}
```

**Reading multiple registers:**
```c
// Read array of input registers
void read_input_registers(uint8_t *response, int count, uint16_t *values) {
    for (int i = 0; i < count; i++) {
        int byte_offset = 2 + (i * 2);  // Skip function code and byte count
        values[i] = (response[byte_offset] << 8) | response[byte_offset + 1];
    }
}
```

**Decoding 32-bit values:**
```c
// Decode 32-bit big-endian value (2 registers)
uint32_t decode_32bit_value(uint8_t *data, int offset) {
    return (data[offset] << 24) | (data[offset + 1] << 16) |
           (data[offset + 2] << 8) | data[offset + 3];
}

// Little-endian 32-bit (word-swapped)
uint32_t decode_32bit_le(uint8_t *data, int offset) {
    return (data[offset + 2] << 24) | (data[offset + 3] << 16) |
           (data[offset] << 8) | data[offset + 1];
}
```

### Scaling to Engineering Units

```c
// Scale raw value to engineering units
float scale_to_units(uint16_t raw_value, float scale_factor) {
    return (float)raw_value / scale_factor;
}

// Example: Scale to temperature in °C
float temperature = scale_to_units(raw_temp, 100.0);  // Scale factor = 100

// Example: Scale to percentage
float percentage = scale_to_units(raw_pct, 10.0);   // Scale factor = 10
```

## Comparison with Holding Registers

| Characteristic | Input Registers | Holding Registers |
|---------------|-----------------|------------------|
| Access type | Read-only | Read/Write |
| Typical use | Measurements, sensor values | Configuration, setpoints |
| Direction of data | Device → Client | Bidirectional |
| Function code | 0x04 (Read) | 0x03 (Read), 0x06 (Write), 0x10 (Write Multiple) |
| Legacy notation | 3xxxx | 4xxxx |
| Example | 30001 = Input Register 1 | 40001 = Holding Register 1 |

## Error Handling

Common exception codes for input register operations:
- **0x01:** ILLEGAL FUNCTION - Device doesn't support function 0x04
- **0x02:** ILLEGAL DATA ADDRESS - Register address out of range
- **0x03:** ILLEGAL DATA VALUE - Invalid quantity (must be 1-125)

## Best Practices

### Polling Strategy
- Poll input registers at appropriate rate for process dynamics
- Consider using change detection for slowly changing values
- Don't poll faster than sensor update rate
- Group related registers in single request when possible

### Scaling and Units
- Always consult device documentation for scaling factors
- Document scaling factors in your application
- Handle both signed and unsigned values correctly
- Be aware of multi-register values and ordering

### Error Handling
- Implement retry logic for failed reads
- Validate returned data (check for out-of-range values)
- Log exceptions for troubleshooting
- Implement timeouts to prevent hanging

## Related Data Objects

Compare with other MODBUS data objects:

| Object | Size | Access | Typical Use |
|--------|------|--------|-------------|
| Input Registers | 16 bits | Read-only | Measurements, status |
| Holding Registers | 16 bits | Read/Write | Configuration, setpoints |
| Discrete Inputs | 1 bit | Read-only | Digital inputs, switches |
| Coils | 1 bit | Read/Write | Digital outputs, relays |

## Related pages

- [holding-registers](/wiki/concepts/holding-registers.md)
- [coils](/wiki/concepts/coils.md)
- [discrete-inputs](/wiki/concepts/discrete-inputs.md)
- [function-codes](/wiki/concepts/function-codes.md)
