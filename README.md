# smartwatch
基于FreeRTOS的智能手表系统设计与实现
# 智能手表源程序 (Smart Watch Firmware)

基于 STM32F401RE + FreeRTOS + LVGL 的智能手表嵌入式固件项目

## 硬件平台

| 组件 | 型号 / 规格 |
|---|---|
| **主控 MCU** | STM32F401RETx (ARM Cortex-M4F, 84MHz) |
| **Flash** | 512 KB |
| **SRAM** | 96 KB |
| **显示屏** | GC9A01 圆形 TFT LCD (240×240, SPI 接口, 16bit 色彩) |
| **触摸屏** | CST816D 电容式触摸控制器 (I2C 接口) |
| **RTC** | MCU 内置 RTC (LSI 32kHz 时钟源) |
| **调试串口** | USART1 (9600bps) |
| **板载 LED** | PA5 |

## 软件架构

### 技术栈

- **RTOS**: FreeRTOS v10.3.1 (CMSIS-RTOS v2 封装)
- **GUI**: LVGL v8.3.11 (轻量级图形库)
- **HAL 库**: STM32F4 HAL Driver
- **工具链**: STM32CubeIDE (GCC) / Keil MDK-ARM (ARMCC)
- **仿真**: Proteus 设计套件

### FreeRTOS 任务列表

| 任务名 | 优先级 | 栈大小 | 功能描述 |
|---|---|---|---|
| `defaultTask` | Normal (24) | 512B | 每 500ms 通过 UART 发送字符 'A' |
| `taskLED0` | Low (8) | 512B | 每 500ms 翻转 PA5 LED 状态 |
| `homeTask` | Normal (24) | 2KB | 等待消息队列中的 RTC 时间，更新 LVGL 时钟标签 |
| `startTaskLvgl` | High (40) | 4KB | 每 10ms 执行 `lv_task_handler()` 刷新 GUI |

### IPC 机制

- **消息队列** `rtcTimerQueue`: 16 元素队列，传递 RTC 时间字符串到 GUI 任务
- **软件定时器** `rtcTimer`: 周期 500ms，读取 RTC 时间并发送到消息队列
- **互斥锁** `lvgl_mutex`: 保护 LVGL API 调用，防止并发访问

### FreeRTOS 配置要点

- 内核版本: v10.3.1
- 堆大小: 15KB (heap_4 算法)
- 系统滴答: 1000Hz
- 最大优先级: 56

### LVGL 配置要点

- 版本: 8.3.11
- 色彩深度: 16bit (RGB565)
- LVGL 堆大小: 10KB
- 启用的字体: Montserrat 8/10/12/14/16/18/20/22/24/26/28/30/32/34/36/38/40/42/44/46/48
- 已启用的部件: 基础对象、标签、按钮
- 已启用的额外部件: 动画、样式、布局

## 目录结构

```
f401LED-main/
├── README.md                        # 项目说明文档
├── f401LED/                         # STM32 固件主目录
│   ├── f401LED.ioc                  # STM32CubeMX 项目配置
│   ├── Core/
│   │   ├── Inc/                     # 头文件
│   │   │   ├── main.h               # 主程序头文件 (LED0 引脚定义等)
│   │   │   ├── FreeRTOSConfig.h     # FreeRTOS 配置文件
│   │   │   ├── cst816d.h            # CST816D 触摸驱动头文件
│   │   │   ├── lcd_gc9a01.h         # GC9A01 LCD 驱动头文件
│   │   │   ├── stm32f4xx_hal_conf.h # HAL 库配置
│   │   │   └── stm32f4xx_it.h       # 中断处理声明
│   │   ├── Src/                     # 源文件
│   │   │   ├── main.c               # ★ 主程序 (应用逻辑)
│   │   │   ├── cst816d.c            # CST816D 触摸驱动 (GPIO 模拟 I2C)
│   │   │   ├── lcd_gc9a01.c         # GC9A01 LCD 驱动 (SPI)
│   │   │   ├── freertos.c           # FreeRTOS 钩子函数
│   │   │   ├── stm32f4xx_it.c       # 中断服务例程
│   │   │   ├── stm32f4xx_hal_msp.c  # 外设引脚初始化
│   │   │   └── system_stm32f4xx.c   # 系统时钟初始化
│   │   └── Startup/                 # 启动文件
│   │       └── startup_stm32f401retx.s
│   ├── Drivers/                     # STM32 HAL 驱动库
│   │   ├── CMSIS/                   # ARM CMSIS 核心文件
│   │   └── STM32F4xx_HAL_Driver/    # HAL 驱动源码
│   ├── Middlewares/                 # 中间件
│   │   └── Third_Party/FreeRTOS/    # FreeRTOS v10.3.1 内核
│   ├── lvgl/                        # LVGL v8.3.11 图形库
│   ├── MDK-ARM/                     # Keil MDK 工程
│   │   ├── f401LED.uvprojx          # Keil uVision 项目文件
│   │   └── f401LED.uvoptx           # Keil 选项配置
│   ├── Debug/                       # STM32CubeIDE 构建输出
│   └── STM32F401RETX_FLASH.ld       # 链接脚本
└── protuesFile/                     # Proteus 仿真文件
    └── LED.pdsprj                   # 仿真项目
```

## 引脚连接

### GC9A01 LCD (SPI1)

| 引脚 | 功能 |
|---|---|
| PA6 (MISO) | SPI1 主入从出 |
| PA7 (MOSI) | SPI1 主出从入 |
| PB3 (SCK) | SPI1 时钟 |
| PB1 | LCD 复位 (RST) |
| PB2 | LCD 数据/命令 (DC) |
| PB4 | LCD 片选 (CS) |

### CST816D 触摸屏 (GPIO 模拟 I2C)

| 引脚 | 功能 |
|---|---|
| PB6 | I2C1 时钟 (SCL) |
| PB7 | I2C1 数据 (SDA) |

### 其他

| 引脚 | 功能 |
|---|---|
| PA5 | 板载 LED |
| PA9 (TX) / PA10 (RX) | USART1 调试串口 |

## 系统时钟

- **HSE**: 4MHz 外部晶振
- **PLL**: PLLM=2, PLLN=84, PLLP=/4 → SYSCLK = 84MHz
- **HCLK**: 84MHz
- **APB1**: 42MHz
- **APB2**: 42MHz
- **LSI**: 32kHz (RTC 时钟源)

## 构建与运行

### 环境准备

- **STM32CubeIDE** (推荐): 导入 `f401LED/` 目录作为现有项目
- **Keil MDK-ARM**: 打开 `f401LED/MDK-ARM/f401LED.uvprojx`
- **Proteus**: 打开 `protuesFile/LED.pdsprj` 进行仿真

### 编译（STM32CubeIDE）

```bash
cd f401LED/Debug
make -j
```

### 烧录

使用 ST-Link 或串口 ISP 将 `f401LED.hex` 烧录到 STM32F401RETx 目标板。

### 仿真

在 Proteus 中打开 `protuesFile/LED.pdsprj`，加载 `f401LED.hex` 即可运行仿真。

## 功能特性

- ✅ FreeRTOS 多任务实时调度
- ✅ LVGL 图形界面 (240×240 圆形屏)
- ✅ RTC 实时时钟显示 (通过消息队列传递时间)
- ✅ CST816D 电容触摸支持 (单击、双击、长按、滑动)
- ✅ UART 调试输出 (printf 重定向)
- ✅ LED 闪烁指示
- ✅ Proteus 仿真支持

## 许可证

本项目基于 STM32CubeMX 生成的项目框架，遵循 STMicroelectronics 相关许可条款。LVGL 和 FreeRTOS 分别遵循各自的许可协议。

## 参考资源

- [STM32F401RE 数据手册](https://www.st.com/resource/en/datasheet/stm32f401re.pdf)
- [FreeRTOS 官方文档](https://www.freertos.org/Documentation/RTOS_book.html)
- [LVGL 官方文档](https://docs.lvgl.io/8.3/)
- [GC9A01 LCD 驱动芯片手册](https://www.waveshare.com/wiki/1.28inch_Touch_LCD)
