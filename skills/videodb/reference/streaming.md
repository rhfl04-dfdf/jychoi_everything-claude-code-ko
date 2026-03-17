# 스트리밍 및 재생

VideoDB는 온디맨드로 스트림을 생성하여 모든 표준 동영상 플레이어에서 즉시 재생 가능한 HLS 호환 URL을 반환합니다. 렌더링 시간이나 내보내기 대기가 없습니다 - 편집, 검색, 합성이 즉시 스트리밍됩니다.

## 사전 조건

스트림을 생성하기 전에 동영상은 컬렉션에 **업로드**되어야 합니다. 검색 기반 스트림의 경우 동영상이 **인덱싱** (음성 단어 및/또는 장면)되어 있어야 합니다. 인덱싱 세부 정보는 [search.md](search.md)를 참고하세요.

## 핵심 개념

### 스트림 생성

VideoDB의 모든 동영상, 검색 결과, 타임라인은 **스트림 URL**을 생성할 수 있습니다. 이 URL은 온디맨드로 컴파일되는 HLS (HTTP Live Streaming) 매니페스트를 가리킵니다.

```python
# From a video
stream_url = video.generate_stream()

# From a timeline
stream_url = timeline.generate_stream()

# From search results
stream_url = results.compile()
```

## 단일 동영상 스트리밍

### 기본 재생

```python
import videodb

conn = videodb.connect()
coll = conn.get_collection()
video = coll.get_video("your-video-id")

# Generate stream URL
stream_url = video.generate_stream()
print(f"Stream: {stream_url}")

# Open in default browser
video.play()
```

### 자막 포함

```python
# Index and add subtitles first
video.index_spoken_words(force=True)
stream_url = video.add_subtitle()

# Returned URL already includes subtitles
print(f"Subtitled stream: {stream_url}")
```

### 특정 구간

타임스탬프 범위를 전달하여 동영상의 일부만 스트리밍합니다:

```python
# Stream seconds 10-30 and 60-90
stream_url = video.generate_stream(timeline=[(10, 30), (60, 90)])
print(f"Segment stream: {stream_url}")
```

## 타임라인 합성 스트리밍

멀티 에셋 합성을 구성하고 실시간으로 스트리밍합니다:

```python
import videodb
from videodb.timeline import Timeline
from videodb.asset import VideoAsset, AudioAsset, ImageAsset, TextAsset, TextStyle

conn = videodb.connect()
coll = conn.get_collection()

video = coll.get_video(video_id)
music = coll.get_audio(music_id)

timeline = Timeline(conn)

# Main video content
timeline.add_inline(VideoAsset(asset_id=video.id))

# Background music overlay (starts at second 0)
timeline.add_overlay(0, AudioAsset(asset_id=music.id))

# Text overlay at the beginning
timeline.add_overlay(0, TextAsset(
    text="Live Demo",
    duration=3,
    style=TextStyle(fontsize=48, fontcolor="white", boxcolor="#000000"),
))

# Generate the composed stream
stream_url = timeline.generate_stream()
print(f"Composed stream: {stream_url}")
```

**중요:** `add_inline()`은 `VideoAsset`만 허용합니다. `AudioAsset`, `ImageAsset`, `TextAsset`은 `add_overlay()`를 사용하세요.

자세한 타임라인 편집은 [editor.md](editor.md)를 참고하세요.

## 검색 결과 스트리밍

모든 매칭 구간을 단일 스트림으로 컴파일합니다:

```python
from videodb import SearchType
from videodb.exceptions import InvalidRequestError

video.index_spoken_words(force=True)
try:
    results = video.search("key announcement", search_type=SearchType.semantic)

    # Compile all matching shots into one stream
    stream_url = results.compile()
    print(f"Search results stream: {stream_url}")

    # Or play directly
    results.play()
except InvalidRequestError as exc:
    if "No results found" in str(exc):
        print("No matching announcement segments were found.")
    else:
        raise
```

### 개별 검색 결과 스트리밍

```python
from videodb.exceptions import InvalidRequestError

try:
    results = video.search("product demo", search_type=SearchType.semantic)
    for i, shot in enumerate(results.get_shots()):
        stream_url = shot.generate_stream()
        print(f"Hit {i+1} [{shot.start:.1f}s-{shot.end:.1f}s]: {stream_url}")
except InvalidRequestError as exc:
    if "No results found" in str(exc):
        print("No product demo segments matched the query.")
    else:
        raise
```

## 오디오 재생

오디오 콘텐츠의 서명된 재생 URL을 가져옵니다:

```python
audio = coll.get_audio(audio_id)
playback_url = audio.generate_url()
print(f"Audio URL: {playback_url}")
```

## 전체 워크플로 예시

### 검색-스트림 파이프라인

검색, 타임라인 합성, 스트리밍을 하나의 워크플로로 결합합니다:

```python
import videodb
from videodb import SearchType
from videodb.exceptions import InvalidRequestError
from videodb.timeline import Timeline
from videodb.asset import VideoAsset, TextAsset, TextStyle

conn = videodb.connect()
coll = conn.get_collection()
video = coll.get_video("your-video-id")

video.index_spoken_words(force=True)

# Search for key moments
queries = ["introduction", "main demo", "Q&A"]
timeline = Timeline(conn)
timeline_offset = 0.0

for query in queries:
    try:
        results = video.search(query, search_type=SearchType.semantic)
        shots = results.get_shots()
    except InvalidRequestError as exc:
        if "No results found" in str(exc):
            shots = []
        else:
            raise

    if not shots:
        continue

    # Add the section label where this batch starts in the compiled timeline
    timeline.add_overlay(timeline_offset, TextAsset(
        text=query.title(),
        duration=2,
        style=TextStyle(fontsize=36, fontcolor="white", boxcolor="#222222"),
    ))

    for shot in shots:
        timeline.add_inline(
            VideoAsset(asset_id=shot.video_id, start=shot.start, end=shot.end)
        )
        timeline_offset += shot.end - shot.start

stream_url = timeline.generate_stream()
print(f"Dynamic compilation: {stream_url}")
```

### 멀티 동영상 스트림

다른 동영상의 클립을 단일 스트림으로 결합합니다:

```python
import videodb
from videodb.timeline import Timeline
from videodb.asset import VideoAsset

conn = videodb.connect()
coll = conn.get_collection()

video_clips = [
    {"id": "vid_001", "start": 0, "end": 15},
    {"id": "vid_002", "start": 10, "end": 30},
    {"id": "vid_003", "start": 5, "end": 25},
]

timeline = Timeline(conn)
for clip in video_clips:
    timeline.add_inline(
        VideoAsset(asset_id=clip["id"], start=clip["start"], end=clip["end"])
    )

stream_url = timeline.generate_stream()
print(f"Multi-video stream: {stream_url}")
```

### 조건부 스트림 조립

검색 가용성에 따라 동적으로 스트림을 구성합니다:

```python
import videodb
from videodb import SearchType
from videodb.exceptions import InvalidRequestError
from videodb.timeline import Timeline
from videodb.asset import VideoAsset, TextAsset, TextStyle

conn = videodb.connect()
coll = conn.get_collection()
video = coll.get_video("your-video-id")

video.index_spoken_words(force=True)

timeline = Timeline(conn)

# Try to find specific content; fall back to full video
topics = ["opening remarks", "technical deep dive", "closing"]

found_any = False
timeline_offset = 0.0
for topic in topics:
    try:
        results = video.search(topic, search_type=SearchType.semantic)
        shots = results.get_shots()
    except InvalidRequestError as exc:
        if "No results found" in str(exc):
            shots = []
        else:
            raise

    if shots:
        found_any = True
        timeline.add_overlay(timeline_offset, TextAsset(
            text=topic.title(),
            duration=2,
            style=TextStyle(fontsize=32, fontcolor="white", boxcolor="#1a1a2e"),
        ))
        for shot in shots:
            timeline.add_inline(
                VideoAsset(asset_id=shot.video_id, start=shot.start, end=shot.end)
            )
            timeline_offset += shot.end - shot.start

if found_any:
    stream_url = timeline.generate_stream()
    print(f"Curated stream: {stream_url}")
else:
    # Fall back to full video stream
    stream_url = video.generate_stream()
    print(f"Full video stream: {stream_url}")
```

### 라이브 이벤트 요약

이벤트 녹화를 여러 섹션이 있는 스트리밍 가능한 요약으로 처리합니다:

```python
import videodb
from videodb import SearchType
from videodb.exceptions import InvalidRequestError
from videodb.timeline import Timeline
from videodb.asset import VideoAsset, AudioAsset, ImageAsset, TextAsset, TextStyle

conn = videodb.connect()
coll = conn.get_collection()

# Upload event recording
event = coll.upload(url="https://example.com/event-recording.mp4")
event.index_spoken_words(force=True)

# Generate background music
music = coll.generate_music(
    prompt="upbeat corporate background music",
    duration=120,
)

# Generate title image
title_img = coll.generate_image(
    prompt="modern event recap title card, dark background, professional",
    aspect_ratio="16:9",
)

# Build the recap timeline
timeline = Timeline(conn)
timeline_offset = 0.0

# Main video segments from search
try:
    keynote = event.search("keynote announcement", search_type=SearchType.semantic)
    keynote_shots = keynote.get_shots()[:5]
except InvalidRequestError as exc:
    if "No results found" in str(exc):
        keynote_shots = []
    else:
        raise
if keynote_shots:
    keynote_start = timeline_offset
    for shot in keynote_shots:
        timeline.add_inline(
            VideoAsset(asset_id=shot.video_id, start=shot.start, end=shot.end)
        )
        timeline_offset += shot.end - shot.start
else:
    keynote_start = None

try:
    demo = event.search("product demo", search_type=SearchType.semantic)
    demo_shots = demo.get_shots()[:5]
except InvalidRequestError as exc:
    if "No results found" in str(exc):
        demo_shots = []
    else:
        raise
if demo_shots:
    demo_start = timeline_offset
    for shot in demo_shots:
        timeline.add_inline(
            VideoAsset(asset_id=shot.video_id, start=shot.start, end=shot.end)
        )
        timeline_offset += shot.end - shot.start
else:
    demo_start = None

# Overlay title card image
timeline.add_overlay(0, ImageAsset(
    asset_id=title_img.id, width=100, height=100, x=80, y=20, duration=5
))

# Overlay section labels at the correct timeline offsets
if keynote_start is not None:
    timeline.add_overlay(max(5, keynote_start), TextAsset(
        text="Keynote Highlights",
        duration=3,
        style=TextStyle(fontsize=40, fontcolor="white", boxcolor="#0d1117"),
    ))
if demo_start is not None:
    timeline.add_overlay(max(5, demo_start), TextAsset(
        text="Demo Highlights",
        duration=3,
        style=TextStyle(fontsize=36, fontcolor="white", boxcolor="#0d1117"),
    ))

# Overlay background music
timeline.add_overlay(0, AudioAsset(
    asset_id=music.id, fade_in_duration=3
))

# Stream the final recap
stream_url = timeline.generate_stream()
print(f"Event recap: {stream_url}")
```

---

## 팁

- **HLS 호환성**: 스트림 URL은 HLS 매니페스트 (`.m3u8`)를 반환합니다. Safari에서는 기본적으로 작동하며, 다른 브라우저에서는 hls.js 등의 라이브러리를 사용하세요.
- **온디맨드 컴파일**: 스트림은 요청 시 서버 측에서 컴파일됩니다. 첫 번째 재생은 짧은 컴파일 지연이 있을 수 있으며, 이후 동일한 합성의 재생은 캐싱됩니다.
- **캐싱**: 인수 없이 `video.generate_stream()`을 두 번째 호출하면 재컴파일 없이 캐싱된 스트림 URL이 반환됩니다.
- **구간 스트림**: `video.generate_stream(timeline=[(start, end)])`은 전체 `Timeline` 객체를 구성하지 않고 특정 클립을 스트리밍하는 가장 빠른 방법입니다.
- **인라인 vs 오버레이**: `add_inline()`은 `VideoAsset`만 허용하며 메인 트랙에 순차적으로 에셋을 배치합니다. `add_overlay()`는 `AudioAsset`, `ImageAsset`, `TextAsset`을 허용하며 지정된 시작 시간에 위에 레이어링합니다.
- **TextStyle 기본값**: `TextStyle`은 기본적으로 `font='Sans'`, `fontcolor='black'`입니다. 텍스트의 배경색에는 `bgcolor`가 아닌 `boxcolor`를 사용하세요.
- **생성과 결합**: `coll.generate_music(prompt, duration)`과 `coll.generate_image(prompt, aspect_ratio)`를 사용하여 타임라인 합성을 위한 에셋을 생성하세요.
- **재생**: `.play()`는 기본 시스템 브라우저에서 스트림 URL을 엽니다. 프로그래밍 방식으로 사용할 때는 URL 문자열로 직접 작업하세요.
