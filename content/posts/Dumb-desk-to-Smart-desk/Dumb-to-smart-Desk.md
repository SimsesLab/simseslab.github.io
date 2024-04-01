+++
title = 'Dumb to a Smart desk'
date = 2024-03-30T14:59:23+01:00
draft = true
+++
# Turning my *dumb* standing desk into a smart standing desk

### A quick story about my little desk:
When I was in my late teens, I really wanted a standing desk. They weren't particularly easy or cheap to find in my area, but my father found an old standing desk at a business-furniture seller, and helped me with bargaining it down in price and transporting it home. It was scratched up, heavy, could only raise and lower with no positional memory, and the whole bottom of the desk was covered in dried buggers.

*But I loved it, and have loved it ever since.*

After getting used to having a standing desk, I decided to look into how I could incorporate position memory into it. First I looked into buying a new first party controller, but those were almost impossible to find. And what I could find was very expensive and not guarantied to work with my desk, because of the age of the desk's electronics.

## Fast forward to today, and the strategy of the project

I did look into communicating with the controller, but decided on going a much easier route:

The current controls was as basic as they could get: 3 wires, 1 for up, one for down, and the common ground you short with the given direction wire.

This as simple as I could want, and all I then had to do, was replace my buttons with a dual relay.

Billede af relæ


A little before this project, I had discovered and dived into [Home Assistant](https://www.home-assistant.io/) and smartening homes, and instantly wanted to see if I could connect my desk to Home Assistant.
Ideas like having my desk switch between standing and sitting based on how much time I have spent on the computer. Or the ability to change the sitting and standing height in Home Assistant.

It was in this idea phase that I found [ESPHome](https://esphome.io/index.html), and coincidentally found a much easier way of programming my desk. I had origially wanted to get my hands dirty with Arduino and C, but the way you program with ESPHome was much easier, especially for my purpose. I did to program a little in C with [Lambda functions](https://esphome.io/guides/automations.html#templates-lambdas), although that was very basic stuff, but I'll get to that later.











## Some math, physics and programming
Cool heading, but pretty basic stuff.

I tried some different things after getting the desk hooked up to Home Assistant. First I tried to do a *while loop* where the desk would move up or down, until the distance read grater or less than the desired distance.
This worked, but was very unstable, and required a bunch of weird workarounds.

For example: Sometimes the desk would just raise indefinitely. It would get stuck completely extended, and would be impossible to lower, because Home Assistant annoyingly don't have a way to halt an automation.
I later figured out, that one of the reasons for the desk to go into this apparent extention to space, was because Home Assistant was running the while loop thousends and thousends a times per second, and would report to the ESP controller *"We are not there yet! Keep going"*, which the poor controller would handle as fast as its slow little processor could, which meant, even when the desired height of the desk was reached and Home Assistant shutted up, the controller had an enormous backlog of request it was still handling as best as it could.

To fix this, I simply added a delay of some milliseconds. Now it worked about 30% of the time, but there was another problem, and one not so easily fixed.

The sensor itself was not very accurate when moving. It was quite fine when stationary, but as soon as the desk moved, it would report a deviation of ± 0.25 meter. This meant that the desk would stop halfway, because it apparently reached its desired height in some brief moment, before reporting back *"Oh! yeah no I'm nowhere near lol"*
I tried adding filters, decreasing the measure times, but it just made me realize what a broken solution it had become to what should be a simple task. What I ended up with was 20 per seconds filtered down to 5 median measurements send to Home Assistant every second, which made the logs saturated and useless. So I decided on finding a simpler solution:

This is what I ended up with, which was the route I should have taken from the start:

Measure the current height periodicly, like once every 5-10s or even less often
upon request to raise or lower the table, calculate the delta distance, and divide by the velocaty:
$$\frac{\Delta d}{v_{direction}}=\Delta t$$

So the basic structure of my program would be something like:
```
raise_table():
	relay_raise_table = on
	delay( (standing_height - current_height) / velocity_up )
	relay_raise_table = off
```

Now ESPHome uses YAML for its "programming", but its more configuring that than actual programming. This is great for simple tasks, but my simple math delay was sadly not simple enough for plain YAML configuration. Luckily ESPHome gives the ability to use Lambda functions, which gave me what I needed:


```yaml
button:
  - platform: template
    id: move_to_stand
    on_press:
      then:
        - output.turn_on: relay_raise_table
        - delay: !lambda "if (id(height_sensor).state < id(standing_height_set).state) return (((id(standing_height_set).state - id(height_sensor).state)/(id(up_vel))))*1000; else return 0;"
        - output.turn_off: relay_raise_table
```




## Final notes and files
My final ESPHome yaml file:
```yaml
esphome:
  name: smartdumbdesk
  friendly_name: SmartDumbDesk

esp8266:
  board: d1_mini

# Enable logging
logger:

# Enable Home Assistant API
api:
  encryption:
    key: "<generated>"

ota:
  password: "<generated>"

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

# # Enable fallback hotspot (captive portal) in case wifi connection fails
#  ap:
#    ssid: "SmartdumbDesk Fallback Hotspot"
#    password: "<fallback wifi password>"

  manual_ip:
    static_ip: <static IP for the device>
    gateway: <gateway address>
    subnet: 255.255.255.0

captive_portal:


globals:
  - id: down_vel
    type: float
    initial_value: '0.0485' # m/s
  - id: up_vel
    type: float
    initial_value: '0.0465' # m/s
  
  - id: table_sensor_offset
    type: float
    initial_value: '0.05' # 50mm offset between tabletop and what the sensor reads

# GPIO
## Relay and LED
output:
  - id: relay_raise_table
    platform: gpio
    pin:
      number: D7
      inverted: true

  - id: relay_lower_table
    platform: gpio
    pin:
      number: D4
      inverted: true


## HC-SR04 Ultra-sonic distance sensor
sensor:
  - platform: ultrasonic
    id: height_sensor
    trigger_pin: D3
    echo_pin: D8
    name: "Table height"
    update_interval: 5s
    accuracy_decimals: 2
    unit_of_measurement: m
    filters:
    - median:
        window_size: 9
        send_every: 4
        send_first_at: 3
    - offset: 0.05 # 50mm offset between tabletop and what the sensor reads


# Physical buttons
binary_sensor:
## Physical button to raise the table
  - platform: gpio
    name: "raise_table_button"
    internal: true
    pin: 
      number: D1
      mode: INPUT_PULLUP
      inverted: true
    filters:
      - delayed_on: 10ms
    on_press:
      then:
        - output.turn_on: relay_raise_table
    on_release:
      then:
        - output.turn_off: relay_raise_table

## Physical button to lower the table
  - platform: gpio
    name: "lower_table_button"
    internal: true
    pin:
      number: D2
      mode: INPUT_PULLUP
      inverted: true
    filters:
      - delayed_on: 10ms
    on_press:
      then:
        - output.turn_on: relay_lower_table
    on_release:
      then:
        - output.turn_off: relay_lower_table

## Physical button for standing position
  - platform: gpio
    id: to_stand_pos
    pin:
      number: D5
      mode: INPUT_PULLUP
      inverted: true
    filters:
      - delayed_on: 10ms
    on_double_click:
      min_length: 50ms
      max_length: 350ms
      then:
          button.press: move_to_stand

## Physical button for sitting position
  - platform: gpio
    id: to_sit_pos
    pin:
      number: D6
      mode: INPUT_PULLUP
      inverted: true
    filters:
      - delayed_on: 10ms
    on_double_click:
      min_length: 50ms
      max_length: 350ms
      then:
          button.press: move_to_sit

# Functions for standing and sitting positions
button:
  - platform: template
    id: move_to_stand
    on_press:
      then:
      # Debugging.
        - lambda: |- 
            ESP_LOGD("custom", "id(height_sensor).state: %f", id(height_sensor).state);
        - lambda: |- 
            ESP_LOGD("custom", "id(standing_height_set): %f", id(standing_height_set).state);
        - lambda: |- 
            ESP_LOGD("custom", "Delta = standing_height_set-height_sensor: %f", (id(standing_height_set).state - id(height_sensor).state));
        - lambda: |- 
            ESP_LOGD("custom", "id(up_vel): %f", id(up_vel));
        - lambda: |- 
            ESP_LOGD("custom", "Delta / up_vel: %f", ((id(standing_height_set).state - id(height_sensor).state)/(id(up_vel))));
      # Actual actions.
        - output.turn_on: relay_raise_table
        - delay: !lambda "if (id(height_sensor).state < id(standing_height_set).state) return (((id(standing_height_set).state - id(height_sensor).state)/(id(up_vel))))*1000; else return 0;"
        - output.turn_off: relay_raise_table

  - platform: template
    id: move_to_sit
    on_press:
      then:
      # Debugging.
        - lambda: |- 
            ESP_LOGD("custom", "id(height_sensor).state: %f", id(height_sensor).state);
        - lambda: |- 
            ESP_LOGD("custom", "id(sitting_height_set): %f", id(sitting_height_set).state);
        - lambda: |- 
            ESP_LOGD("custom", "Delta = sitting_height_set-height_sensor: %f", (id(sitting_height_set).state - id(height_sensor).state));
        - lambda: |- 
            ESP_LOGD("custom", "id(down_vel): %f", id(down_vel));
        - lambda: |- 
            ESP_LOGD("custom", "Delta / down_vel * 1000: %f", ((id(sitting_height_set).state - id(height_sensor).state)/(id(down_vel))*1000));
      # Actual actions.
        - output.turn_on: relay_lower_table
        - delay: !lambda "if (id(height_sensor).state > id(sitting_height_set).state) return (((id(height_sensor).state - id(sitting_height_set).state)/(id(down_vel))))*1000; else return 0;"
        - output.turn_off: relay_lower_table


# Buttons and sliders in Home Assistant
## Buttons to positions
  - platform: template
    name: "Raise table"
    icon: "mdi:human-male"
    on_press:
      then:
        button.press: move_to_stand

  - platform: template
    name: "Lower table"
    icon: "mdi:table-chair"
    on_press:
      then:
        button.press: move_to_sit

## Sliders in Home Assistant
number:
  - platform: template
    name: "Standing height"
    id: standing_height_set
    min_value: 0.60
    max_value: 1.1
    initial_value: 1.05
    step: 0.05
    unit_of_measurement: m
    icon: "mdi:arrow-collapse-up"
    optimistic: true
    restore_value: true

  - platform: template
    name: "Sitting height"
    id: sitting_height_set
    min_value: 0.60
    max_value: 1.1
    initial_value: 0.75
    step: 0.05
    unit_of_measurement: m
    icon: "mdi:arrow-expand-down"
    optimistic: true
    restore_value: true
```
