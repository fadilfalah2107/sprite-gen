<p align="center">
  <img src="docs/claudecy-idle.gif" width="110" alt="claudecy idle" />
  <img src="docs/claudecy-running.gif" width="110" alt="claudecy running" />
  <img src="docs/claudecy-success.gif" width="110" alt="claudecy success" />
  <img src="docs/claudecy-talking.gif" width="110" alt="claudecy talking" />
  <img src="docs/howl-idle.gif" width="110" alt="howl idle" />
  <img src="docs/howl-running.gif" width="110" alt="howl running" />
  <img src="docs/howl-success.gif" width="110" alt="howl success" />
</p>

<h1 align="center">sprite-gen</h1>

<p align="center"><b>只要一张原图。即可产出可直接用于游戏的 sprite 图集。</b></p>

<p align="center">

**English** · [한국어](README.ko.md) · [日本語](README.ja.md) · [简体中文](README.zh-Hans.md) · [Español](README.es.md) · [Français](README.fr.md)

</p>

---

当你向图像模型请求一张“sprite sheet”时，你通常会得到：每帧都在变的角色、无法抠出背景的画面、重叠漂移的动作姿态，以及一张你的游戏引擎根本无法直接使用的 PNG。演示效果很可爱，但资产毫无实用价值。

`sprite-gen` 是一个用于弥补这一鸿沟的 Codex/Claude skill。给它 **一张基础图像** 和一组动作，它会按行驱动生成过程，锁定角色身份，去除色度背景并转换为真实 alpha，提取每个动作为干净透明的帧，并烘焙一个可运行时使用的图集，附带可机器读取的 `manifest.json.frame_layout`。上面所有精灵都是这样生成的。

而对于生成结果中最后 10% 的缺陷，`sprite-gen` 提供了 **curation webview**：并排比较帧、剔除损坏帧、非破坏性微调旋转/缩放/位置、实时预览循环，然后再烘焙。流水线做脏活，你保留审美。

```text
sprite-request.json → layout guides + prompts → image-gen state rows
→ chroma alpha → connected components → transparent frames
→ sprite-sheet-alpha.png + manifest.json.frame_layout
```

<p align="center">
  <img src="docs/architecture-diagram.png" width="640" alt="sprite-gen architecture — component-row pipeline" />
</p>

> 完整架构：[`docs/architecture.md`](docs/architecture.md) · 图表源文件：[`docs/architecture-diagram.html`](docs/architecture-diagram.html)

## 你能实际得到什么

- **透明精灵图集**（`sprite-sheet-alpha.png`）——真实 alpha，无残留色度边缘，并且在白色背景下通过校验。
- **运行时清单**（`manifest.json.frame_layout`）——绝对坐标帧矩形、每个状态的 fps 与循环标记。你的引擎读取矩形，不再猜测网格。
- **可观测 QA**——每个状态都有 GIF 与 contact sheet，在发布前先看动作本身，而不是只看单帧。
- **诚实标注**——短小可读的动作名（idle, jump, attack, wave）作为稳定主路径；循环位移类动作（walk/run）仅在运动 QA 实际通过后才标记为非实验性。没有无声的过度承诺。

## Curation webview

生成通常能拿到 90% 的结果。webview 是把内容推进“可发布”状态的地方——独立运行，无需依赖 Studio 或框架，任何已安装该 skill 的环境都可使用（Claude Code Desktop、Codex app、普通终端）。

![curation webview — characters](docs/demo-character.gif)

- **每个状态两行：**上方是 **播放序列**，下方是 **候选池**（例如第二次或第三次生成结果）。拖动帧上的 ⠿ 把手可重排顺序，或从候选池抽取一帧——从多次生成中重建一条干净的循环。布局会被保存，重新打开后自动恢复。
- **每帧非破坏性变换：**拖拽=移动，滚轮=缩放，顶部手柄=旋转，左下角=剪切，并提供水平翻转开关以输出左右反转。编辑保存在 `curation.json` 副文件中——源 PNG 从不被重写，compose 步骤会确定性地烘焙结果。预览与烘焙共用同一仿射矩阵，所见即所得。
- **实时预览** 按状态 fps 播放序列，支持播放/暂停、逐帧步进，以及 0.25×–4× 的速度控制。
- 不仅适用于 sprite：通过 `unpack_atlas_run.py --pngs-dir` 可指向任何候选图片文件夹（图标、徽标、生成草图），将其当作通用的择优浏览与选择界面。

### 等距地面网格

对于等距地面素材，webview 会叠加来自 `meta.json` tile/anchor 的地面网格，让你可用剪切手柄把家具吸附到菱形坐标轴上。

![curation webview — isometric furniture](docs/demo-furniture.gif)

<img src="docs/curator-iso.png" width="520" alt="等距地面网格叠加" />

### 语言

webview 默认支持英文和韩文。启动时使用 `--lang en|ko` 指定，或在应用内切换语言：

```bash
python3 scripts/serve_curation.py --run-dir <run-dir> --lang en   # 或 ko
```

## Python 支持

`sprite-gen` 支持 CPython 3.10+。CI 在 GitHub 托管的 runner 上会对最低支持版本（3.10）和最高覆盖版本（3.14）进行验证。

快速开始需要可用 `venv`/`ensurepip` 的 Python 安装。如果在本地环境中在安装包之前执行 `python3 -m venv` 失败，请改用标准 CPython 构建（任意受支持版本）并重复执行同样的命令。

## Quickstart

```bash
# 0. 在全新虚拟环境中安装依赖（Pillow）
python3 -m venv .venv && source .venv/bin/activate
pip install -e .

# 1. 使用基础图像准备一次 run
python3 scripts/prepare_sprite_run.py --out-dir <run-dir> --character-id <id> --base-image base.png

# 2. 使用 image-gen 为每个 state 生成一行图像，保存为 raw/<state>.png
# 3. 提取帧
python3 scripts/extract_sprite_row_frames.py --run-dir <run-dir>

# 4. （可选）在 webview 中整理帧
python3 scripts/serve_curation.py --run-dir <run-dir>

# 5. 烘焙运行时图集
python3 scripts/compose_sprite_atlas.py --run-dir <run-dir>
```

### 编辑已完成的图集

当仅剩合并后的图集时，先重建一个可供 curator 使用的 run 目录，再进行整理并导出：

```bash
# 重建帧：显式使用 --grid、--manifest 矩形，或 alpha 自动检测（默认）
python3 scripts/unpack_atlas_run.py --atlas sheet.png            # auto-detect
python3 scripts/unpack_atlas_run.py --manifest manifest.json     # exact rectangles
python3 scripts/unpack_atlas_run.py --pngs-dir furniture/        # 导入一组散列 PNG

# 完成整理后，将修正结果导回命名 PNG
python3 scripts/export_curated_pngs.py --run-dir <run-dir>
```

输出默认是输入旁边可定位的 `<source>-curator` 文件夹。

面向 Agent 的完整工作流与协议位于 [`SKILL.md`](SKILL.md)。

## 安装

在 Codex skill 安装器工作流中，将本仓库作为 root skill 安装：

```bash
python3 ~/.codex/skills/.system/skill-installer/scripts/install-skill-from-github.py \
  --repo aldegad/sprite-gen --path .
```

## Attribution

component-row 工作流受 Apache-2.0 授权的 `hatch-pet` skill 启发，但面向通用游戏精灵图集，并且不包含任何宠物包或宠物视觉资产。

## 许可证

Apache-2.0