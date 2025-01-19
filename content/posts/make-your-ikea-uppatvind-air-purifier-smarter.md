+++
title = "宜家 IKEA UPPÅTVIND 空气净化器智能改造方案"
date = "2025-01-19T18:43:48+08:00"
author = ""
authorTwitter = "" #do not include @
cover = "https://s2.loli.net/2025/01/19/lXbOuAJcghoNRkn.webp"
tags = ["ikea", "uppatvind", "home assistant"]
keywords = ["ikea", "uppatvind", "home assistant"]
description = "让 UPPÅTVIND 也加入到 Home Asssistant 的大家庭里来！"
showFullContent = false
readingTime = false
hideComments = false
color = "" #color from the theme settings
+++
## 为什么要改造它？
`IKEA UPPÅTVIND` 是一款相当具有性价比的空气净化器。只需要不到 200 人民币，你就能获得一个 HEPA 12 过滤等级的净化器，让室内的空气质量变得更好！

然而，pokon548 发现这款机器并不够智能。在闲暇上网冲浪之余，pokon548 发现这款机器可以轻松的改造为智能版，以轻松获得强大的联动能力，遂决定动手研究改造方案。

## 致谢！
在本文开始之前，pokon548 特别感谢 `jonathonlui` 在 GitHub 上开源的[改造方案](https://github.com/jonathonlui/esphome-ikea-uppatvind)。其使用 ESP8266 作为控制器，并引出部分测试点来实现信号接收和控制。

本文很大程度上参考了这个项目的思路，在其基础上，结合该项目的 Issues，做了以下改进：
- 使用 `ESP32` 开发板，利用硬件级 PWM 频率读取器，获得更精确的测量结果；
- 使用 `TP9` 而不是 `TP7`，直接读取风扇的 PWM 值，获得准确的风扇速度；
- 增加电平转换逻辑，双向转换 `3.3V` 与 `5V` 的电平信号，避免 `5V` 电压直接接入 `ESP32` 带来的潜在风险[^1]；
- 引出 `TP6`，远程得知是否需要更换滤芯。

## 改造效果
最终，可以通过 Home Assistant 远程控制 IKEA UPPÅTVIND 空气净化器，以做到：
- 调节风速；
- 开 / 关空气净化器本体；
- 得知当前风速；
- 得知是否需要更换滤芯。

## 改造步骤
> ## 警告！前方有怪兽！
>
> 下文将会从硬件上改装 `IKEA UPPÅTVIND 乌波温` 空气净化器。请注意：
> - 这将导致你不再能够满足宜家的无忧退货政策！
> - 本文改装需要一定技术水平。如果你不熟悉焊接、拆卸以及编程相关的经验，请勿继续！否则你可能会*永久性地*损坏你的空气净化器！继续操作即代表你理解并同意自担风险！

参考 [这个项目](https://github.com/jonathonlui/esphome-ikea-uppatvind)，以及里面的 [这个 issue](https://github.com/jonathonlui/esphome-ikea-uppatvind/issues/6#issuecomment-1786782118)，可以得到如下可行的改造步骤：

0. 确保拥有以下硬件材料：
	- IKEA UPPÅTVIND 一个；
	- esp32 开发板一个（已焊好排插）；
	- MCU-401 电平转换模块一个；
	- 杜邦一公分二母线两条；
	- 杜邦母头单头一头镀锡五条；
	- 3m 双面强力胶若干；
	- 焊接工具、焊料若干；
	- 螺丝刀套装（需要至少能拧开五角星和三角形的螺丝刀！）。
1. 将 esp32 开发板连上电脑，初始化并写入 esphome 基础逻辑。只要确保后续可以无线推送固件代码即可。写入完毕后断开连接备用；
2. 拆开 IKEA UPPÅTVIND，露出原有电路板；
3. 使用单头杜邦线，焊接并引出以下测试点位：
	- TP2：GND，接入杜邦一公分二母线，用于：
		- 为 esp32 提供电源；
		- 为 MCU-401 电平模块（用于 3.3v <-> 5v 电平信号双向转换）提供电源。
	- TP3：5V VCC，接入杜邦一公分二母线，用于：
		- 为 esp32 提供电源；
		- 为 MCU-401 电平模块（用于 3.3v <-> 5v 电平信号双向转换）提供电源。
	- TP4：风速切换按钮，用于 esp32 切换净化器风速；
	- TP6：滤芯更换 LED 灯，用于通知 esp32 需要更换滤芯；
	- TP9：风扇风速 PWM，用于 esp32 识别当前净化器风速。
4. 在 esp32 定义如下针脚：
	1. D18：[GPIO Output](https://esphome.io/components/output/gpio)，配合 [Button Component](https://esphome.io/components/button/)，用来模拟 IKEA UPPÅTVIND 的风速切换按钮；
	2. D19：[GPIO Binary Sensor](https://esphome.io/components/binary_sensor/gpio.html)，用来读取 IKEA UPPÅTVIND 更换滤芯 LED 灯的状态，转而得知滤芯是否需要更换；
	3. D21：[Pulse Counter](https://esphome.io/components/sensor/pulse_counter.html)，用来读取 IKEA UPPÅTVIND 当前的风扇速度。
5. 通过如下方式接线：

| IKEA UPPÅTVIND | 一分二杜邦线             | MCU-401               | ESP32 开发板             |
| -------------- | ------------------ | --------------------- | --------------------- |
| TP2            | 引出 TP2(1) 和 TP2(2) | TP2(1) -> GND (VCC 侧) | TP2(2) -> GND (VIN 侧) |
| TP3            | 引出 TP3(1) 和 TP3(2) | TP3(1) -> VCC         | TP3(2) -> VIN         |
| TP4            | 不需要                | TP4 -> B1             | A1 -> D18             |
| TP6            | 不需要                | TP6 -> B2             | A2 -> D19             |
| TP9            | 不需要                | TP9 -> B3             | A3 -> D21             |

> 同时，别忘了将 `MCU-401` 3.3V 与 GND 接到 esp32 开发板相应的位置上！

6. 用 3m 胶带固定开发板、线材以及电平转换板；
7. 组装改装后的 IKEA UPPÅTVIND，上电，并向 esp32 写入如下 esphome 代码：
```yaml
esphome:
  name: uppatvind

esp32:
  board: esp32dev
  framework:
    type: arduino

# Enable logging
logger:

# Enable Home Assistant API
api:

# Allow Over-The-Air updates
ota:
- platform: esphome

# Allow provisioning Wi-Fi via serial
improv_serial:

# Use ntp server from the mainland of China
time:
  - platform: sntp
    id: sntp_time
    timezone: Asia/Shanghai
    servers:
     - cn.pool.ntp.org
     - ntp.ntsc.ac.cn

# wifi connection
wifi:
  # Set up a wifi access point
  ap: {}

# In combination with the `ap` this allows the user
# to provision wifi credentials to the device via WiFi AP.
captive_portal:

dashboard_import:
  package_import_url: github://esphome/example-configs/esphome-web/esp32.yaml@main
  import_full_config: true

# Sets up Bluetooth LE (Only on ESP32) to allow the user
# to provision wifi credentials to the device.
esp32_improv:
  authorizer: none

sensor:
  - platform: pulse_counter
    pin: 21
    name: "Fan Speed PWM"
    update_interval: 0.5s
    accuracy_decimals: 0
    unit_of_measurement: ''
    icon: mdi:speedometer
    id: pwm
    filters:
      - lambda: !lambda |-   # this section could be better or different...
          int poff = 0;
          int y = int(x);
          if (y < 100) {
            return 0;
          } else if (y <7700) {
            return 1;
          } else if (y <12100) {
            return 2;
          } else if (y < 17000) {
            return 3;
          } else {
            return 0;
          }

output:
  - platform: gpio
    pin:
      number: 18
      inverted: true  
      # "Open drain" used so the Wemos does't pull the pin 
      # up or down when it's not being pressed.
      mode: OUTPUT_OPEN_DRAIN
    id: speed-switch-gpio

binary_sensor:
  - platform: gpio
    pin: 19
    name: "Filter Replacement LED"
    device_class: tamper

button:
  - platform: output
    name: "UPPÅTVIND Button"
    output: speed-switch-gpio
    duration: 100ms
```
8. 等待 Home Assistant 发现你的智能版 `IKEA UPPÅTVIND`。Enjoy！

[^1]: ESP32 开发板在设计上并不支持 `5V` 电平信号直接输入输出。虽然部分开发板实际上可以短时间内承受小电流 `5V` 的电平信号，但是这可能会引入不确定因素。是否忽略该警告取决于你 :) 一些有关的讨论：[https://esp32.com/viewtopic.php?t=877](https://esp32.com/viewtopic.php?t=877)