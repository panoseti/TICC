# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Firmware for the TAPR TICC: a sub-nanosecond time-interval counter shield (two TI TDC7200 chips) on an **Arduino Mega 2560** (ATmega2560, 8-bit AVR). Output is ASCII (or a binary frame format) over USB serial. `main` is the development branch; releases live in branches named by version. `docs/TICC_architecture.md` is the authoritative design doc — read it before touching timing/measurement code. Files in `docs/development_docs/` are AI-generated snapshots and may be stale.

## Build / flash / test

- `TICC/` is an Arduino sketch folder (no Makefile or CI). Build with the Arduino IDE or arduino-cli, board `arduino:avr:mega`:
  - `arduino-cli compile --fqbn arduino:avr:mega TICC`
  - `arduino-cli upload --fqbn arduino:avr:mega -p /dev/ttyACM0 TICC`
- Flash a prebuilt hex: `bin/ticc_updater.py /dev/ttyACM0 binaries/TICC.ino-<ver>.hex` (wraps `avrdude -patmega2560 -cwiring -b115200 -D`). Released hex files are committed to `binaries/`.
- Host-side unit test of the timestamp math (a standalone copy of the logic, not linked to the firmware — keep it in sync manually):
  `g++ -std=c++11 -o bin/verification/test_timestamp_calc bin/verification/test_timestamp_calc.cpp && bin/verification/test_timestamp_calc`
- Other `bin/` and `bin/verification/` Python scripts are host tools for capturing/decoding/analyzing TICC output against real hardware (not part of the core distribution).

## Firmware architecture

- `TICC.ino`: `setup()` is empty; `loop()` calls `ticc_setup()` (`setup.cpp`) and then runs its own `while(1)` processing loop. Returning from `loop()` is the restart mechanism (config change requiring restart, reference clock lost). Version string is `SW_VERSION` / `SW_TAG` at the top of `TICC.ino`.
- **Timing model**: a 10 kHz COARSE_CLOCK (from the external 10 MHz ref) drives an ISR incrementing `PICcount` (100 µs ticks, falling edge). Each channel's gated STOP rising edge fires an ISR capturing `PICstop`. The TDC7200 (mode 2) measures START→STOP time-of-flight; timestamp = `PICstop × PICTICK_PS − tof`, accumulated as `Timestamp64 {int32 seconds, uint64 picos}` (`timestamps.cpp`). TOF/calibration math (incl. `time_dilation` correction, `fixed_TIME2`) is in `tdc7200.cpp`.
- Main loop polls both channels' INTB, processes ready channels (re-checking the other to keep output ordered), then dispatches on `config.MODE` (`MeasureMode` enum in `config.h`: Timestamp, Paired_Timestamp, Binary, Interval, Period, Hat, Debug) to `print_*_mode()` in `print.cpp`. Binary mode bypasses formatting via `process_binary_mode()` and emits 12-byte frames `0x55 0xAA ch PICstop_low32(LE) tof(LE) CRC8` (decoded by `bin/ticc_binary_decode.py`).
- Hot-path config values are copied into globals (`PICTICK_PS`, `CLOCK_PERIOD`, `CAL_PERIODS`, `MODE`) at startup for speed.
- **Performance is the central constraint**: every per-event microsecond matters (~580 µs processing on AVR, dominated by 64-bit math). Existing optimizations to preserve: SPI auto-increment reads + direct-port CSB toggling (`tdc7200.cpp`), `uint32_t` calCount divisor, reciprocal-multiply digit formatting + PROGMEM 2-digit table (`print.cpp`). Avoid introducing 64-bit division, `digitalWrite`, or `Serial.print` of 64-bit values in the hot path; precision loss is not acceptable.
- AVR RAM is tiny: all user-facing strings go in PROGMEM (`config_menu_text.h`) and are printed with `configPrintProg`/`configPrintlnProg`.

## Configuration system

- `config_t` in `config.h` is stored verbatim in EEPROM at `CONFIG_START`; serial number lives separately at `SER_NUM_START` (0x0FF0) and survives resets. Defaults are `DEFAULT_*` macros + `defaultConfig()`.
- **Changing `config_t` layout requires bumping `EEPROM_VERSION`** — `setup.cpp` resets config to defaults when the stored version byte differs.
- The `#` key on serial enters the menu. Menus are table-driven: `CommandEntry`/`CommandConfig` tables in `config_command_table.cpp` (letter, value type, PROGMEM prompt, min/max, pointer to `config` field, confirmation type). Adding a simple parameter is usually a table row + prompt string + `config_t` field + default. Multiple commands per line separated by `;` run non-interactively (`interactive=false` path) for scripting. Logic lives in `config_core.cpp` / `config_menu.cpp`; `config_change_requires_restart()` decides whether exiting the menu restarts.

## Other directories

- `multi-ticc/`: Raspberry Pi server/scripts and enclosure files for racks of TICCs (uses an older pinned hex).
- `picdiv/`: PIC assembly for the frequency divider generating the coarse clock.
- `docs/`: user manual, schematics, datasheets, TI example code.
