# Big Plane Radar

Прошивка для дисплея Waveshare ESP32-S3-Touch-LCD-7. Показывает живой ADS-B
радар вокруг заданной точки: самолёты, подписи, высоту, вертикальную скорость,
кольца дальности и компактный список бортов справа.

Проект не использует LVGL. Интерфейс рисуется напрямую в RGB565 framebuffer, а
для 800x480 RGB LCD и тача GT911 используется официальный стек Waveshare
`ESP32_Display_Panel`.

![Big Plane Radar на Waveshare ESP32-S3-Touch-LCD-7](docs/plane-radar.png)

## 3D-печатный стенд

Подходящий настольный стенд для этого дисплея и прошивки доступен на MakerWorld:

https://makerworld.com/ru/models/3034679-stand-for-esp32-s3-touch-lcd-7-for-plane-radar

## Железо

- Waveshare ESP32-S3-Touch-LCD-7, RGB LCD 800x480
- USB-кабель с передачей данных, подключенный в порт `UART1`
- Переключатель на плате должен стоять в положении `UART1`

На macOS может понадобиться USB serial driver, если плата не появляется как
`/dev/cu.usbmodem*` или `/dev/cu.wchusbserial*`. На Linux устройство обычно
появляется как `/dev/ttyACM*` или `/dev/ttyUSB*`; пользователю может понадобиться
доступ к группе `dialout`.

## Возможности

- setup portal при первом запуске: `PlaneRadar-Setup`;
- сохранение Wi-Fi, центра радара, единиц измерения, отображения ВПП и дальности
  в NVS;
- данные ADS-B из `https://opendata.adsb.fi/api/v3/`;
- локальное предсказание позиции между ADS-B обновлениями с отрисовкой примерно
  `4 FPS`;
- опциональная строка маршрута город-город в правом списке через кэшируемые
  callsign-запросы к `https://api.adsbdb.com/`;
- фоновый реконнект к Wi-Fi после отключений питания или позднего старта роутера;
- управление тачем: короткий тап меняет дальность, длинное нажатие открывает
  setup portal;
- boot setup window: удержание экрана во время запуска принудительно открывает
  setup portal;
- endpoint для скриншота: `/screenshot` и `/screenshot.bmp`;
- консервативные настройки RGB LCD для этой панели: PCLK `14 MHz` и RGB bounce
  buffer `800 * 10`.

## Легенда символов

Символы используют ADS-B поле `category`, если оно есть в ответе.

![Легенда символов самолётов](docs/aircraft-symbol-legend.svg)

## Структура

```text
.
├── big_plane_radar.ino
├── build_arduino_cli.sh
├── esp_panel_board_custom_conf.h
├── lib/
│   └── ArduinoJson/
├── releases/
├── scripts/
│   └── build_iata_airports.py
├── src/
│   ├── airports.h
│   ├── airports_iata.h
│   ├── main.cpp
│   ├── panel_display.cpp
│   └── panel_display.h
└── vendor/
    └── waveshare-libraries/
```

В `vendor/waveshare-libraries` лежат только нужные Arduino-библиотеки:
`ESP32_Display_Panel`, `ESP32_IO_Expander` и `esp-lib-utils`. LVGL не нужен.

## Установка инструментов

Нужно установить:

- `arduino-cli`
- Arduino core `esp32:esp32`
- `esptool` для ручной прошивки и диагностики

Установка ESP32 core:

```sh
arduino-cli core update-index
arduino-cli core install esp32:esp32
```

## Сборка

```sh
bash build_arduino_cli.sh
```

По умолчанию Wi-Fi логин и пароль в прошивку не вшиваются. Центр радара по
умолчанию — Лондон:

```text
Latitude:  51.507400
Longitude: -0.127800
```

Координаты можно переопределить при сборке:

```sh
DEFAULT_LAT=51.507400 \
DEFAULT_LON=-0.127800 \
bash build_arduino_cli.sh
```

Опционально можно вшить Wi-Fi defaults:

```sh
DEFAULT_WIFI_SSID="YourNetwork" \
DEFAULT_WIFI_PASSWORD="YourPassword" \
bash build_arduino_cli.sh
```

## Прошивка

Переведите переключатель платы в `UART1`, подключите USB именно в порт `UART1`,
затем выполните:

```sh
UPLOAD=1 CLEAN=1 PORT=/dev/cu.usbmodem5AE71132621 bash build_arduino_cli.sh
```

Замените `PORT` на свой:

```sh
# macOS
PORT=/dev/cu.usbmodemXXXX
PORT=/dev/cu.wchusbserialXXXX

# Linux
PORT=/dev/ttyACM0
PORT=/dev/ttyUSB0
```

## Прошивка из браузера

Плату можно прошить прямо из браузера через Web Serial. Нужен Chrome, Edge или
другой Chromium-based desktop browser. На iOS этот способ не работает.

### Вариант A: Adafruit WebSerial ESPTool

Это самый простой вариант без своего хостинга: сервис позволяет выбрать локальный
`.bin` файл вручную.

1. Откройте [Adafruit WebSerial ESPTool](https://adafruit.github.io/Adafruit_WebSerial_ESPTool/).
2. Переведите переключатель платы в `UART1` и подключите USB именно в порт `UART1`.
3. Нажмите `Connect` и выберите serial-порт ESP32-S3.
4. Используйте одну строку файла:
   - offset: `0x0`
   - file: `releases/big_plane_radar.ino.merged.bin`
5. Нажмите `Erase`, затем `Program`.

Для браузерной прошивки используйте именно merged binary.

### Вариант B: ESP Web Tools page

В репозитории есть готовая статическая страница `web-installer/`. Она использует
[ESP Web Tools](https://esphome.github.io/esp-web-tools/), где прошивка
ставится через manifest и release binary.

Как использовать:

1. Опубликуйте репозиторий через GitHub Pages или любой другой HTTPS static host.
2. Откройте:

```text
https://<your-github-user>.github.io/big_plane_radar/web-installer/
```

3. Нажмите `Install Big Plane Radar` и выберите serial-порт ESP32-S3.

ESP Web Tools требует HTTPS, а `.bin` файл должен быть доступен браузеру. Готовый
manifest указывает на:

```text
../releases/big_plane_radar.ino.merged.bin
```

## Первый запуск

Если конфигурация ещё не сохранена, плата поднимает Wi-Fi точку:

```text
PlaneRadar-Setup
```

Подключитесь к ней и откройте:

```text
http://192.168.4.1
```

После подключения платы к вашему Wi-Fi страница настроек также доступна по:

```text
http://plane-radar.local
```

Там задаются Wi-Fi, координаты центра радара, единицы измерения и отображение
ВПП. После сохранения плата перезагрузится.

## Скриншот

Когда плата подключена к Wi-Fi, текущий экран можно снять так:

```sh
curl -o docs/screenshot.bmp http://plane-radar.local/screenshot.bmp
```

Прямые URL:

```text
http://plane-radar.local/screenshot
http://plane-radar.local/screenshot.bmp
```

Если mDNS не работает, используйте IP платы:

```sh
curl -o docs/screenshot.bmp http://<device-ip>/screenshot.bmp
```

## Release binaries

Готовые файлы сборки лежат в `releases/`:

- `big_plane_radar.ino.merged.bin`

Вручную прошивать можно тем же merged binary:

```sh
esptool.py --chip esp32s3 --port /dev/cu.usbmodemXXXX --baud 921600 \
  write_flash 0x0 releases/big_plane_radar.ino.merged.bin
```

Самый простой способ для разработки — `UPLOAD=1 ... bash build_arduino_cli.sh`.
