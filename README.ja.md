<p align="center">
  <img src="docs/claudecy-idle.gif" width="110" alt="claudecy アイドル" />
  <img src="docs/claudecy-running.gif" width="110" alt="claudecy 走行" />
  <img src="docs/claudecy-success.gif" width="110" alt="claudecy 成功" />
  <img src="docs/claudecy-talking.gif" width="110" alt="claudecy 会話" />
  <img src="docs/howl-idle.gif" width="110" alt="howl アイドル" />
  <img src="docs/howl-running.gif" width="110" alt="howl 走行" />
  <img src="docs/howl-success.gif" width="110" alt="howl 成功" />
</p>

<h1 align="center">sprite-gen</h1>

<p align="center"><b>描画 1 枚を投入するだけで、ゲーム対応のスプライトアトラスを出力。</b></p>

<p align="center">

**英語** · [한국어](README.ko.md) · [日本語](README.ja.md) · [简体中文](README.zh-Hans.md) · [Español](README.es.md) · [Français](README.fr.md)

</p>

---

画像生成モデルに「スプライトシート」を頼むと、実際にはこんなものが返ってきます。  
フレームごとに表情が変わるキャラクター、抜き取りに使えない背景、重なってずれたポーズ、ゲームエンジンでそのまま読み込めない PNG。  
見た目はかわいいデモに見えるかもしれませんが、実戦で使えるアセットにはなりません。

`sprite-gen` はそのギャップを埋める Codex/Claude skill です。  
**1 つのベース画像**とアクションのリストを渡すと、行単位で生成を制御し、キャラクターのアイデンティティを固定し、チョーククロマ背景を実際のアルファに変換し、各ポーズを透明なフレームとして抽出し、実行時に使えるアトラスを **機械可読な `manifest.json.frame_layout` と共に焼き付けます**。上のすべてのスプライトはこの方法で作成されています。

そして、生成だけでは最後の 10%がうまくいかない部分のために、**キュレーション webview** を用意しています。  
フレームを並べて比較し、壊れたフレームを拒否し、回転・拡大縮小・位置を破壊的でない形で微調整し、ループをライブで確認してから焼き付けます。  
パイプラインが重労働を担当し、最終的な「味」はあなたが決めます。

```text
sprite-request.json → layout guides + prompts → image-gen state rows
→ chroma alpha → connected components → transparent frames
→ sprite-sheet-alpha.png + manifest.json.frame_layout
```

<p align="center">
  <img src="docs/architecture-diagram.png" width="640" alt="sprite-gen architecture — component-row pipeline" />
</p>

> 全体アーキテクチャ: [`docs/architecture.md`](docs/architecture.md) · 図のソース: [`docs/architecture-diagram.html`](docs/architecture-diagram.html)

## 実際に得られるもの

- **透明なスプライトアトラス** (`sprite-sheet-alpha.png`) — 本当のアルファを持ち、チョーク縁が残らず、白背景で検証済み。
- **実行時マニフェスト** (`manifest.json.frame_layout`) — 絶対座標のフレーム矩形、状態ごとの fps とループフラグを保持。エンジンは矩形を参照して読み込むため、グリッド推定は行いません。
- **確認しやすい QA** — 状態ごとの GIF とコンタクトシートを用意し、公開前にモーションとして評価できます。
- **誠実なラベル付け** — 短く読みやすいアクション名（idle, jump, attack, wave）を基本の安定ルートとし、歩行/走行（walk/run）のような循環動作は、モーション QA を実際に通過するまでは実験扱いにします。無意味な過剰約束はしません。

## Curation webview

生成で 90% が完了します。残りは webview で「公開可能」に仕上げます。  
Studio やフレームワークへの依存はありません。スキルが導入されている環境（Claude Code Desktop、Codex app、通常のターミナル）ならどこでも動きます。

![curation webview — characters](docs/demo-character.gif)

- **状態ごとに2行**: 上段は **再生順**、下段は **候補プール**（例: 2 回目/3 回目の生成結果）。フレームの ⠿ グリップをドラッグして順序を変更したり、候補から切り出して投入したりして、ベストなフレームだけで1本のクリーンなループを再構成できます。配置は保存され、再起動時にも復元されます。
- **フレームごとの非破壊変形**: ドラッグで移動、ホイールで拡縮、上側ハンドルで回転、左下ハンドルでせん断、加えて左右反転トグルで左右反転出力を生成します。編集内容は `curation.json` のサイドカーファイルに保存され、元の PNG は上書きされません。compose は決定的に結果を焼き付けます。プレビューと焼き付けで同じアフィン行列を使用するため、見た目と実装が一致します。
- **ライブプレビュー** は状態ごとの fps でシーケンスを再生し、再生/停止、1フレームずつのステップ、0.25×–4× の速度調整が可能です。
- スプライト専用ではありません: `unpack_atlas_run.py --pngs-dir` で任意の画像候補フォルダ（アイコン、ロゴ、生成下書きなど）を指定し、汎用的な「勝ち組選択ビュー」として使えます。

### Isometric ground grid

等角系（アイソメトリック）セットでは、webview が床グリッド（`meta.json` のタイル／アンカー）をオーバーレイし、せん断ハンドルで家具をダイアモンド軸にスナップできます。

![curation webview — isometric furniture](docs/demo-furniture.gif)

<img src="docs/curator-iso.png" width="520" alt="isometric ground grid overlay" />

### 言語

webview は英語と韓国語を搭載しています。起動時に `--lang en|ko` を指定するか、アプリ内の切り替えを使います。

```bash
python3 scripts/serve_curation.py --run-dir <run-dir> --lang en   # or ko
```

## Python サポート

`sprite-gen` は CPython 3.10+ をサポートしています。CI では GitHub-hosted runner 上で最小サポート版（3.10）と最新サポート版（3.14）を実行します。

クイックスタートには `venv` / `ensurepip` が機能する Python インストールが必要です。  
ローカル配布環境で `python3 -m venv` がパッケージインストール前に失敗する場合は、任意のサポート対象バージョンの標準 CPython を使って同じコマンドを再実行してください。

## Quickstart

```bash
# 0. install dependencies (Pillow) into a fresh virtualenv
python3 -m venv .venv && source .venv/bin/activate
pip install -e .

# 1. prepare a run from a base image
python3 scripts/prepare_sprite_run.py --out-dir <run-dir> --character-id <id> --base-image base.png

# 2. generate one row image per state with image-gen, save as raw/<state>.png
# 3. extract frames
python3 scripts/extract_sprite_row_frames.py --run-dir <run-dir>

# 4. (optional) curate frames in the webview
python3 scripts/serve_curation.py --run-dir <run-dir>

# 5. bake the runtime atlas
python3 scripts/compose_sprite_atlas.py --run-dir <run-dir>
```

### 完成したシートの編集

合成後のシートだけが残っている場合は、キュレーション用 run dir を再構築してから調整・書き出しを行います。

```bash
# rebuild frames: explicit --grid, --manifest rectangles, or alpha auto-detect (default)
python3 scripts/unpack_atlas_run.py --atlas sheet.png            # auto-detect
python3 scripts/unpack_atlas_run.py --manifest manifest.json     # exact rectangles
python3 scripts/unpack_atlas_run.py --pngs-dir furniture/        # import a loose PNG set

# after curating, bake corrections back to named PNGs
python3 scripts/export_curated_pngs.py --run-dir <run-dir>
```

出力は既定で、入力ファイルの隣に `'<source>-curator'` という識別しやすいフォルダとして作成されます。

エージェント向けの全体フローと契約は [`SKILL.md`](SKILL.md) にあります。

## インストール

Codex の skill インストーラーワークフローから、このリポジトリをルートスキルとしてインストールします。

```bash
python3 ~/.codex/skills/.system/skill-installer/scripts/install-skill-from-github.py \
  --repo aldegad/sprite-gen --path .
```

## 帰属

コンポーネント行ベースのワークフローは Apache-2.0 ライセンスの `hatch-pet` skill に着想を得ていますが、対象は汎用ゲームスプライトアトラスであり、ペット用のパッケージやペットのビジュアルアセットは含んでいません。

## ライセンス

Apache-2.0