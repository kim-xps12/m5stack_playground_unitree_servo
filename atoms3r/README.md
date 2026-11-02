# AtomS3R Servo Link

Firmware for driving the J288/S288 servo from an M5Stack AtomS3R through the
[Unitree servo board](https://github.com/kim-xps12/m5stack_board_unitree_brushless_digital_servo):
a sine swing demo (`sin_swing`) and a bench firmware with analysis scripts (`bench`). A separate
firmware (`xbox`) prints the state of an Xbox controller connected over Bluetooth LE. In the bench,
the AtomS3R doubles as the measuring instrument: its RMT peripheral generates test patterns and
records pin waveforms with 12.5 ns resolution, so no oscilloscope is needed.

The work started on the [DYNAMIXEL TTL board](https://github.com/kim-xps12/m5stack_board_dynamixel_ttl_rs3485)
(MAX3485 with transistor auto-direction). Its measurements led to the redesign and are kept in
`logs/` and `tools/experiments/`. The firmware and scripts work with both boards; see
[DYNAMIXEL TTL Board](#dynamixel-ttl-board) for the differences.

The servo packets are built and parsed by the unmodified STM32 code in
`../external/digital_servo/stm32/firmware/App/`, Unitree's repository as a git submodule
(`compat/gpio.h` stands in for the STM32 HAL header it includes).

## Wiring

| AtomS3R | Unitree board | Note |
|---------|---------------|------|
| Grove: G2, G1, 5V, GND | J2: Tx, Rx, V, G | Straight Grove cable |
| G5 (bottom header) | J3 pin 3 (servo signal) | **Bus bench steps only** |
| G5 (bottom header) | TP1 (TXEN) | **`txen.py` only** |

The servo pulls its signal up to about 5 V (4.13 V at idle with an S288, measured on the board).
Connect G5 to J3 pin 3 only when no servo is plugged in **and** J1 power is unplugged; the
ESP32-S3 is not 5 V tolerant, and J3 pin 2 carries the servo supply. TP1 is 3.3 V logic, so the
servo and J1 power may stay connected for `txen.py`.

Set the servo to 4 Mbps with Unitree's tool first; the board supports 4 Mbps only.

## Usage

```bash
pio run -t upload                      # build and flash the bench firmware
# bench, G5 on J3 pin 3 (no servo, J1 unplugged)
python3 tools/bench.py check           # wiring sanity: bus visible on G5, RX released while transmitting
python3 tools/bench.py sweep           # driver enable delay vs. preceding idle
python3 tools/bench.py packet          # real packets at 4 / 6 Mbps on the bus
# bench, G5 on TP1
python3 tools/txen.py all              # TXEN hold time, release in gaps, packets (see below)
# servo connected and powered, G5 unplugged
python3 tools/bench.py scan            # find the ID
python3 tools/bench.py servo --id 0    # error statistics (-n 100000, --pattern stop / random)
python3 tools/bench.py margin --id 0   # baud-rate tolerance window
python3 tools/bench.py latency --id 0  # transmit release and response timing
python3 tools/first_edge.py all        # tolerance to the first-edge delay, window with it cancelled
python3 tools/official.py margin       # the same window through Unitree's USB adapter, for comparison
```

Default baud rate is 4 Mbps (`--baud` to change). Raw logs go to `logs/`.
Findings, the board redesign and the prototype results: [docs/ttl_board_redesign.md](docs/ttl_board_redesign.md)
(§7.1 for the Unitree board); explanation of the problem, the fix options and other vendors' circuits:
[docs/s288_link_explained.md](docs/s288_link_explained.md) (both Japanese).
`--pattern random` sends varying command data. On the DYNAMIXEL board it gave a narrower window than
the stop commands (`--pattern stop`, mostly zero bytes); on the Unitree board both give the same window.

## TXEN Timing (`txen.py`)

The board switches the bus to transmit by itself: TXEN (U1 output, on TP1) goes high on the first
falling edge of TX and drops a few microseconds after the last rising edge. `txen.py` drives TX
with the bench firmware and records TXEN on G5.

| Step | Pattern on TX | Reports |
|------|---------------|---------|
| `hold` | One low pulse of 250 ns, 1 µs or 9 µs | Hold after the last rising edge (TXEN high time − pulse) |
| `gap` | Low 250 ns, high G, low 250 ns; G = 2–10 µs in 0.2 µs steps | The shortest gap where TXEN drops in between |
| `packet` | Real packets: a stop command and a stress packet with `0xFF` bytes | Whether TXEN stays high through the packet (one high run), and the hold after it |
| `all` | The three above in order | |

`--reps` sets the captures per point (16), `--bauds` the packet baud rates (4000000).
The design range of the hold is 3.27–7.30 µs (typical 5.2 µs). The delay from the TX falling edge
to TXEN rising (a few ns) is not visible here, so the hold reads a few ns short.
Measured on the prototype (2026-10-21): hold 5.49–5.85 µs after single pulses and 5.39–5.58 µs
after packets, released in gaps of 5.6–6.0 µs, and one TXEN high run in 16/16 packets.

## Sine Swing Demo

`src/sin_swing/` is a port of `python/sin_swing_demo.py`: the same defaults, centre capture,
amplitude ramp, speed feed-forward, `timeout=1` and `mode=0` on exit.

```bash
pio run -e sin_swing -t upload
```

Control it over the USB serial port, or with the front button (start with the last
parameters / stop):

```
start [amp=0.5] [freq=0.5] [kp=10] [kd=0.3] [duration=10] [ramp=1] [rate=200] [print=10] [id=0] [baud=4000000]
stop
status        # one mode=0 exchange, the servo does not move
clear         # clear the servo's error (MError bits that a clear can reset, e.g. overvoltage)
```

`clear` はサーボのエラー消去で，公式ツールの「清除故障」と同じ指令 (mode 6 のトルク欄に -256) を送ります．指令の形は公式ツールの解析 (未公開) によります．ID 0 では `FE EE 60 00 00 FF 00 00 00 00 00 00 00 00 00 00 43 FB E0 1E` です．公式の `protocol.c` は mode を 0〜1 に切り詰めるので，組み立てた後に欄を書き換えて CRC を付け直しています．過電圧 (`MError=4`) は `clear` で消えました (実測 2026-11-02)．運転の終わりには，運転中に読んだ電圧の最小と最大，`MError` が最初に出た時刻とそのときの電圧も表示します．

`clear` sends the servo's error-clear command, the same as "清除故障" in the official tool (mode 6 with -256 in the torque field). The packet comes from an analysis of the official tool (not published); for ID 0 it is `FE EE 60 00 00 FF 00 00 00 00 00 00 00 00 00 00 43 FB E0 1E`. The official `protocol.c` clamps mode to 0–1, so the fields are rewritten after packing and the CRC is recomputed. An overvoltage (`MError=4`) was cleared this way (measured 2026-11-02). At the end of a run the firmware also prints the lowest and highest voltage read during the run, and the time and voltage of the first `MError`.

On the Unitree board, both an S288 and a J288 swung at exactly 4 Mbps with no missed reply in
14,000 cycles each (10 s + 60 s at 200 Hz). A missed command is not fatal: the servo keeps the
previous target until the next one.

## Xbox Controller

`src/xbox/` connects an Xbox Wireless Controller (Model 1914, the Xbox Series X|S controller)
over Bluetooth LE and prints every button, stick and trigger on the USB serial port (115200 bps).
The ESP32-S3 has no Bluetooth Classic, so the controller must speak BLE; Model 1914 does.
The connection and the report parsing come from
[XboxSeriesXControllerESP32_asukiaaa](https://github.com/asukiaaa/arduino-XboxSeriesXControllerESP32)
(on NimBLE-Arduino); `platformio.ini` pins it and its dependencies. No wiring is needed.

```bash
pio run -e xbox -t upload
pio device monitor
```

Hold the pair button on the top of the controller until the Xbox button blinks fast. The firmware
connects to the first Xbox controller it finds and prints one line whenever the state changes
(at most every 50 ms):

```
LX=32768 LY=32768 RX=32768 RY=32768 LT=   0 RT=   0 | A=0 B=0 X=0 Y=0 LB=0 RB=0 LS=0 RS=0 View=0 Menu=0 Xbox=0 Share=0 | Up=0 Down=0 Left=0 Right=0 | bat=0%
```

| Field | Range |
|-------|-------|
| `LX` `LY` `RX` `RY` (sticks) | 0–65535, about 32768 at rest |
| `LT` `RT` (triggers) | 0–1023 |
| Buttons, `LS` `RS` (stick clicks), D-pad | 1 while pressed |
| `bat` | Battery level in percent |

While no controller is connected, `xbox: scanning` is printed every 3 s. After three failed
connection attempts in a row the AtomS3R restarts and scans again.

To see the reports as received, build with `PLATFORMIO_BUILD_FLAGS=-DXBOX_RAW_DUMP pio run -e xbox -t upload`:
each BLE notification is printed in hex instead of the parsed state. In the 16-byte report, bytes 0–11
are the sticks and triggers (16-bit little endian), byte 12 the D-pad (0 = released, 1–8 clockwise
from up), byte 13 A `01` B `02` X `08` Y `10` LB `40` RB `80`, byte 14 View `04` Menu `08` Xbox `10`
LS `20` RS `40`, and byte 15 Share `01` (checked on a Model 1914 on 2026-10-25).

## DYNAMIXEL TTL Board

| AtomS3R | DYNAMIXEL TTL board | Note |
|---------|---------------------|------|
| Grove: G2, G1, 5V, GND | J2: TX, RX, +5V, GND | Straight Grove cable (TX lands on G2; checked on hardware) |
| G5 (bottom header) | J4 pin 3 (bus signal, MAX3485 A) | **Bench steps only** |

The same caution applies: connect G5 only with the servo and J1 power unplugged (J4 pin 2
carries the servo supply).

At exactly 4 Mbps this board misses about half the replies while swinging, so start the sine swing
with `baud=3950617` (80 MHz / (20 + 4/16)). Measured missed replies (A = 0.5 rad, 200 Hz):

| Our baud rate | vs. 4 Mbps | Missed |
|---------------|-----------|--------|
| 4 000 000 | 0 % | ~51 % |
| 3 968 992 | −0.62 % | 30 % |
| 3 956 723 | −0.93 % | 1.7 % |
| 3 950 617 | −1.23 % | 0.08–0.4 % |
| 3 938 461 | −1.54 % | 44 % |

The window is narrow, so treat the offset as a workaround and re-check it on another servo.
