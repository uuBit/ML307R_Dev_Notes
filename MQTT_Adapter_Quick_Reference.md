# MQTT 适配器快速参考

**更新日期**：2025-11-25  
**版本**：v1.0

---

## 🎯 问题与解决方案

### 问题
EC801E GR03 版本不支持 `AT+QMTCFG` 命令，导致 MQTT 连接失败。

### 解决方案
在业务层实现 MQTT 适配器，根据模块型号和固件版本自动选择合适的 AT 指令集。

---

## 📁 文件清单

### 新增文件（6 个）

| 文件路径 | 说明 |
|---------|------|
| `main/protocols/mqtt_adapter.h` | 适配器头文件 |
| `main/protocols/mqtt_adapter.cc` | 适配器实现 |
| `main/boards/lichuang-dev/4g_module_config.json` | 配置示例 |
| `docs/MQTT_Adapter_Implementation.md` | 详细文档 |
| `main/protocols/README_MQTT_ADAPTER.md` | 使用说明 |
| `docs/MQTT_Adapter_Development_Log.md` | 开发日志 |

### 修改文件（4 个）

| 文件路径 | 修改内容 |
|---------|---------|
| `main/protocols/mqtt_protocol.cc` | 添加 `#include "mqtt_adapter.h"`<br>使用适配器连接 |
| `main/boards/common/ml307_board.cc` | 添加 `#include "settings.h"`<br>保存固件版本到 NVS |
| `main/protocols/mqtt_adapter.h/cc` | 使用原始指针 `Mqtt*`<br>使用固件版本判断模块类型 |
| `main/CMakeLists.txt` | 添加 `"protocols/mqtt_adapter.cc"` |

---

## 🔧 适配器类型

| 适配器 | 适用模块 | AT 指令 | 说明 |
|-------|---------|---------|------|
| `Ec801EGR03MqttAdapter` | EC801E GR03 | QMTOPEN/QMTCONN | 不使用 QMTCFG |
| `Ec801EGR04MqttAdapter` | EC801E GR04+ | QMTCFG + QMTOPEN | 使用 QMTCFG |
| `Ml307MqttAdapter` | ML307R | MMQTT | 使用 MMQTT |
| `DefaultMqttAdapter` | WiFi/未知 | 标准 MQTT | 直接调用 |

---

## 🔍 检测逻辑

### 步骤 1：检查配置文件

读取 `4g_module.mqtt_adapter`：
- `"auto"` → 继续自动检测
- `"ec801e_gr03"` → EC801E GR03 适配器
- `"ec801e_gr04"` → EC801E GR04+ 适配器
- `"ml307r"` → ML307R 适配器

### 步骤 2：自动检测

读取 `modem_info.firmware`：
- 包含 `"EC801E"` + `"GR03"` → EC801E GR03 适配器
- 包含 `"EC801E"` + 其他 → EC801E GR04+ 适配器
- 包含 `"ML307"` → ML307R 适配器
- 为空 → 默认适配器（WiFi）
- 其他 → 默认适配器（未知）

---

## 📝 配置文件示例

**文件路径**：`main/boards/lichuang-dev/4g_module_config.json`

```json
{
  "mqtt_adapter": "auto",
  "description": "auto | ec801e_gr03 | ec801e_gr04 | ml307r"
}
```

**手动指定适配器**：
```json
{
  "mqtt_adapter": "ec801e_gr03"
}
```

---

## 🐛 遇到的问题与解决

### 问题 1：缺少头文件
```
error: 'Settings' was not declared in this scope
```
**解决**：添加 `#include "settings.h"`

### 问题 2：智能指针转换
```
error: cannot convert 'std::unique_ptr<Mqtt>' to 'std::shared_ptr<Mqtt>'
```
**解决**：使用原始指针 `Mqtt*`，调用时使用 `mqtt_.get()`

### 问题 3：链接错误
```
undefined reference to `MqttAdapter::Create(Mqtt*)'
```
**解决**：在 `main/CMakeLists.txt` 中添加 `"protocols/mqtt_adapter.cc"`

### 问题 4：RTTI 错误
```
error: cannot use 'typeid' with '-fno-rtti'
```
**解决**：使用固件版本字符串判断模块类型，不使用 `typeid()`

---

## 📊 预期日志

### EC801E GR03
```
I (xxxx) MqttAdapter: 📋 Firmware version from NVS: EC801ECNCGR03A08M02
I (xxxx) MqttAdapter: ✅ Detected EC801E module from firmware version
I (xxxx) MqttAdapter: 🔧 Using EC801E GR03 adapter (no QMTCFG)
```

### ML307R
```
I (xxxx) MqttAdapter: 📋 Firmware version from NVS: ML307R_V1.0.0
I (xxxx) MqttAdapter: ✅ Detected ML307R module from firmware version
I (xxxx) MqttAdapter: 🔧 Using ML307R adapter
```

### WiFi
```
I (xxxx) MqttAdapter: 📋 Firmware version from NVS: 
I (xxxx) MqttAdapter: ℹ️ Firmware version is empty, using default adapter (WiFi mode)
```

---

## 🧪 测试步骤

### 1. 编译
```bash
idf.py build
```

### 2. 烧录
```bash
idf.py flash monitor
```

### 3. 验证
- ✅ 查看日志中的适配器选择信息
- ✅ 验证 MQTT 连接是否成功
- ✅ 验证音频上传是否正常
- ✅ 验证音频接收是否正常

---

## 🚀 后续工作

### 可选优化

1. **修改 managed_components**（可选）
   - 在 `ec801e_mqtt.cc` 中添加环境变量检查
   - 支持跳过 QMTCFG 配置

2. **添加更多模块支持**
   - 新增适配器类
   - 实现 `Connect()` 方法
   - 添加检测逻辑

3. **完善配置文件**
   - 添加超时配置
   - 添加 keepalive 配置
   - 添加 clean_session 配置

---

## 💡 关键经验

1. ⚠️ ESP-IDF 默认禁用 RTTI，不能使用 `typeid()`
2. ⚠️ `unique_ptr` 不能直接转换为 `shared_ptr`
3. ⚠️ 新增源文件必须添加到 `CMakeLists.txt`
4. ⚠️ 使用类之前必须包含对应的头文件
5. ✅ 使用适配器模式可以在不修改原有代码的情况下扩展功能
6. ✅ 通过配置文件可以灵活控制行为

---

## 📚 相关文档

- [MQTT 适配器开发日志](MQTT_Adapter_Development_Log.md) - 完整的开发过程
- [MQTT 适配器实现说明](MQTT_Adapter_Implementation.md) - 详细的技术文档
- [MQTT 适配器使用说明](../main/protocols/README_MQTT_ADAPTER.md) - 用户使用指南

---

**快速参考结束**

