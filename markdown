{% if (states.sensor.average_available_ampere_on_grid_over_last_4_hours.state|int(default=0) * 3 * 230 / 1000) >= 3 %}
  {% set charge_kw = (states.sensor.average_available_ampere_on_grid_over_last_4_hours.state|int(default=0) * 3 * 230 / 1000) %}
{% else %}
  {% set charge_kw = 3 %}
{% endif %}
{% set battery_kwh = 69 %}
{% set charge_limit = states.sensor.tesla_charge_limit_soc.state|int(default=0) %} 
{% set current_soc = states.sensor.tesla_battery_level.state|int(default=0) %}
{% set location = states.sensor.tesla_geofence.state %}
{% set car_type = states.input_select.electric_car_type.state %}
{% set tesla_cable_status = states.binary_sensor.tesla_plugged_in.state %}
{% if states.sensor.easee_home_44981_status.state != 'disconnected' %}
  {% set easee_cable_status = 'on' %}
{% else %}
  {% set easee_cable_status = 'off' %}
{% endif %}

{% if (location == "Landet" and tesla_cable_status == "on") %}
  {% set charge_hours = ((charge_limit - current_soc ) / 100 * battery_kwh / charge_kw)|round(0,'ceil')|int(default=0) %}
  {% set charge_kwh =  ((charge_limit - current_soc ) / 100 * battery_kwh)|float|round(2) %}
{% elif (easee_cable_status == "on" and tesla_cable_status == "off") %}
  {% if car_type == 'Elbil' %}
    {% set charge_kwh = 40 %}    
    {% set charge_hours = (charge_kwh / charge_kw)|round(0,'ceil')|int(default=0) %}
  {% elif car_type == 'Hybrid' %}
    {% set charge_kwh = 5 %}
    {% set charge_kw = 2 %}
    {% set charge_hours = (charge_kwh / charge_kw)|round(0,'ceil')|int(default=0) %}
  {% endif %}
{% else %}
  {% set charge_hours = 0 %}
{% endif %}

{% if (charge_hours != 0) %}
  Need to charge {{ charge_kwh }} kWh in {{ charge_hours }} hours (Avg: {{charge_kw}} kW)
  
  Charge during:
  
  {%- set data = 
  state_attr('sensor.nordpool_kwh_se3_sek_3_10_025', 'raw_today')|selectattr('start', '>=', ((now()-timedelta(hours=1))))|sort(attribute='value') +
  state_attr('sensor.nordpool_kwh_se3_sek_3_10_025', 'raw_tomorrow')[:8]|rejectattr('value', 'eq', None)|list|sort(attribute='value')
  %}
  
  {%- set l=data|sort(attribute='value') %} 
  {%- set ns = namespace(list_of_hours=[]) %}

  {%- if l|list|count < charge_hours %}
    {%- set range_hours = l|list|count %}
  {%- else %}
    {%- set range_hours = charge_hours %}
  {%- endif %}

  {%- for i in range(range_hours) %}
    {%- if now()-timedelta(hours=1) <= l[i].start %}  
      {%- set ns.list_of_hours = ns.list_of_hours + [(l[i].start|string)|as_timestamp|timestamp_custom("%Y-%m-%d %H:%M", True, "%Y-%m-%d %H:%M")] %}
    {%- endif %}
  {%- endfor %}

  {%- for hour in ns.list_of_hours | sort %}
    {{ hour }} - {{ (as_timestamp(hour) + 3600)|timestamp_custom("%Y-%m-%d %H:%M", True, "%Y-%m-%d %H:%M") }}  
  {%- endfor %}
{% else %}
  Will not charge.
{% endif %}
