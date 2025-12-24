# DevLog Day 5: Refining the Kinematics Mode & Script Optimization

## 1. Overview
Today was focused on bridging the gap between the **Inverse Kinematics (IK)** mathematical model and the **physical hardware**. The primary goal was to ensure that when the script calculates a coordinate (e.g., "straight ahead"), the arm actually moves to that position without hitting the desk or over-rotating.

## 2. Key Challenges & Solutions

### The "Offset" Problem (Mathematical vs. Physical Zero)
* **Issue**: The IK solver (using `ikpy`) identified "straight ahead" as **-33°**, while the physical motor calibration expected **0°** at that position.
* **Solution**: Implemented a software-level `OFFSET` dictionary. By applying a `+33` correction to the `shoulder_pan`, the software now aligns perfectly with the physical world.
* **Direction Logic**: Added a `DIRECTION_CORRECTION` toggle to handle cases where the motor’s internal direction is inverted relative to the URDF model.

### The "Table Crash" Safety Fix
* **Issue**: The initial `stow_arm()` (Home) function was set to a position that caused the arm to strike the table (`shoulder_lift: -90`).
* **Solution**: Re-mapped the stow logic to a safe "vertical/neutral" pose.
    * **Action**: Updated `shoulder_lift` to `0` and `elbow_flex` to `60` to ensure a safe, upright resting position.

###  Degree-to-Normalization Mapping
* **Issue**: The `lerobot` bus requires values in a normalized range (**-100 to 100**), but the IK output is in Degrees/Radians.
* **Solution**: Refined the `deg_to_norm` function. It now calculates target steps based on the 4096-step resolution of the STS3215 motors and maps them accurately against the `range_min/max` defined in the calibration JSON.

## 3. Implementation (`control_move.py`)

The core logic now handles the transformation pipeline: 
**XYZ Input** → **IK Solver** → **Degree Offset Correction** → **Normalization (-100/100)** → **Bus Write**.

```python
import numpy as np
import time
import json
import sys
import traceback
from ikpy.chain import Chain
from lerobot.robots.so100_follower import SO100Follower, SO100FollowerConfig
from lerobot.motors.motors_bus import MotorCalibration, MotorNormMode, Motor

# ========================================================
# CONFIGURATION
# ========================================================

# Manual angle offsets for hardware alignment
JOINT_OFFSETS = {
    "shoulder_pan":  0,
    "shoulder_lift": 0,
    "elbow_flex":    0,
    "wrist_flex":    0,
    "wrist_roll":    0,
    "gripper":       0
}

# Toggle for inverted motor directions
INVERT_DIRECTION = {
    "shoulder_pan":  False, 
    "shoulder_lift": False,
    "elbow_flex":    False,
    "wrist_flex":    False,
    "wrist_roll":    False,
    "gripper":       False
}

CALIB_DATA_PATH = "/home/steven/.cache/huggingface/lerobot/calibration/robots/so101_follower/my_follower_arm.json"
ROBOT_URDF_FILE = "URDF_so101.urdf" 

# ========================================================
# CORE FUNCTIONS
# ========================================================

def deg_to_norm(degree, calib, motor_name):
    """Map joint degrees to LeRobot normalization range (-100 to 100)"""
    degree += JOINT_OFFSETS.get(motor_name, 0)
    if INVERT_DIRECTION.get(motor_name, False):
        degree = -degree

    steps_per_deg = 4096.0 / 360.0
    target_steps = degree * steps_per_deg
    center = calib.homing_offset
    
    if target_steps > 0:
        span = max(calib.range_max - center, 1000)
    else:
        span = max(center - calib.range_min, 1000)
        
    return np.clip((target_steps / span) * 100.0, -100.0, 100.0)

def init_hardware():
    """Initialize motor bus and load calibration"""
    print("Connecting to hardware...")
    cfg = SO100FollowerConfig(port="/dev/ttyACM0")
    robot = SO100Follower(cfg)
    
    try:
        with open(CALIB_DATA_PATH, "r") as f:
            calib_dict = {k: MotorCalibration(**v) for k, v in json.load(f).items()}
        
        robot.bus.calibration = calib_dict
        for name, calib in calib_dict.items():
            robot.bus.motors[name] = Motor(calib.id, "sts3215", MotorNormMode.RANGE_M100_100)
            
        robot.bus.connect() 
        robot.bus.enable_torque()
        print("Hardware ready")
    except Exception as e:
        print(f"Init failed: {e}")
        sys.exit(1)
    return robot

def init_kinematics():
    """Load URDF and configure IK chain"""
    print("Loading IK model...")
    try:
        chain = Chain.from_urdf_file(ROBOT_URDF_FILE, base_elements=["base"])
        # Mask active links (first 5 joints)
        mask = [False] + [True if i <= 5 else False for i in range(1, len(chain.links))]
        chain.active_links_mask = mask
        return chain
    except Exception as e:
        print(f"IK load failed: {e}")
        sys.exit(1)

def execute_move(robot, angle_dict):
    """Apply offsets and write goal positions to bus"""
    valid_action = {}
    print(f"  > Target: ", end="")
    
    for name, deg in angle_dict.items():
        if name in robot.bus.motors:
            phys_deg = deg + JOINT_OFFSETS.get(name, 0)
            if INVERT_DIRECTION.get(name, False): phys_deg = -phys_deg
            print(f"{name[:3]}:{phys_deg:.0f}° ", end="")
            
            valid_action[name] = deg_to_norm(deg, robot.bus.calibration[name], name)
            
    print("")
    for name, val in valid_action.items():
        try: robot.bus.write("Goal_Position", name, val)
        except: pass

def move_to_target_xyz(robot, chain, x, y, z):
    """Convert XYZ (mm) to joint angles and execute"""
    res = chain.inverse_kinematics([x/1000.0, y/1000.0, z/1000.0])
    target_angles = {
        "shoulder_pan":  np.rad2deg(res[1]),
        "shoulder_lift": np.rad2deg(res[2]),
        "elbow_flex":    np.rad2deg(res[3]),
        "wrist_flex":    np.rad2deg(res[4]),
        "wrist_roll":    np.rad2deg(res[5]),
        "gripper":       0
    }
    execute_move(robot, target_angles)

def safety_stow(robot):
    """Return arm to safe neutral pose"""
    print("\n Stowing...")
    pose = {"shoulder_pan": 0, "shoulder_lift": 0, "elbow_flex": 60, "wrist_flex": 0, "gripper": 0}
    for name, val in pose.items():
        if name in robot.bus.motors:
            try: robot.bus.write("Goal_Position", name, val)
            except: pass
    time.sleep(2)

def run_cli(robot, chain):
    """Interactive command line interface"""
    print("\n" + "="*40)
    print(" COMMANDS: 'x y z' | 'home' | 'q'")
    print("="*40 + "\n")

    while True:
        try:
            cmd = input(">> ").strip().lower()
            if cmd in ['q', 'exit']:
                safety_stow(robot)
                break
            if cmd == 'home':
                safety_stow(robot)
                continue
            
            pts = cmd.split()
            if len(pts) == 3:
                move_to_target_xyz(robot, chain, *map(float, pts))
                time.sleep(0.1)
        except Exception as e:
            print(f"Error: {e}")

if __name__ == "__main__":
    try:
        arm = init_hardware()
        ik = init_kinematics()
        run_cli(arm, ik)
    except KeyboardInterrupt:
        pass
    finally:
        if 'arm' in locals():
            try: arm.disconnect()
            except: pass
