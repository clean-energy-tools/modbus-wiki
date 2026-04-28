---
title: MODBUS Errors and Exception Responses
Summary: Comprehensive guide to MODBUS error types, exception responses, exception codes, diagnostic procedures, and troubleshooting addressing errors, unsupported functions, and device limitations.
Sources:
  - /raw/MODBUS/MODBUS.md
  - /raw/MODBUS/modbusprotocolspecification.md
  - /raw/MODBUS/messagingimplementationguide.md
Categories:
  - error-handling
  - exceptions
  - troubleshooting
  - diagnostics
type: answer
date-created: 2026-04-23T14:30:00+03:00
last-updated: 2026-04-23T14:30:00+03:00
---

MODBUS communication can fail in many ways, from network problems to device limitations. This guide covers the different types of MODBUS errors, how devices report problems with exception messages, what the exception codes mean, and how to diagnose common issues.

## Quick Reference: Types of Errors

| Error Type | Where It's Detected | Examples |
|----------------|----------------|----------|
| **Network Errors** | Network/Connection | Connection timeout, CRC error, no response |
| **Message Errors** | MODBUS Message Layer | Invalid header, badly formed message |
| **MODBUS Exceptions** | Application Level | Address doesn't exist, unsupported function, invalid value |
| **Device Problems** | Inside the Device | Internal failure, device busy, memory corruption |

## What is a MODBUS Exception Response?

### What It Is

A **MODBUS exception response** is a standardized error message sent by a MODBUS device when it cannot do what you asked.

**How they work:**
- The device MUST send back an exception (not just stay silent when something is wrong)
- The exception format is the same for all MODBUS systems
- Your program can identify the specific error type
- Exception responses look different from normal responses (the function code has bit 7 set)

Source: [function-codes.md](/wiki/concepts/function-codes.md:273)

### When Devices Send Exceptions

MODBUS devices send exception responses when:

1. **Function not supported** - Device doesn't have the function you requested
2. **Bad address** - The address is out of range or doesn't exist
3. **Invalid value** - Quantity too large, or illegal value (like a coil value that's not 0xFF00 or 0x0000)
4. **Device is busy** - Device is still working on a previous request
5. **Internal problem** - Hardware or software error inside the device
6. **Memory corruption** - Device detected corrupted memory
7. **Gateway issues** - Gateway device can't reach the target device

**Important:** The device MUST send a response. Staying silent when there's an error violates the MODBUS specification.

## How Exception Responses Look

### Exception Message Structure

**Normal response function code:** 0x01-0x7F (bit 7 = 0)

**Exception response function code:** Original function code + 0x80 (bit 7 = 1)

**Exception message (2 bytes):**
```
[Exception Function Code: 1 byte][Exception Code: 1 byte]
```

### Complete Exception Messages

**MODBUS TCP Exception (9 bytes total):**
```
┌──────── MBAP Header (7 bytes) ────────┬─ Exception ─┐
│ Trans ID │ Proto ID │ Length │ Unit ID│ Ex Func │ Ex Code│
│  2 bytes │  2 bytes │2 bytes │ 1 byte │ 1 byte  │ 1 byte │
└──────────┴──────────┴────────┴────────┴─────────┴────────┘
```

**MODBUS RTU Exception (5 bytes total):**
```
┌─────── Exception Message ───────┐
│ Address │ Ex Func │ Ex Code │ CRC │
│ 1 byte  │ 1 byte  │ 1 byte  │2 byt│
└─────────┴─────────┴─────────┴─────┘
```

### How to Recognize Exceptions

**How it's calculated:** Exception Function Code = Original Function Code + 0x80

**Examples:**
```
Read Holding Registers:    0x03 → Exception: 0x83 (0x03 + 0x80)
Read Coils:                0x01 → Exception: 0x81 (0x01 + 0x80)
Write Single Register:     0x06 → Exception: 0x86 (0x06 + 0x80)
Write Multiple Registers:  0x10 → Exception: 0x90 (0x10 + 0x80)
```

**In binary:**
```
Normal Function Code:   0x03 = 0b00000011 (bit 7 = 0)
Exception Function Code: 0x83 = 0b10000011 (bit 7 = 1)
                                    ↑
                                  Bit 7 is set
```

**How to detect:** Check if the function code is ≥ 0x80, or test if bit 7 is set.

Source: [function-codes.md](/wiki/concepts/function-codes.md:22)

## Complete Exception Code List

### Standard Exception Codes

| Code | Hex | Name | Description | Common Causes |
|------|-----|------|-------------|---------------|
| **01** | 0x01 | ILLEGAL FUNCTION | Function code not supported | Device doesn't implement function, wrong device type |
| **02** | 0x02 | ILLEGAL DATA ADDRESS | Address out of range | Address doesn't exist, typo in address, wrong memory area |
| **03** | 0x03 | ILLEGAL DATA VALUE | Invalid quantity or value | Quantity too large, coil value invalid, count out of range |
| **04** | 0x04 | SERVER DEVICE FAILURE | Internal error | Device malfunction, sensor failure, hardware error |
| **05** | 0x05 | ACKNOWLEDGE | Long processing request accepted | Used for functions taking >1 second, poll for completion |
| **06** | 0x06 | SERVER DEVICE BUSY | Device currently busy | Processing previous request, retry later |
| **08** | 0x08 | MEMORY PARITY ERROR | Memory parity check failed | Corrupted memory, device needs reset/repair |
| **10** | 0x0A | GATEWAY PATH UNAVAILABLE | Gateway configuration error | Gateway misconfigured, path unreachable |
| **11** | 0x0B | GATEWAY TARGET NO RESPONSE | Target device no response | Device offline, timeout, serial bus issue |

Source: [function-codes.md](/wiki/concepts/function-codes.md:277)

### Exception Code Details

#### 0x01: ILLEGAL FUNCTION

**What it means:** The device doesn't support the function code you requested.

**Common reasons:**
- You tried to write to a read-only device (like requesting Write Multiple Registers from a device that can only be read)
- You used an advanced function on a basic device that doesn't have it
- There's a typo or error in your function code

**Example:**
```
Your request:  Read/Write Multiple Registers (0x17) to a simple PLC
Device response: Exception 0x97, Code 0x01 (ILLEGAL FUNCTION)
```

**How to fix it:**
1. Check the device manual to see which functions it supports
2. Verify your device model and firmware version support this function
3. Try using a simpler function (for example, use 0x03 instead of 0x17)

**Resolution:**
- Use supported function codes only
- Check device capabilities before implementation

#### 0x02: ILLEGAL DATA ADDRESS

**Meaning:** The address (or address range) doesn't exist in the device's memory map.

**Common causes:**
- Address beyond device's register count
- Accessing non-existent coils/registers
- Quantity causes address range to exceed device limits
- Typo in address (e.g., 1000 instead of 100)

**Example:**
```
Request:  Read Holding Registers, Start=5000, Quantity=10
Device has registers 0-999
Response: Exception 0x83, Code 0x02 (ILLEGAL DATA ADDRESS)
```

**Diagnosis:**
1. Check device documentation for valid address ranges
2. Verify start address exists
3. Verify start address + quantity doesn't exceed device range
4. Check if accessing correct memory type (holding vs input)

**Resolution:**
- Use valid address ranges from device manual
- Add address range validation in client code

#### 0x03: ILLEGAL DATA VALUE

**Meaning:** The value or quantity in the request is invalid.

**Common causes:**

**Quantity too large:**
```
Read Holding Registers: Quantity > 125
Read Coils: Quantity > 2000
```

**Quantity is zero:**
```
Read Holding Registers: Quantity = 0 (must be 1-125)
```

**Invalid coil value:**
```
Write Single Coil: Value = 0x0100 (must be 0xFF00 or 0x0000)
```

**Device-specific limits:**
```
Device allows max 50 registers per read
Request: Quantity = 100
```

**Example:**
```
Request:  Write Single Coil, Address=100, Value=0x0001 (invalid!)
Response: Exception 0x85, Code 0x03 (ILLEGAL DATA VALUE)
```

**Diagnosis:**
1. Verify quantity within MODBUS protocol limits
2. Check device-specific limits (may be < protocol max)
3. Verify coil values are exactly 0xFF00 or 0x0000
4. Check register value ranges for device

**Resolution:**
- Respect protocol quantity limits
- Use correct values for coils (0xFF00 ON, 0x0000 OFF)
- Split large reads into multiple smaller requests

Source: [function-codes.md](/wiki/concepts/function-codes.md:81)

#### 0x04: SERVER DEVICE FAILURE

**Meaning:** The device encountered an internal error while processing the request.

**Common causes:**
- Sensor disconnected/failed
- Actuator malfunction
- Internal software error
- Hardware fault
- Watchdog timeout
- Power supply issues

**Example:**
```
Request:  Read Input Registers (temperature sensor)
Device:   Sensor disconnected
Response: Exception 0x84, Code 0x04 (SERVER DEVICE FAILURE)
```

**Diagnosis:**
1. Check device status LEDs/display
2. Review device logs (if available)
3. Verify sensors/actuators connected
4. Test device with known-good configuration
5. Cycle device power

**Resolution:**
- Fix hardware issue (reconnect sensor, replace faulty component)
- Reset device
- Contact device vendor if persistent

#### 0x05: ACKNOWLEDGE

**Meaning:** Request accepted but requires long processing time (>1 second).

**Usage:** Device acknowledges request will be processed asynchronously.

**Client behavior:**
1. Receive ACKNOWLEDGE exception
2. Wait (typically 1-5 seconds)
3. Poll device with same request or query completion status
4. Receive actual response when processing complete

**Example:**
```
Request:  Complex configuration write
Response: Exception 0x86, Code 0x05 (ACKNOWLEDGE)
... wait ...
Request:  Same request repeated
Response: Normal response (configuration complete)
```

**Rare in practice:** Most devices complete requests within typical timeout periods.

#### 0x06: SERVER DEVICE BUSY

**Meaning:** Device is currently processing another request and cannot handle this one.

**Common causes:**
- Device processing previous long-running operation
- Single-threaded device implementation
- Device resource locked

**Client behavior:**
1. Receive BUSY exception
2. Wait (typically 100-500 ms)
3. Retry request
4. Repeat until success or max retries

**Example:**
```
Request:  Write Multiple Registers (large block)
Response: Exception 0x90, Code 0x06 (SERVER DEVICE BUSY)
... wait 200ms ...
Request:  Retry same request
Response: Normal response (success)
```

**Resolution:**
- Implement exponential backoff retry logic
- Reduce request frequency if persistent

#### 0x08: MEMORY PARITY ERROR

**Meaning:** Device detected corrupted memory (parity check failed).

**Severity:** Critical - indicates hardware issue

**Common causes:**
- RAM corruption
- EEPROM failure
- Power glitch during memory write
- Defective memory chip

**Example:**
```
Request:  Read configuration registers
Response: Exception 0x83, Code 0x08 (MEMORY PARITY ERROR)
```

**Diagnosis:**
1. Power cycle device
2. Check for repeated occurrence
3. Review device error logs

**Resolution:**
- Power cycle device (may clear transient error)
- If persistent: Device needs repair/replacement
- Contact vendor for diagnostics

#### 0x0A: GATEWAY PATH UNAVAILABLE

**Meaning:** Gateway cannot reach the target device (configuration/routing issue).

**Common causes:**
- Gateway misconfigured
- Serial port disabled
- Wrong Unit ID mapping
- Network path unavailable

**Example:**
```
Request:  Read Holding Registers, Unit ID=5 (via gateway)
Gateway:  No route configured for Unit ID 5
Response: Exception 0x83, Code 0x0A (GATEWAY PATH UNAVAILABLE)
```

**Diagnosis:**
1. Verify gateway configuration
2. Check Unit ID mapping
3. Verify serial port enabled
4. Test gateway connectivity

**Resolution:**
- Configure gateway routing for Unit ID
- Verify serial port settings
- Check network connectivity

#### 0x0B: GATEWAY TARGET NO RESPONSE

**Meaning:** Gateway sent request to target device but received no response.

**Common causes:**
- Target device offline/powered off
- Serial bus timeout
- Wrong slave address
- Serial communication error
- Baud rate mismatch

**Example:**
```
Request:  Read Holding Registers, Unit ID=3 (via gateway)
Gateway:  Sends to serial slave #3, no response after 1 second
Response: Exception 0x83, Code 0x0B (GATEWAY TARGET NO RESPONSE)
```

**Diagnosis:**
1. Verify target device powered on
2. Check slave address configuration
3. Test serial bus connectivity
4. Verify baud rate/parity settings
5. Check RS-485 termination

**Resolution:**
- Power on target device
- Correct slave address
- Fix serial bus wiring
- Match baud rate on all devices

Source: [modbus-tcp-to-rtu-gateway.md](/wiki/answers/modbus-tcp-to-rtu-gateway.md:439)

## Exception Response Examples

### Example 1: Read Non-Existent Register

**Request:**
```
MODBUS TCP (12 bytes):
00 01 00 00 00 06 01 03 03 E8 00 01
│  Trans ID   │Length│UID│FC│Addr │Qty│
  0x0001        6     1   3  1000   1

Function: Read Holding Registers
Address: 1000 (0x03E8)
Quantity: 1
```

**Device has registers 0-999, address 1000 doesn't exist**

**Exception Response:**
```
MODBUS TCP (9 bytes):
00 01 00 00 00 03 01 83 02
│  Trans ID   │Length│UID│FC│EC│
  0x0001        3     1  0x83 0x02

Exception Function: 0x83 (0x03 + 0x80)
Exception Code: 0x02 (ILLEGAL DATA ADDRESS)
```

**Interpretation:** "Address 1000 is out of range"

### Example 2: Unsupported Function Code

**Request:**
```
MODBUS TCP (12 bytes):
00 02 00 00 00 06 01 17 00 00 00 01
│  Trans ID   │Length│UID│FC│...│
  0x0002        6     1  0x17

Function: 0x17 (Read/Write Multiple Registers)
```

**Device only supports basic functions (0x01-0x06, 0x0F, 0x10)**

**Exception Response:**
```
MODBUS TCP (9 bytes):
00 02 00 00 00 03 01 97 01
│  Trans ID   │Length│UID│FC│EC│
  0x0002        3     1  0x97 0x01

Exception Function: 0x97 (0x17 + 0x80)
Exception Code: 0x01 (ILLEGAL FUNCTION)
```

**Interpretation:** "Function 0x17 not supported"

### Example 3: Invalid Coil Value

**Request:**
```
MODBUS TCP (12 bytes):
00 03 00 00 00 06 01 05 00 64 00 01
│  Trans ID   │Length│UID│FC│Addr │Val│
  0x0003        6     1   5  100   0x0001

Function: Write Single Coil
Address: 100
Value: 0x0001 (INVALID - must be 0xFF00 or 0x0000)
```

**Exception Response:**
```
MODBUS TCP (9 bytes):
00 03 00 00 00 03 01 85 03
│  Trans ID   │Length│UID│FC│EC│
  0x0003        3     1  0x85 0x03

Exception Function: 0x85 (0x05 + 0x80)
Exception Code: 0x03 (ILLEGAL DATA VALUE)
```

**Interpretation:** "Coil value 0x0001 is invalid"

### Example 4: Quantity Too Large

**Request:**
```
MODBUS TCP (12 bytes):
00 04 00 00 00 06 01 03 00 00 00 C8
│  Trans ID   │Length│UID│FC│Addr │Qty│
  0x0004        6     1   3  0     200

Function: Read Holding Registers
Address: 0
Quantity: 200 (exceeds max of 125)
```

**Exception Response:**
```
MODBUS TCP (9 bytes):
00 04 00 00 00 03 01 83 03
│  Trans ID   │Length│UID│FC│EC│
  0x0004        3     1  0x83 0x03

Exception Function: 0x83 (0x03 + 0x80)
Exception Code: 0x03 (ILLEGAL DATA VALUE)
```

**Interpretation:** "Quantity 200 exceeds protocol maximum of 125"

Source: [modbus-tcp-message-format.md](/wiki/answers/modbus-tcp-message-format.md:293)

## Communication Errors (Non-Exception)

These errors occur at the communication level, before MODBUS processing:

### Timeout (No Response)

**Symptom:** Client sends request, no response received within timeout period.

**Possible causes:**
- Device offline/powered off
- Network cable disconnected
- Wrong IP address
- Firewall blocking port 502
- Device processing but response lost
- Serial timeout (gateway scenarios)

**Detection:** Application-level timeout timer expires.

**Client sees:** No MODBUS frame (neither normal nor exception).

**Diagnosis:**
1. Ping device IP address
2. Check network connectivity
3. Verify port 502 accessible (telnet/netcat)
4. Check device power and status
5. Review firewall rules
6. Increase timeout value (test if device is slow)

**Not a MODBUS exception:** Device never responded.

### CRC Error (MODBUS RTU)

**Symptom:** Received frame has invalid CRC-16 checksum.

**Possible causes:**
- Electrical noise on RS-485 bus
- Baud rate mismatch
- Wrong parity settings
- Cable too long (>1200m)
- Poor grounding
- Missing termination resistors

**Detection:** Receiver calculates CRC, doesn't match received CRC.

**Client behavior:** Discard frame silently, wait for timeout, retry.

**Diagnosis:**
1. Verify baud rate matches on all devices
2. Check parity/stop bit configuration
3. Measure RS-485 cable length
4. Add/verify 120Ω termination resistors
5. Improve grounding and shielding
6. Test with different cable

**Not a MODBUS exception:** Frame corrupted in transmission.

Source: [modbus-rtu.md](/wiki/concepts/modbus-rtu.md:124)

### Malformed Frame

**Symptom:** Frame doesn't conform to MODBUS structure.

**MODBUS TCP:**
- Protocol ID ≠ 0x0000
- Length field inconsistent
- Frame too short/long

**MODBUS RTU:**
- Inter-character timing violated (t1.5 exceeded)
- Frame too short (<5 bytes)

**Detection:** Parser rejects frame.

**Client behavior:** Discard frame, no retry (likely programming error).

**Diagnosis:**
1. Capture traffic with Wireshark
2. Verify frame structure
3. Check for buffer overflows
4. Review client implementation

**Not a MODBUS exception:** Invalid protocol frame.

## Diagnostic Procedures

### Diagnosing Addressing Errors

**Problem:** Getting exception 0x02 (ILLEGAL DATA ADDRESS)

**Step-by-step diagnosis:**

**1. Verify device's address range**
```
Read device manual:
- Holding Registers: 0-999
- Input Registers: 0-499
- Coils: 0-255
```

**2. Check your request address**
```
Request: Read Holding Register 1000
Problem: Address 1000 doesn't exist (max is 999)
Solution: Use address 0-999
```

**3. Verify address + quantity doesn't exceed range**
```
Request: Read 10 registers starting at 995
Calculation: 995 + 10 = 1005 (exceeds max 999)
Problem: End address out of range
Solution: Read starting at 990 or reduce quantity to 5
```

**4. Check if using correct memory type**
```
Request: Read Holding Register 100
Device: Address 100 exists only in Input Registers
Problem: Wrong memory type
Solution: Use Read Input Registers (0x04) instead
```

**5. Verify zero-based vs one-based addressing**
```
Device manual: "100 registers, addresses 1-100"
Your code: Read address 0
Problem: Device uses 1-based addressing
Solution: Use address 1 (or 0 if device manual means 0-99)
```

**6. Test with known-good address**
```
Try: Read register 0 (usually exists)
Success? → Original address was invalid
Failure? → Different problem (communication, function code)
```

### Diagnosing Unsupported Function Codes

**Problem:** Getting exception 0x01 (ILLEGAL FUNCTION)

**Step-by-step diagnosis:**

**1. Check device documentation**
```
Device manual: "Supports functions 0x01, 0x02, 0x03, 0x04, 0x06, 0x10"
Your request: Function 0x17 (Read/Write Multiple Registers)
Problem: Function not in supported list
Solution: Use separate 0x03 and 0x10 operations
```

**2. Verify function code value**
```
Your code: send_request(0x30) // Typo!
Problem: 0x30 is not a valid function code
Solution: Use correct code (e.g., 0x03)
```

**3. Check for device-specific limitations**
```
Device: Basic I/O module
Supports: Read/Write Coils and Discrete Inputs only
Your request: Read Holding Registers (0x03)
Problem: Device has no registers (only digital I/O)
Solution: Use Read Coils (0x01) or Read Discrete Inputs (0x02)
```

**4. Firmware version check**
```
Device: PLC model XYZ
Manual: "Function 0x17 supported in firmware v2.0+"
Your device: Firmware v1.5
Problem: Old firmware doesn't support function
Solution: Upgrade firmware or use alternative functions
```

**5. Test with basic function**
```
Try: Read Holding Registers (0x03) - universally supported
Success? → Original function code not supported
Failure? → Different problem (addressing, connectivity)
```

### Diagnosing Device-Specific Limitations

**Problem:** Getting exception 0x03 (ILLEGAL DATA VALUE) or 0x04 (DEVICE FAILURE) on valid requests

**Step-by-step diagnosis:**

**1. Check device-specific quantity limits**
```
Protocol: Max 125 registers per read
Device manual: "Max 50 registers per read operation"
Your request: Read 100 registers
Problem: Exceeds device limit (not protocol limit)
Solution: Split into two reads of 50 registers each
```

**2. Verify register value ranges**
```
Device manual: "Register 100: Setpoint, range 0-1000"
Your write: Write 5000 to register 100
Problem: Value exceeds device range
Exception: 0x03 (ILLEGAL DATA VALUE) or 0x04 (DEVICE FAILURE)
Solution: Validate value before writing
```

**3. Check for read-only registers**
```
Device manual: "Registers 0-99: Configuration (read-only)"
Your request: Write to register 50
Problem: Register is read-only
Exception: 0x02 (ILLEGAL DATA ADDRESS) or 0x03 (ILLEGAL DATA VALUE)
Solution: Write to read-write registers only
```

**4. Verify data type expectations**
```
Device manual: "Register 200: Temperature as IEEE 754 float (2 registers)"
Your read: Read single register 200
Problem: Value spans 2 registers
Solution: Read registers 200-201, combine as float
```

**5. Check for configuration requirements**
```
Device: Requires initialization sequence before normal operation
Your request: Read registers before initialization
Problem: Device not ready
Exception: 0x04 (DEVICE FAILURE) or 0x06 (DEVICE BUSY)
Solution: Perform initialization first
```

**6. Test incrementally**
```
Test 1: Read 1 register → Success
Test 2: Read 10 registers → Success
Test 3: Read 50 registers → Success
Test 4: Read 100 registers → Exception 0x03
Conclusion: Device limit is between 50-100 registers
```

## Error Handling Best Practices

### Client-Side Exception Handling

**Structured error handling:**

```python
def read_registers(client, address, count):
    try:
        response = client.read_holding_registers(address, count)
        return response.registers
    except ModbusException as e:
        if e.exception_code == 0x01:
            # ILLEGAL FUNCTION
            logging.error(f"Function not supported by device")
            raise UnsupportedFunctionError()
        elif e.exception_code == 0x02:
            # ILLEGAL DATA ADDRESS
            logging.error(f"Address {address} out of range")
            raise InvalidAddressError(address)
        elif e.exception_code == 0x03:
            # ILLEGAL DATA VALUE
            logging.error(f"Quantity {count} invalid")
            raise InvalidQuantityError(count)
        elif e.exception_code == 0x04:
            # SERVER DEVICE FAILURE
            logging.error(f"Device internal failure")
            raise DeviceFailureError()
        elif e.exception_code == 0x06:
            # SERVER DEVICE BUSY
            logging.warning(f"Device busy, retrying...")
            time.sleep(0.2)
            return read_registers(client, address, count)  # Retry
        else:
            logging.error(f"Unknown exception: {e.exception_code}")
            raise
    except TimeoutError:
        logging.error(f"Device timeout - no response")
        raise DeviceTimeoutError()
```

### Retry Logic

**Exponential backoff for transient errors:**

```python
def read_with_retry(client, address, count, max_retries=3):
    for attempt in range(max_retries):
        try:
            return client.read_holding_registers(address, count)
        except ModbusException as e:
            if e.exception_code == 0x06:  # DEVICE BUSY
                wait_time = 0.1 * (2 ** attempt)  # Exponential backoff
                logging.warning(f"Device busy, waiting {wait_time}s")
                time.sleep(wait_time)
                continue
            else:
                raise  # Don't retry non-transient errors
        except TimeoutError:
            if attempt < max_retries - 1:
                logging.warning(f"Timeout, retry {attempt + 1}")
                continue
            else:
                raise
    raise MaxRetriesExceededError()
```

### Request Validation

**Validate before sending:**

```python
def validate_read_request(address, count, device_max_registers):
    """Validate request before sending to device"""
    
    # Check protocol limits
    if count < 1:
        raise ValueError("Quantity must be at least 1")
    if count > 125:
        raise ValueError("Quantity exceeds protocol max (125)")
    
    # Check device limits
    if count > device_max_registers:
        raise ValueError(f"Quantity exceeds device max ({device_max_registers})")
    
    # Check address range
    if address < 0 or address > 65535:
        raise ValueError("Address out of range (0-65535)")
    
    # Check end address
    if address + count > device_max_registers:
        raise ValueError(f"Address range exceeds device capacity")
    
    return True
```

### Logging and Diagnostics

**Comprehensive error logging:**

```python
import logging

def log_modbus_error(request, exception):
    """Log detailed error information"""
    
    logging.error("MODBUS Error Details:")
    logging.error(f"  Request Function: 0x{request.function_code:02X}")
    logging.error(f"  Request Address: {request.address}")
    logging.error(f"  Request Quantity: {request.quantity}")
    
    if isinstance(exception, ModbusException):
        logging.error(f"  Exception Function: 0x{exception.function_code:02X}")
        logging.error(f"  Exception Code: 0x{exception.exception_code:02X}")
        logging.error(f"  Exception Name: {get_exception_name(exception.exception_code)}")
    else:
        logging.error(f"  Error Type: {type(exception).__name__}")
        logging.error(f"  Error Message: {str(exception)}")

def get_exception_name(code):
    """Map exception code to name"""
    names = {
        0x01: "ILLEGAL FUNCTION",
        0x02: "ILLEGAL DATA ADDRESS",
        0x03: "ILLEGAL DATA VALUE",
        0x04: "SERVER DEVICE FAILURE",
        0x05: "ACKNOWLEDGE",
        0x06: "SERVER DEVICE BUSY",
        0x08: "MEMORY PARITY ERROR",
        0x0A: "GATEWAY PATH UNAVAILABLE",
        0x0B: "GATEWAY TARGET NO RESPONSE",
    }
    return names.get(code, f"UNKNOWN (0x{code:02X})")
```

## Troubleshooting Decision Tree

```
MODBUS Communication Fails
│
├─ No Response?
│  ├─ YES → Timeout Error
│  │  ├─ Check network connectivity
│  │  ├─ Verify device powered on
│  │  ├─ Check IP address/port
│  │  └─ Increase timeout value
│  │
│  └─ NO → Received Response
│     │
│     ├─ Function Code ≥ 0x80?
│     │  ├─ YES → Exception Response
│     │  │  │
│     │  │  ├─ Exception Code 0x01? → ILLEGAL FUNCTION
│     │  │  │  └─ Check supported functions
│     │  │  │
│     │  │  ├─ Exception Code 0x02? → ILLEGAL DATA ADDRESS
│     │  │  │  ├─ Verify address range
│     │  │  │  └─ Check address + quantity
│     │  │  │
│     │  │  ├─ Exception Code 0x03? → ILLEGAL DATA VALUE
│     │  │  │  ├─ Check quantity limits
│     │  │  │  └─ Verify value ranges
│     │  │  │
│     │  │  ├─ Exception Code 0x04? → DEVICE FAILURE
│     │  │  │  ├─ Check device status
│     │  │  │  └─ Review device logs
│     │  │  │
│     │  │  └─ Exception Code 0x06? → DEVICE BUSY
│     │  │     └─ Retry with delay
│     │  │
│     │  └─ NO → Normal Response
│     │     └─ Success!
│     │
│     └─ CRC Error? (RTU only)
│        ├─ Check baud rate
│        ├─ Verify parity settings
│        └─ Check RS-485 wiring
```

## Common Error Scenarios

### Scenario 1: Reading Too Many Registers

**Error:**
```
Exception 0x03 (ILLEGAL DATA VALUE)
```

**Cause:** Requesting more registers than allowed.

**Investigation:**
```
Protocol max: 125 registers
Device max: 50 registers (from manual)
Your request: 100 registers
```

**Solution:**
```python
# Split into multiple requests
registers = []
registers += read_registers(client, 0, 50)
registers += read_registers(client, 50, 50)
# Now have 100 registers total
```

### Scenario 2: Wrong Function for Device Type

**Error:**
```
Exception 0x01 (ILLEGAL FUNCTION)
```

**Cause:** Using register functions on digital I/O device.

**Investigation:**
```
Device: Digital I/O module (16 inputs, 16 outputs)
Your request: Read Holding Registers (0x03)
Device supports: Read Coils (0x01), Write Coils (0x05)
```

**Solution:**
```python
# Use coil functions instead
coils = client.read_coils(0, 16)  # Read 16 digital inputs
client.write_coil(5, True)  # Turn on output 5
```

### Scenario 3: Gateway Timeout

**Error:**
```
Exception 0x0B (GATEWAY TARGET NO RESPONSE)
```

**Cause:** Serial slave behind gateway not responding.

**Investigation:**
```
TCP Client → Gateway (192.168.1.100) → Serial Slave #5
Gateway logs: "Timeout waiting for slave 5"
```

**Solution:**
```
1. Verify slave 5 powered on
2. Check slave address configured as 5
3. Verify serial baud rate matches
4. Test serial bus with direct connection
5. Increase gateway timeout
```

### Scenario 4: Address Confusion

**Error:**
```
Exception 0x02 (ILLEGAL DATA ADDRESS)
```

**Cause:** Confusion between Modbus address and device documentation.

**Investigation:**
```
Device manual: "Temperature register at address 40001"
Your request: Read address 40001
Problem: "40001" is Modbus convention notation, not actual address
Actual address: 40001 - 40001 = 0 (holding register 0)
```

**Modbus address conventions:**
```
00001-09999: Coils (address = register - 1)
10001-19999: Discrete Inputs (address = register - 10001)
30001-39999: Input Registers (address = register - 30001)
40001-49999: Holding Registers (address = register - 40001)
```

**Solution:**
```python
# Don't use "40001" notation in code
# Manual says "40001" → Use address 0
registers = client.read_holding_registers(0, 1)
```

## Summary: Quick Exception Reference

| Exception | Common Cause | Quick Fix |
|-----------|--------------|-----------|
| **0x01** | Function not supported | Check device manual, use supported function |
| **0x02** | Address out of range | Verify address in valid range |
| **0x03** | Invalid quantity/value | Check protocol limits, split request |
| **0x04** | Device internal error | Check device status, reset device |
| **0x05** | Long processing | Wait and retry |
| **0x06** | Device busy | Retry with delay |
| **0x08** | Memory corruption | Power cycle, replace if persistent |
| **0x0A** | Gateway config error | Fix gateway routing |
| **0x0B** | Target no response | Check slave device, serial bus |

## Related Pages

- [Function Codes](/wiki/concepts/function-codes.md) - Complete function code reference
- [MODBUS Data Validation](/wiki/answers/modbus-data-validation.md) - Input validation guide
- [MODBUS TCP Message Format](/wiki/answers/modbus-tcp-message-format.md) - Frame structure
- [How MODBUS TCP-to-RTU Gateways Work](/wiki/answers/modbus-tcp-to-rtu-gateway.md) - Gateway errors
- [Using Wireshark for MODBUS TCP Analysis](/wiki/answers/wireshark-modbus-analysis.md) - Analyzing exceptions

## Backlinks

- [MODBUS Commissioning Checklist](/wiki/answers/modbus-commissioning-checklist.md)
- [MODBUS Function Codes Guide](/wiki/answers/modbus-function-codes-guide.md)
- [MODBUS Register Addressing](/wiki/answers/modbus-register-addressing.md)
- [RS485 Wiring for MODBUS RTU](/wiki/answers/rs485-wiring-for-modbus-rtu.md)
- [Function Codes](/wiki/concepts/function-codes.md)

------------

MODBUS is a trademark of the Modbus Organization, Inc.

All information on this page is derived by summarizing and analyzing solely the MODBUS specifications published by the Modbus Organization, Inc.  The specifications are copyright by the Modbus Organization.

By deriving this information solely from the specifications we hope to stay true to the specification.

No infringement is intended.
