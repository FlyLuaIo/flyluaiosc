# Add your own device

FlyLuaIoSc recognises two kinds of hardware, and both are declared in JSON — no compiler, no vendor SDK:

| Kind | Folder | What it needs |
|---|---|---|
| USB HID panel | `%LOCALAPPDATA%\cpuwolf\flyluaiosc\joysticks\` | vendor/product ID + the HID report layout |
| MobiFlight serial board | `%LOCALAPPDATA%\cpuwolf\flyluaiosc\mobiflight\` | board name (or serial) + which channels carry buttons, encoders, displays |

Open the folder with *Settings → Scripting → **Open Scripts*** (it opens `data\`; go up one level for
`joysticks\` and `mobiflight\`), or paste `%LOCALAPPDATA%\cpuwolf\flyluaiosc` into Explorer.

Declaring the device is only half the job: it makes FlyLuaIoSc able to *talk* to the panel. What each
button and display does for a given aircraft is a **Lua profile**, covered in [Writing the aircraft
mapping](#writing-the-aircraft-mapping) below.

Reference material, all public:

* [joystick-config.schema.json](https://github.com/FlyLuaIo/flyluaioscripts/blob/master/joysticks/joystick-config.schema.json) — USB HID definition schema
* [mobiflight-config.schema.json](https://github.com/FlyLuaIo/flyluaioscripts/blob/master/mobiflight/mobiflight-config.schema.json) — MobiFlight definition schema
* [flyluaioscripts](https://github.com/FlyLuaIo/flyluaioscripts) — every definition and every aircraft profile that ships with FlyLuaIoSc, readable and forkable
* [LUA_API_DEVELOPER_GUIDE.md](https://github.com/FlyLuaIo/flyluaioscripts/blob/master/LUA_API_DEVELOPER_GUIDE.md) — the Lua side

## USB HID device

Only three fields are required:

| Field | Meaning |
|---|---|
| `ProductName` | Short device id, starts with a letter, no spaces or symbols (`WwAgp`, `WfDap500`). It becomes the name in the **Devices** tab and the prefix of the device's DataRef namespace — pick it once and keep it. |
| `USB_VID` | Vendor ID, hex string (`"0x06A3"`) or integer. |
| `USB_PID` | Product ID, same format. |

Everything else is optional and only needed if the panel does something a plain button box doesn't:

| Field | Meaning |
|---|---|
| `USBDelayMs` | Delay between HID writes. Raise it if a slow panel drops display updates. |
| `LightKey`, `LightKeyName` | HID button index used as the backlight/N1 test key, and its display name. |
| `USB_KEY_OFFSET`, `USB_KEY_MAX` | Byte offset and count of the button bitmap in the input report. |
| `USB_AXIS_OFFSET`, `USB_AXIS_MAX`, `USB_AXIS_NUM` | Axis block in the input report. |
| `USB_ENCODER_KEY_START`, `USB_ENCODER_OFFSET`, `USB_ENCODER_MAX`, `USB_ENCODER_NUM` | Encoder block in the input report. |
| `Functions` | Output report groups — one entry per LED/display/segment block. May be omitted for identity-only configs (a device you only read buttons from). |

A minimal definition is genuinely this short:

```json
{
  "$schema": "./joystick-config.schema.json",
  "ProductName": "MyPanel",
  "USB_VID": "0x1234",
  "USB_PID": "0x5678"
}
```

For a real one with LEDs, encoders and shift registers, read
[saitek_switch.json](https://github.com/FlyLuaIo/flyluaioscripts/blob/master/joysticks/saitek_switch.json) —
it is a shipped definition, not a teaching example, and it shows the `Functions` → `SubFunctions` /
`SubBitFunctions` structure that bit-encodes LEDs into output-report bytes.

Finding the VID/PID and report layout: on Windows, USB Device Tree Viewer or Wireshark's USBPcap will
show the descriptors and the input reports while you press buttons. `0x` prefixes are a convention here —
the schema accepts integers too.

## MobiFlight board

A MobiFlight definition is matched against the board that declares itself over serial. Only `ProductName`
is required; in practice you always set the matching fields:

| Field | Meaning |
|---|---|
| `ProductName` | Device id, letter first (`CfNano`, `KayeRoof`). |
| `ModuleName` | The module name the board reports (`"Rowsfire A107 V3"`, `"CPUWOLF MF NANO"`). |
| `ModuleSerial` | Serial (or a unique substring of it) used to match *this* board. Two identical boards are told apart by this. |
| `outputs` | Grouped by device type. |
| `inputs` | Grouped by device type. |

Channel groups in use by the shipped definitions:

| `outputs` group | Carries |
|---|---|
| `Output` | single LEDs / lamps, by channel name |
| `Display Module` | `LedModule` (segment and LCD displays) |
| `Stepper` | stepper-driven gauges |
| `ShiftRegister` | outputs behind a shift register |
| `CustomDevice` | anything private to your board |

| `inputs` group | Carries |
|---|---|
| `Button` | plain buttons |
| `Encoder` | rotary encoders |
| `AnalogInput` | pots and axes |
| `InputMultiplexer` | buttons behind a multiplexer chip |

The names inside a group (`"AP1"`, `"ANTICE_ENG1_UP"`, `"MAINPNL"`, …) are yours — they are the channel
names the Lua profile refers to, so name them the way the panel is silk-screened.

The smallest shipped definition, in full:

```json
{
  "$schema": "./mobiflight-config.schema.json",
  "ProductName": "CfNano",
  "ModuleName": "CPUWOLF MF NANO",
  "ModuleSerial": "SN-61D-31A",
  "outputs": {
    "Display Module": {
      "LedModule": [
        { "Name": "GA ENG RPM" }
      ]
    }
  }
}
```

For a full board — four multiplexers, dozens of button channels, a backlight channel — see
[rf_a107.json](https://github.com/FlyLuaIo/flyluaioscripts/blob/master/mobiflight/rf_a107.json) or
[MiniFCU.json](https://github.com/FlyLuaIo/flyluaioscripts/blob/master/mobiflight/MiniFCU.json).

MobiFlight's own Connector app must not have the board open at the same time — one process owns the COM
port. Close Connector after flashing firmware; FlyLuaIoSc talks to the board directly.

## Writing the aircraft mapping

Declaring the device makes it visible. Binding it to an aircraft is a Lua profile in `data\`:

```lua
local device = com.sim.qm.Qmcp737c.Open()
if not device then return end

device:CfgCmd(0, "sim/autopilot/heading_down")
GlobalFrameLoopManager:add(function() device:LoopMcp() end)
```

* Profiles are named `{DEVICE}_{AIRCRAFT}.lua` so it is obvious what they cover; the shipped MSFS profiles
  follow the same convention (`QMPE_Fnx.lua`, `QMOVH_A_PMDG737.lua`, …). A `z_{DEVICE}_GA.lua` is the
  general-aviation fallback, `z_{DEVICE}_no_msfs.lua` disables a device on MSFS.
* Save the file and it reloads — no sim restart. *Scripting → Enable debug* plus *Log Window* is the
  intended debugging loop.
* The API itself — configuration calls, DataRef/SimVar access, frame loops, device base classes — is
  documented in the [Lua API developer guide](https://github.com/FlyLuaIo/flyluaioscripts/blob/master/LUA_API_DEVELOPER_GUIDE.md).
  Copy the closest existing profile and change the bindings; that is how every shipped profile started.
* Shipped profiles are `.sec` files (they are the same Lua, packaged). Your own `.lua` files sit next to
  them and are not touched by updates.

If a mapping turns out to be generally useful, send it over — most of the aircraft coverage in FlyLuaIoSc
came in that way, and it keeps working across versions because [flyluaioscripts](https://github.com/FlyLuaIo/flyluaioscripts)
is re-shipped with each release.

## See also

[install.md](install.md) — paths, components, file locations.
[support.md](support.md) — when the device still misbehaves, what to send.
