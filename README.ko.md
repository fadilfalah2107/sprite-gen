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

<p align="center"><b>한 장의 그림을 넣으면, 게임 준비가 된 스프라이트 아틀라스가 나옵니다.</b></p>

<p align="center">

**영어** · [한국어](README.ko.md) · [日本語](README.ja.md) · [简体中文](README.zh-Hans.md) · [Español](README.es.md) · [Français](README.fr.md)

</p>

---

이미지 모델에 “스프라이트 시트”를 요청하면 보통 이런 결과가 나옵니다. 매 프레임마다 얼굴이 달라지는 캐릭터, 키아웃되지 않는 배경, 서로 겹치고 그리드에서 벗어나는 포즈, 그리고 게임 엔진이 실제로 사용할 수 없는 PNG. 귀엽게 보이는 데모지만 쓸모없는 에셋입니다.

`sprite-gen`은 그 간극을 메우는 Codex/Claude 스킬입니다. **하나의 베이스 이미지**와 동작 목록만 주면, 행(row) 단위로 생성을 진행하고, 캐릭터의 정체성을 고정하며, 크로마 키 배경을 실제 알파로 제거하고, 각 포즈를 깨끗한 투명 프레임으로 추출한 뒤, 실행 시 사용할 수 있는 아틀라스를 `manifest.json.frame_layout`이라는 머신이 해석 가능한 형태로 만듭니다. 위에 나온 모든 스프라이트는 이렇게 만들어졌습니다.

그리고 생성이 100% 맞추지 못하는 마지막 10%를 위해 **큐레이션 웹뷰**가 있습니다. 프레임을 나란히 비교하고, 잘못된 프레임을 거부하며, 회전/크기/위치를 비파괴적으로 미세 조정하고, 루프를 실시간으로 보면서 최종 결과를 굽습니다. 파이프라인이 노동을 담당하고, 최종 감각은 사용자가 정합니다.

```text
sprite-request.json → layout guides + prompts → image-gen state rows
→ chroma alpha → connected components → transparent frames
→ sprite-sheet-alpha.png + manifest.json.frame_layout
```

<p align="center">
  <img src="docs/architecture-diagram.png" width="640" alt="sprite-gen 아키텍처 — 컴포넌트-행 파이프라인" />
</p>

> 전체 아키텍처: [`docs/architecture.md`](docs/architecture.md) · 다이어그램 원본: [`docs/architecture-diagram.html`](docs/architecture-diagram.html)

## 실제로 얻을 수 있는 것

- **투명 스프라이트 아틀라스** (`sprite-sheet-alpha.png`) — 실제 알파를 사용하며 남은 크로마 잔향이 없습니다. 흰색 배경에서 검증됨.
- **런타임 매니페스트** (`manifest.json.frame_layout`) — 절대 좌표 프레임 사각형, 상태별 fps와 루프 플래그. 엔진은 사각형을 샘플링할 뿐, 그리드 추측을 하지 않습니다.
- **바로 확인할 수 있는 QA** — 상태별 GIF와 contact sheet를 통해, 실제 배포 전 모션을 모션 기준으로 판단할 수 있습니다.
- **정직한 라벨링** — 짧고 읽기 쉬운 동작 이름(예: idle, jump, attack, wave)을 안정적인 경로로 사용하고, 주기적 움직임(walk/run) 같은 순환 동작은 모션 QA를 통과해야 실험적 플래그가 해제됩니다. 과장된 약속은 없습니다.

## 큐레이션 웹뷰

생성은 90%를 제공합니다. 웹뷰는 사람의 손이 “배포 가능”으로 끌어올리는 구간입니다. 스탠드얼론이며 Studio나 프레임워크 의존성이 없고, 스킬이 설치된 어디서든 실행됩니다(Claude Code Desktop, Codex 앱, 일반 터미널).

![큐레이션 웹뷰 — 캐릭터](docs/demo-character.gif)

- **상태당 두 줄**: 위쪽은 **재생 시퀀스**, 아래는 **후보 풀**(예: 두 번째/세 번째 생성 시도). 프레임의 ⠿ 그립을 드래그해 순서를 바꾸거나, 후보 풀에서 컷을 끌어와 가장 좋은 프레임들로 하나의 깔끔한 루프를 다시 구성할 수 있습니다. 배치는 저장되므로 다시 열었을 때 복원됩니다.
- **프레임 단위 비파괴 변환**: 드래그 = 이동, 휠 = 스케일, 상단 핸들 = 회전, 좌하단 = 전단, 좌우 반전 출력용 수평 플립 토글이 가능합니다. 편집은 `curation.json` 사이드카에 저장되며 원본 PNG는 다시 쓰지 않고, compose 단계에서 결정적으로 굽습니다. 미리보기와 굽기에서 같은 아핀 변환 행렬을 공유하므로, 맞춘 대로 최종 반영됩니다.
- **라이브 미리보기**는 해당 상태의 fps로 시퀀스를 재생하며, 재생/일시정지, 프레임 단위 스텝, 0.25×–4× 속도 제어를 제공합니다.
- 스프라이트 전용이 아닙니다. `unpack_atlas_run.py --pngs-dir`로 어떤 후보 이미지 폴더(아이콘, 로고, 생성 초안)에도 적용해 범용 승자 선별 뷰로 사용할 수 있습니다.

### 아이소메트릭 지면 그리드

아이소메트릭 세트의 경우 웹뷰는 바닥 그리드를 오버레이합니다(`meta.json`의 tile/anchor 기반). 전단 핸들을 사용해 가구를 다이아몬드 축에 맞춰 스냅할 수 있습니다.

![큐레이션 웹뷰 — 아이소메트릭 가구](docs/demo-furniture.gif)

<img src="docs/curator-iso.png" width="520" alt="아이소메트릭 지면 그리드 오버레이" />

### 언어

웹뷰는 영어와 한국어를 지원합니다. 실행 시 `--lang en|ko`를 전달하거나, 인앱 토글을 사용하세요.

```bash
python3 scripts/serve_curation.py --run-dir <run-dir> --lang en   # or ko
```

## Python 지원

`sprite-gen`은 CPython 3.10+를 지원합니다. CI는 GitHub-hosted runner에서 최소 지원 버전(3.10)과 최상위 지원 버전(3.14)을 실행합니다.

빠른 시작을 위해서는 `venv`/`ensurepip`이 동작하는 Python 설치가 필요합니다. 로컬 배포에서 패키지 설치 전에 `python3 -m venv`가 실패하면, 지원되는 어떤 CPython 버전으로든 표준 빌드를 사용해 동일한 명령을 다시 실행하세요.

## 빠른 시작

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

### 완성된 시트 편집

결합된 시트만 남아 있을 때는, 큐레이터용 런 디렉터리를 다시 구성한 뒤 큐레이션하고 내보내세요.

```bash
# rebuild frames: explicit --grid, --manifest rectangles, or alpha auto-detect (default)
python3 scripts/unpack_atlas_run.py --atlas sheet.png            # auto-detect
python3 scripts/unpack_atlas_run.py --manifest manifest.json     # exact rectangles
python3 scripts/unpack_atlas_run.py --pngs-dir furniture/        # import a loose PNG set

# after curating, bake corrections back to named PNGs
python3 scripts/export_curated_pngs.py --run-dir <run-dir>
```

출력은 입력 파일 옆에서 찾기 쉬운 `<source>-curator` 폴더가 기본값으로 생성됩니다.

완전한 에이전트 동작 흐름과 계약은 [`SKILL.md`](SKILL.md)에 있습니다.

## 설치

Codex 스킬 설치 워크플로에서 이 저장소를 루트 스킬로 설치하세요.

```bash
python3 ~/.codex/skills/.system/skill-installer/scripts/install-skill-from-github.py \
  --repo aldegad/sprite-gen --path .
```

## 출처

컴포넌트-행 워크플로우는 Apache-2.0 라이선스의 `hatch-pet` 스킬에서 영감을 받았지만, 범용 게임 스프라이트 아틀라스를 대상으로 하며 반려동물 패키지나 반려동물 비주얼 에셋은 포함하지 않습니다.

## 라이선스

Apache-2.0