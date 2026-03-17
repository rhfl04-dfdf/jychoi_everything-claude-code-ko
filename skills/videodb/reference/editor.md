# 타임라인 편집 가이드

VideoDB는 여러 에셋을 합성하고, 텍스트 및 이미지 오버레이를 추가하고, 오디오 트랙을 믹싱하고, 클립을 트림하는 비파괴적 타임라인 에디터를 제공합니다. 모두 서버 측에서 재인코딩이나 로컬 도구 없이 처리됩니다. 트림, 클립 결합, 비디오에 오디오/음악 오버레이 추가, 자막 추가, 텍스트나 이미지 레이어링에 사용하세요.

## 사전 조건

동영상, 오디오, 이미지는 타임라인 에셋으로 사용하기 전에 컬렉션에 **업로드**되어야 합니다. 자막 오버레이의 경우 동영상이 **음성 단어 인덱싱**도 완료되어야 합니다.

## 핵심 개념

### Timeline

`Timeline`은 가상의 합성 레이어입니다. 에셋은 **인라인** (메인 트랙에 순차적으로) 또는 **오버레이** (특정 타임스탬프에 레이어링)로 배치됩니다. 원본 미디어는 수정되지 않으며, 최종 스트림은 요청 시 컴파일됩니다.

```python
from videodb.timeline import Timeline

timeline = Timeline(conn)
```

### Asset

타임라인의 모든 요소는 **에셋**입니다. VideoDB는 다섯 가지 에셋 타입을 제공합니다:

| 에셋 | 임포트 | 주요 용도 |
|-------|--------|-------------|
| `VideoAsset` | `from videodb.asset import VideoAsset` | 동영상 클립 (트림, 시퀀싱) |
| `AudioAsset` | `from videodb.asset import AudioAsset` | 음악, 효과음, 내레이션 |
| `ImageAsset` | `from videodb.asset import ImageAsset` | 로고, 썸네일, 오버레이 |
| `TextAsset` | `from videodb.asset import TextAsset, TextStyle` | 제목, 캡션, 하단 자막 |
| `CaptionAsset` | `from videodb.editor import CaptionAsset` | 자동 렌더링 자막 (에디터 API) |

## 타임라인 구성

### 동영상 클립 인라인 추가

인라인 에셋은 메인 동영상 트랙에서 순차적으로 재생됩니다. `add_inline` 메서드는 `VideoAsset`만 허용합니다:

```python
from videodb.asset import VideoAsset

video_a = coll.get_video(video_id_a)
video_b = coll.get_video(video_id_b)

timeline = Timeline(conn)
timeline.add_inline(VideoAsset(asset_id=video_a.id))
timeline.add_inline(VideoAsset(asset_id=video_b.id))

stream_url = timeline.generate_stream()
```

### 트림 / 서브클립

`VideoAsset`의 `start`와 `end`를 사용하여 일부만 추출합니다:

```python
# Take only seconds 10–30 from the source video
clip = VideoAsset(asset_id=video.id, start=10, end=30)
timeline.add_inline(clip)
```

### VideoAsset 파라미터

| 파라미터 | 타입 | 기본값 | 설명 |
|-----------|------|---------|-------------|
| `asset_id` | `str` | 필수 | 동영상 미디어 ID |
| `start` | `float` | `0` | 트림 시작 (초) |
| `end` | `float\|None` | `None` | 트림 종료 (`None` = 전체) |

> **경고:** SDK는 음수 타임스탬프를 검증하지 않습니다. `start=-5`를 전달해도 조용히 허용되지만 깨진 출력이나 예기치 않은 결과가 발생합니다. `VideoAsset`을 생성하기 전에 항상 `start >= 0`, `start < end`, `end <= video.length`를 확인하세요.

## 텍스트 오버레이

타임라인의 어느 지점에서든 제목, 하단 자막, 캡션을 추가합니다:

```python
from videodb.asset import TextAsset, TextStyle

title = TextAsset(
    text="Welcome to the Demo",
    duration=5,
    style=TextStyle(
        fontsize=36,
        fontcolor="white",
        boxcolor="black",
        alpha=0.8,
        font="Sans",
    ),
)

# Overlay the title at the very start (t=0)
timeline.add_overlay(0, title)
```

### TextStyle 파라미터

| 파라미터 | 타입 | 기본값 | 설명 |
|-----------|------|---------|-------------|
| `fontsize` | `int` | `24` | 폰트 크기 (픽셀) |
| `fontcolor` | `str` | `"black"` | CSS 색상 이름 또는 헥스 |
| `fontcolor_expr` | `str` | `""` | 동적 폰트 색상 표현식 |
| `alpha` | `float` | `1.0` | 텍스트 불투명도 (0.0–1.0) |
| `font` | `str` | `"Sans"` | 폰트 패밀리 |
| `box` | `bool` | `True` | 배경 박스 활성화 |
| `boxcolor` | `str` | `"white"` | 배경 박스 색상 |
| `boxborderw` | `str` | `"10"` | 박스 테두리 너비 |
| `boxw` | `int` | `0` | 박스 너비 재정의 |
| `boxh` | `int` | `0` | 박스 높이 재정의 |
| `line_spacing` | `int` | `0` | 줄 간격 |
| `text_align` | `str` | `"T"` | 박스 내 텍스트 정렬 |
| `y_align` | `str` | `"text"` | 수직 정렬 기준 |
| `borderw` | `int` | `0` | 텍스트 테두리 너비 |
| `bordercolor` | `str` | `"black"` | 텍스트 테두리 색상 |
| `expansion` | `str` | `"normal"` | 텍스트 확장 모드 |
| `basetime` | `int` | `0` | 시간 기반 표현식의 기준 시간 |
| `fix_bounds` | `bool` | `False` | 텍스트 경계 고정 |
| `text_shaping` | `bool` | `True` | 텍스트 쉐이핑 활성화 |
| `shadowcolor` | `str` | `"black"` | 그림자 색상 |
| `shadowx` | `int` | `0` | 그림자 X 오프셋 |
| `shadowy` | `int` | `0` | 그림자 Y 오프셋 |
| `tabsize` | `int` | `4` | 탭 크기 (스페이스) |
| `x` | `str` | `"(main_w-text_w)/2"` | 수평 위치 표현식 |
| `y` | `str` | `"(main_h-text_h)/2"` | 수직 위치 표현식 |

## 오디오 오버레이

동영상 트랙 위에 배경 음악, 효과음, 보이스오버를 레이어링합니다:

```python
from videodb.asset import AudioAsset

music = coll.get_audio(music_id)

audio_layer = AudioAsset(
    asset_id=music.id,
    disable_other_tracks=False,
    fade_in_duration=2,
    fade_out_duration=2,
)

# Start the music at t=0, overlaid on the video track
timeline.add_overlay(0, audio_layer)
```

### AudioAsset 파라미터

| 파라미터 | 타입 | 기본값 | 설명 |
|-----------|------|---------|-------------|
| `asset_id` | `str` | 필수 | 오디오 미디어 ID |
| `start` | `float` | `0` | 트림 시작 (초) |
| `end` | `float\|None` | `None` | 트림 종료 (`None` = 전체) |
| `disable_other_tracks` | `bool` | `True` | True일 때 다른 오디오 트랙 음소거 |
| `fade_in_duration` | `float` | `0` | 페이드인 초 (최대 5) |
| `fade_out_duration` | `float` | `0` | 페이드아웃 초 (최대 5) |

## 이미지 오버레이

로고, 워터마크, 생성된 이미지를 오버레이로 추가합니다:

```python
from videodb.asset import ImageAsset

logo = coll.get_image(logo_id)

logo_overlay = ImageAsset(
    asset_id=logo.id,
    duration=10,
    width=120,
    height=60,
    x=20,
    y=20,
)

timeline.add_overlay(0, logo_overlay)
```

### ImageAsset 파라미터

| 파라미터 | 타입 | 기본값 | 설명 |
|-----------|------|---------|-------------|
| `asset_id` | `str` | 필수 | 이미지 미디어 ID |
| `width` | `int\|str` | `100` | 표시 너비 |
| `height` | `int\|str` | `100` | 표시 높이 |
| `x` | `int` | `80` | 수평 위치 (왼쪽에서 px) |
| `y` | `int` | `20` | 수직 위치 (위에서 px) |
| `duration` | `float\|None` | `None` | 표시 시간 (초) |

## 자막 오버레이

동영상에 자막을 추가하는 두 가지 방법이 있습니다.

### 방법 1: 자막 워크플로 (가장 간단)

`video.add_subtitle()`을 사용하여 동영상 스트림에 자막을 직접 적용합니다. 내부적으로 `videodb.timeline.Timeline`을 사용합니다:

```python
from videodb import SubtitleStyle

# Video must have spoken words indexed first (force=True skips if already done)
video.index_spoken_words(force=True)

# Add subtitles with default styling
stream_url = video.add_subtitle()

# Or customise the subtitle style
stream_url = video.add_subtitle(style=SubtitleStyle(
    font_name="Arial",
    font_size=22,
    primary_colour="&H00FFFFFF",
    bold=True,
))
```

### 방법 2: 에디터 API (고급)

에디터 API (`videodb.editor`)는 `CaptionAsset`, `Clip`, `Track`, 자체 `Timeline`을 갖춘 트랙 기반 합성 시스템을 제공합니다. 위에서 사용한 `videodb.timeline.Timeline`과는 별개의 API입니다.

```python
from videodb.editor import (
    CaptionAsset,
    Clip,
    Track,
    Timeline as EditorTimeline,
    FontStyling,
    BorderAndShadow,
    Positioning,
    CaptionAnimation,
)

# Video must have spoken words indexed first (force=True skips if already done)
video.index_spoken_words(force=True)

# Create a caption asset
caption = CaptionAsset(
    src="auto",
    font=FontStyling(name="Clear Sans", size=30),
    primary_color="&H00FFFFFF",
    back_color="&H00000000",
    border=BorderAndShadow(outline=1),
    position=Positioning(margin_v=30),
    animation=CaptionAnimation.box_highlight,
)

# Build an editor timeline with tracks and clips
editor_tl = EditorTimeline(conn)
track = Track()
track.add_clip(start=0, clip=Clip(asset=caption, duration=video.length))
editor_tl.add_track(track)
stream_url = editor_tl.generate_stream()
```

### CaptionAsset 파라미터

| 파라미터 | 타입 | 기본값 | 설명 |
|-----------|------|---------|-------------|
| `src` | `str` | `"auto"` | 캡션 소스 (`"auto"` 또는 base64 ASS 문자열) |
| `font` | `FontStyling\|None` | `FontStyling()` | 폰트 스타일링 (이름, 크기, 굵기, 기울임 등) |
| `primary_color` | `str` | `"&H00FFFFFF"` | 기본 텍스트 색상 (ASS 형식) |
| `secondary_color` | `str` | `"&H000000FF"` | 보조 텍스트 색상 (ASS 형식) |
| `back_color` | `str` | `"&H00000000"` | 배경 색상 (ASS 형식) |
| `border` | `BorderAndShadow\|None` | `BorderAndShadow()` | 테두리 및 그림자 스타일링 |
| `position` | `Positioning\|None` | `Positioning()` | 캡션 정렬 및 여백 |
| `animation` | `CaptionAnimation\|None` | `None` | 애니메이션 효과 (예: `box_highlight`, `reveal`, `karaoke`) |

## 컴파일 및 스트리밍

타임라인을 조립한 후 스트리밍 가능한 URL로 컴파일합니다. 스트림은 즉시 생성되며 렌더링 대기 시간이 없습니다.

```python
stream_url = timeline.generate_stream()
print(f"Stream: {stream_url}")
```

더 많은 스트리밍 옵션 (구간 스트림, 검색-스트림, 오디오 재생)은 [streaming.md](streaming.md)를 참고하세요.

## 전체 워크플로 예시

### 제목 카드가 있는 하이라이트 릴

```python
import videodb
from videodb import SearchType
from videodb.exceptions import InvalidRequestError
from videodb.timeline import Timeline
from videodb.asset import VideoAsset, TextAsset, TextStyle

conn = videodb.connect()
coll = conn.get_collection()
video = coll.get_video("your-video-id")

# 1. Search for key moments
video.index_spoken_words(force=True)
try:
    results = video.search("product announcement", search_type=SearchType.semantic)
    shots = results.get_shots()
except InvalidRequestError as exc:
    if "No results found" in str(exc):
        shots = []
    else:
        raise

# 2. Build timeline
timeline = Timeline(conn)

# Title card
title = TextAsset(
    text="Product Launch Highlights",
    duration=4,
    style=TextStyle(fontsize=48, fontcolor="white", boxcolor="#1a1a2e", alpha=0.95),
)
timeline.add_overlay(0, title)

# Append each matching clip
for shot in shots:
    asset = VideoAsset(asset_id=shot.video_id, start=shot.start, end=shot.end)
    timeline.add_inline(asset)

# 3. Generate stream
stream_url = timeline.generate_stream()
print(f"Highlight reel: {stream_url}")
```

### 배경 음악이 있는 로고 오버레이

```python
import videodb
from videodb.timeline import Timeline
from videodb.asset import VideoAsset, AudioAsset, ImageAsset

conn = videodb.connect()
coll = conn.get_collection()

main_video = coll.get_video(main_video_id)
music = coll.get_audio(music_id)
logo = coll.get_image(logo_id)

timeline = Timeline(conn)

# Main video track
timeline.add_inline(VideoAsset(asset_id=main_video.id))

# Background music — disable_other_tracks=False to mix with video audio
timeline.add_overlay(
    0,
    AudioAsset(asset_id=music.id, disable_other_tracks=False, fade_in_duration=3),
)

# Logo in top-right corner for first 10 seconds
timeline.add_overlay(
    0,
    ImageAsset(asset_id=logo.id, duration=10, x=1140, y=20, width=120, height=60),
)

stream_url = timeline.generate_stream()
print(f"Final video: {stream_url}")
```

### 여러 동영상의 멀티클립 몽타주

```python
import videodb
from videodb.timeline import Timeline
from videodb.asset import VideoAsset, TextAsset, TextStyle

conn = videodb.connect()
coll = conn.get_collection()

clips = [
    {"video_id": "vid_001", "start": 5, "end": 15, "label": "Scene 1"},
    {"video_id": "vid_002", "start": 0, "end": 20, "label": "Scene 2"},
    {"video_id": "vid_003", "start": 30, "end": 45, "label": "Scene 3"},
]

timeline = Timeline(conn)
timeline_offset = 0.0

for clip in clips:
    # Add a label as an overlay on each clip
    label = TextAsset(
        text=clip["label"],
        duration=2,
        style=TextStyle(fontsize=32, fontcolor="white", boxcolor="#333333"),
    )
    timeline.add_inline(
        VideoAsset(asset_id=clip["video_id"], start=clip["start"], end=clip["end"])
    )
    timeline.add_overlay(timeline_offset, label)
    timeline_offset += clip["end"] - clip["start"]

stream_url = timeline.generate_stream()
print(f"Montage: {stream_url}")
```

## 두 가지 타임라인 API

VideoDB에는 두 가지 별개의 타임라인 시스템이 있습니다. **서로 교환해서 사용할 수 없습니다**:

| | `videodb.timeline.Timeline` | `videodb.editor.Timeline` (에디터 API) |
|---|---|---|
| **임포트** | `from videodb.timeline import Timeline` | `from videodb.editor import Timeline as EditorTimeline` |
| **에셋** | `VideoAsset`, `AudioAsset`, `ImageAsset`, `TextAsset` | `CaptionAsset`, `Clip`, `Track` |
| **메서드** | `add_inline()`, `add_overlay()` | `Track` / `Clip`과 함께 `add_track()` |
| **적합** | 동영상 합성, 오버레이, 멀티클립 편집 | 애니메이션이 있는 자막/자막 스타일링 |

한 API의 에셋을 다른 API에 혼용하지 마세요. `CaptionAsset`은 에디터 API에서만 작동합니다. `VideoAsset` / `AudioAsset` / `ImageAsset` / `TextAsset`은 `videodb.timeline.Timeline`에서만 작동합니다.

## 제한 사항 및 제약

타임라인 에디터는 **비파괴적 선형 합성**을 위해 설계되었습니다. 다음 작업은 **지원되지 않습니다**:

### 불가능한 것

| 제한 사항 | 세부 정보 |
|---|---|
| **전환 효과 없음** | 클립 간 크로스페이드, 와이프, 디졸브, 전환 없음. 모든 컷은 하드 컷. |
| **동영상 위 동영상 없음 (PiP)** | `add_inline()`은 `VideoAsset`만 허용. 한 동영상 스트림을 다른 것 위에 오버레이 불가. 이미지 오버레이로 정적 PiP를 근사할 수 있지만 실시간 동영상은 불가. |
| **속도 또는 재생 제어 없음** | 슬로우모션, 빨리 감기, 역재생, 타임 리매핑 없음. `VideoAsset`에 `speed` 파라미터 없음. |
| **크롭, 줌, 팬 없음** | 동영상 프레임 영역 크롭, 줌 효과 적용, 프레임 팬 불가. `video.reframe()`은 종횡비 변환 전용. |
| **동영상 필터 또는 색 보정 없음** | 밝기, 대비, 채도, 색조, 색 교정 조정 없음. |
| **텍스트 애니메이션 없음** | `TextAsset`은 전체 시간 동안 정적. 페이드인/아웃, 이동, 애니메이션 없음. 애니메이션 자막은 에디터 API와 함께 `CaptionAsset` 사용. |
| **혼합 텍스트 스타일 없음** | 단일 `TextAsset`은 하나의 `TextStyle`만 가짐. 단일 텍스트 블록 내에서 굵기, 기울임, 색상 혼용 불가. |
| **빈 화면 또는 단색 클립 없음** | 단색 프레임, 검은 화면, 독립적인 제목 카드 생성 불가. 텍스트 및 이미지 오버레이는 인라인 트랙에 아래에 `VideoAsset`이 필요. |
| **오디오 볼륨 제어 없음** | `AudioAsset`에 `volume` 파라미터 없음. 오디오는 최대 볼륨이거나 `disable_other_tracks`로 음소거됨. 줄인 레벨로 믹싱 불가. |
| **키프레임 애니메이션 없음** | 시간에 따라 오버레이 속성을 변경 불가 (예: 위치 A에서 B로 이미지 이동). |

### 제약 사항

| 제약 | 세부 정보 |
|---|---|
| **오디오 페이드 최대 5초** | `fade_in_duration`과 `fade_out_duration`은 각각 5초로 제한됨. |
| **오버레이 위치는 절대값** | 오버레이는 타임라인 시작부터의 절대 타임스탬프 사용. 인라인 클립 재배열이 오버레이를 이동시키지 않음. |
| **인라인 트랙은 동영상만** | `add_inline()`은 `VideoAsset`만 허용. 오디오, 이미지, 텍스트는 `add_overlay()` 사용 필요. |
| **오버레이-클립 바인딩 없음** | 오버레이는 고정된 타임라인 타임스탬프에 배치됨. 오버레이를 특정 인라인 클립에 첨부하여 함께 이동시키는 방법 없음. |

## 팁

- **비파괴적**: 타임라인은 원본 미디어를 수정하지 않습니다. 동일한 에셋으로 여러 타임라인을 만들 수 있습니다.
- **오버레이 스태킹**: 여러 오버레이가 동일한 타임스탬프에 시작할 수 있습니다. 오디오 오버레이는 믹싱되고, 이미지/텍스트 오버레이는 추가 순서대로 레이어됩니다.
- **인라인은 VideoAsset만**: `add_inline()`은 `VideoAsset`만 허용. `AudioAsset`, `ImageAsset`, `TextAsset`은 `add_overlay()` 사용.
- **트림 정밀도**: `VideoAsset`과 `AudioAsset`의 `start`/`end`는 초 단위입니다.
- **동영상 오디오 음소거**: 음악이나 내레이션을 오버레이할 때 원본 동영상 오디오를 음소거하려면 `AudioAsset`에 `disable_other_tracks=True`를 설정하세요.
- **페이드 제한**: `AudioAsset`의 `fade_in_duration`과 `fade_out_duration`은 최대 5초입니다.
- **생성된 미디어**: `coll.generate_music()`, `coll.generate_sound_effect()`, `coll.generate_voice()`, `coll.generate_image()`를 사용하여 즉시 타임라인 에셋으로 사용할 수 있는 미디어를 생성하세요.
