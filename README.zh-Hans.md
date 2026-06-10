<p align="center">
  <img src="docs/claudecy-idle.gif" width="110" alt="claudecy 空闲" />
  <img src="docs/claudecy-running.gif" width="110" alt="claudecy 运行" />
  <img src="docs/claudecy-success.gif" width="110" alt="claudecy 成功" />
  <img src="docs/claudecy-talking.gif" width="110" alt="claudecy 说话" />
  <img src="docs/howl-idle.gif" width="110" alt="howl 空闲" />
  <img src="docs/howl-running.gif" width="110" alt="howl 运行" />
  <img src="docs/howl-success.gif" width="110" alt="howl 成功" />
</p>

<h1 align="center">sprite-gen</h1>

<p align="center"><b>一张输入图，输出可直接用于游戏的精灵图集。</b></p>

<p align="center">

**English** · [한국어](README.ko.md) · [日本語](README.ja.md) · [简体中文](README.zh-Hans.md) · [Español](README.es.md) · [Français](README.fr.md)

</p>

---

让图像模型生成“sprite sheet”时，你通常会得到的结果是：角色面部每帧都在变化、背景无法抠图、姿势彼此重叠且脱离网格，以及游戏引擎实际上无法消费的 PNG。可爱但没用的演示资源。

`sprite-gen` 是一个 Codex/Claude skill，用于弥补这个差距。给它一张**基础图像**和一组动作清单——它会按行驱动生成、锁定角色身份、把色键背景转换为真实 Alpha、提取每个姿势为干净的透明帧，并烘焙出运行时可用的图集，附带可机读的 `manifest.json.frame_layout`。上面的每一张精灵都是这样生成的。

而对于生成最后那 10% 经常出错的部分，还有一个**精选画布（curation webview）**：并排对比帧、剔除错误帧、非破坏性地微调旋转/缩放/位置、实时查看动画循环，然后再烘焙。流水线完成繁琐劳动，最终品质由你把控。

```text
sprite-request.json → layout guides + prompts → image-gen state rows
→ chroma alpha → connected components → transparent frames
→ sprite-sheet-alpha.png + manifest.json.frame_layout
```

## 你会得到什么

- **透明精灵图集**（`sprite-sheet-alpha.png`）——真实 alpha，无残留色键边缘，并已在白色背景下校验。
- **运行时清单**（`manifest.json.frame_layout`）——绝对坐标帧矩形、按状态的 fps 与循环标志。你的引擎按矩形采样，不再猜网格。
- **可监看 QA**——每个状态的 GIF 与 contact sheet，可先判断运动质量再上线。
- **诚实的标签**——短而可读的动作（idle, jump, attack, wave）是稳定流程；循环位移类动作（walk/run）除非运动 QA 通过，否则标记为实验性。不做无声过度承诺。

## Curation webview

生成能做到 90%。webview 是人类将其推向“可上线”的地方——独立运行，无需 Studio 或框架依赖，可在任意安装该 skill 的环境运行（Claude Code Desktop、Codex app、普通终端）。

![curation webview — characters](docs/demo-character.gif)

- **按状态并排对比帧**，并可**选择/拒绝**单独的帧。
- **每帧非破坏性变换**：拖拽=移动，滚轮=缩放，上手柄=旋转，左下手柄=切变。编辑会写入 `curation.json` 副本文件——源 PNG 永远不会被重写，compose 阶段会确定性地烘焙结果。预览与烘焙共享同一个仿射矩阵，所以你对齐的就是最终会得到的。
- **实时预览**按状态 fps 播放已选帧。
- 不仅用于精灵：也可用 `unpack_atlas_run.py --pngs-dir` 指向任意候选图像文件夹（图标、徽标、生成草稿），当作通用的优选比对视图。

### 等距地面网格

对于等距集合，webview 会叠加地面网格（来自 `meta.json` 的 tile/anchor），你可以用切变手柄把家具贴合到菱形轴线上。

![curation webview — isometric furniture](docs/demo-furniture.gif)

<img src="docs/curator-iso.png" width="520" alt="等距地面网格叠加" />

### 语言

webview 默认提供英文和韩文。启动时通过 `--lang en|ko` 指定，或使用应用内切换：

```bash
python3 scripts/serve_curation.py --run-dir <run-dir> --lang en   # 或 ko
```

## 快速上手

```bash
# 0. 在全新虚拟环境中安装依赖（Pillow）
python3 -m venv .venv && source .venv/bin/activate
pip install -e .

# 1. 从一张基础图像准备一次运行
python3 scripts/prepare_sprite_run.py --out-dir <run-dir> --character-id <id> --base-image base.png

# 2. 用 image-gen 按状态生成每行一张图，保存为 raw/<state>.png
# 3. 提取帧
python3 scripts/extract_sprite_row_frames.py --run-dir <run-dir>

# 4. （可选）在 webview 中精修帧
python3 scripts/serve_curation.py --run-dir <run-dir>

# 5. 烘焙运行时图集
python3 scripts/compose_sprite_atlas.py --run-dir <run-dir>
```

### 编辑已完成的图集

当只保留合并后的图集时，重建一个可供 curator 使用的运行目录，然后进行挑选与导出：

```bash
# 重建帧：显式使用 --grid、--manifest 矩形，或 alpha 自动检测（默认）
python3 scripts/unpack_atlas_run.py --atlas sheet.png            # 自动检测
python3 scripts/unpack_atlas_run.py --manifest manifest.json     # 精确矩形
python3 scripts/unpack_atlas_run.py --pngs-dir furniture/        # 导入散落 PNG 集

# 完成筛选后，将修正结果烘焙回命名 PNG
python3 scripts/export_curated_pngs.py --run-dir <run-dir>
```

输出默认到输入旁边可定位到的 `<source>-curator` 文件夹。

完整的 agent-facing 流程与契约定义见 [`SKILL.md`](SKILL.md)。

## 安装

在 Codex skill 安装器工作流中，将本仓库作为根 skill 安装：

```bash
python3 ~/.codex/skills/.system/skill-installer/scripts/install-skill-from-github.py \
  --repo aldegad/sprite-gen --path .
```

## 归属

组件行（component-row）工作流受 Apache-2.0 许可的 `hatch-pet` skill 启发，但面向的是通用游戏精灵图集，不包含任何宠物包或宠物视觉素材。

## 许可

Apache-2.0