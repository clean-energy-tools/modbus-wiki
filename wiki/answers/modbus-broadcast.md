---
title: MODBUS Broadcast
Summary: Comprehensive guide to MODBUS broadcast functionality including broadcast addresses, operation on serial vs TCP networks, effects and limitations, and troubleshooting broadcast issues.
Sources:
  - raw/MODBUS/modbusoverserial.md
  - raw/MODBUS/messagingimplementationguide.md
  - raw/MODBUS/MODBUS.md
Categories:
  - protocol
  - addressing
  - serial-communication
  - troubleshooting
type: answer
date-created: 2026-04-24T13:00:00+03:00
last-updated: 2026-04-24T13:00:00+03:00
---

MODBUS broadcast is a mechanism that allows a master/client to send a single command to all slave/server devices on a network simultaneously. Understanding broadcast addressing, its limitations, and protocol-specific behavior is essential for proper MODBUS network operation.

## What is MODBUS Broadcast?

MODBUS broadcast is a **one-to-many communication mode** where the master sends a single request that is executed by all slaves on the network (source: [modbusoverserial.md](/raw/MODBUS/modbusoverserial.md:391-394)).

### Key Characteristics

| Aspect | Unicast (Normal) | Broadcast |
|--------|------------------|-----------|
| Addressing | Specific slave (1-247) | All slaves (address 0) |
| Response | Single slave responds | No slaves respond |
| Confirmation | Master receives response | No confirmation |
| Typical use | Read/write operations | Write operations only |
| Reliability | High (confirmed) | Lower (unconfirmed) |

**Critical limitation:** Broadcast requests are **necessarily writing commands** (source: [modbusoverserial.md](/raw/MODBUS/modbusoverserial.md:393)). Read operations cannot use broadcast because multiple devices would respond simultaneously, causing bus collisions.

### Communication Flow

**Unicast mode:**
```
Master → Request (Address 5) → Slave 5
Master ← Response             ← Slave 5
```

**Broadcast mode:**
```
Master → Request (Address 0) → All Slaves
                                Slave 1 (executes, no response)
                                Slave 2 (executes, no response)
                                Slave 3 (executes, no response)
                                ...
```

## The Two MODBUS Broadcast Addresses

MODBUS defines **two broadcast address values**, but their usage differs between serial and TCP implementations:

### Address 0: Serial Line Broadcast

**Usage:** MODBUS RTU and MODBUS ASCII on serial lines

**Definition:** Address 0 is reserved as the broadcast address (source: [modbusoverserial.md](/raw/MODBUS/modbusoverserial.md:454)).

**Requirements:**
- All slave nodes **must recognize** the broadcast address
- All slaves **must execute** broadcast write commands
- No slave **may respond** to broadcast requests

**Frame format (MODBUS RTU):**
```
[Address: 0x00][Function Code][Data][CRC-16]
```

**Example: Broadcast write to coil**
```
Request:  00 05 00 0A FF 00 [CRC]
          |  |  |     |
          |  |  |     +------- Value: ON (0xFF00)
          |  |  +------------- Coil address: 10
          |  +---------------- Function code: 0x05 (Write Single Coil)
          +------------------- Address: 0 (broadcast)

Response: (none - all slaves execute but don't respond)
```

### Address 0xFF (255): TCP Direct Connection

**Usage:** MODBUS TCP when addressing a server directly

**Definition:** The value 0xFF is used for the Unit Identifier when addressing a MODBUS server directly connected to TCP/IP (source: [messagingimplementationguide.md](/raw/MODBUS/messagingimplementationguide.md:1917)).

**Important distinction:** This is **not a broadcast address** for MODBUS TCP. Rather, it indicates:
- The MODBUS server is directly connected to the TCP/IP network
- The Unit Identifier field is not significant
- The TCP connection itself uniquely identifies the server

**MBAP header with 0xFF:**
```
[Transaction ID: 2][Protocol ID: 2][Length: 2][Unit ID: 0xFF][PDU...]
                                                        |
                                                        +-- Indicates direct TCP connection
```

**Alternative for TCP:** Address 0 is also accepted for direct MODBUS/TCP communication (source: [messagingimplementationguide.md](/raw/MODBUS/messagingimplementationguide.md:1928-1929)).

### Summary of Addresses

| Address | Protocol | Meaning | Response Expected |
|---------|----------|---------|-------------------|
| 0x00 (0) | Serial (RTU/ASCII) | Broadcast to all slaves | No |
| 0x00 (0) | TCP direct | Direct connection (alternate) | Yes |
| 0xFF (255) | TCP direct | Direct connection (recommended) | Yes |
| 0x01-0xF7 (1-247) | Serial | Specific slave address | Yes |
| 0x01-0xF7 (1-247) | TCP via gateway | Specific slave behind gateway | Yes |

## Effects of a Broadcast Message

### What Happens on the Network

When a broadcast message is sent on a MODBUS serial network:

**1. All slaves receive the message**
- Each slave's MODBUS interface detects the frame
- Each slave checks the address field
- Address 0 is recognized as broadcast

**2. All slaves execute the command**
- Each slave performs the requested write operation
- Coil/register values are updated locally
- Any side effects (relay activation, control logic) occur

**3. No slave responds**
- Slaves suppress their normal response behavior
- No acknowledgment is sent to the master
- The serial bus remains silent after the turnaround delay

**4. Master waits for turnaround delay**
- Master cannot immediately send the next request
- Must wait for "turnaround delay" (typically 100-200ms)
- Allows all slaves time to process the broadcast (source: [modbusoverserial.md](/raw/MODBUS/modbusoverserial.md:626-634))

### Timing Differences

**Unicast timing:**
```
Master sends request
  ↓
Slave processes request
  ↓
Slave sends response (can take 1-several seconds)
  ↓
Master receives response
  ↓
Master can send next request
```

**Broadcast timing:**
```
Master sends broadcast request
  ↓
All slaves process request (no response)
  ↓
Master waits "turnaround delay" (100-200ms)
  ↓
Master can send next request
```

**Key difference:** The turnaround delay is shorter than the response timeout because slaves don't need to formulate and send a response, only process the request (source: [modbusoverserial.md](/raw/MODBUS/modbusoverserial.md:631-634)).

### Valid Broadcast Operations

**Supported broadcast function codes:**
- **0x05:** Write Single Coil
- **0x06:** Write Single Register
- **0x0F:** Write Multiple Coils
- **0x10:** Write Multiple Registers
- **0x16:** Mask Write Register

**Not supported for broadcast:**
- Any read function (0x01, 0x02, 0x03, 0x04)
- Read/Write Multiple Registers (0x17) - has read component
- Diagnostic functions that return data

**Reason:** Read operations would cause multiple slaves to respond simultaneously, creating bus collisions and corrupted data.

### Example: Synchronizing Setpoints

```rust
// Set temperature setpoint to 25.0°C on all controllers
let setpoint: u16 = 2500;  // 25.0°C with scale factor 100

// Broadcast: Write single register
let broadcast_addr = 0;  // Broadcast address
let register_addr = 100;  // Temperature setpoint register

// Build MODBUS RTU frame
let frame = vec![
    broadcast_addr,           // Address: 0 (broadcast)
    0x06,                     // Function: Write Single Register
    (register_addr >> 8) as u8,
    (register_addr & 0xFF) as u8,
    (setpoint >> 8) as u8,
    (setpoint & 0xFF) as u8,
    // CRC would be calculated and appended
];

send_modbus_rtu_frame(&frame);

// Wait turnaround delay (no response expected)
std::thread::sleep(Duration::from_millis(150));

// All controllers now have setpoint = 25.0°C
// No confirmation received - must verify separately if needed
```

## MODBUS Broadcast on Serial vs TCP

### MODBUS RTU/ASCII (Serial Line): Full Broadcast Support

**Broadcast is fully supported** on MODBUS serial networks (source: [modbusoverserial.md](/raw/MODBUS/modbusoverserial.md:391-394)).

**How it works:**
- Serial bus is a shared medium (RS-485 multi-drop)
- All devices physically receive all transmissions
- Address 0 causes all slaves to execute without responding
- Well-defined in the specification

**Typical applications:**
- Synchronizing time across all devices
- Broadcasting configuration updates
- Simultaneous control commands (e.g., emergency stop)
- Updating common parameters

**Requirements:**
- All devices must be on the same serial bus
- All devices must recognize address 0 as broadcast
- Master must wait turnaround delay after broadcast

### MODBUS TCP: No True Broadcast

**MODBUS TCP does NOT support broadcast** in the traditional sense (source: [messagingimplementationguide.md](/raw/MODBUS/messagingimplementationguide.md)).

**Why broadcast doesn't work on TCP:**
1. **Connection-oriented:** TCP uses point-to-point connections
2. **IP addressing:** Each TCP connection addresses a specific server
3. **No shared medium:** Unlike serial RS-485, TCP/IP is not a shared bus
4. **One connection = one device:** Cannot send one TCP packet to multiple servers

**Unit ID behavior on TCP:**
- Unit ID 0xFF (255): Indicates direct connection, not broadcast
- Unit ID 0: Also accepted for direct connection
- Unit ID 1-247: For addressing slaves behind a TCP-to-serial gateway

**To achieve broadcast-like behavior on TCP:**

**Option 1: Multiple unicast connections**
```rust
// Send same command to multiple MODBUS/TCP servers
let servers = vec!["192.168.1.10", "192.168.1.11", "192.168.1.12"];
let command = build_write_command(...);

for server in servers {
    let mut client = ModbusTcpClient::connect(server)?;
    client.send_request(&command)?;
    // Wait for response from each server
    let response = client.receive_response()?;
}
```

**Option 2: Parallel connections**
```rust
// Send to all servers simultaneously using threads/async
use std::thread;

let handles: Vec<_> = servers.iter().map(|server| {
    let server = server.clone();
    let cmd = command.clone();
    thread::spawn(move || {
        let mut client = ModbusTcpClient::connect(&server)?;
        client.send_request(&cmd)?;
        client.receive_response()
    })
}).collect();

// Wait for all to complete
for handle in handles {
    handle.join().unwrap()?;
}
```

**Option 3: UDP multicast (non-standard)**
Some implementations support MODBUS over UDP with multicast, but this is **not part of the standard MODBUS specification**.

### TCP-to-Serial Gateway: Broadcast to Serial Slaves

**Special case:** When using a MODBUS TCP-to-RTU gateway, you can send a broadcast to the serial slaves behind the gateway.

```
MODBUS/TCP Client → TCP/IP → Gateway → Serial Bus (RS-485)
                                  |
                                  +→ Slave 1 (executes)
                                  +→ Slave 2 (executes)
                                  +→ Slave 3 (executes)
```

**How to do this:**
- Connect to the gateway via TCP
- Send MODBUS request with Unit ID = 0
- Gateway forwards as broadcast on serial bus
- No response returns through the gateway

**Implementation:**
```rust
// Connect to TCP-to-RTU gateway
let mut gateway_client = ModbusTcpClient::connect("192.168.1.100:502")?;

// Build request with Unit ID 0 (broadcast to serial slaves)
let mbap_header = MBAPHeader {
    transaction_id: 1,
    protocol_id: 0,
    length: 6,
    unit_id: 0,  // Broadcast to all serial slaves
};

let pdu = write_single_register_pdu(100, 2500);
gateway_client.send_request(&mbap_header, &pdu)?;

// Gateway forwards to serial bus as broadcast
// No response expected - gateway may return timeout or error
```

### Summary Table

| Scenario | Broadcast Supported | Mechanism |
|----------|---------------------|-----------|
| MODBUS RTU on RS-485 | ✅ Yes | Address 0, all slaves execute |
| MODBUS ASCII on serial | ✅ Yes | Address 0, all slaves execute |
| MODBUS TCP direct | ❌ No | No shared medium, TCP is point-to-point |
| MODBUS TCP to multiple servers | ⚠️ Manual | Send unicast to each server individually |
| TCP-to-RTU gateway | ✅ Yes | Unit ID 0, gateway broadcasts on serial side |

## Diagnosing Broadcast Issues

### Common Broadcast Problems

#### Problem 1: No Slaves Execute Broadcast Command

**Symptoms:**
- Send broadcast write command
- Slaves don't update values
- No visible effect on network

**Possible causes:**
1. **Wrong address used**
   - Used address other than 0
   - Check: Verify address field is 0x00

2. **Slaves don't recognize broadcast**
   - Some non-compliant devices ignore address 0
   - Check: Test with known-good device
   - Solution: Update firmware or replace device

3. **CRC/LRC error in broadcast frame**
   - Slaves reject malformed frames silently
   - Check: Verify CRC calculation
   - Tool: Use protocol analyzer to inspect frame

4. **Unsupported function code**
   - Tried to broadcast a read operation
   - Check: Ensure using write functions only (0x05, 0x06, 0x0F, 0x10)

**Diagnostic steps:**
```rust
// 1. Test unicast first
for addr in 1..=247 {
    if can_communicate_with(addr) {
        println!("Slave {} responds to unicast", addr);
    }
}

// 2. Send unicast write to each slave individually
for addr in active_slaves {
    write_single_register(addr, 100, 2500)?;
    verify_value(addr, 100, 2500)?;
}

// 3. If unicast works, try broadcast
write_single_register(0, 100, 2500)?;

// 4. Verify each slave individually
for addr in active_slaves {
    let value = read_holding_register(addr, 100)?;
    if value != 2500 {
        println!("Slave {} did not execute broadcast", addr);
    }
}
```

#### Problem 2: Some Slaves Execute, Others Don't

**Symptoms:**
- Broadcast partially successful
- Some slaves update, others don't
- Inconsistent behavior

**Possible causes:**
1. **Electrical issues on bus**
   - Weak signal strength
   - Insufficient termination
   - Check: Verify RS-485 wiring, termination resistors

2. **Timing violations**
   - Some slaves too slow to process broadcast before next command
   - Check: Increase turnaround delay
   - Typical fix: Increase from 100ms to 200ms or more

3. **Slaves at different baud rates**
   - Some slaves configured incorrectly
   - Check: Verify all slaves use same baud rate, parity, stop bits

4. **Write protection or access control**
   - Some slaves reject certain writes based on operating mode
   - Check: Verify slave operating modes, security settings

**Diagnostic approach:**
```rust
// Test each slave individually to identify which ones fail
let test_value = 9999;
let results = HashMap::new();

// First, broadcast
write_single_register(0, 100, test_value)?;
std::thread::sleep(Duration::from_millis(200));  // Turnaround delay

// Then read back from each slave
for addr in active_slaves {
    let value = read_holding_register(addr, 100)?;
    results.insert(addr, value == test_value);
    
    if value != test_value {
        println!("Slave {} FAILED to execute broadcast (got {})", addr, value);
    } else {
        println!("Slave {} OK", addr);
    }
}

// Identify failing slaves for further investigation
let failing_slaves: Vec<_> = results.iter()
    .filter(|(_, &success)| !success)
    .map(|(addr, _)| addr)
    .collect();
```

#### Problem 3: Bus Collision After Broadcast

**Symptoms:**
- CRC errors immediately after broadcast
- Corrupted frames on bus
- Bus appears "stuck" or unresponsive

**Possible causes:**
1. **Non-compliant slave responding to broadcast**
   - Buggy slave sends response despite broadcast address
   - Check: Disconnect slaves one by one to identify culprit
   - Solution: Update firmware or isolate/replace device

2. **Master sending next request too soon**
   - Didn't wait full turnaround delay
   - Check: Verify turnaround delay timing
   - Solution: Increase delay to 200ms

3. **Multiple masters on bus**
   - Another master transmitting
   - Check: Verify only one master active
   - Solution: Implement proper multi-master arbitration

**Diagnostic technique:**
```rust
// Use oscilloscope or protocol analyzer to observe bus

// Send broadcast
send_broadcast_write(0, 100, 2500);

// Monitor bus for unexpected activity
let start = Instant::now();
loop {
    if let Some(frame) = capture_bus_activity() {
        let elapsed = start.elapsed();
        println!("Unexpected activity at +{:?}: {:?}", elapsed, frame);
        
        // Check if it's a response (would indicate non-compliant slave)
        if is_modbus_response(&frame) {
            println!("ERROR: Slave responded to broadcast!");
            println!("Source address: {}", frame.address);
        }
    }
    
    if start.elapsed() > Duration::from_millis(500) {
        break;  // Monitor for 500ms
    }
}
```

#### Problem 4: Broadcast Appears to Work But Values Don't Update

**Symptoms:**
- No bus errors
- Slaves appear to accept broadcast
- But values don't change

**Possible causes:**
1. **Wrong register address**
   - Broadcasting to non-existent or read-only register
   - Check: Verify register address in device documentation
   - Test: Try unicast write to verify address is writable

2. **Value out of range**
   - Slaves silently reject invalid values
   - Check: Verify value is within valid range for that register
   - Test: Try with known-good value

3. **Write protection enabled**
   - Register locked or protected
   - Check: Verify device operating mode, key switch position
   - Solution: Unlock device or change mode

4. **Scaling/format mismatch**
   - Writing raw value when device expects scaled value
   - Check: Verify scaling factors and data format

**Diagnostic approach:**
```rust
// Verify register is writable via unicast
let test_addr = 5;  // Pick one slave to test
let register = 100;
let test_value = 1234;

// Read original value
let original = read_holding_register(test_addr, register)?;
println!("Original value: {}", original);

// Write test value via unicast
write_single_register(test_addr, register, test_value)?;

// Read back
let readback = read_holding_register(test_addr, register)?;
println!("Readback value: {}", readback);

if readback != test_value {
    println!("ERROR: Register {} is not writable or value rejected", register);
    println!("Expected {}, got {}", test_value, readback);
} else {
    // Restore original
    write_single_register(test_addr, register, original)?;
    
    // Now try broadcast
    write_single_register(0, register, test_value)?;
    std::thread::sleep(Duration::from_millis(200));
    
    // Verify all slaves
    for addr in active_slaves {
        let value = read_holding_register(addr, register)?;
        println!("Slave {}: {} (expected {})", addr, value, test_value);
    }
}
```

### Using Protocol Analyzers

**Hardware protocol analyzer:**
```
Connect analyzer to RS-485 bus
Configure for MODBUS RTU
Set trigger on address 0x00

Capture sequence:
1. Normal traffic (unicast)
2. Broadcast frame
3. Period after broadcast (should be silent)
4. Next master transmission

Verify:
- Broadcast frame has address 0x00
- CRC is correct
- No responses after broadcast
- Turnaround delay is adequate
```

**Software monitoring (if supported):**
```rust
// Enable diagnostic mode on MODBUS interface
let mut modbus = ModbusRTU::new("/dev/ttyUSB0")?;
modbus.set_debug_mode(true);

// Send broadcast
println!("Sending broadcast write...");
modbus.write_single_register(0, 100, 2500)?;

// Debug output shows:
// TX: 00 06 00 64 09 C4 [CRC]
//     ^^ broadcast address
// RX: (none - timeout expected)
// Waited: 150ms (turnaround delay)
```

### Diagnostic Checklist

**Pre-broadcast checks:**
- [ ] All slaves use same baud rate, parity, stop bits
- [ ] RS-485 bus properly terminated (120Ω at each end)
- [ ] Only write function codes used (0x05, 0x06, 0x0F, 0x10, 0x16)
- [ ] Register addresses valid and writable on all slaves
- [ ] Values within valid range for all slaves

**During broadcast:**
- [ ] Address field is 0x00
- [ ] CRC is correctly calculated
- [ ] No response expected or received
- [ ] Master waits turnaround delay (100-200ms)

**Post-broadcast verification:**
- [ ] Read back values from each slave individually
- [ ] Verify all slaves updated correctly
- [ ] Check for any slaves that failed to execute
- [ ] Monitor for bus errors or collisions

### Testing Broadcast Reliability

```rust
/// Test broadcast reliability over multiple iterations
fn test_broadcast_reliability(
    slaves: &[u8],
    register: u16,
    iterations: usize,
) -> Result<BroadcastTestResults, Error> {
    let mut results = BroadcastTestResults::new();
    
    for i in 0..iterations {
        let test_value = (i as u16) % 10000;
        
        // Send broadcast
        write_single_register(0, register, test_value)?;
        std::thread::sleep(Duration::from_millis(200));
        
        // Verify each slave
        for &slave_addr in slaves {
            let value = read_holding_register(slave_addr, register)?;
            
            if value == test_value {
                results.record_success(slave_addr);
            } else {
                results.record_failure(slave_addr, test_value, value);
            }
        }
    }
    
    Ok(results)
}

// Run test
let slaves = vec![1, 2, 3, 4, 5];
let results = test_broadcast_reliability(&slaves, 100, 100)?;

println!("Broadcast reliability test (100 iterations):");
for slave in slaves {
    let success_rate = results.success_rate(slave);
    println!("  Slave {}: {:.1}% success", slave, success_rate * 100.0);
    
    if success_rate < 1.0 {
        println!("    Failures: {:?}", results.failures(slave));
    }
}
```

## Best Practices for Broadcast

### When to Use Broadcast

**Good use cases:**
- **Time synchronization:** Set clock on all devices simultaneously
- **Emergency commands:** Broadcast emergency stop or safe mode
- **Configuration updates:** Update common parameters across all devices
- **Mode changes:** Switch all devices to same operating mode
- **Initialization:** Set startup values during system commissioning

**Poor use cases:**
- **Reading values:** Cannot use broadcast for reads
- **Critical writes requiring confirmation:** No way to verify success
- **Devices with different configurations:** May cause inconsistencies
- **High-reliability operations:** No acknowledgment mechanism

### Broadcast Write Strategy

```rust
/// Safe broadcast write with verification
fn broadcast_write_with_verify(
    slaves: &[u8],
    register: u16,
    value: u16,
    max_retries: usize,
) -> Result<(), Error> {
    let mut retry_count = 0;
    
    loop {
        // Broadcast write
        write_single_register(0, register, value)?;
        std::thread::sleep(Duration::from_millis(200));
        
        // Verify all slaves
        let mut failed_slaves = Vec::new();
        for &slave_addr in slaves {
            let readback = read_holding_register(slave_addr, register)?;
            if readback != value {
                failed_slaves.push(slave_addr);
            }
        }
        
        if failed_slaves.is_empty() {
            println!("Broadcast successful to all {} slaves", slaves.len());
            return Ok(());
        }
        
        if retry_count >= max_retries {
            return Err(Error::BroadcastFailed {
                failed_slaves,
                message: format!(
                    "{} slaves failed after {} retries",
                    failed_slaves.len(),
                    max_retries
                ),
            });
        }
        
        println!("Retry {}: {} slaves failed, retrying failed slaves via unicast",
                 retry_count + 1, failed_slaves.len());
        
        // Retry failed slaves individually
        for &slave_addr in &failed_slaves {
            write_single_register(slave_addr, register, value)?;
        }
        
        retry_count += 1;
    }
}
```

### Turnaround Delay Guidelines

**Minimum safe turnaround delays:**

| Baud Rate | Character Time | Minimum Delay | Recommended Delay |
|-----------|----------------|---------------|-------------------|
| 9600 | 1.04ms | 100ms | 150-200ms |
| 19200 | 0.52ms | 75ms | 100-150ms |
| 38400 | 0.26ms | 50ms | 75-100ms |
| 115200 | 0.09ms | 25ms | 50ms |

**Factors affecting delay:**
- Number of slaves (more slaves = more processing time)
- Slave processing speed (older devices may be slower)
- Command complexity (multiple registers take longer)
- Bus loading (heavily loaded bus needs more time)

## Summary

### Key Points

1. **Broadcast addresses:**
   - **Serial (RTU/ASCII):** Address 0 for broadcast
   - **TCP direct:** 0xFF (255) or 0 for direct connection (not broadcast)
   - **TCP gateway:** Unit ID 0 broadcasts to serial slaves

2. **Effects:**
   - All slaves execute command
   - No slaves respond
   - Master waits turnaround delay (100-200ms)
   - No confirmation of success

3. **Protocol support:**
   - **MODBUS RTU/ASCII:** Full broadcast support
   - **MODBUS TCP:** No native broadcast (use multiple unicast)
   - **TCP-to-serial gateway:** Can broadcast to serial slaves

4. **Limitations:**
   - Write operations only
   - No confirmation mechanism
   - Must verify separately if needed

5. **Troubleshooting:**
   - Test unicast first
   - Verify address is 0x00
   - Check CRC/LRC
   - Verify turnaround delay
   - Use protocol analyzer
   - Read back to verify

### Broadcast Decision Table

| Question | Answer | Action |
|----------|--------|--------|
| Need to write same value to all devices? | Yes | Consider broadcast |
| Need confirmation that write succeeded? | Yes | Use individual unicast + verification |
| Using MODBUS TCP? | Yes | Send individual unicast to each server |
| Using serial (RTU/ASCII)? | Yes | Can use broadcast (address 0) |
| Need to read values? | Yes | Cannot use broadcast, use individual reads |
| Critical safety command? | Yes | Use unicast with confirmation, or broadcast + verify |
| Time synchronization? | Yes | Good use case for broadcast |

## Related Pages

- [Master-Slave Architecture](/wiki/concepts/master-slave.md) - MODBUS master-slave communication model
- [MODBUS RTU](/wiki/concepts/modbus-rtu.md) - MODBUS RTU serial protocol
- [MODBUS TCP](/wiki/concepts/modbus-tcp.md) - MODBUS TCP/IP protocol
- [MODBUS TCP Message Format](/wiki/answers/modbus-tcp-message-format.md) - MBAP header and Unit ID
- [How MODBUS TCP-to-RTU Gateways Work](/wiki/answers/modbus-tcp-to-rtu-gateway.md) - Gateway operation

## Backlinks

None yet.
