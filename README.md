# LeRobot SO-101: 6-Axis Teleoperation & Imitation Learning

![Status](https://img.shields.io/badge/Status-Teleoperation%20Active-success)
![Hardware](https://img.shields.io/badge/Hardware-SO--101%20%7C%20STS3215-blue)
![Tech](https://img.shields.io/badge/Stack-WSL2%20%7C%20LeRobot-orange)

> **Project Goal**: Constructing a low-cost, open-source dual-arm robot system to democratize AI data collection and Imitation Learning.

This repository documents the engineering journey of the **SO-101 Robot Arm**, built on top of the [Hugging Face LeRobot](https://github.com/huggingface/lerobot) framework. It covers mechanical assembly, embedded servo configuration, and the complex software bridge required to run robotics stacks on Windows via WSL2.

---

## Demo (Work in Progress)


https://github.com/user-attachments/assets/f68c8e8b-f97f-4ace-9f6e-6af80a342441


---

## Tech Stack & Architecture

My development environment is designed to bridge **Windows hardware accessibility** with **Linux robotics ecosystems**.

| Component | Specification | Purpose |
| :--- | :--- | :--- |
| **Host OS** | Windows 11 | Driver support & Primary workstation |
| **Runtime** | WSL 2 (Ubuntu 22.04) | ROS-compatible environment for LeRobot |
| **Actuators** | Feetech STS3215 | Serial Bus Servos with position feedback |
| **Middleware** | `usbipd-win` | **Crucial:** Low-latency USB passthrough to WSL |
| **Language** | Python 3.10 (Miniconda) | Control logic & ML pipeline |

## Tech Stack & Environment

My development environment bridges Windows hardware with Linux software capabilities.

* **Host OS**: Windows 11
* **Virtualization**: WSL 2 (Ubuntu 22.04)
* **Package Manager**: Miniconda
* **Language**: Python 3.10
* **Key Tooling**:
    * `usbipd-win` (Crucial for passing USB devices from Windows to WSL)
    * `LeRobot` Python Library
---

## Roadmap & Milestones

- [x] **Mechanical Assembly: 3D printed parts (PLA+) & structural integration.**
- [x] **Firmware Setup: Servo zeroing and ID conflict resolution.**
- [x] **WSL2 Environment: Successful compilation of LeRobot dependencies.**
- [x] **Teleoperation: Real-time Leader-Follower synchronization (Data Collection Mode).**
- [ ] Visual Data Pipeline: Configuring host connectivity for mounted wrist/overhead cameras to synchronize image streams for data collection.
- [ ] Model Training: Training a policy using ACT (Action Chunking with Transformers).

---
## Key Engineering Challenges

### 1. The "Windows-Linux" Hardware Bridge
Running robotics hardware on Windows is notoriously difficult. I utilized `usbipd-win` to bind the Feetech servo controller (UARTHS) to the WSL2 kernel.

```bash
# Example of binding the USB device to WSL instance
usbipd bind --busid 2-1
usbipd attach --wsl --busid 2-1
```

### 2. Dependency Conflicts: ROS2 vs. LeRobot (Log 6)
The LeRobot framework requires specific PyTorch CUDA bindings that conflict with the standard system-level Python environment used by ROS2.
* **Problem:** Attempts to run LeRobot within the global ROS environment caused dependency collisions (detailed in `/logs/log6.md`).
* **Solution:** Enforced strict environment isolation using Conda. I architected a workflow where the hardware interface runs in a dedicated environment, communicating with the planning layer via decoupled sockets.

### 3. USB Hub & Physical Path Instability
Connecting multiple serial devices via a USB hub caused dynamic enumeration issues where port paths (e.g., `/dev/ttyUSB0`) would shift randomly upon reboot.
* **Problem:** LeRobot's configuration files expect static paths, causing initialization failures when the USB hub re-indexed devices.
* **Solution:** Implemented `udev` rules to bind devices by their unique hardware UUIDs instead of their volatile physical bus IDs, ensuring deterministic connection paths regardless of boot order.

### 4. Servo Calibration & ID Mapping
Configured 12 independent degrees of freedom (DoF) across two arms.
* **Leader Arm**: Hand-held teleoperation unit (Servos ID 1-6).
* **Follower Arm**: The actuator unit executing tasks (Servos ID 1-6, mapped logically).
* **Solution**: Wrote custom Python scripts to verify baud rates (1M) and flash EEPROM parameters for PID tuning.
---

## Acknowledgements

Built upon the incredible open-source work of [The Robot Studio](https://github.com/TheRobotStudio/SO-ARM100) and the [Hugging Face LeRobot Team](https://github.com/huggingface/lerobot).

---

## Development Log

### [Day 1: Actuator Configuration & WSL Setu](./log/DevLog_1.md)
Successfully configured the full 12-motor set for both Leader and Follower arms. Key technical achievements included resolving WSL2 USB isolation issues via usbipd-win and fixing serial port permission (chmod) conflicts to enable reliable motor communication.

### [Day 2: Motor Zeroing & Power Diagnostics](./log/DevLog_2.md)
Addressed the removal of the "Auto-Zero" feature in the latest LeRobot stack by developing a custom "Bare Metal" Python script to bypass high-level abstractions and force motors to neutral (`2048`). Diagnosed intermittent communication failures ("Zombie Motor" phenomenon) caused by voltage drops in the 5V daisy chain. Successfully performed mechanical extraction of seized connectors to enable reconfiguration.
(after the assembly, 5v charger seems to be enough to drive the robot arm. The reason why motor fails to response at day 2 remains unknown.)

### [Day 3: Low-Voltage Workarounds & Raw Serial Zeroing](./log/DevLog_3.md)
Successfully zeroed all 6 STS3215 motors using a 5V power supply by bypassing the official LeRobot library's voltage safety checks. Developed a custom "Raw Serial" Python script utilizing a sequential "pulse-and-relax" strategy to zero the motor one by one in sequence. 

### [Day 4: Assembly, Calibration Anomalies & Motion Verification](./log/DevLog_4.md)
Completed the physical assembly and resolved serial bus communication failures caused by OS permission restrictions and USB latch-ups. Corrected a "folded-state" calibration error by manually re-aligning homing offsets to a physical L-shape neutral pose, ensuring software-to-hardware geometric alignment. Verified safe multi-joint actuation across all axes using a custom normalized coordinate script (`move_test.py`) to confirm mechanical integrity before kinematic integration.

### [Day 5: Kinematic Refining, URDF Optimization & Coordinate Control](./log/DevLog_5.md)
Refined Inverse Kinematics (IK) integration by transitioning from normalized motor steps to precise XYZ coordinate control. Modified the source URDF document to calibrate joint reference frames and fix a -33° shoulder offset, aligning the mathematical model with physical reality. Implemented a robust control script featuring degree-to-normalization mapping and safety stow logic to prevent mechanical collisions during testing.

### [Day 6: ROS2 Integration & Hardware-to-Simulation Gap](./log/DevLog_6.md)
Integrated the SO-101 arm into the ROS2 ecosystem, achieving kinematic validation through Rviz simulation. Diagnosed a `StopIteration` deadlock in the teleoperation CLI caused by a logical conflict between custom calibration IDs and hardware driver mapping. Established `udev` rules to grant the ROS2 environment persistent serial permissions without compromising Conda pathing. 
(Simulation confirmed that coordinate frame offsets are accurate, though physical control is currently limited by the software's driver-identification logic.)

### [Day 7: Multi-Arm Integration & USB Hub Topology](./log/DevLog_7.md)
Successfully enabled simultaneous operation of both Leader and Follower arms through a single USB Hub by implementing physical port mapping (`by-path`) to resolve `CH343` identity conflicts. [cite_start]Refactored the `attach.bat` [cite: 1] [cite_start]automation script to search for `CH343` serial devices [cite: 2] [cite_start]and mount multiple devices to WSL simultaneously[cite: 7]. Resolved violent motor "glitching" and overload errors by purging stale calibration caches and performing a hardware-aligned re-calibration for the dual-arm setup. 
(Verified that independent 5V power supply for each arm is essential for stable communication; once ports were correctly mapped, teleoperation functioned without latency despite shared USB bandwidth.)

---

## References
https://huggingface.co/docs/lerobot/so101?example=Linux

https://github.com/TheRobotStudio/SO-ARM100
