# 工业边缘网关：温湿度采集与云端上报

> 基于 **STM32F407VET6 + FreeRTOS + LVGL** 的工业物联网边缘计算网关，实现 RS485 总线 Modbus 温湿度采集、480×320 电容触摸屏实时显示、MQTT 云端上报，以及断网数据本地缓存与手动补传。

**核心硬件平台**：

- 主控：**STM32F407VET6**（Cortex-M4F，168MHz，512KB Flash / 128KB SRAM）
- 屏幕：**3.5 寸 ILI9488 SPI 触摸屏**（480×320，18bit RGB666 色深）
- 无线模块：**ESP-01S**（ESP8266 AT 固件）
- RS485 收发器：**MAX3485**（DE/RE 单线方向控制）
- 传感器：**卡轨式 RS485 温湿度变送器**（DC 8~28V 供电）

---

## 功能特性

- **Modbus RTU 主站**：手写 CRC16（查表法 + 按位法），轮询读取温湿度寄存器（0x0300/0x0301，÷10）
- **LVGL 图形界面**：温湿度仪表盘、在线状态、运行时间实时显示，触摸屏交互
- **MQTT 3.1.1 客户端**：手写 CONNECT / SUBSCRIBE / PUBLISH / PINGREQ 报文，对接巴法云
- **断网续传**：W25Q64 后 4MB 空间做环形缓存，离线存数据、联网自动补传
- **多任务调度**：FreeRTOS 四任务（UI / 采集 / MQTT / 网络心跳）
- **CH340 调试串口**：printf 重定向，实时打印运行日志

---

## 硬件清单

| 物料 | 型号/规格 | 用途 |
|:---|:---|:---|
| 主控 | STM32F407VET6 最小系统板 | 运行 FreeRTOS 与全部业务逻辑 |
| 显示屏 | 3.5 寸 ILI9488 SPI（480×320） | LVGL 触控显示 |
| 触摸芯片 | XPT2046 | 电阻/电容触摸 |
| 无线模块 | ESP-01S（ESP8266 AT 固件） | MQTT 联网 |
| RS485 收发器 | MAX3485 | TTL ↔ RS485 电平转换 |
| 温湿度传感器 | 卡轨式 RS485 温湿度变送器（DC 8~28V） | Modbus RTU 从机 |
| SPI Flash | W25Q64 | 断网数据缓存 |
| USB 转串口 | CH340 | 调试日志输出 |
| 调试器 | ST-Link V2 | 程序下载与调试 |

---

## 引脚分配

| 外设 | MCU 引脚 | 说明 |
|:---|:---|:---|
| ESP-01S TX | PA10（USART1_RX） | 115200-8-N-1 |
| ESP-01S RX | PA9（USART1_TX） | 115200-8-N-1 |
| MAX3485 TX | PA2（USART2_TX） | 9600-8-N-1 |
| MAX3485 RX | PA3（USART2_RX） | 9600-8-N-1 |
| MAX3485 DE/RE | PB0 | 高=发送，低=接收 |
| LCD SCK / MISO / MOSI | PA5 / PA6 / PA7（SPI1） | 触摸屏共用 |
| LCD CS | PB6 | 片选 |
| LCD DC/RS | PB7 | 数据/命令 |
| LCD RST | PA11 | 复位 |
| LCD BL | PA8 | 背光，高电平点亮 |
| 触摸 CS | PB10 | XPT2046 片选 |
| W25Q64 SCK/MISO/MOSI | PB13 / PB14 / PB15（SPI2） | Flash 存储 |
| W25Q64 CS | PB12 | 片选 |
| CH340 TX / RX | PC10 / PC11（USART3） | 调试串口 115200 |

> ESP-01S 注意：VCC 独立 3.3V 供电（峰值电流 >300mA），EN(CH_PD) 接 3.3V，TX/RX 与 MCU 交叉连接。

---

## 系统架构

```
┌──────────────────────────────────────────────┐
│                  巴法云 MQTT                    │
│              (bemfa.com:9501)                 │
└──────────────────────────────────────────────┘
                    ▲  Wi-Fi (MQTT 透传)
                    │
┌───────────────────┼───────────────────────────┐
│              STM32F407VET6 边缘网关            │
│  ┌────────────┐   ┌──────────┐  ┌──────────┐ │
│  │ 480×320    │   │ FreeRTOS │  │ W25Q64   │ │
│  │ LVGL 触控  │   │ 4 任务   │  │ 断网缓存 │ │
│  └────────────┘   └──────────┘  └──────────┘ │
│         │  MAX3485 (TTL ↔ RS485)              │
└─────────┼─────────────────────────────────────┘
          │ RS485 总线 (A/B)
          ▼
   ┌─────────────────┐
   │ 温湿度变送器     │
   │ 地址 01, 9600   │
   └─────────────────┘
```

数据流：`温湿度传感器 --Modbus--> 采集任务 --队列--> MQTT 任务 --Wi-Fi--> 巴法云`，同时采集数据直接推送 UI 实时显示。

---

## 软件设计

### FreeRTOS 任务

| 任务 | 优先级 | 职责 |
|:---|:---|:---|
| UITask | AboveNormal | LVGL 刷新 + 触摸响应 |
| SensorTask | High | Modbus 采集温湿度 |
| MQTTTask | Normal | 在线上报 / 离线缓存 + 补传 |
| NetworkTask | Normal | AT 初始化 + 心跳检测 |

### 关键实现

- **CRC16**：查表法 + 按位法双实现，查表法预计算 256 项高低字节表
- **Modbus 03 功能码**：手写问询帧、手动逐字节轮询接收、双超时保护
- **MQTT 报文**：手写 CONNECT（`MQIsdp` 协议名、Clean Session、Keep Alive 120s）、SUBSCRIBE、PUBLISH、PINGREQ
- **SPI Flash 环形缓存**：每条 128 字节（魔数 + 时间戳 + 长度 + JSON），魔数状态机标记已补传
- **断网检测**：AT 任务每 10 秒发 PINGREQ，连续失败判离线

---

## 快速开始

### 1. 硬件连接

按「引脚分配」表连接。注意：温湿度变送器需 **DC 8~28V 独立供电**，ESP-01S 需 3.3V 独立供电，所有设备 GND 共地。

### 2. 配置 WiFi 与巴法云

编辑 `ESP8266/pal.h`：

```c
#define WIFI_SSID   "你的WiFi名"
#define WIFI_PASS   "你的WiFi密码"
#define BEMFA_UID   "你的巴法云32位私钥"
#define TOPIC_PUB   "你的主题名"
```

### 3. 编译烧录

用 Keil MDK 打开 `MDK-ARM/F4project.uvprojx`，编译后通过 ST-Link 烧录。

### 4. 验证

上电后设备自动：连 WiFi → 连 TCP → MQTT 登录 → 订阅主题。屏幕显示实时温湿度，巴法云控制台可看到数据上报。拔掉 ESP-01S 再插回，可验证断网缓存与补传。

---

## 技术栈

`STM32F407` `FreeRTOS` `LVGL` `ILI9488(18bit RGB666)` `Modbus RTU` `CRC16` `MQTT 3.1.1` `ESP8266 AT 指令` `透传模式` `W25Q64 SPI Flash` `RS485` `MAX3485` `XPT2046` `DMA` `环形缓冲区` `断网续传` `魔数状态机`

---

## 目录结构

```
.
├── Core/                    # HAL 初始化（main / freertos / 外设）
├── Drivers/                 # STM32 HAL + CMSIS
├── Middlewares/             # FreeRTOS
├── code/                    # 应用层（LCD / 触摸 / 任务）
│   ├── lcd.c/h              # ILI9488 驱动（18bit RGB666）
│   ├── touch.c/h            # XPT2046 触摸
│   ├── Taskhandel.c/h       # 四任务业务逻辑
│   └── pin_config.h
├── model/                   # Modbus RTU 协议栈
├── ESP8266/                 # AT 框架 + MQTT 报文
├── SPI_Flash/               # W25Q64 驱动 + 环形缓存
├── Lvgl/                    # LVGL v8.3 + 中文界面
└── MDK-ARM/                 # Keil 工程
```

---

## 踩坑记录

| 问题 | 原因 | 解决 |
|:---|:---|:---|
| 屏幕纯白 | ILI9488 需 18bit RGB666（`0x3A=0x66`），用 16bit 会白屏 | 每像素写 3 字节（RGB666） |
| 屏幕白屏 | 初始化序列参数与屏幕不匹配 | 采用 MSP3520 官方数据手册序列（`0xF7`/`0xBE`/`0xE9`） |
| 温湿度一直 0 | 显示逻辑依赖联网状态，离线时不更新 | 采集任务直接写 UI 队列，本地实时显示 |
| Modbus 无响应 | 触摸检测的调试打印干扰了 RS485 总线数据 | 隔离调试输出与总线通信 |
| Modbus 偶发丢字节 | 接收循环 `vTaskDelay(1)` 在 9600 下丢字节 | 改 `taskYIELD()` + 提高任务优先级 |
| MQTT 一直离线 | `AT+CIPCLOSE` 关掉了刚建立的 TCP | 删除 `AT+CIPCLOSE` |

---

## License

本项目仅供学习交流使用。
