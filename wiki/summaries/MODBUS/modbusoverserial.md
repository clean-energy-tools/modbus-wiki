---
title: MODBUS Over Serial Line Specification
Summary: MODBUS serial line specification covering master-slave protocol, RTU and ASCII modes, RS485 physical layer, and framing requirements for serial communication.
Sources:
  - raw/MODBUS/modbusoverserial.md
Categories:
  - specifications
  - serial-communication
  - physical-layer
type: summary
date-created: 2026-04-18T12:00:00+03:00
last-updated: 2026-04-18T14:43:52+03:00
---

This document describes the MODBUS application protocol for serial line transmission, covering both RTU and ASCII transmission modes, the master-slave protocol model, and the RS-485 physical layer requirements (source: [modbusoverserial.md](/raw/MODBUS/modbusoverserial.md)).

## Protocol Overview

MODBUS Serial Line is a master-slave protocol where:
- One master device initiates all transactions
- Up to 247 slave devices can be addressed (addresses 1-247)
- Address 0 is reserved for broadcast (no response expected)
- Slaves only respond when addressed by the master (source: [modbusoverserial.md](/raw/MODBUS/modbusoverserial.md))

### Communication Models

**Unicast Mode:**
- Master → specific slave → slave responds

**Broadcast Mode:**
- Master → all slaves (address 0) → no response
- All slaves execute the command but do not reply

## Transmission Modes

MODBUS supports two transmission modes over serial lines:

### RTU Mode (Remote Terminal Unit)

[modbus-rtu](/wiki/concepts/modbus-rtu.md) transmits data in binary format with CRC-16 error checking.

**Character Format:**
| Field | Bits | Description |
|-------|------|-------------|
| Start bit | 1 | Always 0 |
| Data bits | 8 | LSB first |
| Parity bit | 1 | Even parity (default), odd, or none |
| Stop bit(s) | 1-2 | 1 with parity, 2 without parity |

**Default configuration:** 8 data bits, even parity, 1 stop bit (8E1)
**Alternative (no parity):** 8 data bits, no parity, 2 stop bits (8N2)

#### RTU Frame Structure

```
[Address: 1][Function Code: 1][Data: 0-252][CRC-16: 2]
```

**Maximum RTU ADU: 256 bytes** (1 address + 253 PDU + 2 CRC)

#### RTU Framing

Frames are delimited by silent intervals:

| Timer | Duration | Purpose |
|-------|----------|---------|
| t1.5 | 1.5 character times | Max gap between characters within frame |
| t3.5 | 3.5 character times | Min gap between frames |

**Timing calculation:**
```
Character time (seconds) = 11 / baud_rate  (with parity)
                         = 10 / baud_rate  (no parity)

t1.5 = 1.5 × character_time
t3.5 = 3.5 × character_time
```

**Timing examples at 9600 baud (11 bits/char):**
- Character time = 1.146 ms
- t1.5 = 1.72 ms
- t3.5 = 4.01 ms

**High baud rate rule:** For baud rates > 19200 bps, use fixed timing:
- t1.5 = 750 µs
- t3.5 = 1.75 ms

#### CRC-16 Calculation

| Parameter | Value |
|-----------|-------|
| Polynomial | 0x8005 (bit-reversed: 0xA001) |
| Initial value | 0xFFFF |
| Input reflection | Yes (LSB first) |
| Output reflection | Yes |
| Final XOR | 0x0000 |

**Important:** CRC bytes are transmitted LSB first (little-endian), unlike all other MODBUS fields.

### ASCII Mode

[modbus-ascii](/wiki/concepts/modbus-ascii.md) transmits each byte as two ASCII hexadecimal characters.

**Character Format:**
| Field | Bits | Description |
|-------|------|-------------|
| Start bit | 1 | Always 0 |
| Data bits | 7 | ASCII character, LSB first |
| Parity bit | 1 | Even parity (default) |
| Stop bit(s) | 1-2 | 1 with parity, 2 without |

**Default configuration:** 7 data bits, even parity, 1 stop bit (7E1)

#### ASCII Frame Structure

```
[':'][Address: 2 chars][Function Code: 2 chars][Data: 2N chars][LRC: 2 chars][CR][LF]
```

| Field | Value |
|-------|-------|
| Start delimiter | ':' (0x3A) |
| End delimiter | CR LF (0x0D 0x0A) |
| Data encoding | Each byte → 2 ASCII hex chars |

**Maximum ASCII ADU: 513 characters**

#### LRC Calculation

LRC (Longitudinal Redundancy Check) is the two's complement of the 8-bit sum of all bytes (address through last data byte).

```
LRC = (-(sum of bytes) AND 0xFF)
```

## Physical Layer: RS-485

### Two-Wire RS-485 Configuration

The standard MODBUS serial configuration uses RS-485 2-wire (half-duplex):

| Signal | Description |
|--------|-------------|
| D1 (B/B') | Data line 1, V1 voltage (V1 > V0 for binary 1/OFF) |
| D0 (A/A') | Data line 0, V0 voltage (V0 > V1 for binary 0/ON) |
| Common (C) | Signal and power supply common |

### Bus Specifications

| Parameter | Value |
|-----------|-------|
| Max devices | 32 (without repeater) |
| Max cable length | 1000m at 9600 baud (AWG24 or wider) |
| Max derivation length | 20m |
| Baud rates (required) | 9600, 19200 (default: 19200) |
| Baud rates (optional) | 1200, 2400, 4800, 38400, 56K, 115K |
| Baud rate tolerance | ±1% transmit, ±2% receive |

### Line Termination

- Required at **both ends** of trunk cable
- **Termination resistor:** 150Ω (0.5W) between D0 and D1
- Alternative: 120Ω + 1nF capacitor in series (for biased lines)

### Line Polarization (Biasing)

When bus is idle, receivers may need bias to prevent noise triggering:

| Resistor | Connection | Value |
|----------|------------|-------|
| Pull-up | D1 to +5V | 450-650Ω |
| Pull-down | D0 to Common | 450-650Ω |

Only implement at **one location** (typically master). Using 650Ω allows more devices on bus.

## Frame State Machine

### RTU Frame Reception Rules

1. Frame starts after t3.5 silence followed by first character
2. Characters must arrive within t1.5 of each other
3. If t1.5 exceeded between characters → frame error, discard
4. Frame ends when t3.5 silence detected after last character
5. Validate CRC-16 before processing

### Master Operations

**Transmission:**
1. Wait for t3.5 silence before sending
2. Transmit frame as continuous byte stream
3. Start response timeout after transmission
4. Handle timeout or process response

**Reception:**
1. Detect frame start after t3.5 silence
2. Accumulate bytes, verify t1.5 inter-character timing
3. Detect frame end on t3.5 silence
4. Validate CRC-16
5. Verify response address matches request

### Slave Operations

**Reception:**
1. Detect frame start after t3.5 silence
2. Validate t1.5 inter-character timing
3. Check CRC-16
4. Verify address (own address or broadcast 0)
5. Discard frames with errors silently (no response)

**Processing:**
1. Parse function code and data
2. Execute requested operation or generate exception
3. For broadcast: execute but don't respond

**Transmission:**
1. Build response with CRC-16
2. Wait for turnaround time if needed
3. Transmit all bytes as continuous stream

## Comparison: RTU vs ASCII

| Characteristic | RTU | ASCII |
|---------------|-----|-------|
| Efficiency | High (binary) | Low (ASCII hex) |
| Error checking | CRC-16 | LRC |
| Timing requirements | Strict (t1.5, t3.5) | Relaxed (1s timeout) |
| Character time | 11 bits | 10 bits |
| Debugging | Requires binary analyzer | Human-readable |
| Maximum frame size | 256 bytes | 513 characters |

## Related pages

- [modbus-rtu](/wiki/concepts/modbus-rtu.md)
- [modbus-ascii](/wiki/concepts/modbus-ascii.md)
- [crc-16](/wiki/concepts/crc-16.md)
- [lrc](/wiki/concepts/lrc.md)
- [function-codes](/wiki/concepts/function-codes.md)
- [modbus-tcp](/wiki/concepts/modbus-tcp.md)
