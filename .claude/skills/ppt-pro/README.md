# PPT Pro Skill — 专业演示文稿制作技能

## 📌 概述

**PPT Pro** 是一个基于 Claude Code Skill 架构设计的专业 PPT 制作技能。它能将用户的自然语言需求转化为结构清晰、设计精美的 PowerPoint 演示文稿，通过 `python-pptx` 库直接生成 `.pptx` 文件。

---

## 🏗️ 架构设计

本技能完全遵循 Claude Code 的 Skill 系统规范：

```
.claude/skills/ppt-pro/
├── SKILL.md              ← 主技能文件（frontmatter + prompt）
├── design-guide.md       ← 设计规范参考文档
└── README.md             ← 本说明文件
```

### 设计灵感来源

借鉴 Claude Code 的以下架构思想：

| Claude Code 概念 | PPT Pro 中的对应 |
|-----------------|-----------------|
| **Tool 系统** — 每个工具有 `call()` + `inputSchema` | 每种页面布局有独立的 `make_xxx_slide()` 函数 |
| **Theme 主题** — 全局统一的视觉风格 | 6 套预设配色主题，8 个色值体系 |
| **QueryEngine 循环** — 分析→执行→验证 | Phase 0~5 的 5 阶段工作流 |
| **Feature Flags** — 条件编译裁剪 | 按 PPT 类型动态选择布局和配色 |
| **ToolUseContext** — 工具执行上下文 | 每个布局函数共享 `THEME`/`FONT` 全局配置 |
| **权限分层** — 最小权限原则 | `allowed-tools` 仅开放文件和 Python 执行权限 |
| **fork 执行模式** — 子代理独立运行 | `context: fork` 独立运行不干扰主对话 |

---

## 🚀 使用方式

### 在 Claude Code 中调用

```bash
# 方式一：斜杠命令触发
/ppt-pro 2026年Q1工作汇报

# 方式二：自然语言触发（模型自动识别）
帮我做一个关于AI发展趋势的PPT，15页左右，科技风格

# 方式三：带参数调用
/ppt-pro 产品发布方案 --theme tech --pages 12
```

### 技能会自动完成

1. 📋 **需求分析** — 判断 PPT 类型，确定页数和结构
2. 🎨 **主题选择** — 根据场景选择/推荐配色方案
3. 📝 **大纲设计** — 生成完整的页面大纲（展示给用户确认）
4. 🔧 **脚本生成** — 编写 python-pptx 脚本
5. ▶️ **执行生成** — 运行脚本输出 `.pptx` 文件
6. ✅ **交付验证** — 确认文件有效，提供优化建议

---

## 📐 支持的页面类型

| 类型 | 函数 | 适用场景 |
|------|------|---------|
| 🎬 封面页 | `make_cover_slide()` | PPT 开篇 |
| 📑 目录页 | `make_toc_slide()` | 章节导航 |
| 🔢 章节过渡页 | `make_section_slide()` | 新章节开头 |
| 📝 标准内容页 | `make_content_slide()` | 文字要点 |
| ⚖️ 双栏对比页 | `make_two_column_slide()` | A vs B 对比 |
| 🃏 卡片网格页 | `make_cards_slide()` | 并列概念展示 |
| ⏱️ 时间轴页 | `make_timeline_slide()` | 时间/流程 |
| 📊 数据高亮页 | `make_data_highlight_slide()` | 关键指标 |
| ✅ 总结页 | `make_summary_slide()` | 要点回顾 |
| 🙏 结束页 | `make_thank_you_slide()` | PPT 结尾 |

---

## 🎨 预设主题

| 主题 | 风格 | 适用场景 |
|------|------|---------|
| `corporate` | 商务蓝 | 工作汇报、商业提案 |
| `tech` | 科技暗 | 技术分享、产品发布 |
| `academic` | 学术红 | 学术答辩、研究报告 |
| `modern_dark` | 暗黑紫 | 创意展示、设计评审 |
| `minimal` | 极简黑白 | 高端品牌、简约汇报 |
| `nature` | 自然绿 | 环保主题、健康领域 |

---

## 📋 设计原则

### 内容原则
- **金字塔原则**：结论先行，自上而下
- **MECE 原则**：互不重叠，完全穷尽
- **3×3 法则**：每页 ≤3 个要点，每点 ≤3 行

### 视觉原则
- **对比**：大小/颜色/字重建立视觉层级
- **对齐**：所有元素基于网格对齐
- **重复**：配色/字体/间距全篇一致
- **留白**：至少 30% 页面空白

---

## 🔧 技术依赖

- **Python 3.7+**
- **python-pptx** — PPT 文件生成
- **Pillow** — 图片处理（可选）
- **lxml** — XML 操作（python-pptx 依赖）

---

## 📁 输出格式

- 文件格式：`.pptx`（PowerPoint 2007+ 兼容）
- 幻灯片尺寸：16:9 宽屏（13.333 × 7.5 英寸）
- 字体：微软雅黑（中文）/ Segoe UI（英文）
- 文件大小：通常 500KB ~ 5MB

---

## 💡 扩展建议

技能生成的 PPT 可以进一步手动优化：

1. **添加图片/图表** — 在占位区域插入实际图片
2. **调整动画** — 在 PowerPoint 中添加入场/强调动画
3. **替换字体** — 根据品牌规范更换字体
4. **嵌入视频** — 在产品演示页插入视频
5. **添加备注** — 在演讲者备注中写入讲稿

---

## 📜 Skill 元数据说明

```yaml
name: PPT Pro                           # 技能显示名
context: fork                           # 独立子代理运行（不干扰主对话）
agent: general-purpose                  # 通用代理类型
model: sonnet                           # 使用 Sonnet 模型（平衡速度和质量）
effort: high                            # 高思考力度
allowed-tools:                          # 最小权限原则
  - Bash(python3:*), Bash(pip:*)        # Python 执行
  - Read, Write, Edit                   # 文件操作
  - Grep, Glob                          # 文件搜索
```
