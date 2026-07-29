# hwdef Change — Boot Without Barometer

This is the exact change applied to the board's hardware definition file to allow ArduPilot to boot without a barometer.

**File:** `libraries/AP_HAL_ChibiOS/hwdef/SPEDIXF405/hwdef.dat`

**Change:** two lines added right after the existing barometer declaration.

```diff
 # SPL06 integrated on I2C1 bus
 define AP_BARO_SPL06_ENABLED 1
 BARO SPL06 I2C:0:0x76
+# allow boot without baro (this board variant may ship without the SPL06)
+define HAL_BARO_ALLOW_INIT_NO_BARO 1
```

The original `BARO SPL06` line is intentionally kept: if a barometer is ever present, it will still be detected and used. The added define only tells the firmware to continue booting when no barometer is found, instead of halting on a fatal config error.

**Full modified source:** the complete file lives in the [`spedixf405-no-baro`](https://github.com/GTC-Git/ardupilot/tree/spedixf405-no-baro) branch of the ArduPilot fork.

**Firmware version:** built from ArduCopter tag `Copter-4.7.0`.
