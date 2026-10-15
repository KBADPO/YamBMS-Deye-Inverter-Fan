# Home Assistant: Керування зовнішніми вентиляторами охолодження / External Cooling Fan Control

Логіка інтелектуального керування зовнішнім охолодженням (вентиляторами) винесена в Home Assistant. Це дозволяє гнучко налаштовувати пороги спрацьовування, криві регулювання обертів (лінійна, логарифмічна) та тригери випереджального охолодження за потужністю без перепрошивки ESP32.  
*Smart external cooling fan control logic is delegated to Home Assistant. This allows flexible tuning of temperature triggers, fan speed curves (linear, logarithmic), and proactive power triggers without reflashing the ESP32.*

---

## 1. Сутності та сенсори / Entity Overview

Усі елементи керування у вкладці **«Вентилятор»** дашборду ([`Dashboard Solar Station.yaml`](Dashboard%20Solar%20Station.yaml)) поділені на три групи:  
*All control elements in the **"Fan"** tab of the dashboard ([`Dashboard Solar Station.yaml`](Dashboard%20Solar%20Station.yaml)) are organized into three groups:*

### 📊 Fan Status (Статус та моніторинг / Status & Monitoring)
| Сутність / Entity | Тип / Type | Опис (UA) | Description (EN) |
| :--- | :--- | :--- | :--- |
| `sensor.deye_inverter_ac_temperature` | Sensor | Температура радіатора AC (інвертор) | Deye inverter AC heatsink temperature |
| `sensor.deye_inverter_dc_temperature` | Sensor | Температура радіатора DC (інвертор) | Deye inverter DC heatsink temperature |
| `number.inverter_cooling_fan_fan_speed_target` | Number | Цільова швидкість обдуву ШІМ (0–100%), передається в ESPHome *(може мати ID `number.deye_yambms_fan_fan_speed_target`)* | PWM fan target speed (0–100%) sent to ESPHome *(may also be named `number.deye_yambms_fan_fan_speed_target`)* |

### 🌡️ Temperature Controls (Температурні пороги та крива обдуву / Temperature Controls & Fan Curve)
| Помічник / Helper | Тип / Type | Замовчування / Default | Діапазон / Range | Опис (UA) | Description (EN) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `input_number.fan_start_speed` | Number | `20 %` | 0 – 50 % (step 1) | Стартова (мінімальна) швидкість кулера при пуску | Initial (minimum) fan speed upon trigger activation |
| `input_number.fan_hysteresis` | Number | `2 °C` | 1 – 5 °C (step 1) | Температурний гістерезис для запобігання деренчанню | Temperature hysteresis to prevent continuous cycling |
| `input_number.fan_ac_min_temp` | Number | `45 °C` | 30 – 70 °C (step 1) | Початкова температура старту кулера по AC | AC temperature threshold to start fan |
| `input_number.fan_ac_max_temp` | Number | `65 °C` | 40 – 90 °C (step 1) | Температура виходу на 100% обертів по AC | AC temperature for 100% fan speed |
| `input_number.fan_dc_min_temp` | Number | `45 °C` | 30 – 70 °C (step 1) | Початкова температура старту кулера по DC | DC temperature threshold to start fan |
| `input_number.fan_dc_max_temp` | Number | `65 °C` | 40 – 90 °C (step 1) | Температура виходу на 100% обертів по DC | DC temperature for 100% fan speed |
| `input_select.fan_control_law` | Select | `Linear` | `Linear`, `Logarithmic`, `Reverse Logarithmic` | Крива зміни обертів (лінійна, логарифмічна або зворотна) | Fan speed response curve |

### ⚡ Power Triggers (Випереджальні тригери за потужністю / Proactive Power Triggers)
Дозволяють вмикати обдув заздалегідь при високих навантаженнях, не чекаючи значного нагріву радіатора:  
*Allow proactive cooling activation under high loads before heatsinks overheat:*

| Помічник / Helper | Тип / Type | Замовчування / Default | Діапазон / Range | Опис (UA) | Description (EN) |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `input_boolean.enable_pv_power_trigger` | Boolean | `off` | on / off | Увімкнення тригера за генерацією PV1 + PV2 | Enable trigger based on PV solar power |
| `input_number.pv_power_threshold` | Number | `3000 W` | 1000 – 10000 W (step 100) | Поріг сонячної генерації для увімкнення | Solar PV power threshold to activate fan |
| `input_boolean.enable_out_power_trigger` | Boolean | `off` | on / off | Увімкнення тригера за вихідною потужністю | Enable trigger based on inverter load power |
| `input_number.out_power_threshold` | Number | `2000 W` | 1000 – 4000 W (step 100) | Поріг навантаження для увімкнення | Inverter load power threshold to activate fan |
| `input_boolean.enable_grid_power_trigger` | Boolean | `off` | on / off | Увімкнення тригера за потужністю мережі | Enable trigger based on grid import/export power |
| `input_number.grid_power_threshold` | Number | `2000 W` | 1000 – 4000 W (step 100) | Поріг потужності мережі для увімкнення | Grid power threshold to activate fan |

---

## 2. Створення помічників / Helper Configuration

### Варіант А: Через файл `configuration.yaml` / Option A: Via `configuration.yaml`
Додайте наступний блок у ваш `configuration.yaml` та перезавантажте Home Assistant:  
*Add the following block to your `configuration.yaml` and reload Home Assistant:*

```yaml
input_number:
  fan_ac_min_temp:
    name: "Fan AC Start Temperature"
    min: 30
    max: 70
    step: 1
    unit_of_measurement: "°C"
    icon: mdi:thermometer-low
  fan_ac_max_temp:
    name: "Fan AC Max Speed Temperature"
    min: 40
    max: 90
    step: 1
    unit_of_measurement: "°C"
    icon: mdi:thermometer-high
  fan_dc_min_temp:
    name: "Fan DC Start Temperature"
    min: 30
    max: 70
    step: 1
    unit_of_measurement: "°C"
    icon: mdi:thermometer-low
  fan_dc_max_temp:
    name: "Fan DC Max Speed Temperature"
    min: 40
    max: 90
    step: 1
    unit_of_measurement: "°C"
    icon: mdi:thermometer-high
  fan_hysteresis:
    name: "Fan Hysteresis"
    min: 1
    max: 5
    step: 1
    unit_of_measurement: "°C"
    icon: mdi:arrow-expand-vertical
  fan_start_speed:
    name: "Fan Initial Speed"
    min: 0
    max: 50
    step: 1
    unit_of_measurement: "%"
    icon: mdi:fan-speed-1
  pv_power_threshold:
    name: "PV Power Fan Trigger Threshold"
    min: 1000
    max: 10000
    step: 100
    unit_of_measurement: "W"
    icon: mdi:solar-power
  out_power_threshold:
    name: "Output Power Fan Trigger Threshold"
    min: 1000
    max: 4000
    step: 100
    unit_of_measurement: "W"
    icon: mdi:flash
  grid_power_threshold:
    name: "Grid Power Fan Trigger Threshold"
    min: 1000
    max: 4000
    step: 100
    unit_of_measurement: "W"
    icon: mdi:transmission-tower

input_boolean:
  enable_pv_power_trigger:
    name: "Enable PV Power Fan Trigger"
    icon: mdi:solar-power
  enable_out_power_trigger:
    name: "Enable Output Power Fan Trigger"
    icon: mdi:home-lightning-bolt
  enable_grid_power_trigger:
    name: "Enable Grid Power Fan Trigger"
    icon: mdi:transmission-tower

input_select:
  fan_control_law:
    name: "Fan Control Law"
    icon: mdi:chart-bell-curve
    options:
      - "Linear"
      - "Logarithmic"
      - "Reverse Logarithmic"
```

### Варіант Б: Через графічний інтерфейс (UI) / Option B: Via GUI
Перейдіть у **Налаштування** ➔ **Пристрої та служби** ➔ вкладка **Помічники** (Helpers) ➔ **+ Створити помічника**:  
*Navigate to **Settings** ➔ **Devices & Services** ➔ **Helpers** tab ➔ **+ Create Helper**:*
1. **Число / Number** (`input_number`): вкажіть відповідне ім'я, мінімум, максимум, крок та одиницю вимірювання.
2. **Перемикач / Toggle** (`input_boolean`): для тригерів потужності.
3. **Випадаючий список / Dropdown** (`input_select`): ім'я `Fan Control Law`, опції: `Linear`, `Logarithmic`, `Reverse Logarithmic`.

---

## 3. Автоматизація розрахунку швидкості обдуву / Smart Fan Speed Automation

Створіть нову автоматизацію (**Налаштування** ➔ **Автоматизації та сцени** ➔ **Створити автоматизацію**), натисніть три крапки у верхньому правому куті, оберіть **«Редагувати в YAML»** та збережіть наступний код:  
*Create a new automation (**Settings** ➔ **Automations & Scenes** ➔ **Create Automation**), click three dots in top right, select **"Edit in YAML"** and paste:*

```yaml
alias: "Inverter Fan Smart Control"
description: "Smart cooling fan speed calculation based on AC/DC temperatures and power triggers / Розрахунок швидкості обдуву за температурами AC/DC або тригерами потужності"
mode: restart
trigger:
  - platform: time_pattern
    seconds: "/5"
action:
  - service: number.set_value
    target:
      # Change to number.deye_yambms_fan_fan_speed_target if needed / За необхідності змініть відповідно до вашого ESPHome
      entity_id: number.inverter_cooling_fan_fan_speed_target
    data:
      value: >
        {% set t_ac = states('sensor.deye_inverter_ac_temperature') | float(0) %}
        {% set t_dc = states('sensor.deye_inverter_dc_temperature') | float(0) %}
        {% set pv_pwr = states('sensor.deye_inverter_pv1_power') | float(0) + states('sensor.deye_inverter_pv2_power') | float(0) %}
        {% set out_pwr = states('sensor.deye_inverter_output_power') | float(0) %}
        {% set grid_pwr = states('sensor.deye_inverter_grid_power') | float(0) | abs %}
        
        {% set ac_start = states('input_number.fan_ac_min_temp') | float(45) %}
        {% set ac_max = states('input_number.fan_ac_max_temp') | float(65) %}
        {% set dc_start = states('input_number.fan_dc_min_temp') | float(45) %}
        {% set dc_max = states('input_number.fan_dc_max_temp') | float(65) %}
        {% set hyst = states('input_number.fan_hysteresis') | float(2) %}
        {% set min_duty = states('input_number.fan_start_speed') | float(20) %}
        
        {% set pv_trig = states('input_boolean.enable_pv_power_trigger') == 'on' and pv_pwr >= states('input_number.pv_power_threshold') | float(3000) %}
        {% set out_trig = states('input_boolean.enable_out_power_trigger') == 'on' and out_pwr >= states('input_number.out_power_threshold') | float(2000) %}
        {% set grid_trig = states('input_boolean.enable_grid_power_trigger') == 'on' and grid_pwr >= states('input_number.grid_power_threshold') | float(2000) %}
        
        {% set power_trigger = pv_trig or out_trig or grid_trig %}
        
        {% set law = states('input_select.fan_control_law') %}
        
        {# AC normalized deviation / Розрахунок нормованого відхилення AC #}
        {% set ac_x = 1.0 if t_ac >= ac_max else (0.0 if t_ac <= ac_start else (t_ac - ac_start) / (ac_max - ac_start)) %}
        {% set ac_fx = log(1 + 9 * ac_x, 10) if law == 'Logarithmic' else (1 - log(1 + 9 * (1 - ac_x), 10) if law == 'Reverse Logarithmic' else ac_x) %}
        {% set ac_duty = min_duty if t_ac <= ac_start else min_duty + (100 - min_duty) * ac_fx %}
        
        {# DC normalized deviation / Розрахунок нормованого відхилення DC #}
        {% set dc_x = 1.0 if t_dc >= dc_max else (0.0 if t_dc <= dc_start else (t_dc - dc_start) / (dc_max - dc_start)) %}
        {% set dc_fx = log(1 + 9 * dc_x, 10) if law == 'Logarithmic' else (1 - log(1 + 9 * (1 - dc_x), 10) if law == 'Reverse Logarithmic' else dc_x) %}
        {% set dc_duty = min_duty if t_dc <= dc_start else min_duty + (100 - min_duty) * dc_fx %}
        
        {% set temp_duty = [ac_duty, dc_duty] | max %}
        
        {% set temp_over_start = t_ac >= ac_start or t_dc >= dc_start %}
        {% set temp_below_hyst = t_ac < (ac_start - hyst) and t_dc < (dc_start - hyst) %}
        
        {% set current_duty = states('number.inverter_cooling_fan_fan_speed_target') | float(0) %}
        {% set is_active = current_duty > 0 %}
        
        {% set new_active = (is_active and not (not power_trigger and temp_below_hyst)) or (not is_active and (power_trigger or temp_over_start)) %}
        
        {% if new_active %}
          {% set duty = [min_duty, temp_duty] | max %}
          {{ [0, [100, duty] | min] | max | round(0) }}
        {% else %}
          0
        {% endif %}
```

---

## 4. Картка Lovelace для вкладки «Вентилятор» / Lovelace Fan Tab Card

Код окремої вкладки для вашого Lovelace Dashboard (вид `sections`), аналогічний реалізованому у [`Dashboard Solar Station.yaml`](Dashboard%20Solar%20Station.yaml):  
*Lovelace Dashboard tab configuration (`sections` view), identical to [`Dashboard Solar Station.yaml`](Dashboard%20Solar%20Station.yaml):*

```yaml
  # - title: Fan
  - icon: mdi:fan
    type: sections
    max_columns: 3
    sections:
      - type: grid
        cards:
          - type: heading
            heading: Fan Status
          - type: tile
            entity: sensor.deye_inverter_ac_temperature
            name: 'AC Температура'
            vertical: false
            features_position: bottom
          - type: tile
            entity: sensor.deye_inverter_dc_temperature
            name: 'DC Температура'
            vertical: false
            features_position: bottom
          - type: history-graph
            entities:
              - number.inverter_cooling_fan_fan_speed_target
            hours_to_show: 24
            show_names: true
            logarithmic_scale: false
            fit_y_data: true
      - type: grid
        cards:
          - type: heading
            heading: Temperature Controls
          - type: tile
            entity: input_number.fan_start_speed
            vertical: false
            features_position: bottom
          - type: tile
            entity: input_number.fan_hysteresis
            vertical: false
            features_position: bottom
          - type: tile
            entity: input_number.fan_ac_min_temp
            vertical: false
            features_position: bottom
          - type: tile
            entity: input_number.fan_ac_max_temp
            vertical: false
            features_position: bottom
          - type: tile
            entity: input_number.fan_dc_min_temp
            vertical: false
            features_position: bottom
          - type: tile
            entity: input_number.fan_dc_max_temp
            vertical: false
            features_position: bottom
          - type: tile
            entity: input_select.fan_control_law
            vertical: false
            features_position: bottom
      - type: grid
        cards:
          - type: heading
            heading: Power Triggers
          - type: tile
            entity: input_boolean.enable_pv_power_trigger
            vertical: false
            features_position: bottom
          - type: tile
            entity: input_number.pv_power_threshold
            vertical: false
            features_position: bottom
          - type: tile
            entity: input_boolean.enable_out_power_trigger
            vertical: false
            features_position: bottom
          - type: tile
            entity: input_number.out_power_threshold
            vertical: false
            features_position: bottom
          - type: tile
            entity: input_boolean.enable_grid_power_trigger
            vertical: false
            features_position: bottom
          - type: tile
            entity: input_number.grid_power_threshold
            vertical: false
            features_position: bottom
```
*(Примітка: якщо вам потрібно приховати налаштування від звичайних користувачів, додайте у відповідні картки секцію `visibility` з фільтром користувачів `condition: user` / Note: to hide controls from non-admin users, add a `visibility` block with `condition: user`).*
