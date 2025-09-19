ESPHome configuration for enabeling WAVESHARE-S3-AUDIO-BOARD (https://www.waveshare.com/esp32-s3-audio-board.htm)
to be used as a HomeAssistant Voice Satellite. Features such as simultanious music/and announcements and
continious on-board wake-word detection.

The device's DAC and ADC share pins for the i2s bus so to be able to configure two i2s_audio points for 
simultanious audio in/out I had to path the DAC (es8311) component, adding a setting to force it to become 
i2s master while still configuring the ESP and ADC to act as i2s slaves. I also had to fix MCLK/BLCK calculations.

example code features:
* complete Voice Assistant setup
* Onboard Wake-Word detection
* Led animations and event sounds
* working control buttons
* exposed announcement and music media_players
* built in alarm and timer

Steps to use the custom ES8311 component:
``` yaml
substitutions:
  i2c_id: internal_i2c
  i2s_mclk_multiple: 256
  i2s_bps_spk: 16bit
  i2s_bps_mic: 16bit
  i2s_sample_rate_spk: 16000
  i2s_sample_rate_mic: 16000 # must be 16000 for voice_assistant to work(?)
  i2s_use_apll: true
  
external_components:
  - source:
      type: git
      url: https://github.com/sw3Dan/waveshare-s2-audio_esphome_voice
      ref: main
    components: [ es8311 ]
    refresh: 0s

i2c:
  - id: $i2c_id
    sda: GPIO11
    scl: GPIO10
    scan: true

i2s_audio:
  - id: i2s_output
    i2s_lrclk_pin: 
      number: GPIO14
      allow_other_uses: true
    i2s_bclk_pin:  
      number: GPIO13
      allow_other_uses: true
    i2s_mclk_pin: # <-- for ESP to know what port to configure MCLK output
      number: GPIO12

  - id: i2s_input
    i2s_lrclk_pin:  
      number: GPIO14
      allow_other_uses: true
    i2s_bclk_pin:  
      number: GPIO13
      allow_other_uses: true

audio_dac:
  - platform: es8311
    id: es8311_dac
    i2c_id: $i2c_id
    use_mclk: true
    sample_rate: $i2s_sample_rate
    bits_per_sample: $i2s_bps_spk
    force_master: true # <-- New: to configure device as i2s master
    mclk_multiple: i2s_mclk_multiple # <-- New: for the i2s bus MCLK/BLCK calculations

audio_adc:
  - platform: es7210
    id: adc_mic
    i2c_id: $i2c_id
    sample_rate: $i2s_sample_rate
    bits_per_sample: $i2s_bps_mic

microphone:
  - platform: i2s_audio
    id: i2s_mics
    i2s_din_pin: GPIO15
    adc_type: external
    pdm: false
    i2s_audio_id: i2s_input
    i2s_mode: secondary # <-- so that ESP wont generate LRCLK/BLCK
    mclk_multiple: $i2s_mclk_multiple
    sample_rate: $i2s_sample_rate_mic # must be 16000 for VA (?)
    bits_per_sample: $i2s_bps_mic
    
speaker:
  - platform: i2s_audio
    id: i2s_audio_speaker
    i2s_dout_pin: GPIO16
    i2s_audio_id: i2s_output
    i2s_mode: secondary # component patch will force ES8311 to take command
    dac_type: external
    timeout: never
    buffer_duration: 100ms
    audio_dac: es8311_dac
    sample_rate: $i2s_sample_rate_spk
    bits_per_sample: $i2s_bps_spk
    use_apll: $i2s_use_apll # dont know if this is enforced when in 'secondary' i2s mode
    mclk_multiple: $i2s_mclk_multiple
    channel: stereo

  - platform: mixer
    id: mixing_speaker
    output_speaker: i2s_audio_speaker
    num_channels: 2
    source_speakers:

** Then create speaker: speaker/resamplers as needed ** 
```

If you need a camera include this yaml package.
This package comes with a preconfigured camera and a bunch of camera control is exposed to HomeAssistant.
``` yaml
packages:
  remote_package_shorthand: github://esphome/sw3Dan/waveshare-s2-audio_esphome_voice/waveshare_camera_pkg.yaml@main
```

General TODO/wishlist for the device.
* UI: disable LEDS
* UI: disable Voice/wake-word
* UI/ESP: Forward output to external speaker (music assistant announcement support?)
* ESP: lower volume on room speakers when wake-word detected
* UI: expose mic and model sensitivity calibration settings
* UI: select buttons behavior
* ESP: long-press (volume) and double-click  (deactivate Voice) button support
* ESP: Volume percentage Led animation
* ESP: rotary rainbow - assistant working
* ESP: alarm flash led animation
* ESP: pulsing red - no connection led animation
* ESP: code cleanup - change code structure and split into packages
* UI: structure/names
* ESP: mmWave module (control mic activation to limit room exposure)
* ESP: stream/listen to audio (mic output volume trigger)
* ESP: fix RTC
* UI: set TimeZone
