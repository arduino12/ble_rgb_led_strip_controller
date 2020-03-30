# ble_rgb_led_strip_controller
 Control a cheap BLE RGB LED strip controller!

## Introduction
I just got 1.7$ [12V BLE RGB LED strip controller](https://www.aliexpress.com/item/4000208329326.html) from AliExpress Shop4661053 Store!  
The QR code links to an unofficial [StripV5.0.1.apk](http://www.easytrack.net.cn/download/111SHENZHENSHUANGHONGYUAN) download page.  
I found official [auraLED](https://play.google.com/store/apps/details?id=wl.smartled.auraled) app from google play to work!  
I read [Uri Shaked- Reverse Engineering a Bluetooth Lightbulb](https://medium.com/@urish/reverse-engineering-a-bluetooth-lightbulb-56580fcb7546) and got inspired!  
After reverse engineering some commands I found that [kquinsland- JACKYLED-BLE-RGB-LED-Strip-controller](https://github.com/kquinsland/JACKYLED-BLE-RGB-LED-Strip-controller) worked on a similar controller!  
Anyway I'll write here what I found- it is still WIP.  
My plan is to buy 10+ more units and control them all via a single ESP32 board!

## Protocol
### Reversing
By sniffing ble packets while pressing buttons on the auraLED app,  
I reverse engineered most of my controller ble commands.  
I was too lazy for the timer commands stuff :)  
The app seems to putting garbage on non used bytes of the 9 bytes packet.  
Here is an example packet captured while pressing on a green button:  

#### `7e07050300ff0000ef`:
| byte | description |
| - | - |
| `7e` | command start |
| `07` | don't care (so I use 0 instad) |
| `05` | command id (this case set_color) |
| `03` | command sub-id (this case rgb color) |
| `00` | command arg1 (this case red value) |
| `FF` | command arg2 (this case green value) |
| `00` | command arg3 (this case blue value) |
| `00` | don't care (so I use 0 instad) |
| `ef` | command end |
	

### Controller have 2 states (use set_power command):
| states | description |
| - | - |
| `on` | LEDs show current mode colors |
| `off` | LEDs are off (current mode is saved) |

### Controller heve 5 modes (use *mode* commands to change)
| modes | description |
| - | - |
| `mode_grayscale` | can set grayscale from black to full white |
| `mode_temperature` | can set brightness and temperature(cold to warm white) |
| `mode_effect` | auto color change, can set patterns, brightness and speed |
| `mode_dynamic` | ? (maybe on-board mic is needed -mine just shows last color) |  
| `mode_rgb` | can set rgb values and brightness |

### Commands
#### `set_brightness(brightness):`
`7e 00 01 brightness 00 00 00 00 ef`  
`7e00010a00000000ef` 10%  
`7e00013200000000ef` 50%  
`7e00016400000000ef` 100%  
`brightness`: 0-100 (0x00-0x64)  
##### does not work on:  
mode_effect: jump gradient blink  
mode_grayscale  
mode_dynamic  

#### `set_effect_speed(speed):`
`7e 00 02 speed 00 00 00 00 ef`  
`7e00020000000000ef` 0  
`7e00020900000000ef` 9  
`7e00026400000000ef` 100  
`speed`: 0-100 (0x00-0x64)  

#### `set_mode_grayscale():`
`7e 00 03 00 01 00 00 00 ef`  
`7e00030001000000ef`
will show last grayscale color  

#### `set_mode_temperature(temperature):`
`7e 00 03 temperature 02 00 00 00 ef`  
`7e00038002000000ef` cold white  
`7e00038502000000ef` natural white  
`7e00038a02000000ef` warm white  
`temperature`: 128-138 (0x80-0x8a)  

#### `set_mode_effect(effect)`
`7e 00 03 effect 03 00 00 00 ef`  
`7e00038703000000ef` jump_rgb  
`7e00039203000000ef` gradient_rg  
`7e00039503000000ef` blink_rgbycmw  
##### `effect`:
| effect | description|
| - | - |
| `0x80` | r (red) |
| `0x81` | g (green) |
| `0x82` | b (blue) |
| `0x83` | y (yellow) |
| `0x84` | c (cyan) |
| `0x85` | m (magenta) |
| `0x86` | w (white) |
| `0x87` | jump_rgb |
| `0x88` | jump_rgbycmw |
| `0x89` | gradient_rgb |
| `0x8a` | gradient_rgbycmw |
| `0x8b` | gradient_r |
| `0x8c` | gradient_g |
| `0x8d` | gradient_b |
| `0x8e` | gradient_y |
| `0x8f` | gradient_c |
| `0x90` | gradient_m |
| `0x91` | gradient_w |
| `0x92` | gradient_rg |
| `0x93` | gradient_rb |
| `0x94` | gradient_gb |
| `0x95` | blink_rgbycmw |
| `0x96` | blink_r |
| `0x97` | blink_g |
| `0x98` | blink_b |
| `0x99` | blink_y |
| `0x9a` | blink_c |
| `0x9b` | blink_m |
| `0x9c` | blink_w |

#### `set_mode_dynamic(val):`
`7e 00 03 val 04 00 00 00 00 ef`  
`7e00030004000000ef` val 0  
Did not do anything on my controller (just freeze the current color).  

#### `set_power(is_on):`
`7e 00 04 is_on 00 00 00 00 ef`  
`7e00040000000000ef` off  
`7e00040100000000ef` on  

#### `set_color_for_grayscale_mode(grayscale):`
`7e 00 05 01 grayscale 00 00 00 ef`  
`7e00050100000000ef` black  
`7e00050132000000ef` gray  
`7e00050164000000ef` white  
`grayscale`: 0-100 (0x00-0x64)

#### `set_color_for_temperature_mode(temperature):`
`7e 00 05 02 temperature 00 00 00 ef`  
`7e00050200000000ef` cold  
`7e00050232000000ef` natural  
`7e00050264000000ef` warm  
`temperature`: 0-100 (0x00-0x64)

#### `set_color_for_rgb_mode(r, g, b):`
`7e 00 05 03 r g b 00 ef`  
`7e000503ff000000ef` red  
`7e00050300ff0000ef` green  
`7e0005030000ff00ef` blue  
`r, g, b`: 0-255 (0x00-0xff)

#### `set_val_for_dynamic_mode(val):`
`7e 00 06 val 00 00 00 00 ef`  
`7e00060000000000ef` val 0  
Did not do anything on my controller.  

#### `set_sensitivity_for_dynamic_mode(sensitivity):`
`7e 00 07 sensitivity 00 00 00 00 ef`  
`7e00070000000000ef` sensitivity 0  
Did not do anything on my controller.  

#### `set_rgb_pin_order(rgb_order):`
`7e 00 08 rgb_order 00 00 00 00 ef`  
`7e00080100000000ef` rgb  
`rgb_order`: 1:rgb 2:rbg 3:grb 4:gbr 5:brg 6:bgr  
Did not do anything on my controller.  

#### `set_rgb_pin_order(c1 c2 c3):`
`7e 00 81 c1 c2 c3 00 00 ef`  
`7e00810102030000ef` rgb  
`7e00810302010000ef` bgr  
`c1 c2 c3`: (1-3) use each value once!  

## Enjoy!
A.E.TECH
