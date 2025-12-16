# Project Log: STS3215 Motor Zeroing & Low Voltage Workaround

**Date:** December 15, 2025
**Project:** LeRobot SO-100 Arm Assembly
**Status:** Zeroing Complete / Hardware Issue Flagged / Ready for Assembly

---

## 1. The Problem
Failed to drive 6 daisy-chained STS3215 motors using a 5V 4A charger with official LeRobot scripts (`single_motor_zero.py`).
* **Symptom:** Terminal showed `No response`, unable to connect.
* **Root Cause:** Voltage drop from the 5V source was too significant. The official library's safety checks (reading voltage register) triggered an early exit, or the combined load was too high.

## 2. Troubleshooting & Solution

### Phase 1: Survival Check
* **Action:** Ran raw serial scanner `scan_everything.py`.
* **Result:** ` Found! Motor ID: 6`.
* **Insight:** Chips are alive and communication works. 5V is sufficient for standby/ping but fails under official library initialization.

### Phase 2: Brute Force Zeroing
* **Action:** Bypassed LeRobot library using `force_zero.py` (Raw Serial commands).
* **Logic:** `Enable Torque` -> `Write Position 2048` (Ignored voltage checks, no wait for response).
* **Result:** Motor moved audibly, shaft locked (Torque Enabled).
* **Insight:** Success. Low-level commands can bypass safety mechanisms to force movement in low-voltage conditions.

### Phase 3: Automation & Verification
* **Action:** Developed `auto_scan_verify.py`.
* **Workflow:**
    1.  Scan IDs 0-10.
    2.  On detection: Lock -> Wiggle Left/Right -> Zero (2048) -> **Relax (Disable Torque)**.
    3.  **Power Management:** Added 1-second sleep between motors to prevent 5V PSU overload.
* **Result:** All 6 motors passed the "wiggle test" and are zeroed.

## 3. Hardware Safety Flag
* **Issue:** Hot glue strain relief detached on one motor's 3-pin connector.
* **Risk:** High risk of solder joint fatigue or snapping during arm movement.
* **Mitigation Strategy:**
    1.  **Repair:** Re-apply hot glue or reinforce with electrical tape before assembly.
    2.  **Deployment:** Assign this specific motor to **ID 1 (Base)**.
    * *Reasoning:* The base motor wires are stationary and secured to the chassis, offering the safest environment for a compromised connector.

## 4. Key Takeaways
1.  **Official vs. Raw:** Official libraries fail safe on low voltage; raw serial commands force execution.
2.  **Load Management:** A 5V PSU cannot drive 6 active motors simultaneously. Sequential activation ("Move one, then rest") solved the power budget issue during calibration.
3.  **Assembly Rule:** Motors are zeroed at 2048 but **powered off**. Do not manually rotate the shaft to fit the horn; adjust the horn angle instead.
