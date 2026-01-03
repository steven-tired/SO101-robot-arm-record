# Day 4: Assembly, Calibration Anomalies & Motion Verification

> **Date**: 2025-12-16
> **Status**: Assembly Complete / Calibration Verified
> **Focus**: Mechanical Build, Homing Offsets, Safety Testing

## Summary

Completed the physical assembly of the SO-100 robotic arm. The process revealed critical discrepancies between the software ID mapping and the physical joint layout, necessitating a hardware audit. The initial calibration attempt failed to produce the expected "L-shape" neutral pose, resulting in a folded, collision-prone state. This was resolved by understanding and re-applying the homing offsets. Finally, a custom `move_test.py` script was deployed to verify safe actuation using normalized coordinates before attempting full kinematic control.

## Technical Challenges & Solutions

### 1. Assembly: Bus Communication Failure
**Problem**: Upon initial power-up after assembly, the motor bus was completely unresponsive. The discovery scripts returned zero connected devices, and the LED status indicators on the motors remained off, despite verified power connections.
* **Analysis**: The issue was identified as a combination of Linux user permissions accessing the `/dev/ttyACM0` serial port and a USB controller driver latch-up. The operating system had locked the port, preventing the Python interface from establishing a handshake.

**Solution**: **System Reset & Port Binding**.
* **Hard Reboot**: Performed a full system reboot to clear the USB controller state.
* **Privilege Escalation**: Executed all Python scripts using `sudo` to bypass user group (`dialout`) permission restrictions.
* **USB Binding**: Manually forced a USB unbind/bind sequence in the terminal to trigger a fresh driver initialization for the serial converter.

### 2. Calibration: The "L-Shape" Failure
**Problem**: After running the initial calibration, commanding the robot to its "Zero" position caused it to curl into a tight, unsafe ball rather than the expected upright "L-shape" (Standard Neutral Pose).
**Analysis**: I initially misunderstood "Zeroing." The motors' internal magnetic encoder zero (0-4096) is arbitrary relative to the arm's geometry. The calibration file requires a specific `homing_offset` that mathematically shifts this arbitrary zero to align with the kinematic model's zero (vertical upright).

**Solution**: **Manual Homing Alignment**.
* Manually moved the arm into the perfect "L-shape" (Upper arm vertical, Lower arm horizontal) while torque was disabled.
* Ran the calibration wizard in this specific pose to capture the correct offsets.
* Verified that `Goal_Position=0` now results in the arm standing straight up.

### 3. Verification: Safe Motion Testing
**aim**: test for functions of robor arm. 

**Solution**: **`move_test.py` Script**.
* **Mechanism**: Uses `MotorNormMode.RANGE_M100_100`. Instead of angles or millimeters, it accepts values from `-100` to `100`.
* **Workflow**: The script connects to the bus using the saved calibration JSON and executes a hardcoded sequence of small, isolated movements (e.g., `shoulder_pan` to 45, `elbow_flex` to -50).
* **Result**: Confirmed that positive values move joints in the correct direction (Right-Hand Rule) and that the range of motion is physically safe.

## Key Scripts

**Motion Verification Logic (`move_test.py`)**:
This snippet demonstrates manually loading the calibration profile to allow safe, normalized motor control without the overhead of the full LeRobot control stack.

```python
import json
import time
from lerobot.motors.feetech import FeetechMotorsBus
from lerobot.motors.motors_bus import Motor, MotorNormMode, MotorCalibration

calib_path = "/home/steven/.cache/huggingface/lerobot/calibration/robots/so101_follower/my_follower_arm.json"

with open(calib_path, "r") as f:
    raw_calib_data = json.load(f)

calibration_objects = {}
for motor_name, data in raw_calib_data.items():
    
    calibration_objects[motor_name] = MotorCalibration(**data)
    
norm_mode = MotorNormMode.RANGE_M100_100

motors_config = {
    "shoulder_pan": Motor(1, "sts3215", norm_mode),
    "shoulder_lift": Motor(2, "sts3215", norm_mode),
    "elbow_flex": Motor(3, "sts3215", norm_mode),
    "wrist_flex": Motor(4, "sts3215", norm_mode),
    "wrist_roll": Motor(5, "sts3215", norm_mode),
    "gripper": Motor(6, "sts3215", MotorNormMode.RANGE_0_100),
}

bus = FeetechMotorsBus(
    port="/dev/ttyACM0", 
    motors=motors_config,
    calibration=calibration_objects 
)

try:
    bus.connect()
    bus.write("Goal_Position", "gripper", 0)
    bus.write("Goal_Position", "wrist_roll", 50)
    bus.write("Goal_Position", "shoulder_lift", -10)
    bus.write("Goal_Position", "elbow_flex",-20)
    time.sleep(1)
    bus.write("Goal_Position", "wrist_flex", 73)
    bus.write("Goal_Position", "wrist_roll", 0)
    bus.write("Goal_Position", "shoulder_pan", 0)
    bus.write("Goal_Position", "shoulder_lift", -90)
    bus.write("Goal_Position", "elbow_flex",50)
    time.sleep(1.5)

except Exception as e:
   
    if 'calibration_objects' in locals():
        print(f"sample: {list(calibration_objects.values())[0]}")

finally:
    bus.disconnect()
    print("disconnect")
```


https://github.com/user-attachments/assets/6a5af689-eff2-4781-aeb4-9aa49643345a


