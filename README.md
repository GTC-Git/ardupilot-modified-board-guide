# ArduPilot on Modified / Clone Flight Controllers — A Practical Guide

A step-by-step guide to **identifying, configuring, and flying limited or modified flight controllers with ArduPilot**, using a Spedix F405 variant as a real-world case study.

Many low-cost FPV flight controllers are sold under the name of a well-known board but ship with **different or missing hardware** than the official specification. When you flash ArduPilot onto one of these, you can hit failures that are hard to diagnose — a board that simply refuses to boot being the most common.

This guide documents the full process: how to detect that you have a non-standard board, how to prove which components are missing, how to build a custom firmware that boots anyway, and how to fly the drone safely within its real limitations.

---

## Motivation

This work was carried out in an R&D laboratory context. The flight controller was not selected based on datasheets or technical specifications — it came from a low-cost drone platform already available in the lab. Several similar platforms are used for research, and one was repurposed here for modification, study, and the development of new solutions.

The drone was originally configured and flown on **Betaflight**. For research purposes, the goal was to migrate it to **ArduPilot** to explore stabilization and, later, non-GPS navigation.

During this migration, ArduPilot failed to boot, halting on a fatal configuration error. Investigation revealed that the flight controller — which identifies itself as a **Spedix F405** — is in fact a **modified variant (clone)** that copies the official board's identity but not its full hardware, most importantly lacking the barometer present on the genuine board.

This repository turns that troubleshooting journey into a reproducible method that applies to any limited or cloned flight controller with an available ArduPilot target.

---

## Glossary — Components and What They Do

Before comparing the boards, here is a plain-language reference for each component and acronym used in this guide:

- **FC (Flight Controller):** the main board that runs the flight software and controls the motors.
- **MCU (Microcontroller Unit):** the processor at the heart of the FC. Here, an **STM32F405**. Its onboard **flash** memory storage determines how much firmware it can hold.
- **IMU (Inertial Measurement Unit):** the sensor that measures rotation and acceleration (gyroscope + accelerometer). It is what lets the drone know its attitude and keep itself level. **ICM42688** is a common IMU chip.
- **Barometer:** a pressure sensor used to estimate altitude. **SPL06** is the specific barometer chip expected by the official board. Without a barometer, altitude-hold flight modes are not available.
- **Magnetometer (Compass):** a sensor that measures heading relative to magnetic north. Needed for reliable yaw and for position-based modes.
- **GPS:** provides absolute position. On these FPV boards it is never built in — it is always an external module. Required for autonomous modes such as Return-to-Launch.
- **OSD (On-Screen Display):** overlays flight data (battery, timer, etc.) onto the analog video feed. **AT7456E / MAX7456** is the chip that does this.
- **DShot:** a digital protocol used to send throttle commands from the FC to the ESCs (the motor controllers). Comes in speeds such as DShot300 and DShot600.
- **hwdef:** in ArduPilot, the hardware definition file that tells the firmware exactly which components a given board has and where they are connected.

---

## How the Differences Were Verified

Every claim in the comparison below is backed by concrete evidence, gathered with four independent methods:

**1. Official specification (reference).**
The genuine board's specification is published in the [ArduPilot hardware documentation](https://ardupilot.org/copter/docs/common-spedixf405.html), which describes the official board (MCU, OSD, power, and more). This is the baseline the installed board is compared against.

**2. Betaflight configuration dump.**
Before flashing ArduPilot, the drone's original Betaflight configuration was fully backed up (`dump all`, `diff all`, and a flash memory backup — all included in this repository). Betaflight auto-detects sensors at boot, so its saved configuration effectively records a hardware scan. The most important line found was `set baro_hardware = NONE`, which means Betaflight's automatic sensor detection found **no barometer on any bus** — a hardware-level result, not a user setting.

**3. STM32CubeProgrammer readout.**
When flashing the firmware over USB (DFU mode), STM32CubeProgrammer reports the target details directly from the chip. It reported **"1 MB - Default"** flash size, versus the 4 MB specified for the official board.

**4. ArduPilot boot behavior.**
After flashing stock ArduPilot, the board halted in a boot loop with `Config Error: Baro: unable to initialise driver`, confirming that the firmware expected a barometer the board could not initialise.

Together, these four independent sources make the conclusion robust: the installed board **identifies as the official model but does not match its hardware**.

---

## Comparison: Official Board vs. Installed Board

The table below compares the **official board specification** against the **board actually installed** in this drone. Rows are marked as *Identical* (both boards match), *Different* (a real, verified divergence), or *Not verified* (not independently confirmed in this work).

| Feature | Official board | Installed board | Status |
|---|---|---|---|
| Board identity | SPEDIXF405 / SPDX | SPEDIXF405 / SPDX | Identical — the installed board reports the official identity |
| MCU (processor) | STM32F405 @ 168 MHz | STM32F405 | Identical |
| Onboard flash | 4 MB | 1 MB | **Different** — limits available firmware features |
| Barometer | SPL06 (I2C) | None | **Different** — root cause of the boot failure |
| IMU (gyro/accel) | ICM42688 | Present (SPI bus 1); exact chip not confirmed | Not verified — sensor present and working, model unconfirmed |
| OSD | AT7456E / MAX7456 | Present | Identical (functionally) |
| Magnetometer (compass) | None (external only) | None | Identical — neither board has an onboard compass |
| GPS | None (external only) | None | Identical — neither board has onboard GPS; both need an external module |

**Key takeaways:**

- The two **critical differences** are the **missing barometer** and the **reduced flash (1 MB vs 4 MB)**.
- The missing barometer is what stops stock ArduPilot from booting (see below).
- The reduced flash is why some ArduPilot features (such as Lua scripting) are not available in the build for this board.
- Everything else either matches the official board or is functionally equivalent.

---

## Root Cause: Why a Missing Barometer Stops the Boot

On the official board, ArduPilot's hardware definition (`hwdef`) declares a barometer with a line equivalent to `BARO SPL06 I2C:0:0x76`. This tells the firmware to expect an **SPL06 barometer on the I2C bus at address 0x76**.

Betaflight tolerates a missing barometer — it simply flies without altitude data. ArduPilot's Copter firmware treats it very differently: a barometer is considered essential, so when the firmware tries to initialise the declared SPL06 and finds nothing at that address, it raises a **fatal configuration error**: `Config Error: Baro: unable to initialise driver`, followed by `Config Error: fix problem then reboot`.

A *Config Error* is not a normal pre-arm warning — it happens during boot initialisation and **halts the main loop entirely**. The board sits in an endless error loop and never finishes starting up. Practical symptoms observed:

- The artificial horizon in the ground station stayed frozen.
- Raw gyro values (`gx`, `gy`, `gz`) read zero even while physically moving the board.
- Accelerometer calibration could not run.

Because this is a boot-time error, it **cannot** be fixed with arming-check parameters (`ARMING_CHECK` / skip-checks) — those act on a later stage the firmware never reaches. The fix must let the firmware complete initialisation **without** a barometer.

---

## The Solution: Build a Custom Firmware That Boots Without a Barometer

ArduPilot has a hwdef define that allows the firmware to finish booting even when no barometer is detected: `HAL_BARO_ALLOW_INIT_NO_BARO`. Adding it to the board's hardware definition and recompiling produces a firmware that no longer halts on the missing barometer.

> **Note:** This makes the board **boot and run** (IMU, attitude, radio, motor output, calibration all work). Actually flying without any altitude source (no baro, no GPS) is limited to **Stabilize / Acro** modes and may require adjusting a few arming checks. Altitude-hold modes are not available without a barometer.

### Step-by-step

**1. Set up the ArduPilot build environment.**
On Windows, use **WSL2 (Ubuntu)** with VS Code connected to it (the terminal must be the Ubuntu shell, not PowerShell). On native Linux/macOS, use the system shell directly.

**2. Fork and clone the ArduPilot source.**
Fork `ArduPilot/ardupilot` on GitHub, then clone your fork with submodules:

`git clone --recurse-submodules https://github.com/<your-user>/ardupilot.git`

**3. Check out the release matching your board.**
Match the version the board was (or will be) running. In this case, ArduCopter 4.7.0:

`git fetch upstream tag Copter-4.7.0`
`git checkout -b spedixf405-no-baro Copter-4.7.0`
`git submodule update --init --recursive`

**4. Edit the board's hwdef.**
Open `libraries/AP_HAL_ChibiOS/hwdef/SPEDIXF405/hwdef.dat` and add the define right after the barometer line:

`# allow boot without baro (this board variant may ship without the SPL06)`
`define HAL_BARO_ALLOW_INIT_NO_BARO 1`

Keep the original `BARO SPL06 I2C:0:0x76` line — if a barometer is ever present, it will still be used.

**5. Install build dependencies.**
Install the ArduPilot prerequisites and the ARM toolchain (`gcc-arm-none-eabi`). On newer Ubuntu releases the prereq script may fail on obsolete packages (e.g. `python3-argparse`); install the remaining packages manually if needed.

**6. Configure and build.**
`./waf configure --board SPEDIXF405`
`./waf copter`

The output firmware is generated under `build/SPEDIXF405/bin/` — including `arducopter_with_bl.hex` (with bootloader, for the first DFU flash) and `arducopter.apj` (for updates via ground station).

**7. Flash the board.**
Put the board in DFU mode (hold the BOOT button while plugging in USB) and flash `arducopter_with_bl.hex` using STM32CubeProgrammer (USB mode, address 0x08000000) or your preferred DFU tool.

**8. Verify.**
Reconnect normally and open the ground station. The `Config Error: Baro` message should be gone, the artificial horizon should track board movement, and accelerometer calibration should now run.

The exact one-line change made to the hwdef is available in this repository under `ardupilot-files/`, and the full modified board file lives in a dedicated branch of the ArduPilot fork.

---

## Reproducible Checklist — Working With a Suspect Board

If you suspect you have a limited, modified, or clone flight controller, follow this checklist to identify the differences, get it running, and fly it safely.

### Phase 1 — Identify the board

- [ ] Before erasing anything, **back up the original configuration** (in Betaflight: `dump all`, `diff all`, and a flash backup). Keep these — they let you revert and they record an automatic hardware scan.
- [ ] Note the board's reported **target/identity** (e.g. `board_name`, `manufacturer_id`).
- [ ] In the backup, check sensor lines such as `baro_hardware`. A value of `NONE` means that sensor was not detected on any bus.
- [ ] When flashing, read the **actual flash size** reported by the programmer (e.g. STM32CubeProgrammer) and compare it to the official spec.
- [ ] Build a comparison table: official spec vs. what your board actually has.

### Phase 2 — Get it booting

- [ ] Flash the **stock** ArduPilot firmware for the matching target first, and observe the boot messages.
- [ ] If you get `Config Error: Baro: unable to initialise driver` (or a similar sensor config error), the board is missing a sensor the hwdef expects.
- [ ] Build a **custom firmware** with the appropriate define (for a missing barometer: `HAL_BARO_ALLOW_INIT_NO_BARO`), following the build steps above.
- [ ] Flash the custom firmware and confirm the board completes boot (horizon moves, gyro reads live values).

### Phase 3 — Configure

- [ ] Set the correct **frame class and type**. For a Betaflight-wired quad, `FRAME_TYPE = 12` (BetaFlightX) maps the motors correctly.
- [ ] Run **accelerometer calibration** and **level calibration**.
- [ ] Disable sensors the board does not have (e.g. compass) to clear health warnings.
- [ ] Set the **motor protocol** to match the ESCs. Check the Betaflight backup — this board used `DSHOT300`. Set `MOT_PWM_TYPE` accordingly and `SERVO_DSHOT_ESC` to the ESC type.
- [ ] Configure and calibrate the **radio**. For ELRS, match the bind phrase and, if the link won't connect, lower the packet rate to a value both ends support.
- [ ] Run **Motor Test** and confirm each motor's **position and spin direction** are correct.

### Phase 4 — Fly safely within the limitations

- [ ] With **no GPS and no baro**, plan to fly in **Stabilize** (or Acro) only. Altitude and position hold are not available.
- [ ] Set failsafes for a no-GPS aircraft: radio failsafe to **Land** (`FS_THR_ENABLE = 3`), battery failsafe to **Land** (`BATT_FS_LOW_ACT = 1`) with a correct low-voltage threshold for your battery.
- [ ] Disable the fence (`FENCE_ENABLE = 0`) if it depends on GPS.
- [ ] Bench-test **arming/disarming** and the **tilt response** (tilt the armed craft, props off — the low side's motors should speed up) before installing props.
- [ ] First flight: open area, no people nearby, good weather, low hover only, finger ready on disarm.

---

## Flying With the Limitations

Once the board boots and is configured, it is important to be clear about what this hardware can and cannot do:

**What works:**
- Full attitude control and self-leveling (IMU-based).
- **Stabilize** and **Acro** flight modes.
- Radio control, motor output, OSD, battery monitoring.

**What does not work (missing hardware):**
- **Altitude hold** modes (AltHold, Loiter) — require a barometer, which this board lacks.
- **Position hold / autonomous** modes (Loiter, RTL, Auto) — require GPS (external module) and, ideally, a compass.

**Practical implications:**
- The pilot controls throttle/altitude manually at all times.
- Failsafes must assume no GPS: the safe action on radio or battery failsafe is **Land**, not Return-to-Launch.
- This makes the board perfectly usable as a **learning and development platform** for manual stabilized flight, while being honest about its ceiling.

---

## Further Verification (Not Performed)

For completeness, one additional test could have been used to prove the absence of the barometer at the electronics level: an **I2C bus scan** (via ArduPilot's Lua `i2c_scan` example script), sweeping every address on every I2C bus and showing that no device answers at any barometer address (0x76, 0x77, and the typical BMP/DPS/MS addresses).

**This test was not performed**, for two reasons:

1. The 1 MB flash build of this board **does not include Lua scripting** (the `SCR_ENABLE` parameter is absent), so the scan script cannot run without recompiling.
2. It is **not necessary**: the Betaflight backup already contains `set baro_hardware = NONE`, which is the result of the firmware's own automatic hardware scan across all buses — effectively an I2C scan already performed and recorded. Combined with the official spec, the boot-time Config Error, and the flash-size mismatch, the evidence is already conclusive.

**For anyone who wants the extra proof**, the I2C scan *can* be performed by rebuilding the firmware with scripting enabled (`--enable-scripting`), with the caveat that on a 1 MB board this may exceed the available flash and require disabling other features to fit.

This decision — knowing the more rigorous test, evaluating it, and consciously skipping it because the existing evidence was already sufficient — is itself part of a sound engineering process.

---

## Repository Contents

- **`README.md`** — this guide.
- **`betaflight-backup/`** — the drone's original Betaflight configuration, saved before flashing ArduPilot. Includes the full `dump all`, the `diff all`, and a flash memory backup. These allow the board to be reverted to its original state and also serve as a hardware scan record (this is where `baro_hardware = NONE` comes from).
- **`ardupilot-files/`** — the one-line hwdef change and related build notes for the custom firmware.
- **`images/`** — screenshots and photos referenced by this guide.

The full modified ArduPilot source lives in a dedicated branch of the fork: [`spedixf405-no-baro`](https://github.com/GTC-Git/ardupilot/tree/spedixf405-no-baro).

---

## Credits and License

This guide documents original troubleshooting and configuration work.

ArduPilot is an open-source project licensed under **GPLv3**. Any modified ArduPilot source code (in the linked fork) remains under GPLv3. This documentation and the accompanying notes are released under the **MIT License**.

The genuine board's reference specification is published in the [ArduPilot hardware documentation](https://ardupilot.org/copter/docs/common-spedixf405.html).

> **Disclaimer:** Drones can cause serious injury and property damage. Always test with propellers removed until the final flight step, fly only in safe, open areas, and comply with local regulations. Use this guide at your own risk.
