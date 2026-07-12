# VTAC Protocol Analysis Documentation

This document logs the known and discovered BLE commands for the VTAC BMS protocol. The protocol is an implementation of the widely-used PACE BMS Modbus standard over BLE (using the Nordic UART Service: `6E400001`, TX: `0002`, RX: `0003`).

*Note: All data payloads begin with the header `0xCD 0xCD`, followed by the Battery ID (`0x01` to `0x0F`), followed by padding, and then the command code at Byte index 7 (`f[7]`). The length of the payload is specified at Byte index 9 (`f[9]`).*

---

## Command `0x01`: System Parallel Info (Fully Implemented)
**Description:** Master-level broadcast summarizing the entire paralleled battery bank.
**Status:** ✅ Fully Implemented.
**Bytes Used:**
- `10-11`: Total Voltage (`/ 100.0` V)
- `12`: Total SOC (%)
- `13-15`: Total Current (Offset by -30000. `(val - 30000) / 100.0` A)
- `16-18`: Remaining Capacity (`surplusSoc`, Ah)
- `19-21`: Max Charge Current (`minCapacitance`)
- `22-24`: Max Discharge Current (`maxCapacitance`)
- `25`: Battery Status
- `26-29`: Primary Alarms Bitmask
- `30-33`: Secondary Alarms Bitmask
- `34`: Parallel Count
- `36-37`: Max Temperature
- `38-39`: Min Temperature
- `40-41`: Highest Pack Voltage
- `42-43`: Lowest Pack Voltage
- `44-46`: Total Charge Energy
- `47-49`: Total Discharge Energy
- `50`: State of Health (SOH)
- `51-53`: Total Capacity
- `54`: Battery Type

---

## Command `0x02`: Basic Info (Single Battery)
**Description:** Essential data broadcast for an individual battery pack. Early attempts to reverse engineer this via the decompiled JavaScript `app-service.js` yielded incorrect offsets. The correct offsets were mapped empirically via raw Hex Dumps.
**Status:** ✅ Fully Implemented.

**Corrected Byte Offsets:**
- `10-11`: Pack Voltage (`/ 100.0` V)
- `12`: SOC (%)
- `13-14`: Pack Current (`(val - 30000) / 100.0` A). *Positive = Charge, Negative = Discharge.*
- `17-18`: Maximum Cell Voltage (`/ 1000.0` V)
- `20-21`: Minimum Cell Voltage (`/ 1000.0` V)
- `23-24`: Maximum Temperature (`(Byte23 - 40) + (Byte24 * 0.1)` °C)
- `26-27`: Minimum Temperature (`(Byte26 - 40) + (Byte27 * 0.1)` °C)
- `31-34`: Primary Alarms (32-bit mask, e.g. `Low Total Voltage`, `Cell Fault`)
- `35-38`: Secondary Alarms (32-bit mask)
- `48`: MOS Status Bitmask (`0x01` = Charge Open, `0x02` = Discharge Open, `0x20` = Heating Open)
- `49-50`: Firmware / Software Version (e.g. Hex `1.5`)
- `71-72`: Remaining Capacity (`/ 100.0` Ah)
- `75-76`: Total Design Capacity (`/ 100.0` Ah)

---

## Command `0x03`: Cell Voltages (Single Battery)
**Description:** Arrays of individual cell voltages for a single battery.
**Status:** ✅ Fully Implemented.
**Bytes Used:**
- `11-40`: Sequentially parses up to 15 Cell Voltages. Each cell is 2 bytes, represented in millivolts (`/ 1000.0` V).
  - Cell 1: `11-12`
  - Cell 2: `13-14`
  - ...
  - Cell 15: `39-40`

*Derived Diagnostic:* The ESPHome integration intercepts these 15 cell voltages simultaneously to calculate exactly which cell index contains the maximum and minimum voltage, outputting a formatted text string to track cell drift.

---

## Command `0x04`: Cell Temperatures (Single Battery)
**Description:** Arrays of individual cell/sensor temperatures for a single battery.
**Status:** ✅ Fully Implemented.
**Bytes Used:**
- `11-26`: Parses up to 7 Temperature probes. Each probe is 2 bytes (Integer Part + Decimal Part offset).
- *Calculation:* `(Byte1 - 40) + (Byte2 * 0.1)` °C

---

## Command `0x09`: Single Battery Charge Energy
**Description:** Total Accumulated Charge Energy (Lifetime Ah) for a specific battery ID.
**Status:** ✅ Fully Implemented.
**Bytes Used:**
- `13-16`: 32-bit integer representing total charge capacity (`/ 100.0` Ah).
*Note:* The native VTAC App does not parse this for the UI, but it is fully supported by the BMS.

---

## Command `0x0A`: Single Battery Discharge Energy
**Description:** Total Accumulated Discharge Energy (Lifetime Ah) for a specific battery ID.
**Status:** ✅ Fully Implemented.
**Bytes Used:**
- `13-16`: 32-bit integer representing total discharge capacity (`/ 100.0` Ah).
*Derived Diagnostic:* This value is combined with the `Total Design Capacity` from Command `0x02` to dynamically calculate the precise **Estimated Cycle Count** of the battery.

---

## Unknown & Autonomous Commands
**Description:** Commands `0x07, 0x08, 0x0B, 0x0C, 0x0D, 0x0E, 0x11, 0x12, 0x14, 0x15, 0x17, 0x18, 0x19, 0x1A, 0x1B, 0x1C, 0x1D, 0x20, 0x31-0x47, 0xFD`
**Status:** ⚠️ Discovered but intentionally ignored.
**Details:** These commands are present in the decompiled JS. However, attempting to poll them sequentially via ESPHome causes the PACE BMS Bluetooth module to completely lock up and crash, requiring a hard reboot of the battery. 
*Workaround:* We have implemented a persistent background logger (`[VTAC_RAW]`) in ESPHome that silently listens for these commands. If the battery autonomously broadcasts one of them (e.g. during a critical fault event), it will be caught and dumped to the logs as a hex string for future analysis.
