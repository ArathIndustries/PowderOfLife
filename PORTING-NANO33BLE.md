# Nano 33 BLE Port Notes

> [!WARNING]
> **⚡ DO NOT WIRE ANYTHING TO 5 V. The Nano 33 BLE is a 3.3 V board and its pins are NOT 5 V tolerant.**
>
> The original video/tutorial wires the potentiometers across **5 V** because the classic Nano
> is a 5 V board. Doing the same on a Nano 33 BLE puts 5 V on an analog pin and can
> **permanently damage the nRF52840**. On this board:
>
> - Pot outer legs → **3V3** and **GND**. Wipers → **A0** / **A1**.
> - Never route **VIN** or the USB 5 V rail to anything that reaches a pin.
> - You lose nothing: readings are still 0–1023 because the ADC reference scales with the supply.

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

5. **Unity package: `SerialNode.cs` — DTR assertion (required for the Nano 33 BLE).**
   Upstream opens the port with only `new SerialPort(...)` + `Open()`. .NET defaults
   `DtrEnable` to false, and native-USB CDC boards (the Nano 33 BLE) do not transmit until
   the host asserts DTR — with the stock package the board enumerates, Unity opens the port,
   and no data ever arrives. This fork's `Unity/PowderOfLife.unitypackage` sets
   `serialPort.DtrEnable = true;` and `serialPort.RtsEnable = true;` before `Open()`. The
   classic Nano's USB-UART bridge streams regardless of DTR, so the change is safe for both
   boards (on a classic Nano, DTR-on-open also triggers the normal auto-reset, same as the
   Arduino Serial Monitor).

Not changed: the serial message protocol (`<channel:value>` text frames), the neuron/engine
model, and all examples. Protocol-compatible with upstream on both sides.

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

## Troubleshooting

### "library install failed … already installed, but with a different version: Powder Of Life@0.1"

Full error: `error: 13 Internal: library install failed: moving extracted archive to destination
dir: library Powder Of Life is already installed, but with a different version: Powder Of Life@0.1`

Cause: upstream's 0.1 library is already installed, and the Arduino IDE will not overwrite a
zip-installed library with a different version. It also cannot update it, because this library
is not in the Library Manager index. The old copy must be deleted by hand:

1. Close the Arduino IDE.
2. Open the sketchbook libraries folder:
   - Windows: `C:\Users\<name>\Documents\Arduino\libraries\`
   - macOS: `~/Documents/Arduino/libraries/`
   - Linux: `~/Arduino/libraries/`
   - Custom sketchbook location: File > Preferences > "Sketchbook location", then `libraries/`.
3. Delete the `PowderOfLife` folder.
4. Reopen the IDE, then Sketch > Include Library > Add .ZIP Library with this fork's zip.
5. Verify: `libraries/PowderOfLife/library.properties` now reads `version=0.2.0`.

### Unity: CS0234 "Ports does not exist in the namespace System.IO" and/or "UI does not exist in the namespace UnityEngine"

Both appear on importing the package into a fresh Unity project; they are project-settings
requirements, not bugs in the package (upstream has the same requirements).

1. `System.IO.Ports` missing (SerialNode.cs, SerialSensor.cs): Unity's default API level,
   .NET Standard 2.1, does not include `SerialPort`. Fix: Edit > Project Settings > Player >
   Other Settings > Configuration > **Api Compatibility Level = .NET Framework**
   (labeled ".NET 4.x" in Unity 2019/2020).
2. `UnityEngine.UI` / `Button` / `Text` / `CanvasScaler` missing (ButtonSensor.cs,
   TextDriver.cs, CanvasDriver.cs): the project lacks the Unity UI package. Fix: Window >
   Package Manager > Packages: Unity Registry > install **Unity UI** (`com.unity.ugui`).

Related console noise that resolves itself once the above compile errors clear: the
`Script updater ... failed to produce updates.txt` message. The
`Color primaries 0 is unknown or unsupported by WindowsMediaFoundation` warning concerns an
example-scene MP4 and is cosmetic (possible color shift in that video only).

## License

Upstream is GPL v2 (code) / CERN OHL-W v2 (hardware) / CC BY-SA 4.0 (docs); this fork keeps
those licenses. Upstream does not accept pull requests by policy, so these changes live here.
