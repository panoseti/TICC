# Building and Flashing the TICC Firmware (Linux)

This guide explains how to install `arduino-cli`, compile the firmware in the
`TICC/` directory, and flash it to the Arduino Mega 2560 that controls the TICC.

The firmware runs on the Mega 2560's ATmega2560. The Python scripts in `bin/`
are helper tools that run on the host computer.

---

## 1. Install arduino-cli

### 1.1 Install with the official script (recommended)

```bash
mkdir -p ~/.local/bin
cd ~
curl -fsSL https://raw.githubusercontent.com/arduino/arduino-cli/master/install.sh | BINDIR=~/.local/bin sh
```

> **Note:** By default this script installs into a `bin/` directory **under the
> current directory**. The TICC repository already has a `bin/` tools directory,
> so always set `BINDIR` and don't run the script from inside the repository.
> Otherwise the executable ends up in the repository.

### 1.2 Check your PATH

```bash
echo $PATH | grep -q "$HOME/.local/bin" || echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
arduino-cli version
```

### 1.3 Install the AVR board support package

```bash
arduino-cli config init          # create a default config file (optional)
arduino-cli core update-index
arduino-cli core install arduino:avr
```

This step also installs the avr-gcc compiler and the avrdude upload tool.

### Other installation methods

- **Manual download:** Download `arduino-cli_*_Linux_64bit.tar.gz` from the
  Releases page of `arduino/arduino-cli` on GitHub, extract it, and put
  `arduino-cli` in a directory on your PATH.
- **Homebrew:** `brew install arduino-cli`

---

## 2. Compile the firmware

`TICC/` is a standard Arduino sketch directory. `TICC.ino` is the main file, and
the `.cpp` and `.h` files in the same directory are compiled with it. The target
board is `arduino:avr:mega`.

From the repository root, run:

```bash
arduino-cli compile --fqbn arduino:avr:mega --output-dir build TICC
```

`build/` is already listed in `.gitignore`, so it won't be committed. A
successful build ends with a resource usage summary, for example:

```
Sketch uses 49964 bytes (19%) of program storage space. Maximum is 253952 bytes.
Global variables use 1774 bytes (21%) of dynamic memory, leaving 6418 bytes for local variables.
```

The firmware to flash is `build/TICC.ino.hex`.

> The build prints a few `-fpermissive` warnings (duplicate declarations,
> repeated default arguments, and so on). They come from the existing code and
> don't affect the build output.

The firmware version is set by `SW_VERSION` and `SW_TAG` at the top of
`TICC/TICC.ino`.

---

## 3. Flash the board

### 3.1 Before you flash

1. **Serial port permissions:** Add your user to the `dialout` group, then
   **log out and log back in**:
   ```bash
   sudo usermod -aG dialout $USER
   ```
2. Connect the TICC (Mega 2560) over USB and confirm that the board is detected:
   ```bash
   arduino-cli board list
   ```
   It usually appears as `/dev/ttyACM0`.
3. Close any program that is using the serial port, such as a serial monitor,
   `screen`, or `bin/ticc_logger.py`.

### 3.2 Option 1: Flash your own build with arduino-cli

```bash
# Use the build/ directory from step 2
arduino-cli upload --fqbn arduino:avr:mega -p /dev/ttyACM0 --input-dir build

# Or compile and flash in one step
arduino-cli compile --fqbn arduino:avr:mega -u -p /dev/ttyACM0 TICC
```

### 3.3 Option 2: Flash a prebuilt release hex

The `binaries/` directory holds official release firmware, so you don't need to
compile anything. The bundled script requires `avrdude` to be installed on the
system:

```bash
sudo apt install avrdude
python3 bin/ticc_updater.py /dev/ttyACM0 binaries/TICC.ino-20251215.1.hex
```

The script runs this command:

```bash
avrdude -v -patmega2560 -cwiring -b115200 -D -P/dev/ttyACM0 -Uflash:w:<hexfile>:i
```

> The files in `binaries/` are official releases. The `main` branch is the
> development branch and may be newer than these hex files. To run the latest
> code, compile it yourself.

On Windows, you can use the AVRDUDESS or XLoader GUI tools in `bin/`. See the
user manual in `docs/` for details.

---

## 4. Verify after flashing

Open the serial port at 115200 baud:

```bash
arduino-cli monitor -p /dev/ttyACM0 -c baudrate=115200
# or
screen /dev/ttyACM0 115200
```

You should see the startup messages and the firmware version. Press `#` at
startup or while the TICC is running to open the configuration menu.

### EEPROM configuration reset

The configuration struct is stored in EEPROM, and its layout version is set by
`EEPROM_VERSION` in `TICC/config.h`. If the new firmware's version differs from
the one stored on the board, all settings are reset to defaults on first boot.
**The board serial number is kept.** You will need to re-apply any settings you
changed before.

---

## 5. Troubleshooting

| Symptom | Fix |
|---------|-----|
| `Permission denied: /dev/ttyACM0` | Add your user to the `dialout` group, then log back in |
| `arduino-cli board list` shows `No boards found` | Make sure the USB cable carries data, try a different USB port, or run `dmesg \| tail` to see whether the system detected the device |
| Flashing fails with a busy port or a timeout | Close serial monitors, loggers, and anything else using the port |
| `arduino-cli: command not found` | Make sure `~/.local/bin` is on your PATH (see section 1.2) |
