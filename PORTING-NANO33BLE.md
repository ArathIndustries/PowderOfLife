# Nano 33 BLE Port Notes

This fork of [andrewfrueh/PowderOfLife](https://github.com/andrewfrueh/PowderOfLife) adds
support for the **Arduino Nano 33 BLE / Nano 33 BLE Sense Rev2** (nRF52840, Mbed-based core)
while remaining compatible with the classic **Arduino Nano** (ATmega328P, AVR core) used in
upstream. The Arduino library source is checked in at `Arduino/PowderOfLife/` (upstream ships
it only as a ZIP); `Arduino/PowderOfLife.zip` is rebuilt from that source.

## Why upstream did not work on the Nano 33 BLE

The two boards share a form factor but not an architecture:

| | Classic Nano (upstream target) | Nano 33 BLE Sense Rev2 |
|---|---|---|
| MCU | ATmega328P, 8-bit AVR, 16 MHz | nRF52840, 32-bit ARM Cortex-M4, 64 MHz |
| Arduino core | `arduino:avr` | `arduino:mbed_nano` (Mbed OS) |
| Logic level | 5 V | **3.3 V, pins are NOT 5 V tolerant** |
| `int` width | 16-bit | 32-bit |
| USB serial | Hardware UART via USB bridge | Native USB CDC |

The library itself is portable Arduino C++ (no AVR registers, no PROGMEM, no
SoftwareSerial), but three latent assumptions broke or degraded on 32-bit ARM.

## Code changes in this fork

1. **`MPUSensor.h` — sign-width bug on 32-bit cores (functional bug).**
   MPU-6050 registers are signed 16-bit. Upstream stored `Wire.read() << 8 | Wire.read()`
   into `int`; on AVR the 16-bit `int` reproduced the sign bit, on ARM the 32-bit `int`
   turned every negative reading into a large positive (e.g. −1 → 65535). The buffer is now
   `int16_t`, and high/low bytes are read in separate statements because C++ does not define
   evaluation order inside that expression.

2. **`AnalogSensor.h` / `Engine.h` — analog pin indexing.**
   Upstream passed bare channel numbers (`setPin(0)`) to `analogRead()` and hardcoded
   `pin + 14` for pin-mode calls. Both are AVR conventions. `setPin(n)` for n < 14 now maps
   to `A0 + n`, which is identical on AVR (A0 = 14 on Nano/Uno) and correct on Mbed cores.

3. **`SerialNode.h` — baud-rate type.**
   `unsigned int` is 16-bit on AVR, so rates above 65535 (e.g. 115200) silently truncated.
   Now `unsigned long` on both architectures.

4. **`library.properties`** — version 0.2.0, fork metadata. `architectures=*` retained.

Not changed: the serial message protocol (`<channel:value>` text frames), the neuron/engine
model, all examples, and the Unity package. The Unity side needs no modification — it sees
the same COM-port text protocol either way.

## Verification status

- Compile-verified with `arduino-cli` for `arduino:mbed_nano:nano33ble` **and**
  `arduino:avr:nano` (no AVR regression). See repo history for the exact run.
- Not yet verified on physical hardware. First hardware test should be
  `File > Examples > Powder Of Life > 01_Starting_Out > Analog_Sensor` with the Serial
  Monitor, then the two-pot digital-twin sketch
  (`03_Inter-Environment_Networks > Analog_Sensor_Lerped_Serial_Output_x2`).

## Hardware wiring for the digital-twin controller (two B10K pots)

**Rule: nothing above 3.3 V touches an nRF52840 pin.** The classic-Nano tutorial wires the
pots across 5 V; on this board that damages the MCU.

- Each pot: one outer leg → **3V3**, other outer leg → **GND**, wiper → **A0** (pot 1) / **A1** (pot 2).
- Reading range is unchanged (0–1023): the ADC reference scales with the supply, so a pot
  across 3V3 still sweeps the full range.
- Do not use the **VIN**/**5V** rail for sensors on this board.

## IDE / toolchain setup

1. Boards Manager → install **Arduino Mbed OS Nano Boards** (`arduino:mbed_nano`).
2. Tools > Board → **Arduino Nano 33 BLE**.
3. Sketch > Include Library > Add .ZIP Library → `Arduino/PowderOfLife.zip` from this fork
   (remove any previously installed upstream copy first).
4. USB serial behavior differs from the classic Nano: the port re-enumerates after upload,
   and opening the port does not reset the board. If an upload hangs, double-tap the reset
   button (onboard LED pulses) and select the new port that appears.

## License

Upstream is GPL v2 (code) / CERN OHL-W v2 (hardware) / CC BY-SA 4.0 (docs); this fork keeps
those licenses. Upstream does not accept pull requests by policy, so these changes live here.
