# ML307R 4G模块完整故障排查记录

## 文档信息

- **创建日期**: 2025-11-13
- **项目**: aibox_s3 (立创开发板 - ESP32-S3)
- **问题**: ML307R 4G模块网络连接问题完整排查过程
- **状态**: 🔄 进行中

---

## 目录

1. [问题背景](#1-问题背景)
2. [第一阶段：复位时序问题](#2-第一阶段复位时序问题)
3. [第二阶段：网络注册失败](#3-第二阶段网络注册失败)
4. [第三阶段：SIM卡状态分析](#4-第三阶段sim卡状态分析)
5. [代码修改记录](#5-代码修改记录)
6. [成功日志记录](#6-成功日志记录)
7. [待解决问题](#7-待解决问题)
8. [经验总结](#8-经验总结)

---

## 1. 问题背景

### 1.1 项目概述

**项目名称**: aibox_s3  
**开发板**: 立创开发板（lichuang-dev）  
**芯片**: ESP32-S3  
**目标**: 添加ML307R 4G模块支持，实现WiFi和4G网络的智能切换

### 1.2 硬件配置

| 组件 | 型号/配置 | 引脚 |
|------|----------|------|
| 主控芯片 | ESP32-S3 | - |
| 4G模块 | ML307R-DL-MBRH0S01 | - |
| 复位引脚 | GPIO5 | 高电平复位，低电平正常工作 |
| UART TX | GPIO17 | ESP32 → ML307R RX |
| UART RX | GPIO18 | ESP32 ← ML307R TX |
| UART端口 | UART_NUM_1 | - |
| 波特率 | 921600 | 自动检测 |

### 1.3 初始需求

1. 在立创开发板项目中添加ML307R 4G模块支持
2. 实现WiFi和4G网络的智能切换功能
3. 硬件引脚配置变更（I2S MCLK从GPIO38改为GPIO20）
4. 保持433MHz UART功能正常（UART2，GPIO6/7）

---

## 2. 第一阶段：复位时序问题

### 2.1 问题现象

**日期**: 2025-11-12

**初始启动日志**（修复前）:
```
I (239) SmartDualNetworkBoard: ML307 reset pin (GPIO5) initialized to LOW (WiFi mode)
I (249) SmartDualNetworkBoard: 🌐 Initialize ML307 4G board
I (759) AtUart: Detecting baud rate...
I (1919) AtUart: Detecting baud rate...
I (3079) AtUart: Detecting baud rate...
I (4239) AtUart: Detecting baud rate...
...（无限重复）
```

**问题描述**:
- ML307R模块无法响应AT指令
- 波特率检测一直失败（每1160ms重复一次）
- 日志显示"WiFi mode"，但实际应该是ML307模式

### 2.2 示波器测量结果

| 测量点 | 观察结果 | 说明 |
|--------|---------|------|
| ML307R的RX引脚 | ✅ 有串口波形 | ESP32的GPIO17（TX）正在发送数据 |
| ML307R的TX引脚 | ❌ 一直是高电平 | ML307R没有发送任何数据 |
| GPIO5复位引脚 | 一直保持LOW | 没有观察到复位脉冲 |

**结论**: 
- ESP32的UART TX工作正常
- ML307R没有响应
- 复位引脚没有执行复位时序

### 2.3 根本原因分析

**代码逻辑顺序错误**:

**修复前的执行顺序**:
```cpp
SmartDualNetworkBoard::SmartDualNetworkBoard(...) {
    // 1. 初始化复位引脚（此时network_type_=WIFI，默认值）
    if (network_type_ == NetworkType::ML307) {
        // ❌ 不会执行，因为network_type_还是默认的WIFI
    } else {
        gpio_set_level(ml307_rst_pin_, 0);  // ✅ 执行这里
    }
    
    // 2. 加载网络类型（此时才知道是ML307）
    network_type_ = LoadNetworkTypeFromSettings(default_net_type);
    
    // 3. 初始化板卡（但复位时序已经错过了）
    InitializeCurrentBoard();
}
```

**问题**:
- 构造函数中，`network_type_`的默认值是`NetworkType::WIFI`
- 复位引脚初始化时，`network_type_`还是默认的`WIFI`
- 所以执行了`else`分支，将GPIO5直接设置为LOW
- 然后才调用`LoadNetworkTypeFromSettings()`加载实际的网络类型（ML307）
- 但此时复位时序已经错过了，ML307R未正常启动

### 2.4 修复方案

**调整构造函数的执行顺序**:

**修复后的代码**:
```cpp
SmartDualNetworkBoard::SmartDualNetworkBoard(gpio_num_t ml307_rst_pin, 
                                             gpio_num_t ml307_tx_pin, 
                                             gpio_num_t ml307_rx_pin, 
                                             gpio_num_t ml307_dtr_pin, 
                                             int32_t default_net_type) 
    : Board(), 
      ml307_rst_pin_(ml307_rst_pin),
      ml307_tx_pin_(ml307_tx_pin), 
      ml307_rx_pin_(ml307_rx_pin), 
      ml307_dtr_pin_(ml307_dtr_pin) {
    
    // 🎯 关键修复：先加载网络类型，再根据类型初始化复位引脚
    network_type_ = LoadNetworkTypeFromSettings(default_net_type);
    
    // 初始化ML307复位引脚
    if (ml307_rst_pin_ != GPIO_NUM_NC) {
        gpio_reset_pin(ml307_rst_pin_);
        gpio_set_direction(ml307_rst_pin_, GPIO_MODE_OUTPUT);

        // 🎯 根据加载的网络类型执行相应的复位时序
        if (network_type_ == NetworkType::ML307) {
            ESP_LOGI(TAG, "🔧 ML307R Hardware Configuration:");
            ESP_LOGI(TAG, "   - Reset Pin: GPIO%d", ml307_rst_pin_);
            ESP_LOGI(TAG, "   - UART TX Pin: GPIO%d (ESP32 → ML307R RX)", ml307_tx_pin_);
            ESP_LOGI(TAG, "   - UART RX Pin: GPIO%d (ESP32 ← ML307R TX)", ml307_rx_pin_);

            // 执行复位时序：先高电平复位100ms，再拉低恢复正常
            gpio_set_level(ml307_rst_pin_, 1);  // 高电平复位
            ESP_LOGI(TAG, "📌 ML307 reset pin (GPIO%d) set to HIGH (resetting module)", ml307_rst_pin_);
            vTaskDelay(pdMS_TO_TICKS(100));

            gpio_set_level(ml307_rst_pin_, 0);  // 低电平，模块正常工作
            ESP_LOGI(TAG, "📌 ML307 reset pin (GPIO%d) set to LOW (normal operation)", ml307_rst_pin_);

            // 等待模块启动（ML307R启动时间约3-5秒）
            ESP_LOGI(TAG, "⏳ Waiting for ML307R module to boot (5 seconds)...");
            vTaskDelay(pdMS_TO_TICKS(5000));
            ESP_LOGI(TAG, "✅ ML307R boot wait completed, starting UART detection...");
        } else {
            // WiFi模式下，保持复位引脚为低电平（正常工作状态）
            gpio_set_level(ml307_rst_pin_, 0);
            ESP_LOGI(TAG, "ML307 reset pin (GPIO%d) initialized to LOW (WiFi mode)", ml307_rst_pin_);
        }
    }
    
    // 初始化当前网络类型对应的板卡
    InitializeCurrentBoard();
    
    // 启动网络监测任务
    StartNetworkMonitor();
}
```

**修改的文件**:
- `main/boards/common/smart_dual_network_board.cc`

**关键改进点**:
1. ✅ 先加载网络类型：`network_type_ = LoadNetworkTypeFromSettings(default_net_type);`
2. ✅ 再根据类型执行复位：`if (network_type_ == NetworkType::ML307)`
3. ✅ 复位时序正确：GPIO5先拉高100ms → 拉低 → 等待5秒
4. ✅ 日志更详细：显示硬件配置、复位时序的每一步

### 2.5 修复后的成功日志

**日期**: 2025-11-12

```
I (244) Board: UUID=7cbe9889-2602-4ce2-9127-c57560300802 SKU=lichuang-dev
I (244) gpio: GPIO[5]| InputEn: 0| OutputEn: 0| OpenDrain: 0| Pullup: 1| Pulldown: 0| Intr:0
I (244) SmartDualNetworkBoard: 🔧 ML307R Hardware Configuration:
I (254) SmartDualNetworkBoard:    - Reset Pin: GPIO5
I (254) SmartDualNetworkBoard:    - UART TX Pin: GPIO17 (ESP32 → ML307R RX)
I (264) SmartDualNetworkBoard:    - UART RX Pin: GPIO18 (ESP32 ← ML307R TX)
I (264) SmartDualNetworkBoard: 📌 ML307 reset pin (GPIO5) set to HIGH (resetting module)
I (374) SmartDualNetworkBoard: 📌 ML307 reset pin (GPIO5) set to LOW (normal operation)
I (374) SmartDualNetworkBoard: ⏳ Waiting for ML307R module to boot (5 seconds)...
I (5374) SmartDualNetworkBoard: ✅ ML307R boot wait completed, starting UART detection...
I (5374) SmartDualNetworkBoard: 🌐 Initialize ML307 4G board
I (5874) uart: queue free spaces: 100
I (5874) AtUart: Detecting baud rate...
I (5904) AtUart: Detected baud rate: 921600
I (5904) AtModem: Detected modem: ML307R-DL-MBRH0S01
I (5904) AtModem: Waiting for network ready...
I (5914) Ml307Board: Network is ready
I (5914) Ml307AtModem: PDP Context 1 IP: 10.155.22.225
I (5924) Ml307Board: ML307 Revision: ML307R-DL-MBRH0S01
I (5924) Ml307Board: ML307 IMEI: 862041074232258
I (5924) Ml307Board: ML307 ICCID: 89860812192380763362
I (5934) Application: STATE: activating
I (6024) Ml307Http: HTTP connection created, ID: 0, protocol: http, host: www.bysxai.com
I (6694) Ml307Http: HTTP request successful, status code: 200
I (6714) Ml307Http: HTTP connection closed, ID: 0
```

**时序分析**:

| 时间点 | 事件 | 说明 |
|--------|------|------|
| 264ms | GPIO5 → HIGH | 开始复位 |
| 374ms | GPIO5 → LOW | 复位完成（持续110ms） |
| 5374ms | 等待完成 | 等待5秒启动 |
| 5874ms | 开始检测波特率 | - |
| 5904ms | ✅ 检测到波特率：921600 | 只用了30ms！ |
| 5914ms | ✅ 网络就绪 | 只用了10ms！ |
| 5914ms | ✅ 获得IP地址 | 10.155.22.225 |
| 6024ms | ✅ HTTP连接创建 | - |
| 6694ms | ✅ HTTP请求成功 | 状态码200 |

**结论**: 
- ✅ 复位时序正确执行
- ✅ ML307R成功启动
- ✅ 波特率检测成功
- ✅ 网络连接成功
- ✅ HTTP通信成功

---

## 3. 第二阶段：网络注册失败

### 3.1 问题现象

**日期**: 2025-11-13

**启动日志**:
```
I (239) SmartDualNetworkBoard: 🔧 ML307R Hardware Configuration:
I (249) SmartDualNetworkBoard:    - Reset Pin: GPIO5
I (249) SmartDualNetworkBoard:    - UART TX Pin: GPIO17 (ESP32 → ML307R RX)
I (259) SmartDualNetworkBoard:    - UART RX Pin: GPIO18 (ESP32 ← ML307R TX)
I (259) SmartDualNetworkBoard: 📌 ML307 reset pin (GPIO5) set to HIGH (resetting module)
I (369) SmartDualNetworkBoard: 📌 ML307 reset pin (GPIO5) set to LOW (normal operation)
I (369) SmartDualNetworkBoard: ⏳ Waiting for ML307R module to boot (5 seconds)...
I (5369) SmartDualNetworkBoard: ✅ ML307R boot wait completed, starting UART detection...
I (5369) SmartDualNetworkBoard: 🌐 Initialize ML307 4G board
I (5869) AtUart: Detecting baud rate...
I (5899) AtUart: Detected baud rate: 921600
I (5899) AtModem: Detected modem: ML307R-DL-MBRH0S01
I (5899) AtModem: Waiting for network ready...
（阻塞在这里，没有后续日志）
I (15769) AudioCodec: Set input enable to false
W (15779) SystemInfo: free sram: 111647 minimal sram: 99411
I (24769) AudioCodec: Set output enable to false
W (25779) SystemInfo: free sram: 111647 minimal sram: 99411
```

**问题描述**:
- ✅ 复位时序正确执行
- ✅ 波特率检测成功（921600）
- ✅ 模块识别成功（ML307R-DL-MBRH0S01）
- ❌ **网络注册失败**（阻塞在"Waiting for network ready..."）
- ⏳ 已等待约20秒，仍未就绪

### 3.2 AT指令手动测试（第一轮）

**测试工具**: 串口助手  
**波特率**: 921600

**测试结果**:

```
[11:04:23.844] AT+CPIN?
+CPIN: READY
OK

[11:04:30.018] （自动上报）
+CEREG: 0

[11:04:36.604] AT+CSQ
+CSQ: 22,99
OK

[11:04:52.667] AT+CREG?
ERROR

[11:05:39.924] AT+CEREG?
+CEREG: 2,0
OK

[11:05:52.459] AT+CGATT?
+CGATT: 0
OK

[11:06:03.724] AT+CGPADDR
OK

[11:06:33.803] AT+CGPADDR=1
OK

[11:08:52.908] AT+MIPCALL?
[11:09:16.837] AT+MIPCALL?
OK
```

**分析结果**:

| AT指令 | 响应 | 状态 | 说明 |
|--------|------|------|------|
| `AT+CPIN?` | `+CPIN: READY` | ✅ 正常 | SIM卡已就绪 |
| `AT+CSQ` | `+CSQ: 22,99` | ✅ 正常 | 信号强度-69dBm（中等偏好） |
| `AT+CREG?` | `ERROR` | ❌ 不支持 | ML307R不支持2G/3G查询 |
| `AT+CEREG?` | `+CEREG: 2,0` | ❌ **未注册** | **未注册，未搜索** |
| `AT+CGATT?` | `+CGATT: 0` | ❌ 未附着 | GPRS未附着 |
| `AT+CGPADDR` | `(空)` | ❌ 无IP | 未分配IP地址 |
| `AT+MIPCALL?` | `OK` | ❌ 无数据 | 无IP连接信息 |

**关键发现**:
- ✅ SIM卡正常
- ✅ 信号强度正常
- ❌ **网络未注册**（`+CEREG: 2,0`）
- ❌ **模块未搜索网络**（`<stat>=0`）

### 3.3 AT指令手动测试（第二轮）

**测试目的**: 诊断网络注册失败原因

**测试结果**:

```
[11:14:46.329] AT+CFUN?
+CFUN: 1
OK

[11:15:14.844] AT+CNMP?
ERROR

[11:15:40.380] AT+CNMP=38
ERROR

[11:16:21.419] AT+CEREG=2
OK

[11:16:41.947] AT+COPS=0
OK

[11:17:03.691] AT+COPS?
+COPS: 0
OK

[11:17:27.579] AT+COPS=?
[11:17:31.974] （自动上报）
+CEREG: 0

[11:18:06.955] （搜索结果）
+COPS: (1,"CHINA MOBILE","CMCC","46000",7),
       (3,"","","46015",7),
       (3,"CHN-CT","CT","46011",7),
       (3,"CHN-UNICOM","UNICOM","46001",7),
       ,(0,1,2,3,4),(0,1,2)
OK
```

**分析结果**:

| AT指令 | 响应 | 状态 | 说明 |
|--------|------|------|------|
| `AT+CFUN?` | `+CFUN: 1` | ✅ 正常 | 全功能模式（非飞行模式） |
| `AT+CNMP?` | `ERROR` | ⚠️ 不支持 | ML307R不支持此指令 |
| `AT+CEREG=2` | `OK` | ✅ 成功 | 启用URC上报 |
| `AT+COPS=0` | `OK` | ✅ 成功 | 触发自动搜索 |
| `AT+COPS?` | `+COPS: 0` | ⚠️ 未注册 | 自动模式，但未注册 |
| `AT+COPS=?` | 搜索到4个运营商 | ✅ 成功 | 能搜索到网络 |
| URC上报 | `+CEREG: 0` | ❌ **未注册** | **网络注册失败** |

**运营商搜索结果分析**:

| 序号 | stat | 运营商名称 | 数字ID | AcT | 说明 |
|------|------|-----------|--------|-----|------|
| 1 | **1** | **CHINA MOBILE** | **46000** | **7** | **当前可用** ✅ |
| 2 | 3 | (空) | 46015 | 7 | 禁止 ❌ |
| 3 | 3 | CHN-CT | 46011 | 7 | 禁止 ❌ |
| 4 | 3 | CHN-UNICOM | 46001 | 7 | 禁止 ❌ |

**关键矛盾**:
- ✅ 能搜索到中国移动4G网络（46000）
- ✅ 中国移动网络状态为"可用"（stat=1）
- ❌ **但无法注册到该网络**

### 3.4 SIM卡状态测试

**测试方法**: 将SIM卡从ML307R中取出，插入手机

**测试结果**:
- ❌ **手机没有流量且也无法上网**

**AT指令测试**:
```
[11:32:46.690] AT+CGDCONT?
OK

[11:33:13.195] AT+CEREG=2
OK

[11:34:51.238] AT+COPS=0
OK
```

**分析**:
- `AT+CGDCONT?` 响应为空，未配置PDP上下文
- `AT+CEREG=2` 和 `AT+COPS=0` 执行成功
- 但SIM卡插入手机后无法上网

**结论**: 
- ❌ **SIM卡本身无法提供数据服务**
- 这是根本原因

---

## 4. 第三阶段：SIM卡状态分析

### 4.1 关键信息确认

**用户确认**: 之前成功的SIM卡和现在使用的是**同一张SIM卡**

**之前成功的日志**（参考）:
```
I (5924) Ml307Board: ML307 ICCID: 89860812192380763362
```

**核心矛盾**:
- ✅ 之前能正常上网（IP: 10.155.22.225）
- ❌ 现在插入手机无法上网
- ❌ 现在ML307R无法注册网络
- ✅ 确认是同一张SIM卡

**结论**: SIM卡状态发生了变化（从"正常"变为"异常"）

### 4.2 SIM卡状态变化的可能原因

#### **原因1: 物联网卡被锁定（插入手机）** ⭐⭐⭐⭐⭐

**ICCID分析**: `89860812192380763362`
- `8986`: 中国
- `08`: 中国移动
- `12`: 可能是物联网卡标识

**物联网卡的特征**:
- ✅ 在物联网设备（ML307R）中能正常使用
- ❌ 插入手机后无法上网（被运营商检测并锁定）
- ✅ 分配的是内网IP（10.155.22.225）
- ✅ 用于M2M（Machine to Machine）通信

**运营商检测机制**:
1. **IMEI检测** - 手机的IMEI通常以`35`开头，物联网设备以`86`开头
2. **TAC检测** - 通过IMEI前8位识别设备型号
3. **网络行为检测** - 手机会发送大量信令，物联网设备只有数据传输
4. **APN检测** - 手机使用`CMNET`，物联网卡使用`CMIOT`

**如果检测到物联网卡插入手机**:
- 运营商会自动锁定数据服务
- SIM卡状态变为"异常"
- 需要联系客服解锁

#### **原因2: SIM卡欠费或套餐到期** ⭐⭐⭐⭐⭐

**可能的情况**:
1. 预付费SIM卡余额用完
2. 月租套餐到期
3. 流量包到期
4. 超出流量限额

#### **原因3: 运营商临时锁定（异常行为）** ⭐⭐⭐⭐

**可能的情况**:
1. 频繁换设备（SIM卡在ML307R和手机之间频繁切换）
2. 短时间内大量数据传输
3. 异常的网络行为

#### **原因4: 测试期到期** ⭐⭐⭐

**可能的情况**:
1. 免费测试期（如7天、30天）到期
2. 赠送的流量包用完

### 4.3 验证方法

**步骤1**: 查询IMSI，确认是否是物联网卡
```
AT+CIMI
```
- 如果以`46004`开头 → 物联网卡
- 如果以`46000`、`46002`、`46007`开头 → 普通手机卡

**步骤2**: 查询ICCID，确认是同一张卡
```
AT+CCID
```
- 对比之前成功的ICCID: `89860812192380763362`

**步骤3**: 拨打运营商客服（10086）
- 查询SIM卡状态
- 确认是否是物联网卡
- 确认是否因插入手机被锁定
- 确认余额和套餐状态

**步骤4**: 根据客服反馈采取行动
- 如果是物联网卡被锁定 → 请求解锁
- 如果是欠费 → 充值
- 如果是其他原因 → 按客服指引操作

---

## 5. 代码修改记录

### 5.1 已完成的修改

#### **修改1: 硬件引脚配置**

**文件**: `main/boards/lichuang-dev/config.h`

**修改内容**:
```c
// I2S MCLK引脚变更
#define I2S_MCLK_PIN               GPIO_NUM_20  // 从GPIO38改为GPIO20

// 添加ML307R 4G模块配置
#define ML307_ENABLE 1
#define ML307_RST_PIN              GPIO_NUM_5
#define ML307_TX_PIN               GPIO_NUM_17  // ESP32 TX -> ML307 RX
#define ML307_RX_PIN               GPIO_NUM_18  // ESP32 RX <- ML307 TX
#define ML307_DTR_PIN              GPIO_NUM_NC
```

**状态**: ✅ 已完成

---

#### **修改2: 创建智能双网络板卡类**

**文件**: `main/boards/common/smart_dual_network_board.h`（新创建）

**主要内容**:
```cpp
class SmartDualNetworkBoard : public Board {
private:
    std::unique_ptr<Board> current_board_;
    NetworkType network_type_ = NetworkType::WIFI;
    
    gpio_num_t ml307_rst_pin_;
    gpio_num_t ml307_tx_pin_;
    gpio_num_t ml307_rx_pin_;
    gpio_num_t ml307_dtr_pin_;
    
public:
    SmartDualNetworkBoard(gpio_num_t ml307_rst_pin, 
                         gpio_num_t ml307_tx_pin, 
                         gpio_num_t ml307_rx_pin, 
                         gpio_num_t ml307_dtr_pin, 
                         int32_t default_net_type);
    
    void ResetWifiConfiguration();
    // ... 其他方法
};
```

**状态**: ✅ 已完成

---

#### **修改3: 修复复位时序问题**

**文件**: `main/boards/common/smart_dual_network_board.cc`

**关键修改**: 调整构造函数的执行顺序

**修改前**:
```cpp
SmartDualNetworkBoard::SmartDualNetworkBoard(...) {
    // 1. 初始化复位引脚（此时network_type_=WIFI）
    if (network_type_ == NetworkType::ML307) {
        // ❌ 不会执行
    } else {
        gpio_set_level(ml307_rst_pin_, 0);  // ✅ 执行这里
    }
    
    // 2. 加载网络类型
    network_type_ = LoadNetworkTypeFromSettings(default_net_type);
    
    // 3. 初始化板卡
    InitializeCurrentBoard();
}
```

**修改后**:
```cpp
SmartDualNetworkBoard::SmartDualNetworkBoard(...) {
    // 🎯 1. 先加载网络类型
    network_type_ = LoadNetworkTypeFromSettings(default_net_type);
    
    // 🎯 2. 根据网络类型初始化复位引脚
    if (network_type_ == NetworkType::ML307) {
        // 执行复位时序
        gpio_set_level(ml307_rst_pin_, 1);  // 高电平复位
        vTaskDelay(pdMS_TO_TICKS(100));
        gpio_set_level(ml307_rst_pin_, 0);  // 低电平正常工作
        vTaskDelay(pdMS_TO_TICKS(5000));    // 等待5秒启动
    } else {
        gpio_set_level(ml307_rst_pin_, 0);
    }
    
    // 🎯 3. 初始化板卡
    InitializeCurrentBoard();
}
```

**状态**: ✅ 已完成

---

#### **修改4: 立创开发板主文件**

**文件**: `main/boards/lichuang-dev/lichuang_dev_board.cc`

**修改内容**:
- 从继承`WifiBoard`改为继承`SmartDualNetworkBoard`
- 移除433MHz UART重复初始化

**修改前**:
```cpp
class LichuangDevBoard : public WifiBoard {
    // ...
};
```

**修改后**:
```cpp
class LichuangDevBoard : public SmartDualNetworkBoard {
public:
    LichuangDevBoard() : SmartDualNetworkBoard(ML307_RST_PIN, ML307_TX_PIN, ML307_RX_PIN, ML307_DTR_PIN, 0),
                        boot_button_(BOOT_BUTTON_GPIO),
                        power_button_(POWER_BUTTON_GPIO, true, 3000, 50, false) {
        InitializeButtons();
        GetBacklight()->RestoreBrightness();
    }
};
```

**状态**: ✅ 已完成

---

### 5.2 待修改的代码问题

#### **问题1: 使用AT+CREG?而不是AT+CEREG?** ❌

**文件**: `managed_components/78__esp-ml307/src/at_modem.cc`

**当前问题**:
```cpp
NetworkStatus AtModem::WaitForNetworkReady(int timeout_ms) {
    at_uart_->SendCommand("AT+CREG?");  // ❌ 查询2G/3G网络
    std::string response = at_uart_->GetResponse();
    
    // ML307R不支持AT+CREG?，返回ERROR
    // 导致解析失败，返回Disconnected
    
    return NetworkStatus::Disconnected;
}
```

**需要修改**:
- 改为使用`AT+CEREG?`查询4G网络
- 或者在`Ml307AtModem`中重写该方法

**状态**: ⏸️ 待SIM卡问题解决后修改

---

#### **问题2: 缺少初始化AT指令** ❌

**文件**: `managed_components/78__esp-ml307/src/ml307/ml307_at_modem.cc`

**当前问题**:
- 没有执行`AT+CEREG=2`（启用URC上报）
- 没有执行`AT+COPS=0`（触发网络搜索）
- 没有配置APN（`AT+CGDCONT=1,"IP","CMNET"`或`"CMIOT"`）

**需要添加**:
```cpp
Ml307AtModem::Ml307AtModem(AtUart* uart) : AtModem(uart) {
    // 初始化网络配置
    at_uart_->SendCommand("AT+CEREG=2");  // 启用URC上报
    at_uart_->SendCommand("AT+COPS=0");   // 触发网络搜索
    
    // 配置APN（根据SIM卡类型）
    at_uart_->SendCommand("AT+CGDCONT=1,\"IP\",\"CMNET\"");  // 普通卡
    // 或
    at_uart_->SendCommand("AT+CGDCONT=1,\"IP\",\"CMIOT\"");  // 物联网卡
}
```

**状态**: ⏸️ 待SIM卡问题解决后修改

---

#### **问题3: 无超时和回退机制** ❌

**文件**: `main/boards/common/ml307_board.cc`

**当前问题**:
```cpp
void Ml307Board::StartNetwork() {
    // 1. 检测模块
    modem_ = AtModem::Detect(...);
    
    // 2. 等待网络就绪
    while (true) {
        auto status = modem_->WaitForNetworkReady(1000);
        if (status == NetworkStatus::Ready) {
            break;
        }
        // ❌ 如果一直未就绪，会永久阻塞
    }
}
```

**需要添加**:
```cpp
void Ml307Board::StartNetwork() {
    // 1. 检测模块
    modem_ = AtModem::Detect(...);
    
    // 2. 等待网络就绪（添加超时）
    int retry_count = 0;
    const int max_retries = 60;  // 60秒超时
    
    while (retry_count < max_retries) {
        auto status = modem_->WaitForNetworkReady(1000);
        if (status == NetworkStatus::Ready) {
            break;
        }
        retry_count++;
    }
    
    if (retry_count >= max_retries) {
        ESP_LOGE(TAG, "Network registration timeout");
        // 回退到WiFi或显示错误
    }
}
```

**状态**: ⏸️ 待SIM卡问题解决后修改

---

## 6. 成功日志记录

### 6.1 完整的成功启动日志

**日期**: 2025-11-12（复位时序修复后）

```
ESP-ROM:esp32s3-20210327
Build:Mar 27 2021
rst:0x1 (POWERON),boot:0xb (SPI_FAST_FLASH_BOOT)
SPIWP:0xee
mode:DIO, clock div:1
load:0x3fce2820,len:0x56c
load:0x403c8700,len:0x4
load:0x403c8704,len:0xc2c
load:0x403cb700,len:0x2e58
entry 0x403c890c

I (36) octal_psram: vendor id    : 0x0d (AP)
I (36) octal_psram: dev id       : 0x02 (generation 3)
I (36) octal_psram: density      : 0x03 (64 Mbit)
I (38) octal_psram: good-die     : 0x01 (Pass)
I (42) octal_psram: Latency      : 0x01 (Fixed)
I (46) octal_psram: VCC          : 0x01 (3V)
I (50) octal_psram: SRF          : 0x01 (Fast Refresh)
I (55) octal_psram: BurstType    : 0x01 (Hybrid Wrap)
I (60) octal_psram: BurstLen     : 0x01 (32 Byte)
I (64) octal_psram: Readlatency  : 0x02 (10 cycles@Fixed)
I (69) octal_psram: DriveStrength: 0x00 (1/1)
I (74) MSPI Timing: PSRAM timing tuning index: 5
I (78) esp_psram: Found 8MB PSRAM device
I (81) esp_psram: Speed: 80MHz

I (99) app_init: Project name:     xiaozhi
I (102) app_init: App version:      2.0.8
I (106) app_init: Compile time:     Nov 12 2025 15:30:45
I (111) app_init: ELF file SHA256:  6f9ab7e89...
I (116) app_init: ESP-IDF:          v5.4.1

I (244) Board: UUID=7cbe9889-2602-4ce2-9127-c57560300802 SKU=lichuang-dev
I (244) gpio: GPIO[5]| InputEn: 0| OutputEn: 0| OpenDrain: 0| Pullup: 1| Pulldown: 0| Intr:0
I (244) SmartDualNetworkBoard: 🔧 ML307R Hardware Configuration:
I (254) SmartDualNetworkBoard:    - Reset Pin: GPIO5
I (254) SmartDualNetworkBoard:    - UART TX Pin: GPIO17 (ESP32 → ML307R RX)
I (264) SmartDualNetworkBoard:    - UART RX Pin: GPIO18 (ESP32 ← ML307R TX)
I (264) SmartDualNetworkBoard: 📌 ML307 reset pin (GPIO5) set to HIGH (resetting module)
I (374) SmartDualNetworkBoard: 📌 ML307 reset pin (GPIO5) set to LOW (normal operation)
I (374) SmartDualNetworkBoard: ⏳ Waiting for ML307R module to boot (5 seconds)...
I (5374) SmartDualNetworkBoard: ✅ ML307R boot wait completed, starting UART detection...
I (5374) SmartDualNetworkBoard: 🌐 Initialize ML307 4G board

I (5874) uart: queue free spaces: 100
I (5874) AtUart: Detecting baud rate...
I (5904) AtUart: Detected baud rate: 921600
I (5904) AtModem: Detected modem: ML307R-DL-MBRH0S01
I (5904) AtModem: Waiting for network ready...
I (5914) Ml307Board: Network is ready
I (5914) Ml307AtModem: PDP Context 1 IP: 10.155.22.225
I (5924) Ml307Board: ML307 Revision: ML307R-DL-MBRH0S01
I (5924) Ml307Board: ML307 IMEI: 862041074232258
I (5924) Ml307Board: ML307 ICCID: 89860812192380763362

I (5934) Application: STATE: activating
I (6024) Ml307Http: HTTP connection created, ID: 0, protocol: http, host: www.bysxai.com
I (6694) Ml307Http: HTTP request successful, status code: 200
I (6714) Ml307Http: HTTP connection closed, ID: 0
```

### 6.2 关键时间节点分析

| 时间点 | 事件 | 耗时 | 说明 |
|--------|------|------|------|
| 0ms | 系统启动 | - | - |
| 244ms | 板卡初始化 | 244ms | - |
| 264ms | GPIO5 → HIGH | - | 开始复位 |
| 374ms | GPIO5 → LOW | 110ms | 复位完成 |
| 5374ms | 等待完成 | 5000ms | 等待ML307R启动 |
| 5874ms | 开始检测波特率 | - | - |
| 5904ms | ✅ 检测到波特率 | 30ms | 921600 |
| 5904ms | ✅ 识别模块 | 0ms | ML307R-DL-MBRH0S01 |
| 5914ms | ✅ 网络就绪 | 10ms | 极快！ |
| 5914ms | ✅ 获得IP | 0ms | 10.155.22.225 |
| 5924ms | ✅ 读取模块信息 | 10ms | IMEI、ICCID |
| 6024ms | ✅ HTTP连接创建 | 100ms | - |
| 6694ms | ✅ HTTP请求成功 | 670ms | 状态码200 |

**总结**:
- 从复位开始到网络就绪：5650ms（5.65秒）
- 波特率检测：30ms
- 网络注册：10ms（极快，说明之前已注册过）
- HTTP请求：670ms

### 6.3 ML307R模块信息

| 信息项 | 值 | 说明 |
|--------|-----|------|
| 模块型号 | ML307R-DL-MBRH0S01 | 中移物联ML307R模块 |
| 波特率 | 921600 | 高速波特率 |
| IMEI | 862041074232258 | 设备唯一标识 |
| ICCID | 89860812192380763362 | SIM卡号（可能是物联网卡） |
| IP地址 | 10.155.22.225 | 内网IP（运营商分配） |
| 网络类型 | LTE/4G | - |
| 运营商 | 中国移动 | - |

---

## 7. 待解决问题

### 7.1 当前主要问题：SIM卡状态异常

**问题描述**:
- 同一张SIM卡，之前能正常上网，现在无法上网
- 插入手机后无流量，无法上网
- ML307R无法注册网络

**最可能的原因**:
1. **⭐⭐⭐⭐⭐ 物联网卡被锁定**（插入手机后被运营商检测并锁定）
2. **⭐⭐⭐⭐⭐ SIM卡欠费或套餐到期**
3. **⭐⭐⭐⭐ 运营商临时锁定**（异常行为）

**下一步行动**:
1. 查询IMSI（`AT+CIMI`），确认是否是物联网卡
2. 查询ICCID（`AT+CCID`），确认是同一张卡
3. 拨打运营商客服（10086）查询SIM卡状态
4. 根据客服反馈采取行动（解锁/充值/更换SIM卡）

**状态**: ⏸️ 等待用户执行

---

### 7.2 次要问题：代码逻辑缺陷

**问题1**: 使用`AT+CREG?`而不是`AT+CEREG?`
- ML307R不支持`AT+CREG?`
- 导致网络检测失败

**问题2**: 缺少初始化AT指令
- 没有执行`AT+CEREG=2`、`AT+COPS=0`
- 没有配置APN

**问题3**: 无超时和回退机制
- 网络注册失败时会永久阻塞
- 无法回退到WiFi模式

**状态**: ⏸️ 待SIM卡问题解决后修改

---

## 8. 经验总结

### 8.1 技术要点

#### **1. ML307R复位时序**

**关键点**:
- 复位引脚逻辑：高电平复位，低电平正常工作
- 复位脉冲宽度：≥100ms
- 复位后启动时间：3~5秒
- 必须等待模块完全启动后才能发送AT指令

**代码实现**:
```cpp
gpio_set_level(ml307_rst_pin_, 1);  // 高电平复位
vTaskDelay(pdMS_TO_TICKS(100));
gpio_set_level(ml307_rst_pin_, 0);  // 低电平正常工作
vTaskDelay(pdMS_TO_TICKS(5000));    // 等待5秒启动
```

---

#### **2. 波特率自动检测**

**检测方法**:
- 依次尝试8种常见波特率：115200, 921600, 460800, 230400, 57600, 38400, 19200, 9600
- 每种波特率发送`AT\r\n`，等待20ms
- 如果收到`OK\r\n`，检测成功
- 所有波特率失败后，等待1秒重试

**ML307R的默认波特率**: 921600（高速模式）

---

#### **3. 网络注册流程**

**正常流程**:
```
1. SIM卡检测（AT+CPIN?）
2. 启用URC上报（AT+CEREG=2）
3. 触发网络搜索（AT+COPS=0）
4. 等待网络注册（+CEREG: 2,1）
5. 附着到GPRS（AT+CGATT=1）
6. 配置APN（AT+CGDCONT=1,"IP","CMNET"）
7. 激活PDP上下文（AT+MIPCALL=1）
8. 获得IP地址（AT+MIPCALL?）
```

**关键AT指令**:
- `AT+CEREG?` - 查询4G网络注册状态（不是`AT+CREG?`）
- `AT+COPS=0` - 自动搜索并注册网络
- `AT+CGDCONT=1,"IP","CMNET"` - 配置APN（中国移动）

---

#### **4. 物联网卡的特殊性**

**特征**:
- ICCID通常以`8986 08`开头（中国移动物联网卡）
- IMSI通常以`46004`开头
- 分配的是内网IP（10.x.x.x）
- 禁止插入手机使用

**运营商检测机制**:
- IMEI检测（手机以`35`开头，物联网设备以`86`开头）
- TAC检测（IMEI前8位）
- 网络行为检测
- APN检测

**如果插入手机**:
- 运营商会自动锁定数据服务
- 需要联系客服解锁
- 解锁后不要再插入手机

---

### 8.2 调试方法

#### **1. 硬件调试**

**示波器测量**:
- 复位引脚波形（应该有100ms的高电平脉冲）
- TX引脚波形（ESP32发送AT指令）
- RX引脚波形（ML307R响应）

**万用表测量**:
- 电源电压（3.3V~4.2V）
- 复位引脚电平

---

#### **2. 软件调试**

**日志分析**:
- 查看复位时序日志
- 查看波特率检测日志
- 查看网络注册日志
- 查看时间戳，分析时序

**AT指令手动测试**:
- 使用串口助手连接ML307R
- 手动发送AT指令
- 观察响应
- 定位问题

---

#### **3. 分层分析**

**硬件层**:
- 引脚连接
- 电源电压
- 复位逻辑

**驱动层**:
- UART配置
- 波特率检测
- AT指令通信

**应用层**:
- 初始化顺序
- 逻辑判断
- 错误处理

---

### 8.3 常见问题

#### **问题1: 波特率检测一直失败**

**可能原因**:
- 复位时序未执行
- 模块未启动
- TX/RX引脚接反
- 波特率不在检测列表中

**解决方法**:
- 检查复位时序日志
- 用示波器验证复位波形
- 检查引脚连接
- 等待足够的启动时间（5秒）

---

#### **问题2: 网络注册失败**

**可能原因**:
- SIM卡未激活/欠费/被锁定
- 信号太弱
- 天线未连接
- 代码使用了错误的AT指令（`AT+CREG?`而不是`AT+CEREG?`）

**解决方法**:
- 检查SIM卡状态（插入手机测试）
- 检查信号强度（`AT+CSQ`）
- 检查天线连接
- 手动测试AT指令

---

#### **问题3: 代码永久阻塞**

**可能原因**:
- 网络注册失败
- 无超时机制
- 无限循环等待

**解决方法**:
- 添加超时机制
- 添加重试次数限制
- 添加回退逻辑（回退到WiFi）

---

### 8.4 最佳实践

#### **1. 初始化顺序**

**正确的顺序**:
```
1. 加载配置（网络类型、引脚等）
2. 根据配置初始化硬件（复位引脚、UART等）
3. 执行复位时序
4. 等待模块启动
5. 检测模块
6. 初始化网络配置（AT+CEREG=2、AT+COPS=0等）
7. 等待网络就绪
```

**错误的顺序**:
```
1. 初始化硬件（此时配置还是默认值）❌
2. 加载配置（但硬件已经初始化了）❌
3. 初始化网络（但复位时序已经错过了）❌
```

---

#### **2. 错误处理**

**必须添加**:
- 超时机制（如60秒）
- 重试次数限制（如3次）
- 回退逻辑（如回退到WiFi）
- 错误日志（详细记录失败原因）

**不要**:
- 无限循环等待
- 忽略错误
- 没有日志

---

#### **3. 日志规范**

**好的日志**:
```
I (264) SmartDualNetworkBoard: 📌 ML307 reset pin (GPIO5) set to HIGH (resetting module)
I (374) SmartDualNetworkBoard: 📌 ML307 reset pin (GPIO5) set to LOW (normal operation)
I (5904) AtUart: Detected baud rate: 921600
I (5914) Ml307Board: Network is ready
```

**特点**:
- 有时间戳
- 有模块名称
- 有详细的状态信息
- 有emoji标记（可选，但更易读）

---

### 8.5 文档记录

**重要性**:
- 记录问题现象
- 记录分析过程
- 记录解决方案
- 记录成功日志
- 记录经验教训

**本文档就是一个很好的例子！**

---

## 附录

### A. 相关文件列表

| 文件路径 | 说明 | 状态 |
|---------|------|------|
| `main/boards/lichuang-dev/config.h` | 硬件引脚配置 | ✅ 已修改 |
| `main/boards/common/smart_dual_network_board.h` | 智能双网络板卡头文件 | ✅ 已创建 |
| `main/boards/common/smart_dual_network_board.cc` | 智能双网络板卡实现 | ✅ 已修改 |
| `main/boards/lichuang-dev/lichuang_dev_board.cc` | 立创开发板主文件 | ✅ 已修改 |
| `managed_components/78__esp-ml307/src/at_uart.cc` | AT UART通信层 | ⏸️ 待修改 |
| `managed_components/78__esp-ml307/src/at_modem.cc` | AT Modem基础功能 | ⏸️ 待修改 |
| `managed_components/78__esp-ml307/src/ml307/ml307_at_modem.cc` | ML307R特定功能 | ⏸️ 待修改 |
| `main/boards/common/ml307_board.cc` | ML307板卡实现 | ⏸️ 待修改 |

### B. AT指令参考

| AT指令 | 功能 | 响应示例 |
|--------|------|---------|
| `AT` | 测试通信 | `OK` |
| `AT+CPIN?` | 查询SIM卡状态 | `+CPIN: READY` |
| `AT+CSQ` | 查询信号强度 | `+CSQ: 22,99` |
| `AT+CIMI` | 查询IMSI | `460048234567890` |
| `AT+CCID` | 查询ICCID | `89860812192380763362` |
| `AT+CFUN?` | 查询功能模式 | `+CFUN: 1` |
| `AT+CEREG?` | 查询4G网络注册状态 | `+CEREG: 2,1` |
| `AT+CEREG=2` | 启用URC上报 | `OK` |
| `AT+COPS?` | 查询当前运营商 | `+COPS: 0,0,"CHINA MOBILE",7` |
| `AT+COPS=0` | 自动搜索网络 | `OK` |
| `AT+COPS=?` | 搜索可用运营商 | `+COPS: (1,"CHINA MOBILE","CMCC","46000",7),...` |
| `AT+CGATT?` | 查询GPRS附着状态 | `+CGATT: 1` |
| `AT+CGDCONT?` | 查询PDP上下文 | `+CGDCONT: 1,"IP","CMNET","0.0.0.0",0,0` |
| `AT+CGDCONT=1,"IP","CMNET"` | 配置APN | `OK` |
| `AT+MIPCALL?` | 查询IP地址 | `+MIPCALL: 1,"10.155.22.225"` |
| `AT+MIPCALL=1` | 激活PDP上下文 | `OK` |

### C. 参考资料

- ESP-IDF UART驱动文档
- ML307R AT指令手册
- ML307R硬件设计手册
- FreeRTOS任务调度文档
- 中国移动物联网卡使用指南

### D. 详细的AT指令测试记录

#### **第一轮测试（2025-11-13 11:04）**

**目的**: 诊断网络注册失败原因

| 时间 | AT指令 | 响应 | 分析 |
|------|--------|------|------|
| 11:04:23 | `AT+CPIN?` | `+CPIN: READY` | ✅ SIM卡正常 |
| 11:04:30 | （URC上报） | `+CEREG: 0` | ❌ 未注册，未搜索 |
| 11:04:36 | `AT+CSQ` | `+CSQ: 22,99` | ✅ 信号强度-69dBm（中等） |
| 11:04:52 | `AT+CREG?` | `ERROR` | ⚠️ ML307R不支持2G/3G查询 |
| 11:05:39 | `AT+CEREG?` | `+CEREG: 2,0` | ❌ 未注册，未搜索 |
| 11:05:52 | `AT+CGATT?` | `+CGATT: 0` | ❌ GPRS未附着 |
| 11:06:03 | `AT+CGPADDR` | `(空)` | ❌ 未分配IP |
| 11:06:33 | `AT+CGPADDR=1` | `(空)` | ❌ PDP上下文1无IP |
| 11:08:52 | `AT+MIPCALL?` | （无响应） | ❌ 超时 |
| 11:09:16 | `AT+MIPCALL?` | `OK` | ❌ 无数据 |

**结论**:
- SIM卡和信号正常
- 网络未注册（`+CEREG: 2,0`）
- 模块未主动搜索网络

---

#### **第二轮测试（2025-11-13 11:14）**

**目的**: 触发网络搜索并观察结果

| 时间 | AT指令 | 响应 | 分析 |
|------|--------|------|------|
| 11:14:46 | `AT+CFUN?` | `+CFUN: 1` | ✅ 全功能模式（非飞行模式） |
| 11:15:14 | `AT+CNMP?` | `ERROR` | ⚠️ ML307R不支持此指令 |
| 11:15:40 | `AT+CNMP=38` | `ERROR` | ⚠️ ML307R不支持此指令 |
| 11:16:21 | `AT+CEREG=2` | `OK` | ✅ 启用URC上报 |
| 11:16:41 | `AT+COPS=0` | `OK` | ✅ 触发自动搜索 |
| 11:17:03 | `AT+COPS?` | `+COPS: 0` | ⚠️ 自动模式，但未注册 |
| 11:17:27 | `AT+COPS=?` | （搜索中...） | - |
| 11:17:31 | （URC上报） | `+CEREG: 0` | ❌ 网络注册失败 |
| 11:18:06 | （搜索结果） | 见下表 | ✅ 搜索到4个运营商 |

**运营商搜索结果**:
```
+COPS: (1,"CHINA MOBILE","CMCC","46000",7),
       (3,"","","46015",7),
       (3,"CHN-CT","CT","46011",7),
       (3,"CHN-UNICOM","UNICOM","46001",7)
```

**分析**:
- 中国移动（46000）状态为"可用"（stat=1）
- 其他运营商被标记为"禁止"（stat=3）
- 但无法注册到中国移动网络

**结论**:
- 能搜索到网络，但无法注册
- 可能是SIM卡问题

---

#### **第三轮测试（2025-11-13 11:32）**

**目的**: 检查PDP上下文配置

| 时间 | AT指令 | 响应 | 分析 |
|------|--------|------|------|
| 11:32:46 | `AT+CGDCONT?` | `OK` | ⚠️ 未配置PDP上下文 |
| 11:33:13 | `AT+CEREG=2` | `OK` | ✅ 启用URC上报 |
| 11:34:51 | `AT+COPS=0` | `OK` | ✅ 触发自动搜索 |

**结论**:
- PDP上下文未配置
- 但这不是主要问题（因为SIM卡本身无法上网）

---

### E. 问题分析流程图

#### **问题诊断流程**

```
开始
  ↓
检查硬件连接
  ├─ 电源电压正常？
  │   ├─ 是 → 继续
  │   └─ 否 → 检查电源
  ↓
检查复位时序
  ├─ 复位引脚有脉冲？
  │   ├─ 是 → 继续
  │   └─ 否 → 检查代码逻辑
  ↓
检查波特率检测
  ├─ 能检测到波特率？
  │   ├─ 是 → 继续
  │   └─ 否 → 检查UART配置
  ↓
检查SIM卡状态
  ├─ AT+CPIN? = READY？
  │   ├─ 是 → 继续
  │   └─ 否 → 检查SIM卡
  ↓
检查信号强度
  ├─ AT+CSQ > 10？
  │   ├─ 是 → 继续
  │   └─ 否 → 检查天线
  ↓
检查网络注册
  ├─ AT+CEREG? = 2,1？
  │   ├─ 是 → 网络正常
  │   └─ 否 → 继续
  ↓
触发网络搜索
  ├─ AT+COPS=0
  ├─ 等待2分钟
  ├─ AT+CEREG? = 2,1？
  │   ├─ 是 → 网络正常
  │   └─ 否 → 继续
  ↓
搜索可用运营商
  ├─ AT+COPS=?
  ├─ 能搜索到运营商？
  │   ├─ 是 → SIM卡问题
  │   └─ 否 → 硬件问题
  ↓
检查SIM卡
  ├─ 插入手机能上网？
  │   ├─ 是 → 代码问题
  │   └─ 否 → SIM卡问题
  ↓
联系运营商
  ├─ 查询SIM卡状态
  ├─ 是否欠费？
  ├─ 是否被锁定？
  ├─ 是否是物联网卡？
  ↓
解决SIM卡问题
  ├─ 充值/解锁/更换
  ↓
重新测试
  ↓
结束
```

---

### F. 故障排查清单

#### **硬件检查清单**

- [ ] 电源电压正常（3.3V~4.2V）
- [ ] 电源电流足够（峰值2A）
- [ ] TX/RX引脚连接正确（未接反）
- [ ] 复位引脚连接正确
- [ ] SIM卡插入正确
- [ ] 天线连接正确
- [ ] 示波器验证复位波形（HIGH 100ms → LOW）
- [ ] 示波器验证TX引脚有波形
- [ ] 示波器验证RX引脚有波形

---

#### **软件检查清单**

- [ ] 复位时序代码正确执行
- [ ] 复位后等待足够时间（5秒）
- [ ] UART配置正确（波特率、数据位、停止位）
- [ ] 波特率检测包含模块的默认波特率（921600）
- [ ] 初始化顺序正确（先加载配置，再初始化硬件）
- [ ] 使用`AT+CEREG?`而不是`AT+CREG?`
- [ ] 执行了初始化AT指令（`AT+CEREG=2`、`AT+COPS=0`）
- [ ] 配置了APN（`AT+CGDCONT=1,"IP","CMNET"`）
- [ ] 添加了超时机制
- [ ] 添加了错误处理

---

#### **日志检查清单**

- [ ] 有复位引脚的日志
- [ ] 复位引脚的电平正确（HIGH → LOW）
- [ ] 有"Detecting baud rate..."日志
- [ ] 有"Detected baud rate: xxx"日志
- [ ] 有"Detected modem: xxx"日志
- [ ] 有"Waiting for network ready..."日志
- [ ] 有"Network is ready"日志（如果成功）
- [ ] 日志时间戳合理

---

#### **AT指令测试清单**

- [ ] `AT` - 测试通信
- [ ] `AT+CPIN?` - 检查SIM卡状态
- [ ] `AT+CSQ` - 检查信号强度
- [ ] `AT+CIMI` - 查询IMSI（确认是否是物联网卡）
- [ ] `AT+CCID` - 查询ICCID（确认SIM卡）
- [ ] `AT+CFUN?` - 检查功能模式
- [ ] `AT+CEREG?` - 检查4G网络注册状态
- [ ] `AT+CEREG=2` - 启用URC上报
- [ ] `AT+COPS=0` - 触发网络搜索
- [ ] `AT+COPS?` - 查询当前运营商
- [ ] `AT+COPS=?` - 搜索可用运营商
- [ ] `AT+CGATT?` - 检查GPRS附着状态
- [ ] `AT+CGDCONT?` - 检查PDP上下文配置
- [ ] `AT+MIPCALL?` - 查询IP地址

---

### G. 常见错误代码

#### **+CEREG状态码**

| 值 | 含义 | 说明 |
|----|------|------|
| 0 | 未注册，未搜索 | 模块未主动搜索网络 |
| 1 | 已注册（本地网络） | 正常状态 ✅ |
| 2 | 未注册，正在搜索 | 搜索中 ⏳ |
| 3 | 注册被拒绝 | 网络拒绝 ❌ |
| 4 | 未知 | 未知状态 ❌ |
| 5 | 已注册（漫游） | 正常状态 ✅ |

---

#### **+COPS状态码**

| 值 | 含义 | 说明 |
|----|------|------|
| 0 | 未知 | - |
| 1 | 可用 | 可以注册 ✅ |
| 2 | 当前 | 已注册 ✅ |
| 3 | 禁止 | 不能注册 ❌ |

---

#### **网络注册拒绝原因码**

| 值 | 含义 | 说明 |
|----|------|------|
| 2 | IMSI unknown in HLR | SIM卡未在运营商数据库中 |
| 3 | Illegal MS | 非法设备 |
| 6 | Illegal ME | 非法设备 |
| 7 | GPRS services not allowed | 不允许GPRS服务 |
| 11 | PLMN not allowed | 不允许该网络 |
| 12 | Location area not allowed | 不允许该位置区域 |
| 13 | Roaming not allowed | 不允许漫游 |

---

### H. 时间线对比

#### **修复前（失败）**

```
0ms     ─── 系统启动
239ms   ─── GPIO5设置为LOW（错误，应该先HIGH再LOW）
249ms   ─── 初始化ML307板卡
759ms   ─── 开始波特率检测
1919ms  ─── 波特率检测失败，重试
3079ms  ─── 波特率检测失败，重试
4239ms  ─── 波特率检测失败，重试
...     ─── 无限循环
```

**问题**: 复位时序未执行，ML307R未启动

---

#### **修复后（成功）**

```
0ms     ─── 系统启动
244ms   ─── 板卡初始化
264ms   ─── GPIO5 → HIGH（开始复位）
374ms   ─── GPIO5 → LOW（复位完成，持续110ms）
5374ms  ─── 等待5秒完成
5874ms  ─── 开始波特率检测
5904ms  ─── ✅ 检测到波特率：921600（30ms成功）
5904ms  ─── ✅ 识别模块：ML307R-DL-MBRH0S01
5914ms  ─── ✅ 网络就绪（10ms成功）
5914ms  ─── ✅ 获得IP：10.155.22.225
5924ms  ─── ✅ 读取模块信息（IMEI、ICCID）
6024ms  ─── ✅ HTTP连接创建
6694ms  ─── ✅ HTTP请求成功（状态码200）
```

**关键改进**: 复位时序正确执行，ML307R成功启动

---

#### **当前（网络注册失败）**

```
0ms     ─── 系统启动
239ms   ─── 板卡初始化
259ms   ─── GPIO5 → HIGH（开始复位）
369ms   ─── GPIO5 → LOW（复位完成，持续110ms）
5369ms  ─── 等待5秒完成
5869ms  ─── 开始波特率检测
5899ms  ─── ✅ 检测到波特率：921600（30ms成功）
5899ms  ─── ✅ 识别模块：ML307R-DL-MBRH0S01
5899ms  ─── ⏳ 等待网络就绪...
15769ms ─── 音频输入关闭（10秒超时）
24769ms ─── 音频输出关闭（19秒超时）
25779ms ─── 系统信息输出（20秒仍在等待）
...     ─── 继续等待（无限循环）
```

**问题**: SIM卡无法提供数据服务，网络注册失败

---

### I. 下一步行动计划

#### **立即执行（P0）**

**步骤1**: 查询IMSI
```
AT+CIMI
```
- 确认是否是物联网卡（以`46004`开头）

**步骤2**: 查询ICCID
```
AT+CCID
```
- 确认是同一张卡（`89860812192380763362`）

**步骤3**: 拨打运营商客服
- 电话：10086（中国移动）
- 询问内容：
  1. 这张SIM卡（ICCID: 89860812192380763362）是物联网卡吗？
  2. 这张卡的当前状态是什么？
  3. 为什么插入手机后无法上网？
  4. 是否因为插入手机被锁定了？
  5. 余额和套餐状态如何？
  6. 如何解锁？

**步骤4**: 根据客服反馈采取行动
- 如果是物联网卡被锁定 → 请求解锁，承诺不再插入手机
- 如果是欠费 → 充值
- 如果是其他原因 → 按客服指引操作

---

#### **SIM卡问题解决后执行（P1）**

**步骤5**: 重新测试ML307R
- 将SIM卡插回ML307R（**不要再插入手机**）
- 重新上电
- 观察启动日志

**步骤6**: 如果仍然失败，手动测试AT指令
```
AT+CIMI
AT+CCID
AT+CPIN?
AT+CSQ
AT+CEREG=2
AT+COPS=0
（等待2分钟）
AT+CEREG?
AT+CGDCONT=1,"IP","CMNET"  // 或 "CMIOT"（物联网卡）
AT+MIPCALL=1
AT+MIPCALL?
```

**步骤7**: 修改代码
- 修改网络检测逻辑（`AT+CREG?` → `AT+CEREG?`）
- 添加初始化AT指令
- 添加超时和回退机制

**步骤8**: 测试修改后的代码
- 编译、烧录
- 观察启动日志
- 验证网络连接

---

### J. 联系信息

**运营商客服**:
- 中国移动：10086
- 中国联通：10010
- 中国电信：10000

**技术支持**:
- ML307R技术支持：中移物联官网
- ESP-IDF技术支持：乐鑫官网

---

**文档版本**: v1.0
**最后更新**: 2025-11-13
**作者**: AI Assistant
**审核**: 待审核
**状态**: 🔄 进行中（等待SIM卡问题解决）

---

**文档结束**

