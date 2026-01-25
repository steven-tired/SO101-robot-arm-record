# LeRobot SO-101 Robot Arm Build Journey

![Status](https://img.shields.io/badge/Status-In%20Progress-yellow)
![Robot](https://img.shields.io/badge/Hardware-SO--101-blue)
![Stack](https://img.shields.io/badge/Tech-WSL2%20%7C%20Python-green)

> **Current Status**: Assembling & Configuring

This repository documents the step-by-step process of assembling, configuring, and collecting data with the **SO-101 Robot Arm**, utilizing the [Hugging Face LeRobot](https://github.com/huggingface/lerobot) framework.

---

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
