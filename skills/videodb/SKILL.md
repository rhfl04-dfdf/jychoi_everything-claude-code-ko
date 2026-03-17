---
name: videodb
description: 비디오와 오디오를 보고, 이해하고, 조작합니다. See- 로컬 파일, URL, RTSP/라이브 피드 또는 데스크톱 라이브 녹화에서 ingest; 실시간 context 및 재생 가능한 스트림 링크를 반환합니다. Understand- 프레임 추출, 시각적/의미적/시간적 index 구축, 타임스탬프와 auto-clip을 사용한 순간 검색. Act- transcode 및 정규화(codec, fps, 해상도, aspect ratio), timeline 편집(자막, 텍스트/이미지 오버레이, 브랜딩, 오디오 오버레이, 더빙, 번역), 미디어 자산 생성(이미지, 오디오, 비디오), 라이브 스트림 또는 데스크톱 캡처 이벤트에 대한 실시간 알림 생성.
origin: ECC
allowed-tools: Read Grep Glob Bash(python:*)
argument-hint: "[task description]"
---

# VideoDB Skill

**비디오, 라이브 스트림, 데스크톱 세션에 대한 Perception + memory + actions.**

## 사용 시기

### 데스크톱 Perception
- **화면, 마이크, 시스템 오디오**를 캡처하는 **데스크톱 세션** 시작/중지
- **라이브 context** 스트리밍 및 **에피소드 형식의 세션 memory** 저장
- 화면에서 말하는 내용과 일어나는 상황에 대해 **실시간 알림/trigger** 실행
- **세션 요약**, 검색 가능한 타임라인, **재생 가능한 근거 링크** 생성

### 비디오 ingest + 스트림
- **파일 또는 URL**을 ingest하고 **재생 가능한 웹 스트림 링크** 반환
- Transcode/정규화: **codec, bitrate, fps, 해상도, aspect ratio**

### Index + 검색 (타임스탬프 + 근거)
- **시각적**, **음성**, **키워드** index 구축
- **타임스탬프**와 **재생 가능한 근거**를 포함한 정확한 순간 검색 및 반환
- 검색 결과에서 **clip** 자동 생성

### Timeline 편집 + 생성
- 자막: **생성**, **번역**, **burn-in**
- 오버레이: **텍스트/이미지/브랜딩**, 모션 캡션
- 오디오: **배경 음악**, **voiceover**, **더빙**
- **timeline 작업**을 통한 프로그래밍 방식의 구성 및 export

### 라이브 스트림 (RTSP) + 모니터링
- **RTSP/라이브 피드** 연결
- **실시간 시각 및 음성 이해** 실행 및 모니터링 워크플로우를 위한 **이벤트/알림** 발생

## 작동 방식

### 일반적인 입력
- 로컬 **파일 경로**, 공개 **URL** 또는 **RTSP URL**
- 데스크톱 캡처 요청: **세션 시작 / 중지 / 요약**
- 원하는 작업: 이해를 위한 context 획득, transcode 사양, index 사양, 검색 쿼리, clip 범위, timeline 편집, 알림 규칙

### 일반적인 출력
- **스트림 URL**
- **타임스탬프**와 **근거 링크**가 포함된 검색 결과
- 생성된 자산: 자막, 오디오, 이미지, clip
- 라이브 스트림을 위한 **이벤트/알림 페이로드**
- 데스크톱 **세션 요약** 및 memory 항목

### Python 코드 실행

VideoDB 코드를 실행하기 전에 프로젝트 디렉토리로 이동하여 환경 변수를 로드하세요:

```python
from dotenv import load_dotenv
load_dotenv(".env")

import videodb
conn = videodb.connect()
```

이는 다음에서 `VIDEO_DB_API_KEY`를 읽습니다:
1. 환경 변수 (이미 내보낸 경우)
2. 현재 디렉토리의 프로젝트 `.env` 파일

키가 누락된 경우, `videodb.connect()`는 자동으로 `AuthenticationError`를 발생시킵니다.

짧은 인라인 명령으로 충분할 때는 스크립트 파일을 작성하지 마세요.

인라인 Python(`python -c "..."`)을 작성할 때는 항상 올바른 형식의 코드를 사용하세요. 세미콜론을 사용하여 문을 구분하고 가독성을 유지하세요. 3개 이상의 문이 필요한 경우 heredoc을 사용하세요:

```bash
python << 'EOF'
from dotenv import load_dotenv
load_dotenv(".env")

import videodb
conn = videodb.connect()
coll = conn.get_collection()
print(f"Videos: {len(coll.get_videos())}")
EOF
```

### 설정

사용자가 "videodb 설정" 또는 이와 유사한 요청을 할 때:

### 1. SDK 설치

```bash
pip install "videodb[capture]" python-dotenv
```

Linux에서 `videodb[capture]`가 실패하면 캡처 추가 기능 없이 설치하세요:

```bash
pip install videodb python-dotenv
```

### 2. API key 구성

사용자는 다음 중 **하나의** 방법으로 `VIDEO_DB_API_KEY`를 설정해야 합니다:

- **터미널에서 내보내기** (Claude를 시작하기 전): `export VIDEO_DB_API_KEY=your-key`
- **프로젝트 `.env` 파일**: 프로젝트의 `.env` 파일에 `VIDEO_DB_API_KEY=your-key` 저장

[console.videodb.io](https://console.videodb.io)에서 무료 API key를 받으세요 (50회 무료 업로드, 신용카드 불필요).

API key를 직접 읽거나 쓰거나 처리하지 마세요. 항상 사용자가 설정하도록 하세요.

### 빠른 참조

### 미디어 업로드

```python
# URL
video = coll.upload(url="https://example.com/video.mp4")

# YouTube
video = coll.upload(url="https://www.youtube.com/watch?v=VIDEO_ID")

# Local file
video = coll.upload(file_path="/path/to/video.mp4")
```

### 스크립트 + 자막

```python
# force=True skips the error if the video is already indexed
video.index_spoken_words(force=True)
text = video.get_transcript_text()
stream_url = video.add_subtitle()
```

### 비디오 내부 검색

```python
from videodb.exceptions import InvalidRequestError

video.index_spoken_words(force=True)

# search() raises InvalidRequestError when no results are found.
# Always wrap in try/except and treat "No results found" as empty.
try:
    results = video.search("product demo")
    shots = results.get_shots()
    stream_url = results.compile()
except InvalidRequestError as e:
    if "No results found" in str(e):
        shots = []
    else:
        raise
```

### 장면 검색

```python
import re
from videodb import SearchType, IndexType, SceneExtractionType
from videodb.exceptions import InvalidRequestError

# index_scenes() has no force parameter — it raises an error if a scene
# index already exists. Extract the existing index ID from the error.
try:
    scene_index_id = video.index_scenes(
        extraction_type=SceneExtractionType.shot_based,
        prompt="Describe the visual content in this scene.",
    )
except Exception as e:
    match = re.search(r"id\s+([a-f0-9]+)", str(e))
    if match:
        scene_index_id = match.group(1)
    else:
        raise

# Use score_threshold to filter low-relevance noise (recommended: 0.3+)
try:
    results = video.search(
        query="person writing on a whiteboard",
        search_type=SearchType.semantic,
        index_type=IndexType.scene,
        scene_index_id=scene_index_id,
        score_threshold=0.3,
    )
    shots = results.get_shots()
    stream_url = results.compile()
except InvalidRequestError as e:
    if "No results found" in str(e):
        shots = []
    else:
        raise
```

### Timeline 편집

**중요:** timeline을 작성하기 전에 항상 타임스탬프를 검증하세요:
- `start`는 >= 0이어야 합니다 (음수 값은 암묵적으로 허용되지만 손상된 출력을 생성합니다)
- `start`는 `end`보다 작아야 합니다
- `end`는 `video.length`보다 작거나 같아야 합니다

```python
from videodb.timeline import Timeline
from videodb.asset import VideoAsset, TextAsset, TextStyle

timeline = Timeline(conn)
timeline.add_inline(VideoAsset(asset_id=video.id, start=10, end=30))
timeline.add_overlay(0, TextAsset(text="The End", duration=3, style=TextStyle(fontsize=36)))
stream_url = timeline.generate_stream()
```

### 비디오 Transcode (해상도 / 품질 변경)

```python
from videodb import TranscodeMode, VideoConfig, AudioConfig

# Change resolution, quality, or aspect ratio server-side
job_id = conn.transcode(
    source="https://example.com/video.mp4",
    callback_url="https://example.com/webhook",
    mode=TranscodeMode.economy,
    video_config=VideoConfig(resolution=720, quality=23, aspect_ratio="16:9"),
    audio_config=AudioConfig(mute=False),
)
```

### Aspect ratio Reframe (소셜 플랫폼용)

**경고:** `reframe()`은 느린 서버 측 작업입니다. 긴 비디오의 경우 몇 분이 소요될 수 있으며 타임아웃이 발생할 수 있습니다. Best practice:
- 가능하면 항상 `start`/`end`를 사용하여 짧은 구간으로 제한하세요
- 전체 길이 비디오의 경우 비동기 처리를 위해 `callback_url`을 사용하세요
- 먼저 `Timeline`에서 비디오를 자른 다음, 더 짧아진 결과를 reframe하세요

```python
from videodb import ReframeMode

# Always prefer reframing a short segment:
reframed = video.reframe(start=0, end=60, target="vertical", mode=ReframeMode.smart)

# Async reframe for full-length videos (returns None, result via webhook):
video.reframe(target="vertical", callback_url="https://example.com/webhook")

# Presets: "vertical" (9:16), "square" (1:1), "landscape" (16:9)
reframed = video.reframe(start=0, end=60, target="square")

# Custom dimensions
reframed = video.reframe(start=0, end=60, target={"width": 1280, "height": 720})
```

### Generative 미디어

```python
image = coll.generate_image(
    prompt="a sunset over mountains",
    aspect_ratio="16:9",
)
```

## 에러 핸들링

```python
from videodb.exceptions import AuthenticationError, InvalidRequestError

try:
    conn = videodb.connect()
except AuthenticationError:
    print("Check your VIDEO_DB_API_KEY")

try:
    video = coll.upload(url="https://example.com/video.mp4")
except InvalidRequestError as e:
    print(f"Upload failed: {e}")
```

### 일반적인 문제

| 시나리오 | 에러 메시지 | 해결 방법 |
|----------|--------------|----------|
| 이미 인덱싱된 비디오 인덱싱 | `Spoken word index for video already exists` | 이미 인덱싱된 경우 건너뛰려면 `video.index_spoken_words(force=True)` 사용 |
| 장면 index가 이미 존재함 | `Scene index with id XXXX already exists` | `re.search(r"id\s+([a-f0-9]+)", str(e))`를 사용하여 에러에서 기존 `scene_index_id` 추출 |
| 검색 결과가 없음 | `InvalidRequestError: No results found` | 예외를 포착하고 빈 결과로 처리 (`shots = []`) |
| Reframe 타임아웃 | 긴 비디오에서 무기한 대기 | `start`/`end`를 사용하여 구간을 제한하거나 비동기 처리를 위해 `callback_url` 전달 |
| Timeline의 음수 타임스탬프 | 암묵적으로 손상된 스트림 생성 | `VideoAsset`을 생성하기 전에 항상 `start >= 0` 검증 |
| `generate_video()` / `create_collection()` 실패 | `Operation not allowed` 또는 `maximum limit` | 플랜에 제한된 기능 — 사용자에게 플랜 한도에 대해 안내 |

## 예시

### 대표적인 프롬프트
- "데스크톱 캡처를 시작하고 비밀번호 필드가 나타나면 알림을 보내줘."
- "내 세션을 녹화하고 세션이 종료되면 실행 가능한 요약을 만들어줘."
- "이 파일을 ingest하고 재생 가능한 스트림 링크를 반환해줘."
- "이 폴더를 인덱싱하고 사람이 있는 모든 장면을 찾아 타임스탬프를 반환해줘."
- "자막을 생성해서 입히고 잔잔한 배경 음악을 추가해줘."
- "이 RTSP URL에 연결하고 사람이 구역에 들어오면 알림을 보내줘."

### 화면 녹화 (데스크톱 캡처)

녹화 세션 중에 WebSocket 이벤트를 캡처하려면 `ws_listener.py`를 사용하세요. 데스크톱 캡처는 **macOS**만 지원합니다.

#### 빠른 시작

1. **상태 디렉토리 선택**: `STATE_DIR="${VIDEODB_EVENTS_DIR:-$HOME/.local/state/videodb}"`
2. **리스너 시작**: `VIDEODB_EVENTS_DIR="$STATE_DIR" python scripts/ws_listener.py --clear "$STATE_DIR" &`
3. **WebSocket ID 확인**: `cat "$STATE_DIR/videodb_ws_id"`
4. **캡처 코드 실행** (전체 워크플로우는 reference/capture.md 참조)
5. **이벤트 기록 위치**: `$STATE_DIR/videodb_events.jsonl`

새로운 캡처 실행을 시작할 때마다 `--clear`를 사용하여 이전 세션의 스크립트나 시각적 이벤트가 새 세션으로 누출되지 않도록 하세요.

#### 이벤트 쿼리

```python
import json
import os
import time
from pathlib import Path

events_dir = Path(os.environ.get("VIDEODB_EVENTS_DIR", Path.home() / ".local" / "state" / "videodb"))
events_file = events_dir / "videodb_events.jsonl"
events = []

if events_file.exists():
    with events_file.open(encoding="utf-8") as handle:
        for line in handle:
            try:
                events.append(json.loads(line))
            except json.JSONDecodeError:
                continue

transcripts = [e["data"]["text"] for e in events if e.get("channel") == "transcript"]
cutoff = time.time() - 300
recent_visual = [
    e for e in events
    if e.get("channel") == "visual_index" and e["unix_ts"] > cutoff
]
```

## 추가 문서

참조 문서는 이 SKILL.md 파일과 인접한 `reference/` 디렉토리에 있습니다. 필요한 경우 Glob 도구를 사용하여 위치를 찾으세요.

- [reference/api-reference.md](reference/api-reference.md) - 전체 VideoDB Python SDK API 참조
- [reference/search.md](reference/search.md) - 비디오 검색 심층 가이드 (음성 및 장면 기반)
- [reference/editor.md](reference/editor.md) - Timeline 편집, 자산 및 구성
- [reference/streaming.md](reference/streaming.md) - HLS 스트리밍 및 즉시 재생
- [reference/generative.md](reference/generative.md) - AI 기반 미디어 생성 (이미지, 비디오, 오디오)
- [reference/rtstream.md](reference/rtstream.md) - 라이브 스트림 ingest 워크플로우 (RTSP/RTMP)
- [reference/rtstream-reference.md](reference/rtstream-reference.md) - RTStream SDK 메서드 및 AI 파이프라인
- [reference/capture.md](reference/capture.md) - 데스크톱 캡처 워크플로우
- [reference/capture-reference.md](reference/capture-reference.md) - 캡처 SDK 및 WebSocket 이벤트
- [reference/use-cases.md](reference/use-cases.md) - 일반적인 비디오 처리 패턴 및 예시

VideoDB가 해당 작업을 지원하는 경우 **ffmpeg, moviepy 또는 로컬 인코딩 도구를 사용하지 마세요**. 다음은 모두 VideoDB에 의해 서버 측에서 처리됩니다 — 자르기, clip 결합, 오디오 또는 음악 오버레이, 자막 추가, 텍스트/이미지 오버레이, transcode, 해상도 변경, aspect ratio 변환, 플랫폼 요구 사항에 따른 크기 조정, 스크립트 추출 및 미디어 생성. reference/editor.md의 Limitations 섹션에 나열된 작업(트랜지션, 속도 변경, crop/zoom, 컬러 그레이딩, 볼륨 믹싱)에 대해서만 로컬 도구를 사용하세요.

### 문제별 해결 방법

| 문제 | VideoDB 해결 방법 |
|---------|-----------------|
| 플랫폼에서 비디오 aspect ratio 또는 해상도를 거부함 | `video.reframe()` 또는 `VideoConfig`를 사용한 `conn.transcode()` |
| Twitter/Instagram/TikTok용 비디오 크기 조정 필요 | `video.reframe(target="vertical")` 또는 `target="square"` |
| 해상도 변경 필요 (예: 1080p → 720p) | `VideoConfig(resolution=720)`를 사용한 `conn.transcode()` |
| 비디오에 오디오/음악 오버레이 필요 | `Timeline`의 `AudioAsset` |
| 자막 추가 필요 | `video.add_subtitle()` 또는 `CaptionAsset` |
| clip 결합/자르기 필요 | `Timeline`의 `VideoAsset` |
| voiceover, 음악 또는 SFX 생성 필요 | `coll.generate_voice()`, `generate_music()`, `generate_sound_effect()` |

## 출처

이 skill에 대한 참조 자료는 로컬의 `skills/videodb/reference/` 아래에 저장되어 있습니다. 런타임에 외부 저장소 링크를 따르는 대신 위의 로컬 복사본을 사용하세요.

**Maintained By:** [VideoDB](https://www.videodb.io/)
