# LeRobot SO-100/101 Robot Arm Build Journey

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

### [Day 3: Low-Voltage Workarounds & Raw Serial Zeroing](./log/DevLog_3.md)
Successfully zeroed all 6 STS3215 motors using a 5V power supply by bypassing the official LeRobot library's voltage safety checks. Developed a custom "Raw Serial" Python script utilizing a sequential "pulse-and-relax" strategy to prevent power supply brown-outs during calibration. 

---

## References
https://huggingface.co/docs/lerobot/so101?example=Linux

https://github.com/TheRobotStudio/SO-ARM100
