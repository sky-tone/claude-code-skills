---
name: PPT Pro
description: 专业PPT制作技能，基于python-pptx生成结构清晰、设计精美的演示文稿
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
  Use when the user wants to create a PowerPoint presentation, make a PPT, 
  generate slides, build a deck, or asks for help with presentation design.
  Examples: 'help me make a PPT', 'create a presentation about X', 
  'generate slides for my project', '帮我做PPT', '制作演示文稿', '写个汇报PPT',
  '竞品拆机报告', '竞品分析PPT', 'teardown report', '问题回溯', '故障分析',
  'root cause analysis', 'RCA报告', '硬件方案分析', '技术架构分析', '硬件架构PPT'
argument-hint: "<topic> [--pages N] [--theme dark|light|corporate|tech|academic] [--lang zh|en]"
arguments:
  - topic
context: fork
agent: general-purpose
model: sonnet
effort: high
shell: bash
---

# 🎯 PPT Pro — 专业演示文稿制作技能

你是一位**资深演示文稿设计师 + 内容策划专家**，精通 `python-pptx` 库，擅长将用户需求转化为结构清晰、视觉精美、逻辑严密的 PowerPoint 演示文稿。

---

## 输入参数

- `$topic`: 用户给出的 PPT 主题 / 内容需求描述

---

## 核心原则

### 内容原则
1. **金字塔原则**：结论先行，自上而下地展开论证
2. **MECE 原则**：各页内容互不重叠、完全穷尽
3. **3×3 法则**：每页核心观点不超过 3 个，每个观点不超过 3 行
4. **故事弧线**：开场（Why）→ 主体（What/How）→ 收尾（So What）

### 设计原则
1. **对比**：通过大小、颜色、字重建立视觉层级
2. **对齐**：所有元素基于网格对齐，不允许随意漂移
3. **重复**：配色、字体、间距在全篇保持一致
4. **亲密**：相关内容物理上靠近，无关内容明确分离
5. **留白**：至少 30% 页面空白，拒绝信息轰炸

---

## 执行流程

### Phase 0: 环境准备

检查并安装 `python-pptx` 依赖:

```bash
pip install python-pptx Pillow 2>/dev/null || pip3 install python-pptx Pillow 2>/dev/null
```

确认安装成功:

```bash
python3 -c "from pptx import Presentation; from pptx.util import Inches, Pt, Emu; from pptx.dml.color import RGBColor; from pptx.enum.text import PP_ALIGN, MSO_ANCHOR; print('✅ python-pptx ready')"
```

### Phase 1: 需求分析与大纲设计

仔细分析用户需求 `$topic`，确定:

1. **PPT 类型判断**：
   | 类型 | 特征 | 页数建议 |
   |------|------|---------|
   | 工作汇报 | 总结成果、数据展示 | 12-18页 |
   | 项目方案 | 背景分析、方案对比 | 15-25页 |
   | 产品介绍 | 功能亮点、用户价值 | 10-15页 |
   | 培训教学 | 知识点、案例、练习 | 20-30页 |
   | 演讲发言 | 观点鲜明、节奏紧凑 | 8-15页 |
   | 学术答辩 | 研究背景、方法、结果 | 15-20页 |
   | 商业计划 | 市场、产品、团队、财务 | 15-25页 |
   | **竞品拆机报告** | BOM成本、PCB布局、器件选型对比 | 20-35页 |
   | **问题回溯报告** | 故障现象、根因分析、纠正预防 | 15-25页 |
   | **硬件技术架构分析** | 系统框图、关键器件、信号链路 | 20-35页 |

   ---

   #### 🔩 竞品拆机报告 — 标准大纲模板

   当判断为竞品拆机报告时，按以下结构组织：

   | 页序 | 页面类型 | 标题 | 内容要点 |
   |------|---------|------|----------|
   | 1 | 封面页 | XX产品竞品拆机分析报告 | 产品型号、分析日期、团队 |
   | 2 | 目录页 | CONTENTS | 全部章节导航 |
   | 3 | 章节页 | 01 拆机对象概览 | — |
   | 4 | 产品对比页 | 产品外观与规格对比 | 我司产品 vs 竞品：尺寸、重量、接口、外观照片描述 |
   | 5 | 参数表格页 | 核心规格参数对比 | 表格：性能指标、频率、功耗、精度等逐项对比 |
   | 6 | 章节页 | 02 整机拆解分析 | — |
   | 7 | 拆解步骤页 | 拆解过程记录 | 拆解步骤、螺丝数量/类型、卡扣结构、可维修性评估 |
   | 8 | 内容页 | 机械结构分析 | 外壳材质、模具工艺、散热结构、防护等级 |
   | 9 | 章节页 | 03 PCB & 电路分析 | — |
   | 10 | PCB布局页 | PCB布局总览 | 板层数、尺寸、布局分区图（描述各功能区位置） |
   | 11-13 | 器件标注页 | 正面/背面关键器件标注 | 主控、电源、射频、存储、传感器等标注说明 |
   | 14 | 章节页 | 04 BOM成本估算 | — |
   | 15 | BOM表格页 | 核心器件BOM清单 | 表格：器件类型、型号、厂商、封装、估价 |
   | 16 | 成本饼图页 | BOM成本构成分析 | 各模块成本占比：主控XX%、电源XX%、结构XX%... |
   | 17 | 数据高亮页 | 关键成本数据 | 总BOM估价、与我司成本差异、关键差价器件 |
   | 18 | 章节页 | 05 关键器件选型分析 | — |
   | 19-21 | 器件对比页 | 主控/电源/传感器选型对比 | 竞品选型 vs 我司选型：性能、成本、供应链 |
   | 22 | 章节页 | 06 设计亮点与不足 | — |
   | 23 | 双栏对比页 | 设计亮点 vs 设计不足 | 左：值得借鉴的设计；右：可改进的问题 |
   | 24 | 内容页 | 可借鉴设计要点 | 具体可应用到我司产品的改进点 |
   | 25 | 章节页 | 07 结论与建议 | — |
   | 26 | 总结页 | 核心发现总结 | 3-5条关键结论 |
   | 27 | 内容页 | 后续行动建议 | 针对我司产品的具体优化建议 |
   | 28 | 结束页 | THANK YOU | 联系方式 |

   ---

   #### 🔍 问题回溯报告 — 标准大纲模板

   当判断为问题回溯/故障分析/RCA报告时，按以下结构组织：

   | 页序 | 页面类型 | 标题 | 内容要点 |
   |------|---------|------|----------|
   | 1 | 封面页 | XX问题回溯分析报告 | 问题编号、严重等级、日期 |
   | 2 | 目录页 | CONTENTS | 全部章节导航 |
   | 3 | 数据高亮页 | 问题速览 | 发生时间、影响范围、严重等级、当前状态 |
   | 4 | 章节页 | 01 问题描述 | — |
   | 5 | 内容页 | 故障现象 | 具体现象描述、复现条件、发生频率 |
   | 6 | 内容页 | 影响范围评估 | 影响产品/批次/客户数量、经济损失估算 |
   | 7 | 时间轴页 | 问题时间线 | 首次发现→上报→临时措施→根因定位→修复验证 |
   | 8 | 章节页 | 02 临时遏制措施 | — |
   | 9 | 内容页 | 紧急遏制方案 | 已采取的临时措施、效果评估 |
   | 10 | 章节页 | 03 根因分析 | — |
   | 11 | 鱼骨图页 | 鱼骨图/5Why分析 | 人、机、料、法、环、测六大维度 |
   | 12-14 | 分析步骤页 | 根因定位过程 | 实验数据、对比测试、仿真结果、波形/日志截图描述 |
   | 15 | 数据高亮页 | 根本原因确认 | 核心根因 + 直接原因 + 间接原因 |
   | 16 | 章节页 | 04 纠正措施 | — |
   | 17 | 内容页 | 短期纠正措施 | 设计修改、参数调整、物料替代等 |
   | 18 | 内容页 | 长期预防措施 | 流程改进、设计规范更新、检测手段加强 |
   | 19 | 章节页 | 05 验证与效果确认 | — |
   | 20 | 验证矩阵页 | 验证方案与结果 | 验证项目、方法、判定标准、结果(PASS/FAIL) |
   | 21 | 数据对比页 | 修复前后数据对比 | 关键指标修复前 vs 修复后 |
   | 22 | 章节页 | 06 经验教训 | — |
   | 23 | 卡片页 | Lessons Learned | 设计/流程/测试/供应链维度的经验总结 |
   | 24 | 总结页 | 总结与后续跟踪 | 关闭条件、跟踪责任人、检查节点 |
   | 25 | 结束页 | THANK YOU | — |

   ---

   #### 🖥️ 硬件方案技术架构分析报告 — 标准大纲模板

   当判断为硬件方案/技术架构分析报告时，按以下结构组织：

   | 页序 | 页面类型 | 标题 | 内容要点 |
   |------|---------|------|----------|
   | 1 | 封面页 | XX硬件方案技术架构分析 | 项目名称、版本、日期 |
   | 2 | 目录页 | CONTENTS | 全部章节导航 |
   | 3 | 章节页 | 01 需求与约束 | — |
   | 4 | 内容页 | 系统需求规格 | 功能需求、性能指标、环境要求、认证要求 |
   | 5 | 卡片页 | 关键设计约束 | 成本目标、尺寸限制、功耗预算、温度范围 |
   | 6 | 章节页 | 02 系统架构总览 | — |
   | 7 | 架构框图页 | 系统架构框图 | 顶层功能模块划分及互联关系（文字描述各模块位置和连接） |
   | 8 | 内容页 | 模块功能说明 | 各功能模块的职责定义 |
   | 9 | 章节页 | 03 核心器件选型 | — |
   | 10 | 器件选型页 | 主控/SoC选型分析 | 候选方案对比：架构、主频、外设、功耗、价格、供应状态 |
   | 11 | 器件选型页 | 电源方案选型 | 电源树设计、LDO/DCDC选择、效率、纹波要求 |
   | 12 | 器件选型页 | 存储方案选型 | Flash/DDR/eMMC 容量、速率、颗粒选择 |
   | 13 | 器件选型页 | 通信接口选型 | WiFi/BT/4G/Ethernet等，模组 vs 芯片方案对比 |
   | 14 | 器件选型页 | 传感器/ADC/DAC选型 | 精度、采样率、通道数、接口类型 |
   | 15 | 章节页 | 04 电源架构设计 | — |
   | 16 | 电源树页 | 电源树架构图 | 输入→各级转换→各模块供电关系（层级式描述） |
   | 17 | 内容页 | 功耗预算分析 | 各模块功耗估算、总功耗、散热方案 |
   | 18 | 数据高亮页 | 电源关键指标 | 效率、纹波、待机功耗、峰值电流 |
   | 19 | 章节页 | 05 关键信号链路分析 | — |
   | 20 | 信号链路页 | 高速信号设计 | DDR/USB/HDMI/PCIe等：阻抗、等长、参考层、过孔 |
   | 21 | 信号链路页 | 模拟信号链路 | ADC/DAC通路：信号调理、滤波、增益、SNR |
   | 22 | 信号链路页 | 射频链路设计 | 天线→滤波器→PA/LNA→收发器，链路预算 |
   | 23 | 章节页 | 06 PCB设计要点 | — |
   | 24 | 内容页 | PCB叠层与布局规划 | 板层设计、阻抗控制、关键区域布局原则 |
   | 25 | 内容页 | EMC/EMI设计策略 | 接地策略、屏蔽、滤波、ESD防护 |
   | 26 | 章节页 | 07 可靠性与测试 | — |
   | 27 | 验证矩阵页 | DV/PV测试计划 | 测试项、标准、样本数、判定准则 |
   | 28 | 内容页 | 关键风险与缓解 | 风险矩阵：技术风险、供应链风险、进度风险 |
   | 29 | 章节页 | 08 成本与进度 | — |
   | 30 | BOM表格页 | BOM成本估算 | 关键器件清单与总成本 |
   | 31 | 时间轴页 | 项目里程碑 | 原理图→PCB→打样→调试→DVT→量产 |
   | 32 | 总结页 | 方案总结与决策建议 | 3-5条核心结论 |
   | 33 | 结束页 | THANK YOU | — |

2. **大纲结构**：设计完整的幻灯片大纲，包括：
   - 每一页的标题和核心信息
   - 页面类型（标题页/目录页/内容页/图表页/总结页等）
   - 预设的布局方案

3. **将大纲展示给用户确认**后再继续

### Phase 2: 主题配色方案

根据 PPT 类型和用户偏好，从以下预设方案中选择或定制：

```python
THEMES = {
    "corporate": {  # 商务正式
        "primary":    "1F4E79",  # 深蓝
        "secondary":  "2E75B6",  # 中蓝
        "accent":     "ED7D31",  # 橙色点缀
        "text_dark":  "2D2D2D",  # 正文深色
        "text_light": "FFFFFF",  # 白色文字
        "bg_dark":    "1F4E79",  # 深色背景
        "bg_light":   "F2F2F2",  # 浅灰背景
        "bg_white":   "FFFFFF",  # 白色背景
    },
    "tech": {  # 科技感
        "primary":    "0D1B2A",  # 深空蓝
        "secondary":  "1B9AAA",  # 青色
        "accent":     "06D6A0",  # 薄荷绿
        "text_dark":  "E0E1DD",  # 浅色文字
        "text_light": "FFFFFF",
        "bg_dark":    "0D1B2A",
        "bg_light":   "1B263B",
        "bg_white":   "FFFFFF",
    },
    "academic": {  # 学术简洁
        "primary":    "8B0000",  # 深红
        "secondary":  "333333",  # 深灰
        "accent":     "C41E3A",  # 中红
        "text_dark":  "333333",
        "text_light": "FFFFFF",
        "bg_dark":    "8B0000",
        "bg_light":   "FBF7F0",  # 米白
        "bg_white":   "FFFFFF",
    },
    "modern_dark": {  # 现代暗黑
        "primary":    "BB86FC",  # 紫色
        "secondary":  "03DAC6",  # 青色
        "accent":     "CF6679",  # 粉红
        "text_dark":  "E1E1E1",
        "text_light": "FFFFFF",
        "bg_dark":    "121212",
        "bg_light":   "1E1E1E",
        "bg_white":   "2D2D2D",
    },
    "minimal": {  # 极简白
        "primary":    "000000",  # 纯黑
        "secondary":  "666666",  # 灰色
        "accent":     "FF6B35",  # 橙色
        "text_dark":  "1A1A1A",
        "text_light": "FFFFFF",
        "bg_dark":    "1A1A1A",
        "bg_light":   "F8F8F8",
        "bg_white":   "FFFFFF",
    },
    "nature": {  # 自然清新
        "primary":    "2D6A4F",  # 森林绿
        "secondary":  "52B788",  # 翠绿
        "accent":     "D4A373",  # 暖棕
        "text_dark":  "2D2D2D",
        "text_light": "FFFFFF",
        "bg_dark":    "2D6A4F",
        "bg_light":   "F0F7F4",
        "bg_white":   "FFFFFF",
    },
    "engineering": {  # 工程蓝灰 — 适合竞品拆机、硬件架构分析
        "primary":    "2C3E50",  # 工程深蓝灰
        "secondary":  "3498DB",  # 信号蓝
        "accent":     "E67E22",  # 警示橙
        "text_dark":  "2C3E50",
        "text_light": "FFFFFF",
        "bg_dark":    "2C3E50",
        "bg_light":   "ECF0F1",  # 工程灰白
        "bg_white":   "FFFFFF",
        "success":    "27AE60",  # PASS绿（验证通过）
        "danger":     "E74C3C",  # FAIL红（验证失败/故障）
        "warning":    "F39C12",  # 警告黄
    },
    "rca_red": {  # 问题回溯红 — 适合问题回溯/RCA报告
        "primary":    "C0392B",  # 警报红
        "secondary":  "2C3E50",  # 沉稳灰蓝
        "accent":     "F39C12",  # 警告黄
        "text_dark":  "2D2D2D",
        "text_light": "FFFFFF",
        "bg_dark":    "1A1A2E",  # 深夜蓝黑
        "bg_light":   "FDF2F2",  # 浅粉警示
        "bg_white":   "FFFFFF",
        "success":    "27AE60",  # 已修复/PASS
        "danger":     "E74C3C",  # 未修复/FAIL
        "warning":    "F39C12",  # 待观察
    },
}
```

### Phase 3: 生成 PPT Python 脚本

创建一个完整的 Python 脚本，使用 `python-pptx` 生成 PPT。脚本必须遵循以下架构：

```python
#!/usr/bin/env python3
"""PPT Generator - Professional Presentation Builder"""

from pptx import Presentation
from pptx.util import Inches, Pt, Emu, Cm
from pptx.dml.color import RGBColor
from pptx.enum.text import PP_ALIGN, MSO_ANCHOR, MSO_AUTO_SIZE
from pptx.enum.shapes import MSO_SHAPE
from pptx.enum.chart import XL_CHART_TYPE
import os

# ============================================================
# 全局配置
# ============================================================
SLIDE_WIDTH  = Inches(13.333)   # 16:9 宽屏
SLIDE_HEIGHT = Inches(7.5)
MARGIN       = Inches(0.6)      # 页面边距

THEME = { ... }  # 从 Phase 2 选定的配色方案

# 字体配置（中英双语）
FONT_CN = "微软雅黑"   # 中文正文
FONT_EN = "Segoe UI"   # 英文正文
FONT_TITLE_CN = "微软雅黑"  # 中文标题
FONT_TITLE_EN = "Segoe UI Semibold"  # 英文标题
FONT_MONO = "Consolas"  # 等宽字体

# ============================================================
# 工具函数
# ============================================================

def hex_to_rgb(hex_str):
    """将十六进制颜色转为 RGBColor"""
    return RGBColor(int(hex_str[:2], 16), int(hex_str[2:4], 16), int(hex_str[4:6], 16))

def add_background(slide, color_hex):
    """设置幻灯片背景色"""
    background = slide.background
    fill = background.fill
    fill.solid()
    fill.fore_color.rgb = hex_to_rgb(color_hex)

def add_shape(slide, left, top, width, height, fill_hex=None, border_hex=None, radius=None):
    """添加矩形形状（支持圆角）"""
    shape_type = MSO_SHAPE.ROUNDED_RECTANGLE if radius else MSO_SHAPE.RECTANGLE
    shape = slide.shapes.add_shape(shape_type, left, top, width, height)
    if fill_hex:
        shape.fill.solid()
        shape.fill.fore_color.rgb = hex_to_rgb(fill_hex)
    else:
        shape.fill.background()  # 透明
    if border_hex:
        shape.line.color.rgb = hex_to_rgb(border_hex)
        shape.line.width = Pt(1)
    else:
        shape.line.fill.background()  # 无边框
    return shape

def add_text_box(slide, left, top, width, height, text, 
                 font_size=14, font_color="2D2D2D", font_name=None,
                 bold=False, italic=False, alignment=PP_ALIGN.LEFT,
                 vertical=MSO_ANCHOR.TOP, line_spacing=1.3):
    """添加文本框（核心排版函数）"""
    txBox = slide.shapes.add_textbox(left, top, width, height)
    tf = txBox.text_frame
    tf.word_wrap = True
    tf.auto_size = MSO_AUTO_SIZE.NONE

    p = tf.paragraphs[0]
    p.text = text
    p.font.size = Pt(font_size)
    p.font.color.rgb = hex_to_rgb(font_color)
    p.font.bold = bold
    p.font.name = font_name or (FONT_CN if any('\u4e00' <= c <= '\u9fff' for c in text) else FONT_EN)
    p.alignment = alignment
    p.space_after = Pt(0)
    p.line_spacing = Pt(font_size * line_spacing)

    tf.paragraphs[0].font.italic = italic
    # 垂直居中
    txBox.text_frame.paragraphs[0]
    # python-pptx 不直接支持 vertical anchor on textbox，
    # 对 shape 类型的文本可设置

    return txBox

def add_bullet_list(slide, left, top, width, height, items,
                    font_size=13, font_color="2D2D2D", bullet_color="2E75B6",
                    line_spacing=1.6):
    """添加项目符号列表"""
    txBox = slide.shapes.add_textbox(left, top, width, height)
    tf = txBox.text_frame
    tf.word_wrap = True
    
    for i, item in enumerate(items):
        p = tf.paragraphs[0] if i == 0 else tf.add_paragraph()
        p.text = item
        p.font.size = Pt(font_size)
        p.font.color.rgb = hex_to_rgb(font_color)
        p.font.name = FONT_CN if any('\u4e00' <= c <= '\u9fff' for c in item) else FONT_EN
        p.level = 0
        p.line_spacing = Pt(font_size * line_spacing)
        p.space_before = Pt(4) if i > 0 else Pt(0)
        # 设置项目符号
        pPr = p._pPr
        if pPr is None:
            from pptx.oxml.ns import qn
            pPr = p._p.get_or_add_pPr()
        # 使用圆点符号
        from pptx.oxml.ns import qn
        from lxml import etree
        buNone = pPr.find(qn('a:buNone'))
        if buNone is not None:
            pPr.remove(buNone)
        buChar = etree.SubElement(pPr, qn('a:buChar'))
        buChar.set('char', '●')
        buClr = etree.SubElement(pPr, qn('a:buClr'))
        srgbClr = etree.SubElement(buClr, qn('a:srgbClr'))
        srgbClr.set('val', bullet_color)
    
    return txBox

def add_line(slide, start_x, start_y, end_x, end_y, color_hex, width_pt=1.5):
    """添加线条"""
    connector = slide.shapes.add_connector(
        1,  # MSO_CONNECTOR.STRAIGHT
        start_x, start_y, end_x, end_y
    )
    connector.line.color.rgb = hex_to_rgb(color_hex)
    connector.line.width = Pt(width_pt)
    return connector

def add_icon_circle(slide, cx, cy, radius, color_hex, text="", text_color="FFFFFF", text_size=16):
    """添加圆形图标（带可选文字/emoji）"""
    left = cx - radius
    top = cy - radius
    shape = slide.shapes.add_shape(MSO_SHAPE.OVAL, left, top, radius*2, radius*2)
    shape.fill.solid()
    shape.fill.fore_color.rgb = hex_to_rgb(color_hex)
    shape.line.fill.background()
    if text:
        tf = shape.text_frame
        tf.word_wrap = False
        p = tf.paragraphs[0]
        p.text = text
        p.font.size = Pt(text_size)
        p.font.color.rgb = hex_to_rgb(text_color)
        p.font.bold = True
        p.alignment = PP_ALIGN.CENTER
        tf.paragraphs[0].alignment = PP_ALIGN.CENTER
    return shape

# ============================================================
# 页面布局模板
# ============================================================

def make_cover_slide(prs, title, subtitle="", date=""):
    """封面页 - 大标题 + 副标题 + 装饰"""
    slide = prs.slides.add_slide(prs.slide_layouts[6])  # 空白布局
    add_background(slide, THEME["bg_dark"])
    
    # 装饰色块
    add_shape(slide, Inches(0), Inches(5.8), SLIDE_WIDTH, Inches(1.7), THEME["accent"])
    add_shape(slide, Inches(0), Inches(5.6), Inches(4), Inches(0.15), THEME["secondary"])
    
    # 主标题
    add_text_box(slide, MARGIN, Inches(1.8), Inches(11), Inches(2),
                 title, font_size=44, font_color=THEME["text_light"],
                 bold=True, alignment=PP_ALIGN.LEFT)
    
    # 副标题
    if subtitle:
        add_text_box(slide, MARGIN, Inches(3.8), Inches(10), Inches(1),
                     subtitle, font_size=20, font_color=THEME["text_light"],
                     alignment=PP_ALIGN.LEFT)
    
    # 日期
    if date:
        add_text_box(slide, MARGIN, Inches(6.1), Inches(4), Inches(0.5),
                     date, font_size=14, font_color=THEME["bg_dark"],
                     bold=True, alignment=PP_ALIGN.LEFT)
    
    return slide

def make_section_slide(prs, section_num, section_title, section_subtitle=""):
    """章节过渡页 - 大数字 + 标题"""
    slide = prs.slides.add_slide(prs.slide_layouts[6])
    add_background(slide, THEME["bg_dark"])
    
    # 大数字
    add_text_box(slide, MARGIN, Inches(1.2), Inches(3), Inches(2.5),
                 f"{section_num:02d}", font_size=96, font_color=THEME["accent"],
                 bold=True, alignment=PP_ALIGN.LEFT)
    
    # 分隔线
    add_shape(slide, MARGIN, Inches(3.6), Inches(2), Inches(0.06), THEME["accent"])
    
    # 标题
    add_text_box(slide, MARGIN, Inches(3.9), Inches(10), Inches(1.5),
                 section_title, font_size=36, font_color=THEME["text_light"],
                 bold=True, alignment=PP_ALIGN.LEFT)
    
    if section_subtitle:
        add_text_box(slide, MARGIN, Inches(5.2), Inches(10), Inches(1),
                     section_subtitle, font_size=16, font_color=THEME["secondary"],
                     alignment=PP_ALIGN.LEFT)
    
    return slide

def make_content_slide(prs, title, bullets, footer_note=""):
    """标准内容页 - 标题 + 要点列表"""
    slide = prs.slides.add_slide(prs.slide_layouts[6])
    add_background(slide, THEME["bg_white"])
    
    # 顶部色条
    add_shape(slide, Inches(0), Inches(0), SLIDE_WIDTH, Inches(0.06), THEME["primary"])
    
    # 标题
    add_text_box(slide, MARGIN, Inches(0.35), Inches(11), Inches(0.8),
                 title, font_size=28, font_color=THEME["primary"],
                 bold=True, alignment=PP_ALIGN.LEFT)
    
    # 标题下分隔线
    add_shape(slide, MARGIN, Inches(1.15), Inches(1.5), Inches(0.04), THEME["accent"])
    
    # 内容列表
    add_bullet_list(slide, Inches(0.9), Inches(1.5), Inches(11), Inches(5),
                    bullets, font_size=16, font_color=THEME["text_dark"],
                    bullet_color=THEME["accent"])
    
    # 页脚
    if footer_note:
        add_text_box(slide, MARGIN, Inches(6.9), Inches(12), Inches(0.4),
                     footer_note, font_size=9, font_color="999999",
                     italic=True)
    
    return slide

def make_two_column_slide(prs, title, left_title, left_items, right_title, right_items):
    """双栏对比页"""
    slide = prs.slides.add_slide(prs.slide_layouts[6])
    add_background(slide, THEME["bg_white"])
    
    # 顶部色条
    add_shape(slide, Inches(0), Inches(0), SLIDE_WIDTH, Inches(0.06), THEME["primary"])
    
    # 标题
    add_text_box(slide, MARGIN, Inches(0.35), Inches(11), Inches(0.8),
                 title, font_size=28, font_color=THEME["primary"],
                 bold=True)
    add_shape(slide, MARGIN, Inches(1.15), Inches(1.5), Inches(0.04), THEME["accent"])
    
    col_w = Inches(5.5)
    
    # 左栏
    add_shape(slide, Inches(0.4), Inches(1.5), col_w, Inches(5.2), 
              THEME["bg_light"], radius=True)
    add_text_box(slide, Inches(0.7), Inches(1.7), Inches(5), Inches(0.6),
                 left_title, font_size=18, font_color=THEME["primary"], bold=True)
    add_bullet_list(slide, Inches(0.9), Inches(2.4), Inches(4.6), Inches(4),
                    left_items, font_size=14, bullet_color=THEME["secondary"])
    
    # 右栏
    add_shape(slide, Inches(6.8), Inches(1.5), col_w, Inches(5.2),
              THEME["bg_light"], radius=True)
    add_text_box(slide, Inches(7.1), Inches(1.7), Inches(5), Inches(0.6),
                 right_title, font_size=18, font_color=THEME["primary"], bold=True)
    add_bullet_list(slide, Inches(7.3), Inches(2.4), Inches(4.6), Inches(4),
                    right_items, font_size=14, bullet_color=THEME["accent"])
    
    return slide

def make_cards_slide(prs, title, cards):
    """卡片网格页 - 3~4 个等宽卡片"""
    slide = prs.slides.add_slide(prs.slide_layouts[6])
    add_background(slide, THEME["bg_white"])
    
    add_shape(slide, Inches(0), Inches(0), SLIDE_WIDTH, Inches(0.06), THEME["primary"])
    add_text_box(slide, MARGIN, Inches(0.35), Inches(11), Inches(0.8),
                 title, font_size=28, font_color=THEME["primary"], bold=True)
    add_shape(slide, MARGIN, Inches(1.15), Inches(1.5), Inches(0.04), THEME["accent"])
    
    n = len(cards)
    gap = Inches(0.3)
    total_w = SLIDE_WIDTH - 2 * MARGIN - gap * (n - 1)
    card_w = total_w // n
    card_h = Inches(4.8)
    top = Inches(1.6)
    
    colors = [THEME["primary"], THEME["secondary"], THEME["accent"], THEME.get("text_dark", "333333")]
    
    for i, card in enumerate(cards):
        left = MARGIN + i * (card_w + gap)
        # 卡片背景
        add_shape(slide, left, top, card_w, card_h, THEME["bg_light"], radius=True)
        # 顶部色条
        add_shape(slide, left, top, card_w, Inches(0.08), colors[i % len(colors)])
        # 图标圆圈
        cx = left + card_w // 2
        add_icon_circle(slide, cx, top + Inches(0.7), Inches(0.4),
                       colors[i % len(colors)], card.get("icon", ""), text_size=18)
        # 卡片标题
        add_text_box(slide, left + Inches(0.2), top + Inches(1.3), card_w - Inches(0.4), Inches(0.6),
                     card["title"], font_size=16, font_color=THEME["text_dark"],
                     bold=True, alignment=PP_ALIGN.CENTER)
        # 卡片内容
        add_text_box(slide, left + Inches(0.3), top + Inches(2.0), card_w - Inches(0.6), Inches(2.5),
                     card["body"], font_size=12, font_color="666666",
                     alignment=PP_ALIGN.CENTER, line_spacing=1.5)
    
    return slide

def make_timeline_slide(prs, title, events):
    """时间轴页 - 水平时间线"""
    slide = prs.slides.add_slide(prs.slide_layouts[6])
    add_background(slide, THEME["bg_white"])
    
    add_shape(slide, Inches(0), Inches(0), SLIDE_WIDTH, Inches(0.06), THEME["primary"])
    add_text_box(slide, MARGIN, Inches(0.35), Inches(11), Inches(0.8),
                 title, font_size=28, font_color=THEME["primary"], bold=True)
    add_shape(slide, MARGIN, Inches(1.15), Inches(1.5), Inches(0.04), THEME["accent"])
    
    # 水平线
    line_y = Inches(3.8)
    add_shape(slide, Inches(1), line_y, Inches(11.3), Inches(0.04), THEME["primary"])
    
    n = len(events)
    for i, evt in enumerate(events):
        cx = Inches(1.5) + i * (Inches(10.3) // max(n-1, 1)) if n > 1 else SLIDE_WIDTH // 2
        # 节点圆
        add_icon_circle(slide, cx, line_y + Inches(0.02), Inches(0.2), THEME["accent"])
        # 上方文字（时间）
        add_text_box(slide, cx - Inches(1), line_y - Inches(1.2), Inches(2), Inches(0.4),
                     evt.get("time", ""), font_size=12, font_color=THEME["accent"],
                     bold=True, alignment=PP_ALIGN.CENTER)
        # 上方标题
        add_text_box(slide, cx - Inches(1), line_y - Inches(0.8), Inches(2), Inches(0.6),
                     evt["title"], font_size=13, font_color=THEME["text_dark"],
                     bold=True, alignment=PP_ALIGN.CENTER)
        # 下方描述
        add_text_box(slide, cx - Inches(1.2), line_y + Inches(0.5), Inches(2.4), Inches(1.5),
                     evt.get("desc", ""), font_size=11, font_color="666666",
                     alignment=PP_ALIGN.CENTER)
    
    return slide

def make_data_highlight_slide(prs, title, metrics):
    """数据高亮页 - 大数字展示关键指标"""
    slide = prs.slides.add_slide(prs.slide_layouts[6])
    add_background(slide, THEME["bg_dark"])
    
    add_text_box(slide, MARGIN, Inches(0.5), Inches(11), Inches(0.8),
                 title, font_size=28, font_color=THEME["text_light"], bold=True)
    add_shape(slide, MARGIN, Inches(1.3), Inches(1.5), Inches(0.04), THEME["accent"])
    
    n = len(metrics)
    for i, m in enumerate(metrics):
        cx = MARGIN + i * (Inches(12) // n) + Inches(12) // n // 2
        # 大数字
        add_text_box(slide, cx - Inches(2), Inches(2.5), Inches(4), Inches(1.8),
                     m["value"], font_size=56, font_color=THEME["accent"],
                     bold=True, alignment=PP_ALIGN.CENTER)
        # 指标名
        add_text_box(slide, cx - Inches(2), Inches(4.3), Inches(4), Inches(0.5),
                     m["label"], font_size=16, font_color=THEME["text_light"],
                     alignment=PP_ALIGN.CENTER)
        # 说明
        if m.get("note"):
            add_text_box(slide, cx - Inches(2), Inches(4.9), Inches(4), Inches(0.8),
                         m["note"], font_size=11, font_color=THEME["secondary"],
                         alignment=PP_ALIGN.CENTER)
    
    return slide

def make_summary_slide(prs, title, key_points, cta=""):
    """总结页 - 核心要点回顾 + CTA"""
    slide = prs.slides.add_slide(prs.slide_layouts[6])
    add_background(slide, THEME["bg_dark"])
    
    add_text_box(slide, MARGIN, Inches(0.8), Inches(11), Inches(1),
                 title, font_size=36, font_color=THEME["text_light"], bold=True)
    add_shape(slide, MARGIN, Inches(1.8), Inches(2), Inches(0.05), THEME["accent"])
    
    for i, point in enumerate(key_points):
        y = Inches(2.3) + i * Inches(0.9)
        add_icon_circle(slide, Inches(1.2), y + Inches(0.15), Inches(0.2), THEME["accent"],
                       str(i+1), text_size=12)
        add_text_box(slide, Inches(1.8), y, Inches(10), Inches(0.7),
                     point, font_size=18, font_color=THEME["text_light"])
    
    if cta:
        add_shape(slide, Inches(4), Inches(5.8), Inches(5.3), Inches(0.8), THEME["accent"], radius=True)
        add_text_box(slide, Inches(4), Inches(5.85), Inches(5.3), Inches(0.7),
                     cta, font_size=18, font_color=THEME["bg_dark"],
                     bold=True, alignment=PP_ALIGN.CENTER)
    
    return slide

def make_thank_you_slide(prs, title="THANK YOU", contact=""):
    """结束页"""
    slide = prs.slides.add_slide(prs.slide_layouts[6])
    add_background(slide, THEME["bg_dark"])
    
    add_text_box(slide, Inches(0), Inches(2.2), SLIDE_WIDTH, Inches(2),
                 title, font_size=60, font_color=THEME["text_light"],
                 bold=True, alignment=PP_ALIGN.CENTER)
    
    add_shape(slide, Inches(5.5), Inches(4.2), Inches(2.3), Inches(0.05), THEME["accent"])
    
    if contact:
        add_text_box(slide, Inches(0), Inches(4.6), SLIDE_WIDTH, Inches(1.5),
                     contact, font_size=14, font_color=THEME["secondary"],
                     alignment=PP_ALIGN.CENTER, line_spacing=1.8)
    
    return slide

# ============================================================
# TOC 目录页
# ============================================================

def make_toc_slide(prs, title="CONTENTS", sections=[]):
    """目录页"""
    slide = prs.slides.add_slide(prs.slide_layouts[6])
    add_background(slide, THEME["bg_white"])
    
    add_shape(slide, Inches(0), Inches(0), Inches(4.5), SLIDE_HEIGHT, THEME["bg_dark"])
    
    add_text_box(slide, Inches(0.6), Inches(2), Inches(3.5), Inches(1.5),
                 title, font_size=36, font_color=THEME["text_light"], bold=True)
    add_shape(slide, Inches(0.6), Inches(3.5), Inches(1.5), Inches(0.05), THEME["accent"])
    
    for i, sec in enumerate(sections):
        y = Inches(1.2) + i * Inches(1.1)
        num = f"{i+1:02d}"
        add_text_box(slide, Inches(5), y, Inches(1), Inches(0.8),
                     num, font_size=28, font_color=THEME["accent"], bold=True)
        add_text_box(slide, Inches(6.2), y + Inches(0.05), Inches(6), Inches(0.4),
                     sec, font_size=18, font_color=THEME["text_dark"], bold=True)
        if i < len(sections) - 1:
            add_shape(slide, Inches(5), y + Inches(0.9), Inches(7.5), Inches(0.01), "E0E0E0")
    
    return slide

# ============================================================
# 🔩 竞品拆机 / 硬件架构 专用布局
# ============================================================

def make_spec_table_slide(prs, title, headers, rows, highlight_col=None):
    """参数规格对比表格页 - 多维度器件/产品参数逐项对比
    
    Args:
        headers: ["参数", "我司方案", "竞品A", "竞品B"]
        rows: [["主频", "1.2GHz", "1.0GHz", "800MHz"], ...]
        highlight_col: 高亮列索引（通常为我司列=1），使用 ACCENT 色底
    """
    slide = prs.slides.add_slide(prs.slide_layouts[6])
    add_background(slide, THEME["bg_white"])
    add_shape(slide, Inches(0), Inches(0), SLIDE_WIDTH, Inches(0.06), THEME["primary"])
    add_text_box(slide, MARGIN, Inches(0.35), Inches(11), Inches(0.8),
                 title, font_size=28, font_color=THEME["primary"], bold=True)
    add_shape(slide, MARGIN, Inches(1.15), Inches(1.5), Inches(0.04), THEME["accent"])
    
    n_cols = len(headers)
    n_rows = len(rows) + 1  # +1 for header
    table_w = SLIDE_WIDTH - 2 * MARGIN
    table_h = min(Inches(5.5), Inches(0.55) * n_rows)
    
    table_shape = slide.shapes.add_table(n_rows, n_cols, MARGIN, Inches(1.5), table_w, table_h)
    table = table_shape.table
    
    # 表头
    for j, h in enumerate(headers):
        cell = table.cell(0, j)
        cell.text = h
        for p in cell.text_frame.paragraphs:
            p.font.size = Pt(12)
            p.font.bold = True
            p.font.color.rgb = hex_to_rgb("FFFFFF")
            p.alignment = PP_ALIGN.CENTER
        cell.fill.solid()
        cell.fill.fore_color.rgb = hex_to_rgb(THEME["primary"])
    
    # 数据行
    for i, row in enumerate(rows):
        for j, val in enumerate(row):
            cell = table.cell(i + 1, j)
            cell.text = str(val)
            for p in cell.text_frame.paragraphs:
                p.font.size = Pt(11)
                p.font.color.rgb = hex_to_rgb(THEME["text_dark"])
                p.alignment = PP_ALIGN.CENTER
            # 高亮列
            if highlight_col is not None and j == highlight_col:
                cell.fill.solid()
                cell.fill.fore_color.rgb = hex_to_rgb("E8F4FD")  # 浅蓝高亮
            # 交替行底色
            elif i % 2 == 1:
                cell.fill.solid()
                cell.fill.fore_color.rgb = hex_to_rgb(THEME["bg_light"])
    
    return slide

def make_component_selection_slide(prs, title, components):
    """器件选型分析页 - 候选方案横向对比卡片
    
    Args:
        components: [
            {"name": "STM32H750", "vendor": "ST", "pros": [...], "cons": [...], 
             "price": "$3.2", "recommended": True},
            ...
        ]
    """
    slide = prs.slides.add_slide(prs.slide_layouts[6])
    add_background(slide, THEME["bg_white"])
    add_shape(slide, Inches(0), Inches(0), SLIDE_WIDTH, Inches(0.06), THEME["primary"])
    add_text_box(slide, MARGIN, Inches(0.35), Inches(11), Inches(0.8),
                 title, font_size=28, font_color=THEME["primary"], bold=True)
    add_shape(slide, MARGIN, Inches(1.15), Inches(1.5), Inches(0.04), THEME["accent"])
    
    n = len(components)
    gap = Inches(0.25)
    card_w = (SLIDE_WIDTH - 2 * MARGIN - gap * (n - 1)) // n
    card_h = Inches(5.0)
    top = Inches(1.5)
    
    for i, comp in enumerate(components):
        left = MARGIN + i * (card_w + gap)
        border_color = THEME["accent"] if comp.get("recommended") else THEME["bg_light"]
        
        # 卡片背景（推荐方案加粗边框）
        shape = add_shape(slide, left, top, card_w, card_h, THEME["bg_light"], radius=True)
        if comp.get("recommended"):
            shape.line.color.rgb = hex_to_rgb(THEME["accent"])
            shape.line.width = Pt(3)
            # "推荐" 标签
            add_shape(slide, left, top, Inches(1.2), Inches(0.35), THEME["accent"], radius=True)
            add_text_box(slide, left + Inches(0.1), top + Inches(0.02), Inches(1), Inches(0.3),
                         "★ 推荐", font_size=11, font_color="FFFFFF", bold=True)
        
        # 器件名称
        add_text_box(slide, left + Inches(0.2), top + Inches(0.5), card_w - Inches(0.4), Inches(0.5),
                     comp["name"], font_size=18, font_color=THEME["primary"], bold=True,
                     alignment=PP_ALIGN.CENTER)
        # 厂商
        add_text_box(slide, left + Inches(0.2), top + Inches(1.0), card_w - Inches(0.4), Inches(0.3),
                     comp.get("vendor", ""), font_size=12, font_color="888888",
                     alignment=PP_ALIGN.CENTER)
        # 价格
        add_text_box(slide, left + Inches(0.2), top + Inches(1.3), card_w - Inches(0.4), Inches(0.4),
                     comp.get("price", ""), font_size=22, font_color=THEME["accent"], bold=True,
                     alignment=PP_ALIGN.CENTER)
        # 优势
        add_text_box(slide, left + Inches(0.2), top + Inches(1.8), card_w - Inches(0.4), Inches(0.3),
                     "✅ 优势", font_size=11, font_color=THEME.get("success", "27AE60"), bold=True)
        pros_text = "\n".join(f"• {p}" for p in comp.get("pros", []))
        add_text_box(slide, left + Inches(0.3), top + Inches(2.1), card_w - Inches(0.6), Inches(1.2),
                     pros_text, font_size=10, font_color=THEME["text_dark"])
        # 不足
        add_text_box(slide, left + Inches(0.2), top + Inches(3.3), card_w - Inches(0.4), Inches(0.3),
                     "⚠️ 不足", font_size=11, font_color=THEME.get("danger", "E74C3C"), bold=True)
        cons_text = "\n".join(f"• {c}" for c in comp.get("cons", []))
        add_text_box(slide, left + Inches(0.3), top + Inches(3.6), card_w - Inches(0.6), Inches(1.2),
                     cons_text, font_size=10, font_color=THEME["text_dark"])
    
    return slide

def make_bom_table_slide(prs, title, bom_items, total_cost=""):
    """BOM清单表格页
    
    Args:
        bom_items: [{"category": "主控", "part": "STM32H750", "vendor": "ST",
                     "package": "LQFP100", "qty": 1, "price": "$3.2"}, ...]
        total_cost: "$28.5"
    """
    slide = prs.slides.add_slide(prs.slide_layouts[6])
    add_background(slide, THEME["bg_white"])
    add_shape(slide, Inches(0), Inches(0), SLIDE_WIDTH, Inches(0.06), THEME["primary"])
    add_text_box(slide, MARGIN, Inches(0.35), Inches(11), Inches(0.8),
                 title, font_size=28, font_color=THEME["primary"], bold=True)
    add_shape(slide, MARGIN, Inches(1.15), Inches(1.5), Inches(0.04), THEME["accent"])
    
    headers = ["类别", "器件型号", "厂商", "封装", "数量", "单价"]
    n_rows = len(bom_items) + 1
    n_cols = len(headers)
    table_w = SLIDE_WIDTH - 2 * MARGIN
    table_h = min(Inches(5.0), Inches(0.45) * n_rows)
    
    table_shape = slide.shapes.add_table(n_rows, n_cols, MARGIN, Inches(1.5), table_w, table_h)
    table = table_shape.table
    
    # 列宽比例: 类别15%, 型号25%, 厂商15%, 封装15%, 数量10%, 单价20%
    col_ratios = [0.15, 0.25, 0.15, 0.15, 0.10, 0.20]
    for j, ratio in enumerate(col_ratios):
        table.columns[j].width = int(table_w * ratio)
    
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
    
    for i, item in enumerate(bom_items):
        vals = [item.get("category",""), item.get("part",""), item.get("vendor",""),
                item.get("package",""), str(item.get("qty","")), item.get("price","")]
        for j, val in enumerate(vals):
            cell = table.cell(i + 1, j)
            cell.text = val
            for p in cell.text_frame.paragraphs:
                p.font.size = Pt(10)
                p.alignment = PP_ALIGN.CENTER
            if i % 2 == 1:
                cell.fill.solid()
                cell.fill.fore_color.rgb = hex_to_rgb(THEME["bg_light"])
    
    # 总成本标注
    if total_cost:
        add_text_box(slide, Inches(9), Inches(6.5), Inches(4), Inches(0.6),
                     f"预估总BOM成本: {total_cost}", font_size=18,
                     font_color=THEME["accent"], bold=True, alignment=PP_ALIGN.RIGHT)
    
    return slide

def make_cost_breakdown_slide(prs, title, categories, note=""):
    """成本构成分析页 - 彩色条形横向占比图
    
    Args:
        categories: [{"name": "主控", "pct": 35, "cost": "$9.8"},
                     {"name": "电源", "pct": 20, "cost": "$5.6"}, ...]
    """
    slide = prs.slides.add_slide(prs.slide_layouts[6])
    add_background(slide, THEME["bg_white"])
    add_shape(slide, Inches(0), Inches(0), SLIDE_WIDTH, Inches(0.06), THEME["primary"])
    add_text_box(slide, MARGIN, Inches(0.35), Inches(11), Inches(0.8),
                 title, font_size=28, font_color=THEME["primary"], bold=True)
    add_shape(slide, MARGIN, Inches(1.15), Inches(1.5), Inches(0.04), THEME["accent"])
    
    bar_colors = ["2C3E50", "3498DB", "E67E22", "27AE60", "9B59B6", "E74C3C", "1ABC9C", "F39C12"]
    bar_left = Inches(3)
    bar_w_max = Inches(8)
    bar_h = Inches(0.6)
    
    for i, cat in enumerate(categories):
        y = Inches(1.8) + i * Inches(0.85)
        # 类别名
        add_text_box(slide, MARGIN, y, Inches(2.2), bar_h,
                     cat["name"], font_size=13, font_color=THEME["text_dark"],
                     bold=True, alignment=PP_ALIGN.RIGHT)
        # 比例条
        w = int(bar_w_max * cat["pct"] / 100)
        color = bar_colors[i % len(bar_colors)]
        add_shape(slide, bar_left, y + Inches(0.05), w, bar_h - Inches(0.1), color, radius=True)
        # 百分比 + 金额标注
        label = f"{cat['pct']}%  {cat.get('cost', '')}"
        add_text_box(slide, bar_left + w + Inches(0.15), y, Inches(2), bar_h,
                     label, font_size=12, font_color=color, bold=True)
    
    if note:
        add_text_box(slide, MARGIN, Inches(6.8), Inches(12), Inches(0.4),
                     note, font_size=9, font_color="999999", italic=True)
    return slide

def make_block_diagram_slide(prs, title, blocks, connections_desc=""):
    """架构/链路框图页 - 用色块+箭头描述系统架构
    
    Args:
        blocks: [{"name": "主控 STM32H750", "x": 5, "y": 3, "w": 3, "h": 1.2,
                  "color": "primary", "sub": "ARM Cortex-M7 @480MHz"}, ...]
        connections_desc: 文字描述模块间连接关系
    """
    slide = prs.slides.add_slide(prs.slide_layouts[6])
    add_background(slide, THEME["bg_white"])
    add_shape(slide, Inches(0), Inches(0), SLIDE_WIDTH, Inches(0.06), THEME["primary"])
    add_text_box(slide, MARGIN, Inches(0.35), Inches(11), Inches(0.8),
                 title, font_size=28, font_color=THEME["primary"], bold=True)
    add_shape(slide, MARGIN, Inches(1.15), Inches(1.5), Inches(0.04), THEME["accent"])
    
    for blk in blocks:
        color_key = blk.get("color", "primary")
        fill_color = THEME.get(color_key, color_key)
        x, y, w, h = Inches(blk["x"]), Inches(blk["y"]), Inches(blk["w"]), Inches(blk["h"])
        shape = add_shape(slide, x, y, w, h, fill_color, radius=True)
        # 块名称
        tf = shape.text_frame
        tf.word_wrap = True
        p = tf.paragraphs[0]
        p.text = blk["name"]
        p.font.size = Pt(13)
        p.font.color.rgb = hex_to_rgb("FFFFFF")
        p.font.bold = True
        p.alignment = PP_ALIGN.CENTER
        # 子标题
        if blk.get("sub"):
            p2 = tf.add_paragraph()
            p2.text = blk["sub"]
            p2.font.size = Pt(9)
            p2.font.color.rgb = hex_to_rgb("E0E0E0")
            p2.alignment = PP_ALIGN.CENTER
    
    # 连接关系文字说明（框图中精确箭头需图片，这里用文字补充）
    if connections_desc:
        add_text_box(slide, MARGIN, Inches(6.2), Inches(12), Inches(1),
                     connections_desc, font_size=10, font_color="888888",
                     italic=True, line_spacing=1.4)
    
    return slide

# ============================================================
# 🔍 问题回溯 专用布局
# ============================================================

def make_fishbone_slide(prs, title, causes):
    """鱼骨图/根因分析页 - 六大维度（人机料法环测）
    
    Args:
        causes: {
            "人 Man": ["操作员未按SOP", "培训不到位"],
            "机 Machine": ["设备精度不足", "未按期保养"],
            "料 Material": ["来料批次差异", "供应商变更"],
            "法 Method": ["工艺参数不当", "设计余量不足"],
            "环 Environment": ["温度超范围", "湿度过高"],
            "测 Measurement": ["测试覆盖不足", "判定标准模糊"],
        }
    """
    slide = prs.slides.add_slide(prs.slide_layouts[6])
    add_background(slide, THEME["bg_white"])
    add_shape(slide, Inches(0), Inches(0), SLIDE_WIDTH, Inches(0.06), THEME.get("danger", THEME["primary"]))
    add_text_box(slide, MARGIN, Inches(0.35), Inches(11), Inches(0.8),
                 title, font_size=28, font_color=THEME["primary"], bold=True)
    add_shape(slide, MARGIN, Inches(1.15), Inches(1.5), Inches(0.04), THEME["accent"])
    
    # 中轴线（鱼骨主干）
    spine_y = Inches(3.8)
    add_shape(slide, Inches(1), spine_y, Inches(10.5), Inches(0.05), THEME["primary"])
    
    # 鱼头（问题结果）
    add_shape(slide, Inches(11), spine_y - Inches(0.5), Inches(2), Inches(1.0),
              THEME.get("danger", THEME["accent"]), radius=True)
    add_text_box(slide, Inches(11), spine_y - Inches(0.35), Inches(2), Inches(0.7),
                 "根本\n原因", font_size=14, font_color="FFFFFF", bold=True,
                 alignment=PP_ALIGN.CENTER)
    
    # 六根鱼骨
    category_colors = ["2C3E50", "3498DB", "E67E22", "27AE60", "9B59B6", "E74C3C"]
    categories = list(causes.keys())
    
    for i, (cat_name, items) in enumerate(causes.items()):
        col = i % 3  # 3列
        row = i // 3  # 上下2行
        x = Inches(1.5) + col * Inches(3.2)
        y = Inches(1.5) if row == 0 else Inches(4.5)
        color = category_colors[i % len(category_colors)]
        
        # 维度标题
        add_shape(slide, x, y, Inches(2.5), Inches(0.4), color, radius=True)
        add_text_box(slide, x, y + Inches(0.02), Inches(2.5), Inches(0.35),
                     cat_name, font_size=13, font_color="FFFFFF", bold=True,
                     alignment=PP_ALIGN.CENTER)
        
        # 原因条目
        for j, item in enumerate(items[:3]):  # 最多3条
            iy = y + Inches(0.5) + j * Inches(0.35) if row == 0 else y - Inches(0.5) - j * Inches(0.35)
            add_text_box(slide, x + Inches(0.2), iy, Inches(2.3), Inches(0.3),
                         f"• {item}", font_size=10, font_color=THEME["text_dark"])
    
    return slide

def make_verification_matrix_slide(prs, title, items):
    """验证结果矩阵页 - 带 PASS/FAIL 状态标记的表格
    
    Args:
        items: [{"item": "高温老化 85°C/1000h", "method": "IEC 60068-2-2",
                 "criteria": "功能正常", "result": "PASS", "note": ""}, ...]
    """
    slide = prs.slides.add_slide(prs.slide_layouts[6])
    add_background(slide, THEME["bg_white"])
    add_shape(slide, Inches(0), Inches(0), SLIDE_WIDTH, Inches(0.06), THEME["primary"])
    add_text_box(slide, MARGIN, Inches(0.35), Inches(11), Inches(0.8),
                 title, font_size=28, font_color=THEME["primary"], bold=True)
    add_shape(slide, MARGIN, Inches(1.15), Inches(1.5), Inches(0.04), THEME["accent"])
    
    headers = ["验证项目", "测试方法", "判定标准", "结果", "备注"]
    n_rows = len(items) + 1
    n_cols = len(headers)
    table_w = SLIDE_WIDTH - 2 * MARGIN
    table_h = min(Inches(5.2), Inches(0.5) * n_rows)
    
    table_shape = slide.shapes.add_table(n_rows, n_cols, MARGIN, Inches(1.5), table_w, table_h)
    table = table_shape.table
    
    col_ratios = [0.28, 0.22, 0.20, 0.10, 0.20]
    for j, ratio in enumerate(col_ratios):
        table.columns[j].width = int(table_w * ratio)
    
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
    
    for i, item in enumerate(items):
        vals = [item.get("item",""), item.get("method",""), item.get("criteria",""),
                item.get("result",""), item.get("note","")]
        for j, val in enumerate(vals):
            cell = table.cell(i + 1, j)
            cell.text = val
            for p in cell.text_frame.paragraphs:
                p.font.size = Pt(10)
                p.alignment = PP_ALIGN.CENTER
                # PASS/FAIL 颜色标记
                if j == 3:  # 结果列
                    p.font.bold = True
                    if val.upper() == "PASS":
                        p.font.color.rgb = hex_to_rgb(THEME.get("success", "27AE60"))
                    elif val.upper() == "FAIL":
                        p.font.color.rgb = hex_to_rgb(THEME.get("danger", "E74C3C"))
                    elif val.upper() in ["N/A", "PENDING", "待验证"]:
                        p.font.color.rgb = hex_to_rgb(THEME.get("warning", "F39C12"))
    
    return slide

def make_before_after_slide(prs, title, metrics):
    """修复前后数据对比页 — 左右对比+箭头+改善率
    
    Args:
        metrics: [{"name": "不良率", "before": "5.2%", "after": "0.3%",
                   "improvement": "↓94%", "good": True}, ...]
    """
    slide = prs.slides.add_slide(prs.slide_layouts[6])
    add_background(slide, THEME["bg_white"])
    add_shape(slide, Inches(0), Inches(0), SLIDE_WIDTH, Inches(0.06), THEME.get("danger", THEME["primary"]))
    add_text_box(slide, MARGIN, Inches(0.35), Inches(11), Inches(0.8),
                 title, font_size=28, font_color=THEME["primary"], bold=True)
    add_shape(slide, MARGIN, Inches(1.15), Inches(1.5), Inches(0.04), THEME["accent"])
    
    # 列标题
    add_text_box(slide, Inches(2.5), Inches(1.5), Inches(3), Inches(0.5),
                 "修复前", font_size=16, font_color=THEME.get("danger", "E74C3C"),
                 bold=True, alignment=PP_ALIGN.CENTER)
    add_text_box(slide, Inches(8), Inches(1.5), Inches(3), Inches(0.5),
                 "修复后", font_size=16, font_color=THEME.get("success", "27AE60"),
                 bold=True, alignment=PP_ALIGN.CENTER)
    
    for i, m in enumerate(metrics):
        y = Inches(2.2) + i * Inches(1.2)
        
        # 指标名
        add_text_box(slide, MARGIN, y, Inches(2), Inches(0.8),
                     m["name"], font_size=14, font_color=THEME["text_dark"], bold=True)
        
        # 修复前 - 红色背景卡片
        add_shape(slide, Inches(2.5), y, Inches(3), Inches(0.8),
                  THEME.get("danger", "E74C3C"), radius=True)
        add_text_box(slide, Inches(2.5), y + Inches(0.1), Inches(3), Inches(0.6),
                     m["before"], font_size=24, font_color="FFFFFF",
                     bold=True, alignment=PP_ALIGN.CENTER)
        
        # 箭头 + 改善率
        arrow_color = THEME.get("success", "27AE60") if m.get("good") else THEME.get("danger", "E74C3C")
        add_text_box(slide, Inches(5.8), y, Inches(2), Inches(0.8),
                     f"→ {m.get('improvement', '')}", font_size=18,
                     font_color=arrow_color, bold=True, alignment=PP_ALIGN.CENTER)
        
        # 修复后 - 绿色背景卡片
        add_shape(slide, Inches(8), y, Inches(3), Inches(0.8),
                  THEME.get("success", "27AE60"), radius=True)
        add_text_box(slide, Inches(8), y + Inches(0.1), Inches(3), Inches(0.6),
                     m["after"], font_size=24, font_color="FFFFFF",
                     bold=True, alignment=PP_ALIGN.CENTER)
    
    return slide

def make_risk_matrix_slide(prs, title, risks):
    """风险评估矩阵页 — 概率×影响 的四象限矩阵
    
    Args:
        risks: [{"name": "主控停产", "probability": "高", "impact": "高", "mitigation": "备选型号"},
                {"name": "PCB交期", "probability": "中", "impact": "中", "mitigation": "双供应商"}, ...]
    """
    slide = prs.slides.add_slide(prs.slide_layouts[6])
    add_background(slide, THEME["bg_white"])
    add_shape(slide, Inches(0), Inches(0), SLIDE_WIDTH, Inches(0.06), THEME["primary"])
    add_text_box(slide, MARGIN, Inches(0.35), Inches(11), Inches(0.8),
                 title, font_size=28, font_color=THEME["primary"], bold=True)
    add_shape(slide, MARGIN, Inches(1.15), Inches(1.5), Inches(0.04), THEME["accent"])
    
    # 四象限背景
    qx, qy = Inches(1), Inches(1.6)
    qw, qh = Inches(5), Inches(2.5)
    # 高概率高影响（红）
    add_shape(slide, qx + qw, qy, qw, qh, "FDEDEC")
    add_text_box(slide, qx + qw + Inches(0.1), qy + Inches(0.05), Inches(2), Inches(0.3),
                 "⚠ 高风险", font_size=10, font_color="E74C3C", bold=True)
    # 高概率低影响（黄）
    add_shape(slide, qx, qy, qw, qh, "FEF9E7")
    add_text_box(slide, qx + Inches(0.1), qy + Inches(0.05), Inches(2), Inches(0.3),
                 "△ 关注", font_size=10, font_color="F39C12", bold=True)
    # 低概率高影响（黄）
    add_shape(slide, qx + qw, qy + qh, qw, qh, "FEF9E7")
    add_text_box(slide, qx + qw + Inches(0.1), qy + qh + Inches(0.05), Inches(2), Inches(0.3),
                 "△ 监控", font_size=10, font_color="F39C12", bold=True)
    # 低概率低影响（绿）
    add_shape(slide, qx, qy + qh, qw, qh, "EAFAF1")
    add_text_box(slide, qx + Inches(0.1), qy + qh + Inches(0.05), Inches(2), Inches(0.3),
                 "○ 可接受", font_size=10, font_color="27AE60", bold=True)
    
    # 坐标轴标签
    add_text_box(slide, qx - Inches(0.5), qy + qh - Inches(0.2), Inches(0.5), Inches(0.4),
                 "概\n率", font_size=12, font_color=THEME["text_dark"], bold=True,
                 alignment=PP_ALIGN.CENTER)
    add_text_box(slide, qx + qw - Inches(0.5), qy + qh * 2 + Inches(0.1), Inches(2), Inches(0.3),
                 "影响程度 →", font_size=12, font_color=THEME["text_dark"], bold=True)
    
    # 右侧风险清单
    add_text_box(slide, Inches(11.5), Inches(1.6), Inches(1.5), Inches(0.4),
                 "风险清单", font_size=14, font_color=THEME["primary"], bold=True)
    for i, risk in enumerate(risks[:6]):
        y = Inches(2.1) + i * Inches(0.75)
        level_color = {"高": "E74C3C", "中": "F39C12", "低": "27AE60"}
        color = level_color.get(risk.get("probability", "中"), "F39C12")
        add_icon_circle(slide, Inches(11.7), y + Inches(0.15), Inches(0.15), color)
        add_text_box(slide, Inches(12), y, Inches(1.2), Inches(0.3),
                     risk["name"], font_size=10, font_color=THEME["text_dark"], bold=True)
        add_text_box(slide, Inches(12), y + Inches(0.3), Inches(1.2), Inches(0.3),
                     risk.get("mitigation", ""), font_size=8, font_color="888888")
    
    return slide

def make_annotated_layout_slide(prs, title, regions, description=""):
    """区域标注说明页 - 用于PCB布局/器件分布标注（纯色块+标注线模拟）
    
    Args:
        regions: [{"name": "主控区", "x": 3, "y": 2, "w": 2.5, "h": 1.5,
                   "color": "3498DB", "items": ["STM32H750", "25MHz晶振"]}, ...]
        description: 补充说明文字
    """
    slide = prs.slides.add_slide(prs.slide_layouts[6])
    add_background(slide, THEME["bg_white"])
    add_shape(slide, Inches(0), Inches(0), SLIDE_WIDTH, Inches(0.06), THEME["primary"])
    add_text_box(slide, MARGIN, Inches(0.35), Inches(11), Inches(0.8),
                 title, font_size=28, font_color=THEME["primary"], bold=True)
    add_shape(slide, MARGIN, Inches(1.15), Inches(1.5), Inches(0.04), THEME["accent"])
    
    # PCB 底板区域（灰色大矩形模拟）
    add_shape(slide, Inches(1.5), Inches(1.5), Inches(8), Inches(5.2), "D5D8DC")
    add_text_box(slide, Inches(1.5), Inches(6.5), Inches(8), Inches(0.3),
                 "▲ PCB 布局示意（非等比例）", font_size=9, font_color="999999",
                 alignment=PP_ALIGN.CENTER, italic=True)
    
    # 各功能区域色块
    for region in regions:
        x, y = Inches(region["x"]), Inches(region["y"])
        w, h = Inches(region["w"]), Inches(region["h"])
        color = region.get("color", THEME["secondary"])
        
        shape = add_shape(slide, x, y, w, h, color, radius=True)
        shape.fill.solid()
        shape.fill.fore_color.rgb = hex_to_rgb(color)
        # 半透明效果 (python-pptx 有限支持)
        
        # 区域名称
        add_text_box(slide, x + Inches(0.1), y + Inches(0.05), w - Inches(0.2), Inches(0.35),
                     region["name"], font_size=11, font_color="FFFFFF", bold=True,
                     alignment=PP_ALIGN.CENTER)
        
        # 器件列表
        items_text = "\n".join(region.get("items", []))
        add_text_box(slide, x + Inches(0.1), y + Inches(0.4), w - Inches(0.2), h - Inches(0.5),
                     items_text, font_size=9, font_color="FFFFFF")
    
    # 右侧图例
    legend_x = Inches(10)
    add_text_box(slide, legend_x, Inches(1.5), Inches(2.5), Inches(0.4),
                 "图例", font_size=13, font_color=THEME["primary"], bold=True)
    for i, region in enumerate(regions):
        ly = Inches(2.0) + i * Inches(0.5)
        color = region.get("color", THEME["secondary"])
        add_shape(slide, legend_x, ly, Inches(0.3), Inches(0.3), color, radius=True)
        add_text_box(slide, legend_x + Inches(0.4), ly, Inches(2), Inches(0.3),
                     region["name"], font_size=10, font_color=THEME["text_dark"])
    
    if description:
        add_text_box(slide, MARGIN, Inches(6.9), Inches(12), Inches(0.4),
                     description, font_size=9, font_color="999999", italic=True)
    
    return slide

# ============================================================
# 主生成函数
# ============================================================

def build_presentation():
    prs = Presentation()
    prs.slide_width = SLIDE_WIDTH
    prs.slide_height = SLIDE_HEIGHT
    
    # === 按大纲逐页生成 ===
    # ... (根据 Phase 1 的大纲填充)
    
    # === 保存 ===
    output_path = "output.pptx"
    prs.save(output_path)
    print(f"✅ PPT 已生成: {os.path.abspath(output_path)}")
    return output_path

if __name__ == "__main__":
    build_presentation()
```

### Phase 4: 执行与验证

1. **运行脚本**生成 `.pptx` 文件
2. **验证文件**：确认文件大小合理（通常 500KB~5MB）
3. **输出摘要**：列出生成的页数、使用的布局类型、输出路径

### Phase 5: 交付

向用户交付：
1. ✅ 生成的 `.pptx` 文件路径
2. 📋 PPT 大纲 + 每页内容摘要
3. 💡 后续优化建议（如添加图片、调整配色等）

---

## 布局选择策略

根据页面内容自动选择最佳布局：

| 内容特征 | 推荐布局 | 函数 |
|---------|---------|------|
| PPT 首页 | 封面页 | `make_cover_slide()` |
| 章节目录 | 目录页 | `make_toc_slide()` |
| 新章节开头 | 过渡页 | `make_section_slide()` |
| 文字要点 (≤6条) | 标准内容页 | `make_content_slide()` |
| A vs B 对比 | 双栏页 | `make_two_column_slide()` |
| 并列概念 (3~4个) | 卡片页 | `make_cards_slide()` |
| 时间/流程 (3~6步) | 时间轴页 | `make_timeline_slide()` |
| 关键数据 (2~4个) | 数据高亮页 | `make_data_highlight_slide()` |
| 要点回顾 | 总结页 | `make_summary_slide()` |
| PPT 结尾 | 感谢页 | `make_thank_you_slide()` |
| **器件规格对比** (多维参数) | 参数对比表格页 | `make_spec_table_slide()` |
| **器件选型分析** (候选方案) | 器件选型分析页 | `make_component_selection_slide()` |
| **PCB/器件区域标注** | 区域标注说明页 | `make_annotated_layout_slide()` |
| **BOM清单** | BOM表格页 | `make_bom_table_slide()` |
| **成本构成** (饼图式) | 成本构成分析页 | `make_cost_breakdown_slide()` |
| **电源树/信号链路** | 架构链路图页 | `make_block_diagram_slide()` |
| **鱼骨图/5Why** | 根因分析页 | `make_fishbone_slide()` |
| **验证矩阵** (PASS/FAIL) | 验证结果矩阵页 | `make_verification_matrix_slide()` |
| **修复前后数据对比** | 数据对比页 | `make_before_after_slide()` |
| **风险矩阵** | 风险评估矩阵页 | `make_risk_matrix_slide()` |

---

## 排版检查清单

生成后自查以下事项（不合格则回去修改）：

### 通用检查
- [ ] 每页核心信息不超过 3 个要点
- [ ] 标题字号 ≥ 24pt，正文字号 ≥ 12pt
- [ ] 配色全篇统一，来自选定的 THEME
- [ ] 页面边距至少 0.5 英寸
- [ ] 所有文本对齐一致（左对齐 or 居中，不混用）
- [ ] 页面间有明确的逻辑过渡
- [ ] 内容无错别字和语法错误
- [ ] 封面页和结尾页风格呼应
- [ ] 总页数在类型建议范围内

### 🔩 竞品拆机报告 专项检查
- [ ] 产品规格对比表至少覆盖 8 个以上关键参数
- [ ] BOM 清单包含完整的：器件类型、型号、厂商、封装、单价
- [ ] 成本构成分析各项占比之和 = 100%
- [ ] 器件选型对比有明确的"推荐"标记和理由
- [ ] "设计亮点 vs 不足"双栏页内容均衡，避免一边倒
- [ ] 所有器件型号名称准确无误
- [ ] 建议使用 `engineering` 主题配色

### 🔍 问题回溯报告 专项检查
- [ ] 封面包含问题编号和严重等级标识
- [ ] 问题速览页的关键数据（发现时间、影响范围、严重等级）完整
- [ ] 时间线覆盖完整的 发现→遏制→分析→修复→验证 全流程
- [ ] 鱼骨图覆盖至少 4 个维度（人/机/料/法/环/测）
- [ ] 根因分析有数据支撑（测试数据、仿真结果、日志证据）
- [ ] 验证矩阵中每项有明确的 PASS/FAIL 状态标记
- [ ] 修复前后对比数据使用红/绿色直观区分
- [ ] 建议使用 `rca_red` 主题配色

### 🖥️ 硬件技术架构分析 专项检查
- [ ] 系统架构框图清晰标注各模块及互联关系
- [ ] 器件选型对比至少包含 2-3 个候选方案
- [ ] 电源树覆盖所有供电节点（输入→各级→各模块）
- [ ] 关键信号链路（高速/模拟/射频）有明确的设计要点
- [ ] BOM 成本估算包含所有关键器件
- [ ] 风险矩阵标注了缓解措施
- [ ] 里程碑时间轴覆盖 原理图→PCB→打样→调试→DVT→量产
- [ ] 建议使用 `engineering` 或 `tech` 主题配色
