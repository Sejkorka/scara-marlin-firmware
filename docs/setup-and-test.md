# Setup and First-Movement Test Guide

This document captures the current, reproducible procedure for building the firmware with PlatformIO, connecting to the controller over USB/serial, and executing the very first motion tests on the SCARA arm. Follow every step in order—especially the safety checks—before attempting to attach a toolhead.

## 1. Required Toolchain

| Component | Notes |
| --- | --- |
| Git | Used to clone/update this repository. |
| Python ≥3.10 | PlatformIO is distributed as a Python package on all major OSes. |
| PlatformIO CLI | Provides the build system, device monitor, and uploader. Install with `pip install -U platformio` or via the VS Code extension pack. |
| C/C++ toolchain | PlatformIO will download the correct GCC packages (e.g., arm-none-eabi) for the environment defined in `platformio.ini`. |
| USB drivers | Ensure the target MCU enumerates as `/dev/ttyACM*`, `/dev/ttyUSB*`, COM*, or `/dev/cu.usbmodem*`. On Linux add a udev rule if needed. |
| Serial terminal | You can rely on `pio device monitor`, or use `screen`, `minicom`, or CoolTerm for raw serial sessions. |

> **Tip:** After installing PlatformIO, run `pio system info` once to confirm the Python environment and toolchain are healthy.

## 2. Build and Upload with PlatformIO

1. **Clone and enter the project**
   ```bash
   git clone <REPO_URL>
   cd <REPO_FOLDER>
   ```
2. **Inspect `platformio.ini`** (at the repository root) to confirm the default environment (e.g., `scara32`, `LPC1769`, or `STM32F4`). Adjust the `upload_port` if you want to hard-code a serial device.
3. **Install dependencies**
   ```bash
   pip install -U platformio
   pio pkg install
   ```
4. **Build**
   ```bash
   pio run -e scara32
   ```
   Swap `scara32` for the environment you actually target.
5. **Upload**
   ```bash
   pio run -e scara32 -t upload
   ```
   PlatformIO will automatically reset the board and flash the freshly built firmware.
6. **Capture the binary (optional)** – `pio run -e scara32 -t buildprog` drops the ELF/BIN files inside `.pio/build/<env>/` for offline flashing.

## 3. Connect to the Board over USB/Serial

1. Plug the controller in via USB and verify it enumerates:
   ```bash
   pio device list
   ```
2. Start a monitor session (115200 baud is typical for Marlin-based SCARA builds):
   ```bash
   pio device monitor -e scara32 --baud 115200
   ```
   or, with `screen`:
   ```bash
   screen /dev/ttyACM0 115200
   ```
3. If no output appears, manually reset the board or power-cycle the 24 V rail. Make sure no other application holds the serial port.
4. Enable verbose logging by sending `M111 S247` when extra diagnostics are required.

## 4. Bring-Up Procedure

> **Safety checklist:** Remove any end-of-arm tooling, keep the work area clear, and be ready to cut power. Never home with tooling installed until you have validated the envelope.

1. **Verify endstops**
   - Send `M119`.
   - Manually actuate each endstop and ensure the reported state switches between `OPEN` and `TRIGGERED`.
   - Investigate wiring if any switch fails to report correctly before continuing.
2. **Home all axes**
   - Send `G28` (or `G28 X Y` if Z is not yet safe).
   - Confirm each joint travels toward its respective limit switch, decelerates, and backs off.
3. **Jog each joint incrementally**
   - Switch to relative mode: `G91`.
   - Move Z up/down in 2–5 mm steps: `G0 Z5 F300`, `G0 Z-5 F300`.
   - Move the SCARA shoulder/elbow by jogging X/Y a few millimeters at a time: `G0 X5 F600`, `G0 X-5`, `G0 Y5`, `G0 Y-5`.
   - Return to absolute positioning afterward: `G90`.
4. **Sanity-check SCARA kinematics**
   - `M360` prints the inverse/forward kinematic solution for the current pose—verify the reported joint angles are physically plausible.
   - `M665` reports the configured arm segment lengths (`L`), offsets, and tower radius. Compare against the actual hardware. Adjust as needed before saving.
5. **Persist known-good values**
   - Use `M500` to store EEPROM-backed settings once you are confident in the configuration.
   - Keep a record of key responses (`M503`) in your lab notebook for reproducibility.

## 5. Mechanical Calibration Guidance

### 5.1 Steps-per-unit verification
- Command a 50 mm radial move (`G0 X50`) and measure the tip displacement.
- Compare the measured distance with the commanded move to compute the scale factor.
- Update the firmware via `M92 X### Y### Z###` or by editing the motion planner constants inside your configuration source (e.g., `Configuration.h`).
- Rebuild, upload, and re-test until the discrepancy is <0.1 mm.

### 5.2 Linkage length tuning
- Measure both arm segments (shoulder and elbow centers) with calipers.
- Issue `M665 L<outer-length> R<inner-length>` to update the geometry at runtime.
- Repeat `M360` at several XY positions; the reported angles should match physical expectations.
- Once validated, hard-code the lengths in your SCARA configuration file (e.g., `Configuration_scara.h`) and store them with `M500`.

### 5.3 Origin alignment
- Home (`G28`), then manually square the arm to the mechanical datum using small `G0` nudges.
- Set this as the workspace origin with `G92 X0 Y0 Z0`.
- Use `M206 X<offset> Y<offset> Z<offset>` to bake in any residual offsets so that future homing results in a usable coordinate frame.

## 6. Minimal Bench-Test G-code

Save the following snippet as `first-movement-test.gcode` and stream it through `pio device monitor --send first-movement-test.gcode` (or any sender that can drip-feed without a toolhead installed):

```
; first-movement-test.gcode
G21            ; millimeters
G90            ; absolute XY, origin after homing
M17            ; enable steppers
G28            ; full home
G4 P500        ; pause to settle
G91            ; switch to relative moves for short jogs
G0 Z5 F400     ; lift Z
G0 Z-5 F400    ; lower Z back
G0 Z5 F400     ; lift again to confirm repeatability
G90            ; back to absolute
G0 X10 Y0 F900 ; small radial move along +X
G0 X0 Y10 F900 ; +Y (elbow lead)
G0 X-10 Y0 F900
G0 X0 Y-10 F900
M360           ; print SCARA joint state
M114           ; report Cartesian position
G0 X0 Y0 Z0 F600
M18            ; disable motors
```

This script exercises Z up/down travel plus small XY excursions without requiring any tool-specific wiring. Observe the motion path carefully for binding or unexpected oscillations.

## 7. Future Laser Support (Placeholders)

### 7.1 Laser hardware bring-up *(TBD)*
Documentation for wiring the laser PSU, interlocks, and cooling loop will be added here.

### 7.2 Laser power calibration *(TBD)*
Procedures for test-firing, PWM scaling, and spot-size verification will live in this section.

### 7.3 Laser motion and safety validation *(TBD)*
Expect detailed checklists covering enclosure interlocks, light curtains, and controlled test patterns.

## 8. Reference Commands and Files

- `platformio.ini` – Defines the target MCU environment, upload speed, and dependencies.
- `Configuration.h`, `Configuration_adv.h`, `Configuration_scara.h` – Typical Marlin-style sources that hold steps-per-unit, SCARA linkage lengths, and safety limits.
- `M92`, `M206`, `M360`, `M365`, `M500`, `M503`, `G0`, `G28`, `G92` – Core G/M-codes referenced throughout this guide.
- `docs/setup-and-test.md` – This document; update it alongside any hardware or configuration changes so the bring-up process remains reproducible.
