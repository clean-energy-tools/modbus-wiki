---
title: MODBUS Commissioning Checklist
Summary: Comprehensive checklist for commissioning MODBUS communications on site, including pre-commissioning preparation, required physical tools, software tools, installation procedures, testing steps, and troubleshooting guidelines.
Sources:
  - /raw/MODBUS/modbusoverserial.md
  - /raw/MODBUS/MODBUS.md
  - /raw/MODBUS/modbusprotocolspecification.md
Categories:
  - commissioning
  - installation
  - testing
  - troubleshooting
  - best-practices
type: answer
date-created: 2026-04-24T16:00:00+03:00
last-updated: 2026-04-24T16:30:00+03:00
---

This comprehensive commissioning checklist covers everything needed to successfully install, configure, and validate MODBUS communications on site. It includes pre-commissioning preparation, required tools, step-by-step procedures, and troubleshooting guidelines.

## Overview: Commissioning Phases

MODBUS commissioning is best approached in distinct phases:

1. **Pre-Commissioning** (office work, before site visit)
   - Gather documentation and device information
   - Plan network architecture and addressing
   - Prepare materials and tools
   - Create site-specific documentation package

2. **On-Site Verification** (first activity on site)
   - Verify physical infrastructure (RS-485 or Ethernet)
   - Check existing installations
   - Validate site conditions

3. **Installation** (physical work)
   - Install cables and devices
   - Wire connections
   - Install termination and grounding

4. **Configuration and Testing** (bring system online)
   - Configure devices
   - Power-up and test
   - Validate communications
   - Performance testing

5. **Documentation and Handover**
   - As-built documentation
   - Training
   - Final acceptance

---

## Pre-Commissioning Checklist

This phase happens **before arriving on site**. Complete preparation ensures efficient site work and reduces costly return visits.

### 1. Device Information Gathering

**For Each MODBUS Device:**

- [ ] **Obtain complete device documentation**
  - User manual with MODBUS section
  - Technical specifications
  - Installation guide
  - Quick reference card (if available)

- [ ] **Extract MODBUS-specific information:**
  - Supported MODBUS variants (RTU, TCP, ASCII)
  - Supported function codes (0x01-0x10, advanced functions)
  - Default communication settings
  - Configuration requirements (if any)

- [ ] **Document complete register map for each device:**
  - Coil addresses and descriptions
  - Discrete Input addresses and descriptions
  - Holding Register addresses, data types, scaling factors, units
  - Input Register addresses, data types, scaling factors, units
  - Read-only vs read-write permissions
  - Reserved or unused address ranges

- [ ] **Identify device-specific limitations:**
  - Maximum registers per read (may be less than 125)
  - Maximum registers per write (may be less than 123)
  - Minimum poll interval / maximum request rate
  - Special initialization requirements
  - Firmware version compatibility

- [ ] **Record device power requirements:**
  - Voltage (24VDC, 120VAC, etc.)
  - Current draw
  - Power supply recommendations

- [ ] **Note physical specifications:**
  - Mounting requirements (DIN rail, panel mount, etc.)
  - Terminal types (screw, spring, pluggable)
  - Environmental ratings (IP rating, temperature range)
  - Dimensions and weight

### 2. Network Architecture Planning

**MODBUS RTU (Serial RS-485):**

- [ ] **Determine network topology:**
  - Linear daisy-chain (required for RS-485)
  - Calculate total trunk cable length
  - Plan device physical locations along cable run
  - Identify first and last devices (for termination)

- [ ] **Calculate cable requirements:**
  - Total trunk cable length (≤1000m at 9600 baud)
  - Drop cable lengths for each device (<20m each)
  - Add 10% extra for routing flexibility
  - Specify cable type (shielded twisted pair, AWG 24)

- [ ] **Select communication parameters:**
  - Baud rate (consider distance: 9600 for 1000m, 19200 for 500m)
  - Serial format (typically 8E1: 8 data, even parity, 1 stop)
  - Determine if polarization needed (long runs >100m, noisy environments)

- [ ] **Assign slave addresses:**
  - Create address assignment table
  - Addresses 1-247 (avoid 0 for broadcast unless needed)
  - Ensure no duplicates
  - Document assignment rationale (e.g., physical order)

- [ ] **Verify device count ≤32** (or check for low-load transceivers)

**MODBUS TCP (Ethernet):**

- [ ] **Network infrastructure planning:**
  - Determine if using existing network or dedicated MODBUS network
  - Identify switch locations and port counts
  - Plan VLAN configuration (if using existing network)
  - Determine if PoE required for any devices

- [ ] **IP addressing scheme:**
  - Assign static IP addresses for all MODBUS devices
  - Document subnet mask and gateway
  - Ensure no IP conflicts with existing network
  - Create IP address assignment table

- [ ] **Network topology:**
  - Star topology (devices to switch)
  - Calculate cable runs
  - Identify cable routes

- [ ] **Cable specifications:**
  - Cat5e or Cat6 Ethernet cable
  - Shielded (STP) vs unshielded (UTP) based on environment
  - Maximum length per run: 100m

**Gateway Configuration (TCP-to-RTU):**

- [ ] **Plan Unit ID to slave address mapping**
  - Document which TCP Unit IDs map to which RTU slave addresses
  - Create mapping table
  - Identify gateway IP address and configuration

### 3. Materials and Tools Preparation

**Create Bill of Materials:**

- [ ] **Cable:**
  - RS-485: Shielded twisted pair (calculated length + 10%)
  - TCP: Cat5e/6 Ethernet cable (calculated length + 10%)
  - Cable labels and markers

- [ ] **RS-485 Components (if applicable):**
  - 2× 120Ω termination resistors (0.5W)
  - 2× 560Ω bias resistors (if polarization needed)
  - Terminal blocks or junction boxes
  - Ferrules for wire termination

- [ ] **Ethernet Components (if applicable):**
  - Network switch (with sufficient ports)
  - Ethernet patch cables
  - RJ45 connectors or patch panel
  - Cable tester

- [ ] **Installation Hardware:**
  - Cable ties and mounts
  - Cable trays or conduit
  - DIN rail (if needed)
  - Mounting hardware for devices
  - Strain relief

- [ ] **Testing Equipment:**
  - Multimeter (essential for RS-485)
  - Laptop with testing software installed
  - USB-to-RS485 adapter (for RTU testing)
  - Ethernet cable tester
  - Wireshark pre-installed and configured
  - ModScan or equivalent testing tool
  - Network cables for test equipment

- [ ] **Documentation and Supplies:**
  - Printed device manuals
  - Printed commissioning checklist
  - Network diagrams (paper copies)
  - Labels and label printer
  - Clipboard and writing materials
  - Camera for documentation photos

### 4. Create Site Documentation Package

**Prepare for Technician:**

- [ ] **Network diagram:**
  - Physical layout showing device locations
  - Cable routing plan
  - Termination locations marked
  - Shield grounding location marked
  - Address/IP assignments labeled

- [ ] **Device configuration sheets (one per device):**
  - Device name and description
  - Physical location
  - MODBUS address or IP
  - Serial settings (RTU) or network settings (TCP)
  - DIP switch or jumper settings required
  - Initial configuration steps

- [ ] **Register map summary (one per device type):**
  - Key registers for testing
  - Test values to write/read
  - Expected data ranges
  - Critical configuration registers

- [ ] **Communication settings summary:**
  - Baud rate, parity, stop bits (RTU)
  - IP addresses, subnet, gateway (TCP)
  - Gateway configuration (if applicable)

- [ ] **Installation procedure:**
  - Step-by-step installation guide
  - Wiring diagrams with color codes
  - Termination installation instructions
  - Grounding procedure

- [ ] **Test plan:**
  - Test procedures in order
  - Expected results for each test
  - Pass/fail criteria
  - Troubleshooting decision tree

### 5. Software Preparation

**On Test Laptop/Equipment:**

- [ ] **Install and configure testing software:**
  - Wireshark with MODBUS dissector verified
  - ModScan or equivalent MODBUS client
  - Serial terminal program
  - Network scanning tools
  - Any vendor-specific configuration tools

- [ ] **Prepare test scripts:**
  - PyModbus scripts for automated testing (if using)
  - Test sequences for each device
  - Data validation scripts

- [ ] **Driver installation:**
  - USB-to-RS485 adapter drivers
  - Ethernet adapter drivers
  - Any device-specific drivers

- [ ] **Create test configuration files:**
  - Pre-configured MODBUS client settings
  - Device connection profiles
  - IP address lists

### 6. Verify Site Prerequisites

**Coordinate with Site Contact:**

- [ ] **Confirm physical infrastructure ready:**
  - Cable trays or conduit installed
  - Device mounting locations prepared
  - Power available at device locations
  - Environmental conditions suitable (temperature, moisture)

- [ ] **Verify electrical:**
  - Power circuits available and tested
  - Grounding/earthing system in place
  - Circuit breaker/disconnect locations identified

- [ ] **Confirm network infrastructure (TCP):**
  - Network switch installed and powered
  - Fiber or backbone connections active
  - Network access credentials available
  - Firewall rules configured (if applicable)

- [ ] **Site access and safety:**
  - Site access permissions arranged
  - Safety requirements understood (PPE, permits, etc.)
  - Contact person and phone number
  - Site hours and any restrictions

- [ ] **Coordinate with other trades:**
  - Electrical work completed
  - Mechanical installation complete
  - No conflicting work scheduled

### 7. Pre-Commissioning Documentation Review

**Final Checks Before Site Visit:**

- [ ] **Review all device documentation:**
  - Confirm nothing missing
  - Verify latest firmware/manual versions
  - Check for known issues or errata

- [ ] **Validate design:**
  - Cable lengths within specifications
  - Device count within limits
  - No addressing conflicts
  - Proper termination plan

- [ ] **Risk assessment:**
  - Identify potential issues
  - Prepare contingency plans
  - Have vendor support contacts ready

- [ ] **Checklist completeness:**
  - All materials sourced
  - All tools packed
  - All documentation printed
  - Test equipment charged/ready

---

## On-Site Verification (First Activity)

This is the **first thing to do upon arriving at the site**, before installation begins.

### RS-485 Network Pre-Installation Verification

**If RS-485 cabling already installed:**

- [ ] **Visual inspection:**
  - Verify cable type (shielded twisted pair)
  - Check cable routing (away from power cables)
  - Inspect for physical damage
  - Verify cable labeling

- [ ] **Measure cable resistance (power OFF):**
  - Disconnect all devices
  - Remove any existing terminators
  - Measure D1 to D0 resistance
    - With no terminators: Should be open circuit (>1MΩ)
    - With 2× 120Ω terminators: Should be ~60Ω
  - Measure D1 to ground: Should be high impedance
  - Measure D0 to ground: Should be high impedance

- [ ] **Check for shorts and miswiring:**
  - Verify no D1-to-D0 short (should be open without terminators)
  - Verify no D1-to-ground short
  - Verify no D0-to-ground short
  - Check shield connection (should be grounded at one point only)

- [ ] **Verify cable continuity:**
  - Short D1-D0 at far end
  - Measure resistance at near end
  - Should show short
  - Remove short and verify open circuit

- [ ] **Document findings:**
  - Actual cable lengths measured
  - Any issues found
  - Corrections needed

**If RS-485 cabling not yet installed:**

- [ ] **Verify cable routing path is available:**
  - Cable tray clear and accessible
  - No obstructions
  - Adequate support along route

- [ ] **Verify separation from power cables:**
  - Minimum 300mm from AC power
  - Crossing at 90° if unavoidable
  - No parallel runs with VFD cables

### Ethernet Network Pre-Installation Verification

**If using existing Ethernet network:**

- [ ] **Verify network switch is operational:**
  - Power LED on
  - Sufficient ports available
  - Correct VLAN configured (if applicable)
  - PoE available if needed

- [ ] **Test network connectivity:**
  - Connect laptop to network
  - Verify DHCP or configure static IP
  - Ping gateway
  - Verify no IP conflicts in planned address range

- [ ] **Network performance check:**
  - Test bandwidth (should be 100Mbps or 1Gbps)
  - Check for packet loss (ping test)
  - Verify latency is low (<10ms local network)

- [ ] **Verify firewall configuration:**
  - Port 502 accessible for MODBUS TCP
  - Port 802 accessible for MODBUS TCP Security (if used)
  - No blocking between client and device subnets

- [ ] **Document network configuration:**
  - Switch IP and credentials
  - VLAN IDs
  - Gateway address
  - DNS (if needed)

**If installing dedicated Ethernet network:**

- [ ] **Verify cable routing path is available:**
  - Cable tray clear
  - Adequate support
  - Maximum run ≤100m per segment

- [ ] **Verify network switch location:**
  - Power available
  - Mounting space
  - Accessible for maintenance

### Device Location Verification

- [ ] **Verify all device mounting locations:**
  - Space available
  - Mounting surface suitable
  - Environmental conditions acceptable (temperature, moisture)
  - Accessible for wiring and maintenance

- [ ] **Verify power availability at each location:**
  - Correct voltage available
  - Circuit capacity adequate
  - Grounding available

- [ ] **Verify cable paths to each device:**
  - Drop cables can reach device location
  - No excessive bends or stress
  - Length within specifications

### Site Conditions

- [ ] **Environmental check:**
  - Temperature within device specifications
  - No excessive moisture or condensation
  - No corrosive atmosphere
  - Adequate ventilation

- [ ] **Safety verification:**
  - Work area safe and clear
  - Required PPE available
  - Lockout/tagout procedures understood
  - Emergency contacts known

- [ ] **Document site conditions:**
  - Photos of installation area
  - Any deviations from plan
  - Issues requiring resolution

---

## Physical Tools Required

### Cable and Installation Hardware

**Essential Components:**
- **Shielded twisted pair cable** (AWG 24 or wider) - Belden 3105A, Belden 9842, or industrial RS-485 cable
- **2× 120Ω termination resistors** (0.5W power rating recommended, 0.25W minimum)
- **2× 560Ω bias resistors** (optional, for polarization if needed)
- **Screw terminal blocks or connectors** (RJ45 or D-Sub-9 as needed)
- **Cable ties, labels, and marking supplies**
- **Ferrules for stranded wire connections**
- **Cable trays or conduit** for cable support
- **Strain relief hardware**

(source: [rs485-wiring-for-modbus-rtu.md](/wiki/answers/rs485-wiring-for-modbus-rtu.md:644-651))

### Measurement and Testing Tools

**Required:**
- **Multimeter** - For resistance and voltage measurements
  - Verify termination (60Ω between D1-D0 with power off)
  - Check for shorts and ground faults
  - Measure idle bus voltage (if polarization used)

**Optional but Recommended:**
- **Oscilloscope** - For advanced signal analysis and troubleshooting
  - View differential signal quality
  - Detect signal reflections
  - Identify noise and EMI issues

(source: [rs485-wiring-for-modbus-rtu.md](/wiki/answers/rs485-wiring-for-modbus-rtu.md:781-804))

## Software Tools Required

### Testing and Analysis Tools

**Network Protocol Analysis:**
- **Wireshark** - Network protocol analyzer for MODBUS TCP
  - Packet capture and inspection
  - MODBUS protocol decoding
  - Frame-by-frame analysis
  - Export to PCAP for documentation

(source: [wireshark-modbus-analysis.md](/wiki/answers/wireshark-modbus-analysis.md))

**MODBUS Client/Simulator:**
- **ModScan** or equivalent MODBUS client simulator
- **PyModbus** or similar library for custom testing scripts
- **modbuspoll** or other commercial testing tools

**Serial Communication:**
- **Terminal program** for serial port configuration and ASCII mode debugging
- **USB-to-RS485 adapter** with drivers (if testing serial from PC)
- **Serial port configuration utilities**

### Development Tools (if writing custom client)

- MODBUS library for your language (PyModbus, libmodbus, etc.)
- Network scanning tools for device discovery
- Logging and monitoring utilities

## Pre-Installation Planning

### 1. Documentation Review

- [ ] **Obtain device manuals** for all MODBUS devices
- [ ] **Identify supported function codes** for each device
  - Common: 0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x0F, 0x10
  - Advanced: 0x17, 0x2B (may not be supported by all devices)
- [ ] **Document register maps and address ranges**
  - Coils: addresses and quantity
  - Discrete Inputs: addresses and quantity
  - Holding Registers: addresses and quantity
  - Input Registers: addresses and quantity
- [ ] **Note device-specific limitations**
  - Maximum quantity per read (may be less than protocol max of 125)
  - Read-only vs read-write registers
  - Valid value ranges for configuration registers
- [ ] **Record default serial settings**
  - Baud rate (common: 9600, 19200)
  - Parity (Even, Odd, None)
  - Stop bits (1 or 2)
  - Default format often: 9600-8E1 or 19200-8E1

(source: [modbus-errors-and-exceptions.md](/wiki/answers/modbus-errors-and-exceptions.md:659-696))

### 2. Network Design

**Topology Planning:**
- [ ] **Plan linear daisy-chain topology** (avoid star configuration for 2-wire RS-485)
- [ ] **Calculate total trunk cable length** (≤1000m at 9600 baud, ≤1200m at lower rates)
- [ ] **Identify first and last devices** (termination points)
- [ ] **Plan drop cable lengths** (<20m each, preferably <5m)
  - Shorter drops reduce signal reflections
  - Keep devices close to trunk cable
- [ ] **Verify device count ≤32** (standard RS-485 limit)
  - Or verify devices use low-load transceivers (1/4 load → 128 devices, 1/8 load → 256 devices)
  - Consider repeaters if more devices needed
- [ ] **Assign unique slave addresses** (1-247)
  - Address 0: Reserved for broadcast
  - Address 248-255: Reserved
  - Document address assignments

(source: [rs485-wiring-for-modbus-rtu.md](/wiki/answers/rs485-wiring-for-modbus-rtu.md:166-246))

**Cable Distance vs Baud Rate:**

| Baud Rate | Maximum Distance |
|-----------|------------------|
| 9600 bps  | 1000m           |
| 19200 bps | 500m            |
| 38400 bps | 250m            |
| 115200 bps| 50m             |

### 3. Configuration Planning

- [ ] **Select baud rate** (common: 9600 or 19200 for RTU)
  - Lower rates allow longer distances
  - Higher rates provide faster response times
- [ ] **Choose serial format**
  - Typical: 8E1 (8 data bits, even parity, 1 stop bit)
  - Alternative: 8N2 (8 data bits, no parity, 2 stop bits)
- [ ] **Decide on polarization/biasing requirements**
  - Check device manuals
  - Recommended for long cable runs (>100m)
  - Required in noisy environments
- [ ] **Plan shield grounding location** (typically at master)
  - Ground at ONE end only to prevent ground loops
  - Ensure good protective earth connection

## Physical Installation

### 1. Cable Installation

- [ ] **Run shielded twisted pair trunk cable** in straight line
  - Use linear topology, avoid creating loops
  - Minimize total cable length
- [ ] **Support cable properly** using cable trays or conduit
  - Prevent cable sag and damage
  - Use proper cable ties (not over-tightened)
- [ ] **Maintain separation from power cables** (avoid EMI)
  - Minimum 300mm separation from AC power
  - Cross at 90° if unavoidable
  - Avoid running parallel to VFD cables
- [ ] **Label cable at both ends**
  - Document cable run in installation records
  - Mark as "MODBUS RS-485"

(source: [rs485-wiring-for-modbus-rtu.md](/wiki/answers/rs485-wiring-for-modbus-rtu.md:683-692))

### 2. Device Wiring

**Wiring Connections:**
- [ ] **Connect D1 (A/yellow)** to all device D1/A terminals
- [ ] **Connect D0 (B/brown)** to all device D0/B terminals
- [ ] **Connect COM (grey)** to all device Common/GND terminals
- [ ] **Keep drop cable lengths short** (<20m, preferably <5m)
- [ ] **Use ferrules on stranded wire** for reliable screw terminal connections
- [ ] **Torque screw terminals to specification** (typically 0.5 Nm)
- [ ] **Verify no D1/D0 reversals**
  - Swapped D1/D0 is most common wiring error
  - Double-check wire colors at each terminal

**Wire Color Conventions:**

| Signal | Recommended Color |
|--------|-------------------|
| D1 (A) | Yellow           |
| D0 (B) | Brown            |
| Common | Grey             |

(source: [rs485-wiring-for-modbus-rtu.md](/wiki/answers/rs485-wiring-for-modbus-rtu.md:283-292))

### 3. Termination

- [ ] **Install 120Ω resistor** between D1-D0 at **first device**
- [ ] **Install 120Ω resistor** between D1-D0 at **last device**
- [ ] **Verify NO terminators** on middle devices
  - Too many terminators loads bus excessively
  - Missing terminators causes signal reflections
- [ ] **If using built-in terminators**, verify correct DIP switch/jumper settings
  - Enable on first and last devices only
  - Disable on all middle devices

**Termination Specifications:**

| Parameter | Value |
|-----------|-------|
| Resistance | 120Ω (preferred) or 150Ω |
| Power rating | 0.25W minimum, 0.5W recommended |
| Location | Both ends of trunk cable ONLY |
| Quantity | Exactly 2 (one at each end) |

(source: [rs485-wiring-for-modbus-rtu.md](/wiki/answers/rs485-wiring-for-modbus-rtu.md:315-402))

### 4. Shield Grounding

- [ ] **Connect shield to protective earth** at master end ONLY
  - Typically master/PLC location
  - Must have good earth connection
- [ ] **Verify shield floating** (insulated) at all other devices
  - Do NOT ground at multiple points
  - Prevents ground loops and circulating currents
- [ ] **Confirm good earth connection** at grounding point
  - Test with multimeter (low resistance to earth)

(source: [rs485-wiring-for-modbus-rtu.md](/wiki/answers/rs485-wiring-for-modbus-rtu.md:293-314))

### 5. Polarization (If Needed)

**Install only if required by devices or for long/noisy networks:**

- [ ] **Install 560Ω pull-up resistor** from D1 to +5V
- [ ] **Install 560Ω pull-down resistor** from D0 to GND
- [ ] **Install at ONE location only** (typically master)
  - Do NOT install on multiple devices
  - Reduces maximum device count by 4

**Polarization Effect:**
- Forces idle bus to known state (D1 high, D0 low)
- Prevents false triggering from noise
- Recommended for cable runs >100m
- Required in very noisy environments

(source: [rs485-wiring-for-modbus-rtu.md](/wiki/answers/rs485-wiring-for-modbus-rtu.md:418-491))

## Pre-Power-On Checks

### 1. Wiring Verification

- [ ] **Visually inspect all connections**
  - Check for loose wires
  - Verify correct terminals used
- [ ] **Verify correct wire colors** at each terminal
  - D1: Yellow
  - D0: Brown
  - COM: Grey
- [ ] **Check all screw terminals are tight**
  - Use appropriate torque
  - Verify no loose connections
- [ ] **Confirm no exposed wire strands**
  - Check for potential shorts
  - Ensure proper strain relief

### 2. Resistance Measurements (Power OFF)

**Critical Measurement - Termination Check:**
- [ ] **Measure D1 to D0 resistance: Should read ~60Ω**
  - Two 120Ω terminators in parallel = 60Ω
  - If reads ~120Ω: Only one terminator present
  - If reads open circuit (>1MΩ): No terminators
  - If with polarization: Still ~60Ω (terminators dominate over 560Ω bias)

**Additional Checks:**
- [ ] **Measure D1 to ground: Should be high impedance**
  - If low resistance: Short to ground
- [ ] **Measure D0 to ground: Should be high impedance**
  - If low resistance: Short to ground

(source: [rs485-wiring-for-modbus-rtu.md](/wiki/answers/rs485-wiring-for-modbus-rtu.md:781-798))

**Expected Resistance Values:**

| Measurement | Expected Value | Problem if Different |
|-------------|----------------|---------------------|
| D1 to D0 (with 2× 120Ω) | ~60Ω | Missing/extra terminators |
| D1 to GND | >1MΩ | Short to ground |
| D0 to GND | >1MΩ | Short to ground |

## Device Configuration

### 1. Serial Port Settings (Must Match on ALL Devices)

**Configure and verify:**
- [ ] **Baud rate:** _____ (e.g., 9600, 19200, 38400, 115200)
- [ ] **Data bits:** 8 (standard for MODBUS)
- [ ] **Parity:** _____ (Even, Odd, or None)
  - Even parity most common for MODBUS RTU
- [ ] **Stop bits:** _____ 
  - 1 stop bit with parity
  - 2 stop bits without parity

**Common Configurations:**
- 9600-8E1 (9600 baud, 8 data bits, even parity, 1 stop bit)
- 19200-8E1 (19200 baud, 8 data bits, even parity, 1 stop bit)
- 9600-8N2 (9600 baud, 8 data bits, no parity, 2 stop bits)

**Critical:** ALL devices on the bus must use identical settings.

### 2. Device Addresses

- [ ] **Verify each device has unique slave address**
- [ ] **Document address assignments** in commissioning records
  - Device name/description
  - Physical location
  - MODBUS slave address
- [ ] **Confirm addresses in range 1-247**
  - 0: Broadcast (no response)
  - 1-247: Valid slave addresses
  - 248-255: Reserved
- [ ] **Verify no duplicate addresses**
  - Use address polling test
  - Document in network diagram

### 3. Gateway Configuration (If Applicable)

For MODBUS TCP-to-RTU gateways:

- [ ] **Configure Unit ID mapping**
  - Map TCP Unit IDs to serial slave addresses
  - Document mapping table
- [ ] **Set timeout values** (typical: 100-1000ms)
  - Serial response timeout
  - TCP connection timeout
- [ ] **Verify serial port enabled** and configured
  - Baud rate, parity, stop bits
  - RS-485 mode enabled
- [ ] **Configure IP address and port**
  - Static IP recommended
  - Port 502 for MODBUS TCP
  - Port 802 for MODBUS TCP Security

(source: [modbus-tcp-to-rtu-gateway.md](/wiki/answers/modbus-tcp-to-rtu-gateway.md))

## Initial Power-On and Testing

### 1. Incremental Power-Up

**Power devices one at a time to isolate problems:**

- [ ] **Power on master only**
- [ ] **Verify master operational** (status LEDs, no errors)
- [ ] **Power on first slave**
- [ ] **Test communication to first slave** (simple read request)
- [ ] **Verify response** (no timeouts, no exceptions)
- [ ] **Power on next slave**
- [ ] **Test communication to all devices** powered on so far
- [ ] **Repeat until all devices online**

**Benefits of incremental approach:**
- Isolates problematic devices immediately
- Prevents cascading failures
- Simplifies troubleshooting
- Validates wiring progressively

(source: [rs485-wiring-for-modbus-rtu.md](/wiki/answers/rs485-wiring-for-modbus-rtu.md:714-719))

### 2. Basic Communication Tests

**Poll Test:**
- [ ] **Poll each device by address**
- [ ] **Verify each device responds**
  - No timeouts
  - No exception responses
  - Transaction ID matches (TCP)
- [ ] **Test with basic function** (Read Holding Registers 0x03)
  - Most universally supported function
  - Try reading register 0
- [ ] **Verify no exception responses**
- [ ] **Check response times are reasonable**
  - Typical: 10-100ms for RTU
  - Typical: 5-50ms for TCP

### 3. Function Code Testing

**Test all required operations:**

- [ ] **Test read operations:**
  - 0x01: Read Coils
  - 0x02: Read Discrete Inputs
  - 0x03: Read Holding Registers
  - 0x04: Read Input Registers
- [ ] **Test write operations:**
  - 0x05: Write Single Coil
  - 0x06: Write Single Register
  - 0x0F: Write Multiple Coils
  - 0x10: Write Multiple Registers
- [ ] **Document any unsupported functions**
  - Exception 0x01 (ILLEGAL FUNCTION)
  - Update device capability documentation

(source: [function-codes.md](/wiki/concepts/function-codes.md))

### 4. Address Range Verification

- [ ] **Read registers at known addresses** from device manual
- [ ] **Verify data values are sensible**
  - Check against expected ranges
  - Confirm units and scaling
- [ ] **Test boundary addresses** (first and last valid addresses)
  - First valid address should work
  - Address before first should return exception 0x02
  - Last valid address should work
  - Address after last should return exception 0x02
- [ ] **Confirm address ranges match documentation**

## Troubleshooting Common Issues

### No Communication at All

**Symptoms:**
- All requests timeout
- No response from any device
- No error exceptions

**Diagnostic Steps:**
- [ ] **Check D1 and D0 not swapped**
  - Most common wiring error
  - Swap wires and test
- [ ] **Verify 120Ω terminators at both ends**
  - Measure resistance with power off
  - Should read ~60Ω between D1-D0
- [ ] **Confirm baud rate matches on all devices**
  - Check master and all slave settings
  - Common mismatch: 9600 vs 19200
- [ ] **Check parity and stop bit settings**
  - Must match exactly on all devices
  - 8E1 most common for RTU
- [ ] **Verify device powered on**
  - Check power LEDs
  - Verify power supply voltage
- [ ] **Check slave address configuration**
  - Verify master polling correct address
  - Check device DIP switches or config

(source: [rs485-wiring-for-modbus-rtu.md](/wiki/answers/rs485-wiring-for-modbus-rtu.md:723-738))

### Intermittent Communication

**Symptoms:**
- Sometimes works, sometimes fails
- Occasional timeouts
- Random CRC errors (RTU)

**Diagnostic Steps:**
- [ ] **Check all screw terminals are tight**
  - Loose terminals cause intermittent contact
  - Re-torque all connections
- [ ] **Inspect cable for damage**
  - Look for cuts, pinches, or abrasion
  - Check cable routing through doors/panels
- [ ] **Verify shield connected at one end only**
  - Ground loops cause noise
  - Shield must float at all but one location
- [ ] **Confirm termination resistors present and correct value**
  - Verify 120Ω at both ends
  - Check no extra terminators on middle devices

(source: [rs485-wiring-for-modbus-rtu.md](/wiki/answers/rs485-wiring-for-modbus-rtu.md:739-752))

### Works Short Distance, Fails Long Distance

**Symptoms:**
- First few devices work
- Distant devices timeout or errors
- Works when cable length reduced

**Diagnostic Steps:**
- [ ] **Verify cable length ≤1000m at 9600 baud**
  - Measure actual cable run
  - Check against baud rate limits
- [ ] **Try lower baud rate for longer distances**
  - 9600 baud: 1000m max
  - 4800 baud: 1200m max
- [ ] **Confirm cable is twisted pair, shielded**
  - Cat5e can work up to 600m
  - Industrial RS-485 cable preferred
- [ ] **Verify device count ≤32**
  - Too many devices load bus excessively
  - Consider repeaters or low-load transceivers

(source: [rs485-wiring-for-modbus-rtu.md](/wiki/answers/rs485-wiring-for-modbus-rtu.md:753-766))

### Random Errors and Noise

**Symptoms:**
- Occasional CRC errors
- Random invalid data
- Works better when equipment turned off

**Diagnostic Steps:**
- [ ] **Verify shield connected one end only** (no ground loops)
  - Check shield grounding at master
  - Verify floating at all slaves
- [ ] **Add polarization resistors if not present**
  - 560Ω pull-up on D1
  - 560Ω pull-down on D0
  - Install at master only
- [ ] **Increase separation from power cables**
  - Minimum 300mm from AC power
  - More separation in noisy environments
- [ ] **Check for nearby noise sources**
  - VFDs (variable frequency drives)
  - Motors and contactors
  - Welding equipment
  - Radio transmitters

(source: [rs485-wiring-for-modbus-rtu.md](/wiki/answers/rs485-wiring-for-modbus-rtu.md:767-780))

## Exception Response Testing

### Test Error Handling

**Verify client handles exceptions gracefully:**

- [ ] **Attempt read of non-existent register**
  - Should receive exception 0x02 (ILLEGAL DATA ADDRESS)
  - Verify client detects and logs exception
- [ ] **Try unsupported function code**
  - Should receive exception 0x01 (ILLEGAL FUNCTION)
  - Verify client handles gracefully
- [ ] **Request too many registers**
  - Request 200 registers (exceeds max 125)
  - Should receive exception 0x03 (ILLEGAL DATA VALUE)
- [ ] **Write invalid coil value**
  - Try writing 0x0001 instead of 0xFF00
  - Should receive exception 0x03
- [ ] **Verify client handles exceptions gracefully**
  - No crashes
  - Proper error messages
  - Appropriate retry logic

**Exception Code Quick Reference:**

| Code | Name | Test Scenario |
|------|------|---------------|
| 0x01 | ILLEGAL FUNCTION | Use unsupported function code |
| 0x02 | ILLEGAL DATA ADDRESS | Read non-existent address |
| 0x03 | ILLEGAL DATA VALUE | Request quantity >125 or invalid coil value |
| 0x04 | SERVER DEVICE FAILURE | (Device-dependent, hard to test) |
| 0x06 | SERVER DEVICE BUSY | (Timing-dependent, may occur under load) |

(source: [modbus-errors-and-exceptions.md](/wiki/answers/modbus-errors-and-exceptions.md:113-228))

## Performance Testing

### 1. Response Time Testing

- [ ] **Measure typical response times**
  - Use Wireshark for TCP
  - Time request to response
- [ ] **Test with minimum poll interval**
  - Gradually reduce interval
  - Find sustainable rate
- [ ] **Verify no timeouts under normal load**
  - Run for extended period (hours)
  - Check for dropped requests
- [ ] **Document maximum sustainable poll rate**
  - Requests per second per device
  - Total network throughput

**Typical Response Times:**
- MODBUS TCP: 5-50ms
- MODBUS RTU at 9600 baud: 50-200ms
- MODBUS RTU at 19200 baud: 25-100ms

### 2. Load Testing

- [ ] **Test with maximum quantity of registers**
  - 125 registers for holding/input registers
  - 2000 coils/discrete inputs
- [ ] **Verify performance with all devices polling**
  - Sequential polling of all slaves
  - Check for timeouts
- [ ] **Check for degradation with multiple clients**
  - Test with concurrent TCP connections
  - Verify fair access

### 3. Stress Testing

- [ ] **Continuous operation test** (24+ hours)
  - Monitor for memory leaks
  - Check for cumulative errors
- [ ] **Power cycle devices during operation**
  - Verify recovery behavior
  - Check reconnection logic
- [ ] **Network disruption testing**
  - Disconnect/reconnect cables
  - Verify timeout handling

## Documentation

### 1. As-Built Documentation

- [ ] **Update network diagram** with actual cable lengths
  - Trunk cable length
  - Drop cable lengths
  - Device positions
- [ ] **Document device addresses and locations**
  - Create address table
  - Physical location descriptions
  - GPS coordinates if applicable
- [ ] **Record serial port settings used**
  - Baud rate
  - Parity
  - Stop bits
  - Format notation (e.g., 19200-8E1)
- [ ] **Note any device-specific quirks or limitations**
  - Maximum quantity restrictions
  - Unsupported functions
  - Special initialization requirements
- [ ] **Photograph installation** (especially terminations)
  - Termination resistor locations
  - Shield grounding point
  - Cable routing
  - Device labeling

### 2. Register Map Documentation

- [ ] **Create register map for each device**
  - Start address
  - Quantity
  - Description
  - Access (R/W or RO)
- [ ] **Document data types and scaling factors**
  - INT16, UINT16, FLOAT32, etc.
  - Byte order (big-endian vs little-endian)
  - Scaling: raw value × factor + offset
  - Engineering units
- [ ] **Note read-only vs read-write registers**
  - Configuration registers (typically R/W)
  - Measurement registers (typically RO)
  - Status registers (typically RO)
- [ ] **Record valid value ranges**
  - Minimum and maximum values
  - Special values (e.g., 0xFFFF = invalid)

(source: [modbus-data-type-mapping.md](/wiki/concepts/modbus-data-type-mapping.md))

### 3. Operational Notes

- [ ] **Document normal response times**
  - Baseline for troubleshooting
  - Expected values per device
- [ ] **Note any periodic maintenance requirements**
  - Firmware updates
  - Configuration backups
  - Cable inspection intervals
- [ ] **Record troubleshooting procedures that worked**
  - Problems encountered during commissioning
  - Solutions that resolved issues
  - Lessons learned
- [ ] **Create contact list for device support**
  - Vendor technical support
  - Device model numbers
  - Serial numbers
  - Purchase dates

### 4. Configuration Backup

- [ ] **Backup all device configurations**
  - Export configuration files
  - Document manual settings (DIP switches)
  - Save to secure location
- [ ] **Document recovery procedures**
  - Steps to restore from backup
  - Factory reset procedures
  - Re-initialization sequence

## Final Validation

### System Acceptance Checklist

- [ ] **All devices respond consistently**
  - 100% success rate over test period
  - No unexplained timeouts
- [ ] **No exception responses during normal operation**
  - All requested addresses valid
  - All function codes supported
- [ ] **Response times within acceptable limits**
  - Meet application requirements
  - Consistent across devices
- [ ] **All required data accessible**
  - Can read all necessary registers
  - Can write all configuration values
- [ ] **Redundancy tested** (if applicable)
  - Failover mechanisms work
  - Backup paths functional
- [ ] **Documentation complete and accessible**
  - As-built drawings available
  - Register maps documented
  - Contact information current
- [ ] **Operators trained on system**
  - Normal operation procedures
  - Basic troubleshooting
  - When to escalate issues

### Handover Package

**Provide to customer/operations team:**
1. As-built network diagram
2. Device address table
3. Register maps for all devices
4. Serial port configuration summary
5. Installation photographs
6. Test results and validation data
7. Troubleshooting guide
8. Vendor contact information
9. Configuration backups
10. Training completion records

## Quick Reference Tables

### Cable Distance vs Baud Rate

| Baud Rate | Maximum Distance | Typical Use |
|-----------|------------------|-------------|
| 2400 bps  | 1200m           | Long runs, few devices |
| 9600 bps  | 1000m           | Standard industrial |
| 19200 bps | 500m            | Medium distance |
| 38400 bps | 250m            | Short distance |
| 115200 bps| 50m             | Very short, high speed |

(source: [modbusoverserial.md](/raw/MODBUS/modbusoverserial.md))

### Termination Specifications

| Parameter | Value |
|-----------|-------|
| Resistance | 120Ω (preferred) or 150Ω |
| Power rating | 0.25W minimum (0.5W better) |
| Location | Both ends of trunk cable ONLY |
| Quantity | Exactly 2 (one at each end) |

### Common Serial Formats

| Format | Meaning | Usage |
|--------|---------|-------|
| 9600-8E1 | 9600 baud, 8 data bits, even parity, 1 stop bit | Most common RTU |
| 19200-8E1 | 19200 baud, 8 data bits, even parity, 1 stop bit | Faster RTU |
| 9600-8N2 | 9600 baud, 8 data bits, no parity, 2 stop bits | Alternative RTU |

### Device Limits

| Configuration | Max Devices |
|---------------|-------------|
| Standard (1 unit load) | 32 |
| With polarization | 28 |
| Low-load (1/4 unit) | 128 |
| Low-load (1/8 unit) | 256 |
| With repeaters | 32 per segment |

(source: [rs485-wiring-for-modbus-rtu.md](/wiki/answers/rs485-wiring-for-modbus-rtu.md:494-545))

### MODBUS Exception Codes

| Code | Name | Quick Fix |
|------|------|-----------|
| 0x01 | ILLEGAL FUNCTION | Check device manual, use supported function |
| 0x02 | ILLEGAL DATA ADDRESS | Verify address in valid range |
| 0x03 | ILLEGAL DATA VALUE | Check protocol limits, split request |
| 0x04 | SERVER DEVICE FAILURE | Check device status, reset device |
| 0x06 | SERVER DEVICE BUSY | Retry with delay |
| 0x0B | GATEWAY TARGET NO RESPONSE | Check slave device, serial bus |

(source: [modbus-errors-and-exceptions.md](/wiki/answers/modbus-errors-and-exceptions.md:1042-1055))

## Related Pages

- [RS-485 Wiring for MODBUS RTU](/wiki/answers/rs485-wiring-for-modbus-rtu.md) - Detailed wiring guide
- [MODBUS Errors and Exception Responses](/wiki/answers/modbus-errors-and-exceptions.md) - Complete error troubleshooting
- [Using Wireshark for MODBUS TCP Analysis](/wiki/answers/wireshark-modbus-analysis.md) - Network troubleshooting tool
- [How MODBUS TCP-to-RTU Gateways Work](/wiki/answers/modbus-tcp-to-rtu-gateway.md) - Gateway configuration
- [MODBUS RTU](/wiki/concepts/modbus-rtu.md) - RTU protocol details
- [MODBUS TCP](/wiki/concepts/modbus-tcp.md) - TCP protocol details
- [Function Codes](/wiki/concepts/function-codes.md) - MODBUS function reference
- [Master-Slave Architecture](/wiki/concepts/master-slave.md) - Communication model

## Backlinks

None yet.
