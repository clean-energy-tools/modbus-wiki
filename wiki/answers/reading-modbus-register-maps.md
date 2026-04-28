---
title: Reading MODBUS Register Maps
Summary: Comprehensive guide to reading and interpreting MODBUS register maps including addressing conventions, register types, function codes, data types, scaling factors, and handling vendor-specific documentation inconsistencies.
Sources:
  - /raw/MODBUS/modbusprotocolspecification.md
  - /raw/MODBUS/MODBUS.md
Categories:
  - register-maps
  - documentation
  - addressing
  - data-types
  - best-practices
type: answer
date-created: 2026-04-24T16:30:00+03:00
last-updated: 2026-04-24T16:30:00+03:00
---

Reading MODBUS register maps is often more complicated than it should be because of inconsistent vendor documentation, multiple addressing systems, unclear data types, and missing information. This guide explains how to understand MODBUS register maps and find the information you need for your code.

## Quick Answer: What You Need From a Register Map

At minimum, a register map should tell you:

1. **Register address** (the actual address to use in your code, not old-style notation)
2. **Register name** (what it's called)
3. **Register type** (Coil, Discrete Input, Holding Register, or Input Register)
4. **Function code** to use (0x01, 0x03, 0x04, etc.)
5. **Data type** (like unsigned 16-bit, signed 16-bit, 32-bit float, etc.)
6. **Can you write to it?** (Read-only or Read/Write)
7. **Units and scaling** (if the value needs to be divided or multiplied)
8. **What it means** (description of its purpose)

However, vendors rarely provide all this information clearly, so you often need to do detective work to figure it out.

## The MODBUS Addressing Confusion

### Two Different Address Systems

**The Core Problem:** MODBUS has TWO completely different ways to write addresses, and vendors mix them inconsistently in their documentation.

#### Protocol Addresses (What You Actually Use in Your Code)

**MODBUS protocol addresses are straightforward:**
- All addresses start at 0
- Addresses are numbers from 0 to 65,535
- This is what you actually put in your MODBUS messages

**Example:**
```
Read Holding Registers (function 0x03):
  Address: 0
  Quantity: 10
```
This reads holding registers 0 through 9.

(source: [modbusprotocolspecification.md](/raw/MODBUS/modbusprotocolspecification.md))

#### Convention Addresses (Old Documentation Style)

**The "Modicon Convention" from old PLCs:**

This old system embedded the register type into the address number itself:

| Address Range | Register Type | How to Get Protocol Address |
|---------------|---------------|------------------------------|
| 00001-09999 | Coils | Subtract 1 |
| 10001-19999 | Discrete Inputs | Subtract 10001 |
| 30001-39999 | Input Registers | Subtract 30001 |
| 40001-49999 | Holding Registers | Subtract 40001 |

**Example - Convention Address 40001:**
- Register type: Holding Register (starts with 4)
- Protocol address: 40001 - 40001 = **0**
- Function code to use: 0x03 (Read Holding Registers)

**Example - Convention Address 30108:**
- Register type: Input Register (starts with 3)
- Protocol address: 30108 - 30001 = **107**
- Function code to use: 0x04 (Read Input Registers)

**Why this old system exists:** Old Modicon PLCs displayed addresses this way on their screens. Modern devices don't use this system, but documentation often still does for historical reasons.

### How to Identify Which Convention Is Used

**Look for these clues in the register map:**

#### Signs of Protocol Addressing (0-based):

✅ **Addresses start at 0**
```
Address | Name           | Type
--------|----------------|------------------
0       | Device ID      | Holding Register
1       | Firmware Ver   | Holding Register
```

✅ **Explicit register type column**
```
Address | Type            | Name
--------|-----------------|-------------
0       | Holding Reg     | Device ID
100     | Input Reg       | Temperature
```

✅ **Function code specified**
```
Address | FC   | Name
--------|------|-------------
0       | 0x03 | Device ID
100     | 0x04 | Temperature
```

✅ **Documentation says "Protocol Address" or "PDU Address"**

#### Signs of Convention Addressing (Modicon):

❌ **Addresses start at 40001, 30001, etc.**
```
Address | Name           | Type
--------|----------------|------------------
40001   | Device ID      | Holding Register
40002   | Firmware Ver   | Holding Register
```

❌ **All addresses are 5-digit numbers**
```
Address | Name
--------|-------------
40001   | Device ID
40002   | Firmware Ver
30001   | Temperature
```

❌ **Address ranges jump by 10000s**
```
Holding Registers: 40001-40100
Input Registers:   30001-30050
Coils:            00001-00064
```

❌ **Documentation says "Modbus Address" or "Register Number"** (ambiguous)

### Converting Convention to Protocol Address

**If you see convention addresses (40001, 30001, etc.):**

**Step 1: Identify register type from first digit**
- 0xxxx: Coil
- 1xxxx: Discrete Input
- 3xxxx: Input Register
- 4xxxx: Holding Register

**Step 2: Subtract base to get protocol address**

| Convention Address | Register Type | Protocol Address |
|-------------------|---------------|------------------|
| 40001 | Holding Register | 0 |
| 40002 | Holding Register | 1 |
| 40123 | Holding Register | 122 |
| 30001 | Input Register | 0 |
| 30050 | Input Register | 49 |
| 00001 | Coil | 0 |
| 00100 | Coil | 99 |
| 10001 | Discrete Input | 0 |

**Formula:**
```
Protocol Address = Convention Address - Base

Where Base is:
  00001 for Coils
  10001 for Discrete Inputs
  30001 for Input Registers
  40001 for Holding Registers
```

**Example Translation:**

**Vendor documentation says:**
```
Register 40108: Setpoint Temperature (Float32, R/W)
Register 30005: Current Temperature (Float32, R)
```

**Your code should use:**
```python
# Setpoint (Holding Register 40108 → Protocol Address 107)
client.write_registers(address=107, values=[high_word, low_word], unit=1)

# Current (Input Register 30005 → Protocol Address 4)
response = client.read_input_registers(address=4, count=2, unit=1)
```

### 0-Based vs 1-Based Addressing

**Another complication:** Some vendors use 1-based addresses even for protocol addressing.

**0-based (standard):**
```
Address | Name
--------|-------------
0       | First register
1       | Second register
2       | Third register
```

**1-based (non-standard):**
```
Address | Name
--------|-------------
1       | First register
2       | Second register
3       | Third register
```

**How to determine:**

1. **Check documentation explicitly** - Look for statements like:
   - "Addresses start at 0" (0-based)
   - "Addresses start at 1" (1-based)
   - "First register is address 0" (0-based)
   - "Register 1 is the first register" (1-based)

2. **Test with known registers:**
   - Try reading address 0
   - If error 0x02 (ILLEGAL DATA ADDRESS): Device uses 1-based
   - If success: Device uses 0-based

3. **Look at register map range:**
   - "Registers 0-999" → 0-based
   - "Registers 1-1000" → 1-based

**When using 1-based devices:**
```python
# Documentation says "Register 1" (1-based)
# Your code uses protocol address 1 (not 0)
response = client.read_holding_registers(address=1, count=1, unit=1)
```

**Important:** If vendor documentation uses convention addresses (40001), they are NOT using 1-based protocol addressing. Convert to 0-based protocol address as shown above.

## Identifying Register Types and Function Codes

### Register Types

MODBUS has 4 distinct register types in 2 categories:

**Discrete (1-bit) Values:**

| Type | Size | Access | Typical Use | Function Codes |
|------|------|--------|-------------|----------------|
| Coil | 1 bit | Read/Write | Digital outputs, control bits | 0x01 (read), 0x05 (write single), 0x0F (write multiple) |
| Discrete Input | 1 bit | Read-only | Digital inputs, status bits | 0x02 (read) |

**Register (16-bit) Values:**

| Type | Size | Access | Typical Use | Function Codes |
|------|------|--------|-------------|----------------|
| Holding Register | 16 bit | Read/Write | Configuration, setpoints, control | 0x03 (read), 0x06 (write single), 0x10 (write multiple) |
| Input Register | 16 bit | Read-only | Measurements, sensor readings, status | 0x04 (read) |

(source: [modbusprotocolspecification.md](/raw/MODBUS/modbusprotocolspecification.md))

### How to Identify Register Type in Documentation

**Look for explicit type column:**
```
Address | Type            | Name           | Access
--------|-----------------|----------------|-------
0       | Holding Reg     | Setpoint       | R/W
1       | Input Reg       | Temperature    | R
100     | Coil            | Enable Output  | R/W
200     | Discrete Input  | Alarm Status   | R
```

**Infer from access permissions:**
- Read/Write + 16-bit → Likely Holding Register
- Read-only + 16-bit → Likely Input Register
- Read/Write + 1-bit → Likely Coil
- Read-only + 1-bit → Likely Discrete Input

**Infer from function code:**
```
Address | FC   | Name
--------|------|-------------
0       | 0x03 | Setpoint       → Holding Register
1       | 0x04 | Temperature    → Input Register
100     | 0x01 | Enable         → Coil
200     | 0x02 | Alarm          → Discrete Input
```

**Infer from convention address (if used):**
- 4xxxx → Holding Register
- 3xxxx → Input Register
- 0xxxx → Coil
- 1xxxx → Discrete Input

**Infer from usage description:**
- "Configuration", "Setpoint", "Control" → Holding Register
- "Measurement", "Sensor", "Reading" → Input Register
- "Output", "Relay", "Switch" → Coil
- "Input", "Status", "Alarm" → Discrete Input

### Function Code Selection

**Once you know the register type, select the appropriate function code:**

**For Reading:**

| Register Type | Function Code | Function Name |
|---------------|---------------|---------------|
| Coil | 0x01 | Read Coils |
| Discrete Input | 0x02 | Read Discrete Inputs |
| Holding Register | 0x03 | Read Holding Registers |
| Input Register | 0x04 | Read Input Registers |

**For Writing:**

| Register Type | Operation | Function Code | Function Name |
|---------------|-----------|---------------|---------------|
| Coil | Write single | 0x05 | Write Single Coil |
| Coil | Write multiple | 0x0F | Write Multiple Coils |
| Holding Register | Write single | 0x06 | Write Single Register |
| Holding Register | Write multiple | 0x10 | Write Multiple Registers |

**Note:** Input Registers and Discrete Inputs are read-only. Attempting to write them will return exception 0x01 (ILLEGAL FUNCTION).

**If documentation lists function code:**
- Use what's specified
- Device may not support all standard functions

**If documentation doesn't list function code:**
- Use standard function for that register type
- Try reading first before writing

(source: [Function Codes](/wiki/concepts/function-codes.md))

## Decoding Data Types

### Common MODBUS Data Types

MODBUS registers are 16 bits, but multi-register data types are common:

**Single-Register Types (16-bit):**

| Data Type | Size | Range | Description |
|-----------|------|-------|-------------|
| UINT16 | 1 reg | 0 to 65535 | Unsigned 16-bit integer |
| INT16 | 1 reg | -32768 to 32767 | Signed 16-bit integer |
| BOOL/BIT | 1 bit | 0 or 1 | Boolean (in coil/discrete input) |
| BIT FIELD | 1 reg | - | 16 individual bits with separate meanings |

**Multi-Register Types:**

| Data Type | Size | Range | Description |
|-----------|------|-------|-------------|
| UINT32 | 2 regs | 0 to 4,294,967,295 | Unsigned 32-bit integer |
| INT32 | 2 regs | -2,147,483,648 to 2,147,483,647 | Signed 32-bit integer |
| FLOAT32 | 2 regs | ±3.4×10³⁸ | IEEE 754 single-precision float |
| UINT64 | 4 regs | 0 to 2⁶⁴-1 | Unsigned 64-bit integer |
| INT64 | 4 regs | -2⁶³ to 2⁶³-1 | Signed 64-bit integer |
| FLOAT64 | 4 regs | ±1.7×10³⁰⁸ | IEEE 754 double-precision float |
| STRING | N regs | - | ASCII text (2 characters per register) |

(source: [MODBUS Register Data Representation](/wiki/answers/modbus-register-data-representation.md))

### How Data Types Appear in Documentation

**Clear documentation (ideal):**
```
Address | Name        | Type    | Size | Access | Units
--------|-------------|---------|------|--------|------
0       | Device ID   | UINT16  | 1    | R      | -
1       | Setpoint    | FLOAT32 | 2    | R/W    | °C
3       | Counter     | UINT32  | 2    | R      | pulses
5       | Serial Num  | STRING  | 8    | R      | ASCII
```

**Ambiguous documentation (common):**
```
Address | Name        | Size | Description
--------|-------------|------|---------------------------
0       | Device ID   | 1    | Device identifier
1       | Setpoint    | 2    | Temperature setpoint (°C)
3       | Counter     | 2    | Pulse counter
5       | Serial Num  | 8    | Device serial number
```

### Inferring Data Types From Documentation

**When data type not explicitly stated:**

**1. Look at size (register count):**
- Size 1 → Likely UINT16 or INT16
- Size 2 → Likely UINT32, INT32, or FLOAT32
- Size 4 → Likely UINT64, INT64, or FLOAT64
- Size 8+ → Likely STRING

**2. Check for decimal values in range:**
- "Range: 0.0 to 100.0" → FLOAT32
- "Range: -40.0 to 85.0" → FLOAT32
- "Range: 0 to 65535" → UINT16
- "Range: -1000 to 1000" → INT16

**3. Examine units and precision:**
- "Temperature (°C), 0.1° resolution" → FLOAT32 or scaled INT16
- "Voltage (mV)" → UINT16 or UINT32
- "Serial number" → STRING or UINT32

**4. Look for signed/unsigned indicators:**
- "Unsigned" → UINT16, UINT32, UINT64
- "Signed" → INT16, INT32, INT64
- Negative values possible → Signed integer or float

**5. Check description for text:**
- "ASCII text", "String", "Text" → STRING
- "Device name", "Serial number" (if long) → STRING

### Handling Bit Fields

**Some registers pack multiple boolean values into bits:**

**Example documentation:**
```
Address: 10
Name: Status Register
Type: UINT16 (bit field)
Bits:
  Bit 0: Running (1=running, 0=stopped)
  Bit 1: Alarm (1=alarm active)
  Bit 2: Warning (1=warning active)
  Bit 3: Manual Mode (1=manual, 0=auto)
  Bits 4-15: Reserved
```

**Reading bit fields:**
```python
# Read register 10
status = client.read_holding_registers(address=10, count=1, unit=1).registers[0]

# Extract individual bits
running = (status & 0x0001) != 0  # Bit 0
alarm   = (status & 0x0002) != 0  # Bit 1
warning = (status & 0x0004) != 0  # Bit 2
manual  = (status & 0x0008) != 0  # Bit 3
```

**Writing bit fields (requires read-modify-write):**
```python
# Read current value
status = client.read_holding_registers(address=10, count=1, unit=1).registers[0]

# Set bit 3 (Manual Mode) to 1
status |= 0x0008

# Write back
client.write_register(address=10, value=status, unit=1)
```

## Scaling Factors and Units

### Why Scaling Exists

MODBUS registers are integers. To represent fractional values, vendors use scaling:

**Example:** Temperature sensor measures -40.0°C to 125.0°C with 0.1°C resolution

**Option 1: Use FLOAT32** (2 registers)
- Direct representation: 23.5°C stored as IEEE 754 float

**Option 2: Use scaled INT16** (1 register, more efficient)
- Store: temperature × 10
- 23.5°C → stored as 235
- Range: -400 to 1250
- Resolution: 0.1°C

(source: [MODBUS Register Data Representation](/wiki/answers/modbus-register-data-representation.md))

### Reading Scaling Information

**Look for these in documentation:**

**Explicit scaling factor:**
```
Address | Name        | Type   | Scaling | Units
--------|-------------|--------|---------|------
0       | Temperature | INT16  | 0.1     | °C
1       | Pressure    | UINT16 | 0.01    | bar
2       | Flow Rate   | UINT16 | 1.0     | L/min
```

**Formula in description:**
```
Address: 0
Name: Temperature
Type: INT16
Description: Temperature in °C. Value = (register / 10)
Range: -400 to 1250 (raw), -40.0 to 125.0°C (actual)
```

**Decimal point in range:**
```
Address | Name        | Range         | Units
--------|-------------|---------------|------
0       | Temperature | -40.0 to 125.0| °C
```
This implies scaling (INT16 can't store decimals directly).

### Applying Scaling Factors

**General formula:**
```
Actual Value = Raw Value × Scaling Factor
```

**Examples:**

**Reading:**
```python
# Documentation: Temperature (INT16, scaling 0.1, °C)
raw = client.read_holding_registers(address=0, count=1, unit=1).registers[0]

# Convert to signed if needed
if raw > 32767:
    raw -= 65536

actual_temp = raw * 0.1  # Result: 23.5°C if raw was 235
```

**Writing:**
```python
# Want to write 23.5°C
desired_temp = 23.5

# Convert to raw value
raw = int(desired_temp / 0.1)  # Result: 235

# Write
client.write_register(address=0, value=raw, unit=1)
```

**Common scaling factors:**
- 0.1 (divide by 10)
- 0.01 (divide by 100)
- 0.001 (divide by 1000)
- 10 (multiply by 10)
- 100 (multiply by 100)

### Units and Engineering Values

**Always check units in documentation:**

**Example - Power measurement:**
```
Address | Name  | Type   | Units
--------|-------|--------|------
10      | Power | UINT16 | W      ← Watts
10      | Power | UINT16 | kW     ← Kilowatts (different!)
10      | Power | UINT16 | mW     ← Milliwatts (different!)
```

**Common unit variations:**
- Temperature: °C, °F, K
- Pressure: Pa, kPa, bar, psi
- Flow: L/s, L/min, m³/h, GPM
- Power: W, kW, MW
- Voltage: V, mV, kV
- Current: A, mA

**Always convert to your application's required units.**

## Word Order (Endianness) for Multi-Register Values

### The Endianness Problem

Multi-register values (UINT32, FLOAT32, etc.) can be stored with different word orders.

**Example: 32-bit value 0x12345678**

**Big-Endian (High Word First):**
```
Register N:   0x1234 (high word)
Register N+1: 0x5678 (low word)
```

**Little-Endian (Low Word First):**
```
Register N:   0x5678 (low word)
Register N+1: 0x1234 (high word)
```

**There are actually 4 possible byte orders for 32-bit values:**
1. Big-Endian (ABCD): 0x12 0x34 0x56 0x78
2. Little-Endian (DCBA): 0x78 0x56 0x34 0x12
3. Big-Endian Byte Swap (BADC): 0x34 0x12 0x78 0x56
4. Little-Endian Byte Swap (CDAB): 0x56 0x78 0x12 0x34

(source: [MODBUS Register Data Representation](/wiki/answers/modbus-register-data-representation.md))

### Identifying Word Order From Documentation

**Look for explicit statement:**
```
"All multi-register values are stored in big-endian format"
"FLOAT32 values use little-endian word order"
"32-bit integers: high word first, low word second"
```

**Look for byte order diagram:**
```
FLOAT32 (2 registers):
Register N:   Bytes 0-1 (MSB first)
Register N+1: Bytes 2-3 (LSB last)
```

**If not documented:**
1. **Test with known value:** Read a register with documented expected value
2. **Try both orders:** See which gives sensible result
3. **Ask vendor:** Word order is critical for correct operation

### Dealing With Missing Information

**When register map is incomplete:**

1. **Contact vendor technical support**
   - Request complete register map
   - Ask specific questions about missing info
   - Request example read/write operations

2. **Test systematically:**
   - Read all documented addresses
   - Check which function codes work
   - Verify data types with known values
   - Test different word orders for multi-register values

3. **Use device configuration software:**
   - Vendor may provide configuration tool
   - Monitor tool's MODBUS traffic with Wireshark
   - Reverse-engineer addresses and data types

4. **Check for updates:**
   - Newer manual versions may have more detail
   - Firmware updates may change register map
   - Look for application notes or tech bulletins

## Practical Example: Decoding a Real Register Map

### Example Device Documentation

**Vendor provides this register map:**

```
Modbus Address | Description              | Size | R/W | Range
---------------|--------------------------|------|-----|------------------
40001          | Device Model             | 1    | R   | -
40002          | Firmware Version         | 1    | R   | -
40003          | Serial Number            | 8    | R   | ASCII
40011          | Temperature Setpoint     | 2    | R/W | 0.0 to 100.0°C
40013          | Current Temperature      | 2    | R   | -40.0 to 125.0°C
40015          | Control Mode             | 1    | R/W | 0=Auto, 1=Manual
40016          | Output Power             | 1    | R/W | 0-100 (%)
40017          | Alarm Status             | 1    | R   | Bit field
40018          | Runtime Hours            | 2    | R   | Hours
```

### Step-by-Step Decoding

**Step 1: Convert convention addresses to protocol addresses**

Convention addresses use 40xxx format, so these are Holding Registers.

| Convention | Protocol Address | Size | Description |
|------------|------------------|------|-------------|
| 40001 | 0 | 1 | Device Model |
| 40002 | 1 | 1 | Firmware Version |
| 40003 | 2 | 8 | Serial Number |
| 40011 | 10 | 2 | Temperature Setpoint |
| 40013 | 12 | 2 | Current Temperature |
| 40015 | 14 | 1 | Control Mode |
| 40016 | 15 | 1 | Output Power |
| 40017 | 16 | 1 | Alarm Status |
| 40018 | 17 | 2 | Runtime Hours |

**Step 2: Identify data types**

| Address | Size | Range/Description | Inferred Type |
|---------|------|-------------------|---------------|
| 0 | 1 | - | UINT16 (model number) |
| 1 | 1 | - | UINT16 (version, likely BCD encoded) |
| 2 | 8 | ASCII | STRING (16 characters) |
| 10 | 2 | 0.0 to 100.0°C | FLOAT32 or scaled INT32 |
| 12 | 2 | -40.0 to 125.0°C | FLOAT32 or scaled INT32 |
| 14 | 1 | 0 or 1 | UINT16 (enum) |
| 15 | 1 | 0-100 | UINT16 |
| 16 | 1 | Bit field | UINT16 (bit field) |
| 17 | 2 | Hours | UINT32 |

**Decimal ranges suggest floating point or scaling.**

**Step 3: Determine function codes**

All are Holding Registers → Use function 0x03 for reading

For writing:
- Single register (size 1) → Use 0x06
- Multiple registers (size 2+) → Use 0x10

**Step 4: Identify scaling (need to test or contact vendor)**

Temperatures show decimal points but data type not specified. Need to determine:
- Are they FLOAT32?
- Or scaled integers?

**Test approach:**
```python
# Read temperature setpoint (2 registers at address 10)
data = client.read_holding_registers(address=10, count=2, unit=1).registers

# Try as FLOAT32 big-endian
import struct
bytes_be = struct.pack('>HH', data[0], data[1])
temp_float_be = struct.unpack('>f', bytes_be)[0]

# Try as FLOAT32 little-endian
bytes_le = struct.pack('<HH', data[1], data[0])
temp_float_le = struct.unpack('<f', bytes_le)[0]

# Try as scaled INT32
raw_int32 = (data[0] << 16) | data[1]
temp_scaled = raw_int32 / 10.0  # Try different scale factors

print(f"FLOAT32 BE: {temp_float_be}")
print(f"FLOAT32 LE: {temp_float_le}")
print(f"Scaled INT32 (/10): {temp_scaled}")

# Whichever gives a reasonable temperature (0-100°C) is correct
```

**Step 5: Create working register map**

After testing, create a complete register map for your code:

```python
REGISTER_MAP = {
    # Device Info
    'device_model': {
        'address': 0,
        'count': 1,
        'type': 'UINT16',
        'function': 0x03,
        'access': 'R'
    },
    'firmware_version': {
        'address': 1,
        'count': 1,
        'type': 'UINT16',
        'function': 0x03,
        'access': 'R'
    },
    'serial_number': {
        'address': 2,
        'count': 8,
        'type': 'STRING',
        'function': 0x03,
        'access': 'R'
    },
    # Temperature
    'temperature_setpoint': {
        'address': 10,
        'count': 2,
        'type': 'FLOAT32',
        'word_order': 'big_endian',  # Determined from testing
        'function': 0x03,  # Read
        'write_function': 0x10,  # Write
        'access': 'R/W',
        'units': 'C',
        'range': (0.0, 100.0)
    },
    'current_temperature': {
        'address': 12,
        'count': 2,
        'type': 'FLOAT32',
        'word_order': 'big_endian',
        'function': 0x03,
        'access': 'R',
        'units': 'C',
        'range': (-40.0, 125.0)
    },
    # Control
    'control_mode': {
        'address': 14,
        'count': 1,
        'type': 'UINT16',
        'function': 0x03,
        'write_function': 0x06,
        'access': 'R/W',
        'values': {0: 'Auto', 1: 'Manual'}
    },
    'output_power': {
        'address': 15,
        'count': 1,
        'type': 'UINT16',
        'function': 0x03,
        'write_function': 0x06,
        'access': 'R/W',
        'units': '%',
        'range': (0, 100)
    },
    'alarm_status': {
        'address': 16,
        'count': 1,
        'type': 'BITFIELD',
        'function': 0x03,
        'access': 'R',
        'bits': {
            0: 'High Temperature Alarm',
            1: 'Low Temperature Alarm',
            2: 'Sensor Fault',
            3: 'Communication Error'
        }
    },
    'runtime_hours': {
        'address': 17,
        'count': 2,
        'type': 'UINT32',
        'word_order': 'big_endian',
        'function': 0x03,
        'access': 'R',
        'units': 'hours'
    }
}
```

## Common Register Map Documentation Problems

### Problem 1: Ambiguous "Size" Column

**Documentation:**
```
Address | Name        | Size | Description
--------|-------------|------|-------------
10      | Temperature | 2    | Temperature
```

**Question:** Does "Size 2" mean:
- 2 registers (32-bit value)?
- 2 bytes (16-bit value)?

**Solution:**
- Context: MODBUS documentation usually means registers
- But verify: Read 1 register vs 2 registers, see which works
- Check description for decimal values (suggests multi-register)

### Problem 2: Missing Word Order

**Documentation:**
```
Address | Name  | Type    | Access
--------|-------|---------|-------
10      | Value | FLOAT32 | R
```

**Missing:** Word order (big-endian vs little-endian)

**Solution:**
- Test both orders with known values
- Check for manufacturer standards (many use big-endian)
- Monitor vendor software with Wireshark

### Problem 3: Unclear Register Type

**Documentation:**
```
Address | Name        | R/W | Description
--------|-------------|-----|-------------
100     | Temperature | R   | Current temperature
```

**Question:** Is this a Holding Register or Input Register?

**Solution:**
- Try both function 0x03 and 0x04
- Read-only often suggests Input Register (0x04)
- But many vendors use Holding Registers for everything

### Problem 4: Undocumented Registers

**Documentation shows:**
```
Address 0-10: Device Info
Address 50-60: Configuration
```

**Missing:** Addresses 11-49

**Possibilities:**
- Reserved for future use
- Not applicable to this model
- Undocumented features
- Documentation incomplete

**Solution:**
- Don't access undocumented addresses
- May return exception or garbage
- Contact vendor for complete map

### Problem 5: BCD Encoded Values

**Some devices use BCD (Binary Coded Decimal) for version numbers:**

**Example:**
```
Address: 1
Name: Firmware Version
Value: 0x0123
```

**BCD interpretation:**
- 0x0123 = Version 1.23 (not decimal 291)
- Each nibble (4 bits) is a decimal digit

**Decoding BCD:**
```python
raw = 0x0123
major = (raw >> 8) & 0x0F
minor = (raw >> 4) & 0x0F
patch = raw & 0x0F
version = f"{major}.{minor}{patch}"  # "1.23"
```

### Problem 6: Gaps in Addressing

**Documentation:**
```
Address | Name
--------|-------------
0       | Register A
1       | Register B
5       | Register C
```

**Addresses 2-4 not documented.**

**Possible reasons:**
- Multi-register value at address 1 (occupies 1-4)
- Reserved addresses
- Model-specific registers
- Documentation error

**Solution:**
- Check if previous register is multi-register type
- Don't access gaps
- Verify with vendor

## Best Practices for Working With Register Maps

### 1. Create Your Own Documentation

**Don't rely solely on vendor docs. Create:**

- Cleaned-up register map with protocol addresses
- Data types explicitly identified
- Tested word orders documented
- Working code examples
- Test results and validation

### 2. Version Control

**Track changes:**
- Device firmware version affects register map
- Document which map version applies to which firmware
- Keep old versions for legacy devices

### 3. Validate Everything

**Don't assume:**
- Test reads/writes on non-critical registers first
- Verify data types with known values
- Confirm word order empirically
- Check address ranges with boundary tests

### 4. Use Wireshark

**Reverse-engineer if needed:**
- Capture vendor software communication
- Identify actual addresses used
- Determine data types from patterns
- Verify function codes

(source: [Using Wireshark for MODBUS TCP Analysis](/wiki/answers/wireshark-modbus-analysis.md))

### 5. Contact Vendor Support

**When stuck:**
- Request complete register map
- Ask specific technical questions
- Request example code or application notes
- Report documentation errors

### 6. Build Abstraction Layer

**Create helper functions:**

```python
def read_temperature(client, unit=1):
    """Read current temperature with proper conversion"""
    data = client.read_holding_registers(address=12, count=2, unit=unit).registers
    # Convert FLOAT32 big-endian
    bytes_data = struct.pack('>HH', data[0], data[1])
    temp = struct.unpack('>f', bytes_data)[0]
    return temp

def write_setpoint(client, temp_celsius, unit=1):
    """Write temperature setpoint with proper conversion"""
    bytes_data = struct.pack('>f', temp_celsius)
    words = struct.unpack('>HH', bytes_data)
    client.write_registers(address=10, values=list(words), unit=unit)
```

This hides complexity and prevents errors.

## Summary Checklist: Reading a Register Map

- [ ] Identify addressing convention (protocol vs Modicon)
- [ ] Convert convention addresses to protocol addresses if needed
- [ ] Determine 0-based vs 1-based addressing
- [ ] Identify register types (Coil, Discrete Input, Holding, Input)
- [ ] Determine function codes for read and write
- [ ] Identify data types (UINT16, INT16, FLOAT32, etc.)
- [ ] Determine scaling factors and units
- [ ] Identify word order for multi-register values
- [ ] Note access permissions (R, W, R/W)
- [ ] Document address ranges and any gaps
- [ ] Test with non-critical registers first
- [ ] Create working register map with verified info
- [ ] Build helper functions for common operations

## Related Pages

- [MODBUS Register Data Representation](/wiki/answers/modbus-register-data-representation.md) - Data type storage details
- [Converting MODBUS Registers to Program Variables](/wiki/answers/converting-modbus-registers-to-program-variables.md) - Implementation examples
- [MODBUS Data Validation](/wiki/answers/modbus-data-validation.md) - Validating register values
- [Function Codes](/wiki/concepts/function-codes.md) - Complete function code reference
- [Holding Registers](/wiki/concepts/holding-registers.md) - Holding register details
- [Input Registers](/wiki/concepts/input-registers.md) - Input register details
- [Coils](/wiki/concepts/coils.md) - Coil details
- [Discrete Inputs](/wiki/concepts/discrete-inputs.md) - Discrete input details
- [Using Wireshark for MODBUS TCP Analysis](/wiki/answers/wireshark-modbus-analysis.md) - Reverse engineering register maps

## Backlinks

- [Converting MODBUS Registers to Program Variables](/wiki/answers/converting-modbus-registers-to-program-variables.md)
- [MODBUS Data Validation](/wiki/answers/modbus-data-validation.md)
- [MODBUS Function Codes Guide](/wiki/answers/modbus-function-codes-guide.md)
- [MODBUS Register Data Representation](/wiki/answers/modbus-register-data-representation.md)
- [Using Wireshark for MODBUS TCP Analysis](/wiki/answers/wireshark-modbus-analysis.md)
- [Coils](/wiki/concepts/coils.md)
- [Discrete Inputs](/wiki/concepts/discrete-inputs.md)
- [Function Codes](/wiki/concepts/function-codes.md)
- [Holding Registers](/wiki/concepts/holding-registers.md)
- [Input Registers](/wiki/concepts/input-registers.md)

------------

MODBUS is a trademark of the Modbus Organization, Inc.

All information on this page is derived by summarizing and analyzing solely the MODBUS specifications published by the Modbus Organization, Inc.  The specifications are copyright by the Modbus Organization.

By deriving this information solely from the specifications we hope to stay true to the specification.

No infringement is intended.
