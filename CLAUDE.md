# Контекст проекта для Claude Code

## Что это

Прошивка ESP32-C6 Super Mini — устройство "Ворота цех" на платформе mpcbstudio.
Построена на библиотеке **mpcb-iot-core** (GitHub: `github.com/remixvit/mpcb-iot-core`).

## Железо

- **Чип:** ESP32-C6FH4, COM10, MAC `58:E6:C5:18:FC:C8`, Device ID `esp32-FCC8`
- **IP:** `192.168.1.41`, mDNS: `http://mpcb-FCC8.local`
- **GPIO8** — встроенный WS2812 (статус)
- **GPIO4** — реле Gate
- **GPIO3** — кнопка (INPUT_PULLUP)
- **Лог:** `curl http://192.168.1.41/api/log-text`
- **Прошивка:** `$env:USERPROFILE\.platformio\penv\Scripts\pio.exe run --target upload`

## ESP32-C6 Super Mini — пины

| Категория | Пины | Причина |
|-----------|------|---------|
| ❌ Forbidden | 12 (USB−), 13 (USB+), 18 (Flash), 19 (Flash) | Нельзя использовать |
| ⚠ Warn | 4–7 (JTAG), 8 (WS2812), 9 (BOOT), 15 (LED) | Осторожно |
| ✅ Safe | 0, 1, 2, 3 (ADC), 14, 20 (RX), 21 (TX), 22 (SDA), 23 (SCL) | Свободно |

**I2C шина:** SDA=GPIO22, SCL=GPIO23 (фиксировано в прошивке — `Wire.begin(22, 23)`)

## Библиотека mpcb-iot-core

Ключевые файлы:
- `src/MpcbIotCore.h/.cpp` — WiFi, MQTT, BLE, FSM
- `src/peripheral/PeriphManager.h/.cpp` — GPIO конструктор, rules engine, MQTT
- `src/web/ConfigServer.cpp` — веб-интерфейс (тёмная тема, #7b2feb accent)
- `src/storage/ConfigStorage.h/.cpp` — NVS хранилище
- `src/log/RingLog.h/.cpp` — кольцевой лог (60 записей, /api/log-text)

## MQTT протокол

**Брокер:** `mqtt.mpcbstudio.com:8883` TLS
**Credentials в NVS:** `user_4` / `DATeyjWZ6sd9N-5wFtj5vg`
**Спецификация:** `github.com/remixvit/mpcbstudio-api` → `FIRMWARE_SPEC.md`

Последовательность при старте:
```
announce → subscribe +/set → config → {key}/state (не для датчиков — они публикуют из loop)
```

## Что сделано ✅

### mpcb-iot-core
- WiFi connect + AP portal (captive)
- MQTT: LWT, announce, config, state publish, +/set subscribe
- BLE provisioning (NimBLE-Arduino v2): WiFi, MQTT, GPIO, Rules через BLE
- OTA через BLE и через веб (HTTP upload .bin)
- ConfigServer (веб UI): Device, WiFi, MQTT, GPIO, Logs, OTA
- mDNS `http://mpcb-XXXX.local` + `/api/reboot`
- **PeriphManager — GPIO типы:**
  - `relay` — цифровой выход ON/OFF/pulse
  - `button` — цифровой вход (30мс debounce)
  - `analog` — ADC, публикует каждые 10с
  - `pwm` — ШИМ 0–255
  - `neopixel` — WS2812 ack-only
  - `dht22` — temp+humidity каждые 30с (auto-detect `__has_include<DHT.h>`)
  - `ds18b20` — температура каждые 30с (auto-detect `__has_include<DallasTemperature.h>`)
  - `aht10` — I2C temp+humidity каждые 30с (auto-detect `__has_include<Adafruit_AHTX0.h>`)
    адреса 0x38/0x39, Wire.begin(22,23) автоматически
- **Rules engine:**
  - Кнопки: `pressed` / `released` / `any` → `on` / `off` / `toggle` / `pulse`
  - Датчики: `temp_above` / `temp_below` / `hum_above` / `hum_below` / `above` / `below`
    с порогом (float) и гистерезисным лэтчем (re-arms когда условие перестаёт выполняться)
- **GPIO конструктор веб UI:**
  - Dropdown пинов ESP32-C6 Super Mini (forbidden/warn/safe)
  - I2C типы: dropdown адреса (0x38/0x39) вместо пина + подсказка `SDA→22 SCL→23`
  - Автопереключение pin↔addr при смене типа GPIO↔I2C
  - Per-type limits (UI + серверная валидация):
    relay:8, button:8, analog:4, pwm:4, neopixel:2, dht22:2, ds18b20:2, aht10:2, vl53:1, pcf8574:2
- **Rules UI:** trigger=только входы (button/analog/dht22/ds18b20/aht10/vl53),
  target=только выходы (relay/pwm/neopixel/pcf8574);
  события и поле порога меняются динамически в зависимости от типа триггера
- Нормализация ключей правил lowercase (BLE + веб)
- Серверная валидация: forbidden pins, дубли, type limits, I2C типы пропускают pin-проверку

### esp32-mpcb
- Прошивка "Ворота цех": NeoPixel статус, реле Gate (GPIO4), кнопка (GPIO3)
- platformio.ini: GitHub URL + все библиотеки датчиков включены
- CLAUDE.md в репо для кросс-машинной работы

## Архитектурный план → многочиповость

```cpp
// Разделить транспорт от периферии:
class ITransport {
    virtual bool publish(const String& topic, const String& payload, bool retain) = 0;
    virtual bool subscribe(const String& topic) = 0;
};
// MpcbIotCore    : public ITransport  ← C6 WiFi  (сейчас)
// MpcbZigbeeCore : public ITransport  ← C6/H2 Zigbee (будущее)
// PeriphManager принимает ITransport* — работает с любым транспортом
```

## Роадмап

### Следующий шаг — VL53L0X и PCF8574

I2C инфраструктура уже готова (`i2cAddr` в Peripheral, Wire.begin, I2C конструктор в UI).
Нужно только добавить реализацию типов:

| Датчик | Тип | Библиотека | Адрес | Особенность |
|--------|-----|------------|-------|-------------|
| VL53L0X | ToF дистанция | `adafruit/Adafruit_VL53L0X` | 0x29 | XSHUT пин для >1 датчика |
| PCF8574 | Порт-экспандер | `robtillaart/PCF8574` | 0x20–0x27 | 8 каналов, нужен `channel` в Peripheral |

Для PCF8574 нужно добавить `uint8_t channel` в struct Peripheral.

### После VL53/PCF — ITransport абстракция

Разделить `MpcbIotCore` от `PeriphManager` через интерфейс `ITransport`.
Даст возможность добавить Zigbee без переписывания периферии.

### OTA via MQTT (ждём сервер)

Сервер пришлёт `{"cmd":"ota","url":"https://..."}` через MQTT.
Реализация: `HTTPClient` скачивает `.bin`, `Update.h` прошивает.

### Новые устройства
| Устройство | Трудозатраты | Основная работа |
|------------|-------------|-----------------|
| C3 (WiFi) | ~полдня | Пиноут C3 в ConfigServer |
| C6 Zigbee | ~2 нед. | MpcbZigbeeCore + ITransport рефакторинг |
| H2 (battery) | ~3 нед. | + PowerManager deep sleep |

### Flutter app (`e:/Projects/mpcb-app`)
- GPIO экран: тумблеры реле, значения датчиков
- Rules экран
- Маркер PS→MC

## Важно: обновление библиотеки

platformio.ini использует GitHub URL — PlatformIO кеширует.
Изменения в `mpcb-iot-core` подхватываются только после:
```powershell
# 1. Запушить изменения в mpcb-iot-core
cd e:/Projects/mpcb-iot-core; git push

# 2. Обновить пакет и перепрошить
cd e:/Projects/esp32-mpcb
git config --global --add safe.directory "E:/Projects/esp32-mpcb/.pio/libdeps/esp32c6-supermini/mpcb-iot-core"
& "$env:USERPROFILE\.platformio\penv\Scripts\pio.exe" pkg update
& "$env:USERPROFILE\.platformio\penv\Scripts\pio.exe" run --target upload

# Если pkg update не помогает — полный сброс:
# Remove-Item -Recurse -Force ".pio\libdeps"
# & "$env:USERPROFILE\.platformio\penv\Scripts\pio.exe" run --target upload
```

## Библиотеки в platformio.ini (текущие)

```ini
lib_deps =
    https://github.com/remixvit/mpcb-iot-core
    adafruit/Adafruit NeoPixel @ ^1.12.0
    adafruit/DHT sensor library @ ^1.4.0
    milesburton/DallasTemperature @ ^3.9.0
    https://github.com/paulstoffregen/OneWire   # GitHub — PlatformIO registry не совместим с ESP32-C6
    adafruit/Adafruit AHTX0 @ ^2.0.5
```
