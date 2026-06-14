# Plan: Migrate claude-desktop-buddy from M5StickC **Plus** to **Plus2**

## Context

The firmware was written for the original **M5StickC Plus**, which manages display power, backlight, battery and the power button through an **AXP192 PMIC** over I²C. The **M5StickC Plus2 removed the AXP192** and replaced it with plain-GPIO power control (power-hold = GPIO4, backlight = GPIO27, ST7789V2 panel, power button = GPIO35, LED = GPIO19).

Because `M5.begin()` in the `m5stack/M5StickCPlus` library enables the backlight via the (absent) AXP192 LDO rail — see `.pio/.../M5StickCPlus/src/AXP192.cpp:10` (`Set LDO2 & LDO3 (TFT_LED & TFT)`) — the Plus2 backlight is never turned on. Result: **black screen**, and every `M5.Axp.*` call is a no-op against a missing chip.

**Outcome:** replace the AXP192-bound library with **M5Unified**, which auto-detects the Plus2 and drives GPIO4/GPIO27/ST7789V2/buzzer/IMU/RTC correctly. After migration the screen lights up and battery/RTC/power features work on Plus2 hardware.

## Approach

Single library migration to `m5stack/M5Unified`. M5Unified keeps `M5.Lcd` as an alias of `M5.Display` and its drawing API is TFT_eSPI-compatible, so the large body of drawing code is untouched. Changes concentrate in: build config, init, the sprite/render-target *types*, and the AXP/IMU/RTC/Beep hardware calls.

### 1. `platformio.ini`
- Rename env to `[env:m5stickc-plus2]` (cosmetic; also gives a fresh `.pio/libdeps`).
- Keep `board = m5stick-c` (M5Unified detects Plus2 at runtime — no Plus2 board JSON exists in platform 6.13.0).
- Add `board_upload.flash_size = 8MB` and `board_build.flash_size = 8MB` (Plus2 = 8MB).
- `lib_deps`: replace `m5stack/M5StickCPlus` with `m5stack/M5Unified` (keep AnimatedGIF + ArduinoJson).
- If M5Unified fails to resolve against platform 6.13.0 / arduino-esp32 2.0.x, pin a compatible M5Unified version (fallback: bump `platform` to a newer espressif32). Flag at build time.

### 2. Sprite + render-target types (mechanical, repo-wide)
- `TFT_eSprite spr(&M5.Lcd)` → `M5Canvas spr(&M5.Display)` in `src/main.cpp:8`.
- `extern TFT_eSprite spr;` → `extern M5Canvas spr;` in `character.cpp`, `buddy.cpp`, and **all 18 `src/buddies/*.cpp`**.
- Render-target pointer type `TFT_eSPI*` → `LovyanGFX*` (the common base of `M5GFX` and `M5Canvas`) in: `buddy.cpp:36`, `buddy.h:11-12`, `character.cpp:47,253`, `character.h:27-28`. Drawing calls through `_tgt->` (`drawPixel`, `setTextColor`, `setCursor`, `print`, `setTextSize`) are API-compatible.
- Replace every `#include <M5StickCPlus.h>` with `#include <M5Unified.h>` (main.cpp, data.h, xfer.h, character.cpp, buddy.cpp, all buddies/*.cpp).

### 3. Init — `src/main.cpp` setup() (~line 939)
- `M5.begin();` → `auto cfg = M5.config(); M5.begin(cfg);`
- Remove `M5.Imu.Init();` and `M5.Beep.begin();` (handled by `M5.begin`).
- Keep `M5.Lcd.setRotation(0)` (alias OK) or switch to `M5.Display`.

### 4. Hardware-call mapping (find/replace across main.cpp, data.h, xfer.h)

| Old (AXP192 / Plus) | New (M5Unified / Plus2) |
|---|---|
| `M5.Axp.ScreenBreath(20+lvl*20)` | `M5.Display.setBrightness(map 0..4 → ~50..255)` |
| `M5.Axp.SetLDO2(true/false)` (screen on/off) | `M5.Display.wakeup()+setBrightness(..)` / `M5.Display.setBrightness(0)` (optionally `sleep()`) |
| `M5.Axp.ScreenBreath(8)` (nap dim) | `M5.Display.setBrightness(~20)` |
| `M5.Axp.PowerOff()` | `M5.Power.powerOff()` |
| `M5.Axp.GetBatVoltage()*1000` (mV) | `M5.Power.getBatteryVoltage()` (already mV) |
| `(vBat-3200)/10` pct hack | `M5.Power.getBatteryLevel()` (real %) |
| `M5.Axp.GetBatCurrent()` (mA) | not available on Plus2 → display `--`/omit current |
| `M5.Axp.GetVBusVoltage()>4.0` (USB present) | `M5.Power.isCharging()` (proxy; note full-battery-on-USB edge) |
| `M5.Axp.GetTempInAXP192()` | `M5.Imu.getTemp()` (IMU temp) or drop the temp line |
| `M5.Axp.GetBtnPress()==0x02` (pwr short-press) | `M5.BtnPWR.wasClicked()` |
| `M5.Imu.getAccelData(&ax,&ay,&az)` | `M5.Imu.getAccel(&ax,&ay,&az)` (same MPU6886, same axes) |
| `M5.Beep.tone(f,d)` / `M5.Beep.update()` | `M5.Speaker.tone(f,d)` / (update handled by `M5.update()`) |
| `LED_PIN = 10` | `LED_PIN = 19` (Plus2 LED; verify active-low on hardware) |

### 5. RTC struct change — `src/main.cpp:352-361,411-417`, `src/data.h:81-85`
- `RTC_TimeTypeDef`/`RTC_DateTypeDef` → `m5::rtc_time_t`/`m5::rtc_date_t`.
- `M5.Rtc.GetTime/GetDate` → `M5.Rtc.getTime/getDate`; `SetTime/SetDate` → `setTime/setDate` (or `setDateTime`).
- Field renames at every access: `.Hours→.hours`, `.Minutes→.minutes`, `.Seconds→.seconds`, `.Month→.month`, `.Date→.date`, `.WeekDay→.weekDay`, `.Year→.year`. Affects `drawClock()`, the mood/clock logic block, and `clockDow()`.

### 6. Color macros
Bare `GREEN`/`RED` (used in main.cpp) come from TFT_eSPI's `#define`s. If M5GFX doesn't expose the unprefixed names, add `#define`s (e.g. `GREEN 0x07E0`, `RED 0xF800`) or switch to `TFT_GREEN`/`TFT_RED`. `BUDDY_*` extern colors are plain `uint16_t` — no change.

## Files to modify
- `platformio.ini`
- `src/main.cpp` (bulk of the work)
- `src/data.h`, `src/xfer.h`
- `src/buddy.cpp`, `src/buddy.h`
- `src/character.cpp`, `src/character.h`
- `src/buddies/*.cpp` ×18 (2-line header/extern swap each)
- `buddy_common.h` — no change

Unaffected: `ble_bridge.*` (independent ESP32 BLE stack), `stats.h`, `character` data files.

## Verification
1. **Headless compile** (no hardware): `pio run -e m5stickc-plus2` from the repo root — this catches all API/type errors. Iterate until clean.
2. **Flash + observe** (user, on the Plus2): `pio run -e m5stickc-plus2 -t upload` then `pio device monitor -b 115200`.
   - Screen lights up, boot "Hello!"/owner splash visible (primary success criterion).
   - Serial shows `M5Unified`/`buddy: ...` lines.
   - Check: brightness cycle (settings), screen-off via power button + wake, face-down nap dim, battery % on DEVICE info page, landscape clock when on USB, buzzer beeps, LED on attention (flip polarity if inverted).
3. Hardware-dependent items to confirm/tune on-device: LED pin polarity (GPIO19), USB-present detection semantics, IMU temp value sanity.

## Risks / notes
- Battery **current (mA)** and **AXP temperature** have no Plus2 equivalent — these info-screen fields are downgraded (omit current; IMU temp or drop).
- M5Unified version vs platform 6.13.0 compatibility — resolve at build step; may need a version pin or platform bump.
- Migration is large in file count but mostly mechanical; the only logic-bearing edits are RTC field renames and the battery/USB/power-button mappings.
