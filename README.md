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

NOTE : esp-idf framework is used here instead of arduino to eanble bluetooth proxy extendend capabilities and 801.11k / 802.11v wifi roaming capabilities. Works on arduino framework, adapt to your configuraition.

```
substitutions:
  device_name: "Niveau Chauffe Eau"
  dallas_hub_1_pin: GPIO22 
  dallas_hub_2_pin: GPIO19 
  dallas_hub_3_pin: GPIO21
  dallas_hub_4_pin: GPIO25 #a changer
  dallas_hub_5_pin: GPIO23

esphome:
  name: "niveau-chauffe-eau"
  min_version: 2025.10.4
  build_path: build/niveu-chauffe-eau

esp32:
  board: esp32dev
  framework:
    type: esp-idf

esp32_ble:
  max_connections: 5

esp32_ble_tracker:
  scan_parameters:
    interval: 320ms
    window: 300ms
    active: true

bluetooth_proxy:
  active: true
  connection_slots: 5


# WiFi Configuration
wifi:
  networks:
    # - ssid: !secret iot_ssid
    #   password: !secret iot_password
    ssid: !secret wifi_ssid
    password: !secret wifi_password
  manual_ip: 
    static_ip: 192.168.0.224 #a changer
    gateway: 192.168.0.1
    subnet: 255.255.255.0
  #output_power: 20.5dB
  #power_save_mode: NONE
# Wifi 802.11k Settings
#  enable_btm: true
#  enable_rrm: true


  ap:
    ssid: "${device_name}"
    password: !secret temp_chauffe_eau_admin

# Enable Web Server for local access
#web_server:
#  local: true
#  port: 80

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




# # Dallas Temperature Sensors
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
#     #address: 0xcf79ddd446e17728
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
#     #address: 0xe13ce10457ac2128
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
#     #address: 0x12839fb30a646128
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


  - platform: wifi_signal
    name: "WiFi Signal"
    update_interval: 60s


safe_mode:
  disabled: False



# --- BUTTONS ---
button:
  - platform: restart
    name: "Restart"
    
  - platform: safe_mode
    name: "Restart Safe Mode"
    entity_category: config



```

## UPDATE / UPGRADE

Using Gemini and Claude AI, i've upgraded the code to make it more faul tolerant, improve documentation and add a new calcuilation mathod based on Hyperbolic Tangent over the whole volume instead of piecewise linear interpolation.

It results in a more "physical" temperature front identification and a more robust energy calculation and availability :

<img width="866" height="642" alt="image" src="https://github.com/user-attachments/assets/af5f4c40-c229-4ab8-b6b3-563568a1a3ff" />

I also changed the GPIO to avoid strapping pin

Here's the upgraded code :


```
substitutions:
  device_name: "Chauffe Eau"
  # Pin Definitions (Matching 1-Wire Buses)
  dallas_hub_1_pin: GPIO22 
  dallas_hub_2_pin: GPIO19 
  dallas_hub_3_pin: GPIO21 
  dallas_hub_4_pin: GPIO25 
  dallas_hub_5_pin: GPIO23

esphome:
  name: "sonde-chauffe-eau"
  friendly_name: Sonde-Chauffe-Eau
  min_version: 2024.11.2
  build_path: build/niveau-chauffe-eau

esp32:
  board: esp32dev
  framework:
    type: esp-idf

# --- BLUETOOTH CONFIGURATION ---
esp32_ble:
  max_connections: 5

esp32_ble_tracker:
  scan_parameters:
    interval: 320ms
    window: 300ms
    active: true

bluetooth_proxy:
  active: true
  connection_slots: 5

# --- WIFI CONFIGURATION ---
wifi:
  networks:
    - ssid: !secret iot_ssid
      password: !secret iot_password
    - ssid: !secret wifi_ssid
      password: !secret wifi_password
  manual_ip: 
    static_ip: 192.168.0.24
    gateway: 192.168.0.1
    subnet: 255.255.255.0

  # Power Save OFF for BT stability
  power_save_mode: NONE
  
  # Roaming Support
  enable_btm: true
  enable_rrm: true

  ap:
    ssid: "${device_name}"
    password: !secret temp_chauffe_eau_admin

api:
  encryption:
    key: !secret temp_chauffe_eau_api_key

logger:
  level: DEBUG

ota:
  - platform: esphome
    password: !secret temp_chauffe_eau_admin

time:
  - platform: homeassistant
    timezone: "Europe/Paris"
    id: homeassistant_time

# --- ONE-WIRE BUSES (1 per Pin) ---
one_wire:
  - platform: gpio
    id: bus_1
    pin: ${dallas_hub_1_pin} # 75%
  - platform: gpio
    id: bus_2
    pin: ${dallas_hub_2_pin} # 50%
  - platform: gpio  
    id: bus_3
    pin: ${dallas_hub_3_pin} # 25%
  - platform: gpio  
    id: bus_4
    pin: ${dallas_hub_4_pin} # 100%
  - platform: gpio  
    id: bus_5
    pin: ${dallas_hub_5_pin} # 0%

# --- SENSORS ---
sensor:
  # System Health Sensors
  - platform: internal_temperature
    name: "Température CPU ESP32"
    update_interval: 60s
    
  - platform: wifi_signal
    name: "${device_name} WiFi Signal"
    update_interval: 30s # 30-second update frequency

  # Common Filter Block for all DS18B20 sensors
    filters: &common_filters
    # Calibration Filter (Applying your linear map)
    - calibrate_linear:
        method: least_squares
        datapoints:
         - 14.9 -> 10.9
         - 46.5 -> 54.5
    # Sanity Check for Temperature Range
    - clamp:
        min_value: 5
        max_value: 85 
        ignore_out_of_range: true
    # Smoothing Filter (Moving Average)
    - sliding_window_moving_average:
        window_size: 10
        send_every: 2
    - round: 1
  
  # Dallas Temperature Sensors (5 Sensors)
  - platform: dallas_temp
    one_wire_id: bus_1
    address: 0xbd3335d446b8a228
    id: sensor_1
    name: "Temperature Niveau 75%"
    update_interval: 60s
    resolution: 11
    filters: *common_filters
    
  - platform: dallas_temp
    one_wire_id: bus_2
    address: 0xcf79ddd446e17728
    id: sensor_2
    name: "Temperature Niveau 50%"
    update_interval: 60s
    resolution: 11
    filters: *common_filters

  - platform: dallas_temp
    one_wire_id: bus_3
    address: 0xef1ff7d446a22628
    id: sensor_3
    name: "Temperature Niveau 25%"
    update_interval: 60s
    resolution: 11
    filters: *common_filters

  - platform: dallas_temp
    one_wire_id: bus_4
    address: 0xe13ce10457ac2128
    id: sensor_4
    name: "Temperature Niveau 100%"
    update_interval: 60s
    resolution: 11
    filters: *common_filters

  - platform: dallas_temp
    one_wire_id: bus_5
    address: 0x12839fb30a646128
    id: sensor_5
    name: "Temperature Niveau 0%"
    update_interval: 60s
    resolution: 11
    filters: *common_filters

  # --- VIRTUAL VOLUME 1: USEFUL VOLUME (GEOMETRIC) ---
  # Calculates the volume physically above 40°C using Tanh/Logit interpolation.
  - platform: template
    name: "Volume utile (> 40°C)"
    unit_of_measurement: "L"
    icon: mdi:chart-bell-curve-cumulative
    device_class: "volume_storage"
    state_class: "measurement"
    update_interval: 60s
    accuracy_decimals: 1
    lambda: |-
      const float TARGET_TEMP = 40.0;
      const float TOTAL_VOL = 300.0;
      
      // Sensor Positions (Height percentage 0.0 to 1.0)
      // Sorted from Bottom (Cold) to Top (Hot)
      const float heights[] = {0.0, 0.25, 0.50, 0.75, 1.0};
      
      // Get Readings (Sorted Bottom to Top)
      float temps[5];
      temps[0] = id(sensor_5).state; // 0%
      temps[1] = id(sensor_3).state; // 25%
      temps[2] = id(sensor_2).state; // 50%
      temps[3] = id(sensor_1).state; // 75%
      temps[4] = id(sensor_4).state; // 100%

      for (float t : temps) { if (isnan(t)) return {}; }

      if (temps[4] < TARGET_TEMP) return 0.0;
      if (temps[0] >= TARGET_TEMP) return TOTAL_VOL;

      // Define Asymptotes for Normalization (Safety margin to avoid log(0))
      float t_min = temps[0] - 1.0; 
      float t_max = temps[4] + 1.0; 

      // Function to transform Temperature to Logit Space (Linearizing the Sigmoid)
      auto to_logit = [&](float t) -> float {
        float norm = (t - t_min) / (t_max - t_min);
        // Clamp to prevent log domain errors
        if (norm <= 0.01) norm = 0.01;
        if (norm >= 0.99) norm = 0.99;
        return log(norm / (1.0 - norm));
      };

      // 1. Find the segment containing 40°C
      int idx = -1;
      for (int i = 0; i < 4; i++) {
        if (temps[i] <= TARGET_TEMP && temps[i+1] >= TARGET_TEMP) {
          idx = i;
          break;
        }
      }

      if (idx == -1) return 0.0;

      // 2. Perform Linear Interpolation in Logit Space
      float y_lower = to_logit(temps[idx]);
      float y_upper = to_logit(temps[idx+1]);
      float y_target = to_logit(TARGET_TEMP);

      float progress_fraction;
      if (abs(y_upper - y_lower) < 0.0001) {
        progress_fraction = 0.5; // Avoid div/0 if temps are perfectly equal
      } else {
        progress_fraction = (y_target - y_lower) / (y_upper - y_lower);
      }

      // 3. Calculate Height of the 40°C point
      float h_lower = heights[idx];
      float h_upper = heights[idx+1];
      float height_of_40 = h_lower + (h_upper - h_lower) * progress_fraction;

      // Volume is the portion ABOVE this height (1.0 - height_of_40)
      float useful_ratio = 1.0 - height_of_40;
      
      // Final Clamp
      if (useful_ratio < 0) useful_ratio = 0;
      if (useful_ratio > 1) useful_ratio = 1;

      return useful_ratio * TOTAL_VOL;


  # --- VIRTUAL VOLUME 2: EXPLOITABLE VOLUME (THERMODYNAMIC) ---
  # Uses Numerical Integration (Riemann Sum) of the Tanh curve to calculate 
  # the equivalent volume of 40°C water that can be obtained by mixing.
  - platform: template
    name: "Volume exploitable (eq. 40°C)"
    unit_of_measurement: "L"
    icon: mdi:function-variant
    device_class: "volume_storage"
    state_class: "measurement"
    update_interval: 60s
    accuracy_decimals: 1
    lambda: |-
      // --- CONSTANTS ---
      const float TARGET_TEMP = 40.0;
      const float TOTAL_VOL = 300.0;
      const float TEMP_COLD_DEF = 15.0;
      const int INTEGRATION_STEPS = 50; 
      
      // Sensor Heights and Temps (Sorted Bottom to Top)
      const float h_sensors[] = {0.0, 0.25, 0.50, 0.75, 1.0};
      float t_sensors[5];
      t_sensors[0] = id(sensor_5).state; t_sensors[1] = id(sensor_3).state;
      t_sensors[2] = id(sensor_2).state; t_sensors[3] = id(sensor_1).state;
      t_sensors[4] = id(sensor_4).state;

      for (float t : t_sensors) { if (isnan(t)) return {}; }

      // --- DYNAMIC PARAMETERS ---
      float t_cold = (t_sensors[0] < TEMP_COLD_DEF) ? t_sensors[0] : TEMP_COLD_DEF;
      
      // Global Asymptotes for Sigmoid Modeling (Safety margin)
      float t_min_limit = t_sensors[0] - 2.0; 
      float t_max_limit = t_sensors[4] + 2.0;
      
      // Helper Function: The Inverse Logit (T -> L)
      auto to_logit = [&](float t) -> float {
          float norm = (t - t_min_limit) / (t_max_limit - t_min_limit);
          if (norm <= 0.001) norm = 0.001;
          if (norm >= 0.999) norm = 0.999;
          return log(norm / (1.0 - norm));
      };
      
      // Helper Function: The Sigmoid (L -> T)
      auto get_temp_at_height = [&](float h_query) -> float {
          // 1. Find segment index
          int idx = 0;
          for(int i=0; i<4; i++) { if (h_query >= h_sensors[i]) idx = i; }
          
          // 2. Convert endpoints to Logit Space
          float L_lower = to_logit(t_sensors[idx]);
          float L_upper = to_logit(t_sensors[idx+1]);

          // 3. Linear Interpolation in Logit Space
          float h_lower = h_sensors[idx];
          float h_upper = h_sensors[idx+1];
          float progress = (h_query - h_lower) / (h_upper - h_lower);
          
          float L_interpolated = L_lower + progress * (L_upper - L_lower);

          // 4. Convert back from Logit to Temp (Sigmoid)
          float norm_interpolated = 1.0 / (1.0 + exp(-L_interpolated));
          return t_min_limit + norm_interpolated * (t_max_limit - t_min_limit);
      };

      // --- NUMERICAL INTEGRATION (Riemann Sum) ---
      float total_enthalpy_ratio = 0.0;
      float slice_height = 1.0 / INTEGRATION_STEPS;
      float denom = TARGET_TEMP - t_cold;

      // Safety check for target temperature relative to cold water
      if (denom < 1.0) denom = 1.0; 

      for (int k = 0; k < INTEGRATION_STEPS; k++) {
        // Sample the temperature at the CENTER of the slice
        float h_center = (k + 0.5) * slice_height;
        float t_slice = get_temp_at_height(h_center);

        // Calculate mixing potential of this slice
        // Ratio = (T_slice - T_cold) / (T_target - T_cold)
        float ratio = (t_slice - t_cold) / denom;
        
        // Physics constraint: only positive energy contributes to 'Exploitable' volume
        if (ratio < 0) ratio = 0;
        
        total_enthalpy_ratio += ratio;
      }

      // Final Calculation: Average Ratio * Total Volume
      float effective_volume = (total_enthalpy_ratio / INTEGRATION_STEPS) * TOTAL_VOL;
      
      return effective_volume;

  - platform: wifi_signal
    name: "WiFi Signal"
    update_interval: 60s


safe_mode:
  disabled: False



# --- BUTTONS ---
button:
  - platform: restart
    name: "Restart"
    
  - platform: safe_mode
    name: "Restart Safe Mode"
    entity_category: config

```

 
