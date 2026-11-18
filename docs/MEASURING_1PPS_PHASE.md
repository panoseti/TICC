# Measuring the Phase Difference Between Two 1PPS Signals

This guide explains how to read TICC measurement results on a host computer, and
how to measure the phase (time) difference between two 1PPS signals.

For building and flashing the firmware, see [BUILD_AND_FLASH.md](BUILD_AND_FLASH.md).

---

## 1. Hardware setup

- **10 MHz reference:** Connect a 10 MHz reference to the TICC's clock input.
  The TICC needs it as its timebase and won't measure without it.
- **1PPS inputs:** Connect the two 1PPS signals to **chA (channel 0)** and
  **chB (channel 1)**. The TICC triggers on the **rising edge**.
- **USB:** Connect the Arduino Mega 2560 to the host computer. Results are sent
  as ASCII text lines over the USB serial port at **115200 baud** by default.

Lines that start with `#` are startup messages and comments, not measurements.

---

## 2. Choose an output mode

There are two ways to get the phase difference.

### Option A: Time Interval mode (simplest)

The firmware computes **chB − chA** and prints one line per second, in seconds:

```
0.00000012345
0.00000012351
...
```

- A **positive** value means B is later than A. A **negative** value means B is
  earlier than A.
- The line has no channel name. A line is printed only after both channels have
  a new measurement, so a pair of 1PPS inputs gives exactly one line per second.
- If one channel misses a pulse, its previous measurement is paired with the
  other channel's next one, which shows up as an outlier of about ±1 s.
  Filter these out on the host.

### Option B: Timestamp mode (default, keeps more information)

Each event is printed as a timestamp followed by the channel name:

```
1.12345678901 chA
1.12345691246 chB
2.12345678899 chA
2.12345691250 chB
```

Pair each chA timestamp with the chB timestamp from the same second, then
subtract to get the phase difference. This mode keeps the absolute timestamps
of each channel, so you can also analyze each channel's jitter and drift on its
own, or load the data straight into tools such as TimeLab.

If you need chA and chB to strictly alternate, use **Paired Timestamp** mode
(`A2`).

> **Binary mode (`A3`)** is for high-rate capture (over 1000 events/second).
> A pair of 1PPS signals only produces two events per second, so text mode is
> enough.

---

## 3. Configure the TICC

1. Open the serial port at 115200 baud, for example:
   ```bash
   screen /dev/ttyACM0 115200
   # or
   arduino-cli monitor -p /dev/ttyACM0 -c baudrate=115200
   ```
2. Press `#` to open the configuration menu.
3. Enter the commands you need:

| Command | Meaning |
|---------|---------|
| `A1` | Timestamp mode (default) |
| `A2` | Paired Timestamp mode |
| `A4` | Time Interval mode (chB − chA) |
| `C`  | Number of output decimal places (0–12, default 11 = 10 ps) |
| `E`  | Channel names (default `A` / `B`) |
| `G3` | Per-channel propagation delay in ps (see section 5) |
| `W`  | Write settings to EEPROM so they survive a restart |
| `2`  | Apply changes and resume measuring |
| `1`  | Discard changes and exit |
| `M`  | Show the menu again |

You can put several commands on one line, separated by `;`, which is handy for
scripts. For example, this switches to Time Interval mode, saves it, and
resumes:

```
A4;W;2
```

---

## 4. Read the results on the host

### 4.1 Log to a file

The repository includes a logger script:

```bash
pip install pyserial
python3 bin/ticc_logger.py /dev/ttyACM0 out.txt --echo
```

By default the script resets the Arduino when it opens the port. Add
`--no-reset` to keep the TICC running without a reset. Run
`python3 bin/ticc_logger.py --help` for all options.

### 4.2 Compute the phase difference in real time (Timestamp mode)

A Python `float` holds only about 15–16 significant digits. Once the timestamps'
integer part grows (after a long run), a `float` can no longer hold picosecond
resolution, so this example parses the values with `decimal.Decimal`:

```python
#!/usr/bin/env python3
# Print chB - chA for each second from TICC Timestamp mode output.
from decimal import Decimal
import serial

PORT = '/dev/ttyACM0'

ser = serial.Serial(PORT, 115200, timeout=5)
last = {}
while True:
    line = ser.readline().decode(errors='ignore').strip()
    if not line or line.startswith('#'):
        continue
    parts = line.split()
    if len(parts) != 2 or not parts[1].startswith('ch'):
        continue
    t, ch = Decimal(parts[0]), parts[1][2:]   # ch is 'A' or 'B'
    last[ch] = t
    if 'A' in last and 'B' in last:
        diff = last['B'] - last['A']
        if abs(diff) < Decimal('0.5'):        # same second
            print(f"B - A = {diff * Decimal(10**9):.3f} ns", flush=True)
            last.clear()
```

In Time Interval mode, each line is already the difference. The values are
small, so `float(line)` keeps full precision.

---

## 5. Accuracy notes

### Channel-to-channel offset

The two channels don't have identical cable and internal path delays, which
adds a fixed offset to the measured difference. To calibrate it:

1. Split one 1PPS signal (for example, with a power splitter or a pair of
   equal-length cables) and feed it to both chA and chB.
2. Measure in Time Interval mode for a while and average the result. The average
   is the channel offset.
3. Either subtract that offset on the host, or compensate in the firmware with
   `G3` (PropDelay) or `G6` (FUDGE0). Both take a value in ps for each channel
   (format `ch0/ch1`, e.g. `0/250`), and the two values are added together.

In the current firmware, a positive value on a channel makes that channel's
timestamps **later** by that many picoseconds (see `TICC/tdc7200.cpp`). Only
non-negative values are accepted. To line up the channels, add the delay to the
channel whose timestamps come out earlier. Verify the result with the split
signal after you change the setting.

### Resolution and noise

- The default output has 11 decimal places (10 ps). Change it with `C`.
- Single-shot measurement noise is on the order of tens of picoseconds.
  Averaging over many seconds reduces it.

### Ambiguity

The phase difference is only unambiguous within ±0.5 s. If the two 1PPS signals
are more than half a second apart, a difference near ±0.5 s may wrap to the
other side.
