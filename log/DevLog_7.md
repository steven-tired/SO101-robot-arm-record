# Day 2: Multi-Arm Integration & USB Hub Topology

> **Date**: 2026-01-25
> **Status**: Completed
> **Focus**: Multi-Device Passthrough, Port Mapping (by-path), Calibration Recovery, USB Topology

## Summary

Successfully enabled simultaneous operation of both **Leader** and **Follower** arms through a single USB Hub. This phase focused on overcoming hardware identity conflicts (duplicate CH343 serial IDs) and ensuring stable communication in a shared bandwidth environment. The day concluded with a successful teleoperation session where the Follower arm precisely mirrored the Leader arm's movements without motor "overload" or logic inversion.

## Technical Challenges & Solutions

### 1. The "Duplicate ID" Conflict
**Problem**: Both arm controllers use identical CH343 USB-to-Serial chips with the same internal Serial Number. This caused Linux to overwrite the `/dev/serial/by-id/` entries, making it impossible to distinguish the Leader from the Follower via standard IDs.

**Solution**: Switched to **Physical Port Mapping** using `by-path`.
* Identified the unique physical bus paths:
    * `platform-vhci_hcd.0-usb-0:1:1.0` (Leader)
    * `platform-vhci_hcd.0-usb-0:2:1.0` (Follower)
* This ensures that as long as the arms are plugged into the same Hub ports, their software identity remains fixed regardless of connection order.

### 2. Motor "Ghosting" & Overload Errors
**Problem**: The arms experienced violent "glitching" and `RuntimeError: Overload error!`. This was caused by the system applying old calibration data from a previous session where the ports were inverted.

**Solution**: Forced a clean state and hardware-software re-alignment.
* **Targeted Re-calibration**: Executed `lerobot-calibrate` using specific `by-path` ports to bind new calibration constants to the correct physical hardware.

### 3. Automated Multi-Device Attachment
**Problem**: The initial `attach.bat` script only supported one device at a time, hindering the setup process for dual arms and future cameras. [cite: 1, 2]

**Solution**: Refactored the PowerShell automation to loop through all detected CH343 devices. [cite: 3, 4]
* **Modified Script Logic**:
    ```powershell
    $devices = usbipd list | Select-String 'CH343' | ForEach-Object { $_.Line.Trim().Split(' ')[0] };
    foreach ($id in $devices) { usbipd attach --wsl --busid $id; } $

## Configuration Workflow

1.  **Connectivity**: Plugged both arms into a powered USB 3.0 Hub, ensuring independent 5V power supply to the motor bus to prevent voltage sags.
2.  **Mapping**: Confirmed `0:2` as the Follower port via iterative testing.
3.  **Command Execution**: 
    ```bash
    lerobot-teleoperate \
        --robot.port=/dev/serial/by-path/platform-vhci_hcd.0-usb-0:2:1.0 \
        --teleop.port=/dev/serial/by-path/platform-vhci_hcd.0-usb-0:1:1.0
    ```
4.  **Verification**: Confirmed "Torque Off" state for the Leader arm and "Active Mirroring" for the Follower arm.

## Hardware Visuals & Teleoperation Demo

### 1. Wiring Topology (Dual-Arm + Hub)
![20260125_173036](https://github.com/user-attachments/assets/c21ea8d0-923a-49db-b1b1-55e6bb68adf7)

> *Description: Visualizing the connection between the 5V independent power supply, the USB Hub, and the PC interfaces.*

### 2. Teleoperation Success Video

> *Description: Showing the real-time response of the Follower arm (0:2) mirroring the Leader arm (0:1) with no latency or overload issues.*


https://github.com/user-attachments/assets/38f229cf-e7a8-4aba-a9f4-63999367fce0

---

**Next Step**: Integrate dual-camera feeds (Top/Side views) into the `by-path` mapping system to begin RL dataset recording.
