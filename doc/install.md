# Install, update, uninstall

FlyLuaIoSc is a standalone Windows application for MSFS 2020 / 2024. It is not a Community-folder package
and it is not a sim plugin: it runs next to MSFS, talks to it over SimConnect, and drives your panels.

## Requirements

* Windows, 64-bit.
* Microsoft Flight Simulator 2020 or 2024 — MS Store or Steam, either or both.
* At least one supported panel (see the list in the [README](../README.md#gamehardware-compatibility-list)),
  or a MobiFlight board.
* Internet access once, for activation.

## Install

### 1. Exit MSFS

The installer checks for `FlightSimulator.exe` / `FlightSimulator2024.exe` and stops with
*"MSFS is Running, Exit MSFS at first."* if either is up. It also kills a running FlyLuaIoSc, so nothing
is left holding the device handles.

### 2. Run the installer

Download `flyluaiosc_<version>_msfs_installer.zip` from
[the releases page](https://gitee.com/FlyLuaIo/flyluaiosc/releases), unzip it, run the `.exe` inside.

The install location is fixed: `%LOCALAPPDATA%\cpuwolf\flyluaiosc`. It is per-user, so no administrator
rights are requested and nothing lands in `Program Files`. The directory page in the wizard is read-only.

### 3. Confirm the Community folders

The wizard scans for all four possible MSFS installs and shows what it found:

| Field | Sim |
|---|---|
| 2020 Store | MSFS 2020 bought in the Microsoft Store |
| 2020 Steam | MSFS 2020 from Steam |
| 2024 Store | MSFS 2024, Microsoft Store |
| 2024 Steam | MSFS 2024, Steam |

Empty fields are sims you don't have — they are disabled. Check the paths: that is where the
`mobiflight-event-module` package is copied, and it has to be the real `Community` folder of each sim.

### 4. Choose components

Only **FlyLuaIoSc** (the core) is mandatory. The rest are optional helpers:

| Component | What it does |
|---|---|
| FlyLuaIoSc | The application itself. Required. |
| Start Menu Shortcut | Adds *FlyLuaIoSC* to the Start Menu. |
| FlyLuaIoSc Auto Start | Shortcut in the Startup folder, so it starts when Windows logs in. |
| FlyLuaIoSc crash dump | Registers a Windows Error Reporting local dump. If FlyLuaIoSc ever crashes, a dump is written that you can send us. Costs nothing at runtime. |
| Reset FlyLuaIoSc configuration | Deletes `%APPDATA%\flyluaio.cfg` on install — a clean-slate install (window position, accepted license, menu state). |
| QCDU MCDU Asobo | Installs the community `z-dementedmonkey-cj4-mcdu` package so the Asobo CJ4 MCDU can feed the QCDU screen. Only relevant if you fly the Asobo CJ4. |

### 5. Finish

Two things happen at the end, automatically:

* **Start with the sim.** FlyLuaIoSc is added to the `exe.xml` of every MSFS found above, so MSFS launches
  it together with the flight — no manual start. The uninstaller removes those entries again.
* **MobiFlight event module.** The `mobiflight-event-module` WASM package is copied into each Community
  folder. MSFS loads it at boot; without it MobiFlight boards cannot be driven.

The wizard then starts FlyLuaIoSc once to register/activate it and leaves it running in the tray.

## First run and activation

FlyLuaIoSc has no main window. It shows a tray icon; right-click it and pick *Settings* to get the window
with five tabs:

| Tab | What it is for |
|---|---|
| Activation | License state. *Register & Activate* does the online activation, or point a browser at the shown URL. Not activated = devices are not driven. |
| Devices | Every supported panel that is connected, ticked = FlyLuaIoSc drives it. Untick a panel to hand it back to the vendor's own software (e.g. WinWing SimApp Pro). Takes effect after a sim restart. |
| Scripting | *Open Scripts* (opens the script folder), *Reload Scripts*, *Enable debug*, *Log Window*, *Update Scripts Patch*, *Upload Log*. |
| Keys | Find and reset the key assignment of one panel: step 1 find the key, step 2 press it, step 3 reset the assignment. Use it when a panel's buttons collide with another device's. |
| About | Version, feature list, and who to blame. |

If the first run reports **"no supported USB device is connected"**, the install is still complete — plug the
panel in and start the sim; activation can be finished at any time from the **Activation** tab.

**Activation privacy.** Activation sends the device UUID, the computer name and the list of connected USB
devices. That data is used for license verification only — the same sentence is printed in the Activation
tab, so it is not buried here. Nothing else is collected silently: the only other network traffic is the
update check (turn it off with *Disable Auto Update*) and a log upload you trigger yourself
(see [support.md](support.md)).

## Where things live

| Path | Content |
|---|---|
| `%LOCALAPPDATA%\cpuwolf\flyluaiosc\` | FlyLuaIoSc itself — `flyluaiosc.exe`, `SimConnect.dll`, `msfstool.exe`, the Lua runtime files |
| `%LOCALAPPDATA%\cpuwolf\flyluaiosc\data\` | Aircraft profiles. Shipped ones are `.sec`; your own `.lua` profiles go here too. |
| `%LOCALAPPDATA%\cpuwolf\flyluaiosc\joysticks\` | USB HID device definitions (`*.json`) |
| `%LOCALAPPDATA%\cpuwolf\flyluaiosc\mobiflight\` | MobiFlight board definitions (`*.json`) |
| `%LOCALAPPDATA%\cpuwolf\flyluaiosc\flyluaio.log` | The log. Rotated — `qmdev.1.log` is the previous one. |
| `%LOCALAPPDATA%\cpuwolf\flyluaioscinstaller\` | Installer/agent logs |
| `%APPDATA%\flyluaio.cfg` | Settings database (SQLite): window position, accepted license, menu state |
| `<Community>\mobiflight-event-module\` | The WASM module, one copy per sim |

*Scripting → Open Scripts* opens `data\` for you, so nobody has to type `%LOCALAPPDATA%` by hand.

## Updating

**Scripts only** (aircraft profiles, device JSON) — *Scripting* → **Update Scripts Patch**. It compares the
files you have against the server's list and downloads only what changed. No installer, no restart of the
sim required for the next flight. This is the normal way to pick up a newly supported aircraft.

**Everything else** — run the installer of the new version. It detects the previous install (including the
very old NSIS-era one under `%LOCALAPPDATA%\cpuwolf\qmdev_sc`, if present), uninstalls it, and installs
fresh. Your own scripts and device JSON files under `data\`, `joysticks\` and `mobiflight\` are yours —
keep a copy if you edited them, because a clean install replaces the shipped files.

## Uninstall

`Uninstaller.exe` in the install folder, or *Windows Settings → Apps*. The uninstaller removes the
`exe.xml` entries first, so MSFS does not keep looking for a binary that is gone. `%APPDATA%\flyluaio.cfg`
is left behind unless you installed with the *Reset FlyLuaIoSc configuration* component; delete it
manually if you want a truly clean slate.

## Troubleshooting

| Symptom | Cause and fix |
|---|---|
| Installer stops with *"MSFS is Running"* | Exit MSFS completely (including the tray/background process), then run it again. |
| Panels are detected but nothing moves in the sim | The panel is unticked in **Devices**, or the vendor's own software has claimed it. Untick it here *or* close the vendor app — not both fighting. |
| A panel is missing from **Devices** | Not in [the supported list](../README.md#gamehardware-compatibility-list), or its USB interface is claimed by another driver. Check with the vendor software closed. |
| MobiFlight board does not appear | The board's serial must match the definition in `mobiflight\*.json`; check that `mobiflight-event-module` is in the Community folder of the sim you are actually running, and that the board is not still open in MobiFlight Connector. |
| QCDU buttons work, screen stays blank | By design — the screen is driven by [MCDUHub](https://x-plane.vip/quickmade/support/). |
| Nothing works and you don't know why | *Scripting* → **Upload Log**. See [support.md](support.md). |

## See also

[custom-device.md](custom-device.md) — add your own USB HID or MobiFlight device.
[support.md](support.md) — logs, script patches, contact.
