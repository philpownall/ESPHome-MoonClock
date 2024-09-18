An ESPHome Moon Clock
=====================
by Phil Pownall


This Moon Clock shows the Moon Phase using a globe, and the Moon Azimuth and Elevation on an attached display.
The device consists of a (Moon) globe printed in two different color hemispheres (I chose Blue and transparent White).
The Moon globe rotates during the lunar month, and the internal LED lights up when the Moon is visible above the horizon.
The device uses ESPHome to program an ESP32, and the IPGeolocation Astronomy API.

[Moon Clock](MoonClock.png)

### Ingredients
- An ESP32 device running ESPHome
- A 608ZZ bearing
- A SG90 or MG90 180-degree servo motor
- A Yellow or Blue LED
- An SSD1306 128x64 display
- A 3D-printed mount consisting of a base, a cog, a platform and a globe

### Optional ingredients
- An ESP32 Expansion board to supply power and additional pins
- A 9V power adapter for the Expansion board


The Mount
---------

The mount was designed in TinkerCAD, providing a base for the SG90 servo
STL files for the design are available in this repository: 
[TinkerCAD Moon Clock v2] (https://www.tinkercad.com/things/8dPIg13zxnB-moon-clock-v2)

The platform consists of a 1X cog and a Moon Globe.  The platform and the !X Cog have a hole in the centre for the LED wires. 
The platform sits on top of the bearing which is in the top of the base.
The 2X cog sits on top of the servo, and interlocks with the 1X cog.

To set up the tracker, 
- Set the servo to its centre position (180 degrees) using the number slider
- Position the base with the Moon Globe with the Full Moon (white side facing you), and 
- Tighten the 2X cog screw into the servo.
- enter your location (latitude and Longitude) and the IPGeolocation Astronomy API key into your secrets file
- compile and install the MoonClock.yaml into your ESP32 device
- plug the ESP32 adapter in to start the Moon Phase Tracker


The ESPHome code
================

The ESPHome yaml code consists of the following key sections:
- the *get_attributes* script which uses *http_request* to fetch the Moon parameters from the IPGeo Astronomy API
- the servo number template which updates the servo position with the Moon Phase Angle
- the display code
- a time automation to update the Moon position

Get_Attributes Script
---------------------

The script uses *http_request* to fetch the current Moon position and phase (for your location) from the IPGeolocation Astronomy API (in a json format).
The json::parse_json* function is used to parse the json, and set the display variables and the Moon Phase servo position.

### Example Script Definition

```
script:
  - id: get_attributes
    then:
      - logger.log: 
          format: "moon-clock: Get Request initiated"
          level: INFO
      - http_request.get:
          capture_response: true
          # max_response_buffer_size: 1024
          headers:
            Content-Type: application/json
          url: !lambda |-
            return "https://api.ipgeolocation.io/astronomy?apiKey=${ipg_api_key}&lat=${my_latitude}&long=${my_longitude}";
          on_response:
            - if:
                condition:
                  lambda: |-
                    return response->status_code == 200;
                then:
                  - logger.log: 
                      format: "moon-clock: Get Request succeeded..."
                      level: INFO
                  - lambda: |-
                      json::parse_json(body, [](JsonObject root) {
                          id(moon_altitude).publish_state(root["moon_altitude"]);
                          id(moon_azimuth).publish_state(root["moon_azimuth"]);
                          // use call.set_value rather than publish state for number template
                          auto call = id(moon_angle).make_call();
                          call.set_value(root["moon_angle"]);
                          call.perform();
                          id(moonrise).publish_state(root["moonrise"]);
                          id(moonset).publish_state(root["moonset"]);
                          id(moon_phase).publish_state(root["moon_phase"]);
                          return true;
                      });
                else:
                - logger.log: 
                    format: "moon-clock: Get Request - error!!!"
                    level: ERROR
      - if:
          condition:
            lambda: 'return id(moon_altitude).state > 0;'
          then:
            - logger.log: 
                format: "moon-clock: The moon is up"
                level: INFO
            - light.turn_on: yellow_led
          else:
            - logger.log: 
                format: "moon-clock: The moon is below the horizon"
                level: INFO
            - light.turn_off: yellow_led
      - logger.log: 
          format: "moon-clock: Get Request finished"
          level: INFO
```

Servo Number Template
---------------------

The number template drives the servo to the correct Phase Angle of the Moon.
The Phase Angle varies from 0 degrees (New Moon) through the waxing phases to 180 degrees (Full Moon), and through the waning phases to 360 degrees (New Moon again).
The 2X Cog generates the full 360-degree rotation from a 180-degree SG90 servo.
The formula translates the Moon Phase Angle from 0-360 to -1 to +1 for the servo.

### Example Definition of a number template to drive the Moon Phase servo

```
number:
  - platform: template
    name: Moon Angle
    id: moon_angle
    min_value: 0
    initial_value: 180
    max_value: 360
    step: 0.1
    optimistic: True
    set_action:
      then:
        - servo.write:
            id: servo_phase
            level: !lambda 'return ((x / 360.0)* 2) - 1.0;'
```


Display Code
------------

The display code shows the Moon position, rise and set times, and Moon Phase Angle.
Font Glyphs are used to display an icon with each item, and small Moon images are used to show the major phases.
The Interval automation cycles the dsiplay page every 5 seconds

### Example Definition of output and servo for Moon Phase Angle
```
display:
  - platform: ssd1306_i2c
    model: "SSD1306 128x64"
    reset_pin: GPIO25 # an unused pin
    address: 0x3C
    id: my_display
    flip_x: True
    flip_y: True
    pages:
      - id: page1
        lambda: |-
          it.strftime(64, 10, id(font3), TextAlign::TOP_CENTER, "%H:%M", id(sntp_time).now());
      - id: page2
        lambda: |-
          it.print(64, 0, id(font2), TextAlign::TOP_CENTER, "MOONRISE");
          it.print(75, 10, id(icons), "󱴗");
          it.printf(5, 20, id(font1), "%s", id(moonrise).state.c_str());
      - id: page3
        lambda: |-
          it.print(64, 0, id(font2), TextAlign::TOP_CENTER, "ALTITUDE");
          it.print(75, 14, id(icons), "󰤷");
          it.printf(25, 21, id(font1), "%.0f°", id(moon_altitude).state);
      - id: page4
        lambda: |-
          it.print(64, 0, id(font2), TextAlign::TOP_CENTER, "AZIMUTH");
          it.print(75, 14, id(icons), "󰆌");
          it.printf(20, 20, id(font1), "%.0f°", id(moon_azimuth).state);
      - id: page5
        lambda: |-
          it.print(64, 0, id(font2), TextAlign::TOP_CENTER, "MOONSET");
          it.print(75, 10, id(icons), "󱴕");
          it.printf(6, 21, id(font1), "%s", id(moonset).state.c_str());
      - id: page6
        lambda: |-
          // Print Moon Phase in top centre
          it.printf(64, 0, id(font2), TextAlign::TOP_CENTER, "%.12s", id(moon_phase).state.c_str());
          it.printf(10, 25, id(font1), "%.0f°", id(moon_angle).state);
          if (id(moon_phase).state == "NEW_MOON")  {
            it.image(70, 16, id(new_moon), COLOR_OFF, COLOR_ON);
          } else if (id(moon_phase).state == "WAXING_CRESCENT")  {
            it.image(70, 16, id(waxing_crescent), COLOR_OFF, COLOR_ON);
          } else if (id(moon_phase).state == "FIRST_QUARTER")  {
            it.image(70, 16, id(first_quarter), COLOR_OFF, COLOR_ON);
          } else if (id(moon_phase).state == "WAXING_GIBBOUS")  {
            it.image(70, 16, id(waxing_gibbous), COLOR_OFF, COLOR_ON);
          } else if ((id(moon_phase).state == "FULL_MOON")) {
            it.image(70, 16, id(full_moon), COLOR_OFF, COLOR_ON);
          } else if (id(moon_phase).state == "WANING_GIBBOUS")  {
            it.image(70, 16, id(waning_gibbous), COLOR_OFF, COLOR_ON);
          } else if (id(moon_phase).state == "LAST_QUART")  {
            it.image(70, 16, id(last_quarter), COLOR_OFF, COLOR_ON);
          } else if (id(moon_phase).state == "WANING_CRESCENT")  {
            it.image(70, 16, id(waning_crescent), COLOR_OFF, COLOR_ON);
          } else {
            it.printf(0, 40, id(font2), "%s", id(moon_phase).state.c_str());
          }

interval:
  # update the display page every 5 seconds
  - interval: 5s
    then:
      - display.page.show_next: my_display
      - component.update: my_display

```

Time Automation
---------------

```
time:
  - platform: sntp
    id: sntp_time
    timezone: ${my_timezone}

    on_time:
      # get Moon data every 10 minutes
      - seconds: 0
        minutes: /10
        then:
          - logger.log:
              format: "moon-clock: every 10 minutes..."
              level: INFO
          - script.execute: get_attributes
          - display.page.show: page1
```


Next Steps
----------

Printing the white Moon hemisphere as a Lithophane... and scaling up to a 6" globe.

