# Контекст проекта для Claude Code

## Что это

Прошивка для ESP32-C6 Super Mini — устройство "Ворота цех" на платформе mpcbstudio.
Построена на библиотеке **mpcb-iot-core** (локально: `e:/Projects/mpcb-iot-core`, GitHub: `github.com/remixvit/mpcb-iot-core`).

## Железо

- **Чип:** ESP32-C6FH4, COM10, MAC `58:E6:C5:18:FC:C8`, Device ID `esp32-FCC8`
- **IP:** `192.168.1.41`, mDNS: `http://mpcb-FCC8.local`
- **GPIO8** — встроенный WS2812 (статус)
- **GPIO4** — реле Gate
- **GPIO3** — кнопка (INPUT_PULLUP)
- **Лог устройства:** `curl http://192.168.1.41/api/log-text`
- **Прошивка:** `$env:USERPROFILE\.platformio\penv\Scripts\pio.exe run --target upload`

## Библиотека mpcb-iot-core

Ключевые файлы:
- `src/MpcbIotCore.h/.cpp` — WiFi, MQTT, BLE, FSM
- `src/peripheral/PeriphManager.h/.cpp` — GPIO конструктор, rules, MQTT publish/subscribe
- `src/web/ConfigServer.cpp` — веб-интерфейс (тёмная тема, #7b2feb accent)

Зависимости (platformio.ini): PubSubClient, ArduinoJson ^7, NimBLE-Arduino ^2, Adafruit NeoPixel.

## MQTT протокол

**Брокер:** `mqtt.mpcbstudio.com:8883` TLS  
**Credentials в NVS:** `user_4` / `DATeyjWZ6sd9N-5wFtj5vg` (обновить через `http://192.168.1.41/mqtt` если изменились)  
**Спецификация:** `github.com/remixvit/mpcbstudio-api` → `FIRMWARE_SPEC.md`

Последовательность при старте (реализована, проверена):
```
announce → subscribe mpcb/devices/{id}/+/set → config → {key}/state для каждой периферии
```

## Известные проблемы

| Проблема | Причина | Статус |
|----------|---------|--------|
| Дублирующиеся устройства на сервере | Сервер делает INSERT вместо UPSERT по device_id при получении config | Баг на сервере, ждём фикса |
| Capabilities пустые в кабинете | Сервер ещё не читает peripherals[] из config автоматически | Серверный разработчик добавит автозаполнение |

## Что нужно сделать (приоритет по порядку)

### 1. Ждём от серверного разработчика
- Upsert по `device_id` при получении `config` топика
- Автозаполнение `capabilities`/`properties` из `peripherals[]` в config JSON

### 2. Следующая разработка
- **Flutter app** (`e:/Projects/mpcb-app`): маркер PS→MC, GPIO экран (тумблеры/значения), Rules экран
- **DHT22** — полная реализация: `adafruit/DHT sensor library`, публикация `{"temp":float,"humidity":float}` каждые 30 с
- **DS18B20** — полная реализация: `milesburton/DallasTemperature`, публикация `{"temp":float}` каждые 30 с

### 3. Будущее
- MpcbZigbeeCore (c6-zigbee, h2)
- PowerManager deep sleep (c3, h2 — батарейки)

## Как запустить с нуля на новом ПК

```bash
git clone https://github.com/remixvit/esp32-mpcb
git clone https://github.com/remixvit/mpcb-iot-core
# В platformio.ini esp32-mpcb прописан lib_extra_dirs = ../mpcb-iot-core/src
# Открыть esp32-mpcb в VSCode + PlatformIO
```
