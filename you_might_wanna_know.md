# version
# Betaflight / STM32H743 (SH74) 4.5.5 Aug 29 2026 / 00:19:46 (norevision) MSP API: 1.46
# config rev: e95b1db

# start the command batch
batch start

# reset configuration to default settings
defaults nosave

board_name MICOAIR743V2
manufacturer_id MICO
mcu_id 002d004a3434510b32333934
signature 

# feature
feature SERVO_TILT
feature RANGEFINDER
feature TELEMETRY
feature LED_STRIP
feature DISPLAY
feature OSD
feature CHANNEL_FORWARDING
feature TRANSPONDER
feature ESC_SENSOR

# serial
serial 20 1 500000 57600 0 115200

# aux
aux 0 27 0 1875 2100 0 0
aux 1 55 3 1400 1600 0 0
aux 2 56 3 900 1125 0 0

# rxrange
rxrange 0 1020 2020
rxrange 1 1020 2020
rxrange 2 1020 2020
rxrange 3 1020 2020

# master
set acc_calibration = 87,-4,7,1
set serialrx_provider = SBUS
set dshot_bidir = ON
set motor_pwm_protocol = DSHOT300
set motor_output_reordering = 2,3,0,1,4,5,6,7
set failsafe_switch_mode = KILL
set enable_stick_arming = ON

profile 0

profile 1

profile 2

profile 3

# restore original profile selection
profile 0

rateprofile 0

rateprofile 1

rateprofile 2

rateprofile 3

# restore original rateprofile selection
rateprofile 0

# save configuration
save