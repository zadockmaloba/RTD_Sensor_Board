# S7-1211C Ladder Logic

## 1. Purpose

This document describes the derived ladder logic for the Siemens S7-1211C CPU with CB1241 communication module used in the thermal test bench. The logic reads RTD temperature data from the IPC via Modbus, evaluates interlock conditions, and controls the coolant valve.

## 2. Assumptions

- The PLC uses the CB1241 for RS-485 Modbus communication with the IPC.
- The IPC exposes RTD temperatures and status values in Modbus registers.
- The PLC executes a read cycle every `100 ms`.
- The valve coil is controlled by output `Q0.0`.
- Valve feedback is available in Modbus register `VALVE_STATUS`.

## 3. Data Block Mapping

Use `DB1` to map Modbus register values into PLC memory.

| DB1 Offset | Description |
|---|---|
| `DB1.W0` | SYSTEM_STATUS |
| `DB1.W1` | NODE_HEALTH |
| `DB1.W2` | RTD_01 |
| `DB1.W3` | RTD_02 |
| `DB1.W4` | RTD_03 |
| `DB1.W5` | RTD_04 |
| `DB1.W6` | RTD_05 |
| `DB1.W7` | RTD_06 |
| `DB1.W8` | RTD_07 |
| `DB1.W9` | RTD_08 |
| `DB1.W10` | RTD_09 |
| `DB1.W11` | RTD_10 |
| `DB1.W12` | RTD_11 |
| `DB1.W13` | RTD_12 |
| `DB1.W14` | RTD_13 |
| `DB1.W15` | RTD_14 |
| `DB1.W16` | RTD_15 |
| `DB1.W17` | RTD_16 |
| `DB1.W18` | RTD_FAULT_BITMAP |
| `DB1.W19` | INTERLOCK_SOURCE |
| `DB1.W20` | VALVE_COMMAND |
| `DB1.W21` | VALVE_STATUS |
| `DB1.W22` | PROCESS_SETPOINT |
| `DB1.W23` | ALARM_THRESHOLD |
| `DB1.W24` | DIAGNOSTICS_CODE |

## 4. Internal Memory Bits

| Bit | Name | Description |
|---|---|---|
| `M0.0` | MODBUS_OK | Modbus communication healthy |
| `M0.1` | RTD_FAULT_PRESENT | Any RTD fault or invalid data present |
| `M0.2` | TEMP_BELOW_SP | Selected temperature below setpoint |
| `M0.3` | TEMP_ABOVE_ALARM | Selected temperature above alarm threshold |
| `M0.4` | VALVE_OPEN_REQUEST | Automatic open request |
| `M0.5` | VALVE_CLOSE_REQUEST | Automatic close request / safe close |
| `M0.6` | MANUAL_OVERRIDE_ACTIVE | Manual command override active |

## 5. Ladder Logic Rungs

### Rung 1: Modbus Health Decode

- Set `M0.0` when `SYSTEM_STATUS.bit1` (`MODBUS_OK`) is true.
- Set `M0.1` when `SYSTEM_STATUS.bit2` (`RTD_FAULT`) is true or `RTD_FAULT_BITMAP <> 0`.

```
  [ DB1.W0.1 ] --------------------( ) M0.0
  [ DB1.W0.2 ] --------------------( ) M0.1
  [ DB1.W18 <> 0 ] ---------------( ) M0.1
```

### Rung 2: Temperature Source Selection

Select the active temperature used for control based on `INTERLOCK_SOURCE`:

- `0` or `1`: maximum of RTD1..RTD16
- `2`: average of RTD1..RTD16

Store the result in `MW100` (`SELECTED_TEMP`).

> Note: This can be implemented with a function block, a structured organization block, or a set of compare/move instructions.

### Rung 3: Temperature Comparison

Compare `SELECTED_TEMP` to the setpoint and alarm threshold.

```
  [ MW100 < DB1.W22 ]  ----------( ) M0.2
  [ MW100 >= DB1.W23 ] -----------( ) M0.3
```

### Rung 4: Safe-close Condition

Request safe close when any fault or alarm exists, or when Modbus is down.

```
  [ NOT M0.0 ] --------------------( ) M0.5
  [ M0.1 ] ------------------------( ) M0.5
  [ M0.3 ] ------------------------( ) M0.5
```

### Rung 5: Auto Valve Decision

In auto mode, open if below setpoint and no fault; otherwise close.

```
  [ M0.0 ] [ NOT M0.1 ] [ M0.2 ] --( ) M0.4
  [ M0.5 ] -----------------------( ) /M0.4
```

### Rung 6: Manual Override

If manual override is active, use `VALVE_COMMAND` from the IPC instead of the auto request.

```
  [ M0.6 ] [ DB1.W20 = 1 ] ------( ) Q0.0
  [ M0.6 ] [ DB1.W20 = 2 ] ------( ) /Q0.0
  [ NOT M0.6 ] [ M0.4 ] ---------( ) Q0.0
  [ NOT M0.6 ] [ /M0.4 ] --------( ) /Q0.0
```

### Rung 7: Valve Feedback Validation

Raise a fault if the commanded valve position does not match feedback.

```
  [ Q0.0 ] [ DB1.W21 = 0 ] -----( ) M0.1
  [ /Q0.0 ] [ DB1.W21 = 1 ] ----( ) M0.1
```

## 6. Control Flow Summary

1. Read Modbus registers from IPC into `DB1`.
2. Decode communication and sensor faults.
3. Select the temperature value used for the safety interlock.
4. Compare against setpoint and alarm threshold.
5. Determine auto open/close commands.
6. Override with manual commands if active.
7. Drive `Q0.0` for the valve coil.
8. Validate feedback and set fault state if mismatch detected.
