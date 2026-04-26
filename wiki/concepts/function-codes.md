---
title: Function Codes
Summary: MODBUS function codes defining operations for reading and writing data including coils, discrete inputs, and registers.
Sources:
  - raw/MODBUS/MODBUS.md
  - raw/MODBUS/modbusprotocolspecification.md
Categories:
  - protocol-components
  - operations
  - data-access
type: concept
date-created: 2026-04-18T12:00:00+03:00
last-updated: 2026-04-18T14:43:24+03:00
---

MODBUS function codes specify the operations to be performed by the server. They are organized into bit access (coils/discrete inputs) and word access (registers) operations (source: [modbusprotocolspecification.md](/raw/MODBUS/modbusprotocolspecification.md)).

## Function Code Overview

Function codes are 1-byte values in the MODBUS PDU that indicate the specific operation to perform:
- Normal function codes: 0x01-0x7F
- Exception responses: Function code + 0x80 (bit 7 set)

## Bit Access Function Codes

### Function 0x01: Read Coils

Read status of coils (read-write bits) from a remote device.

**Request (5 bytes):**
| Offset | Field | Size | Value Range |
|--------|-------|------|-------------|
| 0 | Function Code | 1 | 0x01 |
| 1-2 | Starting Address | 2 | 0x0000 - 0xFFFF |
| 3-4 | Quantity of Coils | 2 | 0x0001 - 0x07D0 (1-2000) |

**Response (2 + N bytes):**
| Offset | Field | Size | Description |
|--------|-------|------|-------------|
| 0 | Function Code | 1 | 0x01 |
| 1 | Byte Count | 1 | ⌈Quantity / 8⌉ |
| 2+ | Coil Status | N | Packed bits, LSB = lowest address |

**Bit packing:** Coil at starting address is in bit 0 (LSB) of first byte.

**Example:** Read 19 coils starting at address 19
```
Request:  01 00 13 00 13
Response: 01 03 CD 6B 05
```

### Function 0x02: Read Discrete Inputs

Read status of discrete inputs (read-only bits) from a remote device.

**Request (5 bytes):**
| Offset | Field | Size | Value Range |
|--------|-------|------|-------------|
| 0 | Function Code | 1 | 0x02 |
| 1-2 | Starting Address | 2 | 0x0000 - 0xFFFF |
| 3-4 | Quantity of Inputs | 2 | 0x0001 - 0x07D0 (1-2000) |

**Response (2 + N bytes):**
| Offset | Field | Size | Description |
|--------|-------|------|-------------|
| 0 | Function Code | 1 | 0x02 |
| 1 | Byte Count | 1 | ⌈Quantity / 8⌉ |
| 2+ | Input Status | N | Packed bits, LSB = lowest address |

### Function 0x05: Write Single Coil

Write a single coil to ON or OFF.

**Request (5 bytes):**
| Offset | Field | Size | Value |
|--------|-------|------|-------|
| 0 | Function Code | 1 | 0x05 |
| 1-2 | Coil Address | 2 | 0x0000 - 0xFFFF |
| 3-4 | Output Value | 2 | 0xFF00 (ON), 0x0000 (OFF) |

**Note:** Only 0xFF00 (ON) and 0x0000 (OFF) are valid. Other values cause exception 0x03.

**Response (5 bytes):** Echo of request.

**Example:** Turn ON coil 172
```
Request:  05 00 AC FF 00
Response: 05 00 AC FF 00
```

### Function 0x0F: Write Multiple Coils

Write multiple coils to ON or OFF.

**Request (6 + N bytes):**
| Offset | Field | Size | Value Range |
|--------|-------|------|-------------|
| 0 | Function Code | 1 | 0x0F |
| 1-2 | Starting Address | 2 | 0x0000 - 0xFFFF |
| 3-4 | Quantity of Coils | 2 | 0x0001 - 0x07B0 (1-1968) |
| 5 | Byte Count | 1 | ⌈Quantity / 8⌉ |
| 6+ | Coil Values | N | Packed bits, LSB = lowest address |

**Response (5 bytes):**
| Offset | Field | Size | Description |
|--------|-------|------|-------------|
| 0 | Function Code | 1 | 0x0F |
| 1-2 | Starting Address | 2 | Address of first coil |
| 3-4 | Quantity of Coils | 2 | Number of coils written |

## Word Access Function Codes

### Function 0x03: Read Holding Registers

Read holding register values (read-write 16-bit words) from a remote device.

**Request (5 bytes):**
| Offset | Field | Size | Value Range |
|--------|-------|------|-------------|
| 0 | Function Code | 1 | 0x03 |
| 1-2 | Starting Address | 2 | 0x0000 - 0xFFFF |
| 3-4 | Quantity of Registers | 2 | 0x0001 - 0x007D (1-125) |

**Response (2 + 2N bytes):**
| Offset | Field | Size | Description |
|--------|-------|------|-------------|
| 0 | Function Code | 1 | 0x03 |
| 1 | Byte Count | 1 | 2 × Quantity |
| 2+ | Register Values | 2N | N registers, 2 bytes each (big-endian) |

**Example:** Read 3 registers starting at address 107
```
Request:  03 00 6B 00 03
Response: 03 06 02 2B 00 00 00 64
```

### Function 0x04: Read Input Registers

Read input register values (read-only 16-bit words) from a remote device.

**Request (5 bytes):**
| Offset | Field | Size | Value Range |
|--------|-------|------|-------------|
| 0 | Function Code | 1 | 0x04 |
| 1-2 | Starting Address | 2 | 0x0000 - 0xFFFF |
| 3-4 | Quantity of Registers | 2 | 0x0001 - 0x007D (1-125) |

**Response (2 + 2N bytes):**
| Offset | Field | Size | Description |
|--------|-------|------|-------------|
| 0 | Function Code | 1 | 0x04 |
| 1 | Byte Count | 1 | 2 × Quantity |
| 2+ | Register Values | 2N | N registers, 2 bytes each (big-endian) |

### Function 0x06: Write Single Register

Write a single register value.

**Request (5 bytes):**
| Offset | Field | Size | Value Range |
|--------|-------|------|-------------|
| 0 | Function Code | 1 | 0x06 |
| 1-2 | Register Address | 2 | 0x0000 - 0xFFFF |
| 3-4 | Register Value | 2 | 0x0000 - 0xFFFF |

**Response (5 bytes):** Echo of request.

**Example:** Write 0x0003 to register 1
```
Request:  06 00 01 00 03
Response: 06 00 01 00 03
```

### Function 0x10: Write Multiple Registers

Write multiple register values.

**Request (6 + 2N bytes):**
| Offset | Field | Size | Value Range |
|--------|-------|------|-------------|
| 0 | Function Code | 1 | 0x10 |
| 1-2 | Starting Address | 2 | 0x0000 - 0xFFFF |
| 3-4 | Quantity of Registers | 2 | 0x0001 - 0x007B (1-123) |
| 5 | Byte Count | 1 | 2 × Quantity |
| 6+ | Register Values | 2N | Values to write (big-endian) |

**Response (5 bytes):**
| Offset | Field | Size | Description |
|--------|-------|------|-------------|
| 0 | Function Code | 1 | 0x10 |
| 1-2 | Starting Address | 2 | Address of first register |
| 3-4 | Quantity of Registers | 2 | Number of registers written |

**Example:** Write 2 registers
```
Request:  10 00 01 00 02 04 00 0A 01 02
Response: 10 00 01 00 02
```

## Other Function Codes

### Function 0x16: Mask Write Register

Modify specific bits within a register using AND/OR masks.

**Request (7 bytes):**
| Offset | Field | Size | Description |
|--------|-------|------|-------------|
| 0 | Function Code | 1 | 0x16 |
| 1-2 | Reference Address | 2 | Register address |
| 3-4 | AND_Mask | 2 | AND mask |
| 5-6 | OR_Mask | 2 | OR mask |

**Operation:** `New value = (Current value AND AND_Mask) OR (OR_Mask AND ~AND_Mask)`

**Response (7 bytes):** Echo of request.

### Function 0x17: Read/Write Multiple Registers

Perform a combined read and write operation.

**Request (10 + 2W bytes):**
| Offset | Field | Size | Description |
|--------|-------|------|-------------|
| 0 | Function Code | 1 | 0x17 |
| 1-2 | Read Starting Address | 2 | Address to read from |
| 3-4 | Quantity to Read | 2 | 0x0001 - 0x007D (1-125) |
| 5-6 | Write Starting Address | 2 | Address to write to |
| 7-8 | Quantity to Write | 2 | 0x0001 - 0x0079 (1-121) |
| 9 | Write Byte Count | 1 | 2 × Quantity to Write |
| 10+ | Write Register Values | 2W | Values to write |

**Response (3 + 2R bytes):**
| Offset | Field | Size | Description |
|--------|-------|------|-------------|
| 0 | Function Code | 1 | 0x17 |
| 1 | Byte Count | 1 | 2 × Quantity to Read |
| 2+ | Read Register Values | 2R | Read values |

### Function 0x2B: Encapsulated Interface Transport (MEI)

Transport encapsulated data for vendor-specific functions.

**Request (3 + N bytes):**
| Offset | Field | Size | Description |
|--------|-------|------|-------------|
| 0 | Function Code | 1 | 0x2B |
| 1 | MEI Type | 1 | Vendor-specific |
| 2+ | Encapsulated Data | N | Vendor-defined |

Common MEI types:
- 0x0D: Read Device Identification
- 0x0E: Read Device Identification (detailed)
- 0x0E: Read File Record
- 0x0F: Write File Record

## Function Code Summary

| Code | Hex | Name | Access | Max Quantity |
|------|-----|------|--------|--------------|
| 01 | 0x01 | Read Coils | Coils | 2000 bits |
| 02 | 0x02 | Read Discrete Inputs | Discrete Inputs | 2000 bits |
| 03 | 0x03 | Read Holding Registers | Holding Registers | 125 registers |
| 04 | 0x04 | Read Input Registers | Input Registers | 125 registers |
| 05 | 0x05 | Write Single Coil | Coils | 1 bit |
| 06 | 0x06 | Write Single Register | Holding Registers | 1 register |
| 15 | 0x0F | Write Multiple Coils | Coils | 1968 bits |
| 16 | 0x10 | Write Multiple Registers | Holding Registers | 123 registers |
| 22 | 0x16 | Mask Write Register | Holding Registers | 1 register |
| 23 | 0x17 | Read/Write Multiple Registers | Holding Registers | R:125, W:121 |
| 43 | 0x2B | Encapsulated Interface Transport | Various | - |

## Exception Codes

When an error occurs, the server responds with function code + 0x80 and an exception code:

| Code | Hex | Name | Description |
|------|-----|------|-------------|
| 01 | 0x01 | ILLEGAL FUNCTION | Function code not supported |
| 02 | 0x02 | ILLEGAL DATA ADDRESS | Address out of range |
| 03 | 0x03 | ILLEGAL DATA VALUE | Invalid quantity or value |
| 04 | 0x04 | SERVER DEVICE FAILURE | Internal error |
| 05 | 0x05 | ACKNOWLEDGE | Request accepted, long processing |
| 06 | 0x06 | SERVER DEVICE BUSY | Retry later |
| 08 | 0x08 | MEMORY PARITY ERROR | Parity check failed |
| 10 | 0x0A | GATEWAY PATH UNAVAILABLE | Gateway issue |
| 11 | 0x0B | GATEWAY TARGET NO RESPONSE | Target failed to respond |

## Related pages

- [Coils](/wiki/concepts/coils.md)
- [Discrete Inputs](/wiki/concepts/discrete-inputs.md)
- [Holding Registers](/wiki/concepts/holding-registers.md)
- [Input Registers](/wiki/concepts/input-registers.md)
- [exception-codes](/wiki/concepts/exception-codes.md)

## Backlinks

- [MODBUS Commissioning Checklist](/wiki/answers/modbus-commissioning-checklist.md)
- [MODBUS Data Validation](/wiki/answers/modbus-data-validation.md)
- [MODBUS Errors and Exception Responses](/wiki/answers/modbus-errors-and-exceptions.md)
- [MODBUS Function Codes Complete Guide](/wiki/answers/modbus-function-codes-guide.md)
- [MODBUS Register Addressing and Address Space Organization](/wiki/answers/modbus-register-addressing.md)
- [Reading MODBUS Register Maps](/wiki/answers/reading-modbus-register-maps.md)
- [Coils](/wiki/concepts/coils.md)
- [Discrete Inputs](/wiki/concepts/discrete-inputs.md)
- [Holding Registers](/wiki/concepts/holding-registers.md)
- [Implementation](/wiki/concepts/implementation.md)
- [Input Registers](/wiki/concepts/input-registers.md)
- [MBAP Header](/wiki/concepts/mbap-header.md)
- [MODBUS ASCII](/wiki/concepts/modbus-ascii.md)
- [MODBUS Data Type Mapping](/wiki/concepts/modbus-data-type-mapping.md)
- [MODBUS Protocol](/wiki/concepts/modbus.md)
- [MODBUS RTU](/wiki/concepts/modbus-rtu.md)
- [MODBUS TCP](/wiki/concepts/modbus-tcp.md)
- [Protocol](/wiki/concepts/protocol.md)
- [TCP Connection Management](/wiki/concepts/tcp-connection-management.md)
- [Summary of MODBUS Application Protocol Specification](/wiki/summaries/MODBUS/MODBUS.md)
- [Summary of MODBUS Messaging on TCP/IP Implementation Guide](/wiki/summaries/MODBUS/messagingimplementationguide.md)
- [Summary of MODBUS over serial line specification and implementation guide](/wiki/summaries/MODBUS/modbusoverserial.md)
- [Summary of MODBUS Protocol Specification for AI Implementation](/wiki/summaries/MODBUS/modbusprotocolspecification.md)

------------

MODBUS is a trademark of the Modbus Organization, Inc.

All information on this page is derived by summarizing and analyzing solely the MODBUS specifications published by the Modbus Organization, Inc.  The specifications are copyright by the Modbus Organization.

By deriving this information solely from the specifications we hope to stay true to the specification.

No infringement is intended.
