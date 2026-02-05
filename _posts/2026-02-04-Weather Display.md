---
title: Weather Display
date: 2026-02-04 19:12:00 -0600
categories: ["Indoor Projects", "Weather Display"]
tags: ["indoor projects", "e-paper", "display", "weather", "home assistant", "esphome", "e-ink"]
author: 1
description: Creating a framed e-paper weather display
toc: false
comments: true
---

# Home Assistant connected weather display
![E-Paper image](PXL_20260124_122417371.jpg){: w="400" }

## The beginning
This idea started with a few different needs. We're always asking Google the current temperature in the mornings, we've also switched ovens and no longer have a clock in our kitchen, and I also wanted a way to display certain home automation information. This 
led me to what some others were doing with displays. I didn't really like the look of most of them until I came across what some people were doing with e-ink/e-paper displays. As a start, I figured just showing some weather and time would be great and so I found
the following blog which heavily influenced by beginning code for integrating into Home Assistant as well as display size and esp code.
<a href="https://blog.wijman.net/e-ink-weather-frame-with-esphome-and-home-assistant/" target="_blank">https://blog.wijman.net/e-ink-weather-frame-with-esphome-and-home-assistant/</a>

## The process
After purchasing the various components and frame, I started off getting the sensor information added into Home Assistant. This involved installing the Studio Code Server add-on to edit the configuration files for Home Assistant and adding a test weather sensor first.
```
- triggers:
    - trigger: time_pattern
      minutes: "/30"
  actions:
    - action: weather.get_forecasts
      data:
        type: hourly
      target:
        entity_id: weather.forecast_home
      response_variable: hourly
  sensor:
    - name: Weather Forecast Hourly
      unique_id: weather_forecast_hourly
      state: "{{ now().isoformat() }}"
      attributes:
        forecast: "{{ hourly['weather.forecast_home'].forecast }}"
```
Once the test weather sensor showed up in HA, I proceeded to install ESPHome add-on that would allow uploading code to the esp32 board that drives the display. The first run of the add-on asked a few questions that uploaded wifi and other basic info onto the esp32 board. Once 
I had that, it allows wirelessly updating the board. To start, I had to get a basic yaml set up to start the display. This required gathering basic information about the display to configure the board to correctly show text. I used 
<a href="https://esphome.io/components/display/waveshare_epaper/#configuration-variables" target="_blank">https://esphome.io/components/display/waveshare_epaper/#configuration-variables</a> which had most of what I needed. Then, I 
created a simple test and realized I didn't set the screen to refresh, so it never displayed. 😄 I quickly updated the test to have a Home Assistant button to refresh the screen manually while testing.
```
# This section gives you three buttons in Home Assistant to pilot it from Home Assistant
button:
  - platform: shutdown
    name: "Eink-Display - Shutdown"
  - platform: restart
    name: "Eink-Display - Restart"
  - platform: template
    name: "Eink-Display - Refresh Screen"
    entity_category: config
    on_press:
      - script.execute: update_screen

# This section exposes sensors to Home Assistant, so you know the state of your screen from your Home Assistant   
# My variables to update
globals:
  - id: recorded_display_refresh
    type: int
    restore_value: yes
    initial_value: "0"
sensor:
  # When was it updated ?
  - platform: template
    name: "Eink-Display - Last Update"
    device_class: timestamp
    entity_category: "diagnostic"
    id: display_last_update

  # How many time was it refreshed since the beginning ? E-ink screens lifetime mainly depends on that.
  - platform: template
    name: "Eink-Display - Recorded Display Refresh"
    accuracy_decimals: 0
    unit_of_measurement: "Refreshes"
    state_class: "total_increasing"
    entity_category: "diagnostic"
    lambda: "return id(recorded_display_refresh);"

  # How is the Wifi signal ?
  - platform: wifi_signal
    name: "Eink-Display - WiFi Signal Strength"
    id: wifisignal
    unit_of_measurement: "dBm"
    entity_category: "diagnostic"
    update_interval: 60s

# Script for updating screen - Refresh display and publish refresh count and time.
script:
  - id: update_screen
    then:
      - logger.log: "Updating screen" # "FYI ESPHome"
      - component.update: eink_display # Do the job
      - lambda: "id(recorded_display_refresh) += 1;" # Increment our counter

# Include custom fonts from GFonts. You can also use local custom files but 'gfonts://' means it will be downloaded by ESPHome and added to your board directly.
# Find your font on https://fonts.google.com/
font:
  - file: "gfonts://Roboto"
    id: roboto
    size: 20
  - file: "gfonts://Roboto"
    id: roboto_title
    size: 50
  - file: 'gfonts://Material+Symbols+Outlined' # Use mdi ! You need to declare the glyphs number from https://fonts.google.com/icons 
    id: mdi_title
    size: 50
    glyphs: ["\U0000F6E9"] # heart-ecg. To understand where to find the glyph number, look at the bottom of the right panel on https://fonts.google.com/icons?selected=Material+Symbols+Outlined:ecg_heart:FILL@0;wght@400;GRAD@0;opsz@24&icon.query=ecg+heart

spi:
  clk_pin: GPIO13
  mosi_pin: GPIO14

display:
  - platform: waveshare_epaper
    id: eink_display
    cs_pin: GPIO15
    dc_pin: GPIO27
    busy_pin:
      number: GPIO25
      inverted: true
    reset_pin: GPIO26
    reset_duration: 20ms
    model: 7.50inV2
    update_interval: never
    rotation: 90°
    lambda: |-
      // Map weather states to MDI characters !
      std::map<std::string, std::string> weather_icon_map
        {
          {"heart-ecg", "\U0000F6E9"}
        };

      // PRINT ICON: x / y / font family / Alignment method / content (in this case an icon from mdi)
      it.printf(240, 330, id(mdi_title), TextAlign::TOP_CENTER, weather_icon_map["heart-ecg"].c_str());
      
      // PRINT TEXT: x / y / font family / Alignment method / content
      it.printf(240, 390, id(roboto), TextAlign::TOP_CENTER, "Success...");
```

After successfully testing the screen worked and it connected to Home Assistant, I basically copied the same layout and weather settings from Stephan's blog. The biggest changes being I used Google for the fonts and icons and also added a clock. A big thing 
with e-paper/e-ink displays is the refresh count though. So, I've added some automation in Home Assistant to turn it on and off automatically while also trying to keep the partial refreshes to a minimum, which is difficult when you want an accurate clock. Below is where I ended up 
with the display config. Enjoy! I'll update everyone as I continue making changes for various things on the display.

```
esphome:
  name: eink-display
  friendly_name: eink display

esp32:
  board: esp32dev
  framework:
    type: esp-idf

# Enable logging
logger:

# Enable Home Assistant API
api:
  encryption:
    key: ""

ota:
  - platform: esphome
    password: ""

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

  # Enable fallback hotspot (captive portal) in case wifi connection fails
  ap:
    ssid: "Eink-Display Fallback Hotspot"
    password: ""

captive_portal:
    
button:
  - platform: template
    name: "Eink - Clear"
    entity_category: config
    on_press:
      then:
        - lambda: id(display_mode) = 0;
  - platform: template
    name: "Eink - Show"
    entity_category: config
    on_press:
      then:
        - lambda: id(display_mode) = 1;
  - platform: template
    name: "Eink - Enable Refresh"
    entity_category: config
    on_press:
      then:
        - lambda: id(enable_refresh) = true;
  - platform: template
    name: "Eink - Disable Refresh"
    entity_category: config
    on_press:
      then:
        - lambda: id(enable_refresh) = false;

globals:
  - id: recorded_display_refresh
    type: int
    restore_value: yes
    initial_value: "0"
  - id: enable_refresh
    type: bool
    restore_value: yes
    initial_value: 'false'
  - id: display_mode
    type: int
    restore_value: yes
    initial_value: '0'

binary_sensor:
  # Check if ESPHome is connected
  - platform: status
    id: connection_status

sensor:
  # When was it updated ?
  - platform: template
    name: "Eink-Display - Last Update"
    device_class: timestamp
    entity_category: "diagnostic"
    id: display_last_update

  # How many time was it refreshed since the beginning ? E-ink screens lifetime mainly depends on that.
  - platform: template
    name: "Eink-Display - Recorded Display Refresh"
    accuracy_decimals: 0
    unit_of_measurement: "Refreshes"
    state_class: "total_increasing"
    entity_category: "diagnostic"
    lambda: "return id(recorded_display_refresh);"

  - platform: wifi_signal
    name: "Eink-Display - WiFi Signal Strength"
    id: wifisignal
    unit_of_measurement: "dBm"
    entity_category: "diagnostic"
    update_interval: 120s

  - platform: homeassistant
    entity_id: sensor.weatherframe_data
    attribute: weather_now_temperature
    id: weather_now_temperature

  - platform: homeassistant
    entity_id: sensor.weatherframe_data
    attribute: weather_hourly_temperature_0
    id: weather_hourly_temperature_0

  - platform: homeassistant
    entity_id: sensor.weatherframe_data
    attribute: weather_hourly_temperature_1
    id: weather_hourly_temperature_1

  - platform: homeassistant
    entity_id: sensor.weatherframe_data
    attribute: weather_hourly_temperature_2
    id: weather_hourly_temperature_2

  - platform: homeassistant
    entity_id: sensor.weatherframe_data
    attribute: weather_hourly_temperature_3
    id: weather_hourly_temperature_3

  - platform: homeassistant
    entity_id: sensor.weatherframe_data
    attribute: weather_daily_temperature_0
    id: weather_daily_temperature_0

  - platform: homeassistant
    entity_id: sensor.weatherframe_data
    attribute: weather_daily_templow_0
    id: weather_daily_templow_0

  - platform: homeassistant
    entity_id: sensor.weatherframe_data
    attribute: weather_daily_temperature_1
    id: weather_daily_temperature_1

  - platform: homeassistant
    entity_id: sensor.weatherframe_data
    attribute: weather_daily_templow_1
    id: weather_daily_templow_1

  - platform: homeassistant
    entity_id: sensor.weatherframe_data
    attribute: weather_daily_temperature_2
    id: weather_daily_temperature_2

  - platform: homeassistant
    entity_id: sensor.weatherframe_data
    attribute: weather_daily_templow_2
    id: weather_daily_templow_2

  - platform: homeassistant
    entity_id: sensor.weatherframe_data
    attribute: weather_daily_temperature_3
    id: weather_daily_temperature_3

  - platform: homeassistant
    entity_id: sensor.weatherframe_data
    attribute: weather_daily_templow_3
    id: weather_daily_templow_3

text_sensor:
  - platform: homeassistant
    entity_id: sensor.weatherframe_data
    attribute: weather_now_condition
    id: weather_now_condition

  - platform: homeassistant
    entity_id: sensor.weatherframe_data
    attribute: weather_hourly_condition_0
    id: weather_hourly_condition_0

  - platform: homeassistant
    entity_id: sensor.weatherframe_data
    attribute: weather_hourly_timestamp_0
    id: weather_hourly_timestamp_0

  - platform: homeassistant
    entity_id: sensor.weatherframe_data
    attribute: weather_hourly_condition_1
    id: weather_hourly_condition_1

  - platform: homeassistant
    entity_id: sensor.weatherframe_data
    attribute: weather_hourly_timestamp_1
    id: weather_hourly_timestamp_1

  - platform: homeassistant
    entity_id: sensor.weatherframe_data
    attribute: weather_hourly_condition_2
    id: weather_hourly_condition_2

  - platform: homeassistant
    entity_id: sensor.weatherframe_data
    attribute: weather_hourly_timestamp_2
    id: weather_hourly_timestamp_2

  - platform: homeassistant
    entity_id: sensor.weatherframe_data
    attribute: weather_hourly_condition_3
    id: weather_hourly_condition_3

  - platform: homeassistant
    entity_id: sensor.weatherframe_data
    attribute: weather_hourly_timestamp_3
    id: weather_hourly_timestamp_3

  - platform: homeassistant
    entity_id: sensor.weatherframe_data
    attribute: weather_daily_condition_0
    id: weather_daily_condition_0

  - platform: homeassistant
    entity_id: sensor.weatherframe_data
    attribute: weather_daily_timestamp_0
    id: weather_daily_timestamp_0

  - platform: homeassistant
    entity_id: sensor.weatherframe_data
    attribute: weather_daily_condition_1
    id: weather_daily_condition_1

  - platform: homeassistant
    entity_id: sensor.weatherframe_data
    attribute: weather_daily_timestamp_1
    id: weather_daily_timestamp_1

  - platform: homeassistant
    entity_id: sensor.weatherframe_data
    attribute: weather_daily_condition_2
    id: weather_daily_condition_2

  - platform: homeassistant
    entity_id: sensor.weatherframe_data
    attribute: weather_daily_timestamp_2
    id: weather_daily_timestamp_2

  - platform: homeassistant
    entity_id: sensor.weatherframe_data
    attribute: weather_daily_condition_3
    id: weather_daily_condition_3

  - platform: homeassistant
    entity_id: sensor.weatherframe_data
    attribute: weather_daily_timestamp_3
    id: weather_daily_timestamp_3

  - platform: template
    name: "Current Time"
    id: current_time
    update_interval: 20s
    lambda: return  id(homeassistant_time).now().strftime("%I:%M %p");

time:
  - platform: homeassistant
    id: homeassistant_time
    on_time:
      - seconds: 30
        then:
          - script.execute: update_screen

script:
  - id: update_screen
    then:
      - if:
          condition:
            lambda: "return id(enable_refresh);"
          then:
            - logger.log: "Updating screen"
            - component.update: eink_display
            - lambda: "id(recorded_display_refresh) += 1;" # Increment our counter

# Include custom fonts from GFonts. You can also use local custom files but 'gfonts://' means it will be downloaded by ESPHome and added to your board directly.
# Find your font on https://fonts.google.com/
# To understand where to find the glyph number, look at the bottom of the right panel on https://fonts.google.com/icons?selected=Material+Symbols+Outlined:ecg_heart:FILL@0;wght@400;GRAD@0;opsz@24&icon.query=ecg+heart
font:
  - file: "gfonts://Roboto"
    id: roboto
    size: 20
  - file: "gfonts://Roboto"
    id: roboto_title
    size: 50
  - file: "gfonts://Material+Symbols+Outlined" # Use mdi ! You need to declare the glyphs number from https://fonts.google.com/icons under code point
    id: font_mdi_large
    size: 80
    glyphs: &mdi-weather-glyphs
      - "\U0000E2BD" # mdi-weather-cloudy
      - "\U0000E818" # mdi-weather-fog
      - "\U0000F67F" # mdi-weather-hail
      - "\U0000E188" # mdi-weather-hazy
      - "\U0000EBD5" # mdi-weather-hurricane
      - "\U0000EBDB" # mdi-weather-lightning
      - "\U0000F34F" # mdi-weather-clear-night
      - "\U0000F174" # mdi-weather-night-partly-cloudy
      - "\U0000F172" # mdi-weather-partly-cloudy
      - "\U0000F60B" # mdi-weather-partly-snowy
      - "\U0000F61F" # mdi-weather-pouring
      - "\U0000F176" # mdi-weather-rainy
      - "\U0000E2CD" # mdi-weather-snowy
      - "\U0000E81A" # mdi-weather-sunny
      - "\U0000E1C6" # mdi-weather-sunset
      - "\U0000E199" # mdi-weather-tornado
      - "\U0000EFD8" # mdi-weather-windy
  - file: "gfonts://Material+Symbols+Outlined"
    id: font_mdi_medium
    size: 36
    glyphs: *mdi-weather-glyphs
  - file: "gfonts://Material+Symbols+Outlined"
    id: font_mdi_status
    size: 18
    glyphs: &mdi-status-glyphs
      - "\U0000F067" # mdi:wifi-strength-0
      - "\U0000EBE4" # mdi:wifi-strength-1
      - "\U0000EBD6" # mdi:wifi-strength-2
      - "\U0000EBE1" # mdi:wifi-strength-3
      - "\U0000E1BA" # mdi:wifi-strength-4
      - "\U0000F063" # mdi:wifi-strength-off
      - "\U0000E88A" # mdi-home
      - "\U0000E4B9" # mdi-home-alert
      - "\U0000EBD2" # mdi-battery-high
      - "\U0000EBE0" # mdi-battery-medium
      - "\U0000EBDC" # mdi-battery-low
      - "\U0000F7EA" # mdi-battery-off-outline
      - "\U0000E855" # mdi-clock
      - "\U0000E857" # mdi-clock-remove

spi:
  clk_pin: GPIO13
  mosi_pin: GPIO14

display:
  - platform: waveshare_epaper
    id: eink_display
    cs_pin: GPIO15
    dc_pin: GPIO27
    busy_pin:
      number: GPIO25
      inverted: true
    reset_pin: GPIO26
    reset_duration: 20ms
    model: 7.50inv2p
    update_interval: never
    full_update_every: 10
    rotation: 90°
    lambda: |-
      if (id(display_mode) != 0) {
        // Map weather states to MDI characters.
        std::map<std::string, std::string> weather_icon_map
        {
          {"cloudy", "\U0000E2BD"},
          {"cloudy-alert", "\U0000E2BD"},
          {"cloudy-arrow-right", "\U0000E2BD"},
          {"fog", "\U0000E818"},
          {"hail", "\U0000F67F"},
          {"hazy", "\U0000E188"},
          {"hurricane", "\U0000EBD5"},
          {"lightning", "\U0000EBDB"},
          {"lightning-rainy", "\U0000EBDB"},
          {"clear-night", "\U0000F34F"},
          {"night", "\U0000F34F"},
          {"night-partly-cloudy", "\U0000F174"},
          {"partlycloudy", "\U0000F172"},
          {"partly-lightning", "\U0000EBDB"},
          {"partly-rainy", "\U0000F176"},
          {"partly-snowy", "\U0000E2CD"},
          {"partly-snowy-rainy", "\U0000F60B"},
          {"pouring", "\U0000F61F"},
          {"rainy", "\U0000F176"},
          {"snowy", "\U0000E2CD"},
          {"snowy-heavy", "\U0000E2CD"},
          {"snowy-rainy", "\U0000F60B"},
          {"sunny", "\U0000E81A"},
          {"sunny-alert", "\U0000E81A"},
          {"sunny-off", "\U0000E81A"},
          {"sunset", "\U0000E1C6"},
          {"sunset-down", "\U0000E1C6"},
          {"sunset-up", "\U0000E1C6"},
          {"tornado", "\U0000E199"},
          {"windy", "\U0000EFD8"},
          {"windy-variant", "\U0000EFD8"},
        };
        // Weather Current + Hourly
        if (id(wifisignal).state >= -30)  {
          it.print(40, 60, id(font_mdi_status), COLOR_ON, "\U0000E1BA");
        } else if (id(wifisignal).state < -30 && id(wifisignal).state >= -67) {
          it.print(40, 60, id(font_mdi_status), COLOR_ON, "\U0000EBE1");
        } else if (id(wifisignal).state < -67 && id(wifisignal).state >= -70) {
          it.print(40, 60, id(font_mdi_status), COLOR_ON, "\U0000EBD6");
        } else if (id(wifisignal).state < -70 && id(wifisignal).state >= -80) {
          it.print(40, 60, id(font_mdi_status), COLOR_ON, "\U0000EBE4");
        } else {
          it.print(40, 60, id(font_mdi_status), COLOR_ON, "\U0000F067");
        } 

        if(id(connection_status).state) {
            it.print(40, 80, id(font_mdi_status), COLOR_ON, "\U0000E88A");
        } else {
            it.print(40, 80, id(font_mdi_status), COLOR_ON, "\U0000E4B9");
        }

        it.printf(255, 74, id(roboto_title), COLOR_ON, TextAlign::TOP_CENTER, "%s", id(current_time).state.c_str());
        it.printf(255, 148, id(roboto), COLOR_ON, TextAlign::TOP_CENTER, "CURRENT");
        it.printf(115, 198, id(font_mdi_large), COLOR_ON, TextAlign::TOP_CENTER, "%s", weather_icon_map[id(weather_now_condition).state.c_str()].c_str());
        if (id(weather_now_temperature).state <= 0) {
          it.printf(331, 204, id(roboto_title), COLOR_ON, TextAlign::TOP_CENTER, "%2.0f°F", id(weather_now_temperature).state);
          it.printf(328, 201, id(roboto_title), COLOR_ON, TextAlign::TOP_CENTER, "%2.0f°F", id(weather_now_temperature).state);
          it.printf(325, 198, id(roboto_title), COLOR_ON, TextAlign::TOP_CENTER, "%2.0f°F", id(weather_now_temperature).state);
        } else {
          it.printf(325, 198, id(roboto_title), COLOR_ON, TextAlign::TOP_CENTER, "%2.0f°F", id(weather_now_temperature).state);
        }
        it.printf(255, 318, id(roboto), COLOR_ON, TextAlign::TOP_CENTER, "HOURLY");
        it.printf(120, 372, id(roboto), COLOR_ON, TextAlign::TOP_CENTER, "%s", id(weather_hourly_timestamp_0).state.c_str());
        it.printf(120, 396, id(font_mdi_medium), COLOR_ON, TextAlign::TOP_CENTER, "%s", weather_icon_map[id(weather_hourly_condition_0).state.c_str()].c_str());
        it.printf(120, 440, id(roboto), COLOR_ON, TextAlign::TOP_CENTER, "%2.0f°F", id(weather_hourly_temperature_0).state);
        it.printf(210, 372, id(roboto), COLOR_ON, TextAlign::TOP_CENTER, "%s", id(weather_hourly_timestamp_1).state.c_str());
        it.printf(210, 396, id(font_mdi_medium), COLOR_ON, TextAlign::TOP_CENTER, "%s", weather_icon_map[id(weather_hourly_condition_1).state.c_str()].c_str());
        it.printf(210, 440, id(roboto), COLOR_ON, TextAlign::TOP_CENTER, "%2.0f°F", id(weather_hourly_temperature_1).state);
        it.printf(300, 372, id(roboto), COLOR_ON, TextAlign::TOP_CENTER, "%s", id(weather_hourly_timestamp_2).state.c_str());
        it.printf(300, 396, id(font_mdi_medium), COLOR_ON, TextAlign::TOP_CENTER, "%s", weather_icon_map[id(weather_hourly_condition_2).state.c_str()].c_str());
        it.printf(300, 440, id(roboto), COLOR_ON, TextAlign::TOP_CENTER, "%2.0f°F", id(weather_hourly_temperature_2).state);
        it.printf(390, 372, id(roboto), COLOR_ON, TextAlign::TOP_CENTER, "%s", id(weather_hourly_timestamp_3).state.c_str());
        it.printf(390, 396, id(font_mdi_medium), COLOR_ON, TextAlign::TOP_CENTER, "%s", weather_icon_map[id(weather_hourly_condition_3).state.c_str()].c_str());
        it.printf(390, 440, id(roboto), COLOR_ON, TextAlign::TOP_CENTER, "%2.0f°F", id(weather_hourly_temperature_3).state);
        //it.printf(135, 372, id(roboto), COLOR_ON, TextAlign::TOP_CENTER, "%s", id(weather_hourly_timestamp_0).state.c_str());
        //it.printf(135, 402, id(font_mdi_large), COLOR_ON, TextAlign::TOP_CENTER, "%s", weather_icon_map[id(weather_hourly_condition_0).state.c_str()].c_str());
        //it.printf(135, 498, id(roboto), COLOR_ON, TextAlign::TOP_CENTER, "%2.0f°F", id(weather_hourly_temperature_0).state);
        //it.printf(255, 372, id(roboto), COLOR_ON, TextAlign::TOP_CENTER, "%s", id(weather_hourly_timestamp_1).state.c_str());
        //it.printf(255, 402, id(font_mdi_large), COLOR_ON, TextAlign::TOP_CENTER, "%s", weather_icon_map[id(weather_hourly_condition_1).state.c_str()].c_str());
        //it.printf(255, 378, id(roboto), COLOR_ON, TextAlign::TOP_CENTER, "%2.0f°F", id(weather_hourly_temperature_1).state);
        //it.printf(375, 372, id(roboto), COLOR_ON, TextAlign::TOP_CENTER, "%s", id(weather_hourly_timestamp_2).state.c_str());
        //it.printf(375, 402, id(font_mdi_large), COLOR_ON, TextAlign::TOP_CENTER, "%s", weather_icon_map[id(weather_hourly_condition_2).state.c_str()].c_str());
        //it.printf(375, 498, id(roboto), COLOR_ON, TextAlign::TOP_CENTER, "%2.0f°F", id(weather_hourly_temperature_2).state);
        // Weather Forecast
        it.printf(255, 504, id(roboto), COLOR_ON, TextAlign::TOP_CENTER, "OUTLOOK");
        if (id(homeassistant_time).now().hour < 12) {
          it.printf(135, 552, id(roboto), COLOR_ON, TextAlign::TOP_CENTER, "%s", id(weather_daily_timestamp_0).state.c_str());
          it.printf(135, 582, id(font_mdi_large), COLOR_ON, TextAlign::TOP_CENTER, "%s", weather_icon_map[id(weather_daily_condition_0).state.c_str()].c_str());
          it.printf(135, 678, id(roboto), COLOR_ON, TextAlign::TOP_CENTER, "%2.0f°F", id(weather_daily_temperature_0).state);
          it.printf(255, 552, id(roboto), COLOR_ON, TextAlign::TOP_CENTER, "%s", id(weather_daily_timestamp_1).state.c_str());
          it.printf(255, 582, id(font_mdi_large), COLOR_ON, TextAlign::TOP_CENTER, "%s", weather_icon_map[id(weather_daily_condition_1).state.c_str()].c_str());
          it.printf(255, 678, id(roboto), COLOR_ON, TextAlign::TOP_CENTER, "%2.0f°F", id(weather_daily_temperature_1).state);
          it.printf(375, 552, id(roboto), COLOR_ON, TextAlign::TOP_CENTER, "%s", id(weather_daily_timestamp_2).state.c_str());
          it.printf(375, 582, id(font_mdi_large), COLOR_ON, TextAlign::TOP_CENTER, "%s", weather_icon_map[id(weather_daily_condition_2).state.c_str()].c_str());
          it.printf(375, 678, id(roboto), COLOR_ON, TextAlign::TOP_CENTER, "%2.0f°F", id(weather_daily_temperature_2).state);
        } else {
          it.printf(135, 552, id(roboto), COLOR_ON, TextAlign::TOP_CENTER, "%s", id(weather_daily_timestamp_1).state.c_str());
          it.printf(135, 582, id(font_mdi_large), COLOR_ON, TextAlign::TOP_CENTER, "%s", weather_icon_map[id(weather_daily_condition_1).state.c_str()].c_str());
          it.printf(135, 678, id(roboto), COLOR_ON, TextAlign::TOP_CENTER, "%2.0f°F", id(weather_daily_temperature_1).state);
          it.printf(255, 552, id(roboto), COLOR_ON, TextAlign::TOP_CENTER, "%s", id(weather_daily_timestamp_2).state.c_str());
          it.printf(255, 582, id(font_mdi_large), COLOR_ON, TextAlign::TOP_CENTER, "%s", weather_icon_map[id(weather_daily_condition_2).state.c_str()].c_str());
          it.printf(255, 678, id(roboto), COLOR_ON, TextAlign::TOP_CENTER, "%2.0f°F", id(weather_daily_temperature_2).state);
          it.printf(375, 552, id(roboto), COLOR_ON, TextAlign::TOP_CENTER, "%s", id(weather_daily_timestamp_3).state.c_str());
          it.printf(375, 582, id(font_mdi_large), COLOR_ON, TextAlign::TOP_CENTER, "%s", weather_icon_map[id(weather_daily_condition_3).state.c_str()].c_str());
          it.printf(375, 678, id(roboto), COLOR_ON, TextAlign::TOP_CENTER, "%2.0f°F", id(weather_daily_temperature_3).state);
        }
      }
```

Thanks!
-Cole
