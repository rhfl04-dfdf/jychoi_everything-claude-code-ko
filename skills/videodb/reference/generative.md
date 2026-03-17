# 생성형 미디어 가이드

VideoDB는 AI 기반 이미지, 동영상, 음악, 효과음, 음성, 텍스트 콘텐츠 생성을 지원합니다. 모든 생성 메서드는 **Collection** 객체에 있습니다.

## 사전 조건

생성 메서드를 호출하기 전에 연결과 컬렉션 참조가 필요합니다:

```python
import videodb

conn = videodb.connect()
coll = conn.get_collection()
```

## 이미지 생성

텍스트 프롬프트에서 이미지를 생성합니다:

```python
image = coll.generate_image(
    prompt="a futuristic cityscape at sunset with flying cars",
    aspect_ratio="16:9",
)

# Access the generated image
print(image.id)
print(image.generate_url())  # returns a signed download URL
```

### generate_image 파라미터

| 파라미터 | 타입 | 기본값 | 설명 |
|-----------|------|---------|-------------|
| `prompt` | `str` | 필수 | 생성할 이미지의 텍스트 설명 |
| `aspect_ratio` | `str` | `"1:1"` | 종횡비: `"1:1"`, `"9:16"`, `"16:9"`, `"4:3"`, 또는 `"3:4"` |
| `callback_url` | `str\|None` | `None` | 비동기 콜백을 수신할 URL |

`.id`, `.name`, `.collection_id`가 있는 `Image` 객체를 반환합니다. `.url` 속성은 생성된 이미지의 경우 `None`일 수 있습니다 — 신뢰할 수 있는 서명된 다운로드 URL을 얻으려면 항상 `image.generate_url()`을 사용하세요.

> **참고:** `Video` 객체 (`.generate_stream()` 사용)와 달리, `Image` 객체는 `.generate_url()`을 사용하여 이미지 URL을 가져옵니다. `.url` 속성은 일부 이미지 타입 (예: 썸네일)에만 채워집니다.

## 동영상 생성

텍스트 프롬프트에서 짧은 동영상 클립을 생성합니다:

```python
video = coll.generate_video(
    prompt="a timelapse of a flower blooming in a garden",
    duration=5,
)

stream_url = video.generate_stream()
video.play()
```

### generate_video 파라미터

| 파라미터 | 타입 | 기본값 | 설명 |
|-----------|------|---------|-------------|
| `prompt` | `str` | 필수 | 생성할 동영상의 텍스트 설명 |
| `duration` | `int` | `5` | 길이 (초, 정수값이어야 함, 5-8) |
| `callback_url` | `str\|None` | `None` | 비동기 콜백을 수신할 URL |

`Video` 객체를 반환합니다. 생성된 동영상은 자동으로 컬렉션에 추가되며 업로드된 동영상과 마찬가지로 타임라인, 검색, 컴파일에 사용할 수 있습니다.

## 오디오 생성

VideoDB는 다양한 오디오 타입을 위한 세 가지 별도 메서드를 제공합니다.

### 음악

텍스트 설명에서 배경 음악을 생성합니다:

```python
music = coll.generate_music(
    prompt="upbeat electronic music with a driving beat, suitable for a tech demo",
    duration=30,
)

print(music.id)
```

| 파라미터 | 타입 | 기본값 | 설명 |
|-----------|------|---------|-------------|
| `prompt` | `str` | 필수 | 음악의 텍스트 설명 |
| `duration` | `int` | `5` | 길이 (초) |
| `callback_url` | `str\|None` | `None` | 비동기 콜백을 수신할 URL |

### 효과음

특정 효과음을 생성합니다:

```python
sfx = coll.generate_sound_effect(
    prompt="thunderstorm with heavy rain and distant thunder",
    duration=10,
)
```

| 파라미터 | 타입 | 기본값 | 설명 |
|-----------|------|---------|-------------|
| `prompt` | `str` | 필수 | 효과음의 텍스트 설명 |
| `duration` | `int` | `2` | 길이 (초) |
| `config` | `dict` | `{}` | 추가 구성 |
| `callback_url` | `str\|None` | `None` | 비동기 콜백을 수신할 URL |

### 음성 (텍스트-음성 변환)

텍스트에서 음성을 생성합니다:

```python
voice = coll.generate_voice(
    text="Welcome to our product demo. Today we'll walk through the key features.",
    voice_name="Default",
)
```

| 파라미터 | 타입 | 기본값 | 설명 |
|-----------|------|---------|-------------|
| `text` | `str` | 필수 | 음성으로 변환할 텍스트 |
| `voice_name` | `str` | `"Default"` | 사용할 음성 |
| `config` | `dict` | `{}` | 추가 구성 |
| `callback_url` | `str\|None` | `None` | 비동기 콜백을 수신할 URL |

세 오디오 메서드 모두 `.id`, `.name`, `.length`, `.collection_id`가 있는 `Audio` 객체를 반환합니다.

## 텍스트 생성 (LLM 통합)

`coll.generate_text()`를 사용하여 LLM 분석을 실행합니다. 이것은 **컬렉션 수준** 메서드입니다 — 프롬프트 문자열에 직접 컨텍스트 (전사, 설명)를 전달합니다.

```python
# Get transcript from a video first
transcript_text = video.get_transcript_text()

# Generate analysis using collection LLM
result = coll.generate_text(
    prompt=f"Summarize the key points discussed in this video:\n{transcript_text}",
    model_name="pro",
)

print(result["output"])
```

### generate_text 파라미터

| 파라미터 | 타입 | 기본값 | 설명 |
|-----------|------|---------|-------------|
| `prompt` | `str` | 필수 | LLM을 위한 컨텍스트가 있는 프롬프트 |
| `model_name` | `str` | `"basic"` | 모델 티어: `"basic"`, `"pro"`, 또는 `"ultra"` |
| `response_type` | `str` | `"text"` | 응답 형식: `"text"` 또는 `"json"` |

`output` 키가 있는 `dict`를 반환합니다. `response_type="text"`일 때 `output`은 `str`입니다. `response_type="json"`일 때 `output`은 `dict`입니다.

```python
result = coll.generate_text(prompt="Summarize this", model_name="pro")
print(result["output"])  # access the actual text/dict
```

### LLM으로 장면 분석

장면 추출과 텍스트 생성을 결합합니다:

```python
from videodb import SceneExtractionType

# First index scenes
scenes = video.index_scenes(
    extraction_type=SceneExtractionType.time_based,
    extraction_config={"time": 10},
    prompt="Describe the visual content in this scene.",
)

# Get transcript for spoken context
transcript_text = video.get_transcript_text()
scene_descriptions = []
for scene in scenes:
    if isinstance(scene, dict):
        description = scene.get("description") or scene.get("summary")
    else:
        description = getattr(scene, "description", None) or getattr(scene, "summary", None)
    scene_descriptions.append(description or str(scene))

scenes_text = "\n".join(scene_descriptions)

# Analyze with collection LLM
result = coll.generate_text(
    prompt=(
        f"Given this video transcript:\n{transcript_text}\n\n"
        f"And these visual scene descriptions:\n{scenes_text}\n\n"
        "Based on the spoken and visual content, describe the main topics covered."
    ),
    model_name="pro",
)
print(result["output"])
```

## 더빙 및 번역

### 동영상 더빙

컬렉션 메서드를 사용하여 동영상을 다른 언어로 더빙합니다:

```python
dubbed_video = coll.dub_video(
    video_id=video.id,
    language_code="es",  # Spanish
)

dubbed_video.play()
```

### dub_video 파라미터

| 파라미터 | 타입 | 기본값 | 설명 |
|-----------|------|---------|-------------|
| `video_id` | `str` | 필수 | 더빙할 동영상의 ID |
| `language_code` | `str` | 필수 | 목표 언어 코드 (예: `"es"`, `"fr"`, `"de"`) |
| `callback_url` | `str\|None` | `None` | 비동기 콜백을 수신할 URL |

더빙된 콘텐츠가 있는 `Video` 객체를 반환합니다.

### 전사 번역

더빙 없이 동영상 전사를 번역합니다:

```python
translated = video.translate_transcript(
    language="Spanish",
    additional_notes="Use formal tone",
)

for entry in translated:
    print(entry)
```

**지원 언어**: `en`, `es`, `fr`, `de`, `it`, `pt`, `ja`, `ko`, `zh`, `hi`, `ar` 등.

## 전체 워크플로 예시

### 동영상에 내레이션 생성

```python
import videodb

conn = videodb.connect()
coll = conn.get_collection()
video = coll.get_video("your-video-id")

# Get transcript
transcript_text = video.get_transcript_text()

# Generate narration script using collection LLM
result = coll.generate_text(
    prompt=(
        f"Write a professional narration script for this video content:\n"
        f"{transcript_text[:2000]}"
    ),
    model_name="pro",
)
script = result["output"]

# Convert script to speech
narration = coll.generate_voice(text=script)
print(f"Narration audio: {narration.id}")
```

### 프롬프트에서 썸네일 생성

```python
thumbnail = coll.generate_image(
    prompt="professional video thumbnail showing data analytics dashboard, modern design",
    aspect_ratio="16:9",
)
print(f"Thumbnail URL: {thumbnail.generate_url()}")
```

### 동영상에 생성된 음악 추가

```python
import videodb
from videodb.timeline import Timeline
from videodb.asset import VideoAsset, AudioAsset

conn = videodb.connect()
coll = conn.get_collection()
video = coll.get_video("your-video-id")

# Generate background music
music = coll.generate_music(
    prompt="calm ambient background music for a tutorial video",
    duration=60,
)

# Build timeline with video + music overlay
timeline = Timeline(conn)
timeline.add_inline(VideoAsset(asset_id=video.id))
timeline.add_overlay(0, AudioAsset(asset_id=music.id, disable_other_tracks=False))

stream_url = timeline.generate_stream()
print(f"Video with music: {stream_url}")
```

### 구조화된 JSON 출력

```python
transcript_text = video.get_transcript_text()

result = coll.generate_text(
    prompt=(
        f"Given this transcript:\n{transcript_text}\n\n"
        "Return a JSON object with keys: summary, topics (array), action_items (array)."
    ),
    model_name="pro",
    response_type="json",
)

# result["output"] is a dict when response_type="json"
print(result["output"]["summary"])
print(result["output"]["topics"])
```

## 팁

- **생성된 미디어는 영구적**: 모든 생성된 콘텐츠는 컬렉션에 저장되며 재사용 가능합니다.
- **세 가지 오디오 메서드**: 배경 음악에는 `generate_music()`, 효과음에는 `generate_sound_effect()`, 텍스트-음성 변환에는 `generate_voice()` 사용. 통합 `generate_audio()` 메서드는 없습니다.
- **텍스트 생성은 컬렉션 수준**: `coll.generate_text()`는 동영상 콘텐츠에 자동으로 접근하지 않습니다. `video.get_transcript_text()`로 전사를 가져와 프롬프트에 전달하세요.
- **모델 티어**: `"basic"`이 가장 빠르고, `"pro"`는 균형 잡혀 있으며, `"ultra"`는 최고 품질입니다. 대부분의 분석 작업에는 `"pro"`를 사용하세요.
- **생성 타입 결합**: 오버레이용 이미지, 배경용 음악, 내레이션용 음성을 생성한 다음 타임라인으로 합성하세요 ([editor.md](editor.md) 참고).
- **프롬프트 품질이 중요**: 구체적이고 설명적인 프롬프트가 모든 생성 타입에서 더 좋은 결과를 만들어냅니다.
- **이미지 종횡비**: `"1:1"`, `"9:16"`, `"16:9"`, `"4:3"`, `"3:4"` 중 선택하세요.
