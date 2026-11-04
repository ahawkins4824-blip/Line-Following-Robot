# Line Following Robot

An autonomous line-following robot built for **ENMT221 - Mechatronics Design 1** at the **University of Canterbury**, Semester 2, 2026.

**Start with [Code_BlackLine.txt](Code_BlackLine.txt).** This is the simpler, recommended firmware for following a **black line on a light background**. It performs much better than the competition version in the project owner's testing.

Group 03 A designed and built the mechanical transmission, custom electronics, sensor array, and embedded controller. The original competition firmware followed a **white line on a black surface** and is retained as a historical reference. It completed all three demonstration runs, with a best lap of **17.68 seconds**, placing **12th in the cohort**. Those results belong to the competition code, rather than the improved black-line version.

<img src="docs/images/robot-final.jpg" alt="Final robot with its PCB chassis, two friction-driven wheels, battery pack, and front sensor board" width="320">

*Competition robot layout, reproduced from Figure 9 of the project report.*

## Repository contents

| File | Description |
| --- | --- |
| [Code_BlackLine.txt](Code_BlackLine.txt) | **Recommended firmware.** Simpler black-line controller with weighted-average position estimation and serial diagnostics. |
| [Safe Final 03A.txt](Safe%20Final%2003A.txt) | Original, more complex white-line firmware used in the competition. |
| [Group 03A Code.zip](Group%2003A%20Code.zip) | Archive containing an identical copy of the competition firmware, `Safe Final 03A.txt`. |
| [LFR Documentation.pdf](LFR%20Documentation.pdf) | Original competition report: requirements, mechanical and PCB design, controller development, testing, and competition results. It does not document the newly added black-line version. |

Both firmware versions are supplied as Arduino C++ text files. Editable PCB designs, CAD models, and ready-to-open Arduino sketches are not included.

## Hardware and mechanical design

| Subsystem | Implementation |
| --- | --- |
| Microcontroller | ATmega4808-AFR |
| Line sensing | Five TCRT5000 reflective infrared sensors, read as analogue inputs |
| Motors | Two Multicomp MM28 DC motors |
| Motor drivers | Two BSR802N low-side N-channel MOSFET stages, controlled independently by PWM |
| Power | Four AA NiMH cells in series, nominally 4.8 V; MCP1700 regulator for the 3.3 V logic supply |
| Transmission | Single-stage friction drive: tape-wrapped motor shafts drive O-ring tyres on 3D-printed wheels |
| Structure | Main PCB doubles as the chassis; separate adjustable sensor PCB with 3D-printed supports and motor mounts |

The final transmission uses approximately **50 mm wheels**, including the O-rings, and **2.7 mm effective shaft diameters**, including electrical tape, giving an effective reduction of **18.5:1**. Rubber-band tensioning maintains contact between each shaft and wheel. Matching the tape thickness on both shafts improved straight-line consistency.

The separate sensor board allowed its distance from the wheels to be adjusted during testing. The report describes a sensor ground clearance of approximately **2.5 mm**. The original competition course specification used a **15-20 mm white line** and a minimum bend radius of **80 mm** to the line centre. Use a black line on a light background when running `Code_BlackLine.txt`.

The motor stages provide forward drive and coasting. An H-bridge was considered during development but was not implemented, so the robot cannot reverse its motors or apply active electrical braking.

## Recommended firmware: Code_BlackLine.txt

The black-line version uses a **continuous weighted average** of the five calibrated sensor readings and a **fixed base PWM**. With the supplied gains, steering is proportional only: `kp = 70` and `kd = 0`. The code calculates a derivative term, but it contributes nothing at the default setting; there is no integral action.

1. **Start up:** hold both motors off and flash the LED twice, with 500 ms on and 500 ms off per flash. Tracking starts after approximately two seconds, without a separate startup kick.
2. **Read and calibrate:** read all five sensors, map each reading from its individual black/white reference values to a common range of 0-1000, and clamp it to that range. Black maps to 0 and white to 1000.
3. **Estimate position:** sum the calibrated readings. When the total is below `cutoff = 4700`, calculate a weighted average using weights `+2`, `+1`, `0`, `-1`, and `-2`. Despite its name, `find_max()` does not select the strongest sensor in this version.
4. **Steer:** add the correction to the left motor command and subtract it from the right. Both commands are limited to 0-120.
5. **Handle a missing line:** when the total is at least 4700, use `lastDirection × 2` as the position. This is a fixed steering fallback, without the competition version's timed search routine.

With `s0` to `s4` representing calibrated readings in processing order, the default calculation is:

```text
total      = s0 + s1 + s2 + s3 + s4
position   = (2*s0 + s1 - s3 - 2*s4) / total  [when total < 4700]
correction = 70 × position                  [with kd = 0]
left PWM   = constrain(80 + correction, 0, 120)
right PWM  = constrain(80 - correction, 0, 120)
```

The weights act on calibrated reflectance readings, where white is high; the code does not invert them into black-line strengths. Its motor correction signs are opposite those in the competition version. Use the calibration, weights, and steering signs together when adapting it to another robot.

The loop repeats continuously and keeps the LED on. At **115200 baud**, serial diagnostics print the total, motor PWM commands, correction, position, and five calibrated readings approximately every **1500 loop iterations**. This is not a fixed time interval, and there is no serial command parser.

### Pin assignments and sensor order

| Output | MCU pin symbol |
| --- | --- |
| Status LED | `PIN_PD7` |
| Left motor PWM | `PIN_PA3` |
| Right motor PWM | `PIN_PA2` |

Both versions use the same pins and sensor processing order. The following calibration values are specifically for **Code_BlackLine.txt**, with the arithmetic expressions in its arrays evaluated.

| Array index | Sensor constant | MCU pin symbol | Weight | Black reference | White reference |
| --- | --- | --- | --- | --- | --- |
| 0 | `SENSOR_4` | `PIN_PD4` | +2 | 469 | 969 |
| 1 | `SENSOR_5` | `PIN_PD5` | +1 | 627 | 966 |
| 2 | `SENSOR_1` | `PIN_PD1` | 0 | 469 | 972 |
| 3 | `SENSOR_2` | `PIN_PD2` | -1 | 469 | 971 |
| 4 | `SENSOR_3` | `PIN_PD3` | -2 | 448 | 965 |

The `black[]` and `white[]` arrays follow sensor order **4, 5, 1, 2, 3**, rather than numeric sensor order. Check physical sensor and motor orientation before adapting the mapping.

## Setting up and running the firmware

1. Clone or download this repository:

   ```bash
   git clone https://github.com/ahawkins4824-blip/Line-Following-Robot.git
   cd Line-Following-Robot
   ```

2. In the Arduino IDE, create a sketch named `LineFollowingRobot` and paste in the contents of [Code_BlackLine.txt](Code_BlackLine.txt). Save it as `LineFollowingRobot/LineFollowingRobot.ino`. Use this file for the recommended black-line version; select `Safe Final 03A.txt` only to reproduce the original white-line competition controller.

3. Configure an Arduino-compatible board core for the **ATmega4808** that supports the supplied `PIN_PA*` and `PIN_PD*` symbols, analogue inputs, and PWM outputs. The code uses Arduino core functions and has no explicit third-party library dependencies. The report identifies **8 MHz** as the best-performing clock setting for the original competition tests; a separate clock configuration for the black-line version is not recorded.

4. Select the programmer or bootloader upload method appropriate to the board. The custom PCB provides **UPDI** and **UART bootloader** programming headers. The exact board package, package version, fuse settings, bootloader, and programmer configuration are not recorded in this repository and must be confirmed for the target hardware.

5. Verify the pin mapping and measure black/white readings for each sensor on the intended track. Update the `black[]` and `white[]` arrays in `calibrate()` in the processing order shown above. Calibration is fixed in the source; there is no automatic calibration routine.

6. Compile and upload with the wheels clear of the surface. For a black-line track run, position the centre sensor over the black line on a light background before resetting or powering the robot. It starts tracking automatically after the **two LED flashes**. Open the Serial Monitor at **115200 baud** to inspect calibration and control values.

### Tuning parameters

These values apply to **Code_BlackLine.txt**. PWM values are commands on the Arduino 0-255 scale; this version caps each output at 120. They are not measured wheel speeds.

| Parameter | Value | Purpose |
| --- | --- | --- |
| `kp` | 70 | Proportional steering gain |
| `kd` | 0 | Disables derivative action; a nonzero value uses the change in error per loop, without time normalisation |
| `baseSpeed` | 80 | Fixed base PWM |
| `minSpeed` / `maxSpeed` | 0 / 120 | Limits for each motor command |
| `cutoff` | 4700 | Total calibrated reading at or above which the last-direction fallback is used |

Recheck sensor calibration after changes to lighting, sensor height, or sensor hardware. The report also found that mechanical slip and unequal shaft wrapping could resemble controller faults, so consistent transmission contact is important when tuning gains and speeds.

## Competition firmware and results

[Safe Final 03A.txt](Safe%20Final%2003A.txt) preserves the more complex controller used in the competition. It selects the strongest calibrated sensor, uses `kp = 18`, reduces base PWM from 100 to 66 away from the centre, and includes a PWM 140 startup kick and timed line-loss search. This remains useful as a record of the competition configuration; **Code_BlackLine.txt is the preferred version for current use**.

All three demonstration runs completed the original white-line course without losing the line, as recorded in Section 8 of the report. **These are historical competition results, not benchmark times for Code_BlackLine.txt.**

| Run | Outcome | Lap time |
| --- | --- | --- |
| 1 | Completed | 18.77 s |
| 2 | Completed | 19.11 s |
| 3 | Completed | **17.68 s** |
| Average | | 18.52 s |

The final competition run used a base PWM of 100 after an additional straight-line acceleration strategy in the second run produced less consistent motion. An earlier test lap of approximately 15 seconds was reported during development, but could not be reproduced reliably. No timed benchmark for the black-line version is included in the repository; its improved behaviour is based on the project owner's testing.

## Limitations and future improvements

- **All-dark input:** `Code_BlackLine.txt` divides by the total calibrated reading when it is below 4700. There is no zero-total guard, so a completely dark input that maps all five sensors to zero causes division by zero.
- **Line-loss behaviour:** the black-line version holds a fixed last-direction correction when the total is at least 4700. Direction memory starts at zero, so it cannot select a search direction until a nonzero position has been observed. Neither version includes finish-line detection, an automatic stop timeout, or a start/stop command interface.
- **Transmission consistency:** friction drive depends on shaft wrapping, contact pressure, and tyre condition. The report proposes investigating a gear-based transmission to reduce slip.
- **Power stability:** motor startup and rapid acceleration caused brownouts during testing. The report identifies the approximately 0.7 V drop across the battery-to-regulator protection diode as a possible contributor and proposes a lower-drop protection solution.
- **Wireless tuning:** a Raspberry Pi Pico 2W BLE-to-UART interface was trialled during the competition project but abandoned because it was unreliable. It is not included in either supplied firmware version, and the bridge code is not included.
- **Packaging:** integrating the sensor and main boards after settling on sensor placement could reduce overall size and inertia.

## Team and documentation

**Group 03 A:** Anton Hawkins, Cody Carter, Khush Patel, and James Buckland-Blair.

**Course:** ENMT221 - Mechatronics Design 1, University of Canterbury, Semester 2, 2026.

The [project report](LFR%20Documentation.pdf), dated **9 October 2026**, documents the original competition project. The recommended firmware description, calibration, and tuning values above have been checked against [Code_BlackLine.txt](Code_BlackLine.txt); the competition configuration is preserved in [Safe Final 03A.txt](Safe%20Final%2003A.txt).
