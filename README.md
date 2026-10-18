README.md

基于 **STM32F103C8T6 + FreeRTOS + ESP8266** 的物联网温湿度监测系统。DHT11 采集温湿度，SSD1306 OLED 本地显示，ESP8266 通过 TCP 上报到服务器，支持按键翻页、长按进入 STOP 低功耗、按键 EXTI 唤醒，以及系统级看门狗监控。

![MCU](https://img.shields.io/badge/MCU-STM32F103C8T6-blue)
![RTOS](https://img.shields.io/badge/RTOS-FreeRTOS-green)
![IDE](https://img.shields.io/badge/IDE-Keil%20MDK%20ARMCC%20V5-orange)
![License](https://img.shields.io/badge/License-MIT-lightgrey)

---

## 目录

- [功能特性](#功能特性)
- [硬件清单](#硬件清单)
- [接线说明](#接线说明)
- [软件架构](#软件架构)
- [关键设计](#关键设计)
- [目录结构](#目录结构)
- [构建与烧录](#构建与烧录)
- [使用说明](#使用说明)
- [配置说明](#配置说明)
- [已知限制](#已知限制)
- [开发过程](#开发过程)
- [后续计划](#后续计划)
- [许可证](#许可证)

---

## 功能特性

| 功能 | 说明 |
| :--- | :--- |
| **温湿度采集** | DHT11 单总线，每 1.5 秒采集一次，含合理范围过滤（湿度 20~90%，温度 0~50°C） |
| **本地显示** | SSD1306 OLED 128×64，第一页显示温湿度，第二页显示错误计数，按键翻页 |
| **WiFi 上报** | ESP8266 AT 指令，TCP 直连服务器，每 1.5 秒一条 `Temp:xx.x,Humi:xx.x\n` |
| **断线重连** | `AT+CWQAP` 清残留 + `AT` 探测 + 40 秒超时 + 指数退避，避免 `busy p...` 死循环 |
| **看门狗监控** | 4 个任务心跳全部新鲜才喂狗，任一任务卡死 6.5 秒自动复位 |
| **低功耗** | 长按按键 3 秒进入 STOP 模式，RTC 分片喂狗，按键 EXTI 唤醒 |
| **唤醒恢复** | 时钟重配、USART/I2C 重初始化、ESP8266 重连、OLED 重显示 |
| **状态指示** | PC13 LED 每秒闪（正常）、快闪 5 下（WiFi 故障）、慢闪 3 下（DHT 故障） |

---

## 硬件清单

| 器件 | 型号 | 数量 | 说明 |
| :--- | :--- | :--- | :--- |
| 主控 | STM32F103C8T6 最小系统板 | 1 | Cortex-M3，72MHz，20KB RAM，64KB Flash |
| 温湿度传感器 | DHT11（3 针模块） | 1 | 单总线，需 4.7kΩ~10kΩ 上拉 |
| OLED 显示屏 | SSD1306 0.96" 128×64 | 1 | 4 针 I2C，地址 0x78 |
| WiFi 模块 | ESP8266（ESP-12F / ESP8266MOD） | 1 | AT 固件，UART 9600bps |
| 按键 | 轻触开关 | 1 | 一端接 PA11，另一端接 3.3V |
| USB-TTL | CH340 / CP2102 | 1 | 用于 USART1 调试输出 |

> **ESP8266 供电注意**：峰值电流可达 300~500mA，建议使用独立 3.3V 电源（如 AMS1117 模块），并与 STM32 共地。

---

## 接线说明

| STM32 引脚 | 外设 | 功能 | 注意事项 |
| :--- | :--- | :--- | :--- |
| **PA1** | DHT11 DATA | 单总线数据 | 外接 4.7kΩ~10kΩ 上拉到 3.3V |
| **PB8** | OLED SCL | I2C1_SCL（重映射） | 4.7kΩ 上拉到 3.3V |
| **PB9** | OLED SDA | I2C1_SDA（重映射） | 4.7kΩ 上拉到 3.3V |
| **PA2** | ESP8266 RX | USART2_TX | 交叉连接 |
| **PA3** | ESP8266 TX | USART2_RX | 交叉连接 |
| **PA9** | USB-TTL RX | USART1_TX | `printf` 调试输出，115200bps |
| **PA10** | USB-TTL TX | USART1_RX | — |
| **PA11** | 按键 | EXTI11，`IT_RISING` + `PULLDOWN` | **另一端接 3.3V** |
| **PB0** | ESP8266 EN | 高电平使能 | 若 CH_PD 已直焊 3.3V 可不接 |
| **PC13** | 板载 LED | 状态指示 | 低电平点亮 |

**ESP8266 启动硬性条件**：

| 引脚 | 状态 |
| :--- | :--- |
| **EN (CH_PD)** | 高电平（3.3V） |
| **GPIO15 (D8)** | 接地（GND） |
| **GPIO0** | 悬空或高电平（拉低会进下载模式） |
| **3V3 / GND** | 独立 3.3V 供电，与 STM32 共地 |

---

## 软件架构

### 任务划分

| 任务 | 优先级 | 栈大小 | 职责 |
| :--- | :--- | :--- | :--- |
| `DHT_Task` | High | 1024 B | 每 1.5s 读 DHT11，投递 `Queue_OLED` + `Queue_WiFi` |
| `Key_Task` | AboveNormal | 1536 B | 20ms 轮询 PA11，短按释放信号量，长按 3s 进 STOP |
| `OLED_Task` | Normal | 1024 B | 取队列数据，翻页显示，`page_changed` 控制清屏 |
| `WiFi_Task` | Low | 1024 B | `ESP8266_Poll()` 推进状态机，`ESP8266_SendData()` 上报 |
| `StatusLED_Task` | Low | 512 B | 正常每秒闪，WiFi/DHT 故障不同闪烁模式 |
| `WatchDog_Task` | Low | 1024 B | 4 个心跳全新鲜才喂狗，每 10s 打印栈高水位 |

### IPC 机制

| 机制 | 用途 | 配置 |
| :--- | :--- | :--- |
| `Queue_OLED` | DHT → OLED 数据 | 长度 1，元素 `DHT_Data_t` |
| `Queue_WiFi` | DHT → WiFi 数据 | 长度 10（约 15 秒缓冲） |
| `Sem_Key` | 按键 → OLED 翻页 | 初值 0，长度 1 |
| 心跳变量 | 任务健康状态 | `volatile uint32_t` × 4 |

### 数据流
