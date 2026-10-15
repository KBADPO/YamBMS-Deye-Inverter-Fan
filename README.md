# Deye Inverter + YamBMS + Smart Cooling Fan (ESPHome & Home Assistant)

[![ESPHome](https://img.shields.io/badge/ESPHome-2024+-orange.svg)](https://esphome.io/)
[![Home Assistant](https://img.shields.io/badge/Home%20Assistant-Ready-blue.svg)](https://www.home-assistant.io/)
[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-green.svg)](https://www.gnu.org/licenses/gpl-3.0)
[![Hardware](https://img.shields.io/badge/Hardware-ESP32--S3-red.svg)](https://www.espressif.com/)

Готове рішення «під ключ» для моніторингу та керування сонячною станцією на базі гібридного інвертора **Deye SG0XLP1 (1-фазний низьковольтний 48V)**, агрегації кількох BMS акумулятора через **YamBMS** по шині CAN, безшумного охолодження 4-піновим кулером (20 кГц ШІМ) та готового комплексного дашборду **Home Assistant**.  
*A turnkey, production-ready solution to monitor and manage a Deye SG0XLP1 hybrid inverter, aggregate battery BMS units via YamBMS over CAN bus, control external high-frequency (20kHz) PWM cooling fans, and visualize everything in Home Assistant.*

---

## ✨ Можливості проекту / Key Features

1. ⚡ **Повний моніторинг і керування Deye (RS485 Modbus RTU)**:
   - Читання всіх параметрів у реальному часі: напруга, струм і потужність генерації PV (сонячних панелей), навантаження будинку, мережі та акумулятора.
   - Температури радіаторів AC та DC інвертора.
   - Повне двостороннє керування режимами роботи інвертора, лімітами струмів заряду/розряду та розкладом Time of Use (TOU) прямо з Home Assistant.
2. 🔋 **Розумне об'єднання батарей (YamBMS CAN Bus)**:
   - Агрегація даних кількох BMS (наприклад, JK-BMS по BLE) в один віртуальний акумулятор.
   - Динамічний розрахунок безпечних струмів заряду та розряду за температурою комірок (таблиці CCL/DCL) та станом заряду (Auto CVL / Float / Rebulk).
   - Емуляція протоколу Pylontech/CANbus та пряма передача даних у CAN BMS порт інвертора Deye.
3. 🌀 **Інтелектуальне 20 кГц ШІМ-охолодження (Smart Cooling Fan)**:
   - Апаратна генерація ШІМ на частоті **20 кГц** (GPIO18) — жодного писку котушок та шуму двигуна на низьких обертах.
   - Гнучкі криві обдуву в Home Assistant (лінійна, логарифмічна) за температурами AC/DC.
   - **Випереджальні тригери**: запуск охолодження при високій генерації PV або високому навантаженні ще до перегріву радіатора.
   - **Апаратний Fail-Safe**: при втраті зв'язку з HA або Wi-Fi кулер автоматично переходить на 10% потужності для захисту обладнання.
4. 📊 **Готовий дашборд для Lovelace**:
   - Інтерфейс у файлі [`Dashboard Solar Station.yaml`](Dashboard%20Solar%20Station.yaml) з красивими анімаціями потоків енергії (Power Flow Card), детальними комірками BMS, керуванням інвертором та панеллю налаштування вентилятора.

---

## 🔌 Апаратне підключення та розпіновка / Hardware & Wiring

Плата: **ESP32-S3 DevKitC-1** (або сумісна з підтримкою USB/UART/CAN).

| Функція / Function | Пін ESP32-S3 / Pin | Призначення / Purpose |
| :--- | :--- | :--- |
| **RS485 RX** | `GPIO16` | Modbus RTU RX від інвертора Deye (порт RS485/Meter через модуль TTL-RS485) |
| **RS485 TX** | `GPIO17` | Modbus RTU TX до інвертора Deye (TTL-to-RS485) |
| **PWM Fan** | `GPIO18` | ШІМ керування швидкістю 4-пінового комп'ютерного кулера (20 кГц) |
| **CAN Bus** | Вбудований TWAI | CAN H / CAN L до BMS-порту інвертора Deye (через трансивер CAN) |
| **RGB LED** | `GPIO48` | Індикація стану плати та мережі (DevKitC-1 LED-48) |

> [!TIP]
> Для підключення до кулера рекомендується звичайний 4-піновий комп'ютерний вентилятор 12V (наприклад, Arctic P12/P14 PWM або Noctua). Пін PWM кулера підключається безпосередньо до GPIO18, живлення кулера — від зовнішнього джерела 12V (земля спільна з ESP32).

---

## 🚀 Інструкція зі встановлення / How to Replicate

Щоб розгорнути цю конфігурацію на власному обладнанні, виконайте наступні прості кроки:

### Крок 1. Клонування репозиторію / Clone Repository
```bash
git clone https://github.com/KBADPO/YamBMS-Deye-Inverter-Fan.git
cd YamBMS-Deye-Inverter-Fan
```

### Крок 2. Налаштування паролів / Configure Secrets
Створіть власний файл `secrets.yaml` на основі шаблону:
```bash
cp secrets.yaml.example secrets.yaml
```
Відкрийте `secrets.yaml` у текстовому редакторі та вкажіть:
- `wifi_ssid` та `wifi_password` — доступ до вашої мережі Wi-Fi;
- `api_key` — ключ підключення ESPHome Native API до Home Assistant (згенеруйте у вебінтерфейсі ESPHome);
- `ota_password` — пароль для оновлення прошивки повітрям (OTA).

### Крок 3. Підготовка конфігурації прошивки / Firmware Config
Скопіюйте готовий робочий шаблон:
```bash
cp "YamBMS+Deye Inverter.example.yaml" "YamBMS+Deye Inverter.yaml"
```
Відкрийте `YamBMS+Deye Inverter.yaml` і перевірте параметри вашої системи:
1. **Батарея**:
   - `yambms_battery_chemistry`: `'1'` (LiFePO4) / `'2'` (Li-ion) / `'3'` (LTO)
   - `yambms_cell_count`: `'16'` (для 48V збірки 16S)
   - `yambms_bulk_v`: напруга повного заряду (наприклад, `'55.5'`)
   - `yambms_float_v`: напруга буферного режиму (наприклад, `'54'`)
   - `yambms_max_requested_charge_current`: максимальний бажаний струм заряду (А)
   - `yambms_max_requested_discharge_current`: максимальний струм розряду (А)
2. **BMS (JK-BMS BLE)**:
   - `bms_ble_mac_address`: MAC-адреса вашої JK-BMS (наприклад, `c8:47:80:1b:78:d7`)
   - `bms_ble_protocol_version`: версія протоколу (`JK02_32S` або `JK02_24S`)
3. **Плата**:
   - За замовчуванням увімкнено пакет `packages/board/board_ESP32-S3_DevKitC-1_LED-48.yaml`. Якщо у вас інша плата — розкоментуйте відповідний рядок у секції `remote_packages:`.

### Крок 4. Компіляція та прошивка / Compile & Flash
Підключіть ESP32-S3 до комп'ютера через USB і виконайте:
```bash
esphome compile "YamBMS+Deye Inverter.yaml"
esphome run "YamBMS+Deye Inverter.yaml"
```
*(Також ви можете імпортувати цей файл в **ESPHome Dashboard** у Home Assistant і прошити плату через браузер).*

### Крок 5. Налаштування в Home Assistant / Home Assistant Setup
1. **Інтеграція ESPHome**: Після підключення плати до Wi-Fi Home Assistant автоматично знайде новий пристрій — підтвердіть його додавання та введіть API-ключ.
2. **Помічники та логіка охолодження**:
   - Додайте Helper-змінні та автоматизацію керування кулером згідно з інструкцією у файлі **[`ha_fan_config.md`](ha_fan_config.md)**.
3. **Дашборд Lovelace**:
   - Створіть новий дашборд у Home Assistant і вставте готовий код із файлу **[`Dashboard Solar Station.yaml`](Dashboard%20Solar%20Station.yaml)**.

---

## 🎛️ Керування охолодженням у Home Assistant / Smart Fan Control

Вкладка **«Вентилятор»** дашборду надає повний контроль над обдувом інвертора:

| Група / Group | Сутність / Helper | Замовчування / Default | Опис / Description |
| :--- | :--- | :--- | :--- |
| **Статус / Status** | `sensor.deye_inverter_ac_temperature` | — | Температура радіатора AC інвертора (°C) |
| **Статус / Status** | `sensor.deye_inverter_dc_temperature` | — | Температура радіатора DC інвертора (°C) |
| **Статус / Status** | `number.inverter_cooling_fan_fan_speed_target` | — | Завдання швидкості обдуву кулера (0–100%) |
| **Температура / Temp** | `input_number.fan_start_speed` | `20 %` | Початкова швидкість кулера при пуску |
| **Температура / Temp** | `input_number.fan_hysteresis` | `2 °C` | Температурний гістерезис вимкнення |
| **Температура / Temp** | `input_number.fan_ac_min_temp` | `45 °C` | Поріг старту вентилятора за AC |
| **Температура / Temp** | `input_number.fan_ac_max_temp` | `65 °C` | Температура 100% обертів за AC |
| **Температура / Temp** | `input_number.fan_dc_min_temp` | `45 °C` | Поріг старту вентилятора за DC |
| **Температура / Temp** | `input_number.fan_dc_max_temp` | `65 °C` | Температура 100% обертів за DC |
| **Температура / Temp** | `input_select.fan_control_law` | `Linear` | Крива обдуву (`Linear`, `Logarithmic`, `Reverse Logarithmic`) |
| **Тригери / Triggers** | `input_boolean.enable_pv_power_trigger` | `off` | Увімкнення пуску за потужністю сонячних панелей (PV) |
| **Тригери / Triggers** | `input_number.pv_power_threshold` | `3000 W` | Поріг потужності сонця для старту кулера |
| **Тригери / Triggers** | `input_boolean.enable_out_power_trigger` | `off` | Увімкнення пуску за навантаженням інвертора |
| **Тригери / Triggers** | `input_number.out_power_threshold` | `2000 W` | Поріг навантаження для старту кулера |
| **Тригери / Triggers** | `input_boolean.enable_grid_power_trigger` | `off` | Увімкнення пуску за потужністю мережі |
| **Тригери / Triggers** | `input_number.grid_power_threshold` | `2000 W` | Поріг потужності мережі для старту кулера |

Детальний YAML-код помічників, оптимізована автоматизація розрахунку швидкості та картки Lovelace описані у файлі **[`ha_fan_config.md`](ha_fan_config.md)**.

---

## 📂 Структура репозиторію / Repository Structure

```text
├── .gitignore                            # Захист secrets.yaml та робочих конфігурацій
├── secrets.yaml.example                  # Шаблон паролів, ключів Wi-Fi та API
├── YamBMS+Deye Inverter.example.yaml     # Готовий шаблон прошивки ESPHome (Deye + YamBMS + Fan)
├── YamBMS BLE.example.yaml               # Готовий шаблон ESPHome для автономного режиму YamBMS BLE
├── pv-inverter.Deye-SG0XLP1.yaml         # Базовий пакет сенсорів та регістрів Deye
├── ha_fan_config.md                      # Повна інструкція з налаштування кулера в Home Assistant
├── Dashboard Solar Station.yaml          # Готовий Lovelace Dashboard станції (BMS, Deye, Fan)
└── Dashboard Overview.yaml               # Альтернативний Lovelace Dashboard
```

---

## 📜 Ліцензія та подяка / License & Acknowledgments

- Дякуємо [Sleeper85](https://github.com/Sleeper85) за створення проекту [YamBMS](https://github.com/Sleeper85/esphome-yambms).
- Дякуємо [Lewa-Reka](https://github.com/Lewa-Reka) за розробку [esphome-deye-inverter](https://github.com/Lewa-Reka/esphome-deye-inverter).
- Код поширюється на умовах ліцензії GNU General Public License v3.0 (GPL-3.0).

