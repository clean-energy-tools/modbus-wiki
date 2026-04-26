---
title: MODBUS Register Addressing and Address Space Organization
Summary: Comprehensive guide to MODBUS register addressing including address space organization, register types, function codes, addressing conventions, 0-based vs 1-based addressing, and legacy notation (40001, 30001, etc.).
Sources:
  - /raw/MODBUS/MODBUS.md
  - /raw/MODBUS/modbusprotocolspecification.md
Categories:
  - addressing
  - data-model
  - register-types
  - conventions
type: answer
date-created: 2026-04-23T15:00:00+03:00
last-updated: 2026-04-23T15:00:00+03:00
---

MODBUS register addressing can be confusing due to multiple addressing schemes and legacy conventions. This comprehensive guide explains how MODBUS organizes its address space, the different register types, addressing conventions, and the critical differences between documentation addresses and protocol addresses.

## Quick Answer: The Four Data Tables

MODBUS organizes its address space into **four separate data tables**:

| Table | Size | Access | Typical Use | Function Codes | Legacy Notation |
|-------|------|--------|-------------|----------------|-----------------|
| **Coils** | 1 bit | Read/Write | Digital outputs, relays | 0x01, 0x05, 0x0F | **0**xxxx (00001-09999) |
| **Discrete Inputs** | 1 bit | Read-only | Digital inputs, switches | 0x02 | **1**xxxx (10001-19999) |
| **Input Registers** | 16 bits | Read-only | Measurements, sensors | 0x04 | **3**xxxx (30001-39999) |
| **Holding Registers** | 16 bits | Read/Write | Configuration, setpoints | 0x03, 0x06, 0x10, 0x16, 0x17 | **4**xxxx (40001-49999) |

**Critical point:** These are **separate address spaces**. Address 0 exists in all four tables independently.

## MODBUS Data Model Overview

### The Four Independent Tables

MODBUS defines four types of data objects, each with its own address space:

```
┌─────────────────────────────────────────────────────────────┐
│                    MODBUS Data Model                         │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  Coils (1-bit Read/Write)                                    │
│  Address 0-65535 → Digital Outputs, Relays                   │
│  ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐                 │
│  │ 0 │ 1 │ 2 │ 3 │ 4 │...│   │   │   │65k│                 │
│  └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘                 │
│                                                               │
├─────────────────────────────────────────────────────────────┤
│  Discrete Inputs (1-bit Read-Only)                           │
│  Address 0-65535 → Digital Inputs, Switches                  │
│  ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐                 │
│  │ 0 │ 1 │ 2 │ 3 │ 4 │...│   │   │   │65k│                 │
│  └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘                 │
│                                                               │
├─────────────────────────────────────────────────────────────┤
│  Input Registers (16-bit Read-Only)                          │
│  Address 0-65535 → Measurements, Sensor Values               │
│  ┌─────┬─────┬─────┬─────┬─────┬───────┬─────┐             │
│  │  0  │  1  │  2  │  3  │ ... │       │ 65k │             │
│  └─────┴─────┴─────┴─────┴─────┴───────┴─────┘             │
│                                                               │
├─────────────────────────────────────────────────────────────┤
│  Holding Registers (16-bit Read/Write)                       │
│  Address 0-65535 → Configuration, Setpoints                  │
│  ┌─────┬─────┬─────┬─────┬─────┬───────┬─────┐             │
│  │  0  │  1  │  2  │  3  │ ... │       │ 65k │             │
│  └─────┴─────┴─────┴─────┴─────┴───────┴─────┘             │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

**Important:** Address 100 in "Holding Registers" is completely separate from address 100 in "Input Registers". They are distinct memory locations accessed by different function codes.

Source: [holding-registers.md](/wiki/concepts/holding-registers.md:20)

### Register Types Explained

#### 1. Coils (0xxxx)

**Properties:**
- **Size:** 1 bit (binary: 0 or 1)
- **Access:** Read/Write
- **Typical use:** Digital outputs, relays, indicator lights
- **Address range:** 0-65535
- **Value range:** ON (1) or OFF (0)

**Function codes:**
- **0x01:** Read Coils (read status)
- **0x05:** Write Single Coil (write one coil)
- **0x0F (15):** Write Multiple Coils (write multiple coils)

**Common applications:**
- Motor start/stop
- Valve open/close
- Pump on/off
- Indicator lights
- Relay control

**Example:**
```
Turn on relay at coil address 5
Write Single Coil: Address=5, Value=0xFF00 (ON)
```

Source: [coils.md](/wiki/concepts/coils.md:16)

#### 2. Discrete Inputs (1xxxx)

**Properties:**
- **Size:** 1 bit (binary: 0 or 1)
- **Access:** Read-only
- **Typical use:** Digital inputs, switches, limit sensors
- **Address range:** 0-65535
- **Value range:** ON (1) or OFF (0)

**Function codes:**
- **0x02:** Read Discrete Inputs (read-only)

**Common applications:**
- Limit switches
- Push buttons
- Proximity sensors
- Safety interlocks
- Status indicators
- Photoelectric sensors

**Example:**
```
Read limit switch at discrete input address 10
Read Discrete Inputs: Address=10, Quantity=1
```

Source: [discrete-inputs.md](/wiki/concepts/discrete-inputs.md:16)

#### 3. Input Registers (3xxxx)

**Properties:**
- **Size:** 16 bits (2 bytes)
- **Access:** Read-only
- **Typical use:** Measurements, sensor values, analog inputs
- **Address range:** 0-65535
- **Value range:** 0-65535 (unsigned) or -32768 to 32767 (signed)
- **Byte order:** Big-endian (MSB first)

**Function codes:**
- **0x04:** Read Input Registers (read-only)

**Common applications:**
- Temperature measurements
- Pressure readings
- Flow rates
- Level measurements
- Voltage/current readings
- Speed measurements (RPM)
- Energy readings (kWh)
- Position feedback

**Example:**
```
Read temperature sensor at input register 100
Read Input Registers: Address=100, Quantity=1
Response: 0x09C4 = 2500 → 25.00°C (with scale factor 100)
```

Source: [input-registers.md](/wiki/concepts/input-registers.md:16)

#### 4. Holding Registers (4xxxx)

**Properties:**
- **Size:** 16 bits (2 bytes)
- **Access:** Read/Write
- **Typical use:** Configuration, setpoints, control values
- **Address range:** 0-65535
- **Value range:** 0-65535 (unsigned) or -32768 to 32767 (signed)
- **Byte order:** Big-endian (MSB first)

**Function codes:**
- **0x03:** Read Holding Registers
- **0x06:** Write Single Register
- **0x10 (16):** Write Multiple Registers
- **0x16 (22):** Mask Write Register
- **0x17 (23):** Read/Write Multiple Registers

**Common applications:**
- Temperature setpoints
- Speed references
- Configuration parameters
- PID parameters (Kp, Ki, Kd)
- Alarm limits
- Calibration values
- Mode selection
- Timer values

**Example:**
```
Set temperature setpoint to 50.0°C at holding register 200
Write Single Register: Address=200, Value=5000 (with scale factor 100)
```

Source: [holding-registers.md](/wiki/concepts/holding-registers.md:16)

## Protocol Addressing: 0-Based vs 1-Based

### Critical Distinction

**MODBUS protocol uses 0-based addressing** in the actual MODBUS frames sent over the wire.

**Device documentation typically uses 1-based addressing** for human readability.

### The Addressing Confusion

**In MODBUS PDU (Protocol Data Unit):**
- Addresses start at **0**
- Address range: **0-65535** (0x0000-0xFFFF)
- This is what goes in the actual MODBUS frame

**In device documentation:**
- Addresses often start at **1** (or 00001, 10001, 30001, 40001)
- This is for human readability
- **You must subtract 1 to get the PDU address**

### Conversion Table

| Documentation Says | PDU Address (Wire) | Calculation |
|-------------------|-------------------|-------------|
| Register 1 | 0 | 1 - 1 = 0 |
| Register 10 | 9 | 10 - 1 = 9 |
| Register 100 | 99 | 100 - 1 = 99 |
| Register 1000 | 999 | 1000 - 1 = 999 |

**Example confusion:**

```
Device manual: "Temperature is in register 100"
Your code should use: Address = 99

Device manual: "First register is address 1"
Your code should use: Address = 0
```

### Why 0-Based?

MODBUS uses 0-based addressing because:
1. **C programming convention:** Arrays in C start at 0
2. **Binary efficiency:** 0x0000-0xFFFF = 65536 addresses
3. **Standard practice:** Most binary protocols use 0-based indexing

### Implementation Rule

**Always use 0-based addresses in your code.**

```c
// Device manual says "Temperature at register 100"
uint16_t address = 99;  // Subtract 1 for PDU address
read_holding_registers(device, address, 1);
```

Source: [holding-registers.md](/wiki/concepts/holding-registers.md:38)

## Legacy Addressing Notation (5-Digit Addresses)

### The 5-Digit Convention

MODBUS documentation often uses a **5-digit addressing convention** to indicate both the register type and the register number:

```
[Table Type Digit][Register Number 0001-9999]
```

### The Four Address Ranges

| Notation Range | Register Type | PDU Address Calculation | Example |
|---------------|---------------|------------------------|---------|
| **00001-09999** | Coils | Address = Number - 1 | 00005 → Coil 4 (PDU address 4) |
| **10001-19999** | Discrete Inputs | Address = Number - 10001 | 10100 → Discrete Input 99 (PDU address 99) |
| **30001-39999** | Input Registers | Address = Number - 30001 | 30050 → Input Register 49 (PDU address 49) |
| **40001-49999** | Holding Registers | Address = Number - 40001 | 40100 → Holding Register 99 (PDU address 99) |

### Examples

**Example 1: Holding Register 40001**
```
Documentation: "Temperature setpoint at address 40001"
Register type: Holding Register (starts with 4)
Register number: 1
PDU address: 40001 - 40001 = 0
Your code: read_holding_registers(address=0, count=1)
```

**Example 2: Input Register 30100**
```
Documentation: "Current measurement at address 30100"
Register type: Input Register (starts with 3)
Register number: 100
PDU address: 30100 - 30001 = 99
Your code: read_input_registers(address=99, count=1)
```

**Example 3: Coil 00050**
```
Documentation: "Pump control at address 00050"
Register type: Coil (starts with 0)
Coil number: 50
PDU address: 50 - 1 = 49
Your code: write_single_coil(address=49, value=ON)
```

**Example 4: Discrete Input 10001**
```
Documentation: "Emergency stop button at address 10001"
Register type: Discrete Input (starts with 1)
Input number: 1
PDU address: 10001 - 10001 = 0
Your code: read_discrete_inputs(address=0, count=1)
```

### Confusion Point: "Register 1" vs "40001"

**Same register, two notations:**

```
Device manual might say EITHER:
- "Holding register 1"
- "Register 40001"

Both mean:
- PDU Address: 0
- Function Code: 0x03 (Read Holding Registers)
```

### Why This Notation Exists

**Historical reasons:**
1. **Pre-dates modern documentation:** Created when displays were limited
2. **Type identification:** First digit instantly shows register type
3. **Legacy compatibility:** Many old devices still use this notation

**Modern trend:** Newer documentation often uses simpler notation (e.g., "HR100" for Holding Register 100) but legacy notation persists.

Source: [holding-registers.md](/wiki/concepts/holding-registers.md:35)

## Address Space Organization Summary

### Complete Address Map

```
MODBUS Address Space (per device)

Coils (Bit Read/Write):
┌──────────────────────────────────────────┐
│ 0-65535 (0x0000-0xFFFF)                  │ Function: 0x01, 0x05, 0x0F
│ Legacy: 00001-09999 (typically)          │ Access: Read/Write
└──────────────────────────────────────────┘

Discrete Inputs (Bit Read-Only):
┌──────────────────────────────────────────┐
│ 0-65535 (0x0000-0xFFFF)                  │ Function: 0x02
│ Legacy: 10001-19999 (typically)          │ Access: Read-only
└──────────────────────────────────────────┘

Input Registers (Word Read-Only):
┌──────────────────────────────────────────┐
│ 0-65535 (0x0000-0xFFFF)                  │ Function: 0x04
│ Legacy: 30001-39999 (typically)          │ Access: Read-only
└──────────────────────────────────────────┘

Holding Registers (Word Read/Write):
┌──────────────────────────────────────────┐
│ 0-65535 (0x0000-0xFFFF)                  │ Function: 0x03, 0x06, 0x10, 0x16, 0x17
│ Legacy: 40001-49999 (typically)          │ Access: Read/Write
└──────────────────────────────────────────┘
```

### Key Observations

1. **Separate spaces:** All four tables have addresses 0-65535 independently
2. **Function code determines table:** The function code in the MODBUS request specifies which table to access
3. **Address 0 exists in all tables:** Coil 0, Discrete Input 0, Input Register 0, Holding Register 0 are all different
4. **Theoretical max 65536 per table:** In practice, devices implement much smaller ranges

## Function Codes for Each Register Type

### Complete Function Code Mapping

| Function Code | Hex | Name | Register Type | Operation | Max Quantity |
|--------------|-----|------|---------------|-----------|--------------|
| **01** | 0x01 | Read Coils | Coils | Read | 2000 bits |
| **02** | 0x02 | Read Discrete Inputs | Discrete Inputs | Read | 2000 bits |
| **03** | 0x03 | Read Holding Registers | Holding Registers | Read | 125 registers |
| **04** | 0x04 | Read Input Registers | Input Registers | Read | 125 registers |
| **05** | 0x05 | Write Single Coil | Coils | Write | 1 bit |
| **06** | 0x06 | Write Single Register | Holding Registers | Write | 1 register |
| **15** | 0x0F | Write Multiple Coils | Coils | Write | 1968 bits |
| **16** | 0x10 | Write Multiple Registers | Holding Registers | Write | 123 registers |
| **22** | 0x16 | Mask Write Register | Holding Registers | Modify | 1 register |
| **23** | 0x17 | Read/Write Multiple Registers | Holding Registers | Read+Write | R:125, W:121 |

### How Function Codes Select Tables

**The function code determines which table is accessed:**

```
Request: Read address 100, quantity 1

Function Code 0x01 → Reads COIL 100
Function Code 0x02 → Reads DISCRETE INPUT 100  
Function Code 0x03 → Reads HOLDING REGISTER 100
Function Code 0x04 → Reads INPUT REGISTER 100

Same address (100), different tables!
```

**This is why you never need to specify "register type" in the address - the function code already specifies it.**

Source: [function-codes.md](/wiki/concepts/function-codes.md:16)

## Practical Addressing Examples

### Example 1: Reading a Temperature Sensor

**Device manual says:**
```
Temperature sensor: Input Register 30101
Scale factor: 100 (2500 = 25.00°C)
```

**Your code:**
```python
# Parse the notation
# "30101" → Input Register (starts with 3)
# Register number: 101
# PDU address: 30101 - 30001 = 100

address = 100  # PDU address (0-based)
quantity = 1
function_code = 0x04  # Read Input Registers

response = client.read_input_registers(address, quantity)
raw_value = response.registers[0]  # e.g., 2500
temperature = raw_value / 100.0    # → 25.00°C
```

### Example 2: Setting a Temperature Setpoint

**Device manual says:**
```
Temperature setpoint: Holding Register 40201
Scale factor: 100 (5000 = 50.00°C)
Range: 0-10000 (0-100.00°C)
```

**Your code:**
```python
# Parse the notation
# "40201" → Holding Register (starts with 4)
# Register number: 201
# PDU address: 40201 - 40001 = 200

address = 200  # PDU address (0-based)
setpoint_celsius = 50.0
raw_value = int(setpoint_celsius * 100)  # 50.0 → 5000
function_code = 0x06  # Write Single Register

client.write_single_register(address, raw_value)
```

### Example 3: Controlling a Pump

**Device manual says:**
```
Pump control: Coil 00010
ON = pump running, OFF = pump stopped
```

**Your code:**
```python
# Parse the notation
# "00010" → Coil (starts with 0)
# Coil number: 10
# PDU address: 10 - 1 = 9

address = 9  # PDU address (0-based)
function_code = 0x05  # Write Single Coil

# Start pump
client.write_single_coil(address, True)  # or 0xFF00

# Stop pump
client.write_single_coil(address, False)  # or 0x0000
```

### Example 4: Reading Multiple Sensors

**Device manual says:**
```
Temperature sensors: Input Registers 30001-30010 (10 sensors)
Each sensor: Scale factor 10 (250 = 25.0°C)
```

**Your code:**
```python
# Parse the notation
# "30001-30010" → Input Registers
# Start register: 30001 → PDU address 0
# End register: 30010 → PDU address 9
# Quantity: 10 registers

start_address = 0  # PDU address for 30001
quantity = 10
function_code = 0x04  # Read Input Registers

response = client.read_input_registers(start_address, quantity)

temperatures = []
for raw_value in response.registers:
    temp_celsius = raw_value / 10.0
    temperatures.append(temp_celsius)

# temperatures = [25.0, 26.5, 24.8, ...]
```

### Example 5: Mixed Register Types

**Device manual says:**
```
Pump 1 status: Discrete Input 10001 (running/stopped)
Pump 1 control: Coil 00001 (start/stop)
Pump 1 speed: Input Register 30001 (RPM)
Pump 1 speed setpoint: Holding Register 40001 (RPM)
```

**Your code:**
```python
# Check if pump is running
status_address = 0  # 10001 - 10001 = 0
status = client.read_discrete_inputs(status_address, 1)
is_running = status.bits[0]

# Read current speed
speed_address = 0  # 30001 - 30001 = 0
speed = client.read_input_registers(speed_address, 1)
current_rpm = speed.registers[0]

# Set speed setpoint
setpoint_address = 0  # 40001 - 40001 = 0
target_rpm = 1500
client.write_single_register(setpoint_address, target_rpm)

# Start pump
control_address = 0  # 00001 - 1 = 0
client.write_single_coil(control_address, True)
```

**Note:** All addresses are 0 because they're the first in each table type!

## Common Addressing Pitfalls

### Pitfall 1: Using Documentation Address in Code

**Wrong:**
```python
# Device manual says "Holding Register 40100"
address = 40100  # WRONG!
client.read_holding_registers(address, 1)
# Will read register 40100, not register 100!
```

**Correct:**
```python
# Device manual says "Holding Register 40100"
address = 40100 - 40001  # = 99 (PDU address)
client.read_holding_registers(address, 1)
```

### Pitfall 2: Forgetting Table Independence

**Wrong assumption:**
```python
# "Register 100 contains temperature"
# Which register? Holding or Input?

# This might be wrong if it's Input Register 100
client.read_holding_registers(100, 1)
```

**Correct:**
```python
# Check device manual for register type
# If it says "Input Register 30101" (PDU address 100)
client.read_input_registers(100, 1)

# If it says "Holding Register 40101" (PDU address 100)
client.read_holding_registers(100, 1)
```

### Pitfall 3: 1-Based Thinking

**Wrong:**
```python
# Device manual: "Registers 1-10 contain sensor data"
# Reading registers 1-10 (thinking 1-based)
for i in range(1, 11):
    value = client.read_holding_registers(i, 1)
# This reads PDU addresses 1-10, missing address 0!
```

**Correct:**
```python
# Device manual: "Registers 1-10 contain sensor data"
# Convert to 0-based PDU addresses
for i in range(0, 10):  # PDU addresses 0-9
    value = client.read_holding_registers(i, 1)
# Or read all at once:
values = client.read_holding_registers(0, 10)
```

### Pitfall 4: Confusing Notation Styles

**Device manual uses mixed notation:**
```
Temperature: Register 30001
Setpoint: HR100
Pump: Coil 5
```

**Interpretation:**
```python
# "Register 30001" → Input Register, PDU address 0
temp = client.read_input_registers(0, 1)

# "HR100" → Holding Register 100, PDU address 99
setpoint = client.read_holding_registers(99, 1)

# "Coil 5" → Usually means PDU address 5 (already 0-based)
# But could mean 00005 (PDU address 4)
# Check device manual for clarification!
client.write_single_coil(5, True)  # Assuming direct PDU address
```

## Addressing Best Practices

### 1. Always Convert to PDU Addresses

**Create a conversion function:**

```python
def parse_modbus_address(doc_address):
    """Convert documentation address to PDU address and register type"""
    if doc_address >= 40001 and doc_address <= 49999:
        return ('holding_register', doc_address - 40001)
    elif doc_address >= 30001 and doc_address <= 39999:
        return ('input_register', doc_address - 30001)
    elif doc_address >= 10001 and doc_address <= 19999:
        return ('discrete_input', doc_address - 10001)
    elif doc_address >= 1 and doc_address <= 9999:
        return ('coil', doc_address - 1)
    else:
        raise ValueError(f"Invalid MODBUS address: {doc_address}")

# Usage
reg_type, pdu_addr = parse_modbus_address(40100)
# → ('holding_register', 99)
```

### 2. Document Your Addresses

```python
# Bad: Magic numbers
value = client.read_holding_registers(99, 1)

# Good: Named constants with explanation
TEMP_SETPOINT_ADDR = 99  # Manual: Register 40100
value = client.read_holding_registers(TEMP_SETPOINT_ADDR, 1)

# Better: Document register type and scale
class DeviceRegisters:
    # Holding Registers (manual uses 40xxx notation)
    TEMP_SETPOINT = 99  # 40100 - Scale: 100 (2500 = 25.00°C)
    SPEED_SETPOINT = 199  # 40200 - Scale: 1 (direct RPM)
    
    # Input Registers (manual uses 30xxx notation)
    TEMP_SENSOR = 0  # 30001 - Scale: 100
    SPEED_FEEDBACK = 100  # 30101 - Scale: 1
```

### 3. Validate Address Ranges

```python
def read_holding_register_safe(client, address, count=1):
    """Read holding register with validation"""
    if address < 0 or address > 65535:
        raise ValueError(f"Address out of range: {address}")
    if count < 1 or count > 125:
        raise ValueError(f"Invalid count: {count}")
    if address + count > 65536:
        raise ValueError(f"Address + count exceeds limit")
    
    return client.read_holding_registers(address, count)
```

### 4. Use Type-Safe Wrappers

```python
class ModbusDevice:
    def __init__(self, client):
        self.client = client
    
    def read_holding(self, doc_address):
        """Read using documentation address (40xxx)"""
        if doc_address < 40001 or doc_address > 49999:
            raise ValueError("Use 40001-49999 for holding registers")
        pdu_address = doc_address - 40001
        return self.client.read_holding_registers(pdu_address, 1)
    
    def read_input(self, doc_address):
        """Read using documentation address (30xxx)"""
        if doc_address < 30001 or doc_address > 39999:
            raise ValueError("Use 30001-39999 for input registers")
        pdu_address = doc_address - 30001
        return self.client.read_input_registers(pdu_address, 1)

# Usage
device = ModbusDevice(client)
temp = device.read_input(30001)  # Automatic conversion
setpoint = device.read_holding(40100)  # Automatic conversion
```

## Quick Reference Cards

### Addressing Conversion Cheat Sheet

| You See in Manual | Register Type | Subtract | PDU Address | Function Code |
|------------------|---------------|----------|-------------|---------------|
| 00001-09999 | Coil | 1 | 0-9998 | 0x01, 0x05, 0x0F |
| 10001-19999 | Discrete Input | 10001 | 0-8998 | 0x02 |
| 30001-39999 | Input Register | 30001 | 0-8998 | 0x04 |
| 40001-49999 | Holding Register | 40001 | 0-8998 | 0x03, 0x06, 0x10 |

### Data Type Summary

| Type | Bits | Values | Read/Write | Typical Use |
|------|------|--------|------------|-------------|
| **Coil** | 1 | ON/OFF | R/W | Digital outputs |
| **Discrete Input** | 1 | ON/OFF | R only | Digital inputs |
| **Input Register** | 16 | 0-65535 | R only | Measurements |
| **Holding Register** | 16 | 0-65535 | R/W | Configuration |

## Related Pages

- [Holding Registers](/wiki/concepts/holding-registers.md) - Detailed holding register information
- [Input Registers](/wiki/concepts/input-registers.md) - Detailed input register information
- [Coils](/wiki/concepts/coils.md) - Detailed coil information
- [Discrete Inputs](/wiki/concepts/discrete-inputs.md) - Detailed discrete input information
- [Function Codes](/wiki/concepts/function-codes.md) - Complete function code reference
- [MODBUS Errors and Exception Responses](/wiki/answers/modbus-errors-and-exceptions.md) - Addressing error diagnosis

## Backlinks

No backlinks (yet)

------------

MODBUS is a trademark of the Modbus Organization, Inc.

All information on this page is derived by summarizing and analyzing solely the MODBUS specifications published by the Modbus Organization, Inc.  The specifications are copyright by the Modbus Organization.

By deriving this information solely from the specifications we hope to stay true to the specification.

No infringement is intended.

None yet.
