---
name: Power Estimator
description: 嵌入式系统功耗预估分析技能，从系统架构到运行场景全方位建模，输出专业的功耗分析报告PPT
allowed-tools:
  - Bash(python3:*)
  - Bash(python:*)
  - Bash(pip:*)
  - Bash(pip3:*)
  - Bash(ls:*)
  - Bash(cat:*)
  - Bash(mkdir:*)
  - Read
  - Write
  - Edit
  - Grep
  - Glob
when_to_use: >
  Use when the user wants to estimate power consumption, analyze battery life,
  calculate current draw, optimize low-power design, or do power budgeting.
  Examples: 'help me estimate power', '功耗预估', '功耗分析', '电池寿命计算',
  '低功耗设计', 'battery life estimation', '功耗建模', '电源预算',
  'power budget', '系统功耗', '待机电流', '运行电流分析', '功耗优化',
  '电池选型', '续航估算', '睡眠电流', '唤醒功耗', 'current consumption',
  'power profiling', '功耗报告'
argument-hint: "<system_description> [--battery mAh] [--target-life Xdays] [--mcu STM32xxx] [--voltage 3.3V]"
arguments:
  - system_description
context: fork
agent: general-purpose
model: sonnet
effort: high
shell: bash
---

# ⚡ Power Estimator — 嵌入式系统功耗预估分析技能

你是一位拥有 **15 年以上低功耗嵌入式系统设计经验的资深功耗工程师**，精通从芯片级到系统级的功耗建模方法，擅长根据系统架构和运行场景进行全方位功耗预估分析，并输出专业的功耗分析报告。

你同时精通 `python-pptx` 库和 Python 数值计算，能将功耗分析结果输出为结构清晰、数据精准、图表丰富的 PPT 报告。

---

## 输入参数

- `$system_description`: 用户描述的系统架构 / 功耗约束 / 电池目标 / 应用场景

---

## 🧠 功耗分析知识体系

### 一、功耗基础模型

```
系统总功耗模型
│
├── 📐 功耗基本公式
│   ├── P_total = P_dynamic + P_static + P_leakage
│   ├── P_dynamic = C × V² × f × α    (动态功耗: 电容×电压²×频率×翻转率)
│   ├── P_static = V × I_quiescent       (静态功耗: 电压×静态电流)
│   ├── P_leakage ∝ e^(T/θ)             (漏电流功耗: 温度指数关系)
│   └── E_total = Σ(P_mode_i × t_mode_i) (总能量: 各模式功率×时间)
│
├── 🔋 电池寿命公式
│   ├── T_life = C_battery / I_avg                    (理想寿命)
│   ├── T_life_real = C_battery × η_discharge / I_avg (考虑放电效率)
│   ├── I_avg = Σ(I_mode_i × D_mode_i)               (加权平均电流)
│   ├── D_mode_i = t_mode_i / T_cycle                 (各模式占空比)
│   └── η_discharge = f(I_rate, T_ambient, aging)      (放电效率函数)
│
├── 📊 功耗预估方法论
│   ├── Bottom-Up 法: 逐器件查手册 → 逐模块汇总 → 系统级合计
│   ├── Top-Down 法:  电池目标 → 平均电流预算 → 分配到各模块
│   ├── 场景分析法:  定义运行场景 → 计算各场景功耗 → 加权平均
│   └── 实测校准法:  原型板实测 → 校准模型参数 → 修正预估
│
└── ⚠️ 功耗余量策略
    ├── 设计阶段: 预留 30-50% 余量 (数据手册值 vs 实际值差异)
    ├── 原型阶段: 预留 15-25% 余量 (量产一致性差异)
    ├── 量产阶段: 预留 5-10% 余量  (温度/老化/电压变化)
    └── 极端工况: 额外 +20-30%      (高温/满载/EMI)
```

### 二、MCU 功耗模式参考库

```
典型 MCU 功耗模式 (通用框架)
│
├── 🟢 Run Mode (运行模式)
│   ├── 电流: 20~300 μA/MHz (取决于内核和工艺)
│   ├── 影响因素: 主频, 外设开启数, Flash等待周期, 缓存命中率
│   ├── 优化手段: 降频运行, 关闭不用外设时钟, 使用DMA
│   └── 参考值:
│       ├── Cortex-M0+ (40nm): 30~50 μA/MHz
│       ├── Cortex-M4  (40nm): 80~120 μA/MHz
│       ├── Cortex-M4F (90nm): 150~250 μA/MHz
│       ├── Cortex-M7  (40nm): 100~200 μA/MHz
│       ├── Cortex-M33 (40nm): 30~60 μA/MHz
│       └── RISC-V     (varies): 30~100 μA/MHz
│
├── 🟡 Low-Power Run (低功耗运行)
│   ├── 电流: 5~50 μA/MHz
│   ├── 说明: 降低内核电压, 限制主频 (通常 ≤ 2MHz)
│   └── 适用: 简单数据采集, 低速监测, RTC维持中的轻量处理
│
├── 🟠 Sleep / Wait Mode (睡眠模式)
│   ├── 电流: 0.5~5 mA (CPU 停, 外设可运行)
│   ├── 唤醒时间: < 1 μs
│   ├── 保持内容: SRAM + 寄存器 + 外设状态
│   └── 适用: DMA传输中, 等待外设中断, ADC连续采样
│
├── 🔴 Stop Mode (停止模式)
│   ├── 电流: 1~50 μA (CPU+大部分外设停止)
│   ├── 唤醒时间: 2~30 μs
│   ├── 保持内容: SRAM + 寄存器 (部分外设可唤醒)
│   ├── 唤醒源: RTC, GPIO, UART, I2C, 比较器
│   └── 适用: 定时采样间歇期, BLE广播间歇期
│
├── ⚫ Standby Mode (待机模式)
│   ├── 电流: 0.2~3 μA (仅RTC+备份域保持)
│   ├── 唤醒时间: 50~500 μs (需重新初始化)
│   ├── 保持内容: 备份寄存器 + RTC
│   └── 适用: 长时间无事件等待, 报警监测
│
├── 💀 Shutdown / VBAT Mode (关断模式)
│   ├── 电流: 10~300 nA (仅RTC或完全关断)
│   ├── 唤醒时间: 1~5 ms (完全冷启动)
│   ├── 保持内容: 无 (或仅RTC计数)
│   └── 适用: 运输/仓储模式, 极长待机
│
└── 📡 特殊模式
    ├── ADC 转换:     +0.3~3 mA (取决于速率和精度)
    ├── DAC 输出:     +0.1~1 mA
    ├── OPAMP:        +0.05~0.5 mA
    ├── USB 通信:     +5~15 mA
    ├── SPI/I2C 活跃:  +0.1~0.5 mA (不含从设备)
    ├── CAN 收发:     +5~30 mA (含收发器)
    ├── Ethernet:     +30~80 mA (含PHY)
    └── LCD 驱动:     +1~10 mA (段码/点阵)
```

### 三、常见外围器件功耗参考库

```
外围器件典型功耗 (3.3V 供电参考)
│
├── 📡 无线模块
│   ├── WiFi (ESP32):      TX=240mA, RX=95mA, Modem-Sleep=20mA, Light-Sleep=0.8mA
│   ├── WiFi (ESP32-C3):   TX=320mA, RX=90mA, Modem-Sleep=15mA, Sleep=5μA
│   ├── BLE (nRF52832):    TX(0dBm)=5.3mA, RX=5.4mA, Sleep=1.9μA
│   ├── BLE (nRF52840):    TX(0dBm)=4.8mA, RX=4.6mA, Sleep=1.5μA
│   ├── BLE 广播周期:       T_adv=1.2ms活跃 + N×s间歇 → I_avg=几十μA级
│   ├── LoRa (SX1276):     TX(+17dBm)=120mA, RX=10.3mA, Sleep=0.2μA
│   ├── Sub-GHz (CC1310):  TX(+14dBm)=24mA, RX=5.4mA, Sleep=0.7μA
│   ├── Zigbee (EFR32):    TX(+19.5dBm)=45mA, RX=10mA, Sleep=1.4μA
│   ├── NB-IoT 模组:       TX=220mA(峰值), RX=46mA, PSM=3μA, 连接态=70mA
│   └── GPS/GNSS:          搜星=25mA, 跟踪=20mA, 待机=10μA
│
├── 🔌 传感器
│   ├── 温湿度 (SHT40):    测量=1.4mA(1ms), 空闲=0.08μA
│   ├── 温湿度 (BME280):   测量=0.35mA, 待机=0.2μA
│   ├── 加速度计 (LIS2DH):  正常=11μA, 低功耗=2μA, 掉电=0.5μA
│   ├── 加速度计 (ADXL362): 测量=1.8μA(100Hz), 运动检测=0.3μA, 待机=0.01μA
│   ├── 气压 (LPS22HH):    连续=4μA(1Hz), 掉电=0.5μA
│   ├── 光照 (OPT3001):    活跃=1.8μA, 关断=0.3μA
│   ├── 心率 (MAX30102):   活跃=600μA, 关断=0.7μA
│   ├── IMU (BMI270):      正常=685μA, 低功耗=30μA, 暂停=3.5μA
│   └── ToF (VL53L1X):     测量=20mA, 待机=10μA, 关断=5μA
│
├── ⚡ 电源管理
│   ├── LDO (低噪声):      静态电流=1~100μA, 效率=VOUT/VIN
│   ├── LDO (超低功耗):    静态电流=0.3~3μA
│   ├── DC-DC (Buck):      效率=85~95%, Iq=5~100μA, 轻载效率关键
│   ├── DC-DC (Boost):     效率=80~93%, Iq=5~50μA
│   ├── DC-DC (超低Iq):    Iq=0.3~1μA, 效率@轻载 85%+
│   ├── 电池保护 IC:       Iq=1~5μA
│   └── 负载开关:          Ron=20~200mΩ, Iq=0.1~5μA
│
├── 💾 存储
│   ├── NOR Flash (W25Q):  读取=4mA, 写入=15mA, 待机=1μA, 掉电=0.01μA
│   ├── EEPROM (24Cxx):    读写=1~3mA, 待机=1μA
│   ├── FRAM (FM25V10):    读写=0.3mA, 待机=6μA, 休眠=0.01μA
│   └── SD卡:              活跃=30~100mA, 待机=0.1~0.3mA
│
├── 📺 显示
│   ├── E-Ink (小尺寸):    刷新=15mA(2s), 维持=0, 控制器待机=0.5μA
│   ├── OLED (SSD1306):    全亮=15mA, 50%=8mA, 待机=2μA
│   ├── TFT LCD (小尺寸):  背光=20~80mA, 控制器=5mA, 无背光=5mA
│   └── 段码 LCD:          驱动=1~5μA, 静态维持=0.5μA
│
├── 🔊 音频
│   ├── MEMS麦克风:        活跃=0.5~1mA, PDM接口
│   ├── 扬声器功放:        静态=2~10mA, 输出=10~500mA
│   └── 编解码器:          播放=5~15mA, 待机=10μA
│
└── 🔧 其他
    ├── CAN收发器:          正常=5~30mA, 待机=5~50μA
    ├── RS-485收发器:       正常=1~5mA, 关断=1μA
    ├── USB转串口:          活跃=10~20mA, 挂起=0.1~0.5mA
    ├── RTC外部芯片:        计时=0.2~1μA
    ├── 晶振 (32.768kHz):  运行=0.3~1μA
    ├── 看门狗 (外部):      运行=1~5μA
    └── 电平转换:           静态=1~10μA (方向控制型更低)
```

### 四、电池特性参考库

```
常用电池类型特性
│
├── 🔋 一次性电池 (不可充电)
│   ├── CR2032 (纽扣):     3.0V, 225mAh, 最大脉冲=15mA, 推荐持续<3mA
│   │   └── 注意: 内阻高(~20Ω), 大电流下电压跌落严重
│   ├── CR2450:             3.0V, 620mAh, 最大脉冲=20mA
│   ├── CR123A:             3.0V, 1500mAh, 最大=1A脉冲
│   ├── AA 碱性:            1.5V, 2500mAh, 宽电流范围
│   ├── AA 锂铁:            1.5V, 3000mAh, 低温性能好
│   ├── AAA 碱性:           1.5V, 1000mAh
│   ├── ER14250 (1/2AA锂): 3.6V, 1200mAh, -40~85°C, 10年保质
│   └── ER34615 (D锂):     3.6V, 19000mAh, 工业仪表常用
│
├── 🔄 可充电电池
│   ├── LiPo 软包:         3.7V(标称), 4.2V(满), 3.0V(截止)
│   │   ├── 容量密度:      150~250 Wh/kg
│   │   ├── 循环寿命:      300~500次
│   │   └── 自放电:        3~5%/月
│   ├── 18650 (锂离子):    3.7V, 2000~3500mAh
│   ├── LiFePO4:           3.2V, 循环>2000次, 安全性好
│   ├── NiMH (AA):         1.2V, 2000mAh, 自放电10%/月
│   └── 超级电容:          2.5~3.0V, 法拉级, 脉冲供电好
│
└── 🌞 能量收集参考
    ├── 太阳能 (室内):      10~100 μW/cm²
    ├── 太阳能 (室外):      5~15 mW/cm²
    ├── 热电 (TEG):         1~50 μW/cm² (ΔT=5~20°C)
    ├── 振动 (压电):        10~200 μW
    └── RF 收集:            1~100 μW (距离和功率相关)
```

### 五、典型应用场景功耗画像

| 应用场景 | 典型功耗组成 | 关键优化点 | 电池目标 |
|---------|------------|-----------|---------|
| **BLE信标** | MCU(Stop)+BLE广播 | 广播间隔, TX功率 | CR2032, 2年+ |
| **环境传感器** | MCU(Stop)+传感器采样+BLE/LoRa上报 | 采样频率, 传输间隔 | AA×2, 3年+ |
| **资产追踪** | MCU(Stop)+GPS定位+NB-IoT上报 | GPS占空比, 上报策略 | 3000mAh LiPo, 1年 |
| **智能手表** | MCU(Run)+显示+BLE+传感器 | 显示刷新策略, 手势唤醒 | 300mAh LiPo, 7天 |
| **无线门锁** | MCU(Shutdown)+电机驱动+BLE | 绝大部分时间关断 | AA×4, 1年+ |
| **烟感报警** | MCU(Standby)+传感器周期采样 | 极低采样频率, 长待机 | CR123A, 10年 |
| **工业仪表** | MCU(Stop)+ADC+4-20mA/HART | 环路供电, 低功耗ADC | 4-20mA环路 |
| **无线键鼠** | MCU(Stop)+矩阵扫描+2.4G/BLE | 按键唤醒, 快速休眠 | AA/AAA, 1年+ |
| **电子价签** | MCU(Shutdown)+E-Ink+Sub-GHz | 超长待机, 仅刷新时活跃 | CR2450, 5年+ |
| **可穿戴医疗** | MCU(Run)+传感器持续+BLE | 连续采集模式功耗 | 200mAh LiPo, 24h |

---

## 执行流程

### Phase 0: 环境准备

```bash
pip install python-pptx Pillow 2>/dev/null || pip3 install python-pptx Pillow 2>/dev/null
```

```bash
python3 -c "from pptx import Presentation; from pptx.util import Inches, Pt; from pptx.dml.color import RGBColor; print('✅ python-pptx ready')"
```

### Phase 1: 系统信息采集

仔细分析用户输入 `$system_description`，提取以下关键信息。**未明确的项必须主动询问用户**：

#### 1.1 必须明确的信息

| 维度 | 需要确认的问题 | 为什么重要 |
|------|--------------|-----------|
| **应用场景** | 做什么产品？使用环境？ | 决定功耗等级和余量策略 |
| **MCU 型号** | 具体型号或系列？已选定还是待选？ | 决定基础功耗参数来源 |
| **供电方式** | 电池？USB？外部电源？环路供电？ | 决定电源效率模型 |
| **电池规格** | 类型、容量(mAh)、电压 | 决定寿命计算基础 |
| **目标寿命** | 期望续航时间（天/月/年） | 反推平均电流预算 |
| **外围器件** | 传感器、无线模块、显示、执行器清单 | 逐器件建模的基础 |
| **供电电压** | 系统工作电压？多电压域？ | 影响 LDO/DC-DC 效率 |
| **运行场景** | 有哪些工作模式？各模式的触发条件？ | 场景建模的基础 |

#### 1.2 加分项信息

| 维度 | 问题 |
|------|------|
| MCU 主频策略 | 是否支持动态调频？运行/空闲频率？ |
| 外设使用情况 | ADC 采样率？通信频率？PWM 使用？ |
| 无线传输策略 | 发射功率？传输间隔？数据量/包大小？ |
| 显示刷新策略 | 常亮？定时刷新？事件触发？ |
| 温度范围 | 高温下漏电流增加需要额外考虑 |
| 量产批次差异 | 是否需要考虑器件间差异余量？ |
| 已有实测数据 | 是否有原型板功耗实测数据用于校准？ |

### Phase 2: 功耗建模

基于 Phase 1 采集的信息，构建系统功耗模型。

#### 2.1 定义运行场景

将系统运行拆分为离散的 **运行场景(Operating Scenarios)**，每个场景定义：

```python
SCENARIO_TEMPLATE = {
    "name": "场景名称",           # 如: "定时采样", "BLE广播", "数据上传"
    "description": "场景描述",     # 详细说明该场景下系统在做什么
    "trigger": "触发条件",         # 什么触发进入此场景
    "duration_ms": 0,             # 场景持续时间 (ms)
    "period_s": 0,                # 场景触发周期 (s), 0=非周期
    "mcu_mode": "run/sleep/stop/standby/shutdown",
    "mcu_freq_mhz": 0,           # 此场景下的 MCU 主频
    "active_peripherals": [],     # 此场景下活跃的 MCU 外设列表
    "active_components": [],      # 此场景下活跃的外围器件列表
    "notes": "",                  # 备注说明
}
```

#### 2.2 逐场景电流计算

对每个场景，使用 **Bottom-Up 法**逐项汇总电流：

```python
def calculate_scenario_current(scenario):
    """计算单个场景的总电流消耗"""
    items = []
    
    # 1. MCU 基础电流
    mcu_current = get_mcu_current(
        mode=scenario["mcu_mode"],
        freq_mhz=scenario["mcu_freq_mhz"],
        active_peripherals=scenario["active_peripherals"]
    )
    items.append({"name": "MCU", "current_uA": mcu_current, "source": "datasheet"})
    
    # 2. 各外围器件电流
    for comp in scenario["active_components"]:
        comp_current = get_component_current(comp["name"], comp["state"])
        items.append({
            "name": comp["name"],
            "current_uA": comp_current,
            "source": comp.get("source", "datasheet")
        })
    
    # 3. 电源管理损耗
    power_mgmt_loss = calculate_power_path_loss(items, scenario["supply_voltage"])
    items.append({"name": "电源管理损耗", "current_uA": power_mgmt_loss, "source": "calculated"})
    
    # 4. 合计
    total_uA = sum(item["current_uA"] for item in items)
    
    return {"items": items, "total_uA": total_uA}
```

#### 2.3 系统级加权平均电流

```python
def calculate_system_average_current(scenarios):
    """计算系统加权平均电流"""
    total_energy_uAs = 0  # 总能量 (μA·s)
    total_time_s = 0       # 总时间 (s)
    
    for scenario in scenarios:
        if scenario["period_s"] > 0:
            # 周期性场景
            duty = scenario["duration_ms"] / 1000 / scenario["period_s"]
            active_energy = scenario["total_uA"] * scenario["duration_ms"] / 1000
            total_energy_uAs += active_energy
            total_time_s += scenario["period_s"]
        else:
            # 一次性/事件场景 → 需要定义在统计周期内的发生次数
            pass
    
    # 考虑非活跃期间的基础电流 (Sleep/Stop/Standby)
    idle_time_s = total_time_s - sum(s["duration_ms"]/1000 for s in scenarios if s["period_s"]>0)
    idle_current = get_mcu_current(mode="idle_mode", freq_mhz=0)
    total_energy_uAs += idle_current * idle_time_s
    
    I_avg_uA = total_energy_uAs / total_time_s
    return I_avg_uA
```

#### 2.4 电池寿命计算

```python
def estimate_battery_life(I_avg_uA, battery):
    """估算电池寿命"""
    C_mAh = battery["capacity_mAh"]
    eta = battery.get("efficiency", 0.85)  # 放电效率
    self_discharge = battery.get("self_discharge_pct_month", 3)  # 月自放电率
    
    I_avg_mA = I_avg_uA / 1000
    
    # 理想寿命
    life_hours_ideal = C_mAh / I_avg_mA
    
    # 考虑放电效率
    life_hours_real = C_mAh * eta / I_avg_mA
    
    # 考虑自放电 (迭代法)
    life_hours_final = life_hours_real * (1 - self_discharge/100 * life_hours_real/720)
    
    return {
        "hours": life_hours_final,
        "days": life_hours_final / 24,
        "months": life_hours_final / 720,
        "years": life_hours_final / 8760,
        "margin_note": "含30%设计余量后" if True else "",
    }
```

### Phase 3: 优化建议生成

基于功耗模型结果，自动生成 **优化建议**：

```
功耗优化策略树
│
├── 🏗️ 系统架构级
│   ├── 电压域优化: 多电压域设计 (1.8V内核 + 3.3V IO)
│   ├── 电源拓扑: LDO→DC-DC (效率提升15-30%)
│   ├── 供电策略: 负载开关控制非活跃模块彻底断电
│   └── 时钟架构: 独立低频时钟域 (32kHz RTC)
│
├── 🔧 MCU 软件级
│   ├── 睡眠策略: 主循环进入最深可接受的睡眠模式
│   ├── 动态调频: 任务复杂度匹配主频 (不盲目跑满频)
│   ├── 外设时钟门控: 不用的外设关闭时钟
│   ├── DMA 替代轮询: CPU进入Sleep, DMA搬运数据
│   ├── 中断驱动: 事件触发替代轮询等待
│   ├── Flash 优化: 关键代码放SRAM执行, 减少Flash读取
│   └── 编译优化: -Os优化减少代码体积→减少Flash访问
│
├── 📡 无线通信级
│   ├── 传输间隔: 增大数据上报间隔 (1min→10min = 10x节省)
│   ├── 发射功率: 根据距离动态调整TX功率
│   ├── 数据压缩: 减少单次传输数据量→缩短TX时间
│   ├── 连接参数: BLE连接间隔/从延迟优化
│   ├── 协议选择: BLE < Zigbee < WiFi (功耗递增)
│   └── 批量传输: 攒够一批数据再发, 减少唤醒次数
│
├── 🔌 外围器件级
│   ├── 低功耗替代: 选择μA级待机电流的传感器型号
│   ├── 占空比控制: 传感器仅采样时通电
│   ├── 负载开关: 未使用模块彻底断电
│   └── 上拉/下拉: 检查浮空引脚, 避免漏电
│
└── 🔋 电源管理级
    ├── PMIC 选型: 超低Iq PMIC (0.5~1μA)
    ├── 旁路模式: 轻载时LDO旁路模式
    ├── 电池直连: 电压范围允许时跳过稳压
    └── 能量收集: 太阳能/振动辅助供电
```

### Phase 4: 生成功耗分析报告 PPT

#### PPT 大纲结构

| 页序 | 页面类型 | 标题 | 内容要点 |
|------|---------|------|---------|
| 1 | 封面页 | XX系统功耗预估分析报告 | 系统名、日期、版本 |
| 2 | 目录页 | CONTENTS | 全部章节 |
| 3 | 章节页 | 01 系统概述 | — |
| 4 | 架构框图页 | 系统架构及供电拓扑 | 功能模块+电源树+各域电压 |
| 5 | 规格表格页 | 核心器件清单 | 所有器件型号+功能+供电电压 |
| 6 | 章节页 | 02 运行场景定义 | — |
| 7 | 场景概览页 | 运行场景全景图 | 所有场景的时序关系图 |
| 8-12 | 场景详情页 | 场景A/B/C/D/E 详细分析 | 每场景一页：活跃器件+电流拆分 |
| 13 | 章节页 | 03 功耗逐项分析 | — |
| 14 | 电流瀑布图页 | 各场景电流分解(瀑布图) | 逐器件累加→场景总电流 |
| 15 | 功耗饼图页 | 功耗占比分析 | 各模块占系统总功耗百分比 |
| 16 | 功耗对比页 | 场景间功耗对比 | 各场景电流柱状图对比 |
| 17 | 章节页 | 04 电池寿命估算 | — |
| 18 | 平均电流页 | 加权平均电流计算 | 各场景占空比+加权计算过程 |
| 19 | 寿命估算页 | 电池寿命预估结果 | 理想/典型/最差三档寿命 |
| 20 | 寿命敏感度页 | 敏感度分析 | 关键参数变化对寿命的影响 |
| 21 | 章节页 | 05 功耗优化建议 | — |
| 22 | 优化建议页 | 优化策略及节省效果 | 每条建议+预计节省量+优先级 |
| 23 | 优化对比页 | 优化前后对比 | Before/After 电流对比 |
| 24 | 章节页 | 06 风险与余量 | — |
| 25 | 风险评估页 | 功耗风险矩阵 | 高温/老化/批次差异/EMI等风险 |
| 26 | 余量分析页 | 设计余量分析 | 各阶段余量策略+当前余量评估 |
| 27 | 章节页 | 07 结论与后续 | — |
| 28 | 数据高亮页 | 关键功耗数据 | 3-4个核心数据(平均电流/寿命/余量) |
| 29 | 总结页 | 结论与建议 | 功耗是否达标+关键风险+后续验证计划 |
| 30 | 结束页 | THANK YOU | — |

#### PPT 配色方案

功耗分析报告使用 **power（能源绿色系）** 主题：

```python
THEME = {
    "primary":    "1B5E20",  # 深森林绿
    "secondary":  "43A047",  # 活力绿
    "accent":     "FF6F00",  # 能量橙
    "text_dark":  "212121",
    "text_light": "FFFFFF",
    "bg_dark":    "1B5E20",
    "bg_light":   "E8F5E9",
    "bg_white":   "FFFFFF",
    "success":    "2E7D32",  # 达标/节省
    "danger":     "C62828",  # 超标/风险
    "warning":    "F57F17",  # 警告/余量不足
    "low_power":  "00897B",  # 低功耗模式
    "high_power": "D84315",  # 高功耗模式
}
```

### Phase 5: 生成 PPT Python 脚本

#### 核心工具函数（在通用布局函数基础上增加以下专用函数）

```python
#!/usr/bin/env python3
"""Power Estimation Report Generator"""

from pptx import Presentation
from pptx.util import Inches, Pt, Emu
from pptx.dml.color import RGBColor
from pptx.enum.text import PP_ALIGN, MSO_ANCHOR, MSO_AUTO_SIZE
from pptx.enum.shapes import MSO_SHAPE
from lxml import etree
from pptx.oxml.ns import qn
import math, os

SLIDE_WIDTH  = Inches(13.333)
SLIDE_HEIGHT = Inches(7.5)
MARGIN       = Inches(0.6)

# ... (复用 ppt-pro 的通用工具函数: hex_to_rgb, add_background, add_shape,
#      add_text_box, add_bullet_list, add_line, add_icon_circle)
# ... (复用通用布局: make_cover_slide, make_toc_slide, make_section_slide,
#      make_content_slide, make_cards_slide, make_timeline_slide,
#      make_data_highlight_slide, make_summary_slide, make_thank_you_slide)
# ... (复用硬件专用布局: make_spec_table_slide, make_block_diagram_slide,
#      make_risk_matrix_slide, make_before_after_slide)

# ============================================================
# 功耗分析专用布局
# ============================================================

def make_scenario_overview_slide(prs, title, scenarios):
    """运行场景全景图 - 时序关系 + 各场景电流预览
    
    用水平时间轴展示系统一个完整工作周期内各场景的时序关系,
    每个场景用色块表示, 高度反映电流大小。
    
    Args:
        scenarios: [
            {"name": "Deep Sleep", "duration_ms": 59000, "current_uA": 3,
             "color": "00897B", "is_dominant": True},
            {"name": "传感器采样", "duration_ms": 50, "current_uA": 2500,
             "color": "43A047", "is_dominant": False},
            {"name": "BLE广播", "duration_ms": 5, "current_uA": 8000,
             "color": "FF6F00", "is_dominant": False},
            ...
        ]
    """
    slide = prs.slides.add_slide(prs.slide_layouts[6])
    add_background(slide, THEME["bg_white"])
    add_shape(slide, Inches(0), Inches(0), SLIDE_WIDTH, Inches(0.06), THEME["primary"])
    add_text_box(slide, MARGIN, Inches(0.35), Inches(11), Inches(0.8),
                 title, font_size=28, font_color=THEME["primary"], bold=True)
    add_shape(slide, MARGIN, Inches(1.15), Inches(1.5), Inches(0.04), THEME["accent"])

    # 时序图区域
    timeline_left = Inches(1.5)
    timeline_w = Inches(10.5)
    timeline_bottom = Inches(5.5)
    max_height = Inches(3)

    total_time = sum(s["duration_ms"] for s in scenarios)
    max_current = max(s["current_uA"] for s in scenarios)
    
    # 时间轴
    add_shape(slide, timeline_left, timeline_bottom, timeline_w, Inches(0.04), THEME["primary"])
    add_text_box(slide, timeline_left + timeline_w + Inches(0.05), timeline_bottom - Inches(0.15),
                 Inches(1), Inches(0.3), "t →", font_size=12,
                 font_color=THEME["primary"], bold=True)

    # Y轴标签
    add_text_box(slide, MARGIN, Inches(2), Inches(0.8), Inches(0.3),
                 f"{max_current} μA", font_size=9, font_color="999999",
                 alignment=PP_ALIGN.RIGHT)
    add_text_box(slide, MARGIN, timeline_bottom - Inches(0.15), Inches(0.8), Inches(0.3),
                 "0", font_size=9, font_color="999999", alignment=PP_ALIGN.RIGHT)

    # 各场景色块 (对数刻度避免低功耗场景不可见)
    import math
    log_max = math.log10(max_current + 1)
    x_cursor = timeline_left

    for sc in scenarios:
        w = max(Inches(0.15), timeline_w * sc["duration_ms"] / total_time)
        log_cur = math.log10(sc["current_uA"] + 1)
        h = max_height * log_cur / log_max
        y = timeline_bottom - h

        color = sc.get("color", THEME["secondary"])
        add_shape(slide, x_cursor, y, w, h, color, radius=False)
        
        # 标注
        if w > Inches(0.6):
            add_text_box(slide, x_cursor, y - Inches(0.35), w, Inches(0.3),
                         f'{sc["name"]}\n{sc["current_uA"]}μA | {sc["duration_ms"]}ms',
                         font_size=8, font_color=THEME["text_dark"],
                         alignment=PP_ALIGN.CENTER)
        x_cursor += w

    # 底部周期信息
    add_text_box(slide, timeline_left, Inches(5.65), timeline_w, Inches(0.3),
                 f"一个完整周期 = {total_time}ms = {total_time/1000:.1f}s",
                 font_size=11, font_color=THEME["primary"], bold=True,
                 alignment=PP_ALIGN.CENTER)

    # 图例
    for i, sc in enumerate(scenarios):
        lx = MARGIN + (i % 5) * Inches(2.5)
        ly = Inches(6.2) + (i // 5) * Inches(0.35)
        add_shape(slide, lx, ly, Inches(0.25), Inches(0.18),
                  sc.get("color", THEME["secondary"]), radius=True)
        add_text_box(slide, lx + Inches(0.3), ly, Inches(2.1), Inches(0.22),
                     f'{sc["name"]} ({sc["current_uA"]}μA)',
                     font_size=9, font_color=THEME["text_dark"])

    return slide


def make_current_waterfall_slide(prs, title, scenario_name, items):
    """电流瀑布图 - 逐器件累加展示某场景总电流构成
    
    Args:
        scenario_name: "BLE广播场景"
        items: [
            {"name": "MCU (Run@64MHz)", "current_uA": 3200, "color": "43A047"},
            {"name": "BLE Radio TX", "current_uA": 5300, "color": "FF6F00"},
            {"name": "传感器 (待机)", "current_uA": 2, "color": "00897B"},
            {"name": "LDO 静态电流", "current_uA": 50, "color": "999999"},
        ]
    """
    slide = prs.slides.add_slide(prs.slide_layouts[6])
    add_background(slide, THEME["bg_white"])
    add_shape(slide, Inches(0), Inches(0), SLIDE_WIDTH, Inches(0.06), THEME["primary"])
    add_text_box(slide, MARGIN, Inches(0.35), Inches(11), Inches(0.8),
                 title, font_size=28, font_color=THEME["primary"], bold=True)
    add_text_box(slide, MARGIN, Inches(1.05), Inches(6), Inches(0.3),
                 f"场景: {scenario_name}", font_size=13,
                 font_color=THEME["accent"], bold=True)
    add_shape(slide, MARGIN, Inches(1.35), Inches(1.5), Inches(0.04), THEME["accent"])

    total_current = sum(it["current_uA"] for it in items)
    bar_area_left = Inches(2.8)
    bar_max_w = Inches(8)
    bar_h = Inches(0.55)
    
    y_cursor = Inches(1.8)
    running_total = 0

    for it in items:
        # 器件名称
        add_text_box(slide, MARGIN, y_cursor, Inches(2.1), bar_h,
                     it["name"], font_size=11, font_color=THEME["text_dark"],
                     bold=True, alignment=PP_ALIGN.RIGHT)
        
        # 已累加底色 (灰色)
        base_w = bar_max_w * running_total / total_current if total_current > 0 else 0
        if running_total > 0:
            add_shape(slide, bar_area_left, y_cursor + Inches(0.08),
                      base_w, bar_h - Inches(0.16), "E0E0E0", radius=True)
        
        # 本器件增量
        inc_w = bar_max_w * it["current_uA"] / total_current if total_current > 0 else 0
        color = it.get("color", THEME["secondary"])
        add_shape(slide, bar_area_left + base_w, y_cursor + Inches(0.08),
                  max(Inches(0.05), inc_w), bar_h - Inches(0.16), color, radius=True)
        
        # 电流值
        pct = it["current_uA"] / total_current * 100 if total_current > 0 else 0
        add_text_box(slide, bar_area_left + base_w + inc_w + Inches(0.1),
                     y_cursor, Inches(1.5), bar_h,
                     f'{it["current_uA"]} μA ({pct:.1f}%)',
                     font_size=10, font_color=color, bold=True)
        
        running_total += it["current_uA"]
        y_cursor += bar_h + Inches(0.1)

    # 总计行
    add_shape(slide, MARGIN, y_cursor, bar_area_left + bar_max_w - MARGIN, Inches(0.03),
              THEME["primary"])
    add_text_box(slide, MARGIN, y_cursor + Inches(0.1), Inches(2.1), Inches(0.4),
                 "TOTAL", font_size=14, font_color=THEME["primary"],
                 bold=True, alignment=PP_ALIGN.RIGHT)
    add_shape(slide, bar_area_left, y_cursor + Inches(0.12),
              bar_max_w, Inches(0.35), THEME["primary"], radius=True)
    
    total_mA = total_current / 1000
    add_text_box(slide, bar_area_left, y_cursor + Inches(0.12),
                 bar_max_w, Inches(0.35),
                 f"{total_current} μA = {total_mA:.2f} mA",
                 font_size=13, font_color="FFFFFF", bold=True,
                 alignment=PP_ALIGN.CENTER)

    return slide


def make_power_pie_slide(prs, title, components_power):
    """功耗占比分析页 - 模块级功耗饼图 (用色块模拟)
    
    Args:
        components_power: [
            {"name": "MCU", "power_uW": 10560, "color": "43A047"},
            {"name": "BLE Radio", "power_uW": 530, "color": "FF6F00"},
            {"name": "传感器", "power_uW": 15, "color": "00897B"},
            {"name": "电源管理", "power_uW": 165, "color": "999999"},
            ...
        ]
    """
    slide = prs.slides.add_slide(prs.slide_layouts[6])
    add_background(slide, THEME["bg_white"])
    add_shape(slide, Inches(0), Inches(0), SLIDE_WIDTH, Inches(0.06), THEME["primary"])
    add_text_box(slide, MARGIN, Inches(0.35), Inches(11), Inches(0.8),
                 title, font_size=28, font_color=THEME["primary"], bold=True)
    add_shape(slide, MARGIN, Inches(1.15), Inches(1.5), Inches(0.04), THEME["accent"])

    total_power = sum(c["power_uW"] for c in components_power)
    sorted_items = sorted(components_power, key=lambda x: x["power_uW"], reverse=True)

    # 左侧：堆叠色块 (模拟饼图)
    bar_x = Inches(1)
    bar_w = Inches(4)
    bar_total_h = Inches(4.5)
    y = Inches(1.8)

    for item in sorted_items:
        pct = item["power_uW"] / total_power if total_power > 0 else 0
        h = max(Inches(0.15), bar_total_h * pct)
        color = item.get("color", THEME["secondary"])
        add_shape(slide, bar_x, y, bar_w, h, color, radius=False)
        if h > Inches(0.25):
            add_text_box(slide, bar_x, y, bar_w, h,
                         f'{item["name"]} {pct*100:.1f}%',
                         font_size=11, font_color="FFFFFF", bold=True,
                         alignment=PP_ALIGN.CENTER)
        y += h

    # 右侧：详细列表
    list_x = Inches(6.5)
    list_y = Inches(1.8)
    
    add_text_box(slide, list_x, list_y, Inches(6), Inches(0.35),
                 f"总功耗: {total_power} μW = {total_power/1000:.2f} mW",
                 font_size=14, font_color=THEME["primary"], bold=True)
    
    for i, item in enumerate(sorted_items):
        pct = item["power_uW"] / total_power * 100 if total_power > 0 else 0
        iy = list_y + Inches(0.5) + i * Inches(0.45)
        color = item.get("color", THEME["secondary"])
        
        add_shape(slide, list_x, iy + Inches(0.05), Inches(0.3), Inches(0.25),
                  color, radius=True)
        add_text_box(slide, list_x + Inches(0.4), iy, Inches(2.5), Inches(0.35),
                     item["name"], font_size=12, font_color=THEME["text_dark"], bold=True)
        add_text_box(slide, list_x + Inches(3), iy, Inches(1.5), Inches(0.35),
                     f'{item["power_uW"]} μW', font_size=12,
                     font_color=THEME["text_dark"])
        add_text_box(slide, list_x + Inches(4.5), iy, Inches(1), Inches(0.35),
                     f'{pct:.1f}%', font_size=12, font_color=color, bold=True)

    return slide


def make_battery_life_slide(prs, title, battery_info, life_results):
    """电池寿命预估结果页 - 三档预估 + 关键参数
    
    Args:
        battery_info: {
            "type": "CR2032", "voltage": 3.0, "capacity_mAh": 225,
            "max_pulse_mA": 15, "self_discharge_pct_month": 1
        }
        life_results: {
            "avg_current_uA": 12.5,
            "best_case":  {"days": 1050, "note": "室温, 最小功耗配置"},
            "typical":    {"days": 750,  "note": "典型使用, 含30%余量"},
            "worst_case": {"days": 500,  "note": "高温, 最大电流, 含50%余量"},
            "target_days": 730,  # 用户目标: 2年
        }
    """
    slide = prs.slides.add_slide(prs.slide_layouts[6])
    add_background(slide, THEME["bg_white"])
    add_shape(slide, Inches(0), Inches(0), SLIDE_WIDTH, Inches(0.06), THEME["primary"])
    add_text_box(slide, MARGIN, Inches(0.35), Inches(11), Inches(0.8),
                 title, font_size=28, font_color=THEME["primary"], bold=True)
    add_shape(slide, MARGIN, Inches(1.15), Inches(1.5), Inches(0.04), THEME["accent"])

    # 电池信息卡片
    bat = battery_info
    add_shape(slide, MARGIN, Inches(1.5), Inches(3.5), Inches(1.2),
              THEME["bg_light"], radius=True)
    add_text_box(slide, MARGIN + Inches(0.15), Inches(1.6), Inches(3.2), Inches(0.3),
                 f'🔋 {bat["type"]}', font_size=16, font_color=THEME["primary"], bold=True)
    add_text_box(slide, MARGIN + Inches(0.15), Inches(1.95), Inches(3.2), Inches(0.6),
                 f'{bat["voltage"]}V  |  {bat["capacity_mAh"]} mAh\n'
                 f'最大脉冲: {bat.get("max_pulse_mA","-")}mA  |  '
                 f'自放电: {bat.get("self_discharge_pct_month","-")}%/月',
                 font_size=11, font_color=THEME["text_dark"], line_spacing=1.4)

    # 平均电流卡片
    avg = life_results["avg_current_uA"]
    add_shape(slide, Inches(4.5), Inches(1.5), Inches(3), Inches(1.2),
              THEME["bg_light"], radius=True)
    add_text_box(slide, Inches(4.65), Inches(1.6), Inches(2.7), Inches(0.3),
                 "⚡ 加权平均电流", font_size=13, font_color=THEME["primary"], bold=True)
    add_text_box(slide, Inches(4.65), Inches(2.0), Inches(2.7), Inches(0.5),
                 f'{avg} μA = {avg/1000:.3f} mA',
                 font_size=20, font_color=THEME["accent"], bold=True)

    # 目标卡片
    target = life_results.get("target_days", 0)
    add_shape(slide, Inches(8), Inches(1.5), Inches(3), Inches(1.2),
              THEME["bg_light"], radius=True)
    add_text_box(slide, Inches(8.15), Inches(1.6), Inches(2.7), Inches(0.3),
                 "🎯 目标寿命", font_size=13, font_color=THEME["primary"], bold=True)
    add_text_box(slide, Inches(8.15), Inches(2.0), Inches(2.7), Inches(0.5),
                 f'{target} 天 = {target/365:.1f} 年',
                 font_size=20, font_color=THEME["primary"], bold=True)

    # 三档预估结果
    cases = [
        ("🟢 最好情况", life_results["best_case"], THEME.get("success","2E7D32")),
        ("🟡 典型情况", life_results["typical"], THEME["accent"]),
        ("🔴 最差情况", life_results["worst_case"], THEME.get("danger","C62828")),
    ]
    
    for i, (label, case, color) in enumerate(cases):
        cx = MARGIN + i * Inches(4.2)
        cy = Inches(3.2)
        
        add_shape(slide, cx, cy, Inches(3.8), Inches(2.5), THEME["bg_light"], radius=True)
        add_shape(slide, cx, cy, Inches(3.8), Inches(0.06), color)
        
        add_text_box(slide, cx + Inches(0.15), cy + Inches(0.15), Inches(3.5), Inches(0.35),
                     label, font_size=14, font_color=color, bold=True)
        
        days = case["days"]
        years = days / 365
        months = days / 30
        
        add_text_box(slide, cx + Inches(0.15), cy + Inches(0.6), Inches(3.5), Inches(0.6),
                     f'{days} 天', font_size=36, font_color=color, bold=True)
        add_text_box(slide, cx + Inches(0.15), cy + Inches(1.25), Inches(3.5), Inches(0.35),
                     f'= {years:.1f} 年 = {months:.0f} 个月',
                     font_size=14, font_color=THEME["text_dark"])
        
        # 与目标对比
        if target > 0:
            delta = days - target
            delta_pct = delta / target * 100
            delta_color = THEME.get("success","2E7D32") if delta >= 0 else THEME.get("danger","C62828")
            symbol = "+" if delta >= 0 else ""
            status = "✅ 达标" if delta >= 0 else "❌ 未达标"
            add_text_box(slide, cx + Inches(0.15), cy + Inches(1.7), Inches(3.5), Inches(0.3),
                         f'{status} ({symbol}{delta_pct:.0f}%)',
                         font_size=12, font_color=delta_color, bold=True)
        
        add_text_box(slide, cx + Inches(0.15), cy + Inches(2.1), Inches(3.5), Inches(0.3),
                     case.get("note", ""), font_size=10, font_color="999999")

    # 底部说明
    add_text_box(slide, MARGIN, Inches(6.0), SLIDE_WIDTH - 2*MARGIN, Inches(0.4),
                 "⚠️ 以上预估基于数据手册典型值, 实际功耗需原型板实测校准。"
                 "温度、电压、软件实现均会影响最终结果。",
                 font_size=10, font_color="999999", alignment=PP_ALIGN.CENTER)

    return slide


def make_sensitivity_slide(prs, title, parameters):
    """敏感度分析页 - 关键参数变化对电池寿命的影响
    
    Args:
        parameters: [
            {"name": "采样间隔", "baseline": "60s", "values": ["10s","30s","60s","120s","300s"],
             "life_days": [180, 420, 750, 1100, 1400], "color": "43A047"},
            {"name": "BLE广播间隔", "baseline": "1s", "values": ["0.1s","0.5s","1s","2s","5s"],
             "life_days": [200, 500, 750, 900, 1050], "color": "FF6F00"},
            {"name": "TX功率", "baseline": "0dBm", "values": ["-20dBm","-10dBm","0dBm","+4dBm","+8dBm"],
             "life_days": [900, 820, 750, 650, 520], "color": "C62828"},
        ]
    """
    slide = prs.slides.add_slide(prs.slide_layouts[6])
    add_background(slide, THEME["bg_white"])
    add_shape(slide, Inches(0), Inches(0), SLIDE_WIDTH, Inches(0.06), THEME["primary"])
    add_text_box(slide, MARGIN, Inches(0.35), Inches(11), Inches(0.8),
                 title, font_size=28, font_color=THEME["primary"], bold=True)
    add_shape(slide, MARGIN, Inches(1.15), Inches(1.5), Inches(0.04), THEME["accent"])
    add_text_box(slide, MARGIN, Inches(1.3), Inches(10), Inches(0.3),
                 "关键参数变化对电池寿命的影响 (其他参数保持不变)",
                 font_size=12, font_color="888888")

    # 每个参数一行水平柱状图
    chart_left = Inches(3.5)
    chart_w = Inches(8.5)
    
    for p_idx, param in enumerate(parameters):
        y_base = Inches(2.0) + p_idx * Inches(1.7)
        color = param.get("color", THEME["secondary"])
        max_life = max(param["life_days"])
        
        # 参数名
        add_text_box(slide, MARGIN, y_base, Inches(2.8), Inches(0.35),
                     param["name"], font_size=14, font_color=THEME["text_dark"], bold=True)
        add_text_box(slide, MARGIN, y_base + Inches(0.3), Inches(2.8), Inches(0.25),
                     f'基准: {param["baseline"]}',
                     font_size=10, font_color="999999")
        
        # 各值的柱状图
        n = len(param["values"])
        bar_h = Inches(0.22)
        for i, (val, life) in enumerate(zip(param["values"], param["life_days"])):
            by = y_base + i * (bar_h + Inches(0.06))
            bw = chart_w * life / max_life if max_life > 0 else 0
            
            is_baseline = (val == param["baseline"])
            bar_color = color if is_baseline else "BDBDBD"
            
            add_text_box(slide, chart_left - Inches(0.7), by, Inches(0.65), bar_h,
                         val, font_size=9, font_color=THEME["text_dark"],
                         alignment=PP_ALIGN.RIGHT)
            add_shape(slide, chart_left, by, max(Inches(0.05), bw), bar_h,
                      bar_color, radius=True)
            add_text_box(slide, chart_left + bw + Inches(0.1), by, Inches(1), bar_h,
                         f'{life}天', font_size=9, font_color=bar_color, bold=True)

    return slide


def make_optimization_slide(prs, title, optimizations):
    """功耗优化建议页 - 每条建议 + 预计节省 + 优先级
    
    Args:
        optimizations: [
            {"category": "MCU", "action": "Sleep→Stop模式优化",
             "save_uA": 150, "save_pct": 12,
             "effort": "低", "priority": "P0", "color": "43A047"},
            {"category": "无线", "action": "BLE广播间隔从0.5s改为2s",
             "save_uA": 80, "save_pct": 6.5,
             "effort": "低", "priority": "P0", "color": "FF6F00"},
            ...
        ]
    """
    slide = prs.slides.add_slide(prs.slide_layouts[6])
    add_background(slide, THEME["bg_white"])
    add_shape(slide, Inches(0), Inches(0), SLIDE_WIDTH, Inches(0.06), THEME["primary"])
    add_text_box(slide, MARGIN, Inches(0.35), Inches(11), Inches(0.8),
                 title, font_size=28, font_color=THEME["primary"], bold=True)
    add_shape(slide, MARGIN, Inches(1.15), Inches(1.5), Inches(0.04), THEME["accent"])

    # 表格
    headers = ["优先级", "类别", "优化措施", "预计节省", "节省占比", "实施难度"]
    n_rows = len(optimizations) + 2  # +header +total
    n_cols = len(headers)
    
    table_w = SLIDE_WIDTH - 2 * MARGIN
    table_h = min(Inches(5.5), Inches(0.45) * n_rows)
    
    table_shape = slide.shapes.add_table(n_rows, n_cols, MARGIN, Inches(1.5),
                                          table_w, table_h)
    table = table_shape.table
    
    for j, h in enumerate(headers):
        cell = table.cell(0, j)
        cell.text = h
        for p in cell.text_frame.paragraphs:
            p.font.size = Pt(11)
            p.font.bold = True
            p.font.color.rgb = hex_to_rgb("FFFFFF")
            p.alignment = PP_ALIGN.CENTER
        cell.fill.solid()
        cell.fill.fore_color.rgb = hex_to_rgb(THEME["primary"])
    
    total_save = 0
    for i, opt in enumerate(optimizations):
        row = i + 1
        table.cell(row, 0).text = opt["priority"]
        table.cell(row, 1).text = opt["category"]
        table.cell(row, 2).text = opt["action"]
        table.cell(row, 3).text = f'{opt["save_uA"]} μA'
        table.cell(row, 4).text = f'{opt["save_pct"]:.1f}%'
        table.cell(row, 5).text = opt["effort"]
        total_save += opt["save_uA"]
        
        for j in range(n_cols):
            for p in table.cell(row, j).text_frame.paragraphs:
                p.font.size = Pt(10)
                if j == 0:
                    p.font.bold = True
                    p.font.color.rgb = hex_to_rgb(
                        THEME.get("danger","C62828") if "P0" in opt["priority"]
                        else THEME["accent"] if "P1" in opt["priority"]
                        else "999999"
                    )
            if i % 2 == 1:
                table.cell(row, j).fill.solid()
                table.cell(row, j).fill.fore_color.rgb = hex_to_rgb(THEME["bg_light"])
    
    # 合计行
    last = n_rows - 1
    table.cell(last, 0).text = ""
    table.cell(last, 1).text = ""
    table.cell(last, 2).text = "总计可优化"
    table.cell(last, 3).text = f'{total_save} μA'
    table.cell(last, 4).text = f'{sum(o["save_pct"] for o in optimizations):.1f}%'
    table.cell(last, 5).text = ""
    for j in range(n_cols):
        for p in table.cell(last, j).text_frame.paragraphs:
            p.font.size = Pt(11)
            p.font.bold = True
        table.cell(last, j).fill.solid()
        table.cell(last, j).fill.fore_color.rgb = hex_to_rgb(THEME["bg_light"])
    
    return slide
```

### Phase 6: 执行与验证

1. **运行脚本**生成 `.pptx` 文件
2. **验证内容**：
   - 所有器件电流数据标注了来源（datasheet / 估算 / 实测）
   - 电流单位统一 (μA)，功率单位统一 (μW / mW)
   - 电池寿命计算过程可追溯
   - 优化建议有量化的节省效果
3. **输出文件路径和摘要**

### Phase 7: 交付

向用户交付：
1. ✅ 生成的 `.pptx` 功耗分析报告
2. 📊 功耗分析摘要（平均电流 / 电池寿命 / 达标评估）
3. 💡 TOP 3 优化建议及预期节省效果
4. ⚠️ 主要功耗风险及建议的实测验证计划

---

## 布局选择策略

| 内容特征 | 推荐布局 | 函数 |
|---------|---------|------|
| 报告首页 | 封面页 | `make_cover_slide()` |
| 章节目录 | 目录页 | `make_toc_slide()` |
| 新章节 | 过渡页 | `make_section_slide()` |
| 文字要点 | 标准内容页 | `make_content_slide()` |
| 系统架构 + 电源树 | 架构框图页 | `make_block_diagram_slide()` |
| 器件清单 | 规格表格页 | `make_spec_table_slide()` |
| 场景时序全景 | 场景概览页 | `make_scenario_overview_slide()` |
| 单场景电流分解 | 场景详情页 | `make_current_waterfall_slide()` |
| 各模块功耗占比 | 功耗饼图页 | `make_power_pie_slide()` |
| 场景间功耗对比 | 功耗对比页 | `make_spec_table_slide()` (复用) |
| 电池寿命预估 | 寿命估算页 | `make_battery_life_slide()` |
| 参数敏感度分析 | 敏感度分析页 | `make_sensitivity_slide()` |
| 优化建议列表 | 优化建议页 | `make_optimization_slide()` |
| 优化前后对比 | Before/After页 | `make_before_after_slide()` |
| 功耗风险评估 | 风险矩阵页 | `make_risk_matrix_slide()` |
| 关键数据 | 数据高亮页 | `make_data_highlight_slide()` |
| 结论总结 | 总结页 | `make_summary_slide()` |
| 报告结尾 | 感谢页 | `make_thank_you_slide()` |

---

## 功耗分析质量检查清单

### 数据准确性
- [ ] 所有器件电流数据标注了来源（datasheet 页码 / 实测 / 估算）
- [ ] MCU 功耗数据与具体型号 datasheet 一致
- [ ] 无线模块功耗区分了 TX / RX / Sleep 各模式
- [ ] 传感器功耗区分了测量态和待机态
- [ ] LDO/DC-DC 效率已计入系统功耗
- [ ] 电池容量使用标称值而非理论最大值

### 模型完整性
- [ ] 所有运行场景都已定义（含最坏情况场景）
- [ ] 每个场景的触发条件和持续时间明确
- [ ] 场景间的切换功耗已考虑（唤醒时间×唤醒电流）
- [ ] 非活跃期间的基础电流已包含（Stop/Standby 基底电流）
- [ ] 电源路径上所有损耗都已计入（LDO Iq, 分压器, 上拉电阻等）
- [ ] 自放电已纳入电池寿命计算

### 工程余量
- [ ] 功耗余量策略已根据项目阶段确定
- [ ] 温度影响已考虑（高温下漏电流增加）
- [ ] 电压变化影响已考虑（电池放电曲线）
- [ ] 量产批次差异已纳入最差情况估算
- [ ] 软件未优化余量已标注

### 报告设计
- [ ] 使用 `power` 主题配色（能源绿色系）
- [ ] 达标/未达标使用绿/红色醒目标记
- [ ] 功耗占比最大的 TOP3 器件有突出标注
- [ ] 电池寿命三档（最好/典型/最差）都有展示
- [ ] 优化建议有量化的节省效果和优先级排序
- [ ] 总页数在 25-35 页范围内
