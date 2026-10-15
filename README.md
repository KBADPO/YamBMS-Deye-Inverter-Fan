# Deye Inverter + YamBMS + Smart Cooling Fan (ESPHome)

Комплексне рішення для моніторингу та керування гібридним інвертором **Deye SG0XLP1 (1-фазний низьковольтний)**, об'єднанням кількох акумуляторних батарей через **YamBMS**, інтелектуальним охолодженням інвертора 4-піновим PWM вентилятором (20 кГц) та візуалізацією в **Home Assistant**.

---

## 📌 Архітектура та авторські проекти

Цей проект інтегрує напрацювання двох потужних рішень спільноти з індивідуальною логікою керування охолодженням:

1. **YamBMS** ([Sleeper85/esphome-yambms](https://github.com/Sleeper85/esphome-yambms)) — агрегація кількох BMS (JK-BMS BLE тощо), розрахунок струмів заряду/розряду за температурними кривими та передача даних в інвертор по шині CAN.
2. **Deye Inverter ESPHome** ([Lewa-Reka/esphome-deye-inverter](https://github.com/Lewa-Reka/esphome-deye-inverter)) — повний моніторинг і двостороннє керування регістрами інвертора Deye через RS485 (Modbus RTU).
3. **Smart PWM Fan Controller** — генерація апаратного ШІМ-сигналу 20 кГц (GPIO18) з плавним регулюванням швидкості кулера за температурою радіатора/DC/AC та аварійним захистом (fail-safe) при втраті зв'язку.
4. **Home Assistant Dashboards** — готові дашборди Lovelace для комплексного моніторингу сонячної станції, батарей та вентилятора.

---

## 🔄 Стратегія оновлення авторських проектів (Fork Workflow)

Щоб мати можливість контролювати власний код, вносити правки та водночас **в 1 клік отримувати оновлення від оригінальних авторів**, використовується підхід через **Fork**:

```mermaid
flowchart TD
    subgraph Upstream["Оригінальні автори (Upstream)"]
        U1["Sleeper85/esphome-yambms"]
        U2["Lewa-Reka/esphome-deye-inverter"]
    end

    subgraph YourForks["Ваші форки на GitHub"]
        F1["github.com/KBADPO/esphome-yambms"]
        F2["github.com/KBADPO/esphome-deye-inverter"]
    end

    subgraph MainRepo["Цей інтеграційний репозиторій / Integration Repo"]
        CFG["YamBMS+Deye Inverter.yaml"]
        DASH["Home Assistant Dashboards"]
        FAN["Fan Automations & Logic"]
    end

    U1 -- "Sync fork (1 клік у GitHub UI / 1 click)" --> F1
    U2 -- "Sync fork (1 клік у GitHub UI / 1 click)" --> F2
    F1 -- "ESPHome remote_packages" --> CFG
    F2 -- "ESPHome pv_inverter package" --> CFG
```

### Крок 1. Створення форків (одноразово) / Step 1. Forking Repositories
1. Відкрийте репозиторій [Sleeper85/esphome-yambms](https://github.com/Sleeper85/esphome-yambms) і натисніть кнопку **Fork** (вгорі праворуч).
2. Відкрийте репозиторій [Lewa-Reka/esphome-deye-inverter](https://github.com/Lewa-Reka/esphome-deye-inverter) і натисніть **Fork**.
3. Тепер у вашому профілі GitHub з'являться незалежні копії обох проектів:
   - `https://github.com/KBADPO/esphome-yambms`
   - `https://github.com/KBADPO/esphome-deye-inverter`

### Крок 2. Як оновлювати ваші форки від авторів / Step 2. Updating Forks
Коли Sleeper85 або Lewa-Reka випускають оновлення чи виправлення:
1. Зайдіть на сторінку вашого форка на GitHub (наприклад, `https://github.com/KBADPO/esphome-yambms`).
2. Натисніть кнопку **Sync fork** під описом репозиторію.
3. Натисніть **Update branch**. Усі оновлення від автора миттєво підтягнуться у ваш форк!

### Крок 3. Підключення ваших форків до прошивки / Step 3. Linking Forks to Firmware
У файлі `YamBMS+Deye Inverter.yaml` у секції `packages:` вкажіть посилання на ваші форки:

```yaml
packages:
  pv_inverter:
    url: https://github.com/KBADPO/esphome-deye-inverter
    ref: main
    refresh: 12h
    files: pv_inverter/deye_hybrid_1p_lv.yaml

  remote_packages:
    url: https://github.com/KBADPO/esphome-yambms
    ref: main
    refresh: 60min
    files:
      - path: 'packages/board/board_ESP32-S3_DevKitC-1_LED-48.yaml'
      # ... other selected packages / інші обрані пакети
```

---

## 🚀 Швидкий старт (Quick Start)

### 1. Клонування репозиторію / Cloning Repository
```bash
git clone https://github.com/KBADPO/YamBMS-Deye-Inverter-Fan.git
cd YamBMS-Deye-Inverter-Fan
```

### 2. Створення локальних робочих файлів / Local Working Configs
Файли з реальними паролями та робочими налаштуваннями додані до `.gitignore` і **не потрапляють на GitHub**.  
*Files with real credentials are in `.gitignore` and **not committed to GitHub**.*  
Створіть робочі файли на основі шаблонів:
```bash
cp secrets.yaml.example secrets.yaml
cp "YamBMS+Deye Inverter.example.yaml" "YamBMS+Deye Inverter.yaml"
```
Відкрийте `secrets.yaml` (або `YamBMS+Deye Inverter.yaml`) та заповніть свої облікові дані:
- `wifi_ssid` / `wifi_password` — доступ до Wi-Fi;
- `api_key` — ключ підключення ESPHome Native API до Home Assistant;
- `ota_password` — пароль бездротової прошивки (OTA).

### 3. Компіляція та завантаження / Compilation & Flashing
Відкрийте **ESPHome Dashboard** у Home Assistant або скомпілюйте локально через CLI:  
*Open **ESPHome Dashboard** in Home Assistant or compile locally via CLI:*
```bash
esphome compile "YamBMS+Deye Inverter.yaml"
esphome run "YamBMS+Deye Inverter.yaml"
```

---

## 🔌 Підключення та розпіновка (ESP32-S3 DevKitC-1) / Pinout & Wiring

| Функція / Function | Пін ESP32-S3 / Pin | Призначення / Purpose |
| :--- | :--- | :--- |
| **RS485 RX** | `GPIO16` | Modbus RTU RX від інвертора Deye (TTL-to-RS485) / from Deye inverter |
| **RS485 TX** | `GPIO17` | Modbus RTU TX до інвертора Deye (TTL-to-RS485) / to Deye inverter |
| **PWM Fan** | `GPIO18` | ШІМ керування швидкістю вентилятора (20 кГц) / PWM fan speed control (20 kHz) |
| **CAN Bus** | Вбудований контролер / Built-in | CAN шина до порту BMS інвертора Deye / CAN bus to Deye BMS port |
| **RGB LED** | `GPIO48` | Індикація стану плати та мережі (DevKitC-1 LED-48) / Status & network LED |

---

## 🎛️ Налаштування охолодження та Home Assistant / Cooling & Home Assistant Setup

Для охолодження інвертора Deye реалізовано дворежимну інтелектуальну систему:  
*A two-tier smart cooling system is implemented for the Deye inverter:*
1. **Апаратний рівень (ESPHome) / Hardware level**:
   - Пін `GPIO18` генерує апаратний ШІМ-сигнал частотою **20 кГц** (без шуму та писку кулера) / *Hardware PWM at 20 kHz prevents motor whine*.
   - **Fail-safe захист**: при втраті зв'язку з Home Assistant (або Wi-Fi) плата автоматично вмикає вентилятор на 10% потужності / *Automatic 10% speed fallback if HA or Wi-Fi connection drops*.
2. **Інтелектуальний рівень (Home Assistant) / Smart level**:
   - Автоматизація розраховує необхідну швидкість кулера (0–100%) за кривою обдуву (лінійна, логарифмічна) / *Automation computes optimal duty cycle dynamically*.
   - Підтримуються **випереджальні тригери за потужністю** (PV, вихідне навантаження, мережа) / *Proactive power triggers activate cooling before heatsinks heat up*.

### 📋 Сутності та помічники (Helpers) у вкладці «Вентилятор» / Entities & Helpers in "Fan" Tab
У дашбордах ([`Dashboard Solar Station.yaml`](Dashboard%20Solar%20Station.yaml) та [`Dashboard Overview.yaml`](Dashboard%20Overview.yaml)) секція кулера (`mdi:fan`) використовує такі компоненти:  
*In dashboards ([`Dashboard Solar Station.yaml`](Dashboard%20Solar%20Station.yaml) and [`Dashboard Overview.yaml`](Dashboard%20Overview.yaml)), the fan section utilizes:*

| Категорія / Category | Сутність / Helper | Тип / Type | Призначення (UA) | Purpose (EN) |
| :--- | :--- | :--- | :--- | :--- |
| **Статус / Status** | `sensor.deye_inverter_ac_temperature` | Sensor | Температура радіатора AC (°C) | AC heatsink temperature |
| **Статус / Status** | `sensor.deye_inverter_dc_temperature` | Sensor | Температура радіатора DC (°C) | DC heatsink temperature |
| **Статус / Status** | `number.inverter_cooling_fan_fan_speed_target` | Number | Поточне завдання швидкості ШІМ (0–100%) | Target PWM fan speed (0–100%) |
| **Температура / Temp** | `input_number.fan_start_speed` | Helper | Стартова швидкість при пуску (замовчування: 20%) | Initial fan speed (default: 20%) |
| **Температура / Temp** | `input_number.fan_hysteresis` | Helper | Гістерезис вимкнення за температурою (замовчування: 2°C) | Temperature hysteresis (default: 2°C) |
| **Температура / Temp** | `input_number.fan_ac_min_temp` | Helper | Температура старту кулера по AC (замовчування: 45°C) | AC start temp threshold (default: 45°C) |
| **Температура / Temp** | `input_number.fan_ac_max_temp` | Helper | Температура 100% обертів по AC (замовчування: 65°C) | AC 100% fan speed temp (default: 65°C) |
| **Температура / Temp** | `input_number.fan_dc_min_temp` | Helper | Температура старту кулера по DC (замовчування: 45°C) | DC start temp threshold (default: 45°C) |
| **Температура / Temp** | `input_number.fan_dc_max_temp` | Helper | Температура 100% обертів по DC (замовчування: 65°C) | DC 100% fan speed temp (default: 65°C) |
| **Температура / Temp** | `input_select.fan_control_law` | Helper | Крива: `Linear`, `Logarithmic`, `Reverse Logarithmic` | Fan response curve |
| **Тригери / Triggers** | `input_boolean.enable_pv_power_trigger` | Helper | Дозвіл пуску за потужністю сонця (PV) | Enable trigger by solar PV generation |
| **Тригери / Triggers** | `input_number.pv_power_threshold` | Helper | Поріг потужності сонця (замовчування: 3000 W) | PV trigger threshold (default: 3000 W) |
| **Тригери / Triggers** | `input_boolean.enable_out_power_trigger` | Helper | Дозвіл пуску за навантаженням інвертора | Enable trigger by inverter output load |
| **Тригери / Triggers** | `input_number.out_power_threshold` | Helper | Поріг вихідної потужності (замовчування: 2000 W) | Load power threshold (default: 2000 W) |
| **Тригери / Triggers** | `input_boolean.enable_grid_power_trigger` | Helper | Дозвіл пуску за потужністю мережі | Enable trigger by grid power |
| **Тригери / Triggers** | `input_number.grid_power_threshold` | Helper | Поріг потужності мережі (замовчування: 2000 W) | Grid power threshold (default: 2000 W) |

> 📖 **Повна покрокова інструкція з кодом / Detailed Guide:**  
> Готовий код створення помічників для `configuration.yaml`, автоматизацію та YAML картки дашборду дивіться у файлі **[`ha_fan_config.md`](ha_fan_config.md)**.  
> *Ready-to-use helper configs, automations, and Lovelace cards are available in [`ha_fan_config.md`](ha_fan_config.md).*

---

## 📂 Структура репозиторію / Repository Structure

```text
├── .gitignore                            # Secrets & local configuration protection
├── secrets.yaml.example                  # Password & keys configuration template
├── YamBMS+Deye Inverter.example.yaml     # Safe template for ESPHome (Deye + YamBMS + Fan)
├── YamBMS BLE.example.yaml               # Safe template for ESPHome YamBMS BLE
├── pv-inverter.Deye-SG0XLP1.yaml         # Base configuration for Deye inverter
├── ha_fan_config.md                      # Detailed guide for fan helpers, automations & dashboard in HA
├── Dashboard Solar Station.yaml          # Lovelace Dashboard for solar station, BMS, inverter & fan
└── Dashboard Overview.yaml               # Alternative Lovelace Dashboard
```

---

## 📜 Ліцензія та подяка / License & Acknowledgments

- Дякуємо [Sleeper85](https://github.com/Sleeper85) за проект **YamBMS** / Thanks to [Sleeper85](https://github.com/Sleeper85) for **YamBMS**.
- Дякуємо [Lewa-Reka](https://github.com/Lewa-Reka) за розробку пакета **esphome-deye-inverter** / Thanks to [Lewa-Reka](https://github.com/Lewa-Reka) for **esphome-deye-inverter**.
- Код поширюється на умовах ліцензії GNU General Public License v3.0 (GPL-3.0) / Distributed under GPL-3.0.

