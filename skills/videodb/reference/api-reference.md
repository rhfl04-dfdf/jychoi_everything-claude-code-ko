# 전체 API 레퍼런스

VideoDB skill의 레퍼런스 자료입니다. 사용 가이드와 워크플로 선택은 [../SKILL.md](../SKILL.md)에서 시작하세요.

## 연결

```python
import videodb

conn = videodb.connect(
    api_key="your-api-key",      # or set VIDEO_DB_API_KEY env var
    base_url=None,                # custom API endpoint (optional)
)
```

**반환값:** `Connection` 객체

### Connection 메서드

| 메서드 | 반환값 | 설명 |
|--------|---------|-------------|
| `conn.get_collection(collection_id="default")` | `Collection` | 컬렉션 가져오기 (ID 없으면 기본값) |
| `conn.get_collections()` | `list[Collection]` | 모든 컬렉션 목록 |
| `conn.create_collection(name, description, is_public=False)` | `Collection` | 새 컬렉션 생성 |
| `conn.update_collection(id, name, description)` | `Collection` | 컬렉션 업데이트 |
| `conn.check_usage()` | `dict` | 계정 사용량 통계 조회 |
| `conn.upload(source, media_type, name, ...)` | `Video\|Audio\|Image` | 기본 컬렉션에 업로드 |
| `conn.record_meeting(meeting_url, bot_name, ...)` | `Meeting` | 회의 녹화 |
| `conn.create_capture_session(...)` | `CaptureSession` | 캡처 세션 생성 ([capture-reference.md](capture-reference.md) 참고) |
| `conn.youtube_search(query, result_threshold, duration)` | `list[dict]` | YouTube 검색 |
| `conn.transcode(source, callback_url, mode, ...)` | `str` | 동영상 트랜스코딩 (작업 ID 반환) |
| `conn.get_transcode_details(job_id)` | `dict` | 트랜스코딩 작업 상태 및 세부 정보 조회 |
| `conn.connect_websocket(collection_id)` | `WebSocketConnection` | WebSocket 연결 ([capture-reference.md](capture-reference.md) 참고) |

### 트랜스코드

URL에서 커스텀 해상도, 품질, 오디오 설정으로 동영상을 트랜스코딩합니다. 처리는 서버 측에서 이루어지며 로컬 ffmpeg가 필요하지 않습니다.

```python
from videodb import TranscodeMode, VideoConfig, AudioConfig

job_id = conn.transcode(
    source="https://example.com/video.mp4",
    callback_url="https://example.com/webhook",
    mode=TranscodeMode.economy,
    video_config=VideoConfig(resolution=720, quality=23),
    audio_config=AudioConfig(mute=False),
)
```

#### transcode 파라미터

| 파라미터 | 타입 | 기본값 | 설명 |
|-----------|------|---------|-------------|
| `source` | `str` | 필수 | 트랜스코딩할 동영상 URL (다운로드 가능한 URL 권장) |
| `callback_url` | `str` | 필수 | 트랜스코딩 완료 시 콜백을 수신할 URL |
| `mode` | `TranscodeMode` | `TranscodeMode.economy` | 트랜스코딩 속도: `economy` 또는 `lightning` |
| `video_config` | `VideoConfig` | `VideoConfig()` | 비디오 인코딩 설정 |
| `audio_config` | `AudioConfig` | `AudioConfig()` | 오디오 인코딩 설정 |

작업 ID(`str`)를 반환합니다. `conn.get_transcode_details(job_id)`로 작업 상태를 확인하세요.

```python
details = conn.get_transcode_details(job_id)
```

#### VideoConfig

```python
from videodb import VideoConfig, ResizeMode

config = VideoConfig(
    resolution=720,              # Target resolution height (e.g. 480, 720, 1080)
    quality=23,                  # Encoding quality (lower = better, default 23)
    framerate=30,                # Target framerate
    aspect_ratio="16:9",         # Target aspect ratio
    resize_mode=ResizeMode.crop, # How to fit: crop, fit, or pad
)
```

| 필드 | 타입 | 기본값 | 설명 |
|-------|------|---------|-------------|
| `resolution` | `int\|None` | `None` | 목표 해상도 높이 (픽셀) |
| `quality` | `int` | `23` | 인코딩 품질 (낮을수록 높은 품질) |
| `framerate` | `int\|None` | `None` | 목표 프레임레이트 |
| `aspect_ratio` | `str\|None` | `None` | 목표 종횡비 (예: `"16:9"`, `"9:16"`) |
| `resize_mode` | `str` | `ResizeMode.crop` | 크기 조정 전략: `crop`, `fit`, 또는 `pad` |

#### AudioConfig

```python
from videodb import AudioConfig

config = AudioConfig(mute=False)
```

| 필드 | 타입 | 기본값 | 설명 |
|-------|------|---------|-------------|
| `mute` | `bool` | `False` | 오디오 트랙 음소거 |

## 컬렉션

```python
coll = conn.get_collection()
```

### Collection 메서드

| 메서드 | 반환값 | 설명 |
|--------|---------|-------------|
| `coll.get_videos()` | `list[Video]` | 모든 동영상 목록 |
| `coll.get_video(video_id)` | `Video` | 특정 동영상 가져오기 |
| `coll.get_audios()` | `list[Audio]` | 모든 오디오 목록 |
| `coll.get_audio(audio_id)` | `Audio` | 특정 오디오 가져오기 |
| `coll.get_images()` | `list[Image]` | 모든 이미지 목록 |
| `coll.get_image(image_id)` | `Image` | 특정 이미지 가져오기 |
| `coll.upload(url=None, file_path=None, media_type=None, name=None)` | `Video\|Audio\|Image` | 미디어 업로드 |
| `coll.search(query, search_type, index_type, score_threshold, namespace, scene_index_id, ...)` | `SearchResult` | 컬렉션 전체 검색 (시맨틱만 지원; 키워드 및 장면 검색은 `NotImplementedError` 발생) |
| `coll.generate_image(prompt, aspect_ratio="1:1")` | `Image` | AI로 이미지 생성 |
| `coll.generate_video(prompt, duration=5)` | `Video` | AI로 동영상 생성 |
| `coll.generate_music(prompt, duration=5)` | `Audio` | AI로 음악 생성 |
| `coll.generate_sound_effect(prompt, duration=2)` | `Audio` | 효과음 생성 |
| `coll.generate_voice(text, voice_name="Default")` | `Audio` | 텍스트에서 음성 생성 |
| `coll.generate_text(prompt, model_name="basic", response_type="text")` | `dict` | LLM 텍스트 생성 — `["output"]`으로 결과 접근 |
| `coll.dub_video(video_id, language_code)` | `Video` | 동영상을 다른 언어로 더빙 |
| `coll.record_meeting(meeting_url, bot_name, ...)` | `Meeting` | 실시간 회의 녹화 |
| `coll.create_capture_session(...)` | `CaptureSession` | 캡처 세션 생성 ([capture-reference.md](capture-reference.md) 참고) |
| `coll.get_capture_session(...)` | `CaptureSession` | 캡처 세션 조회 ([capture-reference.md](capture-reference.md) 참고) |
| `coll.connect_rtstream(url, name, ...)` | `RTStream` | 실시간 스트림 연결 ([rtstream-reference.md](rtstream-reference.md) 참고) |
| `coll.make_public()` | `None` | 컬렉션 공개 설정 |
| `coll.make_private()` | `None` | 컬렉션 비공개 설정 |
| `coll.delete_video(video_id)` | `None` | 동영상 삭제 |
| `coll.delete_audio(audio_id)` | `None` | 오디오 삭제 |
| `coll.delete_image(image_id)` | `None` | 이미지 삭제 |
| `coll.delete()` | `None` | 컬렉션 삭제 |

### 업로드 파라미터

```python
video = coll.upload(
    url=None,            # Remote URL (HTTP, YouTube)
    file_path=None,      # Local file path
    media_type=None,     # "video", "audio", or "image" (auto-detected if omitted)
    name=None,           # Custom name for the media
    description=None,    # Description
    callback_url=None,   # Webhook URL for async notification
)
```

## Video 객체

```python
video = coll.get_video(video_id)
```

### Video 속성

| 속성 | 타입 | 설명 |
|----------|------|-------------|
| `video.id` | `str` | 고유 동영상 ID |
| `video.collection_id` | `str` | 상위 컬렉션 ID |
| `video.name` | `str` | 동영상 이름 |
| `video.description` | `str` | 동영상 설명 |
| `video.length` | `float` | 길이 (초) |
| `video.stream_url` | `str` | 기본 스트림 URL |
| `video.player_url` | `str` | 플레이어 임베드 URL |
| `video.thumbnail_url` | `str` | 썸네일 URL |

### Video 메서드

| 메서드 | 반환값 | 설명 |
|--------|---------|-------------|
| `video.generate_stream(timeline=None)` | `str` | 스트림 URL 생성 (선택적으로 `[(start, end)]` 튜플의 타임라인) |
| `video.play()` | `str` | 브라우저에서 스트림 열기, 플레이어 URL 반환 |
| `video.index_spoken_words(language_code=None, force=False)` | `None` | 검색용 음성 인덱싱. 이미 인덱싱되었으면 `force=True`로 건너뜀. |
| `video.index_scenes(extraction_type, prompt, extraction_config, metadata, model_name, name, scenes, callback_url)` | `str` | 시각적 장면 인덱싱 (scene_index_id 반환) |
| `video.index_visuals(prompt, batch_config, ...)` | `str` | 시각 콘텐츠 인덱싱 (scene_index_id 반환) |
| `video.index_audio(prompt, model_name, ...)` | `str` | LLM으로 오디오 인덱싱 (scene_index_id 반환) |
| `video.get_transcript(start=None, end=None)` | `list[dict]` | 타임스탬프가 있는 전사 가져오기 |
| `video.get_transcript_text(start=None, end=None)` | `str` | 전체 전사 텍스트 가져오기 |
| `video.generate_transcript(force=None)` | `dict` | 전사 생성 |
| `video.translate_transcript(language, additional_notes)` | `list[dict]` | 전사 번역 |
| `video.search(query, search_type, index_type, filter, **kwargs)` | `SearchResult` | 동영상 내 검색 |
| `video.add_subtitle(style=SubtitleStyle())` | `str` | 자막 추가 (스트림 URL 반환) |
| `video.generate_thumbnail(time=None)` | `str\|Image` | 썸네일 생성 |
| `video.get_thumbnails()` | `list[Image]` | 모든 썸네일 가져오기 |
| `video.extract_scenes(extraction_type, extraction_config)` | `SceneCollection` | 장면 추출 |
| `video.reframe(start, end, target, mode, callback_url)` | `Video\|None` | 동영상 종횡비 변환 |
| `video.clip(prompt, content_type, model_name)` | `str` | 프롬프트로 클립 생성 (스트림 URL 반환) |
| `video.insert_video(video, timestamp)` | `str` | 타임스탬프에 동영상 삽입 |
| `video.download(name=None)` | `dict` | 동영상 다운로드 |
| `video.delete()` | `None` | 동영상 삭제 |

### Reframe

서버 측에서 선택적 스마트 객체 추적으로 동영상을 다른 종횡비로 변환합니다.

> **경고:** Reframe은 느린 서버 측 작업입니다. 긴 동영상의 경우 몇 분이 걸릴 수 있으며 타임아웃이 발생할 수 있습니다. 항상 `start`/`end`로 구간을 제한하거나, 비동기 처리를 위해 `callback_url`을 전달하세요.

```python
from videodb import ReframeMode

# Always prefer short segments to avoid timeouts:
reframed = video.reframe(start=0, end=60, target="vertical", mode=ReframeMode.smart)

# Async reframe for full-length videos (returns None, result via webhook):
video.reframe(target="vertical", callback_url="https://example.com/webhook")

# Custom dimensions
reframed = video.reframe(start=0, end=60, target={"width": 1080, "height": 1080})
```

#### reframe 파라미터

| 파라미터 | 타입 | 기본값 | 설명 |
|-----------|------|---------|-------------|
| `start` | `float\|None` | `None` | 시작 시간 (초, None = 처음부터) |
| `end` | `float\|None` | `None` | 종료 시간 (초, None = 동영상 끝까지) |
| `target` | `str\|dict` | `"vertical"` | 프리셋 문자열 (`"vertical"`, `"square"`, `"landscape"`) 또는 `{"width": int, "height": int}` |
| `mode` | `str` | `ReframeMode.smart` | `"simple"` (중앙 크롭) 또는 `"smart"` (객체 추적) |
| `callback_url` | `str\|None` | `None` | 비동기 알림용 웹훅 URL |

`callback_url`이 없으면 `Video` 객체 반환, 있으면 `None` 반환.

## Audio 객체

```python
audio = coll.get_audio(audio_id)
```

### Audio 속성

| 속성 | 타입 | 설명 |
|----------|------|-------------|
| `audio.id` | `str` | 고유 오디오 ID |
| `audio.collection_id` | `str` | 상위 컬렉션 ID |
| `audio.name` | `str` | 오디오 이름 |
| `audio.length` | `float` | 길이 (초) |

### Audio 메서드

| 메서드 | 반환값 | 설명 |
|--------|---------|-------------|
| `audio.generate_url()` | `str` | 재생용 서명된 URL 생성 |
| `audio.get_transcript(start=None, end=None)` | `list[dict]` | 타임스탬프가 있는 전사 가져오기 |
| `audio.get_transcript_text(start=None, end=None)` | `str` | 전체 전사 텍스트 가져오기 |
| `audio.generate_transcript(force=None)` | `dict` | 전사 생성 |
| `audio.delete()` | `None` | 오디오 삭제 |

## Image 객체

```python
image = coll.get_image(image_id)
```

### Image 속성

| 속성 | 타입 | 설명 |
|----------|------|-------------|
| `image.id` | `str` | 고유 이미지 ID |
| `image.collection_id` | `str` | 상위 컬렉션 ID |
| `image.name` | `str` | 이미지 이름 |
| `image.url` | `str\|None` | 이미지 URL (생성된 이미지의 경우 `None`일 수 있음 — 대신 `generate_url()` 사용) |

### Image 메서드

| 메서드 | 반환값 | 설명 |
|--------|---------|-------------|
| `image.generate_url()` | `str` | 서명된 URL 생성 |
| `image.delete()` | `None` | 이미지 삭제 |

## Timeline 및 에디터

### Timeline

```python
from videodb.timeline import Timeline

timeline = Timeline(conn)
```

| 메서드 | 반환값 | 설명 |
|--------|---------|-------------|
| `timeline.add_inline(asset)` | `None` | 메인 트랙에 `VideoAsset`을 순차적으로 추가 |
| `timeline.add_overlay(start, asset)` | `None` | 타임스탬프에 `AudioAsset`, `ImageAsset`, 또는 `TextAsset` 오버레이 |
| `timeline.generate_stream()` | `str` | 컴파일하고 스트림 URL 가져오기 |

### Asset 타입

#### VideoAsset

```python
from videodb.asset import VideoAsset

asset = VideoAsset(
    asset_id=video.id,
    start=0,              # trim start (seconds)
    end=None,             # trim end (seconds, None = full)
)
```

#### AudioAsset

```python
from videodb.asset import AudioAsset

asset = AudioAsset(
    asset_id=audio.id,
    start=0,
    end=None,
    disable_other_tracks=True,   # mute original audio when True
    fade_in_duration=0,          # seconds (max 5)
    fade_out_duration=0,         # seconds (max 5)
)
```

#### ImageAsset

```python
from videodb.asset import ImageAsset

asset = ImageAsset(
    asset_id=image.id,
    duration=None,        # display duration (seconds)
    width=100,            # display width
    height=100,           # display height
    x=80,                 # horizontal position (px from left)
    y=20,                 # vertical position (px from top)
)
```

#### TextAsset

```python
from videodb.asset import TextAsset, TextStyle

asset = TextAsset(
    text="Hello World",
    duration=5,
    style=TextStyle(
        fontsize=24,
        fontcolor="black",
        boxcolor="white",       # background box colour
        alpha=1.0,
        font="Sans",
        text_align="T",         # text alignment within box
    ),
)
```

#### CaptionAsset (에디터 API)

CaptionAsset은 자체 Timeline, Track, Clip 시스템을 가진 에디터 API에 속합니다:

```python
from videodb.editor import CaptionAsset, FontStyling

asset = CaptionAsset(
    src="auto",                    # "auto" or base64 ASS string
    font=FontStyling(name="Clear Sans", size=30),
    primary_color="&H00FFFFFF",
)
```

에디터 API와 함께하는 전체 CaptionAsset 사용법은 [editor.md](editor.md#caption-overlays)를 참고하세요.

## 동영상 검색 파라미터

```python
results = video.search(
    query="your query",
    search_type=SearchType.semantic,       # semantic, keyword, or scene
    index_type=IndexType.spoken_word,      # spoken_word or scene
    result_threshold=None,                 # max number of results
    score_threshold=None,                  # minimum relevance score
    dynamic_score_percentage=None,         # percentage of dynamic score
    scene_index_id=None,                   # target a specific scene index (pass via **kwargs)
    filter=[],                             # metadata filters for scene search
)
```

> **참고:** `filter`는 `video.search()`의 명시적 명명 파라미터입니다. `scene_index_id`는 `**kwargs`를 통해 API로 전달됩니다.
>
> **중요:** `video.search()`는 결과가 없으면 `"No results found"` 메시지와 함께 `InvalidRequestError`를 발생시킵니다. 항상 검색 호출을 try/except로 감싸세요. 장면 검색의 경우 낮은 관련성 노이즈를 필터링하려면 `score_threshold=0.3` 이상을 사용하세요.

장면 검색은 `index_type=IndexType.scene`과 함께 `search_type=SearchType.semantic`을 사용하세요. 특정 장면 인덱스를 대상으로 할 때 `scene_index_id`를 전달하세요. 자세한 내용은 [search.md](search.md)를 참고하세요.

## SearchResult 객체

```python
results = video.search("query", search_type=SearchType.semantic)
```

| 메서드 | 반환값 | 설명 |
|--------|---------|-------------|
| `results.get_shots()` | `list[Shot]` | 매칭 구간 목록 가져오기 |
| `results.compile()` | `str` | 모든 shot을 스트림 URL로 컴파일 |
| `results.play()` | `str` | 브라우저에서 컴파일된 스트림 열기 |

### Shot 속성

| 속성 | 타입 | 설명 |
|----------|------|-------------|
| `shot.video_id` | `str` | 소스 동영상 ID |
| `shot.video_length` | `float` | 소스 동영상 길이 |
| `shot.video_title` | `str` | 소스 동영상 제목 |
| `shot.start` | `float` | 시작 시간 (초) |
| `shot.end` | `float` | 종료 시간 (초) |
| `shot.text` | `str` | 매칭된 텍스트 콘텐츠 |
| `shot.search_score` | `float` | 검색 관련성 점수 |

| 메서드 | 반환값 | 설명 |
|--------|---------|-------------|
| `shot.generate_stream()` | `str` | 이 특정 shot 스트리밍 |
| `shot.play()` | `str` | 브라우저에서 shot 스트림 열기 |

## Meeting 객체

```python
meeting = coll.record_meeting(
    meeting_url="https://meet.google.com/...",
    bot_name="Bot",
    callback_url=None,          # Webhook URL for status updates
    callback_data=None,         # Optional dict passed through to callbacks
    time_zone="UTC",            # Time zone for the meeting
)
```

### Meeting 속성

| 속성 | 타입 | 설명 |
|----------|------|-------------|
| `meeting.id` | `str` | 고유 회의 ID |
| `meeting.collection_id` | `str` | 상위 컬렉션 ID |
| `meeting.status` | `str` | 현재 상태 |
| `meeting.video_id` | `str` | 녹화된 동영상 ID (완료 후) |
| `meeting.bot_name` | `str` | 봇 이름 |
| `meeting.meeting_title` | `str` | 회의 제목 |
| `meeting.meeting_url` | `str` | 회의 URL |
| `meeting.speaker_timeline` | `dict` | 발언자 타임라인 데이터 |
| `meeting.is_active` | `bool` | 초기화 또는 처리 중이면 True |
| `meeting.is_completed` | `bool` | 완료되면 True |

### Meeting 메서드

| 메서드 | 반환값 | 설명 |
|--------|---------|-------------|
| `meeting.refresh()` | `Meeting` | 서버에서 데이터 새로고침 |
| `meeting.wait_for_status(target_status, timeout=14400, interval=120)` | `bool` | 상태에 도달할 때까지 폴링 |

## RTStream 및 캡처

RTStream (실시간 수집, 인덱싱, 전사)에 대해서는 [rtstream-reference.md](rtstream-reference.md)를 참고하세요.

캡처 세션 (데스크탑 녹화, CaptureClient, 채널)에 대해서는 [capture-reference.md](capture-reference.md)를 참고하세요.

## 열거형 및 상수

### SearchType

```python
from videodb import SearchType

SearchType.semantic    # Natural language semantic search
SearchType.keyword     # Exact keyword matching
SearchType.scene       # Visual scene search (may require paid plan)
SearchType.llm         # LLM-powered search
```

### SceneExtractionType

```python
from videodb import SceneExtractionType

SceneExtractionType.shot_based   # Automatic shot boundary detection
SceneExtractionType.time_based   # Fixed time interval extraction
SceneExtractionType.transcript   # Transcript-based scene extraction
```

### SubtitleStyle

```python
from videodb import SubtitleStyle

style = SubtitleStyle(
    font_name="Arial",
    font_size=18,
    primary_colour="&H00FFFFFF",
    bold=False,
    # ... see SubtitleStyle for all options
)
video.add_subtitle(style=style)
```

### SubtitleAlignment 및 SubtitleBorderStyle

```python
from videodb import SubtitleAlignment, SubtitleBorderStyle
```

### TextStyle

```python
from videodb import TextStyle
# or: from videodb.asset import TextStyle

style = TextStyle(
    fontsize=24,
    fontcolor="black",
    boxcolor="white",
    font="Sans",
    text_align="T",
    alpha=1.0,
)
```

### 기타 상수

```python
from videodb import (
    IndexType,          # spoken_word, scene
    MediaType,          # video, audio, image
    Segmenter,          # word, sentence, time
    SegmentationType,   # sentence, llm
    TranscodeMode,      # economy, lightning
    ResizeMode,         # crop, fit, pad
    ReframeMode,        # simple, smart
    RTStreamChannelType,
)
```

## 예외

```python
from videodb.exceptions import (
    AuthenticationError,     # Invalid or missing API key
    InvalidRequestError,     # Bad parameters or malformed request
    RequestTimeoutError,     # Request timed out
    SearchError,             # Search operation failure (e.g. not indexed)
    VideodbError,            # Base exception for all VideoDB errors
)
```

| 예외 | 일반적인 원인 |
|-----------|-------------|
| `AuthenticationError` | `VIDEO_DB_API_KEY` 누락 또는 유효하지 않음 |
| `InvalidRequestError` | 유효하지 않은 URL, 지원되지 않는 형식, 잘못된 파라미터 |
| `RequestTimeoutError` | 서버 응답 시간 초과 |
| `SearchError` | 인덱싱 전 검색, 유효하지 않은 검색 타입 |
| `VideodbError` | 서버 오류, 네트워크 문제, 일반 실패 |
