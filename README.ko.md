<p align="center">
  <img src="docs/claudecy-idle.gif" width="110" alt="claudecy 대기" />
  <img src="docs/claudecy-running.gif" width="110" alt="claudecy 달리기" />
  <img src="docs/claudecy-success.gif" width="110" alt="claudecy 성공" />
  <img src="docs/claudecy-talking.gif" width="110" alt="claudecy 대화" />
  <img src="docs/howl-idle.gif" width="110" alt="howl 대기" />
  <img src="docs/howl-running.gif" width="110" alt="howl 달리기" />
  <img src="docs/howl-success.gif" width="110" alt="howl 성공" />
</p>

<h1 align="center">sprite-gen</h1>

<p align="center"><b>한 장의 드로잉을 넣으면, 게임 사용 가능한 스프라이트 아틀라스가 완성됩니다.</b></p>

<p align="center">

**영어** · [한국어](README.ko.md) · [日本語](README.ja.md) · [简体中文](README.zh-Hans.md) · [Español](README.es.md) · [Français](README.fr.md)

</p>

---

이미지 모델에 “sprite sheet”를 요청하면 대개 다음을 받게 됩니다: 프레임마다 얼굴이 바뀌는 캐릭터, 키잉되지 않는 배경, 겹치거나 격자에서 벗어나는 포즈, 그리고 게임 엔진이 실제로 사용할 수 없는 PNG. 귀엽게 보이는 데모는 쓰레기 자산입니다.

`sprite-gen`은 그 간극을 메우는 Codex/Claude 스킬입니다. **기본 이미지 한 장**과 액션 목록을 주면, 행 단위로 생성 상태를 관리하고 캐릭터의 정체성을 고정한 뒤 크로마키 배경을 실제 알파로 제거하고, 각 포즈를 깨끗한 투명 프레임으로 추출해 런타임용 아틀라스와 **기계가 읽을 수 있는 `manifest.json.frame_layout`**을 만듭니다. 위의 모든 스프라이트가 바로 이 방식으로 만들어졌습니다.

그리고 생성이 90% 못 맞추는 마지막 10%를 위해 **큐레이션 웹뷰**가 있습니다: 프레임을 나란히 비교하고, 깨진 프레임을 거부하고, 회전/크기/위치를 비파괴적으로 미세 조정하고, 루프를 실시간으로 확인한 뒤 구워냅니다. 파이프라인이 반복 작업은 대신하고, 최종 감각은 당신이 유지합니다.

```text
sprite-request.json → layout guides + prompts → image-gen state rows
→ chroma alpha → connected components → transparent frames
→ sprite-sheet-alpha.png + manifest.json.frame_layout
```

## 실제로 얻는 것

- **투명 스프라이트 아틀라스**(`sprite-sheet-alpha.png`) — 실제 알파 채널을 사용하며 잔여 크로마 테두리 없음, 흰 배경에서 검증 완료.
- **런타임 매니페스트**(`manifest.json.frame_layout`) — 상태별 절대 좌표 사각형, fps, 루프 플래그 제공. 엔진은 사각형을 샘플링할 뿐 격자 구조를 추정하지 않습니다.
- **확인 가능한 QA** — 상태별 GIF와 contact sheet를 제공해, 실제로 무엇이 움직이는지 보고 최종 판정을 진행합니다.
- **정직한 라벨** — 짧고 읽기 쉬운 액션(idle, jump, attack, wave)이 기본 경로이고, 순환 이동(walk/run)은 실제 모션 QA를 통과해야만 실험 상태에서 벗어납니다. 과대 포장을 하지 않습니다.

## 큐레이션 웹뷰

생성은 90%만 끝냅니다. 웹뷰는 사람이 이를 **출시 가능한 상태**로 만드는 곳입니다. 스탠드얼론이며 Studio/프레임워크 의존성이 없고, 스킬이 설치된 어디서든 실행됩니다(Claude Code Desktop, Codex 앱, 일반 터미널).

![큐레이션 웹뷰 — 캐릭터](docs/demo-character.gif)

- 상태별로 프레임을 나란히 **비교**하고, 개별 프레임을 **선택/거부**할 수 있습니다.
- 프레임별 **비파괴 변형**: 드래그 = 이동, 휠 = 크기 조절, 상단 핸들 = 회전, 좌측 하단 핸들 = 전단 변형. 편집은 `curation.json` 사이드카에만 저장되며, 원본 PNG는 절대 덮어쓰지 않습니다. compose 단계에서 결과를 결정적으로 구워냅니다. 미리보기와 bake는 같은 어파인 행렬을 사용하므로, 맞춘 결과가 그대로 반영됩니다.
- **실시간 미리보기**는 선택된 프레임을 해당 상태의 fps로 재생합니다.
- 스프라이트만이 아니라, 이미지 후보가 들어 있는 임의 폴더(아이콘, 로고, 생성 초안 등)를 `unpack_atlas_run.py --pngs-dir`로 지정해 일반적인 winner-picks 뷰어로도 사용할 수 있습니다.

### 등각 그리드

등각(아이소메트릭) 세트의 경우 웹뷰는 바닥 그리드를 표시합니다(`meta.json`의 tile/anchor 정보를 기반). 전단 핸들을 사용해 가구를 마름모 축에 맞춰 스냅할 수 있습니다.

![큐레이션 웹뷰 — 등각 가구](docs/demo-furniture.gif)

<img src="docs/curator-iso.png" width="520" alt="등각 바닥 그리드 오버레이" />

### 언어

웹뷰는 영어와 한국어를 지원합니다. 실행 시 `--lang en|ko`를 전달하거나 인앱 토글을 사용하세요.

```bash
python3 scripts/serve_curation.py --run-dir <run-dir> --lang en   # 또는 ko
```

## Quickstart

```bash
# 0. 새 가상환경에 Pillow 의존성 설치
python3 -m venv .venv && source .venv/bin/activate
pip install -e .

# 1. 기본 이미지로 실행 디렉토리 준비
python3 scripts/prepare_sprite_run.py --out-dir <run-dir> --character-id <id> --base-image base.png

# 2. image-gen으로 상태별 행 이미지를 1장씩 생성해 raw/<state>.png로 저장
# 3. 프레임 추출
python3 scripts/extract_sprite_row_frames.py --run-dir <run-dir>

# 4. (선택) 웹뷰에서 프레임 큐레이션
python3 scripts/serve_curation.py --run-dir <run-dir>

# 5. 런타임 아틀라스 bake
python3 scripts/compose_sprite_atlas.py --run-dir <run-dir>
```

### 완성된 시트 수정

결합된 시트만 남아 있을 때는 큐레이션용 실행 디렉토리를 다시 만들고, 큐레이션 후 내보냅니다.

```bash
# 프레임 재생성: 명시적 --grid, --manifest 사각형, 또는 알파 자동 감지(기본값)
python3 scripts/unpack_atlas_run.py --atlas sheet.png            # auto-detect
python3 scripts/unpack_atlas_run.py --manifest manifest.json     # 정확한 사각형
python3 scripts/unpack_atlas_run.py --pngs-dir furniture/        # 느슨한 PNG 집합 가져오기

# 큐레이션 후, 수정본을 이름이 있는 PNG로 bake
python3 scripts/export_curated_pngs.py --run-dir <run-dir>
```

출력은 입력 옆에서 찾기 쉬운 `<source>-curator` 폴더가 기본값입니다.

전체 에이전트 워크플로우와 계약은 [`SKILL.md`](SKILL.md)에 있습니다.

## 설치

Codex 스킬 설치 워크플로우에서 이 저장소를 루트 스킬로 설치하세요.

```bash
python3 ~/.codex/skills/.system/skill-installer/scripts/install-skill-from-github.py \
  --repo aldegad/sprite-gen --path .
```

## 출처 표시

컴포넌트-행 워크플로우는 Apache-2.0 라이선스의 `hatch-pet` 스킬에서 영감을 얻었지만, 일반 게임 스프라이트 아틀라스를 대상으로 하며 펫 패키지나 펫 시각 자산은 포함하지 않습니다.

## 라이선스

Apache-2.0