# Phase C — 背景图 + 外挂可编辑文字

> **核心思想**：不要让图被"压平"。视觉密度高的部分（装饰、图表、艺术字、布局骨架）作为**一张背景图**生成；文字作为**外挂图层**用 HTML 编辑器实时调整，最终按已知坐标 + 已知样式渲染到 PPTX。
>
> 这条路径的特点：
> - 文字**始终是真 TextBox**，PowerPoint / Keynote 里直接可改
> - 不需要任何 OCR / bbox 反推 / 字号反推
> - 不需要"贴框文字漂移"的多道 QA 闸门
> - 每页 imagegen 成本约 2 张（背景完整稿 + 擦字稿）

---

## 何时用 Phase C

| 场景 | 路径 |
|---|---|
| 只要好看视觉稿，做完汇报就结束 | **Phase A**（默认） |
| Phase A 完成后用户在主动询问中说要可编辑 | **Phase C** |
| 用户从一开始就说要可编辑、要后期改字 | 跑完 Phase A → 在主动询问时进 Phase C |

---

## 总流程

```
Phase A Stage 1 / 1.25 / 1.5 / 2 / 2.5 / 2.75 / 3
（沿用 Phase A 的对话式定稿与规划文件）
                       │
                       ▼
            ┌─ Phase C 替代 Phase A Stage 4-5 ─┐
            │                                  │
            ▼                                  │
   Step C1  双轨生成背景：                      │
            (1) imagegen 出"完整稿"             │
                （含装饰文字、给出版式参考）       │
            (2) imagegen edit target = 完整稿    │
                → 出"擦字稿"                   │
                （只保留装饰文字 / 艺术字，      │
                 擦掉所有可编辑标题/正文/说明）   │
                       │                       │
                       ▼                       │
   Step C2  detect_reserved_zones.py 校验：    │
            擦字稿的预留区是否真的留白了        │
            → 不合规：重出 / IOPaint 局部擦    │
                       │                       │
                       ▼                       │
   Step C3  写 deck.json：                     │
            每页 background + 一组 text_boxes  │
            (text_boxes 内容来自 content_report │
             和 slide_blueprint)                │
                       │                       │
                       ▼                       │
   Step C4  inject_editor_deck.py 注入：       │
            assets/editor_shell/index.html     │
            + deck.json → editor.html          │
            → 浏览器打开                        │
                       │                       │
                       ▼                       │
   Step C5  用户在编辑器里：                    │
            - 改文字内容                        │
            - 拖动文字框位置                    │
            - 调字号 / 颜色 / 对齐              │
            - 增删文字框                        │
            → 点 "导出 deck.json"               │
                       │                       │
                       ▼                       │
   Step C6  json_to_pptx.py：                  │
            deck.json → <topic>.pptx           │
            可选 --preview-dir 出每页对照图     │
            └──────────────────────────────────┘
                       │
                       ▼
            交付：<topic>.pptx + deck.json + backgrounds/
```

---

## Step C1 — 双轨生成背景（关键一步）

### 为什么要"双轨"

如果直接让 imagegen "出一张留白的背景"，模型经常翻车：
- 留白区位置不对 / 大小不合规
- 偷偷加 LOREM IPSUM 占位文字
- 整页太空洞，失去 image-first 的视觉密度优势

双轨生成的逻辑：**让模型先正常发挥（出完整稿），再用 imagegen 的图像编辑能力把不要的部分擦掉**。这样：
- 第一稿决定**视觉密度和构图**（保持 Phase A 的视觉风格）
- 第二稿决定**留白位置**（基于第一稿的真实版式而不是凭空猜）

### 操作

**第 1 稿（完整稿）prompt 范例：**

```
按以下 design_spec 出一页 16:9 演示稿封面：
- 主题：<topic>
- 风格方向：<已确认的风格>
- 配色：<已确认的色板>
- 字体：装饰字可用任意；普通正文用 system-safe 字体（PingFang SC / Arial）
- 信息密度：高
内容（按 slide_blueprint）：
- 主标题：<XXX>
- 副标题：<XXX>
- 装饰元素：<XXX>
- 避免大面积纯绿（与下游冲突）
```

**第 2 稿（擦字稿）prompt 范例（必须以第 1 稿为 edit target）：**

```
[先 view_image 第 1 稿]

以刚刚显示的这张图片作为唯一编辑目标 / edit target。

请：
1. 保留所有装饰元素（图表、装饰图形、底纹、艺术字 logo）
2. 保留所有非常规字体的艺术字 / 徽章字
3. 擦除所有可编辑文字（标题、副标、正文、说明、章节号等普通字体文字）
4. 被擦除的区域用与周围背景一致的纯色或渐变填充
5. 不要新增任何文字
6. 不要改变构图、配色、装饰元素的位置和大小
```

### 落盘

```
phaseC/
├── backgrounds/
│   ├── 01-full.png       # 第 1 稿（视觉参考）
│   ├── 01.png            # 第 2 稿（实际用作背景）
│   ├── 02-full.png
│   ├── 02.png
│   └── ...
```

---

## Step C2 — 校验预留区是否留白

为每页写一个 zones.json，对应"应当被擦干净"的矩形：

```json
{
  "zones": [
    {
      "id": "title",
      "x": 0.08, "y": 0.15, "w": 0.55, "h": 0.18,
      "expected_color": "#f5f1ea",
      "tolerance": 25
    },
    {
      "id": "subtitle",
      "x": 0.08, "y": 0.36, "w": 0.55, "h": 0.10
    }
  ]
}
```

跑：

```bash
python3 scripts/detect_reserved_zones.py \
    phaseC/backgrounds/01.png \
    phaseC/01-zones.json \
    --report phaseC/01-zones.report.json
```

输出会逐个 zone 打 ✅/❌，不合规的指明原因（颜色方差太高 / 边缘密度太高 / 平均色偏离）。

**不合规时怎么办**：
- 回 Step C1 重出第 2 稿（调 prompt 强调"必须擦干净 X 区域"）
- 局部用 IOPaint 涂抹擦干净（`launch_iopaint.py --slides-dir phaseC/backgrounds`）
- 实在改不动 → 把该 zone 的位置移到合规区域，调整 deck.json

阈值预设：
- 默认：`std≤12, edge_ratio≤0.02`
- `--strict`：`std≤6, edge_ratio≤0.008`（要求几乎纯净的背景）
- `--loose`：`std≤20, edge_ratio≤0.05`（背景本来就有渐变 / 纹理时）

---

## Step C3 — 写 deck.json

完整 schema 见 `scripts/json_to_pptx.py` 顶部 docstring。最小例子：

```json
{
  "deck": {
    "ratio": "16:9",
    "default_font": "PingFang SC",
    "default_font_size_pt": 18,
    "default_color": "#1a1a1a"
  },
  "slides": [
    {
      "id": "01-cover",
      "background": "backgrounds/01.png",
      "text_boxes": [
        {
          "id": "tb-title",
          "text": "演示稿主标题",
          "x": 0.08, "y": 0.18, "w": 0.55, "h": 0.18,
          "font_size_pt": 56,
          "bold": true,
          "color": "#1a1610"
        },
        {
          "id": "tb-subtitle",
          "text": "副标题或简短说明",
          "x": 0.08, "y": 0.38, "w": 0.55, "h": 0.08,
          "font_size_pt": 22,
          "color": "#5b5247"
        }
      ]
    }
  ]
}
```

**关键约束**：
- 所有 `x / y / w / h` 都是 **fraction (0-1)**，跟 PPTX 渲染器、HTML 编辑器同语义。
- `font_family` 必须从 SAFE_FONT_SET 里选（见 `json_to_pptx.py` 顶部），不在集合里会有警告。
- 默认值放 `deck.default_*`，每页/每框可单独覆盖。
- 背景路径相对 deck.json 解析；绝对路径也支持。

---

## Step C4 — 把 deck.json 注入编辑器

```bash
python3 scripts/inject_editor_deck.py \
    --shell assets/editor_shell/index.html \
    --deck  phaseC/deck.json \
    --out   phaseC/editor.html

# 用浏览器打开
open phaseC/editor.html              # macOS
xdg-open phaseC/editor.html          # Linux
```

模式：
- 默认：背景图重写成 `file://` 绝对路径（适合本地）
- `--inline`：背景图 base64 嵌入 HTML（适合邮件 / 离线分发，文件会大）
- `--keep-paths`：保留原路径（适合背景已经是 HTTPS URL）

> 注意：`editor.html` 导出的最终 deck 里，`background` 可能是 `file://...` 或 `data:...`。
> `scripts/json_to_pptx.py` 必须直接支持这两种形式，不能只认裸本地路径。

---

## Step C5 — 在编辑器里调整

界面布局：

```
┌──────── topbar (导入/导出/添加) ────────┐
├──────┬───────────────────┬─────────────┤
│ 页面 │   画布 (1:背景图) │  属性面板    │
│ 列表 │   + 文字框        │  (字体/字号/ │
│      │                   │   颜色/对齐) │
└──────┴───────────────────┴─────────────┘
```

操作：
- **单击文字框** → 选中（出现 resize handle）
- **拖动** → 改位置（fraction 实时更新）
- **拖 resize handle** → 改尺寸
- **双击** → 进入文字编辑（contenteditable），Esc 退出
- **Delete / Backspace（非编辑态）** → 删除选中的文字框
- **+ 添加文字框** → 在画布中央插入新的
- **属性面板** → 改字体、字号、颜色、对齐、行距
- **导出 deck.json** → 把当前状态的 JSON 复制 / 保存

**重要**：编辑器是单文件 HTML，关闭浏览器不会自动保存——**改完一定要先点导出**。

---

## Step C6 — 渲染 PPTX

```bash
# 拿编辑器导出的 deck.json
python3 scripts/json_to_pptx.py phaseC/deck.json -o phaseC/<topic>.pptx

# 同时出每页预览图（PIL 渲染的近似图，方便和编辑器对照）
python3 scripts/json_to_pptx.py phaseC/deck.json \
    -o phaseC/<topic>.pptx \
    --preview-dir phaseC/preview
```

`json_to_pptx.py` 的实现要点：
- 每页 = 空白版式 + 背景 Picture (占满整页) + 若干 TextBox
- TextBox 的 `font.name` / `font.size` / `font.color` / `bold` / `italic` 严格按 deck.json 写
- 字体不在 SAFE_FONT_SET 会 stderr 警告（仍写入，跨平台可能掉 fallback）
- 颜色支持 `#RRGGBB` / `#RGB`
- 对齐：`left` / `center` / `right`，垂直对齐：`top` / `middle` / `bottom`
- 背景支持相对路径、绝对路径、`file://...`、`data:...`

预览图是 PIL 直接画的（不是真从 PPTX 导出），用于和编辑器视觉对照。要 100% 精确的预览，请用 LibreOffice/Keynote/PowerPoint 转 PNG。

---

## Phase A 与 Phase C 对比

| 模块 | Phase A | Phase C |
|---|---|---|
| Stage 1-3 (对话/内容/规划) | ✅ 跑 | ✅ 复用 Phase A 已生成的 |
| Stage 4-5 出图/评审 | ✅ 整页直出 | **替换**为分层生成 + HTML 编辑器 |
| Stage 5.5 retouch | ✅ 用户驱动 | 也可以用（擦字稿不合规时局部擦） |
| PPTX 渲染 | python-pptx 简单 picture（背景占满整页） | `json_to_pptx.py`（背景 Picture + 文字 TextBox） |
| 用户可编辑性 | 仅整图替换 | **文字真可编辑，背景固化** |
| 每页 imagegen 成本 | 1 张 | 2 张（完整稿 + 擦字稿） |
| QA 闸门数量 | 1 (review) | 1 (detect_reserved_zones) |

---

## 失败排错

| 现象 | 原因 | 处理 |
|---|---|---|
| 第 2 稿 imagegen 没真擦掉文字 | prompt 太弱 / view_image 没指对图 | 重出，prompt 加强"擦除所有可编辑文字"；确认 view_image 指当前页第 1 稿 |
| 第 2 稿擦掉过头，连装饰也没了 | prompt 没强调"保留装饰元素" | 重出，prompt 列出要保留的元素清单 |
| detect_reserved_zones 不合规 | 留白区颜色不一致 / 仍有残留 | IOPaint 局部擦 → 再校验 |
| 文字在编辑器里好看但 PPTX 里偏 | 字体不在 SAFE_FONT_SET，跨平台 fallback | 改用安全字体集里的同类字体 |
| PPTX 里文字溢出框 | 字号太大 / 框太小 | 编辑器里调；或开 `auto_fit: true` |
| 行距异常 | line_spacing 单位用错（应该是倍数，1.0~3.0） | 改回倍数（默认 1.2） |
| 多语言混排字体掉色 | 不同 run 字体覆盖不全 | 每行明确指定 font_family；或全局用 PingFang/Microsoft YaHei（CJK 强） |
| 背景图在编辑器里不显示 | 浏览器跨目录禁访问 | 用 `--inline` 重新注入，或保证 file:// 路径正确 |
| `json_to_pptx.py` 读不出 editor 导出的 deck | 背景是 `file://` / `data:`，旧版渲染器只认本地路径 | 升级到本版渲染器，或先把背景改回相对本地路径 |

---

## 极简清单（agent checklist）

```
- [ ] 走完 Phase A Stage 1 / 1.25 / 1.5 / 2 / 2.5 / 2.75 / 3
- [ ] 用户在 Stage 3 后选择走 Phase C（要可编辑）而不是 Phase A Stage 4-5
- [ ] 对每页：
      - [ ] 出第 1 稿（完整稿）→ phaseC/backgrounds/NN-full.png
      - [ ] view_image 第 1 稿 → 出第 2 稿（擦字稿）→ phaseC/backgrounds/NN.png
      - [ ] 写 phaseC/NN-zones.json，跑 detect_reserved_zones.py 校验
      - [ ] 不合规则重出或 IOPaint 局部擦
- [ ] 写 phaseC/deck.json（背景路径 + text_boxes 初始内容来自 slide_blueprint）
- [ ] 跑 inject_editor_deck.py → 打开 editor.html 让用户调整
- [ ] 用户改完后用导出按钮拿到最终 deck.json
- [ ] 跑 json_to_pptx.py → 交付 .pptx + 每页对照预览图
```
