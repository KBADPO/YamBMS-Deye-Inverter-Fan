# Керівництво з адаптації інвертора, BMS та ізоляції інтерфейсів
# Inverter, BMS Customization & Galvanic Isolation Guide

Цей документ містить детальну інструкцію щодо того, **в яких саме файлах** та **які рядки коду** потрібно змінити для адаптації проекту під іншу модель інвертора, іншу модель BMS, дротовий або бездротовий тип підключення, а також правила обов'язкової **гальванічної ізоляції** для захисту обладнання.

*This guide provides comprehensive instructions on **which files** and **exact lines** to edit when customizing this project for different inverter models, BMS units, wired or wireless connections, as well as mandatory **galvanic isolation rules** to protect hardware.*

---

## 🗺️ Карта файлів проекту / Project File Map

| Файл / File | Призначення (UA) | Purpose (EN) |
| :--- | :--- | :--- |
| [`YamBMS+Deye Inverter.example.yaml`](YamBMS+Deye%20Inverter.example.yaml) | **Головний файл прошивки**. Тут налаштовуються: модель інвертора, пакети BMS, тип шини (CAN/RS485), напруги та струми АКБ, піни вентилятора. | **Main firmware file**. Configures inverter model, BMS packages, communication bus (CAN/RS485), battery voltages/currents, and fan pins. |
| [`secrets.yaml.example`](secrets.yaml.example) | Шаблон приватних ключів: Wi-Fi credentials, Home Assistant API key, OTA пароль. | Template for private keys: Wi-Fi credentials, HA API key, OTA password. |
| [`ha_fan_config.md`](ha_fan_config.md) | Налаштування сутностей, помічників та автоматизації кулера в Home Assistant. | Guide for Home Assistant fan entities, helpers, and automation. |
| [`Dashboard Solar Station.yaml`](Dashboard%20Solar%20Station.yaml) | Готовий інтерфейс Lovelace для моніторингу Deye, BMS та вентилятора. | Complete Lovelace UI dashboard for Deye, BMS telemetry, and fan control. |

---

## 1. ⚡ Адаптація під інший інвертор / Customizing Inverter

Комунікація з інвертором складається з двох незалежних ліній:
1. **Канал батареї (BMS Port)**: передача даних про акумулятор (напруги, струми заряду/розряду, SOC) по протоколу Pylontech (CAN або RS485).
2. **Канал телеметрії інвертора (Modbus Port)**: читання показників інвертора (потужність PV, мережа, навантаження, температури) по протоколу Modbus RTU (RS485).

Файл для редагування: **[`YamBMS+Deye Inverter.example.yaml`](YamBMS+Deye%20Inverter.example.yaml)**

---

### Варіант А: Інша модель Deye (3-фазний або високовольтний) / Other Deye Models

У секції `packages: pv_inverter:` (рядки ~175–183) змініть назву підвантажуваного файлу:

```yaml
packages:
  pv_inverter:
    url: https://github.com/KBADPO/esphome-deye-inverter
    ref: main
    refresh: 12h
    # Оберіть ОДИН файл відповідно до вашої моделі / Select ONE file for your model:
    files: pv_inverter/deye_hybrid_1p_lv.yaml   # Deye 1-фазний низьковольтний 48V (SG01/02/03/04/05LP1) [За замовчуванням]
    # files: pv_inverter/deye_hybrid_3p_lv.yaml # Deye 3-фазний низьковольтний 48V (SG04LP3: 5kW - 12kW)
    # files: pv_inverter/deye_hybrid_3p_hv.yaml # Deye 3-фазний високовольтний HV (SG01/04HP3)
    # files: pv_inverter/deye_string_1p.yaml    # Deye мережевий 1-фазний стрінговий (без АКБ)
    # files: pv_inverter/deye_string_3p.yaml    # Deye мережевий 3-фазний стрінговий (без АКБ)
```

---

### Варіант Б: Інвертор іншого виробника / Non-Deye Inverters
*(Victron GX, Growatt, Solis, Sofar, GoodWe, Luxpower, Megarevo, Voltronic, SMA тощо)*

Практично всі сучасні гібридні інвертори підтримують протокол акумуляторів **Pylontech (CAN 500kbps)** на своєму BMS-порті.

#### 1. Канал батареї (BMS CAN або RS485):
- **CAN-порт (Рекомендовано)**: У файлі конфігурації за замовчуванням увімкнено блок `packages/yambms/yambms_canbus.yaml` (рядок ~318). Він уже транслює стандартні CAN-кадри Pylontech (`0x351`, `0x355`, `0x356`, `0x359`, `0x35E`).
  - В меню вашого інвертора (наприклад, у Growatt, Solis чи Victron) у полі протоколу батареї виберіть **CAN / Pylontech / PYLON / 00**.
  - Якщо інвертор не надсилає відповідь ACK (`0x305`), встановіть:
    ```yaml
    canbus_link_timer_enabled: 'false'
    ```
- **RS485-порт батареї** *(якщо інвертор не має CAN, наприклад, Voltronic Axpert, MPP Solar)*:
  - Закоментуйте блок `yambms_canbus.yaml` (рядок ~318) та розкоментуйте `yambms_rs485_pylon.yaml` (рядок ~335):
    ```yaml
    # - path: 'packages/yambms/yambms_canbus.yaml'
        - path: 'packages/yambms/yambms_rs485_pylon.yaml'
          vars:
            rs485_id: '1'
            rs485_name: 'RS485 Pylon'
            rs485_uart_id: 'uart_esp_1'
            rs485_uart_baud_rate: '9600'
    ```

#### 2. Вимкнення телеметрії Deye:
Якщо у вас інвертор не Deye, видаліть або закоментуйте секцію:
```yaml
# packages:
#   pv_inverter:
#     ...
```
а також видаліть або призначте на інші цілі блок `uart:` та `modbus:` (рядки ~383–389), або підключіть сторонній пакет ESPHome для вашого інвертора (наприклад, `esphome-growatt`).

---

## 2. 🔋 Адаптація під іншу BMS або Multi-BMS / BMS Customization

YamBMS підтримує одночасне читання та об'єднання даних від однієї або кількох BMS.

Файл для редагування: **[`YamBMS+Deye Inverter.example.yaml`](YamBMS+Deye%20Inverter.example.yaml)**  
Розділ: `# >>> BMS / Balancer / Shunt` (рядки ~280–304).

---

### Приклади конфігурацій різних моделей BMS:

#### 1. JK-BMS (Bluetooth / BLE) — встановлено за замовчуванням
```yaml
      - path: 'packages/bms/bms_combine_JK_BLE_standard.yaml'
        vars:
          bms_id: '1'                         # Порядковий номер (1, 2, 3...)
          bms_name: 'JK-BMS 1'                # Назва сутності в Home Assistant
          bms_ble_protocol_version: 'JK02_32S'# 'JK02_32S' (версія HW >= 11) або 'JK02_24S' (HW < 11)
          bms_ble_mac_address: 'c8:47:80:1b:78:d7' # Ваша Bluetooth MAC адреса
          bms_cell_max_cycles: '3000.0'       # Ресурс циклів комірок для розрахунку SoH
```

#### 2. JK-BMS через кабель (RS485 / Modbus)
```yaml
      - path: 'packages/bms/bms_combine_JK_RS485.yaml'
        vars:
          bms_id: '1'
          bms_name: 'JK-BMS RS485'
          bms_uart_id: 'uart_esp_1'
          bms_modbus_address: '0x01'
          bms_cell_max_cycles: '3000.0'
```

#### 3. Daly BMS (RS485 / UART або CAN)
- **RS485 / UART**:
  ```yaml
        - path: 'packages/bms/bms_combine_Daly_RS485.yaml'
          vars:
            bms_id: '1'
            bms_name: 'Daly BMS 1'
            bms_uart_id: 'uart_esp_1'
            bms_modbus_address: '0x01'
            bms_cell_max_cycles: '3000.0'
  ```
- **CAN Bus**:
  ```yaml
        - path: 'packages/bms/bms_combine_Daly_CAN.yaml'
          vars:
            bms_id: '1'
            bms_name: 'Daly CAN BMS'
            canbus_node_id: 'canbus_bms_1'
  ```

#### 4. JBD / Xiaoxiang / Overkill Solar (BLE або UART)
- **Через Bluetooth (BLE)**:
  ```yaml
        - path: 'packages/bms/bms_combine_JBD_BLE.yaml'
          vars:
            bms_id: '1'
            bms_name: 'JBD BMS 1'
            bms_ble_mac_address: 'AA:BB:CC:DD:EE:FF'
            bms_cell_max_cycles: '3000.0'
  ```
- **Через кабель (UART)**:
  ```yaml
        - path: 'packages/bms/bms_combine_JBD_UART.yaml'
          vars:
            bms_id: '1'
            bms_name: 'JBD BMS 1'
            bms_uart_id: 'uart_esp_1'
  ```

#### 5. Seplos BMS (RS485 або CAN)
```yaml
      - path: 'packages/bms/bms_combine_Seplos_RS485.yaml'
        vars:
          bms_id: '1'
          bms_name: 'Seplos BMS'
          bms_uart_id: 'uart_esp_1'
          bms_modbus_address: '0x00'
          bms_cell_max_cycles: '6000.0'
```

#### 6. Victron SmartShunt / BMV (BLE або VE.Direct)
```yaml
      - path: 'packages/shunt/shunt_combine_Victron_SmartShunt_BLE.yaml'
        vars:
          shunt_id: '1'
          shunt_name: 'SmartShunt'
          shunt_ble_mac_address: 'AA:BB:CC:DD:EE:FF'
```

---

### 📦 Як об'єднати декілька батарей (Multi-BMS Aggregation)
Щоб об'єднати 2, 3 або більше акумуляторних збірок, просто додайте їх послідовно одна за одною.

> [!IMPORTANT]
> **Головне правило**: Значення `bms_id` має бути строго послідовним числовим значенням, починаючи з `1` (`1`, `2`, `3`...).

```yaml
      # Батарея 1 / Pack 1
      - path: 'packages/bms/bms_combine_JK_BLE_standard.yaml'
        vars:
          bms_id: '1'
          bms_name: 'Pack 1'
          bms_ble_protocol_version: 'JK02_32S'
          bms_ble_mac_address: 'c8:47:80:1b:78:d7'
          bms_cell_max_cycles: '3000.0'

      # Батарея 2 / Pack 2
      - path: 'packages/bms/bms_combine_JK_BLE_standard.yaml'
        vars:
          bms_id: '2'
          bms_name: 'Pack 2'
          bms_ble_protocol_version: 'JK02_32S'
          bms_ble_mac_address: 'c8:47:80:2a:44:f1'
          bms_cell_max_cycles: '3000.0'
```
**Що робить YamBMS**:
- Підсумовує загальну ємність (Ah) та струми.
- Обчислює середньозважений заряд (SOC).
- Відстежує екстремуми: найвищу та найнижчу напругу комірки серед **усіх** батарей.
- Автоматично знижує струм заряду/розряду (CCL/DCL), якщо хоча б одна збірка нагрілася або наблизилася до верхнього порогу відсічки.

---

## 3. ⚙️ Базові параметри акумулятора / Battery Substitutions

На початку файлу [`YamBMS+Deye Inverter.example.yaml`](YamBMS+Deye%20Inverter.example.yaml) у секції `substitutions:` обов'язково налаштуйте параметри своєї батареї:

| Параметр / Parameter | Приклад (16S LiFePO4) | Опис (UA) | Description (EN) |
| :--- | :--- | :--- | :--- |
| `yambms_battery_chemistry` | `'1'` | Хімія: `1` = LiFePO4 (LFP), `2` = Li-ion, `3` = LTO | Battery chemistry: 1=LFP, 2=Li-ion, 3=LTO |
| `yambms_cell_count` | `'16'` | Кількість послідовних комірок (16S, 15S, 8S тощо) | Number of series cells (e.g. 16S, 15S) |
| `yambms_bulk_v` | `'55.5'` | Напруга основного заряду Bulk/Absorption (3.468 В/ком.) | Absorption charge target voltage |
| `yambms_float_v` | `'53.6'` | Напруга буферного режиму Float (3.35 В/ком.) | Float maintain voltage |
| `yambms_rebulk_v` | `'52.8'` | Напруга перезапуску циклу заряду Rebulk (3.30 В/ком.) | Rebulk trigger voltage threshold |
| `yambms_max_requested_charge_current` | `'150'` | Максимальний сумарний струм заряду (Ампери) | Max allowable total charge current (A) |
| `yambms_max_requested_discharge_current` | `'150'` | Максимальний сумарний струм розряду (Ампери) | Max allowable total discharge current (A) |

---

## 4. ⚠️ КРИТИЧНО: Гальванічна ізоляція інтерфейсів / Galvanic Isolation

> [!CAUTION]
> **НЕБЕЗПЕКА ПОШКОДЖЕННЯ ОБЛАДНАННЯ! / RISK OF HARDWARE DESTRUCTION!**  
> При одночасному **дротовому підключенні** контролера ESP32 до інвертора (Modbus RS485 / CAN) та до BMS (RS485 / UART / CAN) категорично заборонено об'єднувати їхні «землі» (GND) напряму без **гальванічної ізоляції**!

### Чому виникає небезпека (Ground Loop & Floating Potentials)?
1. **Комутація по мінусу (Low-Side Switching)**: Більшість BMS для збірок 48V (JK, Daly, JBD, Ant) встановлюються в розрив **мінусового провідника** батареї (B- та P-). Силові MOSFET знаходяться на мінусовій шині.
2. **Різниця потенціалів**:
   - У нормальному стані під час високого струму на ключах BMS виникає падіння напруги.
   - У разі спрацювання захисту BMS (захист по перезаряду, перерозряду або КЗ) силові транзистори розмикаються. Між клемою `B-` (земля BMS) та клемою `P-` (земля інвертора) миттєво з'являється **повна напруга всієї батареї (від 48V до 60V+)**!
3. **Вигорання обладнання**: Якщо земляні контакти BMS та інвертора з'єднані через спільну земляну шину плати ESP32, струм зворотної полярності піде через сигнальні лінії та плату ESP32. Це призводить до **миттєвого вигорання портів ESP32, мікросхем трансиверів, порту BMS інвертора або процесора самої BMS**!

---

### Схема безпечних підключень / Safe Connection Architecture:

#### Сценарій 1: BLE (Bluetooth) + Дротовий зв'язок з інвертором *(Конфігурація автора)*
- **Статус**: ✅ **100% Безпечно**.
- **Причина**: Радіоканал Bluetooth має абсолютну фізичну гальванічну розв'язку («по повітрю»). Земля інвертора та земля BMS електрично повністю ізольовані одна від одної.

```text
[ BMS Акумулятора ] <· · · Bluetooth (По повітрю / Air Gap) · · ·> [ ESP32-S3 ] === Дріт (UTP) ===> [ Інвертор Deye ]
```

---

#### Сценарій 2: Дротовий зв'язок з інвертором + Дротовий зв'язок з BMS (RS485 / CAN)
- **Статус**: ⚠️ **Вимагає обов'язкової апаратної гальванічної ізоляції!**

```text
[ BMS ] == (Лінія А) ==> [ Гальванічний ізолятор ] ──> [ ESP32-S3 ] ──> [ Гальванічний ізолятор ] == (Лінія Б) ==> [ Інвертор ]
                                                           │
                                             (Живлення від ізольованого DC-DC)
```

#### Необхідні компоненти захисту при дротовому Multi-Connect:
1. **Ізольоване живлення**:
   - Використовуйте модулі ізольованих DC-DC перетворювачів серії **B0505S-1W** (5V ➔ 5V isolation) або **B1205S** для розв'язки ліній живлення перетворювачів.
2. **Гальванічно ізольовані трансивери RS485**:
   - Замість звичайного модуля HW-0519/MAX485 використовуйте модулі з вбудованим ізолятором: на базі **ADM2483**, **ISO3082** або швидких цифрових ізоляторів **ADuM1201 / ADuM1411**.
3. **Гальванічно ізольовані трансивери CAN**:
   - Використовуйте модулі на базі **ISO1050**, **ADM3053** або ізольовані модулі CAN з оптопарами (наприклад, CTM1051KT).
4. **Правило роздільних земель**:
   - Сигнальна земля `GND_ISO` з боку інвертора має з'єднуватися **тільки** з піном GND інвертора.
   - Сигнальна земля `GND_BMS` має з'єднуватися **тільки** з портом BMS.
   - Жодна з них не повинна з'єднуватися напряму з локальною землею `GND` контролера ESP32 без ізолюючого бар'єра!

---

## 📌 Підсумок налаштування / Quick Customization Checklist

- [ ] Визначили модель інвертора ➔ перевірили рядок `files:` у `packages: pv_inverter:`.
- [ ] Визначили модель BMS ➔ обрали відповідний пакет у секції `# >>> BMS / Balancer / Shunt`.
- [ ] Якщо підключення BMS по BLE ➔ вказали MAC-адресу та версію протоколу (`bms_ble_protocol_version`).
- [ ] Якщо підключення BMS дротове ➔ переконалися у наявності **гальванічної ізоляції** між портами!
- [ ] Перевірили та виставили напруги своєї батареї у `substitutions:` (`yambms_cell_count`, `bulk_v`, `float_v`).
- [ ] Скомпілювали прошивку в **ESPHome Dashboard** або через CLI.
