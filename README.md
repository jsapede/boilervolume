Calcul sous ESPHOME de la capacité d'eau chaude restante dans un chauffe-eau à partir de sondes Dallas positionnées a différentes hauteur sur la paroi de la cuve.

precision : 
- extraction eau chaude par le haut
- appoint eau froide par le bas
- 5 sondes dallas a 0% 25%, 50% 75% et 100% de hauteur
- stratification des températures + chauffage en partie basse du chauffe eau => en phase de chauffe si le bas se rechauffe plus vite que le haut il n'est pourtant pas exploitable => ajout de filtres sur les valeurs de temperatures aux niveaux superieurs pour éviter de comptabiliser ces volumes lors des phases de chauffe.
- on calcule d'abord le volume d'eau utile c'edt à dire tout le volume dont la T° est >= 40°C
- on calcule ensuite le volume exploitable, c'est à dire le volume équivalent dilué utilisable à 40°C (mitigeur), en considérant une dilution à l'eau du reseau

Stratégie de calcul :
-  on commence par le point le plus haut et on descend : l'eau chaude est soutirée en hauteur et l'appoint froid se fait en bas
-  pour chaque section de volume :
  - si la temperature du niveau le plus haut est < 40°C alors le volume n'est pas utile, ni exploitable
  - si la temperature du niveau le plus haut ET la temperature du niveau le plus bas sont >= 40°C alors tout le volume est utile et le volume exploitable est calulé à partie de la valeur moyenne de temperature sur la tranche
  - si la temperature du niveau le plus haut est >=40°C ET la temperature du niveau le plus bas <40°C alors on in interpole les temperatures et on calcule la proportion de la tranche de volume utile et exploitable

Sondes : 
Les sondes sont collées à la surface de la cuve, et leur mesure est faussée. On rajoute une calibration linéeaire en 2 points : 
-  la temperature au niveau zero, stabilisée, quand le chauffe eau est a moitié chaud est égale à la température mesurée au réseau (prélèvement d'eau froide au robinet) => 14.9°C lu par la sonde en paroi et 10.9°c lu par la sonde dans le prélevement d'eau froide
-  La température au niveau 100 est la tempréture de soutirage deau chaude au robinet => 46.5°C lus par la sonde en paroi et 54.5°c lu par la sonde dans un prelevement d'eau chaude

A partir de ces deux valeurs on construit une régréssion lineaire qui permet de corriger la temperature sur toute la plage de mesure. Cette correction est directement intégrable aux capteurs sous esphome avec la fonction filters, à ajuster selon vos valeurs et le nombre de points de calibration que vous faites. : 

```
    filters:
      - calibrate_linear:
         method: least_squares
         datapoints:
          # Map 0.0 (from sensor) to 1.0 (true value)
          - 14.9 -> 10.9
          - 46.5 -> 54.5
```

il suffit de prélever dans un thermos et y tremper la sonde dallas

NB : les addressages des sodnes dallas "address: 0xbd3335d446b8a228" sont propres à chaque sonde, il faut checker dans les logs de l'esp pour avoir la bonne valeur

![image](https://github.com/user-attachments/assets/3313c812-fc6f-4803-bf68-7dc6d9b2bd3a)


```

substitutions:
  device_name: "Chauffe Eau"
  dallas_hub_1_pin: GPIO13
  dallas_hub_2_pin: GPIO12
  dallas_hub_3_pin: GPIO14
  dallas_hub_4_pin: GPIO15
  dallas_hub_5_pin: GPIO4

esphome:
  name: "chauffe-eau"
  min_version: 2024.11.2
  build_path: build/chauffe-eau

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
    update_interval: 10s
    resolution: 12
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

  - platform: dallas_temp
    one_wire_id: dallas_hub_2
    address: 0xcf79ddd446e17728
    id: sensor_2
    name: "Temperature Niveau 50%"
    update_interval: 10s
    resolution: 12
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


  - platform: dallas_temp
    one_wire_id: dallas_hub_3
    address: 0xef1ff7d446a22628
    id: sensor_3
    name: "Temperature Niveau 25%"
    update_interval: 10s
    resolution: 12
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


  - platform: dallas_temp
    one_wire_id: dallas_hub_4
    address: 0xe13ce10457ac2128
    id: sensor_4
    name: "Temperature Niveau 100%"
    update_interval: 10s
    resolution: 12
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
          window_size: 5
          send_every: 6

  - platform: dallas_temp
    one_wire_id: dallas_hub_5
    address: 0x12839fb30a646128
    id: sensor_5
    name: "Temperature Niveau 0%"
    update_interval: 10s
    resolution: 12
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


  - platform: template
    name: "Volume utile (> 40°C)"
    unit_of_measurement: "L"
    icon: mdi:gauge
    device_class: "volume"
    update_interval: 10s
    accuracy_decimals: 1
    lambda: |-
      float temp_ref = 40.0;
      float vol_ref = 25.0;
      float temp_reseau = 15.0;
      float temp_reseau_ref = 15.0;
      float vol_total = 300;
      float temp_75 = id(sensor_1).state;
      float temp_50 = id(sensor_2).state;
      float temp_25 = id(sensor_3).state;
      float temp_100 = id(sensor_4).state;
      float temp_0 = id(sensor_5).state;
      float tangente_100_75 = (temp_75 - temp_100) / vol_ref;
      float tangente_75_50 = (temp_50 - temp_75) / vol_ref;
      float tangente_50_25 = (temp_25 - temp_50) / vol_ref;
      float tangente_25_0 = (temp_0 - temp_25) / vol_ref;
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
          vol_utile_100_75 = vol_ref;
        } else {
          vol_utile_100_75 = (temp_ref-temp_100)/tangente_100_75;
          }
      } else {
        vol_utile_100_75 = 0.0;
        }

      if (temp_100 >=temp_ref) {
        if (temp_75 >= temp_ref) {
          if (temp_50 >= temp_ref) {
            vol_utile_75_50 = vol_ref;
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
            vol_utile_50_25 = vol_ref;
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
            vol_utile_25_0 = vol_ref;
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
    device_class: "volume"
    update_interval: 10s
    accuracy_decimals: 1
    lambda: |-
      float temp_ref = 40.0;
      float vol_ref = 25.0;
      float temp_reseau = 15.0;
      float temp_reseau_ref = 15.0;
      float vol_total = 300;
      float temp_75 = id(sensor_1).state;
      float temp_50 = id(sensor_2).state;
      float temp_25 = id(sensor_3).state;
      float temp_100 = id(sensor_4).state;
      float temp_0 = id(sensor_5).state;
      float tangente_100_75 = (temp_75 - temp_100) / vol_ref;
      float tangente_75_50 = (temp_50 - temp_75) / vol_ref;
      float tangente_50_25 = (temp_25 - temp_50) / vol_ref;
      float tangente_25_0 = (temp_0 - temp_25) / vol_ref;
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
          vol_expl_100_75 = vol_ref * (1 + (temp_moy_100_75 - temp_ref)/(temp_ref - temp_reseau));
        } else {
          vol_expl_100_75 = ((temp_ref-temp_100)/tangente_100_75) * (1 + (temp_moy_100_75 - temp_ref)/(temp_ref - temp_reseau));
          }
      } else {
        vol_expl_100_75 = 0.0;
        }

      if (temp_100 >=temp_ref) {
        if (temp_75 >= temp_ref) {
          if (temp_50 >= temp_ref) {
            vol_expl_75_50 = vol_ref * (1 + (temp_moy_75_50 - temp_ref)/(temp_ref - temp_reseau));
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
            vol_expl_50_25 = vol_ref * (1 + (temp_moy_50_25 - temp_ref)/(temp_ref - temp_reseau));
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
            vol_expl_25_0 = vol_ref * (1 + (temp_moy_25_0 - temp_ref)/(temp_ref - temp_reseau));
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
