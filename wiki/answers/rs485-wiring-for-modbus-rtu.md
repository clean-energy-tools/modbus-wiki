---
title: RS-485 Wiring for MODBUS RTU
Summary: Comprehensive guide to RS-485 physical wiring for MODBUS RTU including topology, voltage levels, 2-wire vs 4-wire configurations, shielding, termination resistors, polarization, device limits, and practical installation guidelines.
Sources:
  - raw/MODBUS/modbusoverserial.md
  - raw/MODBUS/MODBUS.md
Categories:
  - physical-layer
  - rs-485
  - wiring
  - installation
  - hardware
type: answer
date-created: 2026-04-24T14:00:00+03:00
last-updated: 2026-04-24T14:00:00+03:00
---

RS-485 is the physical wiring standard used by MODBUS RTU for serial communication. Understanding proper RS-485 wiring is essential for reliable MODBUS networks, because poor wiring causes most communication problems in MODBUS installations.

## RS-485 Physical Wiring Basics

### What is RS-485?

RS-485 (also called TIA/EIA-485) is a wiring standard that allows:
- **Multiple devices on one cable** (multi-drop networks)
- **Long distances** (up to 1200 meters / about 4000 feet)
- **Resistance to electrical noise** (uses differential signaling)
- **High speeds** (up to 10 Mbps, though MODBUS typically uses 9600-115200 bps)

### How Differential Signaling Works

RS-485 uses **differential signaling** with two wires that carry opposite signals (source: [modbusoverserial.md](/raw/MODBUS/modbusoverserial.md)):

```
Device A                        Device B
Sending                         Receiving
  ├─── D1 (A) ─────────────────── D1 (A)
  │                                 │
  │    + 2V (represents 1)          │
  │    or                           │ Reads the
  │    - 2V (represents 0)          │ difference
  │                                 │
  └─── D0 (B) ─────────────────── D0 (B)
```

**Wire names** (these names are used interchangeably):
- **D1 / A / TXD1:** Positive signal wire
- **D0 / B / TXD0:** Negative signal wire  
- **Common / GND:** Signal reference (ground)

**Voltage differences:**
- Binary 1: Voltage between D1 and D0 = +2V to +6V
- Binary 0: Voltage between D1 and D0 = -2V to -6V
- Minimum to detect: ±200mV difference

## Voltage Range for MODBUS RTU

### Signal Voltage Levels

**Differential voltage (D1 - D0):**

| State | Transmitter Output | Receiver Threshold |
|-------|-------------------|-------------------|
| Logic 1 (mark) | +1.5V to +6V | > +200mV |
| Logic 0 (space) | -1.5V to -6V | < -200mV |
| Invalid | -200mV to +200mV | Undefined |

**Common mode voltage (both lines relative to ground):**
- Range: -7V to +12V
- Typical: 0V to +5V

**Maximum voltage between any line and ground:**
- Absolute maximum: ±12V
- Normal operation: ±7V

### Power Supply Voltage

RS-485 transceivers typically powered by:
- **5V DC** (most common for MODBUS devices)
- **3.3V DC** (newer low-power devices)
- **12-24V DC** (industrial devices, with onboard regulators)

**Note:** Signal voltage and power supply voltage are different. RS-485 signals are typically ±3V differential, regardless of power supply voltage.

## 2-Wire vs 4-Wire RS-485

### 2-Wire RS-485 (Half-Duplex)

**This is the standard for MODBUS RTU** (source: [modbusoverserial.md](/raw/MODBUS/modbusoverserial.md:173-174)).

**What you need:**
- 2 signal wires: D0 (B) and D1 (A)
- 1 ground wire: Signal Common/GND
- All devices share the same pair of wires
- **Half-duplex:** Only one device can transmit at a time

**Wiring diagram:**
```
Controller             Device 1              Device 2              Device 3
 ┌───┐                  ┌───┐                 ┌───┐                 ┌───┐
 │485│                  │485│                 │485│                 │485│
 └─┬─┘                  └─┬─┘                 └─┬─┘                 └─┬─┘
   │                      │                     │                     │
   │ D1(A)  yellow ───────┴─────────────────────┴─────────────────────┤
   │ D0(B)  brown  ───────┬─────────────────────┬─────────────────────┤
   │ COM    grey   ───────┼─────────────────────┼─────────────────────┤
   │                      │                     │                     │
  120Ω                   ...                   ...                   120Ω
 Terminator                                                        Terminator
```

**Benefits:**
- **Simple wiring:** Only 3 wires total
- **Lower cost:** Less cable, simpler electronics
- **Matches MODBUS:** MODBUS works this way (controller talks, devices respond)
- **One talker at a time:** Devices must stop transmitting after sending their response

### 4-Wire RS-485 (Full-Duplex)

**Optional for MODBUS**, rarely used (source: [modbusoverserial.md](/raw/MODBUS/modbusoverserial.md:174-175)).

**What you need:**
- 4 signal wires: TXD0, TXD1 (for transmitting), RXD0, RXD1 (for receiving)
- 1 ground wire: Signal Common/GND
- Separate paths for sending and receiving
- **Full-duplex:** Can send and receive at the same time

**Wiring diagram (star topology):**
```
        Master
         ┌───┐
         │485│
         └─┬─┘
        TX │ RX
     ┌─────┴─────┐
     │           │
  TX │           │ RX
   ┌─┴─┐       ┌─┴─┐       ┌───┐
   │485│       │485│       │485│
   └───┘       └───┘       └───┘
  Slave 1     Slave 2     Slave 3
```

**Characteristics:**
- **More complex wiring:** 5 wires total (4 signal + 1 common)
- **Higher cost:** More cable, more complex transceivers
- **Limited MODBUS benefit:** MODBUS is inherently half-duplex (master-slave)
- **Rare in practice:** 2-wire is standard for MODBUS

### When is 4-Wire Used?

**Rarely for MODBUS**, because:
- MODBUS is master-slave (inherently half-duplex)
- Only master initiates transactions
- Slaves only respond when addressed
- No benefit to full-duplex in MODBUS

**4-wire might be used for:**
- Point-to-point connections (not multi-drop)
- Non-MODBUS protocols on same hardware
- Special applications requiring simultaneous TX/RX
- Legacy systems

**Recommendation:** Use 2-wire RS-485 for MODBUS RTU unless you have a specific requirement for 4-wire.

## Multiple Device Connection (Multi-Drop Topology)

### Daisy-Chain Topology (Recommended)

**Trunk cable with short drops:**

```
      120Ω Term                                                    120Ω Term
        │                                                              │
  ┌─────┴────────────────────────────────────────────────────────────┴─────┐
  │                                                                          │
  │  Main Trunk Cable (up to 1000m)                                         │
  │                                                                          │
  └──┬────────┬────────────┬────────────┬────────────┬────────────┬────────┘
     │        │            │            │            │            │
     │        │            │            │            │            │
   Master   Slave 1     Slave 2      Slave 3      Slave 4      Slave 5
  (short)   (<20m)       (<20m)       (<20m)       (<20m)       (<20m)
   drop      drop         drop         drop         drop         drop
```

**Guidelines:**
- **Trunk cable:** Main bus runs full length
- **Termination:** 120Ω resistors at **both ends only**
- **Device drops:** Short connections from trunk to devices
  - Maximum drop length: 20m
  - Shorter is better (reduces reflections)
- **No stubs:** Avoid T-connections mid-cable
- **Linear topology:** Devices connected along a line

**Maximum lengths (source: [modbusoverserial.md](/raw/MODBUS/modbusoverserial.md:2349-2356)):**
- Trunk cable: 1000m at 9600 baud (AWG24 or wider)
- Derivation (drop) cables: 20m maximum
- Multi-port tap with N derivations: 40m / N per derivation

### Star Topology (Not Recommended for 2-Wire)

```
           Master
             │
        ┌────┼────┐
        │    │    │
     Slave Slave Slave
```

**Problems with star for 2-wire:**
- Creates impedance mismatches
- Causes signal reflections
- Difficult to terminate properly
- Not recommended for 2-wire RS-485

**Star is only suitable for:**
- 4-wire RS-485 configurations
- Very short cable runs
- Specialized repeater/hub devices

### Bus Connection Best Practices

**DO:**
- ✅ Use linear daisy-chain topology
- ✅ Keep drops short (<20m, shorter is better)
- ✅ Run trunk cable in straight line
- ✅ Terminate at both ends of trunk
- ✅ Connect devices close to trunk

**DON'T:**
- ❌ Create T-junctions
- ❌ Use star topology with 2-wire
- ❌ Put termination on drop cables
- ❌ Exceed maximum trunk length
- ❌ Exceed maximum drop length

## Shielded Cable Requirements

### Must Use Shielded Cable

**RS-485 for MODBUS requires shielded cable** (source: [modbusoverserial.md](/raw/MODBUS/modbusoverserial.md:2968-2969)).

**Shield requirements:**
- Shield connected to **protective ground at one end only**
- If using connectors, connect shield to connector shell
- Use twisted pair for D0/D1 (reduces EMI)
- Shield provides EMI protection

### Why Shielding is Critical

**Without shielding:**
- Susceptible to electrical noise (motors, relays, VFDs)
- Long cable runs act as antennas
- Industrial environments have high EMI
- Data corruption and communication errors

**With shielding:**
- Shield intercepts electromagnetic interference
- Protects differential signal pair
- Allows reliable operation in noisy environments
- Required for industrial installations

### Cable Specifications

**Recommended cable (source: [modbusoverserial.md](/raw/MODBUS/modbusoverserial.md:2971-3022)):**

| Parameter | Value |
|-----------|-------|
| Type | Shielded twisted pair |
| Wire gauge | AWG 24 or wider (AWG 22, 20, 18) |
| Pairs | 1 pair + common (2-wire), 2 pairs + common (4-wire) |
| Characteristic impedance | 100-130Ω (>100Ω preferred for high baud rates) |
| Maximum capacitance | Depends on length and baud rate |
| Shield | Foil or braid, connected one end |

**Standard cables:**
- **Belden 3105A:** 24 AWG, 1 pair, foil shield (common for MODBUS)
- **Belden 9842:** 24 AWG, 2 pair, foil shield (if 4-wire needed)
- **Category 5/5e/6:** Can work up to 600m (source: [modbusoverserial.md](/raw/MODBUS/modbusoverserial.md:3019))
- Industrial RS-485 cable (various manufacturers)

**Recommended wire colors (source: [modbusoverserial.md](/raw/MODBUS/modbusoverserial.md:2978-3010)):**

| Signal | Recommended Color |
|--------|-------------------|
| D1 (A) / TXD1 | Yellow |
| D0 (B) / TXD0 | Brown |
| Common / GND | Grey |
| RXD0 (4-wire optional) | White |
| RXD1 (4-wire optional) | Blue |

### Shield Grounding

**Ground shield at ONE end only:**
```
Device 1 (grounded end)              Device 2 (floating end)
 ┌──────────┐                         ┌──────────┐
 │          │                         │          │
 │  Shield──┴─── Connected to PE      │  Shield (floating, insulated)
 │          │     Protective Earth    │          │
 └──────────┘                         └──────────┘
```

**Why one end only?**
- Prevents ground loops
- Avoids circulating currents
- Eliminates potential differences between grounds

**Which end to ground?**
- Typically master end
- Or central/main grounding point
- Ensure good earth connection

## Termination Resistors

### Purpose of Termination

**Without termination:**
```
Signal travels down cable → Reaches end → Reflects back
→ Causes ringing, distortion → Data errors
```

**With termination:**
```
Signal travels down cable → Absorbed by termination resistor
→ No reflection → Clean signal
```

**Termination eliminates signal reflections** that occur at impedance discontinuities (cable ends) (source: [modbusoverserial.md](/raw/MODBUS/modbusoverserial.md:2365-2370)).

### Termination Requirements

**Mandatory specifications (source: [modbusoverserial.md](/raw/MODBUS/modbusoverserial.md:2383-2390)):**

| Requirement | Value |
|-------------|-------|
| **Location** | Both ends of trunk cable ONLY |
| **Value** | 120Ω (preferred) or 150Ω |
| **Connection** | Between D0 and D1 |
| **Power rating** | 0.25W minimum (0.5W better) |
| **Topology** | 2-wire: 2 terminators, 4-wire: 4 terminators (2 per pair) |

**Termination circuit options:**

**Option 1: Simple resistor (no biasing):**
```
D1 ───┬───[120Ω]───┬─── D0
      │            │
```

**Option 2: RC termination (with biasing):**
```
D1 ───┬───[120Ω]───┬─── D0
      │            │
     [1nF]        [1nF]
      │            │
     ─┴─          GND
```

### Where to Place Terminators

**Correct placement:**
```
 Term                                                              Term
 120Ω                                                              120Ω
  ├──────────────────────────────────────────────────────────────┤
  │                                                                │
  │                    Trunk Cable                                 │
  │                                                                │
  └──┬──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┬───┘
     │      │      │      │      │      │      │      │      │
   Dev 1  Dev 2  Dev 3  Dev 4  Dev 5  Dev 6  Dev 7  Dev 8  Dev 9
```

**Incorrect placement:**
```
❌ NO terminators → Signal reflections, data corruption
❌ Only one terminator → Partial reflections
❌ Terminator in middle → Wrong impedance matching
❌ Terminator on drop cable → Creates stub
❌ More than 2 terminators → Loads bus excessively
```

### Termination Values

**Standard values:**
- **120Ω:** Matches typical twisted pair impedance (recommended)
- **150Ω:** Alternative, slightly less loading

**Why 120Ω?**
- Matches characteristic impedance of twisted pair cable
- Standard RS-485 specification
- Best reflection reduction

**RC termination (120Ω + 1nF capacitor):**
- Provides bias for idle bus (see Polarization section)
- Capacitor blocks DC, passes AC
- Used when polarization needed

### Built-In vs External Terminators

**Built-in (switchable):**
- Many modern devices have built-in 120Ω terminator
- Enabled/disabled via DIP switch or jumper
- Convenient for end devices

**External (discrete resistors):**
- Screw terminal blocks with resistor
- Separate termination module
- Traditional approach

**Configuration:**
- Enable terminators on **first and last** devices only
- Disable terminators on all middle devices

## Line Polarization (Biasing)

### What is Polarization?

When RS-485 bus is **idle** (no device transmitting):
- All transmitters are tri-stated (high impedance)
- Bus voltage is undefined
- Receivers may see noise as data
- Can cause false triggering

**Polarization (biasing) solves this** by forcing idle bus to known state (source: [modbusoverserial.md](/raw/MODBUS/modbusoverserial.md:2396-2426)).

### Polarization Circuit

**Bias resistors force idle state:**
```
     +5V
      │
    [560Ω] Pull-up resistor
      │
D1 ───┼──────────────────────────────────
      │
     ...
      │
D0 ───┼──────────────────────────────────
      │
    [560Ω] Pull-down resistor
      │
     GND (Common)
```

**Effect:**
- Pull-up resistor: Forces D1 high when idle
- Pull-down resistor: Forces D0 low when idle
- Creates differential voltage (D1 - D0 > 200mV)
- Receiver sees valid "mark" (logic 1) when idle

### Polarization Requirements

**When to use polarization:**
- Some devices require it (check device documentation)
- Long cable runs (>100m)
- Noisy environments
- Systems with many devices

**Resistor values (source: [modbusoverserial.md](/raw/MODBUS/modbusoverserial.md:2420-2421)):**
- **450-650Ω** (both pull-up and pull-down)
- 560Ω typical
- 650Ω allows more devices on bus (less loading)

**Critical rules:**
- Implement at **ONE location only** (typically master)
- Do not implement on multiple devices (bus loading)
- Reduces maximum device count by 4

### RC Termination with Bias

**Combined termination and polarization:**
```
     +5V
      │
    [560Ω] Pull-up
      │
D1 ───┼───[1nF]───[120Ω]───[1nF]───┼─── D0
      │                              │
     ...                          [560Ω] Pull-down
                                     │
                                    GND
```

**Advantages:**
- Single circuit provides both functions
- Capacitors block DC (no loading from bias resistors)
- Standard solution for many MODBUS networks

## Maximum Number of Devices

### Standard Limit: 32 Devices

**Without repeaters: 32 devices maximum** (source: [modbusoverserial.md](/raw/MODBUS/modbusoverserial.md:2321)).

**Why 32?**
- RS-485 driver/receiver "unit load" specification
- Standard transceivers = 1 unit load each
- RS-485 standard allows 32 unit loads maximum
- Ensures signal integrity

### Exceeding 32 Devices

**Options to exceed 32 devices:**

**1. Use low-load transceivers:**
- 1/4 unit load devices → 128 devices
- 1/8 unit load devices → 256 devices
- Check device specifications

**2. Use repeaters:**
```
  Segment 1 (32 devices)    Repeater    Segment 2 (32 devices)
  ────────────────────────────┤├──────────────────────────────
  Master + 31 slaves          │├         32 slaves
```
- Repeater isolates segments
- Each segment can have 32 devices
- Multiple segments possible

**3. Use RS-485 hubs/splitters:**
- Active devices that buffer signals
- Can support more devices per segment

### Polarization Impact

**If using polarization:** Maximum devices reduced by 4 (source: [modbusoverserial.md](/raw/MODBUS/modbusoverserial.md:2426)).

- Without polarization: 32 devices
- With polarization: 28 devices
- Bias resistors count as ~4 unit loads

### MODBUS Address Limit

**MODBUS protocol limit: 247 slave addresses** (1-247)
- Address 0: Broadcast
- Address 248-255: Reserved

**But RS-485 physical limit is usually 32 devices**, so:
- Typical MODBUS RTU network: 1 master + 31 slaves = 32 total
- With repeaters or low-load transceivers: more possible

## Terminal Blocks and Connectors

### Screw Terminal Blocks

**Most common for industrial MODBUS:**

```
  ┌─────────────────┐
  │  [D1] [D0] [COM]│
  │   │    │    │   │
  │  ─┴─  ─┴─  ─┴─  │
  │  Screw terminals│
  └─────────────────┘
```

**Advantages:**
- Easy field wiring
- No special tools required
- Reliable connection
- Standard on industrial devices

**Best practices:**
- Label terminals clearly (D1/A, D0/B, COM/GND)
- Use ferrules on stranded wire
- Torque to specification
- Provide strain relief

### RJ45 Connectors

**Sometimes used for MODBUS** (source: [modbusoverserial.md](/raw/MODBUS/modbusoverserial.md:2444-2458)):

**Standard MODBUS pinout (RJ45):**

| Pin | Signal | Color (standard) |
|-----|--------|------------------|
| 1-3 | (unused) | - |
| 4 | D1 (A) | Yellow or Blue/White |
| 5 | D0 (B) | Brown or Blue |
| 6-7 | (unused) | - |
| 8 | Common/GND | Grey or Orange |
| 9 | (unused) | - |

**Advantages:**
- Quick connect/disconnect
- Compact
- Pre-made cables available

**Disadvantages:**
- Not as robust as screw terminals
- Can be confused with other protocols (Ethernet)
- Shield connection may not be reliable

**Important:** If using RJ45, use **shielded connectors** and **shielded cables**.

### D-Sub 9-Pin Connectors

**Also used for MODBUS RS-485:**

**Standard MODBUS pinout (DE-9 / DB-9):**

| Pin | Signal | Direction |
|-----|--------|-----------|
| 1 | Common/GND | - |
| 2 | (unused) | - |
| 3 | D0 (B) / TXD0 | Transmit/Receive |
| 4 | D1 (A) / TXD1 | Transmit/Receive |
| 5-9 | (unused or optional) | - |

**Note:** Pinouts vary by manufacturer. Always check device documentation.

### Pre-Made Terminal Blocks

**Commercial MODBUS terminal blocks available:**

**Features:**
- Built-in 120Ω termination resistors (switchable)
- Built-in bias resistors (optional)
- LED indicators (TX, RX, power)
- DIN rail mounting
- Clear labeling

**Example products:**
- Phoenix Contact TERMITRAB series
- Weidmüller RS-485 terminal blocks
- WAGO RS-485 junction blocks
- Advantech USB-485 series (USB to RS-485 converters with terminals)

**Benefits:**
- Faster installation
- Reduced wiring errors
- Built-in protection (sometimes)
- Professional appearance

## Complete Wiring Example

### Small Industrial Network (5 Devices)

**Scenario:** 1 PLC master, 4 temperature controller slaves

**Bill of materials:**
- Shielded twisted pair cable (AWG 24): ~150m total
- 2× 120Ω termination resistors (0.5W)
- 2× 560Ω bias resistors (optional, if needed)
- Screw terminal blocks or connectors
- Cable ties, labels

**Wiring diagram:**
```
  PLC Master                        Temp 1    Temp 2    Temp 3    Temp 4
  (ground end)                       20m       40m       60m       80m
  ┌─────────┐                       from      from      from      from
  │ RS-485  │                      Master    Master    Master    Master
  │         │
  │ D1  D0  COM                     ┌───┐     ┌───┐     ┌───┐     ┌───┐
  └──┬──┬──┬┘                       │485│     │485│     │485│     │485│
120Ω │  │  │ Shield to ground       └─┬─┘     └─┬─┘     └─┬─┘     └─┬─┘
Term │  │  │                          │         │         │         │ 120Ω
  ┌──┴──┴──┴──────────────────────────┴─────────┴─────────┴─────────┴─┐ Term
  │  │  │                              │         │         │         │ │
  │ D1 D0 COM                          D1 D0 COM D1 D0 COM D1 D0 COM │
  │  │  │                              │  │  │   │  │  │   │  │  │   │
  └──┴──┴──────────────────────────────┴──┴──┴───┴──┴──┴───┴──┴──┴───┘
     │  │  Yellow                       Short drops (<5m each)
     │  └─ Brown
     └──── Grey
  
  Trunk cable (straight run, AWG 24 shielded twisted pair)
```

**Configuration:**
- Baud rate: 19200 bps
- Format: 8E1 (8 data, even parity, 1 stop)
- Termination: 120Ω at PLC and Temp 4
- Polarization: Optional 560Ω resistors at PLC (if needed)
- Shield: Grounded at PLC end only

### Installation Steps

**1. Plan the layout:**
- Linear topology (straight line)
- Shortest practical cable run
- Identify first and last devices (for termination)

**2. Install trunk cable:**
- Run shielded twisted pair along entire length
- Support cable properly (cable trays, conduit)
- Avoid running parallel to power cables (maintain separation)

**3. Connect devices:**
- Wire D1 (yellow) to all D1/A terminals
- Wire D0 (brown) to all D0/B terminals
- Wire COM (grey) to all Common/GND terminals
- Keep drop lengths short

**4. Install termination:**
- 120Ω resistor between D1-D0 at first device
- 120Ω resistor between D1-D0 at last device
- Verify no terminators on middle devices

**5. Ground shield:**
- Connect shield to protective earth at one end (typically master)
- Ensure shield floating at all other ends

**6. Install polarization (if needed):**
- 560Ω pull-up from D1 to +5V
- 560Ω pull-down from D0 to GND
- Install at one location only (master)

**7. Test:**
- Measure resistance D1 to D0 with all devices powered off
  - Should see ~60Ω (two 120Ω terminators in parallel)
  - With polarization: ~60Ω (terminators dominate)
- Power on devices one at a time
- Verify communication before adding next device

## Troubleshooting Wiring Issues

### Common Problems and Solutions

**Problem 1: No communication at all**

**Possible causes:**
- Reversed D1/D0 wiring
- Missing terminators
- Wrong baud rate or serial parameters
- Addressing conflicts

**Diagnostic steps:**
1. Check D1 and D0 are not swapped
2. Verify 120Ω terminators at both ends
3. Measure resistance: ~60Ω between D1-D0 (powered off)
4. Verify all devices same baud rate, parity, stop bits

**Problem 2: Intermittent communication**

**Possible causes:**
- Poor connections (loose screws)
- Cable damage
- Electrical noise interference
- Reflections (missing/wrong termination)

**Diagnostic steps:**
1. Check all screw terminals tight
2. Inspect cable for damage
3. Verify shield connected at one end
4. Verify termination resistors present and correct value

**Problem 3: Works short distance, fails long distance**

**Possible causes:**
- Cable too long for baud rate
- Wrong cable type (too much capacitance)
- Missing termination
- Too many devices (loading)

**Diagnostic steps:**
1. Verify cable length ≤ 1000m at 9600 baud
2. Use lower baud rate for longer distances
3. Check cable is twisted pair, shielded
4. Verify device count ≤ 32

**Problem 4: Works sometimes, random errors**

**Possible causes:**
- Electrical noise (EMI)
- Ground loops
- Inadequate shielding
- Polarization needed but not present

**Diagnostic steps:**
1. Verify shield connected one end only
2. Add polarization resistors if not present
3. Increase separation from power cables
4. Check for nearby noise sources (VFDs, motors)

### Measurement Techniques

**Resistance measurements (power OFF):**
```
Between D1 and D0:
- With 2 terminators (120Ω each): ~60Ω
- With 1 terminator: ~120Ω
- With 0 terminators: Open circuit (>1MΩ)
- With terminator + bias: ~60Ω (bias resistors >> terminator)
```

**Voltage measurements (power ON, idle):**
```
Between D1 and D0:
- Without polarization: 0V to small voltage (undefined)
- With polarization: +200mV to +1V (D1 pulled high)
```

**Oscilloscope (advanced):**
- View differential signal (D1 - D0)
- Should see clean square waves during transmission
- Ringing indicates termination problems
- Noise indicates EMI or grounding issues

## Best Practices Summary

### DO:

✅ **Use shielded twisted pair cable** (AWG 24 or larger)
✅ **Use linear daisy-chain topology** (trunk with short drops)
✅ **Terminate at BOTH ends only** (120Ω resistors)
✅ **Ground shield at ONE end only** (prevents ground loops)
✅ **Keep drop cables short** (<20m, shorter better)
✅ **Use screw terminals** for industrial reliability
✅ **Label all connections clearly** (D1/A, D0/B, COM)
✅ **Document the installation** (device addresses, locations)
✅ **Test incrementally** (add devices one at a time)
✅ **Use polarization if needed** (one location only)

### DON'T:

❌ **Use unshielded cable** (invites noise problems)
❌ **Create star topology** (2-wire RS-485)
❌ **Omit termination** (causes reflections)
❌ **Terminate on drop cables** (creates impedance mismatch)
❌ **Ground shield at both ends** (creates ground loops)
❌ **Exceed 32 devices** (without repeaters or special transceivers)
❌ **Run parallel to power cables** (maintain separation)
❌ **Exceed maximum lengths** (trunk: 1000m, drops: 20m)
❌ **Use Category 5 beyond 600m** (capacitance too high)
❌ **Mix terminators** (use 120Ω at both ends)

## Quick Reference Tables

### Cable Specifications

| Parameter | Value |
|-----------|-------|
| **Type** | Shielded twisted pair |
| **Gauge** | AWG 24 (or 22, 20, 18) |
| **Impedance** | 100-130Ω |
| **Max trunk length** | 1000m @ 9600 baud |
| **Max drop length** | 20m (shorter better) |
| **Shield** | Grounded one end only |

### Termination

| Parameter | Value |
|-----------|-------|
| **Value** | 120Ω (preferred) or 150Ω |
| **Power** | 0.25W minimum (0.5W better) |
| **Location** | Both ends of trunk only |
| **Quantity** | Exactly 2 (one each end) |

### Polarization (Optional)

| Parameter | Value |
|-----------|-------|
| **Pull-up** | 450-650Ω, D1 to +5V |
| **Pull-down** | 450-650Ω, D0 to GND |
| **Location** | One location only (master) |
| **Effect** | Reduces max devices by 4 |

### Device Limits

| Configuration | Max Devices |
|---------------|-------------|
| Standard (1 unit load) | 32 |
| With polarization | 28 |
| Low-load (1/4 unit) | 128 |
| With repeaters | 32 per segment |

### Signal Levels

| Parameter | Value |
|-----------|-------|
| **Differential (logic 1)** | +1.5V to +6V |
| **Differential (logic 0)** | -1.5V to -6V |
| **Receiver threshold** | ±200mV |
| **Common mode** | -7V to +12V |

## Related Pages

- [MODBUS RTU](/wiki/concepts/modbus-rtu.md) - MODBUS RTU protocol details
- [MODBUS RTU vs ASCII Comparison](/wiki/answers/modbus-rtu-vs-ascii.md) - Protocol comparison
- [Master-Slave Architecture](/wiki/concepts/master-slave.md) - MODBUS communication model
- [MODBUS Errors and Exception Responses](/wiki/answers/modbus-errors-and-exceptions.md) - Troubleshooting guide

## Backlinks

- [MODBUS Commissioning Checklist](/wiki/answers/modbus-commissioning-checklist.md)
- [MODBUS Over RS232](/wiki/answers/modbus-over-rs232.md)
- [MODBUS RTU Daisy Chain Wiring](/wiki/answers/modbus-rtu-daisy-chain-wiring.md)
- [MODBUS RTU vs ASCII Comparison](/wiki/answers/modbus-rtu-vs-ascii.md)
- [Master-Slave Architecture](/wiki/concepts/master-slave.md)
- [MODBUS RTU](/wiki/concepts/modbus-rtu.md)

------------

MODBUS is a trademark of the Modbus Organization, Inc.

All information on this page is derived by summarizing and analyzing solely the MODBUS specifications published by the Modbus Organization, Inc.  The specifications are copyright by the Modbus Organization.

By deriving this information solely from the specifications we hope to stay true to the specification.

No infringement is intended.
