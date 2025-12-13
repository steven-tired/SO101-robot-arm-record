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

---

## References
https://huggingface.co/docs/lerobot/so101?example=Linux

https://github.com/TheRobotStudio/SO-ARM100
