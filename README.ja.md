<p align="center">
  <img src="docs/claudecy-idle.gif" width="110" alt="claudecy 待機" />
  <img src="docs/claudecy-running.gif" width="110" alt="claudecy 走行" />
  <img src="docs/claudecy-success.gif" width="110" alt="claudecy 成功" />
  <img src="docs/claudecy-talking.gif" width="110" alt="claudecy 会話" />
  <img src="docs/howl-idle.gif" width="110" alt="howl 待機" />
  <img src="docs/howl-running.gif" width="110" alt="howl 走行" />
  <img src="docs/howl-success.gif" width="110" alt="howl 成功" />
</p>

<h1 align="center">sprite-gen</h1>

<p align="center"><b>1枚の図面を入力するだけ。ゲームで使えるスプライトアトラスを出力。</b></p>

<p align="center">

**English** · [한국어](README.ko.md) · [日本語](README.ja.md) · [简体中文](README.zh-Hans.md) · [Español](README.es.md) · [Français](README.fr.md)

</p>

---

画像モデルに「スプライトシート」を依頼すると、たいてい次のようになります。フレームごとに表情が変わるキャラクター、キーイングできない背景、重なり合ってグリッドからずれたポーズ、ゲームエンジンが実際には読み込めない PNG。かわいいデモにはなるかもしれないが、実用的なアセットではありません。

`sprite-gen` はそのギャップを埋める Codex/Claude スキルです。**1枚のベース画像**とアクションのリストを渡すと、行ごとに生成を進め、キャラクターの同一性を固定し、クロマ背景を実際のアルファへ変換し、各ポーズを綺麗な透過フレームとして抽出し、実行時に使えるアトラスを **機械可読な `manifest.json.frame_layout`** とともに焼き込みます。上のすべてのスプライトはこの方法で作られています。

そして、生成が最後まで正しくやり切れない 10% については、**キュレーション用 Webview** があります。フレームを並べて比較し、崩れたフレームを除外し、回転・拡大縮小・位置を非破壊で微調整し、ループをライブで確認して焼き込み。作業はパイプラインが担い、最終的な調整感はあなたが持ちます。

```text
sprite-request.json → レイアウトガイド + プロンプト → image-gen state rows
→ クロマアルファ → 連結成分抽出 → 透過フレーム
→ sprite-sheet-alpha.png + manifest.json.frame_layout
```

## 実際に得られるもの

- **透過スプライトアトラス**（`sprite-sheet-alpha.png`）— 実際のアルファで、残存するクロマの縁がなく、白背景で検証済み。
- **実行時マニフェスト**（`manifest.json.frame_layout`）— 絶対座標の矩形、状態ごとの fps とループフラグ。エンジンは矩形を参照してサンプリングし、グリッドを推測しません。
- **確認可能な QA** — 状態ごとの GIF とコンタクトシートを用意し、リリース前にモーションをモーションとして評価できます。
- **誠実なラベル** — 短く分かりやすいアクション（idle, jump, attack, wave）は安定ルートとして扱い、周期移動系（walk/run）は実際にモーション QA を通過するまで実験扱い。過剰な期待を静かに煽らない。

## キュレーション用 Webview

生成で 90% が完成します。Webview は、そこから人間が「出荷可能」まで仕上げる場所です。スタンドアロンで、Studio やフレームワーク依存なし。スキルがインストールされた環境ならどこでも動きます（Claude Code Desktop、Codex app、通常のターミナル）。

![curation webview — characters](docs/demo-character.gif)

- 状態ごとにフレームを並べて比較し、個別フレームを **選択 / 除外** できます。
- フレームごとの非破壊変形: ドラッグで移動、ホイールで拡大縮小、上側ハンドルで回転、左下ハンドルでシア。編集は `curation.json` サイドカーに記録され、元の PNG は上書きされません。compose ステップで結果は決定的に焼き込まれます。プレビューと焼き込みは同じアフィン行列を使用するため、見た目どおりの位置合わせになります。
- **ライブプレビュー**で、選択したフレームを状態の fps で再生します。
- スプライトに限りません。`unpack_atlas_run.py --pngs-dir` を使って画像候補が入った任意のフォルダ（アイコン、ロゴ、生成下書き）を指定し、勝者選択ビューとして使えます。

### 等角（アイソメトリック）地面グリッド

等角セットの場合、Webview は床グリッド（`meta.json` の tile/anchor）を重ねて表示し、シアハンドルで家具を菱形軸にスナップできます。

![curation webview — isometric furniture](docs/demo-furniture.gif)

<img src="docs/curator-iso.png" width="520" alt="等角地面グリッドオーバーレイ" />

### 言語

Webview は英語と韓国語を搭載しています。起動時に `--lang en|ko` を指定するか、アプリ内トグルを使用します。

```bash
python3 scripts/serve_curation.py --run-dir <run-dir> --lang en   # または ko
```

## クイックスタート

```bash
# 0. 依存関係（Pillow）を新規 virtualenv にインストール
python3 -m venv .venv && source .venv/bin/activate
pip install -e .

# 1. ベース画像から実行ディレクトリを準備
python3 scripts/prepare_sprite_run.py --out-dir <run-dir> --character-id <id> --base-image base.png

# 2. image-gen で状態ごとに1行画像を生成し raw/<state>.png に保存
# 3. フレームを抽出
python3 scripts/extract_sprite_row_frames.py --run-dir <run-dir>

# 4. （任意）Webview でフレームをキュレーション
python3 scripts/serve_curation.py --run-dir <run-dir>

# 5. 実行時アトラスを焼き込み
python3 scripts/compose_sprite_atlas.py --run-dir <run-dir>
```

### 完成済みシートの編集

結合済みシートのみが残っている場合、キュレーター向けの実行ディレクトリを再構築してからキュレーションしてエクスポートします。

```bash
# フレーム再構築: --grid、--manifest 矩形、またはアルファ自動検出（デフォルト）
python3 scripts/unpack_atlas_run.py --atlas sheet.png            # 自動検出
python3 scripts/unpack_atlas_run.py --manifest manifest.json     # 矩形を完全一致指定
python3 scripts/unpack_atlas_run.py --pngs-dir furniture/        # 緩い PNG セットを読み込み

# キュレーション後、修正を名前付き PNG に焼き戻す
python3 scripts/export_curated_pngs.py --run-dir <run-dir>
```

出力は、入力ファイルと同じ場所に見つけやすい `<source>-curator` フォルダがデフォルトで作成されます。

エージェント向けの完全なワークフローと契約は [`SKILL.md`](SKILL.md) にあります。

## インストール

Codex スキルインストーラのワークフローから、このリポジトリをルートスキルとしてインストールします。

```bash
python3 ~/.codex/skills/.system/skill-installer/scripts/install-skill-from-github.py \
  --repo aldegad/sprite-gen --path .
```

## クレジット

コンポーネント行のワークフローは Apache-2.0 ライセンスの `hatch-pet` スキルにインスパイアされていますが、汎用ゲームスプライトアトラスを対象としており、ペット専用パッケージやペットのビジュアルアセットは含みません。

## ライセンス

Apache-2.0