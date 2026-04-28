---
title: MODBUS Function Codes Complete Guide
Summary: Comprehensive guide to MODBUS function codes including what they do, how they are grouped, when to use each one, efficiency comparisons between single and multiple operations, and practical decision criteria.
Sources:
  - /raw/MODBUS/modbusprotocolspecification.md
  - /raw/MODBUS/MODBUS.md
Categories:
  - function-codes
  - protocol-operations
  - efficiency
  - best-practices
type: answer
date-created: 2026-04-24T17:00:00+03:00
last-updated: 2026-04-24T17:00:00+03:00
---

MODBUS function codes tell the device which operation to perform. This guide explains what each function code does, how they're organized, when to use one instead of another, and whether reading/writing multiple values at once is more efficient than one at a time.

## Quick Answer: What Are Function Codes?

**Function codes are 1-byte numbers (0x01 to 0x7F) that specify what to do:**
- Read data (coils, discrete inputs, registers)
- Write data (coils, registers)
- Perform special operations (diagnostics, device identification)

**The function code determines:**
- Which type of data to access (bits vs 16-bit values)
- Whether you're reading or writing
- How many values you can transfer at once
- What the request and response messages look like

(source: [modbusprotocolspecification.md](/raw/MODBUS/modbusprotocolspecification.md))

## Complete Function Code List

### Public Function Codes (Standard)

These are defined in the MODBUS specification and supported by most devices:

| Code | Hex | Name | Operation | Data Type | Max Qty | Common Use |
|------|-----|------|-----------|-----------|---------|------------|
| **01** | 0x01 | Read Coils | Read | Coils (R/W bits) | 2000 | Read output states |
| **02** | 0x02 | Read Discrete Inputs | Read | Discrete Inputs (RO bits) | 2000 | Read input states |
| **03** | 0x03 | Read Holding Registers | Read | Holding Registers (R/W words) | 125 | Read config/setpoints |
| **04** | 0x04 | Read Input Registers | Read | Input Registers (RO words) | 125 | Read measurements |
| **05** | 0x05 | Write Single Coil | Write | Coils | 1 | Set one output |
| **06** | 0x06 | Write Single Register | Write | Holding Registers | 1 | Write one value |
| **15** | 0x0F | Write Multiple Coils | Write | Coils | 1968 | Set multiple outputs |
| **16** | 0x10 | Write Multiple Registers | Write | Holding Registers | 123 | Write multiple values |
| **22** | 0x16 | Mask Write Register | Modify | Holding Registers | 1 | Modify specific bits |
| **23** | 0x17 | Read/Write Multiple Registers | Read+Write | Holding Registers | R:125, W:121 | Atomic read-modify-write |
| **43** | 0x2B | Encapsulated Interface Transport | Various | - | - | Device ID, file access |

(source: [modbusprotocolspecification.md](/raw/MODBUS/modbusprotocolspecification.md))

### User-Defined Function Codes

**Codes 65-72 and 100-110:** Reserved for user-defined functions (vendor-specific)

**Warning:** Not portable between vendors. Use only if:
- Standard functions insufficient
- Vendor documentation specifies their use
- Client and server from same vendor

## Function Code Grouping

### By Data Type

Function codes are organized by the type of data they access:

**Discrete (1-bit) Operations:**

| Code | Name | Data Type | Access |
|------|------|-----------|--------|
| 0x01 | Read Coils | Coils | Read |
| 0x05 | Write Single Coil | Coils | Write |
| 0x0F | Write Multiple Coils | Coils | Write |
| 0x02 | Read Discrete Inputs | Discrete Inputs | Read |

**Register (16-bit word) Operations:**

| Code | Name | Data Type | Access |
|------|------|-----------|--------|
| 0x03 | Read Holding Registers | Holding Registers | Read |
| 0x06 | Write Single Register | Holding Registers | Write |
| 0x10 | Write Multiple Registers | Holding Registers | Write |
| 0x16 | Mask Write Register | Holding Registers | Modify |
| 0x17 | Read/Write Multiple Registers | Holding Registers | Read+Write |
| 0x04 | Read Input Registers | Input Registers | Read |

### By Operation Type

**Read Operations (Queries):**

| Code | Description | Typical Use |
|------|-------------|-------------|
| 0x01 | Read Coils | Check relay/output states |
| 0x02 | Read Discrete Inputs | Check switch/sensor states |
| 0x03 | Read Holding Registers | Read configuration, setpoints, control values |
| 0x04 | Read Input Registers | Read sensor measurements, status |

**Write Operations (Commands):**

| Code | Description | Typical Use |
|------|-------------|-------------|
| 0x05 | Write Single Coil | Turn relay on/off |
| 0x06 | Write Single Register | Change setpoint, write config value |
| 0x0F | Write Multiple Coils | Set multiple outputs at once |
| 0x10 | Write Multiple Registers | Write configuration block, update multiple setpoints |

**Special Operations:**

| Code | Description | Typical Use |
|------|-------------|-------------|
| 0x16 | Mask Write Register | Modify bits without read-modify-write cycle |
| 0x17 | Read/Write Multiple | Atomic exchange (read before write) |
| 0x2B | Encapsulated Interface | Device identification, file operations |

### By Read-Only vs Read-Write Data

**Read-Only Data (Inputs):**
- Discrete Inputs (0x02) - Digital inputs, cannot be written
- Input Registers (0x04) - Measurements, sensor data, cannot be written

**Read-Write Data (Outputs/Configuration):**
- Coils (0x01, 0x05, 0x0F) - Digital outputs, can read and write
- Holding Registers (0x03, 0x06, 0x10, 0x16, 0x17) - Configuration, setpoints, can read and write

**Why the distinction matters:**
- Trying to write to Input Registers or Discrete Inputs → Exception 0x01 (ILLEGAL FUNCTION)
- Reading from Coils/Holding Registers works (they're read-write, not write-only)

## What Each Function Code Does

### 0x01: Read Coils

**Purpose:** Read the ON/OFF status of output coils (relays, digital outputs)

**When to use:**
- Check which outputs are currently active
- Verify output state before/after control operation
- Monitor relay status for diagnostics

**Example:**
```
Request:  Read 10 coils starting at address 20
Response: Returns 10 bits (2 bytes) with current ON/OFF states
```

**Typical applications:**
- Reading relay states in a PLC
- Checking valve positions
- Verifying motor control outputs

### 0x02: Read Discrete Inputs

**Purpose:** Read the ON/OFF status of discrete inputs (switches, sensors)

**When to use:**
- Read digital input states (limit switches, buttons, sensors)
- Monitor alarm conditions
- Check status bits

**Difference from 0x01:**
- Discrete Inputs are read-only (physical inputs)
- Coils are read-write (controllable outputs)

**Example:**
```
Request:  Read 8 discrete inputs starting at address 0
Response: Returns 8 bits (1 byte) with current input states
```

**Typical applications:**
- Reading limit switch states
- Checking alarm input status
- Monitoring button presses

### 0x03: Read Holding Registers

**Purpose:** Read 16-bit register values for configuration, setpoints, and control data

**When to use:**
- Read device configuration settings
- Get current setpoint values
- Read control parameters
- Retrieve stored data

**Most commonly used function code** - works with most MODBUS data

**Example:**
```
Request:  Read 3 registers starting at address 100
Response: Returns 6 bytes (3 × 16-bit values)
```

**Typical applications:**
- Reading temperature setpoint
- Getting PID controller parameters
- Retrieving device configuration
- Reading counters and accumulators

### 0x04: Read Input Registers

**Purpose:** Read 16-bit register values for measurements and sensor data

**When to use:**
- Read sensor measurements (temperature, pressure, flow)
- Get calculated values
- Retrieve status information
- Access read-only data

**Difference from 0x03:**
- Input Registers are read-only (measurements, inputs)
- Holding Registers are read-write (configuration, setpoints)

**Example:**
```
Request:  Read 2 registers starting at address 0
Response: Returns 4 bytes (2 × 16-bit values)
```

**Typical applications:**
- Reading current temperature measurement
- Getting pressure sensor value
- Retrieving flow rate
- Accessing device status registers

### 0x05: Write Single Coil

**Purpose:** Write one coil (output) to ON or OFF

**When to use:**
- Turn single relay on or off
- Control one valve
- Set one digital output
- Toggle one bit

**Valid values:**
- 0xFF00: ON (true, 1)
- 0x0000: OFF (false, 0)
- Any other value → Exception 0x03 (ILLEGAL DATA VALUE)

**Example:**
```
Request:  Write coil 172 to ON (0xFF00)
Response: Echo of request (confirms write successful)
```

**Typical applications:**
- Turning pump on/off
- Opening/closing valve
- Enabling/disabling output

### 0x06: Write Single Register

**Purpose:** Write one 16-bit register value

**When to use:**
- Write single setpoint
- Update one configuration parameter
- Set one control value
- Change one register

**Example:**
```
Request:  Write value 0x0003 to register 1
Response: Echo of request (confirms write successful)
```

**Typical applications:**
- Setting temperature setpoint
- Writing speed reference
- Updating configuration parameter
- Clearing counter

### 0x0F: Write Multiple Coils

**Purpose:** Write multiple coils (outputs) in one operation

**When to use:**
- Set multiple outputs simultaneously
- Write more than one coil
- Initialize output pattern
- More efficient than multiple 0x05 operations

**Range:** 1 to 1968 coils per request

**Example:**
```
Request:  Write 10 coils starting at address 19 to pattern 1100110011
Response: Confirms starting address and quantity written
```

**Typical applications:**
- Setting multiple relay states
- Initializing output configuration
- Batch control of outputs

### 0x10: Write Multiple Registers

**Purpose:** Write multiple 16-bit register values in one operation

**When to use:**
- Write multiple setpoints
- Initialize device configuration
- Write multi-register data types (FLOAT32, UINT32)
- More efficient than multiple 0x06 operations

**Range:** 1 to 123 registers per request

**Example:**
```
Request:  Write 2 registers starting at address 1 with values [0x000A, 0x0102]
Response: Confirms starting address and quantity written
```

**Typical applications:**
- Writing configuration block
- Setting multiple parameters
- Writing FLOAT32 value (2 registers)
- Initializing device on startup

### 0x16: Mask Write Register

**Purpose:** Modify specific bits within a register without affecting other bits

**When to use:**
- Change control bits in status register
- Set/clear flags without read-modify-write
- Atomic bit manipulation
- Avoid race conditions

**Operation:**
```
New Value = (Current Value AND AND_Mask) OR (OR_Mask AND ~AND_Mask)
```

**Example - Set bit 3, clear bit 1, leave others unchanged:**
```
AND_Mask: 0xFFFD (bit 1 = 0, others = 1)
OR_Mask:  0x0008 (bit 3 = 1, others = 0)
```

**Typical applications:**
- Setting control flags
- Enabling/disabling features via bits
- Modifying mode registers

### 0x17: Read/Write Multiple Registers

**Purpose:** Read and write multiple registers in single atomic operation

**When to use:**
- Read-modify-write scenario (get current, calculate new, write back)
- Exchange data with device
- Update parameters based on current values
- Guarantee no other operation occurs between read and write

**Atomicity:** Device processes read then write without interruption

**Range:**
- Read: 1 to 125 registers
- Write: 1 to 121 registers

**Example:**
```
Request:  Read 3 registers from address 3, write 2 registers to address 14
Response: Returns 3 register values read
```

**Typical applications:**
- PID controller parameter update
- Configuration exchange
- Data synchronization

### 0x2B: Encapsulated Interface Transport (MEI)

**Purpose:** Access additional functions via encapsulation

**When to use:**
- Read device identification (vendor, model, serial number)
- Access file records
- Vendor-specific operations

**Common MEI Type 0x0E: Read Device Identification**

**Categories:**
- Basic: Vendor name, product code, version
- Regular: Model name, user application name
- Extended: All identification objects

**Example:**
```
Request:  MEI Type 0x0E, Read Device ID, Category 0 (Basic)
Response: Vendor name, product code, firmware version
```

**Typical applications:**
- Device discovery
- Inventory management
- Firmware version checking
- Automated device configuration

## When to Use One Function Code Over Another

### Single vs Multiple Operations

**Use Single (0x05, 0x06) when:**
- Writing only one value
- Quick one-off change
- Interactive control (user button press)
- Device may not support multiple write functions

**Use Multiple (0x0F, 0x10) when:**
- Writing 2 or more values
- More efficient (fewer transactions)
- Initializing device
- Batch updates
- Multi-register data types (FLOAT32, UINT32, etc.)

### Read Holding vs Input Registers

**Use 0x03 (Read Holding Registers) when:**
- Reading configuration values
- Reading setpoints
- Reading read-write data
- Device uses only holding registers (common in simple devices)

**Use 0x04 (Read Input Registers) when:**
- Reading measurements
- Reading sensor data
- Reading read-only status
- Device documentation specifies input registers

**When unsure:** Try 0x03 first (more universally supported)

### Read Coils vs Discrete Inputs

**Use 0x01 (Read Coils) when:**
- Reading controllable outputs
- Reading relay states
- Reading read-write bits

**Use 0x02 (Read Discrete Inputs) when:**
- Reading physical inputs
- Reading sensor states
- Reading read-only status bits

### Write Single Register vs Mask Write Register

**Use 0x06 (Write Single Register) when:**
- Writing entire register value
- Simple value update
- Device may not support 0x16

**Use 0x16 (Mask Write Register) when:**
- Modifying specific bits only
- Preserving other bits in register
- Avoiding read-modify-write race condition
- Working with control/status registers with independent bit flags

### Write Multiple vs Read/Write Multiple

**Use 0x10 (Write Multiple Registers) when:**
- Only need to write data
- Don't need to read current values
- Simpler operation

**Use 0x17 (Read/Write Multiple Registers) when:**
- Need current values before writing
- Read-modify-write operation
- Atomic exchange required
- Prevent other operations between read and write

## Efficiency: Single vs Multiple Operations

### Network Efficiency Comparison

**Scenario:** Write 5 register values

**Option 1: Five separate Write Single Register (0x06) operations**

```
Transaction 1: Write register 10 → 1 request + 1 response
Transaction 2: Write register 11 → 1 request + 1 response
Transaction 3: Write register 12 → 1 request + 1 response
Transaction 4: Write register 13 → 1 request + 1 response
Transaction 5: Write register 14 → 1 request + 1 response

Total: 5 request/response pairs
```

**MODBUS TCP overhead:**
- Each request: 12 bytes (MBAP header 7 + FC 1 + Addr 2 + Value 2)
- Each response: 12 bytes (MBAP header 7 + FC 1 + Addr 2 + Value 2)
- Total bytes: 5 × (12 + 12) = **120 bytes**
- Network transactions: **5**

**Option 2: One Write Multiple Registers (0x10) operation**

```
Transaction 1: Write 5 registers starting at 10 → 1 request + 1 response

Total: 1 request/response pair
```

**MODBUS TCP overhead:**
- Request: 23 bytes (MBAP 7 + FC 1 + Addr 2 + Qty 2 + ByteCnt 1 + Values 10)
- Response: 12 bytes (MBAP 7 + FC 1 + Addr 2 + Qty 2)
- Total bytes: 23 + 12 = **35 bytes**
- Network transactions: **1**

**Efficiency gain:**
- Bytes: 71% reduction (35 vs 120 bytes)
- Transactions: 80% reduction (1 vs 5 transactions)
- Time: ~80% faster (single round-trip vs five)

### Time Comparison

**Assumptions:**
- Network RTT (round-trip time): 20ms
- Processing time: negligible

**5 × Single Write:**
- Time = 5 × 20ms = **100ms**

**1 × Multiple Write:**
- Time = 1 × 20ms = **20ms**

**Result:** 5× faster using multiple write

### Serial (RTU) Efficiency

**MODBUS RTU has additional considerations:**

**Single write (0x06) frame:**
```
[Addr][FC][Addr_Hi][Addr_Lo][Value_Hi][Value_Lo][CRC_Lo][CRC_Hi]
8 bytes per transaction
```

**Multiple write (0x10) frame for 5 registers:**
```
Request:
[Addr][FC][Addr_Hi][Addr_Lo][Qty_Hi][Qty_Lo][ByteCnt][Val1_Hi][Val1_Lo]...[Val5_Lo][CRC_Lo][CRC_Hi]
19 bytes

Response:
[Addr][FC][Addr_Hi][Addr_Lo][Qty_Hi][Qty_Lo][CRC_Lo][CRC_Hi]
8 bytes
```

**5 × Single write:**
- Request+Response: 5 × (8 + 8) = 80 bytes
- Silent intervals: 5 × (t3.5 before + t3.5 after) = 10 × t3.5
- At 9600 baud, t3.5 ≈ 4ms → 40ms silent time

**1 × Multiple write:**
- Request+Response: 19 + 8 = 27 bytes
- Silent intervals: 2 × t3.5 = 7ms (one transaction)

**Time at 9600 baud:**
- 5 × Single: (80 bytes × 10 bits/byte) / 9600 bps + 40ms = 123ms
- 1 × Multiple: (27 bytes × 10 bits/byte) / 9600 bps + 7ms = 35ms

**Result:** 3.5× faster using multiple write on RTU

### When Single Operations Are Acceptable

**Use single operations when:**

1. **Writing only one value** (no efficiency gain from multiple)
2. **Interactive control** (user-initiated, not time-critical)
3. **Device doesn't support multiple functions** (some simple devices)
4. **Error isolation important** (want to know which specific write failed)
5. **Code simplicity** matters more than performance

### When Multiple Operations Are Essential

**Use multiple operations when:**

1. **Writing 2+ values** (significant efficiency gain)
2. **Polling multiple devices** (minimize total scan time)
3. **High-frequency updates** (performance critical)
4. **Writing multi-register data types** (FLOAT32, UINT32, etc.)
5. **Bandwidth limited** (serial bus, slow network)
6. **Initialization sequences** (writing many config values)

## Maximum Quantities

Each function code has protocol-defined limits:

| Function Code | Operation | Maximum Quantity | Reason for Limit |
|---------------|-----------|------------------|------------------|
| 0x01 | Read Coils | 2000 bits | PDU size limit (250 bytes) |
| 0x02 | Read Discrete Inputs | 2000 bits | PDU size limit |
| 0x03 | Read Holding Registers | 125 registers | PDU size limit (250 bytes) |
| 0x04 | Read Input Registers | 125 registers | PDU size limit |
| 0x05 | Write Single Coil | 1 bit | Single-value function |
| 0x06 | Write Single Register | 1 register | Single-value function |
| 0x0F | Write Multiple Coils | 1968 bits | PDU size limit |
| 0x10 | Write Multiple Registers | 123 registers | PDU size limit |
| 0x17 | Read/Write Multiple | Read: 125, Write: 121 | PDU size limit (combined) |

**PDU (Protocol Data Unit) size limit:** 253 bytes (256 - 1 FC - 2 overhead)

**Device limits may be lower:**
- Check device documentation
- Some devices limit to 50, 100 registers per read
- Test incrementally if limit unknown

(source: [modbusprotocolspecification.md](/raw/MODBUS/modbusprotocolspecification.md))

## Practical Decision Guide

### Choosing Read Function

```
┌─ Need to read data?
│
├─ Discrete (1-bit) values?
│  ├─ Outputs/controllable? → 0x01 (Read Coils)
│  └─ Inputs/read-only? → 0x02 (Read Discrete Inputs)
│
└─ Register (16-bit) values?
   ├─ Configuration/setpoints? → 0x03 (Read Holding Registers)
   ├─ Measurements/sensors? → 0x04 (Read Input Registers)
   └─ Unsure? → Try 0x03 first (most common)
```

### Choosing Write Function

```
┌─ Need to write data?
│
├─ Discrete (1-bit) values?
│  ├─ Writing 1 coil? → 0x05 (Write Single Coil)
│  └─ Writing 2+ coils? → 0x0F (Write Multiple Coils)
│
└─ Register (16-bit) values?
   ├─ Writing 1 register?
   │  ├─ Modify specific bits only? → 0x16 (Mask Write)
   │  └─ Write entire value? → 0x06 (Write Single Register)
   │
   ├─ Writing 2+ registers?
   │  ├─ Need to read first? → 0x17 (Read/Write Multiple)
   │  └─ Just writing? → 0x10 (Write Multiple Registers)
   │
   └─ Multi-register data type (FLOAT32, etc.)? → 0x10 (Write Multiple)
```

### Performance Optimization Decision

```
┌─ Writing multiple values?
│
├─ How many? 
│  ├─ 1 value → 0x06 (Single) is fine
│  ├─ 2-5 values → 0x10 (Multiple) recommended for efficiency
│  └─ 6+ values → 0x10 (Multiple) strongly recommended
│
├─ How often?
│  ├─ Once at startup → Either works (use 0x10 for simplicity)
│  ├─ Occasionally → Either works
│  └─ Frequently (>1Hz) → Use 0x10 (Multiple) for efficiency
│
└─ What medium?
   ├─ Fast Ethernet → Less critical, but 0x10 still better
   └─ Serial RS-485 → Critical, always use 0x10 for 2+ values
```

## Device Support Considerations

### Function Code Support Varies

**Not all devices support all function codes:**

**Minimal device (digital I/O module):**
- 0x01 (Read Coils)
- 0x02 (Read Discrete Inputs)
- 0x05 (Write Single Coil)
- Maybe 0x0F (Write Multiple Coils)

**Basic device (temperature controller):**
- 0x03 (Read Holding Registers)
- 0x04 (Read Input Registers)
- 0x06 (Write Single Register)
- 0x10 (Write Multiple Registers)

**Advanced device (PLC, gateway):**
- All basic functions (0x01-0x06, 0x0F, 0x10)
- Advanced functions (0x16, 0x17, 0x2B)

### Checking Device Support

**1. Read device documentation:**
```
"Supported Function Codes: 0x03, 0x04, 0x06, 0x10"
```

**2. Test empirically:**
```python
# Try function code
try:
    response = client.read_holding_registers(address=0, count=10, unit=1)
    print("Function 0x03 supported")
except ModbusException as e:
    if e.exception_code == 0x01:  # ILLEGAL FUNCTION
        print("Function 0x03 not supported")
```

**3. Use device identification (if supported):**
```python
# Function 0x2B, MEI Type 0x0E
response = client.read_device_information(unit=1)
# May include supported functions in response
```

### Fallback Strategy

**If device doesn't support multiple writes:**

```python
def write_registers(client, address, values, unit=1):
    """Write registers with automatic fallback"""
    try:
        # Try multiple write first (more efficient)
        client.write_registers(address, values, unit=unit)
    except ModbusException as e:
        if e.exception_code == 0x01:  # ILLEGAL FUNCTION
            # Fallback to single writes
            for i, value in enumerate(values):
                client.write_register(address + i, value, unit=unit)
        else:
            raise
```

## Summary: Efficiency Comparison Table

| Scenario | Function | Transactions | Relative Speed | When to Use |
|----------|----------|--------------|----------------|-------------|
| Write 1 register | 0x06 | 1 | 100% (baseline) | Single value |
| Write 5 registers (5× single) | 0x06 × 5 | 5 | 20% | Device doesn't support 0x10 |
| Write 5 registers (multiple) | 0x10 | 1 | 100% | Always preferred for 2+ values |
| Write 10 registers (10× single) | 0x06 × 10 | 10 | 10% | Never (too slow) |
| Write 10 registers (multiple) | 0x10 | 1 | 100% | Always preferred for 2+ values |
| Read 1 register | 0x03 | 1 | 100% | Single value |
| Read 20 registers (20× single) | 0x03 × 20 | 20 | 5% | Never (too slow) |
| Read 20 registers (multiple) | 0x03 | 1 | 100% | Always preferred for 2+ values |

**Key Takeaway:** Multiple-register functions (0x10, 0x03 with count >1) are dramatically more efficient than repeated single-register operations (0x06, 0x03 with count=1).

## Best Practices

### 1. Prefer Multiple Operations

**Always use multiple read/write for 2+ values:**
- 0x03 with quantity >1 instead of multiple 0x03 calls with quantity=1
- 0x10 instead of multiple 0x06

### 2. Batch Related Operations

**Bad (inefficient):**
```python
client.write_register(10, temp_setpoint, unit=1)
client.write_register(11, pressure_setpoint, unit=1)
client.write_register(12, flow_setpoint, unit=1)
# 3 transactions
```

**Good (efficient):**
```python
client.write_registers(10, [temp_setpoint, pressure_setpoint, flow_setpoint], unit=1)
# 1 transaction
```

### 3. Use Appropriate Function for Data Type

**Use correct register type:**
- Configuration/setpoints → Holding Registers (0x03, 0x06, 0x10)
- Measurements/sensors → Input Registers (0x04)
- Outputs → Coils (0x01, 0x05, 0x0F)
- Inputs → Discrete Inputs (0x02)

### 4. Respect Device Limits

**Check documentation for:**
- Maximum quantity per operation (may be less than protocol max)
- Supported function codes
- Special requirements or restrictions

### 5. Handle Multi-Register Data Types Properly

**FLOAT32, UINT32 require 2 registers:**

```python
# Write FLOAT32 temperature setpoint (2 registers)
import struct

temp = 23.5
bytes_data = struct.pack('>f', temp)
words = struct.unpack('>HH', bytes_data)

# Use 0x10 (Write Multiple Registers) for 2 registers
client.write_registers(address=10, values=list(words), unit=1)

# Don't use 0x06 twice (word order may be wrong, non-atomic)
```

### 6. Use Atomic Operations When Needed

**For read-modify-write:**
- Use 0x17 (Read/Write Multiple) if supported
- Or use 0x16 (Mask Write) for bit modifications
- Avoids race conditions

## Related Pages

- [Function Codes](/wiki/concepts/function-codes.md) - Detailed function code reference
- [Coils](/wiki/concepts/coils.md) - Coil data type details
- [Discrete Inputs](/wiki/concepts/discrete-inputs.md) - Discrete Input details
- [Holding Registers](/wiki/concepts/holding-registers.md) - Holding Register details
- [Input Registers](/wiki/concepts/input-registers.md) - Input Register details
- [MODBUS Errors and Exception Responses](/wiki/answers/modbus-errors-and-exceptions.md) - Exception handling
- [Reading MODBUS Register Maps](/wiki/answers/reading-modbus-register-maps.md) - Identifying function codes from documentation

## Backlinks

No backlinks (yet)

------------

MODBUS is a trademark of the Modbus Organization, Inc.

All information on this page is derived by summarizing and analyzing solely the MODBUS specifications published by the Modbus Organization, Inc.  The specifications are copyright by the Modbus Organization.

By deriving this information solely from the specifications we hope to stay true to the specification.

No infringement is intended.

None yet.
