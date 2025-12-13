# Day 2: Actuator Configuration & Environment Setup

> **Date**: 2025-12-13
> **Status**:  Completed
> **Focus**: Hardware Interface, WSL Integration, Motor ID Assignment

##  Summary

Successfully configured 12x **Feetech STS3215** bus servos for the LeRobot SO-100/101 system. This includes the full set for the **Follower Arm** (Robot) and the **Leader Arm** (Teleoperation). The process required bridging the gap between Windows hardware drivers and the Linux (WSL2) development environment.

## Technical Challenges & Solutions

### 1. The WSL2 Hardware Isolation Issue
**Problem**: The LeRobot scripts running in Ubuntu (WSL) could not detect the Waveshare Bus Servo Adapter because WSL2 isolates USB devices from the host OS by default.

**Solution**: Implemented USB Passthrough using `usbipd-win`.
* Identified the Bus ID via PowerShell: `usbipd list`
* Attached the device to the virtual environment:
    ```powershell
    usbipd bind --busid <ID>
    usbipd attach --wsl --busid <ID>
    ```
* **Workflow Optimization**: Adopted a strict "Hot-Swap" protocol—disconnecting only the motor cables while keeping the USB connection active to prevent the WSL serial link (`/dev/ttyACM0`) from dropping.

### 2. Serial Port Permissions
**Problem**: Encountered `PermissionError: [Errno 13]` when accessing the serial port.

**Solution**: Modified port privileges to allow read/write access for the user:
```bash
sudo chmod 666 /dev/ttyACM0
```
Configuration Workflow
----------------------

Since all motors ship with a default ID of 1, connecting them simultaneously causes bus conflicts. I utilized a specific "One-by-One" configuration strategy:

1.  **Isolation**: Connected a single motor to the driver board.
    
2.  Bash# Example command for Followerlerobot-setup-motors --robot.type=so101\_follower --robot.port=/dev/ttyACM0
    
3.  **Labeling**: Physically labeled each motor immediately after software configuration to ensure correct assembly order.
    
4.  **Hardware Swap**: Replaced the driver board to configure the secondary set of motors for the Leader Arm.
    

Hardware Visuals
----------------

1.  **The "Brains": Waveshare Bus Servo Adapter (A)**
    
    *   Analyzed the board layout: used the DC Jack for 12V power supply and USB-C for serial data.
        
    *   Configuration jumpers were verified in Mode B (USB -> SERVO).
        
2.  **The "Muscles": Configured Motors**
    
    *   All 12 motors are now configured with unique IDs and labeled (Shoulder, Elbow, Wrist, Gripper) for both Leader and Follower arms.
