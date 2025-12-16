# Day 2: Motor Zeroing, Custom Scripting & Power Diagnostics

> **Date**: 2025-12-14
> **Status**: In Progress (Blocked by PSU)
> **Focus**: Firmware Zeroing, Python Scripting, Voltage Drop Analysis

## Summary

Focused on preparing the 12x **Feetech STS3215** motors for physical assembly. Discovered that the latest LeRobot software stack removed the "Auto-Zero" feature. Developed custom Python tools to force motors to the neutral position (`2048`). Identified a critical hardware bottleneck: the 5V power supply causes communication blackouts ("No Response") due to voltage drop in the daisy chain.

## Technical Challenges & Solutions

### 1. Software: The "Calibration Loop" Crash
**Problem**: The standard `lerobot-calibrate` script failed with `ValueError: Magnitude 2048 exceeds 2047`.
* **Analysis**: The software attempted to calculate a homing offset for a motor that was physically unaligned (off by ~180 degrees), exceeding the 11-bit variable limit in the motor's EEPROM.

**Solution**: Developed a custom "Bare Metal" script (`simple_zero_final.py`).
* Bypassed the high-level `Robot` class abstraction.
* Directly accessed `FeetechMotorsBus` to broadcast `Goal_Position = 2048`.
* **Implementation**: Switched from batch writing to an iterative `for` loop to ensure reliable command delivery to each ID.

### 2. Hardware: The "Zombie Motor" Phenomenon
**Problem**: Motors showed LED activity (Power ON) but returned `No response` during read operations.
* **Observation**: Single motor connection (ID 3) worked intermittently, but the full daisy chain failed completely.

**Solution**: Diagnosis confirmed **Undervoltage**.
* **Root Cause**: The **5V 4A PSU** is insufficient for **7.4V** specced motors. Voltage drop across the daisy chain pushed the MCU below its logic threshold (Brown-out), while LEDs remained lit due to lower voltage requirements.
* **Action Plan**: Discarded 5V PSU. Scheduled Makerspace visit to inject **7.5V - 8.4V** via Variable DC Power Supply.

### 3. Mechanical: Stuck Connectors
**Problem**: Dual 3-pin daisy-chain connectors were seized in the motor housing, preventing reconfiguration.

**Solution**: Mechanical Extraction.
* Successfully removed the stuck housing using pliers (sacrificed a tweezer in the process) without damaging the internal PCB pins.

## Workarounds & Scripts

**Custom Zeroing Logic**:
Since official scripts now assume a pre-assembled robot, I implemented a standalone tool for pre-assembly zeroing:

```python
# simple_zero_final.py snippet
target_ids = list(range(1, 7))
for motor_id in target_ids:
    # Force Enable Torque & Move to Center
    bus.write("Torque_Enable", motor_id, 1)
    bus.write("Goal_Position", motor_id, 2048)
```

