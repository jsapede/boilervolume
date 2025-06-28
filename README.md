# electrical boiler hot water reserve 

Use ESPHOMe ans Dallas temperature sensors to compute hot water volume in belectrical boiler

## Principles : 

Volume calculation is made with Dallas sensors sticked on the inner smertallic surface fo the electrical boiler. 

**Main assumptions are :**
- boiler tank is a vertical cylinder
- hot water is extrated from top of the boiler tank
- Cold water supply is made ate the bottom of the boiler tank
- Temperature is stratified inside the tank whith high steep gradient making temperature front
- Boiler tank volume is divided in 4 subvolumes, using 5 Dallas temperature sensors set a 0%, 25%, 50%, 75% and 100% height of the tank

**Data computed**
- Available hot water volume (available volume of water that is above 40°C)
- Useful hot water volume (available volume of hot water diluted to 40°C at the tap/shower)

**Calculation Strategy :**
- top down calculation, from higher subvolume to lower
- for each subvolume :
  - if upper temperature sensor < 40°c => subvolume = 0
  - if upper ANd lower temperature sensor > 40°c => subvolume = full subvolume
  - if upper temperaure sensor > 40°C and lower temperature sensor < 40°C => subvolume = linear interpolation between upper and lower TT and calculation of the part of the subvolume that is >40°C

**Sensors calibration :**
Sensors are sticked against the inner metallic tank of the boiler using thermal paste

- 0% level is cold water supply => take some tapwater and check T° using the sensor then compare to the 0% level once the tank is at last 50% full of cold water (i.e. for example after daily usage)
- 100% level is max water temperature => take some hot water at the tap and put it in dewar bottle, and check temperature using the sensor. Compare to the 100% tempearture value measured on tank metal surface 

From measurements build calibration : 

```
    filters:
      - calibrate_linear:
         method: least_squares
         datapoints:
          # Map 0.0 (from sensor) to 1.0 (true value)
          - 14.9 -> 10.9 #at 0% surface tank -> at tap water
          - 46.5 -> 54.5 #at 100% surfacetank -> at hot tap water
```

**Sensor hardware addresses :**

Dallas sensor addresses "address: 0xbd3335d446b8a228" are unique. Should be checked after first start of the ESPhome firmware to adjust config :

![image](https://github.com/user-attachments/assets/3313c812-fc6f-4803-bf68-7dc6d9b2bd3a)

**Parameters**

Set of parameters used for th ecalculation, adjust according to your system :

```
      float temp_ref = 40.0; # reference temperature => useful temperature at shower
      float vol_ref = 25.0; # subvolume, depends on the number of sensors
      float temp_reseau = 15.0; # cold water temperature
      float temp_reseau_ref = 15.0; # default cold water temperature
      float vol_total = 300; # total useful volume of the tank
```

*ESPHome Code*


```
substitutions:
  device_name: "Chauffe Eau"
  dallas_hub_1_pin: GPIO13
  dallas_hub_2_pin: GPIO12
  dallas_hub_3_pin: GPIO14
  dallas_hub_4_pin: GPIO15
  dallas_hub_5_pin: GPIO4

esphome:
  name: "sonde-chauffe-eau"
  min_version: 2024.11.2
  build_path: build/chauffe-eau

esp32:
  board: esp32dev
  framework:
    type: arduino

# WiFi Configuration
wifi:
  ssid: !secret iot_ssid
  password: !secret iot_password
  output_power: 20.5dB
  passive_scan: True
  fast_connect: True
  power_save_mode: NONE
  domain: .lan

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
  level: DEBUG

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
  - platform: gpio  
    id: dallas_hub_4
    pin: ${dallas_hub_4_pin}
  - platform: gpio  
    id: dallas_hub_5
    pin: ${dallas_hub_5_pin}




# Dallas Temperature Sensors
sensor:
  - platform: dallas_temp
    one_wire_id: dallas_hub_1
    address: 0xbd3335d446b8a228
    id: sensor_1
    name: "Temperature Niveau 75%"
    update_interval: 60s
    resolution: 11
    device_class: "temperature"
    state_class: "measurement"
    unit_of_measurement: "°C"
    icon: mdi:thermometer
    filters:
      - calibrate_linear:
         method: least_squares
         datapoints:
          # Map 0.0 (from sensor) to 1.0 (true value)
          - 14.9 -> 10.9
          - 46.5 -> 54.5
      - clamp:
          min_value: 5
          max_value: 65
          ignore_out_of_range: true
      - sliding_window_moving_average:
          window_size: 10
          send_every: 2
      - round: 1

  - platform: dallas_temp
    one_wire_id: dallas_hub_2
    address: 0xcf79ddd446e17728
    id: sensor_2
    name: "Temperature Niveau 50%"
    update_interval: 60s
    resolution: 11
    device_class: "temperature"
    state_class: "measurement"
    unit_of_measurement: "°C"
    icon: mdi:thermometer
    filters:
      - calibrate_linear:
         method: least_squares
         datapoints:
          # Map 0.0 (from sensor) to 1.0 (true value)
          - 14.9 -> 10.9
          - 46.5 -> 54.5
      - clamp:
          min_value: 5
          max_value: 65
          ignore_out_of_range: true
      - sliding_window_moving_average:
          window_size: 10
          send_every: 2
      - round: 1


  - platform: dallas_temp
    one_wire_id: dallas_hub_3
    address: 0xef1ff7d446a22628
    id: sensor_3
    name: "Temperature Niveau 25%"
    update_interval: 60s
    resolution: 11
    device_class: "temperature"
    state_class: "measurement"
    unit_of_measurement: "°C"
    icon: mdi:thermometer
    filters:
      - calibrate_linear:
         method: least_squares
         datapoints:
          # Map 0.0 (from sensor) to 1.0 (true value)
          - 14.9 -> 10.9
          - 46.5 -> 54.5
      - clamp:
          min_value: 5
          max_value: 65
          ignore_out_of_range: true
      - sliding_window_moving_average:
          window_size: 10
          send_every: 2
      - round: 1


  - platform: dallas_temp
    one_wire_id: dallas_hub_4
    address: 0xe13ce10457ac2128
    id: sensor_4
    name: "Temperature Niveau 100%"
    update_interval: 60s
    resolution: 11
    device_class: "temperature"
    state_class: "measurement"
    unit_of_measurement: "°C"
    icon: mdi:thermometer
    filters:
      - calibrate_linear:
         method: least_squares
         datapoints:
          # Map 0.0 (from sensor) to 1.0 (true value)
          - 14.9 -> 10.9
          - 46.5 -> 54.5
      - clamp:
          min_value: 5
          max_value: 65
          ignore_out_of_range: true
      - sliding_window_moving_average:
          window_size: 10
          send_every: 2
      - round: 1

  - platform: dallas_temp
    one_wire_id: dallas_hub_5
    address: 0x12839fb30a646128
    id: sensor_5
    name: "Temperature Niveau 0%"
    update_interval: 60s
    resolution: 11
    device_class: "temperature"
    state_class: "measurement"
    unit_of_measurement: "°C"
    icon: mdi:thermometer
    filters:
      - calibrate_linear:
         method: least_squares
         datapoints:
          # Map 0.0 (from sensor) to 1.0 (true value)
          - 14.9 -> 10.9
          - 46.5 -> 54.5
      - clamp:
          min_value: 5
          max_value: 65
          ignore_out_of_range: true
      - sliding_window_moving_average:
          window_size: 10
          send_every: 2
      - round: 1

  - platform: template
    name: "Volume utile (> 40°C)"
    unit_of_measurement: "L"
    icon: mdi:gauge
    device_class: "volume_storage"
    state_class: "measurement"
    update_interval: 60s
    accuracy_decimals: 1
    lambda: |-
      float temp_ref = 40.0;
      float temp_reseau = 15.0;
      float temp_reseau_ref = 15.0;
      float vol_total = 300;
      float vol_ref_100_75 = (30.0/140.0)*100.0;
      float vol_ref_75_50 = (40.0/140.0)*100.0;
      float vol_ref_50_25 = (40.0/140.0)*100.0;
      float vol_ref_25_0 = (30.0/140.0)*100.0;
      float temp_75 = id(sensor_1).state;
      float temp_50 = id(sensor_2).state;
      float temp_25 = id(sensor_3).state;
      float temp_100 = id(sensor_4).state;
      float temp_0 = id(sensor_5).state;
      float tangente_100_75 = (temp_75 - temp_100) / vol_ref_100_75;
      float tangente_75_50 = (temp_50 - temp_75) / vol_ref_75_50;
      float tangente_50_25 = (temp_25 - temp_50) / vol_ref_50_25;
      float tangente_25_0 = (temp_0 - temp_25) / vol_ref_25_0;
      float temp_moy_100_75 = (temp_100 + temp_75) / 2.0;
      float temp_moy_75_50 = (temp_75 + temp_50) / 2.0;
      float temp_moy_50_25 = (temp_50 + temp_25) / 2.0;
      float temp_moy_25_0 = (temp_25 + temp_0) / 2.0;
      float vol_utile_100_75 = 0.0;
      float vol_utile_75_50 = 0.0;
      float vol_utile_50_25 = 0.0;
      float vol_utile_25_0 = 0.0;
      float vol_utile = 0.0;

      if (temp_0 < temp_reseau_ref) {
        temp_reseau = temp_0;
      } else {
        temp_reseau = temp_reseau_ref;
      }

      if (temp_100 >= temp_ref) {
        if (temp_75 >= temp_ref) {
          vol_utile_100_75 = vol_ref_100_75;
        } else {
          vol_utile_100_75 = (temp_ref-temp_100)/tangente_100_75;
          }
      } else {
        vol_utile_100_75 = 0.0;
        }

      if (temp_100 >=temp_ref) {
        if (temp_75 >= temp_ref) {
          if (temp_50 >= temp_ref) {
            vol_utile_75_50 = vol_ref_75_50;
          } else {
            vol_utile_75_50 = (temp_ref-temp_75)/tangente_75_50;
          }
        } else {
          vol_utile_75_50 = 0.0;
        }
      } else {
          vol_utile_75_50 = 0.0;
      }  

      if (temp_75 >= temp_ref and temp_100 >=temp_ref) {
        if (temp_50 >= temp_ref) {
          if (temp_25 >= temp_ref) {
            vol_utile_50_25 = vol_ref_50_25;
          } else {
            vol_utile_50_25 = (temp_ref-temp_50)/tangente_50_25;
          }
        } else {
          vol_utile_50_25 = 0.0;
        }
      } else {
          vol_utile_50_25 = 0.0;
      }      

      if (temp_50 >= temp_ref and temp_75 >= temp_ref and temp_100 >=temp_ref) {
        if (temp_25 >= temp_ref) {
          if (temp_0 >= temp_ref) {
            vol_utile_25_0 = vol_ref_25_0;
          } else {
            vol_utile_25_0 = (temp_ref-temp_25)/tangente_25_0;
          }
        } else {
          vol_utile_25_0 = 0.0;
        }
      } else {
          vol_utile_25_0 = 0.0;
      }

      vol_utile = (vol_utile_100_75 + vol_utile_75_50 + vol_utile_50_25 + vol_utile_25_0)*vol_total/100.0;
      return vol_utile;




  - platform: template
    name: "Volume exploitable (eq. 40°C)"
    unit_of_measurement: "L"
    icon: mdi:gauge
    device_class: "volume_storage"
    state_class: "measurement"
    update_interval: 60s
    accuracy_decimals: 1
    lambda: |-
      float temp_ref = 40.0;
      float vol_ref_100_75 = (30.0/140.0)*100.0;
      float vol_ref_75_50 = (40.0/140.0)*100.0;
      float vol_ref_50_25 = (40.0/140.0)*100.0;
      float vol_ref_25_0 = (30.0/140.0)*100.0;
      float temp_reseau = 15.0;
      float temp_reseau_ref = 15.0;
      float vol_total = 300;
      float temp_75 = id(sensor_1).state;
      float temp_50 = id(sensor_2).state;
      float temp_25 = id(sensor_3).state;
      float temp_100 = id(sensor_4).state;
      float temp_0 = id(sensor_5).state;
      float tangente_100_75 = (temp_75 - temp_100) / vol_ref_100_75;
      float tangente_75_50 = (temp_50 - temp_75) / vol_ref_75_50;
      float tangente_50_25 = (temp_25 - temp_50) / vol_ref_50_25;
      float tangente_25_0 = (temp_0 - temp_25) / vol_ref_25_0;
      float temp_moy_100_75 = (temp_100 + temp_75) / 2.0;
      float temp_moy_75_50 = (temp_75 + temp_50) / 2.0;
      float temp_moy_50_25 = (temp_50 + temp_25) / 2.0;
      float temp_moy_25_0 = (temp_25 + temp_0) / 2.0;
      float vol_expl_100_75 = 0.0;
      float vol_expl_75_50 = 0.0;
      float vol_expl_50_25 = 0.0;
      float vol_expl_25_0 = 0.0;
      float vol_expl = 0.0;

      if (temp_0 < temp_reseau_ref) {
        temp_reseau = temp_0;
      } else {
        temp_reseau = temp_reseau_ref;
      }

      if (temp_100 >= temp_ref) {
        if (temp_75 >= temp_ref) {
          vol_expl_100_75 = vol_ref_100_75 * (1 + (temp_moy_100_75 - temp_ref)/(temp_ref - temp_reseau));
        } else {
          vol_expl_100_75 = ((temp_ref-temp_100)/tangente_100_75) * (1 + (temp_moy_100_75 - temp_ref)/(temp_ref - temp_reseau));
          }
      } else {
        vol_expl_100_75 = 0.0;
        }

      if (temp_100 >=temp_ref) {
        if (temp_75 >= temp_ref) {
          if (temp_50 >= temp_ref) {
            vol_expl_75_50 = vol_ref_75_50 * (1 + (temp_moy_75_50 - temp_ref)/(temp_ref - temp_reseau));
          } else {
            vol_expl_75_50 = ((temp_ref-temp_75)/tangente_75_50) * (1 + (temp_moy_75_50 - temp_ref)/(temp_ref - temp_reseau));
          }
        } else {
          vol_expl_75_50 = 0.0;
        }
      } else {
          vol_expl_75_50 = 0.0;
      }

      if (temp_75 >= temp_ref and temp_100 >=temp_ref) {
        if (temp_50 >= temp_ref) {
          if (temp_25 >= temp_ref) {
            vol_expl_50_25 = vol_ref_50_25 * (1 + (temp_moy_50_25 - temp_ref)/(temp_ref - temp_reseau));
          } else {
            vol_expl_50_25 = ((temp_ref-temp_50)/tangente_50_25) * (1 + (temp_moy_50_25 - temp_ref)/(temp_ref - temp_reseau));
          }
        } else {
          vol_expl_50_25 = 0.0;
        }
      } else {
          vol_expl_50_25 = 0.0;
      }


      if (temp_50 >= temp_ref and temp_75 >= temp_ref and temp_100 >=temp_ref) {
        if (temp_25 >= temp_ref) {
          if (temp_0 >= temp_ref) {
            vol_expl_25_0 = vol_ref_25_0 * (1 + (temp_moy_25_0 - temp_ref)/(temp_ref - temp_reseau));
          } else {
            vol_expl_25_0 = ((temp_ref-temp_25)/tangente_25_0) * (1 + (temp_moy_25_0 - temp_ref)/(temp_ref - temp_reseau));
          }
        } else {
          vol_expl_25_0 = 0.0;
        }
      } else {
          vol_expl_25_0 = 0.0;
      }

      vol_expl = (vol_expl_100_75 + vol_expl_75_50 + vol_expl_50_25 + vol_expl_25_0)*vol_total/100.0;
      return vol_expl;

  - platform: wifi_signal # Reports the WiFi signal strength/RSSI in dB
    name: "WiFi Signal dB"
    id: wifi_signal_db_sonde_chauffe_eau
    update_interval: 60s
    entity_category: "diagnostic"

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
