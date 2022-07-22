# Desky-Config
Config for Desky custom control panel 

```
####################################################################################################
##Notes/Info
####################################################################################################
## Home Assistant Thread: https://community.home-assistant.io/t/desky-standing-desk/383790/3
## ssieb who generously donated his time and knowledge to developing the custom component may offer some support on the Discord thread (please read both threads first).
## https://discord.com/channels/429907082951524364/952464939480645642

#Notes:
# 1. Use this component at own risk. It has not had extensive testing. May void warranty or cause desk faults etc.
# 2. This controller may bypass desk safety features like collision detection.
# 3. This component doesn't know about the values stored in your desk. Values in this config are (and must be) set independently.

#Troubleshooting Tips: 
#If you ever get a flashing ASr message, you might need to reset your desk.
#https://desky.com.au/blogs/news/reset-standing-desk-control-panel#:~:text=When%20ready%2C%20press%20and%20hold,Hooray!

#TODO
  # Test collision detection
  # Smooth stop

substitutions:
  devicename: desky-custom-control-panel
  friendly_name: Desky
  device_description: Desky Standing Desk Wifi Control Panel
############################################################################
#Pin Mappings - Found it easy to organise them up here during dev.
#Priority 1 Pins: https://www.youtube.com/watch?v=LY-1DHTxRAk&t=360s
############################################################################
#Desky
  desky_request_height_pin: GPIO32 #Request desk height | white wire
  desky_rx_pin: GPIO34 #Receive height messages from controller | brown wire
  desky_up_pin: GPIO05  #Move desk up | green wire
  desky_down_pin: GPIO23 #Move desk down | yellow wire
  
#Other
  keyboard_drawer_endstop_height_pin: GPIO03 #Desk Keyboard Drawer Endstop | Labelled RX. 
  i2c_screen2_scl_pin: GPIO18 #Screen 2
  i2c_screen2_sda_pin: GPIO19 #Screen 2
  i2c_bus1_sda_pin: GPIO21 #many devices share i2c_bus1 
  i2c_bus1_scl_pin: GPIO22 #many devices share i2c_bus1 
  led_red_pin: GPIO25 #Red left led
  led_green_pin: GPIO26 #Green right led
  button1_left_pin: GPIO33 #Left push button
  button2_right_pin: GPIO02 #Right push button
  rttl_buzzer_pin: GPIO27 #RTTL buzzer
  pir_pin: GPIO35 #Motion Sensor
  
 
esphome:
  name: $devicename
  comment: ${device_description}
  platform: ESP32
  board: ttgo-t7-v14-mini32   
  #LILYGO TTGO T7 V1.5
  #pinout: https://ae01.alicdn.com/kf/H32f84415f675426b84fdae1a7809eef2r/LILYGO-TTGO-T7-Mini32-V1-5-ESP32-WROVER-B-Dual-Core-PSRAM-Wireless-Wi-Fi-Bluetooth.jpg

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  manual_ip:
      static_ip: 192.168.1.63
      gateway: 192.168.1.1
      subnet: 255.255.255.0

api:
ota:

logger:
    baud_rate: 0
    # level: VERY_VERBOSE ##Uncomment if using UART debug

external_components:
#Fetch ssieb's custom component: https://github.com/ssieb/custom_components/tree/master/components/desky
  - source:
      type: git
      url: https://github.com/ssieb/custom_components
    components: [ desky ]

# esp32_ble_tracker:
# #For tracking smart band presence
  # scan_parameters:
    # interval: 320ms
    # window: 30ms

uart:
  - id: desk_uart
    baud_rate: 9600
    rx_pin: ${desky_rx_pin} #Brown Wire
    # #You can uncomment the debug section below to see UART messages.
    # debug:
      # direction: RX
      # dummy_receiver: true
      # after:
        # bytes: 4
      # sequence:     
        # - lambda: UARTDebug::log_int(direction, bytes, ',');


##########################################################################################
# Desky custom component
##########################################################################################  
desky:
  id: my_desky
  up:  # green wire
    number: ${desky_up_pin}
    inverted: true 
  down: #yellow wire
    number: ${desky_down_pin}
    inverted: true
  request:  #Request height update at boot so height is known after reboot | #White Wire
    number: ${desky_request_height_pin}
    inverted: true
  stopping_distance: 15  # optional offset distance from target to turn off moving in mm (default 15mm).
  timeout: 15s  # Optional time limit for moving, default is none (safety feature). 15sec is more than time to move from lowest point to highest point.
  height:  # Sensor publishing the current height. Make this internal and process another copy for sending to HA.
    name: Desky Height
    id: desky_height
    accuracy_decimals: 0
    internal: true
    filters:
    - delta: 0.1 #Only pass values if they change
    
text_sensor:
# Reports the ESPHome Version with compile date
  - platform: version
    name: ${friendly_name} ESPHome Version
    #Tracks last movement direction to help with the one-touch sit/stand toggle button.
  - platform: template
    id: desky_last_movement_direction 
    name: "Desky Last Movement Direction"
    icon: "mdi:arrow-up-down-bold"
    
    #Infers a sit/stand type category from height
  - platform: template
    id: desky_current_sit_stand_position
    name: "Desky Current Sit or Stand Position"
    icon: "mdi:seat-recline-extra"
    lambda: |-
      return {};
    on_value:
    #Reset stopwatch (tracks time in current state)
    - button.press: desky_start_stopwatch
    
    #Makes near/far type categories from the ToF sensor (to help with desk occupancy detection)
  - platform: template
    id: desky_tof_proximity_category
    name: "Desky Tof Proximity Category"
    icon: "mdi:arrow-expand-horizontal"
   
######################
#Buzzer
######################    
rtttl:
  output: rtttl_out
  on_finished_playback:
    - logger.log: 'Song ended!'

output:
#Rttl buzzer
  - platform: ledc
    pin: ${rttl_buzzer_pin} 
    id: rtttl_out
    
######################
#LEDs
######################  
  #Red
  - platform: ledc
    id: desky_red_led
    pin: ${led_red_pin} 
  #Green
  - platform: ledc
    id: desky_green_led
    pin: ${led_green_pin}

light:
  - platform: monochromatic
    name: "Desky Left Red LED"
    output: desky_red_led
    id: desky_light_led_left_red
    internal: true
  - platform: monochromatic
    name: "Desky Right Green LED"
    output: desky_green_led
    id: desky_light_led_right_green
    internal: true
  
##########################################################################################
# I2C
##########################################################################################
i2c:
#Main bus
  - id: i2c_bus1 #Screen 1, tof
    scan: true
    scl: ${i2c_bus1_scl_pin} #clock blue wire
    sda: ${i2c_bus1_sda_pin} #data green wire
      
#Had to use another bus for second screen as couldn't change address.
  - id: i2c_bus2 #Screen 2
    scan: true
    scl: ${i2c_screen2_scl_pin} #clock blue wire
    sda: ${i2c_screen2_sda_pin} #data green wire
   
binary_sensor:
#Someone is sitting? Sit dection using a multi-pronged approach.
  - platform: template
    id: someone_is_sitting
    name: Someone Is Sitting
    icon: "mdi:seat-recline-normal"   
    #Motion is detected, ToF is in range, keyboard drawer is open, desk height is in range
    lambda: |-
      return
      id(desky_motion_detected_processed).state &&
      id(desky_tof_seems_occupied).state &&
      id(desky_keyboard_drawer_endstop).state &&
      (id(desky_current_sit_stand_position).state == "Sitting")
      ;
    on_press:
    #Red light on if sitting is detected.
      then:
        - delay: 2s
        - light.turn_on:
            id: desky_light_led_left_red
            brightness: 20%      
    on_release:
    #When it turns off, check if "simple sit detection" is still on. Helps diagnose ToF/PIR false negatives.
      then:
        - light.turn_off:
            id: desky_light_led_left_red
        - delay: 1s
        - while:
          #If presence isn't detected (PIR/ToF) but height is in range and keyboard drawer is out, slow flash red led
            condition:
              and:
                - binary_sensor.is_on: desky_simple_sit_detection
                - binary_sensor.is_off: someone_is_sitting
            then:
              - light.turn_on:
                    id: desky_light_led_left_red
                    brightness: 100%
              - delay: 1s
              - light.turn_off:
                    id: desky_light_led_left_red
              - delay: 1s
          
#Someone is sitting? (Simpler method)
  - platform: template
    id: desky_simple_sit_detection
    name: Desky Simple Sit Detection
    icon: "mdi:seat-passenger"
    #On if height is in sitting range and keyboard drawer is out.
    lambda: |-
      return
      id(desky_keyboard_drawer_endstop).state &&
      (id(desky_current_sit_stand_position).state == "Sitting")
      ;
    on_press:
       - switch.turn_on: desky_auto_stand_mode    
       - delay: 2s
       - while:
       #If presence isn't detected and "simple sit" is, slow flash red led.
            condition:
              and:
                - binary_sensor.is_on: desky_simple_sit_detection
                - binary_sensor.is_off: someone_is_sitting
            then:
              - light.turn_on:
                    id: desky_light_led_left_red
                    brightness: 100%
              - delay: 1s
              - light.turn_off:
                    id: desky_light_led_left_red
              - delay: 1s
                
#Is Desky in motion? 
#Only works if activated by move_to function.
  - platform: template
    id: desky_is_moving
    name: Desky Is Moving
    lambda: return id(my_desky).current_operation != desky::DESKY_OPERATION_IDLE;
    icon: "mdi:motion"
    
#PIR Motion detection. Raw/unprocessed.     
  - platform: gpio
    id: desky_motion_detected
    pin: ${pir_pin} 
    name: "Desky PIR Sensor"
    device_class: motion
    icon: "mdi:motion-sensor"
    internal: true
        
#Is motion detected? Processed. Add timeout.
  - platform: copy
    source_id: desky_motion_detected
    id: desky_motion_detected_processed
    name: "Desky PIR Sensor Processed"
    icon: "mdi:motion-sensor"
    filters:
      - delayed_off: 120s #Largish timeout worked best.
      
  - platform: template
    id: desky_tof_seems_occupied
    name: Desky ToF Seems Occupied
    #On if ToF is in range
    lambda: return id(desky_tof_proximity_category).state == "At Desk";
    filters:
      - delayed_off: 60s #Timeout helps with false negatives.

  # - platform: ble_presence
    # mac_address: !secret mb_mi_band_mac_address 
    # id: desky_mb_mi_band_detected
    # name: "Desky MB Mi Band Detected"

         
###############################################################
#Keyboard Drawer (Reed switch  based endstop)
###############################################################
  - platform: gpio
    pin:
      number: ${keyboard_drawer_endstop_height_pin}
      mode: INPUT_PULLUP
    name: "Desky Keyboard Drawer Endstop"
    id: "desky_keyboard_drawer_endstop"
    internal: FALSE
    device_class: opening
    icon: "mdi:keyboard-close"
    filters:
      - delayed_on_off: 1s #Debounce
    on_state:
      #Make a noise
      - script.execute: short_beep_buzzer
      #Reset stopwatch
      - button.press: desky_start_stopwatch
    on_release:
    #Keyboard has been stowed under desk. Move desk to height so I can stow my chair too.
      then:
        - delay: 2s
        - button.press: desky_move_to_stow_height1
    on_press:
    #Keyboard has been pulled out for use. Move to sit height.
      then:
        - delay: 2s
        - button.press: desky_move_to_sit_height1

###############################################################
#Buttons
###############################################################
#Button 1 - Left hand side
#Main function: One press toggle sit/stand
  - platform: gpio
    pin:
      number: ${button1_left_pin}
      mode: INPUT_PULLUP
    name: "Desky Toggle Sit/Stand"
    id: "desky_button1_left_toggle_sit_stand"
    internal: true
    icon: "mdi:swap-vertical-bold"
    filters:
      - invert:
    on_press:
      then:
        #Make a noise.
        - script.execute: short_beep_buzzer
        #One touch sit/stand toggle
        - button.press: toggle_sit_stand

#Button 2 - Right hand side
  - platform: gpio
    pin:
      number: ${button2_right_pin}
      mode: INPUT_PULLUP
    name: "Desky Toggle Auto Up Mode"
    id: "desky_button2_right_toggle_auto_up_mode"
    internal: true
    icon: "mdi:radiobox-marked"
    filters:
      - invert:
    on_multi_click: #Long press to toggle auto stand mode
    - timing:
          - ON for at least 1s
      then:
        - switch.toggle: desky_auto_stand_mode
    on_press:
    #Make a noise. Just to know the leading edge is working ok.
    - script.execute: short_beep_buzzer
    #Extend countdown    
    - if:
        condition:
          lambda: |-
            return
            int(id(countdown_timer_remaining_time)) <= int(id(countdown_warning_threshold)) &&
            id(countdown_timer_remaining_time) >= 0
            ;

        then:
        - globals.set:
            id: countdown_timer_remaining_time
            value: !lambda return id(countdown_extention).state;
        # Show the remaining time.
        - sensor.template.publish:
            id: desky_minutes_until_stand_up
            state: !lambda return id(countdown_extention).state;
        - delay: 1s
        - light.turn_on:
            id: desky_light_led_right_green
            brightness: 20%

    
# ##############################################################
# Screens
# ##############################################################
# ##############
# Fonts
# ##############
font:
  - file: "fonts/Roboto-Bold.ttf"
    id: roboto10
    size: 10
  - file: "fonts/Roboto-Bold.ttf"
    id: roboto12
    size: 12
  - file: "fonts/Roboto-Bold.ttf"
    id: roboto16
    size: 16
  - file: "fonts/Roboto-Bold.ttf"
    id: roboto30
    size: 30
  - file: "fonts/Roboto-Bold.ttf"
    id: roboto40
    size: 40
 
#Icons: https://community.home-assistant.io/t/display-materialdesign-icons-on-esphome-attached-to-screen/199790/13
 
  - file: "fonts/materialdesignicons-webfont.ttf"
    id: matdes11
    size: 11
    glyphs:
      - "\U000F075F" # mdi-volume-mute
      - "\U000F057E" # mdi-volume-high
      - "\U000F0D91" #mdi-motion-sensor
      - "\U000F1435" #mdi-motion-sensor-off
      - "\U000F0E64" #mdi-signal-distance-variant (ToF)
      - "\U000F030F" #mdi-keyboard-close
      - "\U000F1249" #mdi-seat-passenger
      - "\U000F02E6" #mdi-human 
      - "\U000F0792" #mdi-arrow-collapse-down
      - "\U000F05E0" #mdi-check-circle
      - "\U000F073A" #mdi-cancel
    
  - file: "fonts/materialdesignicons-webfont.ttf"
    id: matdes30
    size: 30
    glyphs:
         
      - "\U000F075F" # mdi-volume-mute
      - "\U000F057E" # mdi-volume-high
      - "\U000F0D91" #mdi-motion-sensor
      - "\U000F1435" #mdi-motion-sensor-off
      - "\U000F0E64" #mdi-signal-distance-variant (ToF)
      - "\U000F030F" #mdi-keyboard-close
      - "\U000F1249" #mdi-seat-passenger
      - "\U000F02E6" #mdi-human 
      - "\U000F0792" #mdi-arrow-collapse-down
      - "\U000F05E0" #mdi-check-circle
      - "\U000F073A" #mdi-cancel
      - "\U000F04B9" #mdi-sofa
      - "\U000F154B" #mdi-sort-clock-descending
      - "\U000F0513" #mdi-thumb-up
    
  - file: "fonts/materialdesignicons-webfont.ttf"
    id: matdes40
    size: 40
    glyphs:
      - "\U000F04B9" #mdi-sofa
      - "\U000F154B" #mdi-sort-clock-descending
      - "\U000F0513" #mdi-thumb-up
      
########################################################
#Displays
########################################################
#SSD1306 128x64 Display | 3.3V | https://esphome.io/components/display/ssd1306.html
display:

# Screen 1 - Left hand side     
  - platform: ssd1306_i2c
    model: "SSD1306 128x64"
    id: desky_display_1_left
    i2c_id: i2c_bus2
    update_interval: 5s
    address: 0x3C
    pages:
      - id: desky_display_1_left_page1
        lambda: |-
          static int num_icons = 6;
          static int left_offset_1 = -1;
          static int top_offset_1 = 1;
          static int vertical_padding = 0.8;
          static int left_offset_2 = left_offset_1 + 13;
          
          //Sitting detection helpers: text describers
          it.printf(left_offset_2, 1*(it.get_height() / num_icons)-3, id(roboto12), TextAlign::CENTER_LEFT , "Sitting");
          it.printf(left_offset_2, 2*(it.get_height() / num_icons) + top_offset_1 + (1 * vertical_padding), id(roboto10), TextAlign::CENTER_LEFT , "PIR");
          it.printf(left_offset_2, 3*(it.get_height() / num_icons) + top_offset_1 + (2 * vertical_padding), id(roboto10), TextAlign::CENTER_LEFT , "ToF");
          it.printf(left_offset_2, 4*(it.get_height() / num_icons) + top_offset_1 + (3 * vertical_padding), id(roboto10), TextAlign::CENTER_LEFT , "Height");
          it.printf(left_offset_2, 5*(it.get_height() / num_icons) + top_offset_1 + (4 * vertical_padding), id(roboto10), TextAlign::CENTER_LEFT , "Keyboard");
          it.printf(left_offset_2, 6*(it.get_height() / num_icons) + top_offset_1 + (5 * vertical_padding), id(roboto10), TextAlign::CENTER_LEFT , "BandXX");
          
          //Sitting detection helpers: Icons (idicate on/off)
          
          //Overall sitting detection     
          if (id(someone_is_sitting).state) {
            it.print(left_offset_1, 1*(it.get_height() / num_icons)-3, id(matdes11), TextAlign::CENTER_LEFT, "\U000F05E0");
          } else {
            it.print(left_offset_1, 1*(it.get_height() / num_icons)-3, id(matdes11), TextAlign::CENTER_LEFT, "\U000F073A");
          }
          
          //PIR
          if (id(desky_motion_detected_processed).state) {
            it.print(left_offset_1, 2*(it.get_height() / num_icons) + top_offset_1 + (1 * vertical_padding), id(matdes11), TextAlign::CENTER_LEFT, "\U000F05E0");
          } else {
            it.print(left_offset_1, 2*(it.get_height() / num_icons) + top_offset_1 + (1 * vertical_padding), id(matdes11), TextAlign::CENTER_LEFT, "\U000F073A");
          }
          
          //Occupancy from ToF
          if (id(desky_tof_seems_occupied).state) {
            it.print(left_offset_1, 3*(it.get_height() / num_icons) + top_offset_1 + (2 * vertical_padding), id(matdes11), TextAlign::CENTER_LEFT, "\U000F05E0");
          } else {
            it.print(left_offset_1, 3*(it.get_height() / num_icons) + top_offset_1 + (2 * vertical_padding), id(matdes11), TextAlign::CENTER_LEFT, "\U000F073A");
          }
          
          //Sit or stand from height
          if (id(desky_current_sit_stand_position).state == "Sitting") {
            it.print(left_offset_1, 4*(it.get_height() / num_icons) + top_offset_1 + (3 * vertical_padding), id(matdes11), TextAlign::CENTER_LEFT, "\U000F05E0");
          } else {
            it.print(left_offset_1, 4*(it.get_height() / num_icons) + top_offset_1 + (3 * vertical_padding), id(matdes11), TextAlign::CENTER_LEFT, "\U000F073A");
          }
          
          //Keyboard
          if (id(desky_keyboard_drawer_endstop).state) {
            it.print(left_offset_1, 5*(it.get_height() / num_icons) + top_offset_1 + (4 * vertical_padding), id(matdes11), TextAlign::CENTER_LEFT, "\U000F05E0");
          } else {
            it.print(left_offset_1, 5*(it.get_height() / num_icons) + top_offset_1 + (4 * vertical_padding), id(matdes11), TextAlign::CENTER_LEFT, "\U000F073A");
          }
          
          //Show sit/stand categories and time in current state
           it.printf(it.get_width()*0.75, 6, id(roboto16), TextAlign::CENTER , id(desky_current_sit_stand_position).state.c_str());
           it.printf(it.get_width()*0.75,33, id(roboto40), TextAlign::CENTER , "%.0f", id(desky_stopwatch).state);
           it.printf(it.get_width()*0.75, 57, id(roboto12), TextAlign::CENTER , "Minutes");          
          
  # //BLE Mi Band
  # if (id(desky_keyboard_drawer_endstop).state) {
    # it.print(left_offset_1, 6*(it.get_height() / num_icons) + top_offset_1 + (5 * vertical_padding), id(matdes11), TextAlign::CENTER_LEFT, "\U000F05E0");
  # } else {
    # it.print(left_offset_1, 6*(it.get_height() / num_icons) + top_offset_1 + (5 * vertical_padding), id(matdes11), TextAlign::CENTER_LEFT, "\U000F073A");
  # }
          

                
          
# Screen 1 - Right hand side
  - platform: ssd1306_i2c
    model: "SSD1306 128x64"
    id: desky_display_2_right
    i2c_id: i2c_bus1
    update_interval: 5s
    address: 0x3C
    pages:
      - id: desky_display_2_right_page1
        lambda: |-
          //Show either countdown to standing (if sitting) or "lazy mode"
          if (id(desky_auto_stand_mode).state) 
            {
              if (id(desky_simple_sit_detection).state)
                {
                  it.printf(it.get_width()/2, -2, id(roboto16), TextAlign::TOP_CENTER , "Auto Stand In:"); 
                  it.printf(it.get_width()*0.75, 35, id(roboto40), TextAlign::CENTER , "%.0f", id(desky_minutes_until_stand_up).state); 
                  it.printf(it.get_width()*0.75, 57, id(roboto12), TextAlign::CENTER , "Minutes");
                  it.print((it.get_width()*0.25)+15, it.get_height()/2+10, id(matdes40), TextAlign::CENTER, "\U000F154B");
                }
              else
                {
                  it.printf(it.get_width()/2, -2, id(roboto16), TextAlign::TOP_CENTER , "Not On Arse"); 
                  it.print((it.get_width()/2), it.get_height()/2+10, id(matdes40), TextAlign::CENTER, "\U000F0513");
                }
              
            }
          else 
            {
              it.printf(it.get_width()/2, -2, id(roboto16), TextAlign::TOP_CENTER , "Auto Stand: OFF");
              it.print(it.get_width()/2, (it.get_height()/2)-2, id(matdes40), TextAlign::CENTER, "\U000F04B9");         
              it.print(it.get_width()/2, 55, id(roboto16), TextAlign::CENTER , "Lazy Mode !!");
            }

sensor:
#############################################
#Time of Flight  / Distance
#############################################
#VCC connects to 3V3 | https://esphome.io/components/sensor/vl53l0x.html
  - platform: vl53l0x
    i2c_id: i2c_bus1
    id: desky_tof_distance
    icon: mdi:arrow-expand-horizontal
    name: "Desky ToF Distance"
    address: 0x29
    update_interval: 1s
    long_range: true #Seemed to be required with my distances.
    filters:
      - multiply: 100 #Convert to cm
      - median: #Use a moving median for spike smoothing.
          window_size: 5
          send_every: 5
    unit_of_measurement: "cm"
    accuracy_decimals: 0 #Keep it simple. To the cm.
    on_value:
      then:
        - lambda: |-
            if ((id(desky_tof_distance).state) < 20) { return id(desky_tof_proximity_category).publish_state("Very Close");}
            else if ((id(desky_tof_distance).state) < 50) { return id(desky_tof_proximity_category).publish_state("Something Quite Close");}
            else if ((id(desky_tof_distance).state) < 77) { return id(desky_tof_proximity_category).publish_state("At Desk");}
            else if ((id(desky_tof_distance).state) >= 77) { return id(desky_tof_proximity_category).publish_state("Something Quite far");}
            else { return id(desky_tof_proximity_category).publish_state("Unknown");}
 
##############################
#Desk Height post processing
##############################
  - platform: copy
    source_id: desky_height
    id: desky_height_cm
    name: "Desky Desk Height (cm)"
    accuracy_decimals: 1
    internal: false
    icon: mdi:arrow-expand-vertical
    unit_of_measurement: cm
    filters:
    - delta: 0.1 #Only send values to HA if they change    
    - throttle: 250ms #Limit values sent to Ha to 4 per sec.    
    - multiply: 0.1 #convert from mm to cm
    on_value:
      then:
        #Track the last movement direction for use later
        - lambda: |-
            if (id(my_desky).current_operation == desky::DESKY_OPERATION_RAISING){ return id(desky_last_movement_direction).publish_state("Up");}
            else if (id(my_desky).current_operation == desky::DESKY_OPERATION_LOWERING){return id(desky_last_movement_direction).publish_state("Down"); }
        #Track a sit/stand categorisation of desk height for use later
        - lambda: |-
            if ((id(desky_height_cm).state) < (id(desky_sit_height_1).state + 5.0)) { return id(desky_current_sit_stand_position).publish_state("Sitting");}
            else if ((id(desky_height_cm).state) > (id(desky_stand_height_1).state - 5.0)) { return id(desky_current_sit_stand_position).publish_state("Standing");}
            else if ((id(desky_height_cm).state) < (id(desky_stow_height_1).state + 5.0)) { return id(desky_current_sit_stand_position).publish_state("Stowed");}
            else { return id(desky_current_sit_stand_position).publish_state("Other");}

 
################################################
# Countdown Timer
################################################
 # Countdown timer until you have to get off yo butt. 
  - platform: template
    name: Desky Minutes Until Stand Up
    id: desky_minutes_until_stand_up
    accuracy_decimals: 0
    unit_of_measurement: min
    icon: mdi:timer
    update_interval: 1s
    lambda: return id(countdown_timer_remaining_time);
    filters:
    - delta: 0.1
    on_value_range:
    #Warning if desky is approaching auto movement.
      - above: 0
        below: !lambda return id(countdown_warning_threshold).state;
        then:
          #Audible first early audible warning    
          - rtttl.play:
              rtttl: 'moving_soon:d=32,o=5,b=100:c,c#,d#,e,f#,g#,a#,b'    
          - while:
              condition:
                lambda: |-
                  return
                   id(desky_minutes_until_stand_up).state > 0 &&
                   id(desky_minutes_until_stand_up).state <= id(countdown_warning_threshold).state
                  ;
              then:
                #Quick flash green light if countdown < 10sec  
                - light.turn_on:
                    id: desky_light_led_right_green
                    brightness: 100%
                - delay: 500ms
                - light.turn_off: desky_light_led_right_green
                - delay: 500ms      
    on_value:
    #When countdown hits 0, raise desk and do other stuff.
      then:
        - if:
            condition:
              - lambda: return id(countdown_timer_remaining_time) == 0;
            then:
              - if:
                  condition: #Check if desk is in simple sitting state 
                    - lambda: return (id(desky_simple_sit_detection).state);
                  then:
                    - if:
                        condition: #Check if someone is in front of desk.
                          - lambda: return (id(someone_is_sitting).state);
                        then:
                          - switch.turn_off: desky_sitting_countdown_on
                          - light.turn_on:
                              id: desky_light_led_right_green
                              brightness: 20%
                          #Audible last warning before movement    
                          - rtttl.play:
                              rtttl: 'moving_now:d=32,o=5,b=20:c,c,g' 
                          - delay: 2s
                          - button.press: desky_move_to_stand_height1  
                        else:
                        #Extend time and complain...
                          - globals.set:
                              id: countdown_timer_remaining_time
                              value: !lambda return id(countdown_extention).state;
                          # Show the remaining time.
                          - sensor.template.publish:
                              id: desky_minutes_until_stand_up
                              state: !lambda return id(countdown_extention).state;
                          - delay: 1s
                          - light.turn_on:
                              id: desky_light_led_right_green
                              brightness: 20%
                          - script.execute: multi_beep_buzzer

 
################################################
# Desk State stopwatch. Counts up time in current state.
################################################
  - platform: template
    name: Desky Stopwatch
    id: desky_stopwatch
    accuracy_decimals: 0
    unit_of_measurement: minutes
    icon: mdi:timer
    update_interval: 1s
    lambda: return id(desky_stopwatch_time);
    filters:
    - delta: 0.1


button:
#Single button sit/stand toggle 
  - platform: template
    name: Desky Toggle Sit Stand  
    id: toggle_sit_stand
    icon: mdi:swap-vertical-bold
    on_press: 
    #If moving: Stop
    #If not moving: Do opposite movement as last one.
      then: 
        - lambda: |-
            if (id(my_desky).current_operation != desky::DESKY_OPERATION_IDLE) {id(desky_stop_desk).press();}
            else if (id(desky_last_movement_direction).state == "Up") {return id(desky_move_to_sit_height1).press();}
            else if (id(desky_last_movement_direction).state == "Down") {return id(desky_move_to_stand_height1).press();}
            else if (id(desky_current_sit_stand_position).state == "Standing") {return id(desky_move_to_sit_height1).press();}
            else {return id(desky_move_to_stand_height1).press();}

#Timer: Start Sitting Countdown Timer
  - platform: template
    name: Start Sitting Countdown Timer
    id: desky_intialise_sitting_countdown_timer
    icon: mdi:timer-sand-full
    on_press: 
      then:
        # Start the countdown timer.
        - globals.set:
            id: countdown_timer_remaining_time
            value: !lambda return id(desky_max_sitting_time).state;
        # Show the remaining time.
        - sensor.template.publish:
            id: desky_minutes_until_stand_up
            state: !lambda return id(desky_max_sitting_time).state;

#Start Stopwatch
  - platform: template
    name: Start stop watch
    id: desky_start_stopwatch
    icon: mdi:timer-sand-full
    on_press: 
      then:
        # Start the countdown timer.
        - globals.set:
            id: desky_stopwatch_time
            value: !lambda return 0;
        # Show the remaining time.
        - sensor.template.publish:
            id: desky_stopwatch
            state: !lambda return 0;


#Stop Movement
  - platform: template
    name: Stop Desk
    id: desky_stop_desk
    icon: mdi:pause-octagon
    on_press:
      then:
        - lambda: id(my_desky).stop();
        - delay: 200ms
          #Do it twice for good luck. Might not be needed.
        - lambda: id(my_desky).stop();

# #Move to function
  - platform: template
    name: Move Desky to Target Height
    id: desky_move_to_desky_target_height
    icon: mdi:target
    on_press:
      then:
        - lambda: id(my_desky).move_to(int((id(desky_target_height).state) * 10));  

#Define as many single press presets like this as you like  
  - platform: template
    name: Move Desky to Sit Height 1
    id: desky_move_to_sit_height1
    icon: mdi:seat-passenger
    on_press:
      then:
        - lambda: id(my_desky).move_to(int((id(desky_sit_height_1).state) * 10));  
             
  - platform: template
    name: Move Desky to Stand Height 1
    id: desky_move_to_stand_height1
    icon: mdi:karate
    on_press:
      then:
        - lambda: id(my_desky).move_to(int((id(desky_stand_height_1).state) * 10));  
        
  - platform: template
    name: Move Desky to Stow Height 1
    id: desky_move_to_stow_height1
    icon: mdi:table-chair
    on_press:
      then:
        - lambda: id(my_desky).move_to(int((id(desky_stow_height_1).state) * 10));  
        

number:
#########################################################
#Height Presets
#########################################################

#Target Height ("Move desk to height x").
#You should probably limit the range you can move the desk to to within the limits you've set via the control panel, perhaps offset a little within the range.
#Sending commands higher/lower than this may cause error messages and require desk reset (or maybe worse).
#Target height which can be changed in HA.
  - platform: template
    id: desky_target_height
    icon: mdi:target-variant
    name: "Desky Target Height"
    optimistic: true
    unit_of_measurement: cm
    min_value: 80
    max_value: 130.0
    step: 0.1
    restore_value: true
    initial_value: 85.3

#Various height presets (independant of what you've set via concole)
  - platform: template
    id: desky_stand_height_1
    name: "Desky Stand Height 1"
    icon: mdi:human-male-height
    optimistic: true
    unit_of_measurement: cm
    min_value: 100.0
    max_value: 130.0
    step: 1.0
    restore_value: true
    initial_value: 124.0
    
  - platform: template
    id: desky_sit_height_1
    icon: mdi:seat
    name: "Desky Sit Height 1"
    optimistic: true
    unit_of_measurement: cm
    min_value: 83.0
    max_value: 90.0
    step: 1.0
    restore_value: true
    initial_value: 85.3
    
  - platform: template
    id: desky_stow_height_1
    name: "Desky Stow Height 1"
    icon: mdi:seat-outline
    optimistic: true
    unit_of_measurement: cm
    min_value: 96.0
    max_value: 98.0
    step: 1.0
    restore_value: true
    initial_value: 97.0    

#########################################################
#Sit / stand schedule/timer helpers.
#########################################################    
 
#How long you're allowed to sit for before you'll be "pushed-up" to standing 
  - platform: template
    id: desky_max_sitting_time
    name: "Desky Max Sitting Time"
    icon: mdi:timer-alert
    optimistic: true
    unit_of_measurement: min
    min_value: 5
    max_value: 60
    step: 1 
    restore_value: true
    initial_value: 60
    set_action:
      then:
        # Start the countdown timer.
        - globals.set:
            id: countdown_timer_remaining_time
            value: !lambda return x;
        # Show the remaining time.
        - sensor.template.publish:
            id: desky_minutes_until_stand_up
            state: !lambda return x;
    # # lambda: |-
      # # if 
        # # (id(desky_max_sitting_time).state == id(desky_max_sitting_time).state)
        # # {return {};}
      # # else
         # # {return id(desky_max_sitting_time).to_max();}
    # set_action:
      # then:
        # # Start the countdown timer.
        # - globals.set:
            # id: countdown_timer_remaining_time
            # value: !lambda return id(desky_max_sitting_time).state;
        # # Show the remaining time.
        # - sensor.template.publish:
            # id: desky_minutes_until_stand_up
            # state: !lambda return id(desky_max_sitting_time).state;

    
#When should Desky give you an early heads up it will move soon? 
  - platform: template
    id: countdown_warning_threshold
    icon: mdi:timer-alert
    name: "Countdown warning threshold"
    unit_of_measurement: min
    optimistic: true
    min_value: 2
    max_value: 10
    step: 1
    restore_value: true
    initial_value: 2
    
#Extend the countdown (like for if you're in a meeting etc)
  - platform: template
    id: countdown_extention
    icon: mdi:clock-plus-outline
    name: "Countdown Extention"
    optimistic: true
    unit_of_measurement: min
    min_value: 5
    max_value: 60
    step: 1
    restore_value: true
    initial_value: 3
   
switch:
  - platform: restart
    name: "Restart Desky"
    
#########################################################
#Desky controls
#########################################################
#Wake up the desk and request it sends its height
#The custom component should handle this on boot but this is a manual option. Sometimes it doesn't seem to work on boot. Don't know why yet.
  - platform: gpio
    id: wake_desk_and_get_height
    name: "Request Desk Height"
    pin:
      number: ${desky_request_height_pin}
      inverted: true
    on_turn_on:
    - delay: 100ms
    - switch.turn_off: wake_desk_and_get_height

#Raise the desk | Green Wire
  - platform: gpio
    id: desky_raise_desk
    name: "Desky Raise Desk"
    pin:
      number: ${desky_up_pin}
      inverted: true
    interlock: desky_lower_desk
    on_turn_on:
    #Auto off after 15s just in case
    - delay: 15s
    - switch.turn_off: desky_raise_desk
    
#Lower the desk | Yellow wire
  - platform: gpio
    id: desky_lower_desk
    name: "Desky Lower Desk" 
    pin:
      number: ${desky_down_pin}
      # mode: INPUT_PULLUP
      inverted: true
    interlock: desky_raise_desk
    on_turn_on:
   #Auto off after 15s just in case
    - delay: 15s
    - switch.turn_off: desky_lower_desk

#########################################################
#Push-Up Mode! 
#########################################################
#Switch to turn on/off "Auto Stand Mode"
#If on, desk moves to standing after x minutes of sitting. 
  - platform: template
    name: Desk Auto Stand Mode
    id: desky_auto_stand_mode
    optimistic: true
    restore_state: true
    turn_on_action:
      - light.turn_on:
            id: desky_light_led_right_green
            brightness: 10%
      #Make a noise
      - script.execute: rtttl_play_scale_up
      #Start the timer
      - lambda: |-
          if (id(desky_current_sit_stand_position).state == "Sitting") {
            id(desky_sitting_countdown_on).turn_on();
          }
          
    turn_off_action:
      - light.turn_off: desky_light_led_right_green
      - script.execute: rtttl_play_scale_down
      - switch.turn_off: desky_sitting_countdown_on

  - platform: template
    name: Desky Sitting Countdown On
    id: desky_sitting_countdown_on
    optimistic: true
    internal: FALSE
    turn_on_action:
        - button.press: desky_intialise_sitting_countdown_timer  
    turn_off_action:
    #Set countdown to -1
        - globals.set:
            id: countdown_timer_remaining_time
            value: '-1'
        # Show the remaining time.
        - sensor.template.publish:
            id: desky_minutes_until_stand_up
            state: -1
        
script:
#leds

  - id: short_flash_left_red_led
    then:
      - light.turn_on: desky_light_led_left_red
      - delay: 500ms
      - light.turn_off: desky_light_led_left_red
      
  - id: short_flash_right_green_led 
    then:
      - light.turn_on:
            id: desky_light_led_right_green
            brightness: 20%
      - delay: 500ms
      - light.turn_off: desky_light_led_right_green
      
  - id: short_double_flash_right_green_led 
    then:
      - light.turn_on:
            id: desky_light_led_right_green
            brightness: 15%
      - delay: 500ms
      - light.turn_off: desky_light_led_right_green
      - delay: 500ms
      - light.turn_off: desky_light_led_right_green
      
  - id: short_double_flash_left_red_led
    then:
      - light.turn_on: desky_light_led_left_red
      - delay: 200ms
      - light.turn_off: desky_light_led_left_red
      - delay: 200ms
      - light.turn_off: desky_light_led_left_red
      
  - id: long_flash_desky_light_led_left_red
    then:
      - light.turn_on:
            id: desky_light_led_left_red
            brightness: 10%
      - delay: 1s
      - light.turn_off:
            id: desky_light_led_left_red
      - delay: 1s
      
#Sounds 
  - id: rtttl_play_scale_up
    then:
    - rtttl.play:
        rtttl: 'scale_up:d=32,o=5,b=100:c,c#,d#,e,f#,g#,a#,b'
        
  - id: rtttl_play_scale_down
    then:
    - rtttl.play:
        rtttl: 'scale_down:d=32,o=5,b=100:b,a#,g#,f#,e,d#,c#,c'

#buzzer
  - id: short_beep_buzzer
    then:
    - rtttl.play:
        rtttl: 'beep:d=4,o=7,b=100:32a'   

  - id: multi_beep_buzzer
    then:
    - rtttl.play:
        rtttl: 'beep:d=4,o=7,b=100:32a,g#,d#'          
        
 
#TODO: Nag alert if you seem to be away from desk for a while but haven't stowed everything.
  # - id: desky_drawer_is_open_nag
    # then:
      # #Audible first warning    
      # - rtttl.play:
          # rtttl: 'twoshort:d=4,o=5,b=100:16e6,16e6...,a'    
      # - while:
          # condition:
            # - script.is_running: desky_is_about_to_raise
          # then:
            # - light.turn_on: desky_light_led_right_green
            # - delay: 500ms
            # - light.turn_off: desky_light_led_right_green
            # - delay: 500ms
      # - delay: 10s
      # #Audible last warning        
      # - rtttl.play:
          # rtttl: 'moving_now:d=32,o=5,b=20:c,c,g'   
      # - delay: 1s
   
 
globals:
#Helpers for the timers

#Countdown Timer helpers
  - id: countdown_timer_remaining_time
    type: int
    restore_value: yes
    initial_value: "-1"

  - id: countdown_timer_remaining_time_previous
    type: int
    restore_value: yes
    initial_value: "0"
 
 #Stopwatch helpers (Time in current state)
  - id: desky_stopwatch_time
    type: int
    initial_value: '0'
    restore_value: yes
    
  - id: desky_stopwatch_time_previous
    type: int
    initial_value: '0'
    restore_value: yes

      
interval:
#Drives the timers    
#Timers adopted from: https://brianhanifin.com/posts/diy-irrigation-controller-esphome-home-assistant/
  - interval: 1min #Update on the minute
    then:    
      - lambda: |-
          if (id(countdown_timer_remaining_time) > -1) {
             // Store the previous time.
             id(countdown_timer_remaining_time_previous) = id(countdown_timer_remaining_time);
             
            // When the relay is on.
            if (id(desky_current_sit_stand_position).state == "Sitting") {
              // Decrement the timer.
              id(countdown_timer_remaining_time) -= 1;
              
              // Do something when time is up
              if (id(countdown_timer_remaining_time) <= 0) {
                id(short_beep_buzzer).execute();
                id(countdown_timer_remaining_time) = 0;
                id(desky_display_2_right).update();
              }
            }
            
            // Update the remaining time display.
            if (id(countdown_timer_remaining_time_previous) != id(countdown_timer_remaining_time)) {
              id(desky_minutes_until_stand_up).publish_state( (id(countdown_timer_remaining_time)) );
              id(desky_display_2_right).update();
            }
          }
          
#Countup timer
  - interval: 1min #Update on the minute
    then:    
      - lambda: |-
          if (id(desky_stopwatch_time) > -1)
            {
             // Store the previous time.
             id(desky_stopwatch_time_previous) = id(desky_stopwatch_time);
             id(desky_stopwatch_time) += 1;
             
             // Update the time display.
             if (id(desky_stopwatch_time) != -1)
               {
                id(desky_stopwatch).publish_state( (id(desky_stopwatch_time)) );
                id(desky_display_1_left).update();
               }
            }





