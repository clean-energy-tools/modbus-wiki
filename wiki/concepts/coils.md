---
title: Coils
Summary: Single-bit read-write data objects in MODBUS used for digital outputs and control relays.
Sources:
  - raw/MODBUS/MODBUS.md
  - raw/MODBUS/modbusprotocolspecification.md
Categories:
  - data-model
  - digital-outputs
  - bit-access
date-created: 2026-04-18T12:00:00+03:00
last-updated: 2026-04-18T14:43:37+03:00
---

Coils are single-bit read-write data objects in the MODBUS data model, typically used for digital outputs, control relays, and other binary control signals (source: [modbusprotocolspecification.md](raw/MODBUS/modbusprotocolspecification.md)).

## Overview

Coils represent binary output data that can be both read and written by MODBUS clients. They are one of four primary data tables defined by the MODBUS protocol (source: [modbusprotocolspecification.md](raw/MODBUS/modbusprotocolspecification.md)):

| Table | Object Type | Size  | Access     | Typical Use             |
| ----- | ----------- | ----- | ---------- | ----------------------- |
| Coils | Bit         | 1 bit | Read/Write | Digital outputs, relays |

## Coil Properties

| Property        | Value                                             |
| --------------- | ------------------------------------------------- |
| Data size       | 1 bit                                             |
| Access type     | Read/Write                                        |
| Typical usage   | Digital outputs, control relays, indicator lights |
| Address range   | 0-65535 (PDU address)                             |
| Legacy notation | 0xxxx (e.g., coil 1 = 00001)                      |

## Addressing

**PDU addresses are 0-based**, while documentation often uses 1-based numbering:

| Documentation | PDU Address | Wire Value |
|---------------|---------------|------------|
| Coil 1 | 0 | 0x0000 |
| Coil 100 | 99 | 0x0063 |
| Coil 1000 | 999 | 0x03E7 |

**Legacy notation:** Coils are often referenced using the 0xxxx convention:
- 00001 = Coil 1
- 01000 = Coil 1000
- 00001 = PDU address 0

For implementation, always use 0-based addresses in PDUs.

## Function Codes for Coils

### Read Operations

#### Function 0x01: Read Coils

Read status of one or more coils from a remote device.

**Request:**
```
[Function Code: 0x01][Starting Address: 2][Quantity of Coils: 2]
```

**Maximum quantity:** 2000 coils

**Response:**
```
[Function Code: 0x01][Byte Count: 1][Coil Status: N bytes]
```

**Bit packing:**
- Coil at starting address is in bit 0 (LSB) of first byte
- Bits are packed sequentially
- Byte count = ⌈Quantity / 8⌉

**Example:** Read 19 coils starting at address 19
```
Request:  01 00 13 00 13
Response: 01 03 CD 6B 05
```

**Decoding the response:**
- Byte 0: 0xCD = 11001101
  - Coil 19 (bit 0): 1 (ON)
  - Coil 20 (bit 1): 0 (OFF)
  - Coil 21 (bit 2): 1 (ON)
  - Coil 22 (bit 3): 1 (ON)
  - ...etc

### Write Operations

#### Function 0x05: Write Single Coil

Write a single coil to ON or OFF.

**Request:**
```
[Function Code: 0x05][Coil Address: 2][Output Value: 2]
```

**Valid values:**
- 0xFF00 = ON
- 0x0000 = OFF

**Note:** Only 0xFF00 and 0x0000 are valid. Other values cause exception 0x03 (ILLEGAL DATA VALUE).

**Response:** Echo of request (confirms write).

**Example:** Turn ON coil 172
```
Request:  05 00 AC FF 00
Response: 05 00 AC FF 00
```

#### Function 0x0F: Write Multiple Coils

Write multiple coils to ON or OFF.

**Request:**
```
[Function Code: 0x0F][Starting Address: 2][Quantity of Coils: 2][Byte Count: 1][Coil Values: N]
```

**Maximum quantity:** 1968 coils

**Response:**
```
[Function Code: 0x0F][Starting Address: 2][Quantity of Coils: 2]
```

**Bit packing:** Same format as read coils - LSB of first byte is coil at starting address.

**Example:** Write 10 coils
```
Request:  0F 00 00 00 0A 02 CD 01
```

## Typical Applications

Coils are commonly used for:
- Digital outputs (DO) in industrial control systems
- Control relays and contactors
- Indicator lights and status LEDs
- Motor start/stop commands
- Valve open/close commands
- Pump on/off control
- Fan speed control (high/medium/low/off)
- Mode selection switches

## Data Representation

**Binary values:**
- 0 = OFF / False / Closed
- 1 = ON / True / Open

**In MODBUS requests:**
- Write Single Coil (0x05): 0xFF00 = ON, 0x0000 = OFF
- Write Multiple Coils (0x0F): Individual bits in packed bytes

## Implementation Considerations

### Reading Coils

**Address conversion:**
```
Documentation coil 100 → PDU address 99
Documentation coil 1 → PDU address 0
```

**Bit extraction from response:**
```
byte_index = (coil_index - start_address) / 8
bit_index = (coil_index - start_address) % 8
coil_value = (response_bytes[byte_index] >> bit_index) & 0x01
```

### Writing Coils

**Write Single Coil (0x05):**
```
value = 0xFF00 if ON else 0x0000
```

**Write Multiple Coils (0x0F):**
```
// Pack coil values into bytes
for i in 0..num_coils:
    byte_index = i / 8
    bit_index = i % 8
    if coils[i]:
        data[byte_index] |= (1 << bit_index)
```

## Error Handling

Common exception codes for coil operations:
- **0x01:** ILLEGAL FUNCTION - Device doesn't support this function
- **0x02:** ILLEGAL DATA ADDRESS - Coil address out of range
- **0x03:** ILLEGAL DATA VALUE - Invalid coil value for write single coil (not 0xFF00 or 0x0000)

## Related Data Objects

Compare with other MODBUS data objects:

| Object | Size | Access | Typical Use |
|--------|------|--------|-------------|
| Coils | 1 bit | Read/Write | Digital outputs, relays |
| Discrete Inputs | 1 bit | Read-only | Digital inputs, switches |
| Input Registers | 16 bits | Read-only | Analog measurements |
| Holding Registers | 16 bits | Read/Write | Configuration, setpoints |

## Related pages

- [discrete-inputs](/wiki/concepts/discrete-inputs.md)
- [holding-registers](/wiki/concepts/holding-registers.md)
- [input-registers](/wiki/concepts/input-registers.md)
- [function-codes](/wiki/concepts/function-codes.md)
