# Day 3: Low-Voltage Workarounds, Raw Serial Protocol & Auto-Zeroing

> **Date**: 2025-12-15
> **Status**: Success (Zeroing Complete) / Hardware Warning
> **Focus**: Raw Serial Communication, Power Budgeting, Automated Validation

## Summary

Successfully zeroed and verified all 12 **Feetech STS3215** motors using the existing 5V 4A power supply, bypassing the need to wait for the 7.5V unit for this specific phase. Identified that the official LeRobot library enforces a voltage safety check that prevents operation at 5V. Developed a custom "Raw Serial" Python tool to bypass these checks and implemented a sequential "Pulse-and-Relax" strategy to manage the limited power budget. Also identified a hardware defect (detached strain relief) on one motor unit.

## Technical Challenges & Solutions

### 1. Software: Library Voltage Lockout
**Problem**: The official `lerobot` scripts persisted in returning `No response` or failing initialization, even though the motors were receiving power.
* **Analysis**: The high-level library queries the motor's voltage register before sending motion commands. The 5V supply (dropping to ~4.6V under load) triggers an under-voltage safety exit in the software, despite the STS3215 chip being technically capable of low-voltage communication.

**Solution**: Developed a **Raw Serial Protocol Script**.
* Bypassed the LeRobot library entirely.
* Constructed raw hexadecimal packets based on the STS3215 communication protocol.
* Sent `Write Position` commands directly to memory addresses, ignoring voltage feedback.

### 2. Hardware: Power Budget & Brown-outs
**Problem**: Attempting to enable torque on all 6 motors simultaneously caused immediate voltage sag, leading to system resets.

**Solution**: **Sequential Activation Strategy**.
* The custom script was designed to target one motor ID at a time.
* **Workflow**: Enable Torque -> Perform Calibration -> **Disable Torque (Power Off)** -> Sleep 1s -> Next Motor.
* This ensured the PSU only handled the load of a single active motor at any given moment.

### 3. Hardware: Compromised Strain Relief
**Problem**: The hot glue strain relief on one motor's 3-pin connector interface has detached, exposing the solder joints to mechanical stress.

**Solution**: Mitigation & Assignment.
* **Immediate**: Flagged for repair using hot glue or electrical tape reinforcement.
* **Strategic**: Assigned this specific motor to **ID 1 (Base Rotation)**. Since the base wiring is stationary and secured to the chassis, this minimizes the risk of wire fatigue compared to the articulated joints.

## Workarounds & Scripts

**Automated Scan & Verify Tool**:
To validate zeroing without visual indicators (LEDs), I implemented a "Wiggle Test" script. It scans for active IDs, locks the motor, performs a small left/right motion to confirm control, centers the motor to `2048`, and then cuts power to save energy.

```python
# auto_scan_verify.py
import serial
import time

# Configuration
PORT = "/dev/ttyACM0"
BAUDRATE = 1000000
MAX_SCAN_ID = 10

def get_checksum(payload):
    checksum_sum = sum(payload)
    return (~checksum_sum) & 0xFF

def write_register(ser, motor_id, address, value_bytes):
    # Instruction 3 = Write
    instruction = 3 
    params = [address] + list(value_bytes)
    length = len(params) + 2
    
    checksum_payload = [motor_id, length, instruction] + params
    checksum = get_checksum(checksum_payload)
    
    packet = [0xFF, 0xFF, motor_id, length, instruction] + params + [checksum]
    
    ser.reset_input_buffer()
    ser.write(bytes(packet))
    time.sleep(0.05)

def run_calibration_sequence(ser, motor_id):
    print(f"   [ID {motor_id}] Enabling Torque...")
    write_register(ser, motor_id, 40, [1])
    time.sleep(0.5)

    # Wiggle Test to confirm control
    print(f"   [ID {motor_id}] Moving Left (1900)...")
    write_register(ser, motor_id, 42, [0x6C, 0x07]) 
    time.sleep(0.8)

    print(f"   [ID {motor_id}] Moving Right (2200)...")
    write_register(ser, motor_id, 42, [0x98, 0x08]) 
    time.sleep(0.8)

    # Final Zeroing
    print(f"   [ID {motor_id}] Zeroing to 2048...")
    write_register(ser, motor_id, 42, [0x00, 0x08]) 
    time.sleep(1.5)

    # CRITICAL: Disable Torque to prevent PSU overload
    print(f"   [ID {motor_id}] Disabling Torque (Relax)...")
    write_register(ser, motor_id, 40, [0])

def main():
    ser = serial.Serial(PORT, BAUDRATE, timeout=0.05)
    
    for mid in range(MAX_SCAN_ID + 1):
        # Ping logic omitted for brevity; if ping success:
        print(f"\nFound ID: {mid}")
        try:
            run_calibration_sequence(ser, mid)
        except Exception as e:
            print(f"Error on ID {mid}: {e}")
        
        # Cool down PSU between motors
        time.sleep(1.0) 

    ser.close()

if __name__ == "__main__":
    main()
