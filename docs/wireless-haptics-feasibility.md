# Feasibility study: wireless and haptic feedback

Status: research notes, not a design. Based on the R2.002 schematic (`hardware/electronics/schematic.pdf`), the RevA enclosure, and the shipped firmware images.

## 1. What the board has today

| Block | Part | Notes for wireless |
|---|---|---|
| MCU | RP2040 + W25Q16JV flash, 12 MHz crystal | No radio. Draws roughly 20–30 mA when active and has no real low-power BLE story. |
| Touch controller | Microchip ATMXT1066TD-NHU, I2C (SDA/SCL), CHG interrupt, RESET, I2CMODE | Can stay. It has configurable idle/active acquisition (T7 power config), and CHG can wake the MCU. |
| Haptics | ERM vibration motor, CSD16301Q2 low-side N-FET, 4 Ω series (quad 1 Ω), driven by `HapticMotorEN` | Runs directly from **V_USB**, so it stops working once USB is gone. |
| Power | USB-C (DX07S016JA1R1500) → AP7330 LDO (≈3.31 V) | Single source, with no battery, charger or power path. The comment notes the touch IC "wants a 300 mA LDO". |
| USB | IP4220 ESD, 22 Ω series, 5.1k-ish CC pull-downs | Can stay for charging and wired mode. |

Firmware: the shipped `.uf2` files contain no haptic-related strings. The motor hardware is populated, but nothing indicates the stock firmware does much with it. Check this against the QMK fork (`ploopyco/qmk_firmware`, branch `multitouch_experiment`).

Enclosure: the RevA assembly envelope is about 184 × 132 × 24 mm. That leaves a lot of area for a flat LiPo pouch, but the internal clearance under the PCB still needs checking in the STEP file.

## 2. Wireless: options

The hard requirement is that the trackpad keeps working as a **multitouch precision touchpad** (Windows PTP / HID digitizer) and not just as a boot mouse. That rules out off-the-shelf USB→Bluetooth HID bridges, which normally only pass mouse/keyboard reports and would lose gestures.

### Option A: add a BLE co-processor next to the RP2040 (lowest hardware risk, highest power)
- Add an nRF52840 (or nRF52833) module on a spare RP2040 UART/SPI. The RP2040 keeps running QMK and the maXTouch driver, and forwards HID reports to the module, which acts as a HID-over-GATT peripheral.
- Pros: the touch firmware stays as it is, and the layout change is small (the RP2040 has plenty of free GPIO).
- Cons: two MCUs to power and maintain. The RP2040's draw dominates the battery budget. Upstream QMK Bluetooth support is limited (Bluefruit LE SPI and RN42-era drivers), so the transport is custom work either way.

### Option B: replace the RP2040 with an nRF52840 (recommended for a real product)
- Use one nRF52840 module (e.g., a Raytac MDBT50Q-class module with antenna and certification). It provides native USB, so wired mode and the UF2 bootloader can stay, plus BLE and very low sleep current.
- The touch controller, I2C bus, CHG/RESET lines and haptics carry over unchanged.
- Cons: firmware port. QMK does not support nRF52 upstream. ZMK/Zephyr runs on nRF52 but, as far as I can tell, has no maXTouch driver, so that driver (object table parsing, T100 multitouch messages) and the PTP HID descriptor would need porting. This is the bulk of the effort.
- Layout: the module needs a keep-out at the board edge with no copper on any of the six layers under the antenna, and the enclosure must not put metal or tape (the 3M 467MP layer) over it.

### Option C: RP2040 + CYW43439 (the Pico W radio)
- Possible with BTstack, but it is power-hungry, has no QMK support and uses a large external RF chip. Not recommended.

### 2.4 GHz dongle instead of BLE
- Possible on nRF52 (ESB) with lower latency, but it needs a second device (the dongle) and its firmware. Consider it only if latency matters more than convenience.

## 3. Power budget (estimates, verify with measurements)

| Consumer | Active | Idle/sleep |
|---|---|---|
| maXTouch 1066 at a high report rate | tens of mA (largest unknown) | sub-mA in idle acquisition |
| RP2040 (option A) | ~20–30 mA | a few mA |
| nRF52840 BLE connected (option B) | ~3–6 mA average | µA range |
| Haptic clicks | 60–150 mA peaks, ~15 ms each | 0 |

With option B and aggressive idle settings (short active timeout, T7 idle scan, MCU sleep woken by CHG), a 1000–2000 mAh LiPo could plausibly last weeks of typical use. With option A, expect days. The touch controller's real current at the chosen scan rate is the number to measure first, using the existing board and a USB power meter.

### Required power hardware additions
- 1S LiPo plus a protection circuit (or a protected cell).
- A charger with power path from USB-C, e.g., BQ24075 or similar, so the board runs from USB while charging.
- Feed the 3.3 V LDO from the power-path output instead of V_USB. The AP7330 dropout at 300 mA needs checking against a LiPo near 3.4–3.5 V; a low-dropout LDO or buck-boost may be needed.
- Battery voltage sense: a divider into an ADC pin, gated by a FET so it doesn't drain the cell.
- A power switch, or a soft-latch so the board can be turned off.

## 4. Haptic feedback

### Running the current ERM on battery
- Move the motor supply from V_USB to the battery or power-path rail (3.0–4.2 V). Most coin ERMs are rated about 3 V, so the 4 Ω series resistor still limits current. The CSD16301Q2 turns on fully at 3.3 V gate drive.
- Keep firmware pulses short (10–30 ms). ERMs spin up slowly (tens of ms), so clicks feel "buzzy" rather than crisp.

### Recommended upgrade: LRA + DRV2605L
- The TI DRV2605L sits on the **existing I2C bus**. Its address is 0x5A; the maXTouch uses 0x4A/0x4B, so there is no conflict. Use its EN pin in place of `HapticMotorEN`.
- Pair it with a linear resonant actuator (LRA) rather than an ERM. You get crisp, fast "click" waveforms (closed-loop auto-resonance, a built-in effect library) and less energy per click.
- **QMK already supports the DRV2605L** (`HAPTIC_ENABLE`, `HAPTIC_DRIVER = drv2605l`), so even the wired build gains good haptics with little firmware work. Zephyr also has a DRV2605 driver for the option B path.
- Firmware hooks: tap-to-click, physical button press, gesture start/end, and optionally a "force click" from the maXTouch touch area/amplitude data.
- Mechanics: the LRA must be firmly coupled to the touch surface or PCB, not the base, or the feedback gets damped. That means checking the RevA mounting and the friction pads.
- Battery impact: about 0.1 J per click, so even thousands of clicks per day cost only a few mAh.

## 5. Suggested plan

1. **Measure** the current R2.002 board: total current at the shipped scan rate, and with maXTouch idle settings applied.
2. **Haptics first (cheap win):** bodge a DRV2605L + LRA breakout onto SDA/SCL/3V3 and enable QMK haptics. Tune the effects.
3. **Wireless prototype:** wire an nRF52840 dev board to the existing maXTouch (I2C, CHG, RESET) and port the driver plus the PTP descriptor to Zephyr/ZMK. Confirm Windows/macOS/Linux recognize it as a multitouch touchpad over BLE.
4. **New PCB revision:** nRF52840 module, charger with power path, battery connector, battery sense, DRV2605L + LRA, antenna keep-out. Update the enclosure for the battery and switch.

## 6. Risks / open questions
- maXTouch power-sequencing notes on the schematic: keep the RESET-after-power-stable behaviour on battery too.
- Whether BLE HID digitizer/PTP works reliably on every OS; macOS is the most uncertain.
- RF performance: a 6-layer board with large ground pours under a touch sensor, plus the adhesive layer, near the antenna.
- Firmware licensing: QMK is GPLv3 and ZMK is MIT, so a port mixing code needs care.
