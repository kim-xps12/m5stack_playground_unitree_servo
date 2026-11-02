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

## Top Speed of the J288 (J288 の最高速度)

±90° で往復させるサンプル (`swing90`) の振動数を決めるために測った記録です．`swing90` は，FB の表示と，読めているかを緑と赤で示す判定を付けて，独立したリポジトリ [m5stack_example_unitree_servo](https://github.com/kim-xps12/m5stack_example_unitree_servo) に移しました．このリポジトリにはコミットしていません (2026-11-02)．下の記録の `swing90` は，移す前の版 (最初の版は周期 4 s．どちらも FB の表示はなし) です．この節の測定は，すべて AtomS3 (R なし) で行いました．書き込みツールの表示に PSRAM が無いことで確かめました．

These are the measurements taken to choose the frequency of the ±90° swing sample (`swing90`). `swing90` moved, with an FB display and a green / red verdict on whether the feedback reads correctly, to the standalone repository [m5stack_example_unitree_servo](https://github.com/kim-xps12/m5stack_example_unitree_servo); it was never committed here (2026-11-02). `swing90` in the records below is the version before the move (4 s period in the first version; neither had the FB display). All measurements in this section used an AtomS3 (not the AtomS3R), identified by the absence of PSRAM in the flashing tool's chip report.

最初の版 (周期 4 s，2026-11-02，ID 0，サーボの電圧 7.5 V) の実測: 目標との差は振幅が上がる途中 (開始から 2 s) で最大 2.4° でした．開始から 17〜37 s の 101 行 (0.2 s ごと) では，位置は −89.9°〜+89.8°，目標との差は 0.34° 以内，トルクの絶対値は 0.04 N·m 以下でした．巻線温度は 37 s で 24 ℃ から 28 ℃ に上がりました．書き込み直後の起動では，動き始めるまでに 6〜8 s (2 回目)，12 s 以上 (1 回目) ID 0 の応答を待ちました．リセットでの起動では待たずに始まりました．書き込み直後だけ待つ原因は分かっていません．

Measured with the first version (4 s period, 2026-11-02, ID 0, 7.5 V at the servo): the tracking error peaked at 2.4° while the amplitude was rising (the first 2 s). In the 101 lines (every 0.2 s) from 17 s to 37 s after start, the position ran from −89.9° to +89.8°, the tracking error stayed within 0.34° and the absolute torque at or below 0.04 N·m. The winding temperature rose from 24 °C to 28 °C in 37 s. Right after flashing, the firmware waited for ID 0 to answer for 6–8 s (second flash) and more than 12 s (first flash) before starting; after a reset it started without waiting. Why it waits only after flashing is not known.

**最高速度 (電源 7.5 V)．** `sin_swing` で振幅を ±90° (Kp 10，Kd 0.3，500 Hz) に固定し，振動数を上げて追従を測りました (2026-11-02，ID 0 の J288，サーボの電圧 7.5 V)．目標の最大速度は 振幅 × 2π × 振動数，実速度は 0.02 s ごとの位置の差から求めた最大値，差とトルクは振幅が 90° に上がった後の最大値です．実速度は 10.5 rad/s で頭打ちになり，それより速い目標では差が増えるだけでした．カタログの無負荷最高速度 (25.2 V で 35 rad/s，12 V で 16.5 rad/s) を電圧に比例させると 7.5 V で約 10.4 rad/s で，実測と合います．トルクは最大 0.42 N·m で，最大トルク (1.5 N·m) に届く前に速度が頭打ちになるので，この速度は電圧で決まっています．そこで既定の振動数を，この速度にちょうど届く 1.1 Hz にしました．25 V の電源での結果は下の節にあります．

**Top speed (7.5 V supply).** With `sin_swing` at a fixed ±90° amplitude (Kp 10, Kd 0.3, 500 Hz), the frequency was raised step by step (2026-11-02, J288 at ID 0, 7.5 V at the servo). Target top speed is amplitude × 2π × frequency; actual speed is the largest position difference over 0.02 s; error and torque are maxima after the amplitude reached 90°. The actual speed levels off at 10.5 rad/s; faster targets only add error. Scaling the catalogue no-load speed (35 rad/s at 25.2 V, 16.5 rad/s at 12 V) to 7.5 V gives about 10.4 rad/s, which matches. Torque peaks at 0.42 N·m, below the 1.5 N·m maximum, so the voltage sets this limit. The default frequency is therefore 1.1 Hz, just reaching it. Results on a 25 V supply are in the section below.

| 振動数 / Frequency | 目標の最大速度 / Target speed | 実速度 / Actual speed | 差の最大 / Max error | トルクの最大 / Max torque |
|---|---|---|---|---|
| 0.50 Hz | 4.93 rad/s | 4.92 rad/s | 0.6° | 0.046 N·m |
| 0.75 Hz | 7.40 rad/s | 7.34 rad/s | 1.0° | 0.060 N·m |
| 0.90 Hz | 8.88 rad/s | 8.78 rad/s | 1.4° | 0.064 N·m |
| 1.00 Hz | 9.87 rad/s | 9.71 rad/s | 1.6° | 0.064 N·m |
| 1.10 Hz | 10.86 rad/s | 10.51 rad/s | 1.8° | 0.107 N·m |
| 1.20 Hz | 11.84 rad/s | 10.52 rad/s | 6.3° | 0.162 N·m |
| 1.30 Hz | 12.83 rad/s | 10.53 rad/s | 12.5° | 0.301 N·m |
| 1.40 Hz | 13.82 rad/s | 10.54 rad/s | 18.6° | 0.417 N·m |

1.1 Hz の `swing90` を 30 s 動かしたときの実測 (2026-11-02): 振幅が上がった後の位置は −88.1°〜+88.1° (0.2 s ごとの表示から．上の測定でも 1.1 Hz の振幅は ±88.2° で，速いほど 90° に少し届かなくなる)，目標との差は最大 1.8°，トルクの絶対値は最大 0.11 N·m，巻線温度は 33 ℃ から 38 ℃ に上がりました．このとき (`sin_swing` と `swing90` の書き込み) は，書き込み直後でも待たずに始まりました．

Measured with `swing90` at 1.1 Hz for 30 s (2026-11-02): after the amplitude rose, the position ran from −88.1° to +88.1° (from the 0.2 s status lines; the sweep above also gives ±88.2° at 1.1 Hz, the amplitude falls slightly short of 90° at speed), the tracking error peaked at 1.8°, the absolute torque at 0.11 N·m, and the winding temperature rose from 33 °C to 38 °C. This time (flashing `sin_swing`, then `swing90`) it started right after flashing without waiting.

**25 V の電源 (安定化電源)．** 同じ方法 (`sin_swing`，振幅 ±90°，Kp 10，Kd 0.3，500 Hz) で，電源を 25 V の安定化電源に替えて測りました (2026-11-02，ID 0 の J288，サーボの電圧 24.5 V)．1 回の運転は 7 s (振幅を 2 s で上げ，2 s で下げる) です．実速度は 0.01 s ごとの位置の差から求めた最大値，振幅と差とトルクは振幅が上がった後の値，電圧は運転中にサーボが報告した最大値 (0.5 V 刻み．2 ms ごとに読むので，その合間の瞬間的な山は見えない) です．電圧の記録は 4.0 Hz の 2 回目から始めました．

**25 V supply (bench power supply).** Same method (`sin_swing`, ±90° amplitude, Kp 10, Kd 0.3, 500 Hz) with the supply changed to a 25 V bench power supply (2026-11-02, J288 at ID 0, 24.5 V at the servo). Each run lasts 7 s (amplitude up over 2 s, down over 2 s). Actual speed is the largest position difference over 0.01 s; amplitude, error and torque are taken after the amplitude rose; voltage is the highest value the servo reported during the run (0.5 V steps, read every 2 ms, so short spikes in between are not seen). Voltage logging started with the second 4.0 Hz run.

| 振動数 / Frequency | 目標の最大速度 / Target speed | 実速度 / Actual speed | 振幅 / Amplitude | 差の最大 / Max error | トルクの最大 / Max torque | 電圧の最大 / Max voltage | 結果 / Result |
|---|---|---|---|---|---|---|---|
| 1.10 Hz | 10.86 rad/s | 10.70 rad/s | ±88.2° | 3.0° | 0.079 N·m | — | 正常 / OK |
| 1.50 Hz | 14.80 rad/s | 14.36 rad/s | ±87.1° | 3.8° | 0.123 N·m | — | 正常 / OK |
| 2.00 Hz | 19.74 rad/s | 18.83 rad/s | ±85.4° | 5.2° | 0.175 N·m | — | 正常 / OK |
| 2.50 Hz | 24.67 rad/s | 23.12 rad/s | ±84.0° | 7.0° | 0.243 N·m | — | 正常 / OK |
| 3.00 Hz | 29.61 rad/s | 27.34 rad/s | ±82.7° | 8.9° | 0.348 N·m | — | 正常 / OK |
| 3.50 Hz | 34.54 rad/s | 31.47 rad/s | ±81.9° | 11.1° | 0.457 N·m | — | 正常 / OK |
| 4.00 Hz (1 回目 / 1st) | 39.48 rad/s | 35.52 rad/s | ±81.2° | 13.3° | 0.592 N·m | — | 正常 / OK |
| 4.00 Hz (2 回目 / 2nd) | 39.48 rad/s | 35.18 rad/s | ±81.2° | 13.2° | 0.570 N·m | 27.0 V | 正常 / OK |
| 4.25 Hz | 41.95 rad/s | 36.60 rad/s | ±81.3° | 14.4° | 0.778 N·m | 27.5 V | 正常 / OK |
| 4.50 Hz (2 回 / twice) | 44.41 rad/s | — | — | — | — | 27.5 V | 過電圧で停止 / overvoltage stop |

- 4.0 Hz で実速度は 35 rad/s を超え，カタログの無負荷最高速度 (25.2 V で 35 rad/s) に届きました．速度はまだ伸びていましたが，伸び方は鈍っていました．4.25 Hz で 36.6 rad/s が，この電源で止まらずに動いた最高です．
- 4.5 Hz では 2 回とも `MError=4` (持続的な過電圧．公式ツールのエラー表による) で止まり，軸は +78° 付近で脱力しました．2 回目は開始から 1.82 s (振幅を上げている途中) に出て，そのとき報告された電圧は 27.5 V でした．運転中の電圧は，速いほど高くなっています (4.0 Hz で 27.0 V，4.25 Hz で 27.5 V)．止まる軸の回生 (モーターが発電して電源へ電気を戻すこと) で電圧が上がり，安定化電源はその電気を吸収できないためと考えています (推定)．エラーは `sin_swing` の `clear` で消えました．
- 熱: `swing90` の振動数を一時的に 4.0 Hz にして続けて動かすと，巻線温度が 40 ℃ から 16 s で 70 ℃ に達し，温度で自動停止しました (約 2 ℃/s で，落ち着く様子はありませんでした)．7 s の測定では見えなかった制約です．このため `swing90` は 25 V でも 1.1 Hz のままにしています．続けて動かせる最高の振動数は測っていません．

- At 4.0 Hz the actual speed passed 35 rad/s, the catalogue no-load top speed at 25.2 V. Speed was still rising, but more slowly. 36.6 rad/s at 4.25 Hz is the fastest run that completed on this supply.
- At 4.5 Hz both runs stopped with `MError=4` (sustained overvoltage, from the error table of the official tool), leaving the shaft limp near +78°. In the second run it appeared 1.82 s after start (while the amplitude was rising), with 27.5 V reported. The voltage during a run rises with speed (27.0 V at 4.0 Hz, 27.5 V at 4.25 Hz). Presumably regeneration while the shaft slows down (the motor acting as a generator and feeding current back) raises the voltage, which a bench power supply cannot absorb. The error was cleared with `clear` in `sin_swing`.
- Heat: with `swing90` temporarily set to 4.0 Hz and running continuously, the winding went from 40 °C to 70 °C in 16 s and the temperature stop triggered (about 2 °C/s, with no sign of levelling off). The 7 s runs did not show this limit. `swing90` therefore stays at 1.1 Hz on 25 V as well. The highest frequency that can run continuously was not measured.

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
