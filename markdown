{% set charge_kw = 3.7 %}
{% set battery_kwh = 69 %}
{% set charge_limit = states.sensor.tesla_charge_limit_soc.state|int %} 
{% set current_soc = states.sensor.tesla_battery_level.state|int %}
{% set charge_hours = (( charge_limit - current_soc ) / 100 * battery_kwh / charge_kw)|round|int %}
Need to charge {{ (( charge_limit - current_soc ) / 100 * battery_kwh)|float|round(2)  }} kWh in {{ charge_hours }} hours (Avg: {{charge_kw}} kW)

Charge during:

{%- set data = 
state_attr('sensor.nordpool_kwh_se3_sek_3_10_025', 'raw_today')|selectattr('start', '>=', (now()))|sort(attribute='value') +
state_attr('sensor.nordpool_kwh_se3_sek_3_10_025', 'raw_tomorrow')[:8]|sort(attribute='value')
%}

{%- set l=data|sort(attribute='value') %}

{%- set ns = namespace(list_of_hours=[]) %}

{%- for i in range(charge_hours) %}
  {%- if now() <= l[i].start %}  
    {%- set ns.list_of_hours = ns.list_of_hours + [(l[i].start|string)|as_timestamp|timestamp_custom("%Y-%m-%d %H:%M")] %}
  {%- endif %}
{%- endfor %}

{%- for hour in ns.list_of_hours | sort %}
  {{ hour }} - {{ (as_timestamp(hour) + 3600)|timestamp_custom("%Y-%m-%d %H:%M") }}  
{%- endfor %}


