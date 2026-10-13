# ESP32-C3 移植 + BLE 调试经验总结

项目：Respooler 步进电机控制板（ESP32-S3 → 合宙 CORE-ESP32-C3 移植）
日期：2026-10-13

---

## 1. 合宙 CORE-ESP32-C3 引脚铁律（血泪教训）

### ❌ 绝对不能用的脚（连着 SPI Flash，一配置就无限重启）

| GPIO | 用途 |
|---|---|
| GPIO11 | Flash 供电 VDD_SPI（特殊，需配置才能当 GPIO） |
| GPIO14 | Flash 片选 CS |
| GPIO15 | Flash 时钟 CLK |
| GPIO16 | Flash 数据 D0 |
| GPIO17 | Flash 数据 D1 |

**症状**：串口只打出第一行日志就重启（`[BOOT]1` 无限循环）。
因为 `pinMode()`/`ledc` 一碰这些脚 → Flash 总线断 → 取指失败 → 秒重启。

### ⚠️ Strap 脚（能用但别在上电时拉错电平）

| GPIO | 注意事项 |
|---|---|
| GPIO2 | 上电须为高（下载模式判断） |
| GPIO8 | 上电别外部下拉（影响烧录） |
| GPIO9 | BOOT 按键脚，上电别拉低 |

### ✅ 完全可用

- GPIO0~10（除 2/8/9 strap 注意事项）、GPIO12、GPIO13
- GPIO12/13 仅 QIO 模式被 Flash 占用；**板子是 DIO 模式 → 空闲**
- GPIO18/19（原生 USB D-/D+，不用原生 USB 就空闲）、GPIO20/21（UART0，连 CH340 调试口，别抢）

### 官方资料
- 引脚表：https://wiki-zh.luatos.org/chips/esp32c3/board.html
- 板载 LED：**D4=GPIO12、D5=GPIO13**（高电平亮）；按键 BOOT=GPIO9
- 原理图：`D:\WorkPRJ\工作同事\合宙ESP32\CORE-ESP32-C3设计原理图_V1.2.pdf`

### 本项目最终引脚分配（已实测跑通 Web Bluetooth 全链路）

| GPIO | 功能 | | GPIO | 功能 |
|---|---|---|---|---|
| 2 | 断丝检测 | | 9 | NeoPixel |
| 4 | 步进使能 EN | | 10 | 按钮 |
| 5 | 连接状态灯 | | 12/13 | TMC UART RX/TX（=板载D4/D5脚，DIO模式空闲） |
| 6/7 | STEP/DIR | | 19 | 风扇 |
| 8 | 舵机 | | | |

---

## 2. BLE 扫不到的排查清单（按踩坑顺序）

1. **先确认固件真的活着**：串口心跳 `[LOOP] alive` 每 5 秒打一次。没心跳 = 先修崩溃，别怀疑蓝牙。
2. **广播包 31 字节上限**：名字 + UUID + 厂商数据全塞一个包 → 超限 → NimBLE 静默发畸形包 → 所有扫描器丢弃。
   - 标准做法：**广播包** = Flags + 名字 + 服务UUID（3+8+18=29 ✅）；**扫描响应** = 厂商数据
   - 想加 TX power 就得砍别的，`addTxPower()` 占 3 字节
3. **`NimBLEDevice::setMTU(512)` 别在 C3 上用**：内存分配失败 → 协议栈半初始化 → 不广播也不报错。默认 247 够用。
4. **nRF 能扫到 ≠ 网页能扫到**：
   - nRF Connect 无过滤，显示一切广播
   - 系统蓝牙设置（Win/Android/iOS）只列标准 profile（耳机/键鼠），自定义 GATT 设备被隐藏——**正常现象**
   - **Web Bluetooth（control.lts-design.com）用 `filters:[{services:[UUID]}]` 按 UUID 匹配，广播包里必须带服务 UUID**，否则列表永远空白
5. **NimBLE 1.x API 坑**：
   - `NimBLEAdvertisementData` 没有 `addServiceUUID()`，要用 **`setCompleteServices(NimBLEUUID(...))`**
   - 自定义包（`setAdvertisementData`/`setScanResponseData`）会**完全覆盖** `pAdvertising->addServiceUUID()` 加的内容
   - `setManufacturerData()` 会自动加 0xFF 类型字节，payload 里别再带 0xFF
6. **连上后广播自动停**：nRF 里点了 CONNECT，其他设备就扫不到了。断开恢复。

### Web Bluetooth 客户端注意
- 仅 Chrome/Edge（桌面 + Android）；**iOS Safari 不支持**
- 网站过滤条件（webble-manager.js）：service `9E05D06D-...AC101`，特征 `7CB2F1B4-...B1A8`（WRITE+NOTIFY）

---

## 3. 排查方法论

- **崩溃循环**：先看串口停在哪个 `[BOOT] n` → 卡在哪一步 → 对照该步初始化的外设查引脚冲突
- **移植项目**：先把所有 `#define xx_PIN` 列出来对照目标芯片 datasheet 过一遍再编译，别等运行时炸
- **验证链路**：把可观测的输出（板载 LED）挂到被测数据（亮度滑条）上，一眼确认端到端通没通

---

## 4. 本次修改记录

| 文件 | 修改 |
|---|---|
| main.cpp | STEP 14→6、DIR 16→7（Flash 脚）；TMC UART 12/13→1/3（腾出板载 LED）；删 `setMTU(512)`；广播包重排（名字+UUID 主包，厂商数据扫描响应）；删除误用的 `addTxPower` |
| main.cpp | 新增 D4 亮度镜像：GPIO12 走 LEDC CH4/Timer0，在 `calculatePWM()` 里统一更新，网页滑条 → BLE → pwmValue → D4 逐级可验证 |

**改板接线**：TMC-TX→GPIO13、TMC-RX→GPIO12、STEP→GPIO6、DIR→GPIO7。
（验证技巧：临时把输出镜像到板载 D4=GPIO12 拖滑条看渐变，即可端到端确认链路，验证完撤掉。）
