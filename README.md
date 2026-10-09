# Tube VSR

Desktop software for the **ESP32 + BME690** VOC sensor. It runs the measurement, saves every reading to a local database, and helps compare treatments using an odor fingerprint from the metal-oxide gas sensor. It uses the same gas measurement as the Plant Stress Analyzer, without the spectral sensor.

> **Download:** get the latest installer from [Releases](../../releases/latest).
> Use `TubeVSR_<version>_x64-setup.exe` (recommended) or the `.msi`. Windows 10/11, 64-bit.

---

## Features

- **Data acquisition**: start and stop measurements, with live charts of every gas reading and the odor fingerprint.
- **Gas resistance timeline**: every timed gas read of a measurement (conditioning and channel scan), live and in the Visualization.
- **Odor signal share**: the total odor signal (Σ log₁₀ R_blank / R_sample over all channels, an index of VOC emission) and each channel's share of it, live and in the Visualization. The total is also available as a single-value bar chart.
- **Experiments and factors**: organize samples by experiment, factor (e.g. *Treatment*) and group (e.g. *Control*). Experiments can be password protected.
- **Visualization**:
  - add charts for single samples or a group average
  - **Mean ± SD**, **Difference (A − B)** and **Group** boxes, with a colour you pick for each series
  - **PCA** boxes: add sample groups one by one and see them on PC1 / PC2, with a colour for each group and a 95 % ellipse
  - single-value bar charts
  - PNG export of any chart
- **Data Management**: a spreadsheet-like table of every measurement. Sort and search it, change a sample's group in its cell or for many selected samples at once, edit notes, delete wrong samples (a database backup is saved first) and export CSV.
- **Firmware check and update**: when the ESP32 is plugged in, the app reads its firmware version. If it differs from the firmware bundled with the app, one click flashes the matching firmware. No Arduino IDE is needed.
- **Automatic app updates**: the app checks this repository for new versions and can download, install and restart by itself. Updates are signed.

## Getting started

1. Install the app from [Releases](../../releases/latest). The first install is manual; later versions arrive as in-app updates.
2. Plug in the ESP32 with a USB cable. In the **Connect the ESP32** dialog, pick its serial port (baud rate 115200).
3. If the firmware status shows **Update** or **Install**, click it and wait until it reports *up to date*. Keep the cable plugged in while it runs.
4. Choose or create an experiment. Add factors and groups under **Factors/Treatments**.
5. With no sample in the setup, press **Run blank** on **Data Acquisition**. This is needed after every app start and every hour.
6. Put the sample in, select its group, set the number of cycles and press **Start**.

If the USB cable is pulled during a measurement, the app stops it, keeps the cycles that had finished and tells you; plug the ESP32 back in and start again.

The ESP32 warms up its gas sensor for about 90 s after every power-on or reset. The firmware check also resets the board, so a measurement started straight after plugging in begins once the warm-up is done.

## Hardware

| Part | Connection |
|---|---|
| ESP32 dev board (`esp32:esp32:esp32`) | USB to the PC |
| Bosch BME690 (gas, temperature, humidity, pressure) | I²C bus 0: SDA 21, SCL 22, address 0x76 |
| Status LED (optional) | GPIO 4 |

## Measurement protocol (v17)

Each cycle runs these steps in order:

1. **Gas sensor conditioning**: 40 °C (1 s) ↔ 400 °C (1 s), repeated until the 40 °C reading is stable (10–60 cycles).
2. **Gas channel scan**: 13 heater targets from 40 °C to 400 °C in 30 °C steps. Each channel runs:
   - **PRE**: 40 °C for 1 s
   - **TARGET**: a quick read at about 0 s (T0), then reads at 1, 2 and 3 s (T1–T3)
   - **POST**: 400 °C for 1 s

The **odor fingerprint** is log₁₀(R_blank / R_sample) of the T3 reading for each channel. Values above 0 mean the sample lowers the resistance. The **total odor signal** is the sum of the fingerprint over all channels (negative channels count as 0); each channel's **share** is its part of that total, which describes the odor pattern independently of how strong the emission is.

### Blank

A **blank** is one measurement cycle with no sample in the setup. It is the reference for the fingerprints of the samples measured after it. The app asks for a new blank after every app start and once the blank is an hour old. In the Visualization, blanks appear as the group *Blank*.

When a sample measurement finishes, the app asks you to take the sample out, and then **cleans the sensor**: rounds of 400 °C heating, each followed by a check of the 40, 70 and 100 °C channels. Rounds repeat until 40 and 70 °C read the 102.4 MΩ ceiling and 100 °C reads the ceiling or the same as the round before (within 3 %); the next blank or sample can start only after that.

Each new blank is checked:

- **Low temperature:** 40 and 70 °C must read the 102.4 MΩ ceiling; 100 °C the ceiling, or the same as the blank before (within 3 %). Until then, the app discards the blank and runs another one by itself (press **Stop** to end).
- **Previous blanks:** a blank whose 40–100 °C reads differ by more than 10 % from both of the last two accepted blanks is run again; two blanks in a row that agree (within 3 %) become the new baseline.
- **Recent blanks:** at 70–250 °C it should not read about 20 % or more below the last five accepted blanks; otherwise the app offers to discard it and run it again.

Each analyzer reports its own ID (the ESP32's MAC address), and blanks are kept per analyzer: a blank only serves samples measured on the same analyzer. Plugging in another analyzer asks for a new blank.

While the ESP32 is plugged in and idle, its heater keeps pulsing (**keep-warm**), so the sensor does not cool down and drift between measurements. A new sensor needs some hours of heating before its readings settle; leave a new analyzer plugged in before the first experiment.

## Data

- All readings are stored in a SQLite database at
  `%LOCALAPPDATA%\CMU BME690 Research\Tube VSR\bme690_data.sqlite`.
  The database is kept when the app is updated or reinstalled, and is separate from the Plant Stress Analyzer's.
- Experiments can be exported to CSV on the **Data Management** page. The default export has one row per sample measurement: its factor groups and the odor fingerprint log₁₀(R_blank / R_sample) at each heater temperature. A detailed export of all measurement features is also available.

## Version numbers

The app and the ESP32 firmware always share the same version number. Each app release bundles its matching firmware, so updating the app and then clicking **Update** on the firmware status keeps both in step.

## Credits

Copyright © 2026 Assistant Professor Dr. Chun-I Chiu,
Department of Entomology and Plant Pathology, Faculty of Agriculture, Chiang Mai University.

The installer includes [esptool](https://github.com/espressif/esptool) by Espressif Systems (GPL-2.0), used unmodified to flash the ESP32 firmware. Its licence is installed alongside it (`firmware/esptool-LICENSE.txt`).

## Development

The repository holds the desktop app; the ESP32 firmware is `../BME690_Basic.ino` (one folder up, next to the Bosch BME69X driver files).

| Command | What it does |
|---|---|
| `npm install` | Install the frontend and Tauri CLI dependencies. |
| `npm run tauri dev` | Run the desktop app with serial-port access (development database in the `BME690_Basic` folder). |
| `npm run build:firmware` | Compile the firmware with `arduino-cli` and bundle `firmware.bin` and `esptool.exe` into `src-tauri/resources/firmware/`. Runs before every app build. |
| `npm run release -- "notes"` | Build the signed installers and write `release/v<version>/` (setup.exe, its .sig, .msi, `latest.json`) for a GitHub release. |
| `cargo test` (in `src-tauri/`) | Run the backend tests. |

Requirements: Node.js, Rust, the Arduino IDE (its `arduino-cli`) with the `esp32:esp32` core, and the updater signing key (`~/.tauri/plant-stress-analyzer.key`, or `TAURI_SIGNING_PRIVATE_KEY`).

**Releasing a version**

1. Set the same version in `package.json`, `src-tauri/tauri.conf.json`, `src-tauri/Cargo.toml`, `APP_VERSION` in `src/App.tsx`, and `FIRMWARE_VERSION` and the header of `BME690_Basic.ino`. The firmware build stops if the firmware and app versions differ.
2. Run `npm run release -- "What changed"`.
3. Create a GitHub release tagged `v<version>`, upload the four files from `release/v<version>/` and mark it as the latest release. Installed apps pick it up from `releases/latest/download/latest.json`.

Keep the signing key safe: without it no further update can reach installed apps.

**Layout**

| Path | Contents |
|---|---|
| `src/App.tsx`, `src/App.css` | React frontend (all pages). |
| `src-tauri/src/lib.rs` | Rust backend: serial logging, SQLite, blank and cleaning checks, exports, firmware flashing. |
| `src-tauri/tauri.conf.json` | App name, identifier, bundle and updater settings. |
| `scripts/` | Firmware build and release scripts. |
| `public/` | Logo and the optional setup photos (`blank-*.jpeg`, `sample-*.jpeg`) shown in the blank and sample dialogs. |

Tube VSR is derived from the Plant Stress Analyzer and shares its measurement protocol (v17); the spectral (VNIR) parts are switched off (`HAS_VNIR = false` in `lib.rs`).
