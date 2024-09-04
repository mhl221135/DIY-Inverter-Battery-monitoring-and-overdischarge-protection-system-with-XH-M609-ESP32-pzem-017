# DIY Inverter + Battery + Monitoring and Overdischarge Protection System with XH-M609, ESP32, and PZEM-017

This project demonstrates a DIY inverter, battery, and monitoring system with overdischarge protection. It utilizes the XH-M609 battery controller, a DC-DC stabilizer, and a PZEM-017 energy monitor connected to an ESP32 via an RS485-to-TTL converter. The system includes a relay that controls the power to the inverter, ensuring automatic power-on and power-off based on battery status.

## System Overview

- **XH-M609 Battery Controller**: Manages battery charging, discharging, and provides overdischarge protection. It controls the relay that powers the inverter.
- **DC-DC Stabilizer**: Converts voltage to a stable 12V for relay control.
- **PZEM-017**: Monitors electrical parameters like current, voltage, and power. It is connected to the ESP32 for data logging and monitoring.
- **ESP32**: Collects data from the PZEM-017 and integrates with ESPHome for remote monitoring and control.
- **Relay**: Powers the inverter on/off depending on the battery status as monitored by the XH-M609 controller.

## Features

- **Relay control** based on battery state (automatically powers on/off the inverter).
- **Energy monitoring** with PZEM-017.
- **ESPHome integration** for remote monitoring and control via customizable configuration.
- **Modbus communication** for adjusting the current range of the system based on the shunt used.

## Wiring Diagram

![Wiring Diagram](./wiring.png)


## Code Snippet for ESPHome Configuration

The following YAML code use pzemdc esphome library and comunicate with pzem017

```yaml
esphome:
  name: agm-bms
  platform: ESP32
  board: lolin32

wifi:
  networks:
    - ssid: "ssidname"
      password: "passwordforwifi"

captive_portal:

ota:
  platform: esphome
  password: "optionalpassword"  

logger:


web_server:
  port: 80


uart:
  tx_pin: GPIO17
  rx_pin: GPIO16
  baud_rate: 9600

sensor:
  - platform: uptime
    name: "Uptime"
    update_interval: 5s

  - platform: wifi_signal
    name: "WiFi strenght"
    update_interval: 5s

  - platform: pzemdc
    address: 0x01 #adress of pzemdc device 
    voltage:
      name: "DC-Voltage"
      id: DC_Voltage
      accuracy_decimals: 3
    current:
      name: "DC-Current"
      id: DC_Current
      accuracy_decimals: 3
    power:
      name: "DC-Power"
      id: DC_Power
      accuracy_decimals: 2
    energy:
      name: "DC-Energy"
      id: DC_Energy
      accuracy_decimals: 3
    id: pzemdc_1
    update_interval: 1s

  # Battery Percentage Sensor
  - platform: template
    name: "Battery SOC %"
    id: Battery_Percentage
    lambda: |-
      if (id(DC_Voltage).state < 10.5) return 0.0;
      if (id(DC_Voltage).state > 12.9) return 100.0;
      return (id(DC_Voltage).state - 10.5) / (12.9 - 10.5) * 100.0;
    unit_of_measurement: "%"
    accuracy_decimals: 1

button:
  - platform: template
    name: "Reset energy count"
    icon: "mdi:restart"
    id: pzemdc_reset_energy
    on_press:
      then:
        - pzemdc.reset_energy: pzemdc_1
```

The following YAML code use just modbus for comunication with pzem017 instead of esphome pzemdc library
here you can see how to change schunt current range to 300A (adjustable based on the shunt value in use):

```yaml
esphome:
  name: agm-bms-modbus
  platform: ESP32
  board: lolin32

wifi:
  networks:
    - ssid: "ssidname"
      password: "passwordforwifi"

captive_portal:

ota:
  platform: esphome
  password: "optionalpassword"  

logger:

web_server:
  port: 80

uart:
  id: uart4
  tx_pin: GPIO17
  rx_pin: GPIO16
  baud_rate: 9600
  stop_bits: 2

modbus:
  id: modbus1
  uart_id: uart4

modbus_controller:
  - id: modbus_device
    address: 0x01   # Address of the Modbus slave device on the bus
    modbus_id: modbus1
    setup_priority: -10

sensor:
  - platform: modbus_controller
    modbus_controller_id: modbus_device
    id: pzem_high_voltage_alarm
    name: "High Voltage Alarm Threshold"
    address: 0x0000
    unit_of_measurement: "V"
    register_type: holding
    value_type: U_WORD
    filters:
      - multiply: 0.01

  - platform: modbus_controller
    modbus_controller_id: modbus_device
    id: pzem_low_voltage_alarm
    name: "Low Voltage Alarm Threshold"
    address: 0x0001
    unit_of_measurement: "V"
    register_type: holding
    value_type: U_WORD

  - platform: modbus_controller
    modbus_controller_id: modbus_device
    id: pzem_slave_address
    name: "Slave Address"
    address: 0x0002
    unit_of_measurement: ""
    register_type: holding
    value_type: U_WORD

  - platform: modbus_controller
    modbus_controller_id: modbus_device
    id: pzem_current_range
    name: "Current Range"
    address: 0x0003
    unit_of_measurement: "A"
    register_type: holding
    value_type: U_WORD
    filters:
      - lambda: |-
          if (x == 0) return 100;
          else if (x == 1) return 50;
          else if (x == 2) return 200;
          else if (x == 3) return 300;
          else return x;

  - platform: modbus_controller
    modbus_controller_id: modbus_device
    name: "Current Range Setting"
    address: 0x0003
    register_type: holding
    value_type: U_WORD
    unit_of_measurement: "bit"

switch:
  - platform: modbus_controller
    modbus_controller_id: modbus_device
    name: "Reset Energy"
    address: 0x42
    register_type: holding
    write_lambda: |-
      return 1;

button:
  - platform: template
    name: "Set Current Range to 300A"
    on_press:
      then:
        - lambda: |-
            std::vector<uint16_t> current_range_payload = {0x0003};  // 300A value change to your schunt

            esphome::modbus_controller::ModbusController *controller = id(modbus_device);
            esphome::modbus_controller::ModbusCommandItem set_current_range_command =
                esphome::modbus_controller::ModbusCommandItem::create_write_single_command(controller, 0x0003, current_range_payload[0]);

            controller->queue_command(set_current_range_command);
            ESP_LOGI("ModbusLambda", "Set Current Range to 300A");

  - platform: template
    name: "Set ADRESS"
    on_press:
      then:
        - lambda: |-
            std::vector<uint16_t> current_range_payload = {0x01};  // adress change command

            esphome::modbus_controller::ModbusController *controller = id(modbus_device);
            esphome::modbus_controller::ModbusCommandItem set_current_range_command =
                esphome::modbus_controller::ModbusCommandItem::create_write_single_command(controller, 0x0002, current_range_payload[0]);

            controller->queue_command(set_current_range_command);
            ESP_LOGI("ModbusLambda", "Set ADRESS");



