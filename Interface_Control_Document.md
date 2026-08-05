# Interface Control Document (ICD)

## 1. System Overview

This ICD covers the runtime interface for the lab-scale thermal test bench, with the following logical components:

- **RTD field sensors**: 16 PT1000 / PT100 RTDs distributed along the heated circuit.
- **Sensor Node**: local acquisition board that reads all 16 RTDs and forwards temperature data over CAN.
- **IPC (Industrial PC)**: collects CAN data and exposes it to the PLC via Modbus RTU. Also publishes higher-level telemetry by MQTT.
- **PLC (Siemens S7-1211C)**: performs control and a safety interlock, and actuates the coolant admission valve.
- **Valve**: final control element driven from the PLC output.

Physical connections:

- 24 VDC power to PLC, Sensor Node, and IPC.
- CAN bus between Sensor Node and IPC using DB9 connectors.
- Modbus RTU over RS-485 between IPC and PLC.
- Valve output from the PLC and RTD inputs to the Sensor Node.

This ICD defines the communication and timing interfaces for the sensor-to-controller/actuator path.

---

## 2. Modbus Register Map

### 2.1 Modbus Role and Transport

- **Modbus mode**: RTU
- **Byte order**: 8-bit bytes, least-significant bit first in each byte.
- **Frame format**: [Address][Function][Data][CRC16]
- **Character format**: 8 data bits, no parity, 1 stop bit (8N1) unless otherwise configured.
- **Default baud rate**: 19200 bps. Implementation must support up to 115200 bps.
- **Slave address**: `1` (modifiable by configuration).
- **Master**: Siemens S7-1211C PLC.
- **Slave**: IPC / Modbus bridge.

### 2.2 Function Codes Supported

- `0x03` Read Holding Registers
- `0x04` Read Input Registers
- `0x06` Write Single Register
- `0x10` Write Multiple Registers

### 2.3 Register Addressing

Modbus register addresses are shown in the conventional 40001-style notation. Implementation uses zero-based register indices internally.

- `40001` → register index `0`
- `40002` → register index `1`
- ...

### 2.4 Data Types and Scaling

- `INT16` — signed 16-bit integer.
- `UINT16` — unsigned 16-bit integer.
- `INT32` — signed 32-bit integer (uses two consecutive registers; high order register first).
- Scalar units:
  - Temperature: `0.01 °C` per count.
  - Valve command: unitless percentage / discrete state.

### 2.5 Byte and Word Order

- Within each Modbus register, bytes are transmitted MSB first.
- For 32-bit values spanning two registers, the **high-order register is transmitted first**, followed by the low-order register.
- Example: `INT32` value `0x12345678` is mapped to registers `[0x1234, 0x5678]`.

### 2.6 Register Map

| Address | Name | Type | Access | Scaling | Description |
|---|---|---|---|---|---|
| 40001 | SYSTEM_STATUS | UINT16 | R | Bitfield | Node and bridge status. Bit definitions below. |
| 40002 | NODE_HEALTH | UINT16 | R | Raw | Sensor node health code / error code. |
| 40003 | RTD_01 | INT16 | R | 0.01 °C | RTD channel 1 temperature. |
| 40004 | RTD_02 | INT16 | R | 0.01 °C | RTD channel 2 temperature. |
| 40005 | RTD_03 | INT16 | R | 0.01 °C | RTD channel 3 temperature. |
| 40006 | RTD_04 | INT16 | R | 0.01 °C | RTD channel 4 temperature. |
| 40007 | RTD_05 | INT16 | R | 0.01 °C | RTD channel 5 temperature. |
| 40008 | RTD_06 | INT16 | R | 0.01 °C | RTD channel 6 temperature. |
| 40009 | RTD_07 | INT16 | R | 0.01 °C | RTD channel 7 temperature. |
| 40010 | RTD_08 | INT16 | R | 0.01 °C | RTD channel 8 temperature. |
| 40011 | RTD_09 | INT16 | R | 0.01 °C | RTD channel 9 temperature. |
| 40012 | RTD_10 | INT16 | R | 0.01 °C | RTD channel 10 temperature. |
| 40013 | RTD_11 | INT16 | R | 0.01 °C | RTD channel 11 temperature. |
| 40014 | RTD_12 | INT16 | R | 0.01 °C | RTD channel 12 temperature. |
| 40015 | RTD_13 | INT16 | R | 0.01 °C | RTD channel 13 temperature. |
| 40016 | RTD_14 | INT16 | R | 0.01 °C | RTD channel 14 temperature. |
| 40017 | RTD_15 | INT16 | R | 0.01 °C | RTD channel 15 temperature. |
| 40018 | RTD_16 | INT16 | R | 0.01 °C | RTD channel 16 temperature. |
| 40019 | RTD_FAULT_BITMAP | UINT16 | R | Bitfield | RTD channel fault indicators: bit 0 = RTD1 fault, ... bit 15 = RTD16 fault. |
| 40020 | INTERLOCK_SOURCE | UINT16 | R/W | Raw | Interlock source selection; 0=automatic, 1=RTD max, 2=RTD average, reserved values. |
| 40021 | VALVE_COMMAND | UINT16 | R/W | 0=close, 1=open, 2=auto | Valve request command for supervisory override. |
| 40022 | VALVE_STATUS | UINT16 | R | 0/1 | Actual valve feedback state: 0=closed, 1=open. |
| 40023 | PROCESS_SETPOINT | INT16 | R/W | 0.01 °C | Desired process temperature setpoint. |
| 40024 | ALARM_THRESHOLD | INT16 | R/W | 0.01 °C | High temperature alarm threshold. |
| 40025 | DIAGNOSTICS_CODE | UINT16 | R | Raw | Bridge diagnostics or fault code. |
| 40026 | TIMESTAMP_LOW | UINT16 | R | ms | Low 16 bits of sample timestamp, optional. |
| 40027 | TIMESTAMP_HIGH | UINT16 | R | ms | High 16 bits of sample timestamp, optional. |

### 2.7 SYSTEM_STATUS Bitfield

| Bit | Name | Description |
|---|---|---|
| 0 | CAN_OK | 1 = CAN uplink active, 0 = CAN error detected |
| 1 | MODBUS_OK | 1 = Modbus connection healthy, 0 = Modbus error |
| 2 | RTD_FAULT | 1 = any RTD fault present |
| 3 | VALVE_FAULT | 1 = valve feedback mismatch or actuator fault |
| 4 | NODE_HEALTH_WARN | 1 = warning state |
| 5 | IPC_HEALTH_WARN | 1 = IPC bridge warning |
| 6 | RESERVED | Reserved for future use |
| 7 | RESERVED | Reserved for future use |
| 8..15 | RESERVED | Reserved for future expansion |

### 2.8 Access Rights Summary

- Read-only: RTD_01..RTD_16, RTD_FAULT_BITMAP, VALVE_STATUS, DIAGNOSTICS_CODE, TIMESTAMP_LOW/HIGH.
- Read/write: INTERLOCK_SOURCE, VALVE_COMMAND, PROCESS_SETPOINT, ALARM_THRESHOLD.
- Write access should only be granted to authorized PLC logic and must be validated by the IPC bridge.

### 2.9 Error Handling

- Modbus exception codes must be returned per Modbus RTU specification.
- If a temperature value is invalid or out of range, the IPC shall set the corresponding RTD register to `0x8000` and set the RTD fault bit.
- CRC checks are mandatory on every RTU frame.

---

## 3. CAN Frame Catalogue

### 3.1 CAN Bus Overview

- **CAN type**: Classic CAN
- **Identifier**: 11-bit standard IDs
- **Nominal bitrate**: `500 kbps`
- **Bus topology**: single-wire differential pair over DB9.
- **Physical node**: Sensor Node is producer of temperature frames; IPC is consumer and Modbus bridge.

### 3.2 Frame Priority Rationale

CAN arbitration uses lower IDs as higher priority. The mapping ensures:

- safety and status frames win arbitration over periodic temperature data.
- temperature data is grouped into equal-sized frames for deterministic bandwidth.

### 3.3 Frame Definitions

| CAN ID | Name | Direction | Data Length | Frequency | Description |
|---|---|---|---|---|---|
| `0x100` | RTD_GROUP_1 | Node → IPC | 8 | 10 Hz | RTD1..RTD4 temperatures. |
| `0x101` | RTD_GROUP_2 | Node → IPC | 8 | 10 Hz | RTD5..RTD8 temperatures. |
| `0x102` | RTD_GROUP_3 | Node → IPC | 8 | 10 Hz | RTD9..RTD12 temperatures. |
| `0x103` | RTD_GROUP_4 | Node → IPC | 8 | 10 Hz | RTD13..RTD16 temperatures. |
| `0x110` | NODE_DIAGNOSTICS | Node → IPC | 8 | 1 Hz | Node health, scan status, and fault bitmap. |
| `0x120` | BRIDGE_STATUS | IPC → PLC/Node | 8 | 1 Hz | IPC-to-PLC bridge heartbeat and system health. |
| `0x130` | VALVE_COMMAND | IPC/PLC → Node | 8 | Event-driven up to 10 Hz | Supervisory valve command or override. |

### 3.4 RTD Multiplexing Layout

Each RTD group frame carries four channels; this is the multiplexing strategy used so 16 channels fit within the 8-byte CAN payload.

#### `0x100` `RTD_GROUP_1`

| Byte | Contents |
|---|---|
| 0..1 | RTD1 temperature (INT16, 0.01 °C) |
| 2..3 | RTD2 temperature (INT16, 0.01 °C) |
| 4..5 | RTD3 temperature (INT16, 0.01 °C) |
| 6..7 | RTD4 temperature (INT16, 0.01 °C) |

#### `0x101` `RTD_GROUP_2`

| Byte | Contents |
|---|---|
| 0..1 | RTD5 temperature |
| 2..3 | RTD6 temperature |
| 4..5 | RTD7 temperature |
| 6..7 | RTD8 temperature |

#### `0x102` `RTD_GROUP_3`

| Byte | Contents |
|---|---|
| 0..1 | RTD9 temperature |
| 2..3 | RTD10 temperature |
| 4..5 | RTD11 temperature |
| 6..7 | RTD12 temperature |

#### `0x103` `RTD_GROUP_4`

| Byte | Contents |
|---|---|
| 0..1 | RTD13 temperature |
| 2..3 | RTD14 temperature |
| 4..5 | RTD15 temperature |
| 6..7 | RTD16 temperature |

### 3.5 Diagnostic Frame Layout

#### `0x110` `NODE_DIAGNOSTICS`

| Byte | Field | Description |
|---|---|---|
| 0 | STATUS_FLAGS | Bitfield; bit 0=CAN_OK, bit1=RTD_FAULT, bit2=MODBUS_OK, bit3=VALVE_FAULT. |
| 1 | ERROR_CODE | Node-specific health code. |
| 2 | SAMPLE_COUNTER | Rolling frame counter (0..255). |
| 3 | RESERVED | Reserved, set to 0. |
| 4..5 | FAULT_BITMAP | 16-bit RTD fault bitmap (big-endian). |
| 6..7 | RESERVED | Reserved, set to 0. |

### 3.6 Valve Command Frame Layout

#### `0x130` `VALVE_COMMAND`

| Byte | Field | Description |
|---|---|---|
| 0 | COMMAND_TYPE | `0x00` = no action / auto, `0x01` = open, `0x02` = close, `0x03` = override. |
| 1 | COMMAND_VALUE | 0..100 for percentage or duty; reserved if discrete. |
| 2 | FLAGS | bit0=valid, bit1=override active, others reserved. |
| 3 | RESERVED | Reserved, set to 0. |
| 4..5 | INTERLOCK_SETPOINT | INT16 0.01°C if command is temperature-targeted. |
| 6..7 | CHECKSUM | Simple ID+data checksum for application-layer validation. |

### 3.7 CAN Timing and Bus Loading

#### Frame timing

- `RTD_GROUP_*` frames at `10 Hz` each.
- `NODE_DIAGNOSTICS` at `1 Hz`.
- `BRIDGE_STATUS` at `1 Hz`.
- `VALVE_COMMAND` event-driven; worst-case `10 Hz`.

#### Frame bit count

For Classic CAN standard frame with 8 bytes of data:

- Payload bits = `8 bytes * 8 = 64 bits`
- Overhead estimate = `47 bits`
- Total per frame ≈ `111 bits`

#### Worst-case bus loading (500 kbps)

- 4 RTD frames at 10 Hz: `4 * 111 * 10 = 4,440 bits/s`
- `NODE_DIAGNOSTICS` at 1 Hz: `111 bits/s`
- `BRIDGE_STATUS` at 1 Hz: `111 bits/s`
- `VALVE_COMMAND` at 10 Hz worst-case: `1,110 bits/s`

**Total worst-case** ≈ `5,772 bits/s` or `1.16%` of a `500 kbps` bus.

#### Worst-case bus loading (250 kbps)

**Total worst-case** ≈ `2.31%` of the bus.

For additional safety, the design budget should reserve at least `10%` of bus capacity for future frames, retransmissions, and error recovery.

### 3.8 Multiplexing Justification

Classic CAN affords only 8 bytes per frame, while the system must transport 16 temperature channels. The chosen multiplexing groups four RTDs per frame and transmits all four group frames at 10 Hz. This delivers all 16 temperatures every 100 ms while keeping bus loading extremely low.

---

## 4. Timing Requirements

### 4.1 Sensor Node Acquisition

- Sample interval: `100 ms` nominal.
- Conversion latency: `20 ms` maximum for a full RTD scan and temperature calculation.
- CAN transmit delay: `< 10 ms` from measurement ready to first eligible CAN frame.

### 4.2 PLC Polling and Scan

- PLC poll interval for Modbus: `100 ms` nominal.
- PLC scan interval / logic update: `50 ms` nominal, with worst-case `100 ms`.
- Combined worst-case PLC update interval: `200 ms` if polling and scan align unfavorably.

### 4.3 Modbus Response Latency

At `19200 bps`, a single Modbus RTU request/response pair of roughly 8 bytes each takes approximately `27 ms` each direction, plus up to `10 ms` processing inside the bridge.

- Request latency: `27 ms`
- Response latency: `27 ms`
- Total Modbus round-trip: `~60 ms`

### 4.4 End-to-End Latency Budget

The chain is:

1. RTD analog acquisition and digital conversion.
2. Sensor node CAN transmission.
3. IPC CAN reception and Modbus register update.
4. PLC Modbus request, scan, and logic execution.
5. Valve output update and coil response.

#### Worst-case budget estimate

| Stage | Budget |
|---|---|
| RTD acquisition and conversion | 20 ms |
| CAN transmission + arbitration | 50 ms |
| IPC processing / Modbus register write | 10 ms |
| PLC Modbus poll wait | 100 ms |
| PLC logic scan and output update | 20 ms |
| Valve coil actuation delay | 50 ms |
| **Total worst-case** | **250 ms** |

### 4.5 Value Age for Safety Interlock

- The PLC reads the latest RTD values from Modbus once per poll cycle.
- The worst-case age of the RTD value used by the interlock is:
  - RTD sample age at the moment of CAN transmission: up to `100 ms`
  - CAN delivery and IPC update: up to `50 ms`
  - Modbus poll interval: `100 ms`

**Worst-case interlock age**: `~250 ms`

### 4.6 Safety Margin and Refresh Rate

- The RTD data refresh rate should remain at least `10 Hz`.
- The interlock decision interval must be no slower than `200 ms`.
- The recommended implementation is to maintain the last good temperature sample in the IPC and continue supplying it to the PLC until a new sample completes successfully.

---

## 5. Use Cases and Data Flow

### 5.1 Normal Operation

1. The Sensor Node reads 16 RTDs and computes temperatures.
2. Every `100 ms`, the Sensor Node sends four CAN frames (`0x100`..`0x103`) containing the 16 temperature values.
3. The IPC receives the CAN frames, updates its internal temperature table, and presents values in Modbus registers `40003`..`40018`.
4. The PLC polls those registers at `100 ms` intervals, computes control and interlock decisions, and updates the valve output.

### 5.2 Fault Handling

- If any RTD is invalid, the Sensor Node sets the corresponding `RTD_FAULT_BITMAP` bit and sends `0x110` status with fault details.
- The IPC sets `SYSTEM_STATUS` accordingly and reports the fault to the PLC via `40019`.
- The PLC may enter a safe state if a fault bit is asserted.

### 5.3 Valve Override

- Supervisory or maintenance command may be issued using `VALVE_COMMAND` via Modbus or by CAN command `0x130`.
- The PLC must validate override requests and compare them against `INTERLOCK_SOURCE`, `PROCESS_SETPOINT`, and `ALARM_THRESHOLD`.

---

## 6. Assumptions and Notes

This ICD is derived from the provided system architecture, available schematic annotations, and typical industrial network patterns. Specific hardware parameters that are not present in the repository are handled using standard industrial defaults:

- Modbus RTU at 19200 bps unless otherwise constrained.
- Classic CAN with 11-bit identifiers and 500 kbps nominal speed.
- RTD scaling to `0.01 °C` for precision.

If the final implementation uses alternate baud rates, a longer Modbus register range, or extended CAN identifiers, those details must be updated in this ICD.

---

## 7. Revision History

- `1.0` — Initial ICD for RTD-to-valve interface, including Modbus register map, CAN frame catalogue, and timing budgets.
