# ESP32 based Smart Clock
Because I was lazy to set time each time on old clock after power outages (and natural slow deviations).  
I decided to build "Smarter" clock that would sync time from internet automatically (NTP). With a posibility to add something more on it in future.
![Old clock](/Photos/OldClock.jpg "Old Clock")

As I run HomeAssitant I wanted something compatible with it and not too much complicated.

- [ESP32 based Smart Clock](#esp32-based-smart-clock)
  - [Hardware](#hardware)
  - [ESPhome](#esphome)
    - [Bus, GPIOs, Pins](#bus-gpios-pins)
    - [Switches](#switches)
    - [Screen](#screen)
    - [Time](#time)
    - [TouchScreen](#touchscreen)
      - [Images, Colors, Fonts](#images-colors-fonts)
    - [LVGL](#lvgl)
    - [Scripts](#scripts)
  - [End Result](#end-result)
    - [Home Assistant](#home-assistant)
    - [Photos](#photos)
  - [3D File](#3d-file)
  - [ToDo](#todo)


## Hardware

After some research I found:
[CrowPanel ESP32 3.5" Model: DIS05035H](https://www.elecrow.com/esp32-display-3-5-inch-hmi-display-spi-tft-lcd-touch-screen.html)
![CrowPanel](/Photos/CrowPanel35.png "CrowPanel")

It is ESP32 based and compatible with ESPhome. 3.5" was choosen as it would fit into same place as Old Clock was. Bigger would not fit.

I Ordered it in AliExpress

## ESPhome

 [Original full yaml file here](hnr-clock.yaml)


Initialization of values and boot (because I can :-)) screen:
 ```YAML
 esphome:
  name: hnr-clock
  friendly_name: hnr-clock
  on_boot:
    - delay: 4s
    - lvgl.widget.hide: boot_screen # hide boot screen
    - lvgl.page.show:
        id: clock_page
    - script.execute: script_update_seconds # make sure  labels are updated on boot
    - script.execute: script_update_time # make sure  labels are updated on boot
    - script.execute: script_update_date # make sure  labels are updated on boot
 ```

### Bus, GPIOs, Pins

```YAML
i2c:
  - sda: 22
    scl: 21

spi:
  - id: tft
    clk_pin: 14
    mosi_pin: 13
    miso_pin: 33 #12 on v2.0
    interface: hardware

output:
  - id: gpio_backlight_pwm
    platform: ledc
    pin: 27
    inverted: False

light:
  - id: backlight
    name: Backlight
    platform: monochromatic
    output: gpio_backlight_pwm
    restore_mode: ALWAYS_ON
```

Additional Pins on manufacturer documentation (not used in my implementation):
| Used for | Pins |
| -------- | ----- |
| SPK | IO26 |
| UART |RX(IO3);TX(IO1)|
| SD Card Slot(SPI) | MOSI(IO23); MISO(IO19); CLK(IO18); CS(IO5) |

### Switches

- The board has GPIOD connector which I just defined as switches (for possible future use)
- I created 2 templated switches
  - Antiburn - used to enable screen antiburn function
  - Nigth mode - used to enable different style and backlight on screen


```YAML
switch:
  - platform: gpio # for possible future use 
    pin: GPIO25
    id: GPIO25
    # name: ""

  - platform: gpio # for possible future use 
    pin: GPIO32
    id: GPIO32
    # name: ""

  - platform: template
    name: Antiburn
    id: switch_antiburn
    icon: mdi:television-shimmer
    optimistic: true
    entity_category: "config"
    turn_on_action:
      - logger.log: "Starting Antiburn"
      - lvgl.pause:
          show_snow: true
      - light.turn_off: backlight 
    turn_off_action:
      - logger.log: "Stopping Antiburn"
      - if:
          condition: lvgl.is_paused
          then:
            - lvgl.resume:
            - lvgl.widget.redraw:
            - light.turn_on: backlight

  - platform: template
    name: "Night mode"
    id: switch_night
    icon: mdi:lightbulb-night-outline
    optimistic: true
    entity_category: "config"
    turn_on_action:
      - lambda: |-
          //lv_obj_remove_style_all(id(clock_label));
          lv_obj_add_style(id(clock_label), id(time_style_night), LV_PART_MAIN);
      - light.turn_on: 
          id: backlight
          brightness: 0.5
          transition_length: 3s
    turn_off_action:
      - lambda: |-
          //lv_obj_remove_style_all(id(clock_label));
          lv_obj_add_style(id(clock_label), id(time_style), LV_PART_MAIN);
      - light.turn_on: 
          id: backlight
          brightness: 1
          transition_length: 3s
```

### Screen

I used transformation to position screen correctly which is recomended way to do it as it is hardware based (`rotation` option is not hw based). 

```YAML
display:
  - id: main_display
    platform: ili9xxx
    model: ILI9488_A
    dimensions:
      height: 320
      width: 480
    transform:
      mirror_y: true
      mirror_x: true
      swap_xy: true
    spi_id: tft
    cs_pin:
      number: 15
      ignore_strapping_warning: true
    dc_pin:
      number: 2
      ignore_strapping_warning: true
    invert_colors: false
    data_rate: 20MHz
    update_interval: never
    auto_clear_enabled: false
```

### Time

- Time element used NTP as a source and each second updates LVGL elements.  
- For displaying day of the week (in words) I had to use lambda to calculate visualization of it.
- Actions are done based on Second, Minute and Hour change
  - on each second it updates `second` label
  - on each minute it updates `Hours and Minutes` label
  - on each hour it updates `date` label
  
There can be more efficient logic done for updating these time labels, but I thought maybe it is sufficient to leave like this  

```YAML
time:
  - platform: sntp
    id: time_comp
    timezone: "Europe/Vilnius"
    on_time:
      - seconds: /1
        then:
          - script.execute: script_update_seconds
      - seconds: 0
        minutes: /1
        then:
          - script.execute: script_update_time
      - seconds: 0
        minutes: 0
        hours: /1
        then:
          - script.execute: script_update_date
```

### TouchScreen

I implemented stop antiburn action when screen gets a touch action (just in case I need to see time when it is in `antiburn` mode).

```YAML
touchscreen:
  - id: main_touchscreen
    platform: xpt2046
    cs_pin:
      number: 12 #33 on v2.0
      ignore_strapping_warning: true
    interrupt_pin: 36
    update_interval: 50ms
    threshold: 400
    calibration:
      x_min: 280
      x_max: 3860
      y_min: 340
      y_max: 3860
    transform:
      mirror_y: false
      swap_xy: true
    on_touch:
      - switch.turn_off: switch_antiburn
```

#### Images, Colors, Fonts

My defined style elements:
* Images - used as boot image, or any other image I will use in future
* Colors - multiple colors defined to be able to use them in Styles
* Fonts - For nice 7 segment style time show I found [digital-dismay](https://www.dafont.com/digital-dismay.font) font

```YAML
image:
  - file: Images/hnr_logo.png
    id: boot_logo
    resize: 300x300
    type: RGB565
    transparency: alpha_channel

color:
  - id: my_red
    red: 100%
    green: 0%
    blue: 0%
  - id: my_darkred
    red: 50%
    green: 0%
    blue: 0%
  - id: my_yellow
    red: 100%
    green: 100%
    blue: 0%
  - id: my_green
    red: 0%
    green: 100%
    blue: 0%
  - id: my_darkgreen
    red: 0%
    green: 70%
    blue: 0%
  - id: my_blue
    red: 0%
    green: 0%
    blue: 100%
  - id: my_gray
    red: 50%
    green: 50%
    blue: 50%
  - id: my_white
    red: 100%
    green: 100%
    blue: 100%
  - id: my_black
    red: 0%
    green: 0%
    blue: 0%

font:
  - file: "fonts/digital-dismay.regular.otf"
    id: font_clock
    size: 200
    glyphs: ":0123456789"
  - file: "fonts/digital-dismay.regular.otf"
    id: font_clock_seconds
    size: 32
    glyphs: ":0123456789"
  - file: "gfonts://Roboto"
    id: roboto
    size: 40
    bpp: 4
    glyphs: |-
      !"%()+=,-_.:°0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ abcdefghijklmnopqrstuvwxyzĄČĘĖĮŠŲŪŽąčęėįšųūž
```

### LVGL

1. I wanted to use was styles, this allows better control of style used (easy to change)
2. I wanted to use LVGL pages, as it makes easy in future to define additional page and even switch between them
3. I found grid style organization fits me best. Main page was designe to have more grids just in case I want to display more icons there (future use). For curent setup it would have been enough to use one collumn.

```YAML
lvgl:
  buffer_size: 25%
  disp_bg_color: 0x000000
  style_definitions:
    - id: time_style
      text_color: my_green
      text_font: font_clock
    - id: time_style_night
      text_color: my_darkgreen
      text_font: font_clock
    - id: seconds_style
      text_color: my_green
      text_font: font_clock_seconds
    - id: date_style
      text_color: my_green
      text_font: roboto
  pages:
    - id: boot_page
      widgets:
        - obj:
            id: boot_screen
            x: 0
            y: 0
            width: 100%
            height: 100%
            bg_color: 0x000000
            bg_opa: COVER
            radius: 0
            pad_all: 0
            border_width: 0
            widgets:
              - image:
                  align: CENTER
                  src: boot_logo
              - spinner:
                  align: CENTER
                  height: 50
                  width: 50
                  spin_time: 1s
                  arc_length: 60deg
                  arc_width: 8
                  indicator:
                    arc_color: 0x18bcf2
                    arc_width: 8
            on_press:
              - lvgl.widget.hide: boot_screen 
    - id: clock_page 
      bg_color: 0x000000 
      widgets:
        - obj: # a properly placed coontainer object for all these controls
            align: CENTER
            width: 100%
            height: 100%
            bg_opa: TRANSP
            border_opa: TRANSP
            pad_all: 1
            layout: # enable the GRID layout for the children widgets
              type: GRID # split the rows and the columns proportionally
              grid_columns: [FR(1), FR(2), FR(2), FR(2), FR(2), FR(1)] # equal
              grid_rows: [FR(15), FR(10), FR(50), FR(5), FR(15)] # like percents
            widgets:
              - label:
                  id: clock_label_date
                  grid_cell_column_pos: 1
                  grid_cell_row_pos: 0
                  grid_cell_column_span: 4
                  grid_cell_row_span: 1
                  grid_cell_x_align: CENTER
                  grid_cell_y_align: END
                  # border_color: 0xffffff
                  # border_width: 1
                  align: CENTER
                  styles: date_style
                  text: "1999-01-01"
              - label:
                  id: clock_label
                  grid_cell_column_pos: 1
                  grid_cell_row_pos: 2
                  grid_cell_column_span: 4
                  grid_cell_row_span: 1
                  grid_cell_x_align: CENTER
                  grid_cell_y_align: CENTER
                  # border_color: 0xffffff
                  # border_width: 1
                  align: CENTER
                  styles: time_style
                  text: "99:99"
              - label:
                  id: clock_label_seconds
                  grid_cell_column_pos: 5
                  grid_cell_row_pos: 3
                  grid_cell_column_span: 1
                  grid_cell_row_span: 1
                  grid_cell_x_align: END
                  grid_cell_y_align: END
                  # border_color: 0xffffff
                  # border_width: 1
                  align: CENTER
                  styles: seconds_style
                  text: "99s"
              
              - label:
                  id: clock_label_dateofweek
                  grid_cell_column_pos: 1
                  grid_cell_row_pos: 4
                  grid_cell_column_span: 4
                  grid_cell_row_span: 1
                  grid_cell_x_align: CENTER
                  grid_cell_y_align: START
                  # border_color: 0xffffff
                  # border_width: 1
                  align: CENTER
                  styles: date_style
                  text: "Sunday"
```

### Scripts

I used scripts to update seconds, time(hour and minute), date (month day and day of the week) LVGL labels. 

As I use then in multiple places thats why I use scripts.
Scripts are used on boot that would set initial time after boot and not wait for time to come to update LVGL object.

```YAML
script:
  - id: script_update_seconds
    then:
      - lvgl.label.update:
          id: clock_label_seconds
          text: 
            format: "%02d"
            args:
              - id(time_comp).now().second
  - id: script_update_time
    then:
      - lvgl.label.update:
          id: clock_label
          text: 
            format: "%02d:%02d"
            args:
              - id(time_comp).now().hour
              - id(time_comp).now().minute
  - id: script_update_date
    then:
      - lvgl.label.update:
          id: clock_label_date
          text: 
            format: "%04d-%02d-%02d"
            args:
              - id(time_comp).now().year
              - id(time_comp).now().month
              - id(time_comp).now().day_of_month
      - lvgl.label.update:
          id: clock_label_dateofweek
          text: !lambda |-
            auto day_of_week = id(time_comp).now().day_of_week;
            //if (!time.is_valid()) return std::string("Invalid time");
            const char* days[] = {"Sekmadienis","Pirmadienis", "Antradienis", "Trečiadienis", "Ketvirtadienis", "Penktadienis", "Šeštadienis"};
            char buf[48];
          
            snprintf(buf, sizeof(buf), "%s",
                     days[day_of_week-1]);
            return std::string(buf);
```      


## End Result

### Home Assistant

![Result](/Photos/HomeAssistant1.png "Result")
- `Antiburn` switch enables antiburn of screen. Still thinking when to use it. Maybe during 3am-5am or when nobody is home (contorlled by HomeAssitant) or ...
- `Night mode` switch can toggles night mode by applying different style, backlight or ... 

### Photos

![Result](/Photos/result.jpg "Result")
![Back Side](/Photos/backside.jpg "Back Side")
![Clock](/Photos/alone.jpg "Clock")

## 3D File

The Manyfacturer has shared 3d STEP file for case.  You can even buy screen with already made Acrylic case. It was not suitable for me as I need it to be standing.  
I took it organized a bit (grouped properly) and modified (added holder) to be able to stand with 30° angle. 

[Modified STEP file here](3Dprint/3D_File_CrowPanel_3.5-ESP32_HMI-2.2.step)

![3DView](/3Dprint/3Dview.png "View1")
![3DView](/3Dprint/3Dview2.png "View2")


## ToDo

- Decide when/if to use `Antiburn` mode
- Maybe add additional icons that would show HomeAssitant things
- Maybe create separate LVGL page that could show weather forecast