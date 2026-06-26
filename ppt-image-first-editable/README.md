# ppt-image-first-editable

> **conversation-first + image-first 的一条龙 PPT 技能。默认可先走 Phase A 图片版；用户也可以明确要求直接进入 Phase C-only，跳过图片版，输出文字可编辑 PPTX。单技能自包含，安装一份即可。**

---

## 两条路径

| 阶段 | 做什么 | 默认与否 |
|---|---|---|
| **Phase A** | 对话式定稿，每页出一张高密度视觉稿，合成图片型 PPTX | **默认** |
| **Phase A 完成后** | **主动询问用户是否要可编辑文字版** | 强制询问 |
| **Phase C** | 先直编 Phase A 成品图，失败再回退到重生成背景 + HTML 编辑器调文字 + 渲染原生可编辑 PPTX | 用户同意才走 |
| **Phase C-only** | 直接进入可编辑版流程，跳过 Phase A | 用户明确要求时走 |

---

## 流程图

```
[用户的模糊需求]
        │
        ▼
┌── Phase A（默认主路径）─────────────────┐
│  Stage 1     Intake + 需求确认           │
│  Stage 1.25  content_report.md          │
│  Stage 1.5   风格边界（3 个短问题）       │
│  Stage 2     多套风格预览（首/目/正）     │
│  Stage 2.5   风格 refinement             │
│  Stage 2.75  风格反演 + 风格确认          │
│  Stage 3     design_spec /              │
│              slide_blueprint /          │
│              spec_lock + 生成前确认       │
│  Stage 4     全页定稿图                  │
│  Stage 5     review + 用户批准           │
│  Stage 5.5   ★ retouch（可选）            │
└──────────────┬──────────────────────────┘
               │
               ▼   交付：图片型 PPTX + 全页定稿图 + 规划文档
   ┌───────────┴───────────┐
   │ ★ 必须主动询问用户 ★   │
   │ "是否需要可编辑文字版   │
   │  （Phase C）？"        │
   └─────────┬─────────────┘
             │
        用户同意 → 进 Phase C
             │
             ▼
┌── Phase C（文字可编辑路径）─────────────┐
│  C1  先直编 Phase A 成品图                │
│      (直编不干净时才回退到完整稿→擦字稿)   │
│  C2  detect_reserved_zones 校验          │
│  C3  写 deck.json                        │
│  C4  inject_editor_deck.py 注入          │
│  C5  HTML 编辑器调文字（用户）            │
│  C6  json_to_pptx.py 渲染 PPTX           │
└──────────────┬──────────────────────────┘
               ▼
[追加交付：文字可编辑 PPTX + deck.json + backgrounds/]

┌── Phase C-only（直接进入可编辑版）────────┐
│  C0  收集最小输入或复用现有产物           │
│  C1  先直编 Phase A 成品图                │
│  C2  detect_reserved_zones 校验          │
│  C3  写 deck.json                        │
│  C4  inject_editor_deck.py 注入          │
│  C5  HTML 编辑器调文字（用户）            │
│  C6  json_to_pptx.py 渲染 PPTX           │
└──────────────────────────────────────────┘
```

---

## 主要交付物

### Phase A（默认）
1. `content_report.md` —— 内容基底（当用户给的材料不完整时）
2. `design_spec.md` —— 整套 deck 的视觉系统
3. `slide_blueprint.md` —— 每页意图与内容 payload
4. `spec_lock.md` —— 执行约束
5. `phaseA/slides/*.png` —— 每页定稿图（可选 `phaseA/slides_clean/` 为 Stage 5.5 修过的版本）
6. **图片型 PPTX** —— 高完成度视觉稿，直接拿去汇报

### Phase C（用户同意才追加）
7. `phaseC/backgrounds/NN.png` —— 每页背景图（直编成功时可直接复用）
8. `phaseC/deck.json` —— 单一真相源（编辑器、渲染器共同消费）
9. **文字可编辑 PPTX** —— 文字是真 TextBox，PowerPoint / Keynote 里可直接改

---

## 安装

### 通用步骤

```bash
# 1. 把 skill 目录放到你的 AI 客户端的 skills 路径下
#    （路径因客户端而异，见下方"在不同 AI 里使用"）

# 2. 不需要手动装依赖——agent 首次跑会自动装。
#    如果你想提前手动确认环境，可以：
cd <skill 目录>
python3 scripts/preflight.py
# preflight 会：
#   - 检查 Python ≥ 3.10
#   - 自动 pip 装 4 个必备包（python-pptx / Pillow / numpy / opencv-python）
#   - 调 doctor 看其它项（字体 / 磁盘 / 网络 / IOPaint）
# 首次 2-5 分钟，之后秒过
```

### 在不同 AI 里使用

| AI 客户端 | skills 目录 | 兼容程度 |
|---|---|---|
| **Codex CLI** | `~/.codex/skills/` 或 `$CODEX_HOME/skills/` | ✅ 完全兼容 |
| **Claude Code（CLI）** | `~/.claude/skills/` | ✅ 完全兼容 |
| **QoderWork** | 视客户端文档而定，通常是 skills 配置目录 | ⚠️ 取决于其对 skill 协议的支持程度 |
| **其他能跑 skill + Bash + imagegen 的 agent** | 自行查文档 | ⚠️ 看是否同时满足三个条件 |

### 不会自动跑 preflight 的客户端怎么办

本 skill 的 SKILL.md 里写了"用户首次触发必须先跑 `python3 scripts/preflight.py`"。**任何能读 SKILL.md 并按指令跑 Bash 的 agent**都会照办。

如果你的客户端不会自动跑：

```bash
cd <skill 目录>
python3 scripts/preflight.py
```

手动跑一次再开始任务。preflight 会自动安装缺的 pip 包，遇到不可自动修的项目（Python 版本、磁盘、字体）会清楚告诉你怎么修。

### 我的 AI 不支持 skill 怎么办

那就用不了这个 skill。可选项：

- 用 Codex CLI 或 Claude Code CLI（两个都免费可下载）
- 或者把这个 skill 当成一个工具集，手动跑 `scripts/*.py`（失去对话流程，但脚本本身是平台无关的）

### Stage 5.5 IOPaint 自动安装

`launch_iopaint.py` 首次触发时会自动装：

- 装到专属 venv `~/.cache/ppt-image-first-editable/venv/`（不污染系统 Python）
- 含 IOPaint + Torch CPU + LaMa 模型权重
- 大小：~3GB，首次耗时 5–10 分钟
- 国内默认开 `hf-mirror.com` 镜像
- 装失败有明确兜底（角落水印批处理 / ImageMagick）

依赖运行环境内可用的 imagegen 通道（Codex 用内置 `imagegen` / GPT Image 2）。

---

## 目录结构

```
ppt-image-first-editable/
├── SKILL.md                            # 主入口（agent 自动识别）
├── README.md                           # 本文件
├── references/
│   ├── pipeline.md                     # 端到端运行手册
│   ├── phaseA/
│   │   ├── workflow.md                 # Stage 1 → Stage 5 完整流程
│   │   ├── conversation_framework.md
│   │   ├── preview-flow.md
│   │   ├── shell-injection.md          # 工作流壳子图片注入
│   │   ├── retouch.md                  # Stage 5.5 retouch (去水印 / IOPaint)
│   │   └── style-system.md
│   └── phaseC/
│       └── workflow.md                 # Phase C 完整流程 (C1-C6)
├── templates/                          # 4 个核心规划文件模板
│   ├── content_report_reference.md
│   ├── design_spec_reference.md
│   ├── slide_blueprint_reference.md
│   └── spec_lock_reference.md
├── assets/                             # 4 个工作流壳子（HTML）
│   ├── preview_shell/                  # Stage 2 风格预览
│   ├── candidate_picker_shell/         # Stage 4 多候选选图
│   ├── review_shell/                   # Stage 5 评审返修
│   └── editor_shell/                   # Phase C 文字编辑器
└── scripts/                            # 10 个脚本
    ├── preflight.py                     # ★ Stage 0 自动自检 + 自动装包（agent 默默跑）
    ├── doctor.py                        # 纯环境检查（不动用户系统）
    ├── render_review_markup.py          # Phase A 评审标注渲染
    ├── inject_shell_images.py          # 把真实图注入 3 个 Phase A 壳子
    ├── remove_corner_watermark.py      # Stage 5.5 批量去角落水印
    ├── setup_iopaint.py                # IOPaint 首装 + 模型预热（幂等）
    ├── launch_iopaint.py               # 启动 IOPaint（未装则自动装）
    ├── detect_reserved_zones.py        # Phase C 擦字稿留白校验
    ├── inject_editor_deck.py           # 把 deck.json 注入编辑器壳子
    └── json_to_pptx.py                 # Phase C 渲染器（deck.json → PPTX）
```

---

## 设计原则

- **单技能自包含**：所有依赖都在本目录内或运行时自装。
- **Phase A 默认交付图片版**：图片型 PPTX 本身就是高完成度可交付物。
- **必须主动询问 Phase C**：Phase A 交付后必须问一次"是否要可编辑文字版"，不要默认沉默也不要默认进入。
- **Phase C 用户驱动**：只有用户主动说要才执行，绝不强推。
- **Phase C-only**：用户明确要求跳过图片版时，直接进入 Phase C，不再跑 Phase A。
- **Phase C 优先直编**：先直接编辑 Phase A 成品图；直编不干净再回退到重生成背景 + 擦字稿。
- **deck.json 是 Phase C 单一真相源**：HTML 编辑器和 PPTX 渲染器都消费它。
- **字体限定 SAFE_FONT_SET**：跨平台稳定。
- **逐页可恢复**：所有 manifest 都保留，中断可从未完成页继续。
- **失败兜底链条**：IOPaint 装不上 → 角落水印批处理 → ImageMagick 矩形遮罩 → 最差也能交付 Phase A 原图。

---

## 致谢

本技能起源于 [`ppt-image-first`](https://linux.do/) 的 Phase A 工作流原型。Phase C 直编优先 + 回退式擦字流程、Stage 5.5 retouch 工具链、IOPaint 自动安装与启动器、HTML 编辑器 + JSON→PPTX 渲染器 —— 本 skill 自有。商用时请同时标明上述来源。
