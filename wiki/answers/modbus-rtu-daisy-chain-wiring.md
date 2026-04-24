---
title: MODBUS RTU Daisy-Chain Wiring Explained
Summary: Clarification that MODBUS RTU daisy-chain wiring is simply the correct RS-485 bus topology, not a special MODBUS feature, with detailed explanation of proper linear topology vs incorrect alternatives.
Sources:
  - raw/MODBUS/modbusoverserial.md
  - raw/MODBUS/MODBUS.md
Categories:
  - physical-layer
  - rs-485
  - wiring
  - topology
type: answer
date-created: 2026-04-24T15:00:00+03:00
last-updated: 2026-04-24T15:00:00+03:00
---

"MODBUS RTU daisy-chain wiring" is **simply the correct way to wire an RS-485 bus** - it's not a special MODBUS feature or something distinct from "normal RS-485." The term "daisy-chain" describes the **linear topology** required by the RS-485 electrical specification.

## Daisy-Chain IS Correct RS-485 Wiring

### No Distinction

**Key point:** There is **no difference** between:
- "MODBUS RTU daisy-chain wiring"
- "Correctly wired RS-485 bus"
- "RS-485 linear topology"
- "RS-485 multi-drop network"

These all describe the **same thing**: a linear trunk cable with devices connected along its length.

### Why the Term "Daisy-Chain"?

The term describes how devices are connected **in series along a line**, like daisies in a chain:

```
Term ──[Device 1]──[Device 2]──[Device 3]──[Device 4]──[Device 5]── Term
```

Each device is connected to the main trunk, one after another, forming a "chain."

**Origin:** The term comes from the physical appearance when devices are connected sequentially, similar to linking daisies together to form a garland.

## What IS Daisy-Chain Wiring?

### Correct Linear Topology

**Daisy-chain topology (CORRECT):**

```
Termination                                                    Termination
   120Ω                                                            120Ω
     │                                                                │
  ───┴────────────────────────────────────────────────────────────┴───
  │                                                                    │
  │                    Main Trunk Cable                                │
  │                    (continuous run)                                │
  └──┬──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┬───────┘
     │      │      │      │      │      │      │      │      │
     │      │      │      │      │      │      │      │      │
   Master Slave1 Slave2 Slave3 Slave4 Slave5 Slave6 Slave7 Slave8
  (short (short (short (short (short (short (short (short (short
   drop)  drop)  drop)  drop)  drop)  drop)  drop)  drop)  drop)
```

**Characteristics:**
- **Linear trunk cable** runs the full length of the network
- **Devices connected at points along trunk** via short "drop" cables
- **Two terminators** at the extreme ends of the trunk only
- **No branches or stars** in the trunk cable itself

This is the **RS-485 electrical specification requirement**, not a MODBUS-specific design choice.

## Why Daisy-Chain is Required

### RS-485 Electrical Physics

RS-485 uses **transmission line theory** where signals travel as electromagnetic waves along the cable (source: [modbusoverserial.md](/raw/MODBUS/modbusoverserial.md:2365-2370)).

**What happens when a signal travels:**

1. **Transmitter sends signal** → Creates electromagnetic wave on cable
2. **Wave travels down cable** → Propagates at ~60-80% speed of light
3. **Wave reaches discontinuity** → Impedance change causes reflection
4. **Reflected wave travels back** → Can interfere with new signals

**Termination resistors (120Ω) match the cable's characteristic impedance** (~120Ω for twisted pair), absorbing the signal and preventing reflections.

### Why Linear Topology Matters

**Problem with branches/stubs:**

```
                     ┌─── Stub creates
                     │    impedance mismatch
  ─────────┬─────────┼─────────
           │         └─ Device
        Reflection
        occurs here
```

- Stub creates impedance discontinuity
- Causes signal reflections
- Reflections corrupt data
- Communication errors result

**Linear topology avoids this:**

```
  ─────────┬─────────────────  Short drop OK
           │                   (minimal reflection)
         Device
```

- Short drops (<20m) acceptable
- Minimal impedance discontinuity
- Reflections negligible

## Daisy-Chain vs Incorrect Topologies

### Correct: Daisy-Chain (Linear)

```
  120Ω                                                              120Ω
  Term                                                              Term
    ├─────────────────────────────────────────────────────────────┤
    │                                                               │
    │                  Trunk Cable (linear)                        │
    │                                                               │
    └──┬──────┬──────┬──────┬──────┬──────┬──────┬──────┬──────┬──┘
       │      │      │      │      │      │      │      │      │
     Dev 1  Dev 2  Dev 3  Dev 4  Dev 5  Dev 6  Dev 7  Dev 8  Dev 9
```

**✅ Advantages:**
- Proper impedance matching
- Clean signal transmission
- Minimal reflections
- Reliable communication
- Easy to terminate (both ends)

**✅ This is the ONLY correct RS-485 topology for multi-drop.**

### Incorrect: Star Topology

```
                      Master
                        │
                ┌───────┼───────┐
                │       │       │
              Slave1  Slave2  Slave3
```

**❌ Problems:**
- Multiple stubs create reflections
- Where to place terminators? (No clear "ends")
- Impedance mismatches everywhere
- Unreliable communication
- **Does not work with 2-wire RS-485**

**Only acceptable for:**
- 4-wire RS-485 (separate TX/RX)
- Very short distances (<5m)
- Specialized hubs/repeaters

**❌ Do NOT use star topology for MODBUS RTU (2-wire RS-485).**

### Incorrect: Multiple T-Junctions

```
  ──────┬───────────┬───────────┬──────────
        │           │           │
        │           │           │
      Stub        Stub        Stub
        │           │           │
      Dev 1       Dev 2       Dev 3
```

**❌ Problems:**
- Multiple stubs (long drop cables)
- Each stub causes reflections
- Cumulative signal degradation
- Communication unreliable at higher baud rates

**If you must use longer drops:**
- Keep them as short as possible
- Use only a few devices
- Lower baud rate
- But better: rewire to proper daisy-chain

### Incorrect: Ring/Loop Topology

```
  Master──Slave1──Slave2──Slave3──Slave4
    │                                  │
    └──────────────────────────────────┘
```

**❌ Problems:**
- Creates two parallel paths
- Terminators can't be placed correctly
- Signal reflections from both directions
- **Does NOT work with RS-485**

**Note:** Some protocols support ring topology (PROFIBUS, some CAN implementations), but **RS-485 does NOT**.

## Anatomy of Daisy-Chain Connection

### Trunk Cable

**Main bus running full length:**

```
  [Start]════════════════════════════════════════════[End]
           Continuous twisted pair cable
           (no breaks except at device connections)
```

**Requirements (source: [modbusoverserial.md](/raw/MODBUS/modbusoverserial.md:2349-2356)):**
- **Maximum length:** 1000m at 9600 baud (AWG24 or wider)
- **Cable type:** Shielded twisted pair
- **Impedance:** 100-130Ω characteristic impedance
- **Continuous run:** No breaks except at device connections

### Device "Drops" (Short Connections)

**Short cables from trunk to devices:**

```
     Trunk ═══════╤═══════
                  │
                  │ Drop cable
                  │ (<20m)
                  │
               ┌──┴──┐
               │ Dev │
               └─────┘
```

**Requirements:**
- **Maximum drop length:** 20m (source: [modbusoverserial.md](/raw/MODBUS/modbusoverserial.md:2355))
- **Shorter is better:** <5m ideal, <10m good, <20m maximum
- **Same cable type:** Shielded twisted pair
- **No termination on drops:** Terminators only on trunk ends

**Why short drops?**
- Long drops create "stubs" (unterminated cable segments)
- Stubs cause signal reflections
- Short stubs have negligible reflections
- Long stubs degrade signal quality

### Terminators

**120Ω resistors at extreme ends only:**

```
  120Ω                                                            120Ω
  Term                                                            Term
   ├───────────────────────────────────────────────────────────────┤
   │                                                                │
   └─ First device                                Last device ─────┘
```

**Critical rules:**
- **Exactly 2 terminators** (one at each end)
- **At the extreme ends** of trunk cable
- **Not on drop cables** (creates incorrect impedance)
- **Not in the middle** of trunk

## Physical Wiring Methods

### Method 1: Direct Connection (In-Out)

**Devices with built-in bus connections:**

```
  Trunk In ──→ │Device│ ──→ Trunk Out
               │  1   │
               │      │
               └──────┘
                                    
  Trunk In ──→ │Device│ ──→ Trunk Out
               │  2   │
               │      │
               └──────┘
```

**How it works:**
- Device has "In" and "Out" terminals
- Trunk cable passes through device
- Internal connection to D1, D0, COM

**Advantages:**
- True daisy-chain (cable continuity)
- Clean wiring
- No external junction needed

**Common in:**
- DIN-rail mounted devices
- Professional industrial equipment
- Terminal blocks with pass-through

### Method 2: Junction/Tap Point

**Using junction blocks or connectors:**

```
  Trunk ════════╤════════
                │
           ┌────┴────┐
           │ Junction│
           │  Block  │
           └────┬────┘
                │ Drop
              Device
```

**How it works:**
- Trunk cable interrupted at junction
- Junction block connects trunk + drop
- D1, D0, COM all connected together

**Advantages:**
- Flexible device positioning
- Easy to add/remove devices
- Standard terminal blocks

**Common in:**
- Field installations
- Retrofit applications
- Test/development

### Method 3: Parallel Splice

**Direct wire splice (less common):**

```
  Trunk ═══════╪═══════
               │
            Splice
               │
             Device
```

**How it works:**
- Trunk cable stripped at device location
- Drop cable wires spliced to trunk wires
- Insulated and protected

**Advantages:**
- No additional hardware needed
- Direct electrical connection

**Disadvantages:**
- Difficult to remove device later
- Potential reliability issues
- Not recommended for professional installations

**Better alternative:** Use junction block (Method 2)

## Practical Daisy-Chain Examples

### Small System (5 Devices)

```
Master                  Slave1    Slave2    Slave3    Slave4
(0m)                    (20m)     (40m)     (60m)     (80m)
 ┌───┐                   ┌───┐     ┌───┐     ┌───┐     ┌───┐
 │485├─ 120Ω Term        │485│     │485│     │485│     │485├─ 120Ω Term
 └─┬─┘                   └─┬─┘     └─┬─┘     └─┬─┘     └─┬─┘
   │                       │         │         │         │
   └───────────────────────┴─────────┴─────────┴─────────┘
            80m trunk cable (straight run)
            Each device <5m from trunk
```

**Wiring:**
- 80m trunk cable (main bus)
- Terminator at Master (first device)
- Terminator at Slave4 (last device)
- No terminators at Slaves 1-3 (middle devices)
- All drops very short (<5m)

### Large System (20 Devices)

```
Master                                                     Repeater
(0m)    S1    S2    S3    S4    S5 ... S9               (500m)
 ├───────┼─────┼─────┼─────┼─────┼─────┼─────────────────┤
 │                                                        │
 │              Segment 1 (500m)                          │
 │                                                        │
 └────────────────────────────────────────────────────────┘
 
                                                         Repeater
                                                         (500m)    S11... S20
                                                          ├─────────┼─────┼───
                                                          │                │
                                                          │   Segment 2    │
                                                          │   (500m)       │
                                                          └────────────────┘
```

**Wiring:**
- Two segments (repeater isolates them)
- Each segment: 500m trunk + terminators
- Total length: 1000m
- 10 devices per segment
- All devices along linear trunk

### Office/Building Installation

```
Floor:
  Server Room        Office 1        Office 2        Lab
  (Master)           (Sensor 1)      (Sensor 2)      (Equipment)
     │                   │               │               │
     └───────────────────┴───────────────┴───────────────┘
              Trunk cable runs through cable tray
              or conduit along hallway
```

**Installation considerations:**
- Run trunk in cable tray/conduit
- Keep away from power cables
- Short drops into each room
- Terminators in server room and lab

## Common Mistakes and Corrections

### Mistake 1: Star from Master

**Wrong:**
```
           Master
             │
      ┌──────┼──────┐
      │      │      │
    Slave1 Slave2 Slave3
```

**Correct:**
```
  Master ─── Slave1 ─── Slave2 ─── Slave3
  (120Ω)                             (120Ω)
```

### Mistake 2: Long Drop Cables

**Wrong:**
```
  Trunk ═══════════════════════════
                   │
                   │ 50m drop
                   │ (too long!)
                 Device
```

**Correct:**
```
  Trunk ═══════════════════════════
                   │
                   │ <20m drop
                   │
                 Device
```

Or better: Route trunk closer to device.

### Mistake 3: Terminator on Every Device

**Wrong:**
```
  Dev1──Term  Dev2──Term  Dev3──Term  Dev4──Term
  (Too many terminators!)
```

**Correct:**
```
  Dev1──Term  Dev2  Dev3  Dev4──Term
  (First)                    (Last)
```

### Mistake 4: No Terminators

**Wrong:**
```
  Master ─── Slave1 ─── Slave2 ─── Slave3
  (No terminators - signal reflections!)
```

**Correct:**
```
  Master ─── Slave1 ─── Slave2 ─── Slave3
  (120Ω)                             (120Ω)
```

## Why MODBUS Uses Daisy-Chain

### MODBUS Doesn't Specify Topology

**Important:** MODBUS protocol specification **does not define physical wiring topology**.

MODBUS specifies:
- Application layer protocol (function codes, data model)
- Transmission modes (RTU, ASCII)
- Serial parameters (baud rate, parity, etc.)

MODBUS **does not** specify:
- Physical layer topology
- Cable routing
- Termination requirements

### RS-485 Specifies Topology

The daisy-chain requirement comes from **RS-485 electrical specification** (TIA/EIA-485), not MODBUS.

**RS-485 standard defines:**
- Differential signaling
- Transmission line characteristics
- Termination requirements
- Multi-drop capability

Since MODBUS RTU typically uses RS-485, it **inherits** RS-485's daisy-chain topology requirement.

### Any Protocol on RS-485 Uses Daisy-Chain

**Other protocols using RS-485 also require daisy-chain:**
- PROFIBUS DP (process field bus)
- BACnet MS/TP (building automation)
- DMX512 (lighting control)
- MODBUS RTU (industrial automation)

This is **RS-485 physics**, not protocol choice.

## Terminology Clarification

### All These Mean the Same Thing:

- "Daisy-chain wiring"
- "Linear topology"
- "Bus topology"
- "Multi-drop network"
- "Trunk and drop topology"
- "Correctly wired RS-485"

### Marketing Terms to Ignore:

Some manufacturers use confusing marketing:
- ❌ "MODBUS daisy-chain mode" (no such mode exists)
- ❌ "Special MODBUS daisy-chain feature" (it's just RS-485)
- ❌ "MODBUS chain configuration" (it's RS-485 wiring)

**Reality:** If it's RS-485, it needs daisy-chain wiring. Period.

## Quick Reference

### Daisy-Chain Checklist

**✅ Correct daisy-chain wiring:**
- [ ] Linear trunk cable (straight run)
- [ ] Shielded twisted pair cable
- [ ] Terminators at both ends ONLY (120Ω)
- [ ] No terminators on middle devices
- [ ] No terminators on drop cables
- [ ] Short drop cables (<20m, preferably <10m)
- [ ] No star/branch topology
- [ ] No ring/loop topology
- [ ] Trunk runs full length of network
- [ ] Total trunk length ≤ 1000m @ 9600 baud

### Key Dimensions

| Parameter | Value |
|-----------|-------|
| **Trunk maximum length** | 1000m @ 9600 baud |
| **Drop maximum length** | 20m (shorter better) |
| **Drop ideal length** | <5m |
| **Terminator location** | Both ends of trunk only |
| **Terminator value** | 120Ω (±5%) |
| **Number of terminators** | Exactly 2 |

## Summary

### Key Points

1. **"Daisy-chain" is simply correct RS-485 wiring** - not a MODBUS-specific feature
2. **Linear topology is required by RS-485 physics** - prevents signal reflections
3. **All RS-485 protocols use daisy-chain** - MODBUS, PROFIBUS, BACnet MS/TP, etc.
4. **Star/branch topologies don't work** - create impedance mismatches and reflections
5. **Short drops are critical** - long drops create problematic stubs
6. **Two terminators at trunk ends** - absorb signals, prevent reflections

### The Answer to Your Question

> "Is daisy-chain simply what happens from correctly wiring an RS-485 bus?"

**YES, exactly.** Daisy-chain **IS** correctly wired RS-485. There's no distinction between:
- MODBUS RTU daisy-chain wiring
- Correctly wired RS-485 bus

They are **the same thing**. The term "daisy-chain" just describes the linear topology required by RS-485 electrical specifications.

> "Or is RTU daisy chaining distinct from normal RS-485?"

**NO distinction.** RTU uses RS-485, so it inherits RS-485's topology requirements. "MODBUS RTU daisy-chain" is just another way of saying "RS-485 linear topology for MODBUS."

### Practical Takeaway

When someone says "wire MODBUS RTU in daisy-chain," they mean:
1. Use linear topology (trunk cable with short drops)
2. Terminate both ends (120Ω resistors)
3. Don't create stars or branches

This is **standard RS-485 wiring** - nothing special about MODBUS.

## Related Pages

- [RS-485 Wiring for MODBUS RTU](/wiki/answers/rs485-wiring-for-modbus-rtu.md) - Complete RS-485 wiring guide
- [MODBUS RTU](/wiki/concepts/modbus-rtu.md) - MODBUS RTU protocol specification
- [Master-Slave Architecture](/wiki/concepts/master-slave.md) - MODBUS communication model

## Backlinks

None yet.
