# Multi-purpose MP3 doorbell for home assistant

An ESP32 based doorbell for use with home assistant with the doorbell sound of your choice via SD card.

- A robust 3D printed structure for mounting on a DIN rail
- provide your own sounds via MP3 SD card
- support for doorbell light including light effects when ringing  
- Integrated into home assistant via ESP home so that you can use the doorbell to trigger subsequent actions and automations.  
- low cost hardware <10 USD
- can be used as audio output device for any automation - not limited to doorbell usecase
- Example code provided (e.g. trigger camera snapshot)

![rendering](./print/rendering/rendering.png)


## Mechanics

The only mechanical part required is the housing for the electronics. 
The case consist of a base and a side wall connected via screws. 
It can fit on a DIN rail and is tightened via a hook. 

If you have two color print capability, you can optionally print label for the screw connectors and the top of the case.


All external connection are via screw connectors.
- Power supply is 5V DC (you can use low cost DIN rail power supply for this. Do not use your old fashioned bell transformer as this is AC!)
- Can be connected to a single 8 Ohm speaker for output, output volume is controlled via SW.
- The bell button is a simple closing contact to ground with internal pullup.
- Optionally, an LED can be attached via GPIO output connected to ground, e.g. to light up the bell button at night.



### 3D-Printed Parts

| Filename                     | Thumbnail                                                           | Required |
| ---------------------------- | --------------------------------------------------------------------| -------- |
| `./print/case.stl`           | <img src="./print/rendering/case.png" alt="frame" width="300"/>     | 1        |
| `./print/sidewall.stl`       | <img src="./print/rendering/sidewall.png" alt="side wall"/>         | 1        |
| `./print/hook.stl`           | <img src="./print/rendering/hook.png" alt="hook"/>                  | 1        |
| `./print/label.stl`          | <img src="./print/rendering/label.png" alt="label"/>                | optional |
| `./print/label_top.stl`      | <img src="./print/rendering/label_top.png" alt="label_top"/>        | optional |
| `./print/label_bottom.stl`   | <img src="./print/rendering/label_bottom.png" alt="label_bottom"/>  | optional |

All printed parts designed for PETG. 
Best experience on my printer was to print the case using tree style support. 
No rafts/brim etc. reguired for any model.

### Required screws

| Name              | Spec    | Required |
| ----------------- | ------- | -------- |
| Countersunk screw | M4 6mm  | 6        |

## Assembly

Use hotglue to hold all components in place (screw terminals, ESP32 and RF player).


## Electronics

### Part list

| Name                         | Required          | Note      |
| ---------------------------- | ----------------- | --------- |
| ESP32 WROOM USB-C Dev Module | 1                 |           |
| DF Player mini               | 1                 |           |
| micro SD card                | 1                 | to Hold MP3 files. Check DF player specs to pick the right card, e.g. max capacity |
| 8 Ohm Speaker                | 1                 |           |
| Doorbell button              | 1                 | optional LED lighting |




### Schematics

TBD

- DF Player RX (Pin 2) = ESP GPIO26 TX (Pin 15)
- DF Player TX (Pin 3) = ESP GPIO27 RX (Pin 16)
- DF Player Spk1+Spk2 = Speaker 8 Ohm
- Doorbell button = GPIO33 (internal pullup)
- Doorbell LED = GPIO32



# Example usages 

Adopting the ESP as device to home assistant via ESP home allows for flexible usage for different scenarios for which the following chapter illustrates some examples.



## MP3 doorbell

The YAML provided as ESP home integration already includes this doorbell usecase "hardcoded" for robustnes reasons as it does not require an automation in HA and no signals need to be send to home assistant back and forth
If you don't need this, just strip down the code to make it become a generic device to play MP3 files triggered by home assistant. 

Features
- Doorbell usecase implemented without any HA interaction
- Configurable volume and doorbell sound via HA GUI
- Test buttons and diagnosis signals provided to HA GUI
- Full DF player API provided to HA for use in automations

You can find the YAML source code for home assistant in `./esphome_src/doorbell-sound.yaml`



Note: DF player only supports certain SD cards and requires it to be formated in the right file system. See DF player documentation for details. 
It is also noteworthy that DF player plays the MP3 files on the SD card by the order of the FAT entry. Advising the player to play file #3 plays the third file that was written to the sd card - no matter of its filename or folder location.


Tip: Consider seasonal modes such as special Halloween sounds for your doorbell

## Camera snapshot

As the doorbell exposes the doorbell button as entity in HA, you can use a change to this entity state to trigger automations.
This can be anything, e.g. switching the light on at the door when its dark.
A more advanced usecase would be to take a snapshot of the visitor and interpret it via Gemini integration in order to send you a push notification including information about who rang the bell.

This can be achieved by adding the Gemini add on to HA and some automation.
I use the following prompt:
```
Very briefly describe what you see in the brick semi-circle area
in front of my frontdoor in this image taken when the doorbell was rung.
Your message needs to be very short to fit in a phone notification and
must not be longer than a single sentence. Don't describe stationary
objects or buildings or vehicles in driveway. If you cannot identify
people for sure, then tell me so. Otherwise tell me whether it seems
to be visitors such as kids or a family or if it is a handcraftsman, a
postman or a delivery man.  Please don't describe their clothing. Please
tell me if there is no one but there is a parcel or other item that they
left in front of the door."
```

Example automation (using some helper scripts to create and uptate push notification to mobile devices for illustrative reasons):
```yaml
  triggers:
  - trigger: state
    entity_id:
    - binary_sensor.doorbell_sound_doorbell
    from: 'off'
    to: 'on'
  conditions: []
  actions:
  - metadata: {}
    data: {}
    target:
      entity_id: camera.camera_frontdoor
    action: camera.turn_on
  - data:
      filename: '{{ snapshot_create_file_path }}'
    enabled: true
    target:
      entity_id: camera.camera_frontdoor
    action: camera.snapshot
  - metadata: {}
    data:
      notify_devices:
      - YOURDEVICEID
      notify_home_or_away: Both
      data_notification_icon: mdi:doorbell
      notify_title: Türklingel {{ time }}
      notify_message: Please wait for Gemini to interpret the image
      data_visibility: public
      data_ios_interruption_level: time-sensitive
      notify_tts_in_car: true
      data_tag: '{{this.context.id}}'
      data_camera: '{{ snapshot_access_file_path }}'
    action: script.notify_devices
  - parallel:
    - choose: []
      default:
      - entity_id: camera.camera_frontdoor
        data:
          filename: /media/frontdoor_camera/archive/motion_{{now().strftime("%Y%m%d-%H%M%S")
            }}.jpg
        enabled: true
        action: camera.snapshot
      - entity_id: camera.camera_frontdoor
        data:
          filename: /media/frontdoor_camera/last_motion.jpg
        action: camera.snapshot
    - choose: []
      default:
      - data:
          prompt: "Very briefly describe what you see in the brick semi-circle area
            in front of my frontdoor in this image taken when the doorbell was rung.
            \nYour message needs to be very short to fit in a phone notification and
            must not be longer than a single sentence. \nDon't describe stationary
            objects or buildings or vehicles in driveway. \nIf you cannot identify
            people for sure, then tell me so. \nOtherwise tell me whether it seems
            to be visitors such as kids or a family or if it is a handcraftsman, a
            postman or a delivery man.  Please don't describe their clothing. Please
            tell me if there is no one but there is a parcel or other item that they
            left in front of the door."
          image_filename: '{{ snapshot_create_file_path }}'
        response_variable: generated_content
        action: google_generative_ai_conversation.generate_content
      - metadata: {}
        data:
          notify_devices:
          - YOURDEVICEID
          notify_home_or_away: Both
          data_notification_icon: mdi:doorbell
          notify_title: Türklingel {{ time }}
          notify_message: '{{ generated_content.text }}'
          data_visibility: public
          data_ios_interruption_level: time-sensitive
          notify_tts_in_car: true
          data_tag: '{{this.context.id}}'
          data_camera: '{{ snapshot_access_file_path }}'
        action: script.notify_devices
  variables:
    generated_content: undefined
    camera: camera.camera_frontdoor
    camera_name: '{{ states[camera].name }}'
    time: '{{ now().strftime("%H:%M") }}'
    date: '{{ now().strftime("%Y-%m-%d") }}'
    snapshot_create_file_path: /config/www/tmp/snapshot_{{ states[camera].object_id}}.jpg
    snapshot_access_file_path: '{{ snapshot_create_file_path | replace(''/config/www'',''/local'')}}'
  mode: parallel
  max: 10
```


## Home Reminder

As the doorbell provides an API with full control of the DF player, you can use this to play sounds on certain triggers via automations.
An example would be, to play a sound to not forget to switch off the lights when leaving home (e.g. when your front door contact is triggered).

For this, you can add to your SD card suitable sound effects or even prepare TTS output in MP3s (there are free tools on the internet to create such MP3s for predefined text input via TTS services).

Source code for these scripts is under `./ha_scripts`

```yaml
triggers:
- entity_id:
- binary_sensor.doorstate_frontdoor_contact
to: 'on'
id: start
for:
    hours: 0
    minutes: 0
    seconds: 5
from: 'off'
trigger: state
conditions: []
actions:
- action: esphome.doorbell_sound_dfplayer_play
data:
    file: 46
mode: single
```
