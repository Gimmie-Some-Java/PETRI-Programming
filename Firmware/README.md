# PETRI Dish Handling Klipper Configuration

This directory contains the Klipper configuration, macros, and UI definitions for an automated Petri dish handling system. It extends a standard Klipper setup with additional axes, motion constraints based on fluid sloshing analysis, and a button‑driven main program to load stacked Petri dishes in rows.

The configuration targets a machine with:
- Cartesian base kinematics (`X`, `Y`, `Z`)
- Two additional linear axes implemented as manual steppers (`U` interlock pusher, `W` staging pusher)
- TMC2209 drivers with stall detection
- A limit switch button for Petri dish sensing which activates transverse motion / program trigger
- KlipperScreen + Mainsail + Moonraker for UI and API access

---

## File Overview

- `printer.cfg` – Main Klipper configuration:
  - MCU definitions (`[mcu]`, `[mcu CB1]`)
  - Stepper configuration for `X`, `Y`, `Z` and manual steppers `U` (interlock) and `W` (staging)
  - TMC2209 driver settings (currents, stall detection, UART pins)
  - Global printer motion limits tuned from fluid sloshing analysis
  - Button and idle timeout configuration
  - Includes other configuration fragments (`macro.cfg`, `debug.cfg`, `uw_axes.cfg`, `petri_control.cfg`, `stall_detection.cfg`, `mainsail.cfg`, etc.)

- `petri_control.cfg` – Core Petri dish handling logic:
  - `_PETRI_SETTINGS` macro stores all tunable parameters (push positions, dish height, row count, feedrates, dwell times, and program state).
  - Macros for single operations:
    - `PUSH_TRANSVERSE`, `RETRACT_TRANSVERSE`
    - `PUSH_INTERLOCK`, `RETRACT_INTERLOCK`
    - `PUSH_STAGING`, `RETRACT_STAGING`
    - `PUSH_ELEVATOR`, `RETRACT_ELEVATOR`
  - Combined cycle macros:
    - `CYCLE_TRANSVERSE`, `CYCLE_INTERLOCK`, `CYCLE_STAGING`
  - Main state machine:
    - `MAIN_PROGRAM` – full multi‑row Petri dish loading sequence
    - `START_ROW_LOADING_SEQUENCE`, `WAIT_FOR_TRANSVERSE_BUTTON`, `CONTINUE_SEQUENCE`
    - `EXECUTE_STAGING_AND_ELEVATOR`, `CHECK_ROW_COMPLETION`, `EXECUTE_INTERLOCK_SEQUENCE`
    - `LOAD_ROW` – alternative row‑by‑row loading macro
  - Button‑driven control:
    - `_TRANSVERSE_BUTTON_HANDLER` – interrupt‑style handler called from a physical button
    - `TEST_TRANSVERSE_BUTTON`, `CONTINUE_TRANSVERSE`, `WAIT_FOR_BUTTON_CYCLES`
  - Parameter adjustment and persistence macros:
    - Setters/getters for dish height, number of rows, and push positions for all axes
    - `SAVE_PETRI_CONFIG` / `_LOAD_PETRI_SETTINGS` with `SAVE_VARIABLE`
    - `SHOW_PETRI_SETTINGS` for a quick overview of the current configuration

- `uw_axes.cfg` – 5‑axis XYZUW control layer:
  - Extends `G1` and `G28` to understand `U` and `W` axes and synchronize them with standard motion.
  - Tracks positions and homing state via `_UW_STATUS`.
  - Extends `M114`, `M112`, and `M18` so reports, emergency stop, and motors‑off also affect `U`/`W`.
  - `UW_AXES_INIT` delayed gcode initializes the extra axes on startup.

- `maf.cfg` – Multi‑Axis Framework (MAF) for Klipper:
  - Generic framework (by Rene K. Mueller) to map logical axes/tools onto Klipper steppers.
  - Provides extended handling of `G0/G1`, `G28`, `G90/G91`, `G92`, `M82/M83`, `M104/M109`, and `T0..T9`.
  - Used here to support additional axes and complex motion coordination.

- `macro.cfg` – Utility macros:
  - `home_all` – homes all axes including `U` and `W`.
  - `recover_all` – recovery sequence after an emergency stop.
  - `warmup_cycle` – repeated X/Z motion to wear in bearings and distribute lubricant.
  - `retract_all` – retracts all pushers/elevator via the `RETRACT_*` macros.

- `debug.cfg` – Debugging helpers:
  - `DEBUG_HOME` – sets kinematic positions / manual stepper axis mapping to allow motion without full homing.

- `KlipperScreen.conf` – KlipperScreen menu integration:
  - Custom main menu with entries for:
    - `MAIN_PROGRAM` / `RESTART_PROGRAM`
    - Homing (`HOME_ALL`, and per‑axis `G28` variants for transverse, elevator, interlock, staging)
    - Tuning dish height, dish rows, transverse/interlock/staging positions, and elevator positions
    - Quick test actions for each axis (e.g. `PUSH_TRANSVERSE`, `PUSH_INTERLOCK`, `PUSH_STAGING`)

- `mainsail.cfg` – Web UI (Mainsail) configuration (panel, macros exposure, etc.).
- `moonraker.conf` – Moonraker configuration for API/UI access.
- `crowsnest.conf` – Optional camera/stream configuration.
- `stall_detection.cfg`, `print_area_bed_mesh.cfg`, `sonar.conf`, etc. – Additional machine features (stall detection, mesh, sonar, timelapse).

---

## Prerequisites

This configuration assumes:

- A working Klipper installation on your host (e.g. SBC running Linux) and MCU flashed with matching Klipper firmware.
- TMC2209 drivers wired as per the pin assignments in `printer.cfg`.
- Manual steppers `stepper_u` and `stepper_w` wired and configured with appropriate endstops (if used).
- KlipperScreen, Mainsail, and Moonraker installed and pointing to this configuration directory.
- A physical "transverse" button wired to the pin specified in `printer.cfg` (`[gcode_button transverse_button]`).

Make sure you understand your hardware wiring and safety limits before enabling or modifying any of these configs.

---

## Installation & Usage

1. **Place the config files**
   - Copy all `*.cfg` files in this directory into your Klipper configuration directory (often `~/printer_data/config/`).
   - Ensure `printer.cfg` includes the other files via the existing `[include ...]` lines.

2. **Restart Klipper services**
   - Restart Klipper and Moonraker using your usual method (e.g. from Mainsail, `sudo service klipper restart`, or `systemctl restart klipper moonraker`).

3. **Verify configuration**
   - In your web UI (Mainsail/Fluidd) or KlipperScreen terminal, run:
     - `STATUS`
     - `SHOW_PETRI_SETTINGS`
   - Check that no configuration errors are reported and that all axes are listed as expected.

4. **Home the machine**
   - From KlipperScreen menu or G-code console:
     - Run `HOME_ALL` (or `G28` plus per‑axis homing if needed).
   - Verify that `X`, `Y`, `Z`, `U`, and `W` home correctly.

5. **Test individual motions**
   - Use KlipperScreen:
     - Tuning → Transverse / Interlock / Staging / Elevator menus.
     - Use the **Test** actions (e.g. `PUSH_TRANSVERSE`, `PUSH_INTERLOCK`, `PUSH_STAGING`, `PUSH_ELEVATOR`) at safe speeds.
   - Adjust positions as needed using the `INCREASE_*` / `DECREASE_*` macros and re‑test.

6. **Configure Petri dish parameters**
   - Set dish height and number of rows:
     - `SET_DISH_HEIGHT HEIGHT=<mm>`
     - `SET_MAX_ROWS ROWS=<n>`
   - Optionally, adjust push positions using:
     - `SET_TRANSVERSE_POSITION POSITION=<mm>`
     - `SET_INTERLOCK_POSITION POSITION=<mm>`
     - `SET_STAGING_POSITION POSITION=<mm>`
     - Elevator: via the KlipperScreen tuning menu or the `INCREASE_ELEVATOR_*` / `DECREASE_ELEVATOR_*` macros.
   - Save your configuration so it persists across reboots:
     - `SAVE_PETRI_CONFIG`

7. **Run the main program**
   - From KlipperScreen main menu:
     - `Main Program → Start` (runs `MAIN_PROGRAM`).
   - The sequence will:
     - Home the machine.
     - Move the elevator to loading position.
     - Wait for transverse button presses to execute a series of `CYCLE_TRANSVERSE` moves per row.
     - Perform staging cycles and drop the elevator between rows.
     - After the configured number of rows, perform an interlock sequence and restart for the next batch.

8. **Restarting / Recovery**
   - `RESTART_PROGRAM` – retracts all pushers, returns the elevator, resets the state machine, and starts loading again.
   - `recover_all` – homing and operator prompt after an emergency stop.
   - `retract_all` – retracts all axes to safe positions.

---

## Safety Notes

- This system moves stacked, fluid‑filled Petri dishes; incorrect parameters can cause spills or mechanical collisions.
- Always start with conservative feedrates and accelerations.
- Verify new settings with single‑row tests before running unattended multi‑row programs.
- Use the `estop_button` and `M112` emergency stop functionality as a last resort; verify they are reachable and working.

---

## Customization

You can extend or modify this setup by:

- Editing `petri_control.cfg` to change the state machine, timing, or number of transverse cycles per row.
- Adjusting motion limits and driver currents in `printer.cfg` for your hardware.
- Adding new KlipperScreen menu items in `KlipperScreen.conf` for frequently used macros.
- Extending `maf.cfg` mappings if you add more axes or tools.

After making changes, always:

1. Restart Klipper.
2. Check for configuration errors in the console.
3. Test motions with no load before running full programs.

---

## License / Attribution

- Multi‑Axis Framework (`maf.cfg`) is by Rene K. Mueller as indicated in the file header; see that file for licensing and attribution details.
- Other configuration files in this directory are part of the PETRI dish handling project; apply your project’s preferred license as appropriate.
