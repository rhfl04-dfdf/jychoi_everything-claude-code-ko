# RTStream 레퍼런스

RTStream 작업의 코드 수준 세부 정보입니다. 워크플로 가이드는 [rtstream.md](rtstream.md)를 참고하세요.
사용 가이드와 워크플로 선택은 [../SKILL.md](../SKILL.md)에서 시작하세요.

[docs.videodb.io](https://docs.videodb.io/pages/ingest/live-streams/realtime-apis.md) 기반.

---

## Collection RTStream 메서드

RTStream을 관리하기 위한 `Collection`의 메서드:

| 메서드 | 반환값 | 설명 |
|--------|---------|-------------|
| `coll.connect_rtstream(url, name, ...)` | `RTStream` | RTSP/RTMP URL로 새 RTStream 생성 |
| `coll.get_rtstream(id)` | `RTStream` | ID로 기존 RTStream 가져오기 |
| `coll.list_rtstreams(limit, offset, status, name, ordering)` | `List[RTStream]` | 컬렉션의 모든 RTStream 목록 |
| `coll.search(query, namespace="rtstream")` | `RTStreamSearchResult` | 모든 RTStream에서 검색 |

### RTStream 연결

```python
import videodb

conn = videodb.connect()
coll = conn.get_collection()

rtstream = coll.connect_rtstream(
    url="rtmp://your-stream-server/live/stream-key",
    name="My Live Stream",
    media_types=["video"],  # or ["audio", "video"]
    sample_rate=30,         # optional
    store=True,             # enable recording storage for export
    enable_transcript=True, # optional
    ws_connection_id=ws_id, # optional, for real-time events
)
```

### 기존 RTStream 가져오기

```python
rtstream = coll.get_rtstream("rts-xxx")
```

### RTStream 목록

```python
rtstreams = coll.list_rtstreams(
    limit=10,
    offset=0,
    status="connected",  # optional filter
    name="meeting",      # optional filter
    ordering="-created_at",
)

for rts in rtstreams:
    print(f"{rts.id}: {rts.name} - {rts.status}")
```

### 캡처 세션에서 가져오기

캡처 세션이 활성화된 후 RTStream 객체를 가져옵니다:

```python
session = conn.get_capture_session(session_id)

mics = session.get_rtstream("mic")
displays = session.get_rtstream("screen")
system_audios = session.get_rtstream("system_audio")
```

또는 `capture_session.active` WebSocket 이벤트의 `rtstreams` 데이터를 사용합니다:

```python
for rts in rtstreams:
    rtstream = coll.get_rtstream(rts["rtstream_id"])
```

---

## RTStream 메서드

| 메서드 | 반환값 | 설명 |
|--------|---------|-------------|
| `rtstream.start()` | `None` | 수집 시작 |
| `rtstream.stop()` | `None` | 수집 중지 |
| `rtstream.generate_stream(start, end)` | `str` | 녹화된 구간 스트리밍 (Unix 타임스탬프) |
| `rtstream.export(name=None)` | `RTStreamExportResult` | 영구 동영상으로 내보내기 |
| `rtstream.index_visuals(prompt, ...)` | `RTStreamSceneIndex` | AI 분석으로 시각 인덱스 생성 |
| `rtstream.index_audio(prompt, ...)` | `RTStreamSceneIndex` | LLM 요약으로 오디오 인덱스 생성 |
| `rtstream.list_scene_indexes()` | `List[RTStreamSceneIndex]` | 스트림의 모든 장면 인덱스 목록 |
| `rtstream.get_scene_index(index_id)` | `RTStreamSceneIndex` | 특정 장면 인덱스 가져오기 |
| `rtstream.search(query, ...)` | `RTStreamSearchResult` | 인덱싱된 콘텐츠 검색 |
| `rtstream.start_transcript(ws_connection_id, engine)` | `dict` | 실시간 전사 시작 |
| `rtstream.get_transcript(page, page_size, start, end, since)` | `dict` | 전사 페이지 가져오기 |
| `rtstream.stop_transcript(engine)` | `dict` | 전사 중지 |

---

## 시작 및 중지

```python
# Begin ingestion
rtstream.start()

# ... stream is being recorded ...

# Stop ingestion
rtstream.stop()
```

---

## 스트림 생성

녹화된 콘텐츠에서 재생 스트림을 생성하려면 (초 오프셋이 아닌) Unix 타임스탬프를 사용합니다:

```python
import time

start_ts = time.time()
rtstream.start()

# Let it record for a while...
time.sleep(60)

end_ts = time.time()
rtstream.stop()

# Generate a stream URL for the recorded segment
stream_url = rtstream.generate_stream(start=start_ts, end=end_ts)
print(f"Recorded stream: {stream_url}")
```

---

## 동영상으로 내보내기

녹화된 스트림을 컬렉션의 영구 동영상으로 내보냅니다:

```python
export_result = rtstream.export(name="Meeting Recording 2024-01-15")

print(f"Video ID: {export_result.video_id}")
print(f"Stream URL: {export_result.stream_url}")
print(f"Player URL: {export_result.player_url}")
print(f"Duration: {export_result.duration}s")
```

### RTStreamExportResult 속성

| 속성 | 타입 | 설명 |
|----------|------|-------------|
| `video_id` | `str` | 내보낸 동영상의 ID |
| `stream_url` | `str` | HLS 스트림 URL |
| `player_url` | `str` | 웹 플레이어 URL |
| `name` | `str` | 동영상 이름 |
| `duration` | `float` | 길이 (초) |

---

## AI 파이프라인

AI 파이프라인은 실시간 스트림을 처리하고 WebSocket으로 결과를 전송합니다.

### RTStream AI 파이프라인 메서드

| 메서드 | 반환값 | 설명 |
|--------|---------|-------------|
| `rtstream.index_audio(prompt, batch_config, ...)` | `RTStreamSceneIndex` | LLM 요약으로 오디오 인덱싱 시작 |
| `rtstream.index_visuals(prompt, batch_config, ...)` | `RTStreamSceneIndex` | 화면 콘텐츠 시각 인덱싱 시작 |

### 오디오 인덱싱

간격을 두고 오디오 콘텐츠의 LLM 요약을 생성합니다:

```python
audio_index = rtstream.index_audio(
    prompt="Summarize what is being discussed",
    batch_config={"type": "word", "value": 50},
    model_name=None,       # optional
    name="meeting_audio",  # optional
    ws_connection_id=ws_id,
)
```

**오디오 batch_config 옵션:**

| 타입 | 값 | 설명 |
|------|-------|-------------|
| `"word"` | 개수 | N 단어마다 구간 분할 |
| `"sentence"` | 개수 | N 문장마다 구간 분할 |
| `"time"` | 초 | N초마다 구간 분할 |

예시:
```python
{"type": "word", "value": 50}      # every 50 words
{"type": "sentence", "value": 5}   # every 5 sentences
{"type": "time", "value": 30}      # every 30 seconds
```

결과는 `audio_index` WebSocket 채널로 전달됩니다.

### 시각 인덱싱

시각적 콘텐츠의 AI 설명을 생성합니다:

```python
scene_index = rtstream.index_visuals(
    prompt="Describe what is happening on screen",
    batch_config={"type": "time", "value": 2, "frame_count": 5},
    model_name="basic",
    name="screen_monitor",  # optional
    ws_connection_id=ws_id,
)
```

**파라미터:**

| 파라미터 | 타입 | 설명 |
|-----------|------|-------------|
| `prompt` | `str` | AI 모델에 대한 지시 (구조화된 JSON 출력 지원) |
| `batch_config` | `dict` | 프레임 샘플링 제어 (아래 참고) |
| `model_name` | `str` | 모델 티어: `"mini"`, `"basic"`, `"pro"`, `"ultra"` |
| `name` | `str` | 인덱스 이름 (선택사항) |
| `ws_connection_id` | `str` | 결과 수신용 WebSocket 연결 ID |

**시각 batch_config:**

| 키 | 타입 | 설명 |
|-----|------|-------------|
| `type` | `str` | 시각에는 `"time"`만 지원 |
| `value` | `int` | 윈도우 크기 (초) |
| `frame_count` | `int` | 윈도우당 추출할 프레임 수 |

예시: `{"type": "time", "value": 2, "frame_count": 5}`는 2초마다 5개 프레임을 샘플링하여 모델에 전송합니다.

**구조화된 JSON 출력:**

구조화된 응답을 위해 JSON 형식을 요청하는 프롬프트 사용:

```python
scene_index = rtstream.index_visuals(
    prompt="""Analyze the screen and return a JSON object with:
{
  "app_name": "name of the active application",
  "activity": "what the user is doing",
  "ui_elements": ["list of visible UI elements"],
  "contains_text": true/false,
  "dominant_colors": ["list of main colors"]
}
Return only valid JSON.""",
    batch_config={"type": "time", "value": 3, "frame_count": 3},
    model_name="pro",
    ws_connection_id=ws_id,
)
```

결과는 `scene_index` WebSocket 채널로 전달됩니다.

---

## Batch Config 요약

| 인덱싱 타입 | `type` 옵션 | `value` | 추가 키 |
|---------------|----------------|---------|------------|
| **오디오** | `"word"`, `"sentence"`, `"time"` | 단어/문장/초 | - |
| **시각** | `"time"` 전용 | 초 | `frame_count` |

예시:
```python
# Audio: every 50 words
{"type": "word", "value": 50}

# Audio: every 30 seconds  
{"type": "time", "value": 30}

# Visual: 5 frames every 2 seconds
{"type": "time", "value": 2, "frame_count": 5}
```

---

## 전사

WebSocket을 통한 실시간 전사:

```python
# Start live transcription
rtstream.start_transcript(
    ws_connection_id=ws_id,
    engine=None,  # optional, defaults to "assemblyai"
)

# Get transcript pages (with optional filters)
transcript = rtstream.get_transcript(
    page=1,
    page_size=100,
    start=None,   # optional: start timestamp filter
    end=None,     # optional: end timestamp filter
    since=None,   # optional: for polling, get transcripts after this timestamp
    engine=None,
)

# Stop transcription
rtstream.stop_transcript(engine=None)
```

전사 결과는 `transcript` WebSocket 채널로 전달됩니다.

---

## RTStreamSceneIndex

`index_audio()` 또는 `index_visuals()`를 호출하면 `RTStreamSceneIndex` 객체가 반환됩니다. 이 객체는 실행 중인 인덱스를 나타내며 장면 및 알림 관리를 위한 메서드를 제공합니다.

```python
# index_visuals returns an RTStreamSceneIndex
scene_index = rtstream.index_visuals(
    prompt="Describe what is on screen",
    ws_connection_id=ws_id,
)

# index_audio also returns an RTStreamSceneIndex
audio_index = rtstream.index_audio(
    prompt="Summarize the discussion",
    ws_connection_id=ws_id,
)
```

### RTStreamSceneIndex 속성

| 속성 | 타입 | 설명 |
|----------|------|-------------|
| `rtstream_index_id` | `str` | 인덱스의 고유 ID |
| `rtstream_id` | `str` | 상위 RTStream의 ID |
| `extraction_type` | `str` | 추출 타입 (`time` 또는 `transcript`) |
| `extraction_config` | `dict` | 추출 구성 |
| `prompt` | `str` | 분석에 사용된 프롬프트 |
| `name` | `str` | 인덱스 이름 |
| `status` | `str` | 상태 (`connected`, `stopped`) |

### RTStreamSceneIndex 메서드

| 메서드 | 반환값 | 설명 |
|--------|---------|-------------|
| `index.get_scenes(start, end, page, page_size)` | `dict` | 인덱싱된 장면 가져오기 |
| `index.start()` | `None` | 인덱스 시작/재개 |
| `index.stop()` | `None` | 인덱스 중지 |
| `index.create_alert(event_id, callback_url, ws_connection_id)` | `str` | 이벤트 감지를 위한 알림 생성 |
| `index.list_alerts()` | `list` | 이 인덱스의 모든 알림 목록 |
| `index.enable_alert(alert_id)` | `None` | 알림 활성화 |
| `index.disable_alert(alert_id)` | `None` | 알림 비활성화 |

### 장면 가져오기

인덱스에서 인덱싱된 장면을 폴링합니다:

```python
result = scene_index.get_scenes(
    start=None,      # optional: start timestamp
    end=None,        # optional: end timestamp
    page=1,
    page_size=100,
)

for scene in result["scenes"]:
    print(f"[{scene['start']}-{scene['end']}] {scene['text']}")

if result["next_page"]:
    # fetch next page
    pass
```

### 장면 인덱스 관리

```python
# List all indexes on the stream
indexes = rtstream.list_scene_indexes()

# Get a specific index by ID
scene_index = rtstream.get_scene_index(index_id)

# Stop an index
scene_index.stop()

# Restart an index
scene_index.start()
```

---

## 이벤트

이벤트는 재사용 가능한 감지 규칙입니다. 한 번 생성하고 알림을 통해 어떤 인덱스에도 연결할 수 있습니다.

### Connection 이벤트 메서드

| 메서드 | 반환값 | 설명 |
|--------|---------|-------------|
| `conn.create_event(event_prompt, label)` | `str` (event_id) | 감지 이벤트 생성 |
| `conn.list_events()` | `list` | 모든 이벤트 목록 |

### 이벤트 생성

```python
event_id = conn.create_event(
    event_prompt="User opened Slack application",
    label="slack_opened",
)
```

### 이벤트 목록

```python
events = conn.list_events()
for event in events:
    print(f"{event['event_id']}: {event['label']}")
```

---

## 알림

알림은 이벤트를 인덱스에 연결하여 실시간 알림을 제공합니다. AI가 이벤트 설명과 일치하는 콘텐츠를 감지하면 알림이 전송됩니다.

### 알림 생성

```python
# Get the RTStreamSceneIndex from index_visuals
scene_index = rtstream.index_visuals(
    prompt="Describe what application is open on screen",
    ws_connection_id=ws_id,
)

# Create an alert on the index
alert_id = scene_index.create_alert(
    event_id=event_id,
    callback_url="https://your-backend.com/alerts",  # for webhook delivery
    ws_connection_id=ws_id,  # for WebSocket delivery (optional)
)
```

**참고:** `callback_url`은 필수입니다. WebSocket 전달만 사용하는 경우 빈 문자열 `""`을 전달하세요.

### 알림 관리

```python
# List all alerts on an index
alerts = scene_index.list_alerts()

# Enable/disable alerts
scene_index.disable_alert(alert_id)
scene_index.enable_alert(alert_id)
```

### 알림 전달

| 방법 | 지연 | 사용 사례 |
|--------|---------|----------|
| WebSocket | 실시간 | 대시보드, 라이브 UI |
| 웹훅 | 1초 미만 | 서버 간, 자동화 |

### WebSocket 알림 이벤트

```json
{
  "channel": "alert",
  "rtstream_id": "rts-xxx",
  "data": {
    "event_label": "slack_opened",
    "timestamp": 1710000012340,
    "text": "User opened Slack application"
  }
}
```

### 웹훅 페이로드

```json
{
  "event_id": "event-xxx",
  "label": "slack_opened",
  "confidence": 0.95,
  "explanation": "User opened the Slack application",
  "timestamp": "2024-01-15T10:30:45Z",
  "start_time": 1234.5,
  "end_time": 1238.0,
  "stream_url": "https://stream.videodb.io/v3/...",
  "player_url": "https://console.videodb.io/player?url=..."
}
```

---

## WebSocket 통합

모든 실시간 AI 결과는 WebSocket으로 전달됩니다. 다음에 `ws_connection_id`를 전달하세요:
- `rtstream.start_transcript()`
- `rtstream.index_audio()`
- `rtstream.index_visuals()`
- `scene_index.create_alert()`

### WebSocket 채널

| 채널 | 소스 | 콘텐츠 |
|---------|--------|---------|
| `transcript` | `start_transcript()` | 실시간 음성-텍스트 변환 |
| `scene_index` | `index_visuals()` | 시각적 분석 결과 |
| `audio_index` | `index_audio()` | 오디오 분석 결과 |
| `alert` | `create_alert()` | 알림 알람 |

WebSocket 이벤트 구조와 ws_listener 사용법은 [capture-reference.md](capture-reference.md)를 참고하세요.

---

## 전체 워크플로

```python
import time
import videodb
from videodb.exceptions import InvalidRequestError

conn = videodb.connect()
coll = conn.get_collection()

# 1. Connect and start recording
rtstream = coll.connect_rtstream(
    url="rtmp://your-stream-server/live/stream-key",
    name="Weekly Standup",
    store=True,
)
rtstream.start()

# 2. Record for the duration of the meeting
start_ts = time.time()
time.sleep(1800)  # 30 minutes
end_ts = time.time()
rtstream.stop()

# Generate an immediate playback URL for the captured window
stream_url = rtstream.generate_stream(start=start_ts, end=end_ts)
print(f"Recorded stream: {stream_url}")

# 3. Export to a permanent video
export_result = rtstream.export(name="Weekly Standup Recording")
print(f"Exported video: {export_result.video_id}")

# 4. Index the exported video for search
video = coll.get_video(export_result.video_id)
video.index_spoken_words(force=True)

# 5. Search for action items
try:
    results = video.search("action items and next steps")
    stream_url = results.compile()
    print(f"Action items clip: {stream_url}")
except InvalidRequestError as exc:
    if "No results found" in str(exc):
        print("No action items were detected in the recording.")
    else:
        raise
```
