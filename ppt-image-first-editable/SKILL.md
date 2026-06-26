---
name: ppt-image-first-editable
description: >-
  端到端 PPT 全流程，单技能自包含。用户首次触发本 skill 时，agent 必须默默跑 scripts/preflight.py 做环境自检 + 自动安装缺失的 Python 包；FAIL 才打断用户。
  环境通过后走 conversation-first + image-first 工作流：多阶段对话、内容补强、风格预览、规划锁定、出每页定稿图，并在用户驱动下做图像级 retouch（去水印 / 去瑕疵，内置 IOPaint 自动安装）。最终交付高完成度图片型 PPTX。
  Phase A 结束后必须主动询问用户是否要可编辑文字版（Phase C）；若同意，进入 Phase C：分层重新生成"带装饰、留白文字区"的背景图 + HTML 编辑器调文字 → 一键渲染原生可编辑 PPTX。
  用户明确要求"只做可编辑版 / 只跑 Phase C / 跳过图片版"时，直接进入 Phase C-only 模式，不执行 Phase A 的任何生成、预览、review、retouch。
  ★ 当对话中出现 ===PPT-IMAGE-FIRST-EDITABLE / DECK FROM EDITOR=== 或 ===DECK JSON BEGIN/END=== 标记时，agent 必须立刻把标记之间的 JSON 原样写入 phaseC/deck.json 并跑 scripts/json_to_pptx.py 渲染，不解读不修改不追问。
  当用户需要做汇报 / 答辩 / 路演 / 提案 PPT，或只丢一个主题想要完整成品时使用。
---

# ppt-image-first-editable — 一条龙 PPT 技能（自包含）

---

## ★★★ Stage 0 — 用户首次触发本 skill，agent 默认自动跑环境自检 + 自动修复 ★★★

**这是本 skill 最高优先级的规则，覆盖一切其他规则。**

用户首次说要做 PPT、首次触发本 skill 时，agent **不需要问用户**，**直接默默跑**：

```bash
python3 scripts/preflight.py
```

`preflight.py` 会：
1. 检查 Python 版本（不够则停，让用户处理）
2. 检查 4 个必备 Python 包，**缺的自动 pip install**（含 --user fallback；首次可能 2-5 分钟，OpenCV 较大）
3. 装完后自动调 `doctor.py` 做完整体检（字体 / 磁盘 / 网络 / IOPaint 状态 / skill 完整性）
4. 全部通过 → 退码 0 → agent 进入 Stage 1
5. 仍有阻塞 → 退码 1 → agent 把缺什么和怎么修念给用户听，等用户处理

### preflight 自动会做的事 vs 不会做的事

| 类别 | preflight 会自动做 | 不会做（请求用户处理） |
|---|---|---|
| Python 包 | ✅ `pip install python-pptx pillow numpy opencv-python` | — |
| Python 版本 | — | ❌ 不替用户装新 Python（动用户系统太危险） |
| 系统字体 | — | ❌ 跨平台不一致 + 需要 sudo |
| IOPaint | — | ❌ 那是 ~3GB / 5-10 分钟的活，只在 Stage 5.5 真正用到时再装 |
| 磁盘空间 | ✅ **早期就查并按三档处理**（见下） | ❌ 不会自动清磁盘 |
| imagegen 通道 | — | ❌ 那是 agent harness 的事，preflight 探不到 |

### 磁盘空间的三档处理

preflight 在装任何东西**之前**就先查磁盘。这一项很重要——磁盘满会导致 pip 装包到一半失败、生成图片时写盘失败、IOPaint 模型下载失败，错误信息往往很难懂。

| 磁盘剩余 | preflight 行为 | agent 该怎么说 |
|---|---|---|
| **< 0.5 GB** | 直接 ❌ 退码 1 阻塞 | 念给用户："您的磁盘只剩 X GB，连基础依赖都装不下。请清理出至少 4 GB 后再用本 skill。可以删的常见目录：下载目录、回收站、~/.cache、不用的 Docker 镜像。" |
| **0.5 – 2 GB** | ⚠️ 警告但继续 | 装包前告诉用户："您的磁盘只剩 X GB，能装但很紧张。如果后面要修瑕疵（用 IOPaint），还要再 3GB。建议现在清出更多空间。" |
| **2 – 4 GB** | ✅ 通过 + 提示 IOPaint 需要 3GB | 一笔带过 |
| **≥ 4 GB** | ✅ 通过 | 不用特别提 |

如果 agent 在 Phase A / Phase C 跑到一半看到 `No space left on device`、`OSError: [Errno 28]`、`Could not write file` 这类错误，**立刻停下告诉用户磁盘满了**，别重试浪费 imagegen 调用。

### agent 的行为对照表

| preflight 退出码 | 屏幕上看到 | agent 怎么办 |
|---|---|---|
| **0** | 末尾"✅ 环境就绪" | 简短报告"环境就绪"，进入 Stage 1（**不要再问用户**） |
| **0** + 中间有⚠️ | 末尾"✅ 环境就绪"但 doctor 段有 WARN | 同上 + 顺带提一句"有 N 个可选项缺失，影响 XXX，要不要现在修？" |
| **1** | 末尾"❌ 自检仍有阻塞项" | **必须停**，把 FAIL 项和修复命令念给用户听 |

### 跑 preflight 的反模式（不要做）

- ❌ 不跑 preflight 直接进 Stage 1
- ❌ 跑 preflight 之前先问用户"要不要装 Python 包"——用户不一定知道答案；自动装是默认行为
- ❌ preflight 报 FAIL 还硬着头皮走下去（迟早卡在 Stage 2 或 Phase C 渲染）
- ❌ 把 preflight 当成"每轮都要跑一次"（只在用户首次触发或环境变更后跑）

### 何时算"首次触发"

- 用户在当前对话里第一次说要做 PPT
- 或者用户明说"换了机器 / 重装了环境 / 重跑一次自检"
- 或者上次跑 preflight 后已经过了很长时间（agent 自行判断）

后续轮次只要环境没变，**不需要重跑**——浪费时间。

### 给用户的"我在装东西"提示

如果 preflight 真的在装包（首次场景，可能 2-5 分钟），agent 在等待之前应当告诉用户：

> 第一次使用，正在自动安装 4 个 Python 依赖包（python-pptx / Pillow / numpy / opencv-python）。
> 首次安装可能需要 2-5 分钟，OpenCV 单独就 ~100MB，期间终端会看起来在卡，请稍等。
> 装完会自动继续，您不用做任何操作。

这样用户不会以为卡死了去 Ctrl+C。

### imagegen 通道的额外验证

preflight 和 doctor 都无法直接探测 agent harness 的 imagegen 通道。所以**在 Stage 2 出第一张真实预览之前**，agent 还要再做一次"出图通道可用性测试"：用最小 prompt 出一张极小尺寸的图，确认通道通。这一步失败不算 preflight 失败，但必须**停在 Stage 2 之前**，告诉用户出图通道不可用、不能继续。

---

## ★★★ Phase A HTML 硬门禁（agent 必须自检，不可跳过）★★★

Phase A 流程中**必须有两次让用户在浏览器里看 HTML**——这是这条 skill 区别于"只在对话里贴图说说"的根本动作。
agent 在长对话里很容易忘记打开壳子，所以下面这两个动作是**硬门禁**，跑漏了视同 Phase A 没做完：

| 门禁 | 时机 | 必须打开的文件 | 注入命令 |
|---|---|---|---|
| **G-Preview** | Stage 2 风格预览出图后、用户做"风格确认"前 | `phaseA/previews/preview_filled.html` | `python3 scripts/inject_shell_images.py preview --shell assets/preview_shell/index.html --data phaseA/previews/preview_data.json --out phaseA/previews/preview_filled.html` |
| **G-Review** | Stage 5 全页定稿图出来后、用户做"终图批准"前 | `phaseA/review/review_filled.html` | `python3 scripts/inject_shell_images.py review --shell assets/review_shell/index.html --data phaseA/review/review_data.json --out phaseA/review/review_filled.html` |
| **G-Candidate**（条件触发） | Stage 4 用户选了"多候选 picker"模式时 | `phaseA/candidates/picker_filled.html` | `python3 scripts/inject_shell_images.py candidate --shell assets/candidate_picker_shell/index.html --data phaseA/candidates/candidate_data.json --out phaseA/candidates/picker_filled.html` |

### 每个门禁的最小自检（agent 跑完必须勾选）

- [ ] `*_filled.html` 已生成在上表列的精确路径
- [ ] 已经尝试用 `open` / `xdg-open` / `start` 主动为用户打开它（失败再退路径）
- [ ] 在对话里**显式告诉用户**"已在浏览器打开 `<filename>`，请在里面确认 / 选图 / 给反馈"
- [ ] 等用户反馈（确认 / 复制粘贴回 JSON / 选图编号）后才进入下一 Stage

### 反模式（一旦发生立刻自我纠正）

- ❌ 出完图直接在对话里贴几张缩略图、问"喜欢哪套？" —— 跳过了 G-Preview，**不算过门禁**
- ❌ "我已经生成了 9 张预览图在 phaseA/previews/，请查看" —— 用户不会打开原图；必须开壳子
- ❌ 出完终图直接说"以下是 N 页定稿，您看看？" —— 跳过了 G-Review
- ❌ 打开了 `assets/preview_shell/index.html` 或 `assets/review_shell/index.html`（裸壳，里面是占位 SVG / 示例数据）—— **必须打开 `*_filled.html`**
- ❌ 跑了 `build_*.py` 当成"已经注入" —— `build_*.py` 只还原模板不注入数据，跑完图还是占位
- ❌ G-Preview 还没过就跳到 Stage 3 写 `design_spec.md` —— 风格没经过浏览器确认，规划文件作废
- ❌ G-Review 还没过就导出图片型 PPTX 或主动询问 Phase C —— 终图没经过浏览器评审，交付作废

### 如果环境真的打不开浏览器

只在三件事都成立时才允许跳过"自动打开"动作：
1. agent 真的试过 `open` / `xdg-open` / `start` 都报错
2. 已经在对话里**明确说明**为什么打不开
3. 已经把 `*_filled.html` 的**绝对路径**给到用户，提示他自己双击打开

即使浏览器打不开，**注入步骤本身不能跳**（`*_filled.html` 仍然必须生成）。

---

## Phase C-only 入口

当用户明确说出以下任一意图时，默认进入 **Phase C-only**：

- 只做可编辑版
- 只跑 Phase C
- 跳过图片版
- 直接给我文字可编辑 PPTX

### Phase C-only 必守
- **不执行 Phase A**：不跑 Stage 1-5，不做风格预览，不做图片型 PPTX
- **优先复用现有成果**：如果用户已经给了 Phase A 产物，直接拿来用
- **缺输入先补最小集**：如果缺 `design_spec.md` / `slide_blueprint.md` / `deck.json`，先收齐再进 C1-C6
- **不再主动询问 Phase C**：因为当前会话已经在 Phase C 路径里
- **Phase C 完成即终态**：拿到可编辑 PPTX 后结束，不折返到其他路径

### Phase C-only 的最小输入
- 主题或已有报告稿
- 页数 / 受众 / 目的
- 可复用的内容稿或页面结构
- 如已存在，优先直接使用 `design_spec.md`、`slide_blueprint.md`、`spec_lock.md`

---

把两套互补能力融成一个 skill：

- **Phase A — 对话式定稿 + retouch（默认主路径）**
  Conversation-first + image-first 工作流：多阶段对话 → 内容基底 → 风格预览 → 风格反演确认 → 规划文件 → 每页定稿图 → 用户驱动的图像级 retouch（去水印 / 去瑕疵）。
- **Phase C — 分层生成 + HTML 文字编辑（可编辑路径）**
  Phase A 完成后**主动询问用户是否要可编辑文字版**。若用户同意，优先直接把 Phase A 的成品图当作 edit target 去字，再把文字作为外挂图层在 HTML 编辑器里调整 → 一键渲染成原生可编辑 PPTX。若直接编辑不干净，再回退到重生成背景 + 擦字稿。文字始终是真 TextBox（PPT 里可改）。
  如果用户一开始就明确要求只做可编辑版，则直接走 **Phase C-only**，不进入 Phase A。

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
│  Stage 5.5   ★ retouch（可选，用户驱动）  │
└──────────────┬──────────────────────────┘
               │
               ▼   交付：图片型 PPTX + 全页定稿图 + 规划文档
               │
   ┌───────────┴───────────┐
   │ ★ 必须主动询问用户 ★    │
   │ "PPT 已交付。是否需要   │
   │  可编辑文字版（Phase C）│
   │  ——文字能在 PPT 里直接   │
   │  改？这会重新生成背景，  │
   │  每页约 2 张 imagegen。" │
   └─────────┬─────────────┘
             │
        用户同意 → 进 Phase C
             │
             ▼
┌── Phase C（可编辑文字版）─────────────────┐
│  C1  双轨生成背景                         │
│      (完整稿 → imagegen 擦字稿)           │
│  C2  detect_reserved_zones 校验           │
│  C3  写 deck.json                         │
│  C4  inject_editor_deck.py 注入编辑器     │
│  C5  HTML 编辑器调文字（用户）             │
│  C6  json_to_pptx.py 渲染 PPTX            │
└──────────────┬──────────────────────────┘
               ▼
[追加交付：文字可编辑 PPTX + deck.json + backgrounds/]
```

---

## I/O Contract

### Phase A 必交付（默认）
- `content_report.md`（用户没给完整内容稿时）
- `design_spec.md` / `slide_blueprint.md` / `spec_lock.md`
- 每页定稿图 `phaseA/slides/NN-*.png`
- **图片型 `.pptx`**（高完成度视觉稿，用于汇报/展示——这是默认终交付物）
- `phaseA/imagegen-manifest.json`

### Phase C 追加交付（用户同意走可编辑路径时）
- 每页背景图 `phaseC/backgrounds/NN.png`（+ 第 1 稿 `NN-full.png` 备查）
- `phaseC/deck.json`（单一真相源，编辑器 / 渲染器共同消费）
- 每页 `phaseC/NN-zones.json` + `phaseC/NN-zones.report.json`
- **文字可编辑 `.pptx`**（背景 = Picture，文字 = 真 TextBox，跨平台稳定）
- 可选 `phaseC/preview/slide_NN.png` 近似对照图

### 输入
PPT 主题 / 粗略目标 / 零散材料 / 已有报告稿；可选锚点（受众、页数、身份锚点、用途场景、参考图、风格倾向）。

### 确认门禁
- **3 个必有门禁**（Phase A）：需求确认 → 风格确认 → 生成前确认 → 终图评审
- **第 4 个门禁（强制主动询问）**：Phase A 终图交付后**必须主动问**用户是否需要 Phase C 可编辑文字版
- **Phase C 内的门禁**：用户在 HTML 编辑器里点 "导出 deck.json" 表示满意

### 比例
默认 `16:9`。**Phase A 与 Phase C 全程必须同一比例**，禁止中途切换。

---

## ★ 主动询问 Phase C 的规则（最关键的行为）

**Phase A 全部交付完成后，必须主动询问一次**用户是否要走 Phase C。这是这条 skill 的硬约束。

### 询问时机
- Phase A Stage 5（review）通过 + Stage 5.5 retouch 处理完（如果跑了）
- 已经把图片型 PPTX 和所有定稿图给到用户
- **在结束对话前**

### 询问话术（参考，可调措辞）

> 图片版 PPT 已经做好了：
> - 图片型 PPTX：`<path>`
> - 全 N 页定稿图：`<path>`
> - 规划文档：`<paths>`
>
> 现在的成品是**图片版**，每页是一整张图——好处是视觉密度最高、跨平台不会跑版；缺点是文字不能在 PowerPoint 里直接改。
>
> 如果您后续可能要**改文字**（比如换日期、换名字、换关键数字、改标题措辞），我可以继续走 **Phase C** 给您出一份**文字可编辑版**：
> - 重新生成背景图（保留装饰、擦掉可编辑文字）
> - 文字作为独立图层，在浏览器编辑器里调
> - 一键渲染成原生可编辑 PPTX
> - 大致成本：每页约 2 张 imagegen（背景完整稿 + 擦字稿），N 页约 X 次调用
>
> 要不要进 Phase C？（不需要也可以，图片版本身就能直接拿去汇报）

### 询问后的分支
- 用户说**要 / 好 / 进 Phase C / 我后期还要改字** → 按 `references/phaseC/workflow.md` 跑 C1-C6
- 用户说**不用 / 这样就够了 / 先这样** → 结束对话，不要再追问
- 用户**没正面回答**（比如继续聊别的） → 不要打断，等用户自己提

### 反模式（不要这样做）
- ❌ 不询问就直接进入 Phase C
- ❌ 不询问就直接结束对话（用户可能不知道有可编辑选项）
- ❌ 反复追问、施压让用户走 Phase C
- ❌ 把"是否进 Phase C"作为 confirmation gate 强卡用户（它是一个开放问题，不是必跨门禁）

---

## ★ 用户从编辑器贴回 deck 时的处理规则（极重要）

Phase C Stage 5 用户在 HTML 编辑器调完文字后会点"导出"按钮。**编辑器导出的不是裸 JSON，而是一段以下面这两个 sentinel 包住的"指令包"**：

```
===PPT-IMAGE-FIRST-EDITABLE / DECK FROM EDITOR===
... (说明文字 + 步骤) ...

===DECK JSON BEGIN===
{ ... 完整 deck.json ... }
===DECK JSON END===
```

### 当 agent 在对话里看到这个标记时，**必须**：

1. **绝对不要"解读" / "总结" / "评论" JSON 里的内容**——这是用户已经定稿的数据，你的任务只是搬运 + 渲染。
2. **绝对不要修改 JSON 任何字段**——不优化字号、不调对齐、不改颜色、不重排版面，不替用户做"我觉得这样更好看"的二次决策。
3. **绝对不要用之前生成的 deck.json 老版本**——以贴回来的这一份为最新真相源。
4. **必须执行以下操作**（按顺序）：
   - 把 `===DECK JSON BEGIN===` 和 `===DECK JSON END===` 之间的**完整、原样**内容写入 `phaseC/deck.json`（覆盖旧文件）
   - 跑 `python3 scripts/json_to_pptx.py phaseC/deck.json -o phaseC/edited.pptx --preview-dir phaseC/preview`
   - 把 `phaseC/edited.pptx` 的绝对路径告诉用户，结束本任务

### 反模式（不要做）

- ❌ "我看到您改了某某字，我建议把字号也调一下..." —— 不要建议，直接渲染
- ❌ "JSON 里第 3 页似乎有点问题，要不要..." —— 不要质疑用户的决策
- ❌ "我已经收到 deck.json，请问您要我做什么？" —— sentinel 已经写明了要做什么
- ❌ 用之前生成的 deck.json 老版本去渲染 —— 必须用贴回来的这一份
- ❌ 把贴回来的内容当成"用户要重新走 Phase C" —— 不是，是渲染 PPTX

### 用户行为提示（agent 该告诉用户的）

当 Phase C Stage 5 进入编辑器阶段时，agent 应该明确告诉用户：

> 编辑器已打开。改完文字后：
> 1. 点右上角 **"📋 复制整段（推荐）"** 按钮
> 2. 把复制到剪贴板的**整段内容**（不是只复制 JSON）直接粘贴回这个对话框
> 3. 我看到带 `===DECK JSON===` 标记的内容会自动保存 + 渲染成 PPTX
>
> 如果嫌粘贴麻烦，也可以点 **"下载 deck.json 文件"**，把文件给我。

### Stage 5 失败兜底

如果用户只粘了裸 JSON（没有 sentinel）：
- agent 仍应保存为 `phaseC/deck.json` + 跑渲染
- 但应当告诉用户："下次记得复制整段（带 ===DECK JSON=== 标记），避免我误判您的意图"

如果用户粘了 sentinel 但 JSON 里有错（缺 slides 字段等）：
- agent 报错给用户，**不要**自动猜补
- 让用户回编辑器修后重新导出

---

## 何时使用本技能 / 何时只走 Phase A / 何时上 Phase C

| 用户场景 | 路径 |
|---|---|
| 只要好看的视觉稿、做完汇报就结束 | **Phase A**（默认） |
| 没明确点名，只丢主题、要完整成品 | **Phase A**（默认） |
| Phase A 出图有水印 / 小瑕疵想擦掉 | Phase A + Stage 5.5 retouch |
| Phase A 完成后用户说要可编辑 | **Phase A → Phase C**（按上面"主动询问"规则） |
| 用户一开始就说"要可编辑 / 后期改字" | **Phase C-only**（先补最小输入，再跑 C1-C6） |
| 用户明确说"只做可编辑版 / 跳过图片版" | **Phase C-only** |

---

## Progressive Loading

按需读，不要一上来全读：

| 何时读 | 文件 |
|---|---|
| **★ 用户首次触发本 skill（agent 默默跑）** | `scripts/preflight.py`（自动装 pip 包 + 调 doctor；不需要问用户） |
| 单纯环境检查（不动用户系统） | `scripts/doctor.py`（不会自动装东西，纯报告） |
| 路由 / 决定本技能怎么跑（必读） | 本 `SKILL.md` |
| 跑 Phase A 总流程 | `references/phaseA/workflow.md` |
| Phase A intake / 对话框架 | `references/phaseA/conversation_framework.md` |
| Phase A 出风格预览、候选选图、review 页面 | `references/phaseA/preview-flow.md` |
| Phase A 三个壳子的图片/数据**注入**（必读） | `references/phaseA/shell-injection.md` |
| Phase A 风格提案卡 / V1-V8 内部 | `references/phaseA/style-system.md` |
| Stage 5.5 retouch（去水印 / IOPaint / ImageMagick 兜底） | `references/phaseA/retouch.md` |
| 写 `content_report.md` | `templates/content_report_reference.md` |
| 写 `design_spec.md` | `templates/design_spec_reference.md` |
| 写 `slide_blueprint.md` | `templates/slide_blueprint_reference.md` |
| 写 `spec_lock.md` | `templates/spec_lock_reference.md` |
| **Phase C 总流程**（用户同意进 Phase C 时） | `references/phaseC/workflow.md` |
| 端到端运行手册 + 失败排错 | `references/pipeline.md` |

---

## 内置工作流壳子（assets/）

Phase A 必须使用以下 3 个本技能自带的 HTML 壳子，不要替换或自造同类页面：

- `assets/preview_shell/index.html` — 风格预览比较（Stage 2）
- `assets/candidate_picker_shell/index.html` — 多候选选图（Stage 4 多候选模式）
- `assets/review_shell/index.html` — 评审与返修（Stage 5）

Phase C 增加一个：

- `assets/editor_shell/index.html` — 文字编辑器（C4-C5）

> ⚠️ **四个壳子的 `index.html` 默认是空架子 / 示例数据**：preview_shell 的 9 张图是写死的 SVG 占位图；candidate / review / editor 是写死的示例数据。**真正使用之前必须显式注入真实数据**，否则用户打开页面看到的会是占位。
>
> **统一注入命令**（强制）：
>
> ```bash
> # Stage 2 风格预览
> python3 scripts/inject_shell_images.py preview \
>   --shell assets/preview_shell/index.html \
>   --data  preview_data.json \
>   --out   phaseA/previews/preview_filled.html
>
> # Stage 4 多候选选图
> python3 scripts/inject_shell_images.py candidate \
>   --shell assets/candidate_picker_shell/index.html \
>   --data  candidate_data.json \
>   --out   phaseA/candidates/picker_filled.html
>
> # Stage 5 评审
> python3 scripts/inject_shell_images.py review \
>   --shell assets/review_shell/index.html \
>   --data  review_data.json \
>   --out   phaseA/review/review_filled.html
>
> # Phase C 文字编辑器
> python3 scripts/inject_editor_deck.py \
>   --shell assets/editor_shell/index.html \
>   --deck  phaseC/deck.json \
>   --out   phaseC/editor.html
> ```
>
> - 各 data JSON 的 schema 见对应脚本顶部 docstring。
> - **打开时必须打开 `*_filled.html` / `editor.html`**，不要打开 `assets/.../index.html` 原模板。
> - 默认保留相对路径，配合 HTML 与背景图同目录使用；要打包分发再加 `--inline` 把图片 base64 嵌入 HTML，或加 `--file-url` 改成绝对 `file://`。
> - `editor.html` 默认应与 `phaseC/backgrounds/` 旁置，避免把背景内嵌成 7MB+ 的单文件。
> - `build_*.py` 三个脚本只是把壳子从内嵌 base64 还原成 `index.html`，**它们不注入真实数据**，不要把 build 和 inject 搞混。

---

## Phase A 规则（Conversation-first + Image-first）

**按 `references/phaseA/workflow.md` 跑完所有 Stage**。这里只列死规则。

### Working Principles
- 把用户当成甲方，本技能当作提出方向的设计侧。
- 不强迫用户填一堆设计参数；把自然语言意图翻译成设计决策。
- 默认展示 baseline judgment / proposal cards / 预览 / review 界面；规划文件原文按需出示。
- 标注 `user_provided` / `inferred` / `needs_confirmation`；不擅自编造未授权事实、数据、引用、机构结论。

### Hard Rules（Phase A 必守）
- **Preview-first**：最终风格确认必须基于真实生成的「首页 + 目录页 + 正文页」预览，不能用文字 mockup / ASCII 草图 / 占位壳代替。
- **Shells are mandatory**：必须使用 `assets/preview_shell/index.html`、`assets/candidate_picker_shell/index.html`、`assets/review_shell/index.html`。**打开前必须先用 `scripts/inject_shell_images.py` 把真实图注入到 `*_filled.html`，不要直接打开壳子原模板**。
- **Image-first 不退化**：定稿出图必须 image-first，不允许悄悄退化到"用 PPT shape 拼页面"或"代码画图"兜底。
- 用户要"加字 / 改字 / 补字"也仍然属于图像生成/编辑任务；不要默认用 PIL / Canvas / SVG / HTML 截图 / PPT 原生文本框去后期补字（除非用户明确要求这种 workaround）。
- **生成 ID 不入 prompt**：slide id / candidate code / 文件名 / 批次标签等可以出现在规划文件、文件名、映射表、review UI、对话里，但**不要拼进发给图像模型的 prompt 文本**。
- **不要在第一轮就导出最终 PPT**；只有 review 通过后才导出。
- **`slide_blueprint.md` 不能在风格反演确认之前写**。
- **生成前必须问**：单图直出 / 多候选 picker。
- **Phase A 交付后必须主动询问 Phase C**（见上面"主动询问"段）。

### 内置工作流的 5+ 个 Stage（极简版索引）
1. **Stage 1** — Intake + baseline judgment → `需求确认`
2. **Stage 1.25** — 风格前内容研究 → `content_report.md`（除非用户已给完整稿）
3. **Stage 1.5** — 风格边界对齐（3 个短问题）
4. **Stage 2** — 风格提案与真实预览（首页 / 目录页 / 正文页），打开 `preview_shell`；可选 **Stage 2.5** refinement；必跑 **Stage 2.75** 风格反演确认 → `风格确认`
5. **Stage 3** — 顺序写 `design_spec.md` → `slide_blueprint.md` → `spec_lock.md` → `生成前确认`
6. **Stage 4** — 单图直出 / 多候选 picker（若多候选先打开 `candidate_picker_shell`）出齐全页定稿图
7. **Stage 5** — 打开 `review_shell` 做评审；不通过则用 `scripts/render_review_markup.py` 渲染标注图 + 文本反馈再喂回去返修
8. **Stage 5.5** — ★ **图像级 retouch（用户驱动，可选）**：详见 `references/phaseA/retouch.md`
9. **交付 + 主动询问 Phase C** — 把图片型 PPTX + 定稿图 + 规划文档交给用户，**主动问**是否要可编辑文字版

---

## Stage 5.5 Retouch（默认提供，按需使用）

Phase A 出图被叠加了工具水印（典型如 Qoder 的 "Qoder AI生成"）、或某页角落有想擦掉的小瑕疵时，**默认提供以下两个工具**：

```bash
# 工具 1：批量去固定位置的角落水印（最省事，适合工具水印）
python3 scripts/remove_corner_watermark.py phaseA/slides/ -o phaseA/slides_clean/ --batch

# 工具 2：IOPaint 手工 inpaint（任意位置 / 复杂背景；效果最强）
# 首次自动调 setup_iopaint.py 装 IOPaint + LaMa（约 5–10 分钟、~3GB）
python3 scripts/launch_iopaint.py --slides-dir phaseA/slides
```

**关键性质**（agent 必须遵守）：
- **绝不在用户没要求时强制启动 IOPaint**。Phase A 走完先告诉用户图在哪、PPTX 在哪、有没有发现水印 / 瑕疵，再问要不要进 Stage 5.5。
- **首次启动 IOPaint 会等待 5–10 分钟装环境 + 下模型**，这点必须事先告诉用户。
- **安装完全幂等**：标记文件在 `~/.cache/ppt-image-first-editable/.lama-installed`，二次启动秒开。
- **装失败不阻塞 Phase A 主路径**：launch_iopaint.py / setup_iopaint.py 都内置失败兜底（重试 / `remove_corner_watermark.py` / ImageMagick 矩形遮罩）。
- **retouch 不替代 review**：实质内容 / 视觉改动应回 Stage 5 重出图；Stage 5.5 只解决"图整体没问题，那一小块要擦掉"。
- **IOPaint 在 Phase C 里也能用**——擦字稿不合规时局部涂抹擦干净比让 imagegen 重出整页省钱：`python3 scripts/launch_iopaint.py --slides-dir phaseC/backgrounds`。

完整工具说明、操作流程、何时该用 / 不该用，详见 [`references/phaseA/retouch.md`](references/phaseA/retouch.md)。

---

## Phase C 规则（可编辑文字路径）

**仅当用户在 Phase A 交付后的"主动询问"中同意进入时才执行**。完整流程在 `references/phaseC/workflow.md`，这里只列死规则。

### Hard Rules（Phase C 必守）
- **沿用 Phase A 的 Stage 1-3**：需求 / 内容 / 风格 / 规划文件全部走 Phase A 已经做好的成果，不要重做。
- **优先直编 Phase A 成品图**：Phase C 默认先把 Phase A 定稿图作为 imagegen edit target 去字；如果去字后不干净或留白不合规，再回退到重生成背景 + 擦字稿。
- **回退时再走两稿**：只有在直编失败时，才先出"完整稿"再以完整稿为 edit target 出"擦字稿"。不要把两稿逻辑当成默认必走。
- **edit target 先看 Phase A 图**：默认先 `view_image` Phase A 定稿图，再调 imagegen，prompt 写"以刚刚显示的这张图片作为唯一编辑目标"。不要只写本地路径。
- **detect_reserved_zones 不可跳**：每页擦字稿必须用 `scripts/detect_reserved_zones.py` 校验。不合规 → 重出 / IOPaint 局部擦。
- **deck.json 是单一真相源**：编辑器 / 渲染器都消费 deck.json。**不要在 PPTX 渲染后再单独改 PPTX**——回到 deck.json 改，重出 PPTX。
- **字体限定 SAFE_FONT_SET**：见 `scripts/json_to_pptx.py` 顶部。不在集合里会有警告，跨平台可能掉 fallback。可用：PingFang SC / Microsoft YaHei / Hiragino Sans GB / Arial / Helvetica 等系统字体。
- **坐标用 fraction**：所有 x/y/w/h 都是 0-1，跟 HTML 编辑器和 PPTX 渲染器同语义。
- **字号用 pt**：直接写 `font_size_pt`，不要算像素。
- **背景引用可为相对路径 / 绝对路径 / `file://` / `data:`**：编辑器导出的 deck 可能带这些形式，`json_to_pptx.py` 必须直接兼容。
- **编辑器默认走旁置文件模式**：`inject_editor_deck.py` 默认保留相对路径，`editor.html` 与 `phaseC/backgrounds/` 放同一目录；只有明确需要时才用 `--inline`。
- **背景图固化、文字外挂**：背景一旦生成就不再改（除非重出该页）；文字始终是外挂层。
- **编辑器是单文件 HTML**：关闭浏览器不会自动保存。改完一定要先点"导出 deck.json"再关。
- **Phase C 完成 = 终态**：拿到可编辑 PPTX 后不要再追问 / 折返到其他路径。

### Phase C 流程极简版（C1-C6）
1. **C1** 先直编 Phase A 成品图；不干净再回退到完整稿 → imagegen 擦字稿
2. **C2** `scripts/detect_reserved_zones.py` 校验预留区
3. **C3** 写 `phaseC/deck.json`（每页 background + text_boxes）
4. **C4** `scripts/inject_editor_deck.py` 把 deck.json 注入 `assets/editor_shell/index.html`，默认生成旁置文件版 `editor.html`
5. **C5** 用户在 HTML 编辑器里调文字 → 导出新的 deck.json
6. **C6** `scripts/json_to_pptx.py` 渲染 → 可编辑 PPTX

---

## 关键命令（脚本都在 `scripts/`）

```bash
# ─── Stage 0：用户首次触发，agent 默默跑（无需问用户）──────

python3 scripts/preflight.py
# 自动安装缺失的 4 个 pip 包（首次 2-5 分钟），装完调 doctor
# 退出码 0 = 可继续；退出码 1 = 有不可自动修的项目，让用户处理

# 单纯想看环境状态（不动用户系统）
python3 scripts/doctor.py

# ─── 基础依赖（preflight 失败时手动装的兜底方案）─────────
pip3 install python-pptx pillow numpy opencv-python

# ─── Phase A ───────────────────────────────────────────────

# 把真实图注入工作流壳子（每个 Stage 出图后必跑）
python3 scripts/inject_shell_images.py preview   --shell assets/preview_shell/index.html           --data preview_data.json   --out phaseA/previews/preview_filled.html
python3 scripts/inject_shell_images.py candidate --shell assets/candidate_picker_shell/index.html  --data candidate_data.json --out phaseA/candidates/picker_filled.html
python3 scripts/inject_shell_images.py review    --shell assets/review_shell/index.html            --data review_data.json    --out phaseA/review/review_filled.html

# review markup 渲染（用户在 review_shell 给的坐标标注 → 标注图）
python3 scripts/render_review_markup.py <review.json> --out-dir <dir>

# ─── Stage 5.5 Retouch（用户驱动，可选）────────────────────

# 批量去角落水印
python3 scripts/remove_corner_watermark.py phaseA/slides/ -o phaseA/slides_clean/ --batch

# IOPaint：首次自动装环境 + 下 LaMa（5–10 分钟、~3GB），之后秒开
python3 scripts/launch_iopaint.py --slides-dir phaseA/slides
# Phase C 也可以用（擦字稿不合规时局部涂抹）
python3 scripts/launch_iopaint.py --slides-dir phaseC/backgrounds

# 单独管理 IOPaint
python3 scripts/setup_iopaint.py --check-only     # 检查
python3 scripts/setup_iopaint.py                  # 装
python3 scripts/setup_iopaint.py --reinstall      # 强制重装

# ─── Phase C（用户同意可编辑后才跑）─────────────────────────

# C2 校验擦字稿的预留区是否真的留白
python3 scripts/detect_reserved_zones.py phaseC/backgrounds/01.png phaseC/01-zones.json --report phaseC/01-zones.report.json

# C4 把 deck.json 注入编辑器壳子
python3 scripts/inject_editor_deck.py \
    --shell assets/editor_shell/index.html \
    --deck  phaseC/deck.json \
    --out   phaseC/editor.html

# C6 deck.json → 可编辑 PPTX（+ 可选 PIL 近似预览图）
python3 scripts/json_to_pptx.py phaseC/deck.json -o phaseC/<topic>.pptx --preview-dir phaseC/preview
```

---

## 输出目录结构（建议）

```
<topic-slug>/
├── content_report.md                    # 仅当 Stage 1.25 生成
├── design_spec.md
├── slide_blueprint.md
├── spec_lock.md
│
├── phaseA/                              # Phase A 产物（默认交付）
│   ├── previews/                        # 各阶段风格预览（保留备查）
│   ├── candidates/                      # 多候选模式时使用
│   ├── slides/NN-*.png                  # 评审通过的终图
│   ├── slides_clean/                    # Stage 5.5 retouch 后的图（可选）
│   ├── review/                          # 评审产物
│   ├── imagegen-manifest.json
│   └── <topic>-image-deck.pptx          # 图片型 PPTX
│
└── phaseC/                              # Phase C 产物（用户同意才出现）
    ├── backgrounds/
    │   ├── NN-full.png                  # 第 1 稿（完整稿，备查）
    │   └── NN.png                       # 第 2 稿（擦字稿，实际背景）
    ├── NN-zones.json                    # 每页预留区声明
    ├── NN-zones.report.json             # 校验报告
    ├── deck.json                        # 单一真相源
    ├── editor.html                      # 注入后的编辑器
    ├── preview/slide_NN.png             # PIL 近似预览图（可选）
    └── <topic>-editable.pptx            # 文字可编辑 PPTX
```

外置缓存（IOPaint 用，不在本目录里）：

```
~/.cache/ppt-image-first-editable/
├── venv/                                # IOPaint 专属 venv
├── .lama-installed                      # 安装标记 + 元数据
└── setup.log                            # 安装日志
```

---

## 安装 / 依赖

把整个 `ppt-image-first-editable/` 复制到 agent 的 skills 目录即可。**完全自包含**，不依赖其他外部 skill。

```bash
# Codex
cp -R ppt-image-first-editable "${CODEX_HOME:-$HOME/.codex}/skills/ppt-image-first-editable"
```

Python 依赖（基础）：

```bash
pip3 install python-pptx pillow numpy opencv-python
```

Stage 5.5 IOPaint 是**用户首次触发时自动安装**：

- 装到专属 venv `~/.cache/ppt-image-first-editable/venv/`，不污染系统
- 自动 + 预下 LaMa 模型
- 国内默认走 hf-mirror 镜像
- 首次约 5–10 分钟、~3GB 磁盘；之后秒启
- 装失败有明确兜底（remove_corner_watermark.py / ImageMagick）

图像生成路径：依赖运行环境内可用的 imagegen 通道（Codex 用内置 `imagegen` / GPT Image 2）。如果运行时不可用，必须停在该 Stage 并向用户说明阻塞原因，不允许用 PIL/SVG/Canvas/PPT shapes 兜底。

---

## 致谢

本技能起源于：
- `ppt-image-first`（Phase A 工作流、references、templates、assets 部分原型） — Linux.do 社区

编排层、Stage 5.5 retouch 工具链、IOPaint 自动安装与启动器、Phase C 全套（双轨生成 + detect_reserved_zones + HTML 编辑器 + json_to_pptx 渲染器）—— 本 skill 自有。商用时请同时标明上述来源。
