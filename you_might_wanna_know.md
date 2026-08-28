# diff all

# version
# Betaflight / STM32H743 (SH74) 4.5.5 Aug 28 2026 / 00:27:13 (norevision) MSP API: 1.46
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

# aux
aux 0 27 0 1900 2100 0 0
aux 1 50 3 975 1075 0 0

# rxrange
rxrange 0 1020 2020
rxrange 1 1020 2020
rxrange 2 1020 2020
rxrange 3 1020 2020

# master
set dyn_notch_count = 1
set dyn_notch_q = 500
set acc_calibration = 36,20,10,1
set serialrx_provider = SBUS
set dshot_bidir = ON
set motor_pwm_protocol = DSHOT300
set motor_output_reordering = 2,3,0,1,4,5,6,7
set failsafe_switch_mode = KILL
set auto_disarm_delay = 0
set runaway_takeoff_prevention = OFF
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