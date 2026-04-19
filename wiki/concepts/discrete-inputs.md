---
title: Discrete Inputs
Summary: Single-bit read-only data objects in MODBUS used for digital inputs and limit switches.
Sources:
  - raw/MODBUS/MODBUS.md
  - raw/MODBUS/modbusprotocolspecification.md
Categories:
  - data-model
  - digital-inputs
  - bit-access
date-created: 2026-04-18T12:00:00+03:00
last-updated: 2026-04-18T14:43:24+03:00
---

Discrete Inputs are single-bit read-only data objects in the MODBUS data model, typically used for digital inputs, limit switches, and other binary status signals (source: [modbusprotocolspecification.md](/raw/MODBUS/modbusprotocolspecification.md)).

## Overview

Discrete Inputs represent binary input data that can only be read (not written) by MODBUS clients. They are one of four primary data tables defined by the MODBUS protocol (source: [modbusprotocolspecification.md](/raw/MODBUS/modbusprotocolspecification.md)):

| Table | Object Type | Size | Access | Typical Use |
|-------|-------------|------|--------|-------------|
| Discrete Inputs | Bit | 1 bit | Read-only | Digital inputs, switches |

## Discrete Input Properties

| Property | Value |
|----------|-------|
| Data size | 1 bit |
| Access type | Read-only |
| Typical usage | Digital inputs, limit switches, status indicators |
| Address range | 0-65535 (PDU address) |
| Legacy notation | 1xxxx (e.g., discrete input 1 = 10001) |

## Addressing

**PDU addresses are 0-based**, while documentation often uses 1-based numbering:

| Documentation | PDU Address | Wire Value |
|---------------|---------------|------------|
| Discrete Input 1 | 0 | 0x0000 |
| Discrete Input 100 | 99 | 0x0063 |
| Discrete Input 1000 | 999 | 0x03E7 |

**Legacy notation:** Discrete Inputs are often referenced using the 1xxxx convention:
- 10001 = Discrete Input 1
- 11000 = Discrete Input 1000
- 10001 = PDU address 0

For implementation, always use 0-based addresses in PDUs.

## Function Codes for Discrete Inputs

### Function 0x02: Read Discrete Inputs

Read status of one or more discrete inputs from a remote device.

**Request:**
```
[Function Code: 0x02][Starting Address: 2][Quantity of Inputs: 2]
```

**Maximum quantity:** 2000 discrete inputs

**Response:**
```
[Function Code: 0x02][Byte Count: 1][Input Status: N bytes]
```

**Bit packing:**
- Input at starting address is in bit 0 (LSB) of first byte
- Bits are packed sequentially
- Byte count = ⌈Quantity / 8⌉

**Example:** Read 10 discrete inputs starting at address 100

**Request:**
```
02 00 64 00 0A
|  |     |
|  |     +-- Quantity: 10 inputs
|  +-------- Start address: 100 (0x0064)
+----------- Function code: 0x02
```

**Response (assuming inputs 100-109 = ON,OFF,ON,ON,OFF,ON,OFF,ON,OFF,ON):**
```
02 02 D5 0A
|  |  |  |
|  |  +-- Byte 1: 0x0A = 00001010 (inputs 107-109)
|  +----- Byte 0: 0xD5 = 11010101 (inputs 100-106)
+---------- Byte count: 2 bytes
```

**Decoding Byte 0 (0xD5 = 11010101):**
- Input 100 (bit 0): 1 (ON)
- Input 101 (bit 1): 0 (OFF)
- Input 102 (bit 2): 1 (ON)
- Input 103 (bit 3): 1 (ON)
- Input 104 (bit 4): 0 (OFF)
- Input 105 (bit 5): 1 (ON)
- Input 106 (bit 6): 0 (OFF)
- Input 107 (bit 7): 1 (ON)

**Decoding Byte 1 (0x0A = 00001010):**
- Input 108 (bit 0): 0 (OFF)
- Input 109 (bit 1): 1 (ON)

## Typical Applications

Discrete Inputs are commonly used for:
- Digital inputs (DI) in industrial control systems
- Limit switches and proximity sensors
- Push buttons and selector switches
- Level switches (high/low level)
- Pressure switches (high/low pressure)
- Temperature switches (high/low temperature)
- Flow switches
- Photoelectric sensors
- Status indicators (machine on/off, fault detected)
- Safety interlocks

## Data Representation

**Binary values:**
- 0 = OFF / False / Open circuit
- 1 = ON / True / Closed circuit

**Common sensing methods:**
- Dry contact: Physical contact closure (open/closed)
- Optocoupler: Optically isolated digital input
- TTL/CMOS: Logic level signals (0V/5V, 0V/24V)
- Voltage sensing: Presence/absence of voltage

## Implementation Considerations

### Reading Discrete Inputs

**Address conversion:**
```
Documentation discrete input 100 → PDU address 99
Documentation discrete input 1 → PDU address 0
```

**Bit extraction from response:**
```c
// Extract single discrete input value
int get_discrete_input(uint8_t *response, int start_addr, int input_addr) {
    int offset = input_addr - start_addr;
    int byte_index = offset / 8;
    int bit_index = offset % 8;
    return (response[byte_index] >> bit_index) & 0x01;
}
```

**Reading multiple inputs:**
```c
// Read and decode array of discrete inputs
void read_discrete_inputs(uint8_t *response, int start_addr,
                         int count, uint8_t *inputs) {
    for (int i = 0; i < count; i++) {
        int byte_index = i / 8;
        int bit_index = i % 8;
        inputs[i] = (response[byte_index] >> bit_index) & 0x01;
    }
}
```

## Comparison with Coils

| Characteristic | Discrete Inputs | Coils |
|---------------|-----------------|-------|
| Access type | Read-only | Read/Write |
| Typical use | Digital inputs, sensors | Digital outputs, relays |
| Direction of data | Device → Client | Bidirectional |
| Function code | 0x02 (Read) | 0x01 (Read), 0x05 (Write), 0x0F (Write Multiple) |
| Legacy notation | 1xxxx | 0xxxx |
| Example | 10001 = Input 1 | 00001 = Coil 1 |

## Error Handling

Common exception codes for discrete input operations:
- **0x01:** ILLEGAL FUNCTION - Device doesn't support function 0x02
- **0x02:** ILLEGAL DATA ADDRESS - Input address out of range
- **0x03:** ILLEGAL DATA VALUE - Invalid quantity (must be 1-2000)

## Best Practices

### Polling Strategy
- Poll discrete inputs at appropriate rate for application
- Don't poll faster than input debounce time
- Consider change-of-state monitoring for slow-changing inputs

### Debouncing
- Mechanical switches may have contact bounce
- Implement hardware or software debouncing
- Typical debounce time: 5-50ms depending on switch type

### Signal Conditioning
- Use appropriate input voltage levels
- Implement noise filtering for long cable runs
- Consider optocouplers for electrical isolation

## Related Data Objects

Compare with other MODBUS data objects:

| Object | Size | Access | Typical Use |
|--------|------|--------|-------------|
| Discrete Inputs | 1 bit | Read-only | Digital inputs, switches |
| Coils | 1 bit | Read/Write | Digital outputs, relays |
| Input Registers | 16 bits | Read-only | Analog measurements |
| Holding Registers | 16 bits | Read/Write | Configuration, setpoints |

## Related pages

- [coils](/wiki/concepts/coils.md)
- [holding-registers](/wiki/concepts/holding-registers.md)
- [input-registers](/wiki/concepts/input-registers.md)
- [function-codes](/wiki/concepts/function-codes.md)
