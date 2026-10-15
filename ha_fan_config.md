# Home Assistant Fan Logic Configuration

Оскільки ми перенесли логіку в Home Assistant, вам потрібно додати ці налаштування у ваші конфігураційні файли HA (або створити їх через графічний інтерфейс).

## 1. Створення "Помічників" (Helpers)
Додайте цей код у ваш файл `configuration.yaml` (або створіть їх вручну через Налаштування -> Пристрої та служби -> Помічники):

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
  fan_start_speed:
    name: "Fan Initial Speed"
    min: 0
    max: 50
    step: 1
    unit_of_measurement: "%"
  pv_power_threshold:
    name: "PV Power Fan Trigger Threshold"
    min: 1000
    max: 10000
    step: 100
    unit_of_measurement: "W"
  out_power_threshold:
    name: "Output Power Fan Trigger Threshold"
    min: 1000
    max: 4000
    step: 100
    unit_of_measurement: "W"
  grid_power_threshold:
    name: "Grid Power Fan Trigger Threshold"
    min: 1000
    max: 4000
    step: 100
    unit_of_measurement: "W"

input_boolean:
  enable_pv_power_trigger:
    name: "Enable PV Power Fan Trigger"
  enable_out_power_trigger:
    name: "Enable Output Power Fan Trigger"
  enable_grid_power_trigger:
    name: "Enable Grid Power Fan Trigger"

input_select:
  fan_control_law:
    name: "Fan Control Law"
    options:
      - "Linear"
      - "Logarithmic"
      - "Reverse Logarithmic"
```

## 2. Автоматизація
Створіть нову автоматизацію, натисніть "Редагувати в YAML" і вставте цей код (він вже оптимізований для графічного інтерфейсу):

```yaml
alias: "Inverter Fan Smart Control"
mode: restart
trigger:
  - platform: time_pattern
    seconds: "/5"
action:
  - service: number.set_value
    target:
      entity_id: number.deye_yambms_fan_fan_speed_target
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
        
        {% set ac_x = 1.0 if t_ac >= ac_max else (0.0 if t_ac <= ac_start else (t_ac - ac_start) / (ac_max - ac_start)) %}
        {% set ac_fx = log(1 + 9 * ac_x, 10) if law == 'Logarithmic' else (1 - log(1 + 9 * (1 - ac_x), 10) if law == 'Reverse Logarithmic' else ac_x) %}
        {% set ac_duty = min_duty if t_ac <= ac_start else min_duty + (100 - min_duty) * ac_fx %}
        
        {% set dc_x = 1.0 if t_dc >= dc_max else (0.0 if t_dc <= dc_start else (t_dc - dc_start) / (dc_max - dc_start)) %}
        {% set dc_fx = log(1 + 9 * dc_x, 10) if law == 'Logarithmic' else (1 - log(1 + 9 * (1 - dc_x), 10) if law == 'Reverse Logarithmic' else dc_x) %}
        {% set dc_duty = min_duty if t_dc <= dc_start else min_duty + (100 - min_duty) * dc_fx %}
        
        {% set temp_duty = [ac_duty, dc_duty] | max %}
        
        {% set temp_over_start = t_ac >= ac_start or t_dc >= dc_start %}
        {% set temp_below_hyst = t_ac < (ac_start - hyst) and t_dc < (dc_start - hyst) %}
        
        {% set current_duty = states('number.deye_yambms_fan_fan_speed_target') | float(0) %}
        {% set is_active = current_duty > 0 %}
        
        {% set new_active = (is_active and not (not power_trigger and temp_below_hyst)) or (not is_active and (power_trigger or temp_over_start)) %}
        
        {% if new_active %}
          {% set duty = [min_duty, temp_duty] | max %}
          {{ [0, [100, duty] | min] | max | round(0) }}
        {% else %}
          0
        {% endif %}
```
