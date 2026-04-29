---
title: Coils
Summary: Single-bit read-write data objects in MODBUS used for digital outputs and control relays.
Sources:
  - raw/MODBUS/MODBUS.md
  - raw/MODBUS/modbusprotocolspecification.md
Categories:
  - data-model
  - bit-access
  - digital-outputs
type: concept
date-created: 2026-04-18T12:00:00+03:00
last-updated: 2026-04-18T14:43:37+03:00
---

Coils are single-bit read-write data in the MODBUS data model, typically used for digital outputs, control relays, and other on/off signals (source: [modbusprotocolspecification.md](/raw/MODBUS/modbusprotocolspecification.md)).

## Overview

Coils represent binary (on/off) output data that you can both read and write. They are one of four main data types defined by the MODBUS protocol (source: [modbusprotocolspecification.md](/raw/MODBUS/modbusprotocolspecification.md)):

| Table | What It Is | Size  | Can You Write?     | What It's For             |
| ----- | ----------- | ----- | ---------- | ----------------------- |
| Coils | Single bit         | 1 bit | Yes (Read/Write) | Digital outputs, relays |

## Coil Properties

| Property        | Value                                             |
| --------------- | ------------------------------------------------- |
| Size       | 1 bit (on or off)                                             |
| Can you write to it?     | Yes (Read/Write)                                        |
| What it's used for   | Digital outputs, control relays, indicator lights |
| How many   | 0 to 65,535 addresses                            |
| Old-style notation | 0xxxx (like coil 1 = 00001)                      |

## Addressing

**Addresses in MODBUS messages start at 0**, while documentation often counts starting from 1:

| Documentation Says | Actual Address in Message | On the Wire |
|---------------|---------------|------------|
| Coil 1 | 0 | 0x0000 |
| Coil 100 | 99 | 0x0063 |
| Coil 1000 | 999 | 0x03E7 |

**Old-style notation:** Coils are sometimes written using the 0xxxx convention:
- 00001 = Coil 1
- 01000 = Coil 1000
- 00001 = Actual address 0

In your code, always use addresses starting from 0.

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

**Note:** Only these two values are valid. Any other value will cause the device to send back error code 0x03 (ILLEGAL DATA VALUE).

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

- [Discrete Inputs](/wiki/concepts/discrete-inputs.md)
- [Holding Registers](/wiki/concepts/holding-registers.md)
- [Input Registers](/wiki/concepts/input-registers.md)
- [Function Codes](/wiki/concepts/function-codes.md)

## Backlinks

- [Converting MODBUS Registers to Program Variables](/wiki/answers/converting-modbus-registers-to-program-variables.md)
- [MODBUS Data Validation](/wiki/answers/modbus-data-validation.md)
- [MODBUS Function Codes Guide](/wiki/answers/modbus-function-codes-guide.md)
- [MODBUS Register Addressing](/wiki/answers/modbus-register-addressing.md)
- [MODBUS Register Data Representation](/wiki/answers/modbus-register-data-representation.md)
- [Reading MODBUS Register Maps](/wiki/answers/reading-modbus-register-maps.md)
- [Discrete Inputs](/wiki/concepts/discrete-inputs.md)
- [Function Codes](/wiki/concepts/function-codes.md)
- [Holding Registers](/wiki/concepts/holding-registers.md)
- [Input Registers](/wiki/concepts/input-registers.md)
- [MODBUS](/wiki/concepts/modbus.md)
- [MODBUS Usage and Applications](/wiki/concepts/modbus-usage-and-applications.md)
- [Summary of MODBUS](/wiki/summaries/MODBUS/MODBUS.md)
- [Summary of modbusprotocolspecification](/wiki/summaries/MODBUS/modbusprotocolspecification.md)

------------

MODBUS is a trademark of the Modbus Organization, Inc.

All information on this page is derived by summarizing and analyzing solely the MODBUS specifications published by the Modbus Organization, Inc.  The specifications are copyright by the Modbus Organization.

By deriving this information solely from the specifications we hope to stay true to the specification.

No infringement is intended.
