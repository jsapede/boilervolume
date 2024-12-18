precision : 
- extraction eau chaude par le haut
- appoint eau froide par le bas
- 3 sondes dallas a 25%, 50% et 75% de hauteur
- la temperature a 100% de hauteur est considérée constante a 50°C
- la température 0% de hauteur est considérée constante a 20°C
- on calcule d'abord le volume d'eau exploitable => tout le volume dont la T° est > 38°C
- on calcule ensuite le volume réel dilué utilisable à 38°C, en considérant une dilution à l'eau froide a 20°C


```
####################
## Config existante
####################


substitutions:
  device_name: "Temperature Chauffe Eau"
  dallas_hub_1_pin: GPIO13
  dallas_hub_2_pin: GPIO12
  dallas_hub_3_pin: GPIO14

esphome:
  name: "sonde-chauffe-eau"
  min_version: 2024.11.2
  build_path: build/sonde-chauffe-eau

esp32:
  board: esp32dev
  framework:
    type: arduino

# WiFi Configuration
wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

  ap:
    ssid: "${device_name}"
    password: !secret temp_chauffe_eau_admin

# Enable Web Server for local access
web_server:
  local: true
  port: 80

# Enable API for Home Assistant integration
api:
  encryption:
    key: !secret temp_chauffe_eau_api_key

# Logger Configuration
logger:
  level: INFO

# OTA (Over-the-Air) updates
ota:
  - platform: esphome
    password: !secret temp_chauffe_eau_admin

# Time Configuration (sync with Home Assistant)
time:
  - platform: homeassistant
    timezone: "Europe/Paris"
    id: homeassistant_time

# Binary Sensor for device status
binary_sensor:
  - platform: status
    name: "${device_name} Status"

# Button to restart the device
button:
  - platform: restart
    name: "${device_name} Restart"

# One-wire temperature sensors
one_wire:
  - platform: gpio
    id: dallas_hub_1
    pin: ${dallas_hub_1_pin}
  - platform: gpio
    id: dallas_hub_2
    pin: ${dallas_hub_2_pin}
  - platform: gpio  
    id: dallas_hub_3
    pin: ${dallas_hub_3_pin}

# Dallas Temperature Sensors
sensor:
  - platform: dallas_temp
    one_wire_id: dallas_hub_1
    address: 0xbd3335d446b8a228
    id: sensor_1
    name: "Chauffe-Eau Temperature 75%"
    update_interval: 60s
    resolution: 12
  - platform: dallas_temp
    one_wire_id: dallas_hub_2
    address: 0xcf79ddd446e17728
    id: sensor_2
    name: "Chauffe-Eau Temperature 50%"
    update_interval: 60s
    resolution: 12
  - platform: dallas_temp
    one_wire_id: dallas_hub_3
    address: 0xef1ff7d446a22628
    id: sensor_3
    name: "Chauffe-Eau Temperature 25%"
    update_interval: 60s
    resolution: 12


  - platform: template
    name: "Volume exploitable (38°C)"
    unit_of_measurement: "L"
    icon: mdi:gauge
    lambda: |-
      float sensor1_temp = id(sensor_1).state;
      float sensor2_temp = id(sensor_2).state;
      float sensor3_temp = id(sensor_3).state;

      const float full_volume = 300.0;  // Total volume of the boiler (in liters)
      const float temp_100_percent = 50.0;  // Temperature at 100% height
      const float temp_0_percent = 20.0; // temperature at 0% height
      const float temp_ref = 38.0;  // Temperature reference for useful volume calculation

      float useful_volume = 0.0;
      float tangent = 1.0;
      
      if (sensor1_temp >= temp_ref) {
        if (sensor2_temp >= temp_ref) {
          if (sensor3_temp >= temp_ref) {
            // Linear interpolation between 25% and 0% height
            float tangent = (sensor3_temp - temp_0_percent) / 25.0;
            float interpolated_height = 25 - (sensor3_temp - temp_ref) / tangent;
            useful_volume = full_volume * ((100 - interpolated_height) * 0.01);
            return useful_volume;  // Return after calculation
          } else {
            // Linear interpolation between 50% and 25% height
            float tangent = (sensor2_temp - sensor3_temp) / 25.0;
            float interpolated_height = 50 - (sensor2_temp - temp_ref) / tangent;
            useful_volume = full_volume * ((100 - interpolated_height) * 0.01);
            return useful_volume;  // Return after calculation
          }
        } else {
          // Linear interpolation between 75% and 50% height
          float tangent = (sensor1_temp - sensor2_temp) / 25.0;
          float interpolated_height = 75 - (sensor1_temp - temp_ref) / tangent ;
          useful_volume = full_volume * ((100 - interpolated_height) * 0.01);
          return useful_volume;  // Return after calculation
        }
      } else {
        // Linear interpolation between 100% and 75% height
        float tangent = (temp_100_percent - sensor1_temp) / 25.0;
        float interpolated_height = 100 - (temp_100_percent - temp_ref) / tangent ;
        useful_volume = full_volume * ((100 - interpolated_height) * 0.01);
        return useful_volume;  // Return after calculation
      }

      // If the temperature is too low, return useful_volume (it will be 0 in this case)
      return useful_volume;





  - platform: template
    name: "Volume équivalent 38°C"
    unit_of_measurement: "L"
    icon: mdi:gauge
    lambda: |-
      float sensor1_temp = id(sensor_1).state;
      float sensor2_temp = id(sensor_2).state;
      float sensor3_temp = id(sensor_3).state;

      const float full_volume = 300.0;  // Total volume of the boiler (in liters)
      const float temp_ref = 38.0;  // Temperature reference for useful volume calculation
      const float temp_100_percent = 50.0;  // Temperature at 100% height
      const float temp_0_percent = 20.0; // temperature at 0% height

      float useful_volume = 0.0;
      float temp_moy = temp_ref;  // Default to temp_ref for initial calculation

      // Calculate useful volume based on sensor readings
      if (sensor1_temp >= temp_ref) {
        // Section from 75% to 100% volume
        float temp_moy100_75 = (temp_100_percent + sensor1_temp) / 2.0;
        useful_volume += full_volume * 0.25 * (1 + ((temp_moy100_75 - temp_ref) / (temp_ref - temp_0_percent)));

        if (sensor2_temp >= temp_ref) {
          // Section from 50% to 75% volume
          float temp_moy75_50 = (sensor1_temp + sensor2_temp) / 2.0;
          useful_volume += full_volume * 0.25 * (1 + ((temp_moy75_50 - temp_ref) / (temp_ref - temp_0_percent)));

          if (sensor3_temp >= temp_ref) {
            // Section from 25% to 50% volume
            float temp_moy50_25 = (sensor2_temp + sensor3_temp) / 2.0;

            // section from 0% to 25%
            float tangent25_0 = (sensor3_temp - temp_0_percent) / 25.0;
            float interpolated_height25_0 = 25 - (sensor3_temp - temp_ref) / tangent25_0;
            float temp_moy25_0_lin = (sensor3_temp + temp_0_percent) / 2.0;
            useful_volume +=  full_volume * 0.25 * (1 + ((temp_moy50_25 - temp_ref) / (temp_ref - temp_0_percent)));
            useful_volume +=  full_volume * 0.25 * (1 + ((temp_moy25_0_lin - temp_ref) / (temp_ref - temp_0_percent))) * ((25 - interpolated_height25_0) * 0.01);
          } else {
            // Section from 50% to 25% volume with interpolation
            float tangent50_25 = (sensor2_temp - sensor3_temp) / 25.0;
            float interpolated_height50_25 = 25 - (sensor2_temp - temp_ref) / tangent50_25;
            float temp_moy50_25_lin = (sensor2_temp + temp_ref) / 2.0;
            useful_volume += full_volume * 0.25 * (1 + ((temp_moy50_25_lin - temp_ref) / (temp_ref - temp_0_percent))) * ((25 - interpolated_height50_25) * 0.01);
          }
        } else {
          // Section from 75% to 50% volume with interpolation
          float tangent75_50 = (sensor1_temp - sensor2_temp) / 25.0;
          float interpolated_height75_50 = 25 - (sensor1_temp - temp_ref) / tangent75_50;
          float temp_moy75_50 = (sensor1_temp + temp_ref) / 2.0;
          useful_volume += full_volume * 0.25 * (1 + ((temp_moy75_50 - temp_ref) / (temp_ref - temp_0_percent))) * ((25 - interpolated_height75_50) * 0.01);
        }
      } else {
        // Section from 100% to 75% volume
        float tangent100_75 = (temp_100_percent - sensor1_temp) / 25.0;
        float interpolated_height100_75 = 25 - (temp_100_percent - temp_ref) / tangent100_75;
        float temp_moy100_75_lin = (temp_100_percent + temp_ref) / 2.0;
        useful_volume += full_volume * 0.25 * (1 + ((temp_moy100_75_lin - temp_ref) / (temp_ref - temp_0_percent))) * ((25 - interpolated_height100_75) * 0.01);
      }

      return useful_volume;  // Return the calculated useful volume




text_sensor:
  - platform: wifi_info
    ip_address:
      name: "${device_name} IP Address"
    mac_address:
      name: "${device_name} Mac Address"
    ssid:
      name: "${device_name} Wifi SSID"
    bssid:
      name: "${device_name} Wifi BSSID"



```
