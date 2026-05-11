# ESP32-C6 Super Mini — mpcbstudio IoT

## Что делает прошивка

- Подключается к WiFi и MQTT брокеру `mqtt.mpcbstudio.com:8883` (TLS)
- Подписывается на топик `mpcb/devices/test1/command`
- При команде `{"on": true}` — зажигает встроенный RGB LED зелёным
- При команде `{"on": false}` — гасит LED
- Публикует текущее состояние в `mpcb/devices/test1/state`

## Индикация RGB LED (GPIO8)

| Цвет    | Статус                    |
|---------|---------------------------|
| Синий   | Загрузка, подключение WiFi |
| Жёлтый  | Подключение к MQTT         |
| Зелёный | Устройство включено (on)   |
| Выкл.   | Устройство выключено (off) |
| Красный | Нет соединения с MQTT      |

## Настройка перед прошивкой

Открой `src/main.cpp` и измени:

```cpp
#define WIFI_SSID     "YOUR_WIFI_SSID"      // имя твоей WiFi сети
#define WIFI_PASSWORD "YOUR_WIFI_PASSWORD"   // пароль WiFi
```

Остальные параметры уже настроены для тестового устройства.

## Прошивка через VSCode + PlatformIO

1. Открой папку проекта в VSCode
2. Подключи ESP32-C6 по USB-C
3. Нажми кнопку **Upload** (стрелка вправо) в нижней панели PlatformIO
4. После прошивки открой **Serial Monitor** (иконка розетки)

## Тест через MQTT

Отправить команду вручную с сервера:

```bash
# Включить
mosquitto_pub -h mqtt.mpcbstudio.com -p 8883 --capath /etc/ssl/certs \
  -u user_3 -P 'Da6vInaZXM5rUFFkobcEgA' \
  -t mpcb/devices/test1/command -m '{"on":true}'

# Выключить
mosquitto_pub -h mqtt.mpcbstudio.com -p 8883 --capath /etc/ssl/certs \
  -u user_3 -P 'Da6vInaZXM5rUFFkobcEgA' \
  -t mpcb/devices/test1/command -m '{"on":false}'
```

## Голосовое управление

После прошивки скажи Алисе:
- **"Алиса, включи тестовую лампу"**
- **"Алиса, выключи тестовую лампу"**
