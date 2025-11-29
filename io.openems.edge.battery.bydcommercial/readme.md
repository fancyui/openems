= BYD Battery-Box Commercial

Implemented Natures::
- Battery

https://github.com/OpenEMS/openems/tree/develop/io.openems.edge.battery.bydcommercial[Source Code icon:github[]]

# OpenEMS Edge BYD Commercial Battery 分析

## 概述

`io.openems.edge.battery.bydcommercial` 是 OpenEMS 项目中专门针对 BYD (比亚迪) 商用电池箱 C130 系列的实现模块。该模块通过 Modbus 协议与 BYD 电池管理系统通信，并提供完整的电池监控、保护控制功能。

## 目录结构分析

```
io.openems.edge.battery.bydcommercial/
├── src/io/openems/edge/battery/bydcommercial/
│   ├── BydBatteryBoxCommercialC130.java          # BYD C130 电池接口定义
│   ├── BydBatteryBoxCommercialC130Impl.java   # BYD C130 电池主要实现类
│   ├── Config.java                              # OSGi 配置接口
│   ├── BatteryProtectionDefinitionBydC130.java # BYD C130 保护参数配置
│   ├── enums/                                  # 枚举定义
│   │   ├── BatteryWorkState.java               # 电池工作状态枚举
│   │   └── PowerCircuitControl.java           # 功率电路控制枚举
│   ├── statemachine/                            # 状态机实现
│   │   ├── Context.java                         # 状态机上下文
│   │   ├── ErrorHandler.java                  # 错误状态处理器
│   │   ├── GoRunningHandler.java              # 运行状态处理器
│   │   ├── GoStoppedHandler.java              # 停止状态处理器
│   │   ├── RunningHandler.java                # 运行状态处理器
│   │   ├── StoppedHandler.java                # 停止状态处理器
│   │   ├── UndefinedHandler.java              # 未定义状态处理器
│   │   └── StateMachine.java                  # 主状态机类
│   └── utils/                                  # 工具类
│       └── Constants.java                       # 常量定义
├── doc/                                       # 文档目录
│   └── 20200323（New）RS485 MODBUS based communication protocol between BMS and PCS V2.1_20190323_.pdf
└── test/                                      # 单元测试
    └── io/openems/edge/battery/bydcommercial/
```

## 核心组件功能分析

### 1. BydBatteryBoxCommercialC130.java - BYD C130 电池接口

**主要功能：**
- 定义 BYD C130 电池系统的专用数据通道
- 继承通用 Battery 接口
- 提供功率电路控制功能
- 定义大量电池模块数据通道（电压、温度等）

**关键特性：**
- **模块化设计**：支持最多 20 个电池模块
- **单体监控**：每个模块提供单体电压和温度数据
- **功率控制**：通过 `PowerCircuitControl` 控制主接触器
- **状态管理**：最大启动/停止次数控制
- **多层报警**：Level 1、2、3 三级报警系统

**专用数据通道：**
- `POWER_CIRCUIT_CONTROL`: 功率电路控制（预充电、充电、运行等状态）
- `CLUSTER_1_VOLTAGE`: 电池组 1 总电压
- `CLUSTER_1_CURRENT`: 电池组 1 电流
- `CLUSTER_1_SOH`: 电池组 1 健康状态
- `CLUSTER_1_MAX/MIN_CELL_VOLTAGE`: 最高/最低单体电压
- `CLUSTER_1_MAX/MIN_CELL_TEMPERATURE`: 最高/最低单体温度
- `CLUSTER_1_BATTERY_XXX_VOLTAGE`: 单体电池电压（XXX: 000-239）
- `CLUSTER_1_BATTERY_XX_TEMPERATURE`: 单体电池温度（XX: 00-47）

### 2. BydBatteryBoxCommercialC130Impl.java - 主实现类

**主要功能：**
- 实现 BYD C130 电池系统的完整 Modbus 通信
- 集成电池保护机制
- 管理状态机和错误处理
- 提供完整的电池监控数据

**Modbus 通信架构：**
- **读取任务（FC3）**：从 BMS 读取状态和数据
  - 0x2010: 功率电路控制状态
  - 0x2100-0x210C: 电压、电流、SOC、SOH、单体电压、单体温度
  - 0x2140-0x2147: 多级报警状态
  - 0x216C-0x216D: BMS 充放电电流限制
  - 0x2183-0x2185: 从模块通信和故障状态
  - 0x2800-0x28D6: 单体电池电压（240 个电池）
  - 0x2C00-0x2C2F: 单体电池温度（48 个温度传感器）

- **写入任务（FC6）**：向 BMS 发送控制命令
  - 0x2010: 功率电路控制命令

**电池保护集成：**
```java
// 初始化电池保护
this.batteryProtection = BatteryProtection.create(this)
    .applyBatteryProtectionDefinition(new BatteryProtectionDefinitionBydC130(), this.componentManager)
    .build();

// 在每个周期应用保护逻辑
this.batteryProtection.apply();
```

**状态机管理：**
```java
// 状态机状态流转
switch (state) {
    case UNDEFINED -> UndefinedHandler
    case GO_RUNNING -> GoRunningHandler
    case RUNNING -> RunningHandler
    case GO_STOPPED -> GoStoppedHandler
    case STOPPED -> StoppedHandler
    case ERROR -> ErrorHandler
}
```

### 3. BatteryProtectionDefinitionBydC130.java - 保护参数配置

**充电保护参数：**
- **初始充电电流**：80 A
- **充电电压保护曲线**：
  - 3000 mV: 10% 允许充电电流
  - 3000-3350 mV: 线性增长到 100%
  - 3350-3450 mV: 100% 允许充电电流
  - 3450-3600 mV: 线性下降到 2%
  - 3600-3650 mV: 2% 允许充电电流
  - >3650 mV: 0% 允许充电电流

- **充电温度保护曲线**：
  - 0-18°C: 0-1% 允许充电电流
  - 18-35°C: 100% 允许充电电流
  - 35-40°C: 线性下降到 1%
  - >40°C: 0% 允许充电电流

**放电保护参数：**
- **初始放电电流**：80 A
- **放电电压保护曲线**：
  - 2900 mV: 0% 允许放电电流
  - 2900-2920 mV: 增长到 1%
  - 2920-3000 mV: 线性增长到 100%
  - 3000-3700 mV: 100% 允许放电电流
  - >3700 mV: 线性下降到 0%

- **放电温度保护曲线**：
  - 0-12°C: 0-1% 允许放电电流
  - 12-45°C: 100% 允许放电电流
  - 45-55°C: 线性下降到 1%
  - >55°C: 0% 允许放电电流

**电流增长限制**：0.1 A/秒

### 4. 状态机系统

**状态定义：**
- `UNDEFINED` (-1): 未定义状态
- `GO_RUNNING` (10): 正在启动过程
- `RUNNING` (11): 正常运行状态
- `GO_STOPPED` (20): 正在停止过程
- `STOPPED` (21): 已停止状态
- `ERROR` (30): 错误状态

**状态处理器功能：**
- **UndefinedHandler**: 处理未知状态，根据当前条件确定下一步
- **GoRunningHandler**: 处理启动过程，控制接触器和充电状态
- **RunningHandler**: 处理正常运行，监控电池状态
- **GoStoppedHandler**: 处理停止过程，安全关闭电池系统
- **StoppedHandler**: 处理停止状态，等待启动命令
- **ErrorHandler**: 处理错误状态，记录错误并尝试恢复

### 5. 枚举类型

**BatteryWorkState（电池工作状态）：**
- `UNDEFINED` (-1): 未定义
- `STANDBY` (0): 待机
- `DISCHARGE` (1): 放电
- `CHARGE` (2): 充电

**PowerCircuitControl（功率电路控制）：**
- `UNDEFINED` (-1): 未定义
- `SWITCH_OFF` (0x0): 主功率接触器关闭，预充电关闭
- `PRE_CHARGING_1` (0x1): 中间状态：预充电第1阶段
- `PRE_CHARGING_2` (0x2): 中间状态：预充电第2阶段
- `SWITCH_ON` (0x3): 主功率接触器闭合，运行模式
- `PRE_CHARGE_FAIL` (0x4): 预充电失败 & 主功率接触器故障

### 6. 报警系统

**三级报警机制：**
1. **Level 1 (PRE_ALARM)**: 预警级别，轻微异常
2. **Level 2 (LEVEL1_ALARM)**: 一级报警，需要关注
3. **Level 3 (LEVEL2_ALARM)**: 二级报警，严重异常

**报警类型：**
- **电压相关**：单体电压高/低、电压不平衡、系统电压异常
- **温度相关**：充放电温度高/低、温差过大、电池组温度高
- **电流相关**：充放电电流过大
- **系统相关**：绝缘电阻低、SOC 异常、SOH 低
- **通信相关**：从模块通信故障
- **硬件相关**：接触器故障、熔断器故障、传感器故障
- **控制相关**：启动/停止失败、初始化失败

## 配置参数

**基本配置：**
- **组件ID**：唯一标识符，默认 "battery0"
- **别名**：用户友好的名称
- **启用状态**：是否启用该组件
- **启动/停止行为**：自动/强制启动/强制停止
- **Modbus 配置**：Modbus ID 和 Unit ID
- **从模块数量**：电池模块数量（1-20）

**容量计算：**
- **每模块容量**：6.9 Wh
- **系统总容量**：模块数量 × 6.9 Wh
- **电压范围**：34-42 V 每模块

## 通信协议

**Modbus 特性：**
- **通信协议**：RS485 Modbus
- **波特率**：57600 bps
- **数据格式**：符合 BYD BMS 通信协议 V2.1
- **地址映射**：详细的寄存器地址映射表
- **错误处理**：通信超时、重试机制

**协议版本兼容性：**
- 支持新旧硬件版本的自动检测
- 老版本使用默认电压参数
- 新版本读取动态配置参数

## 安全特性

### 1. 多层保护机制
- **硬件保护**：BMS 提供的硬件级别保护
- **软件保护**：OpenEMS 电池保护算法
- **状态机保护**：防止不安全的状态转换
- **参数保护**：电压、温度、SOC 多维度限制

### 2. 通信安全
- **超时检测**：通信超时自动处理
- **数据校验**：Modbus CRC 校验
- **重试机制**：失败自动重试，最多 30 次
- **连接监控**：实时监控通信状态

### 3. 故障处理
- **分级响应**：根据故障级别采取不同响应
- **自动恢复**：部分故障自动尝试恢复
- **安全关闭**：严重故障时安全关闭系统
- **故障记录**：详细记录故障信息和时间

## 使用场景

该模块主要用于：
- **工商业储能系统**：中大型储能电站
- **微电网应用**：社区微电网、离网系统
- **可再生能源配套**：太阳能、风能储能配套
- **电动汽车充电站**：快速充电站储能系统
- **UPS 系统**：不间断电源和应急供电

## 系统要求

**硬件要求：**
- BYD C130 商用电池箱
- RS485 Modbus 接口
- 支持的从模块数量：1-20 个

**软件要求：**
- OpenEMS 框架
- Java 运行环境
- OSGi 容器支持

**性能特性：**
- **高精度监控**：单体级电压和温度监控
- **实时响应**：毫秒级状态响应
- **可扩展性**：支持多种电池配置
- **可靠性**：多重故障检测和恢复机制

---

*此文档基于对 io.openems.edge.battery.bydcommercial 模块的源码分析生成，详细说明了 BYD C130 商用电池系统的功能、架构设计和使用方式。*