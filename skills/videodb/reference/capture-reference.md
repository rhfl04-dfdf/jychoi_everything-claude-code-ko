# 캡처 레퍼런스

VideoDB 캡처 세션의 코드 수준 세부 정보입니다. 워크플로 가이드는 [capture.md](capture.md)를 참고하세요.

---

## WebSocket 이벤트

캡처 세션 및 AI 파이프라인에서 발생하는 실시간 이벤트. 웹훅이나 폴링이 필요하지 않습니다.

[scripts/ws_listener.py](../scripts/ws_listener.py)를 사용하여 연결하고 이벤트를 `${VIDEODB_EVENTS_DIR:-$HOME/.local/state/videodb}/videodb_events.jsonl`에 덤프합니다.

### 이벤트 채널

| 채널 | 소스 | 콘텐츠 |
|---------|--------|---------|
| `capture_session` | 세션 생명주기 | 상태 변경 |
| `transcript` | `start_transcript()` | 음성-텍스트 변환 |
| `visual_index` / `scene_index` | `index_visuals()` | 시각적 분석 |
| `audio_index` | `index_audio()` | 오디오 분석 |
| `alert` | `create_alert()` | 알림 알람 |

### 세션 생명주기 이벤트

| 이벤트 | 상태 | 주요 데이터 |
|-------|--------|----------|
| `capture_session.created` | `created` | — |
| `capture_session.starting` | `starting` | — |
| `capture_session.active` | `active` | `rtstreams[]` |
| `capture_session.stopping` | `stopping` | — |
| `capture_session.stopped` | `stopped` | — |
| `capture_session.exported` | `exported` | `exported_video_id`, `stream_url`, `player_url` |
| `capture_session.failed` | `failed` | `error` |

### 이벤트 구조

**전사 이벤트:**
```json
{
  "channel": "transcript",
  "rtstream_id": "rts-xxx",
  "rtstream_name": "mic:default",
  "data": {
    "text": "Let's schedule the meeting for Thursday",
    "is_final": true,
    "start": 1710000001234,
    "end": 1710000002345
  }
}
```

**시각 인덱스 이벤트:**
```json
{
  "channel": "visual_index",
  "rtstream_id": "rts-xxx",
  "rtstream_name": "display:1",
  "data": {
    "text": "User is viewing a Slack conversation with 3 unread messages",
    "start": 1710000012340,
    "end": 1710000018900
  }
}
```

**오디오 인덱스 이벤트:**
```json
{
  "channel": "audio_index",
  "rtstream_id": "rts-xxx",
  "rtstream_name": "mic:default",
  "data": {
    "text": "Discussion about scheduling a team meeting",
    "start": 1710000021500,
    "end": 1710000029200
  }
}
```

**세션 활성 이벤트:**
```json
{
  "event": "capture_session.active",
  "capture_session_id": "cap-xxx",
  "status": "active",
  "data": {
    "rtstreams": [
      { "rtstream_id": "rts-1", "name": "mic:default", "media_types": ["audio"] },
      { "rtstream_id": "rts-2", "name": "system_audio:default", "media_types": ["audio"] },
      { "rtstream_id": "rts-3", "name": "display:1", "media_types": ["video"] }
    ]
  }
}
```

**세션 내보내기 이벤트:**
```json
{
  "event": "capture_session.exported",
  "capture_session_id": "cap-xxx",
  "status": "exported",
  "data": {
    "exported_video_id": "v_xyz789",
    "stream_url": "https://stream.videodb.io/...",
    "player_url": "https://console.videodb.io/player?url=..."
  }
}
```

> 최신 세부 정보는 [VideoDB 실시간 컨텍스트 문서](https://docs.videodb.io/pages/ingest/capture-sdks/realtime-context.md)를 참고하세요.

---

## 이벤트 영속성

`ws_listener.py`를 사용하여 모든 WebSocket 이벤트를 JSONL 파일에 덤프하고 나중에 분석할 수 있습니다.

### 리스너 시작 및 WebSocket ID 가져오기

```bash
# Start with --clear to clear old events (recommended for new sessions)
python scripts/ws_listener.py --clear &

# Append to existing events (for reconnects)
python scripts/ws_listener.py &
```

또는 커스텀 출력 디렉토리 지정:

```bash
python scripts/ws_listener.py --clear /path/to/output &
# Or via environment variable:
VIDEODB_EVENTS_DIR=/path/to/output python scripts/ws_listener.py --clear &
```

스크립트는 첫 번째 줄에 `WS_ID=<connection_id>`를 출력한 후 무한 대기합니다.

**ws_id 가져오기:**
```bash
cat "${VIDEODB_EVENTS_DIR:-$HOME/.local/state/videodb}/videodb_ws_id"
```

**리스너 중지:**
```bash
kill "$(cat "${VIDEODB_EVENTS_DIR:-$HOME/.local/state/videodb}/videodb_ws_pid")"
```

**`ws_connection_id`를 허용하는 함수:**

| 함수 | 용도 |
|----------|---------|
| `conn.create_capture_session()` | 세션 생명주기 이벤트 |
| RTStream 메서드 | [rtstream-reference.md](rtstream-reference.md) 참고 |

**출력 파일** (출력 디렉토리, 기본값 `${XDG_STATE_HOME:-$HOME/.local/state}/videodb`):
- `videodb_ws_id` - WebSocket 연결 ID
- `videodb_events.jsonl` - 모든 이벤트
- `videodb_ws_pid` - 쉬운 종료를 위한 프로세스 ID

**기능:**
- 새 세션 시작 시 이벤트 파일을 초기화하는 `--clear` 플래그
- 연결 끊김 시 지수 백오프로 자동 재연결
- SIGINT/SIGTERM 시 정상 종료
- 연결 상태 로깅

### JSONL 형식

각 줄은 타임스탬프가 추가된 JSON 객체입니다:

```json
{"ts": "2026-03-02T10:15:30.123Z", "unix_ts": 1772446530.123, "channel": "visual_index", "data": {"text": "..."}}
{"ts": "2026-03-02T10:15:31.456Z", "unix_ts": 1772446531.456, "event": "capture_session.active", "capture_session_id": "cap-xxx"}
```

### 이벤트 읽기

```python
import json
import time
from pathlib import Path

events_path = Path.home() / ".local" / "state" / "videodb" / "videodb_events.jsonl"
transcripts = []
recent = []
visual = []

cutoff = time.time() - 600
with events_path.open(encoding="utf-8") as handle:
    for line in handle:
        event = json.loads(line)
        if event.get("channel") == "transcript":
            transcripts.append(event)
        if event.get("unix_ts", 0) > cutoff:
            recent.append(event)
        if (
            event.get("channel") == "visual_index"
            and "code" in event.get("data", {}).get("text", "").lower()
        ):
            visual.append(event)
```

---

## WebSocket 연결

실시간 전사 및 인덱싱 파이프라인에서 AI 결과를 수신하기 위해 연결합니다.

```python
ws_wrapper = conn.connect_websocket()
ws = await ws_wrapper.connect()
ws_id = ws.connection_id
```

| 속성 / 메서드 | 타입 | 설명 |
|-------------------|------|-------------|
| `ws.connection_id` | `str` | 고유 연결 ID (AI 파이프라인 메서드에 전달) |
| `ws.receive()` | `AsyncIterator[dict]` | 실시간 메시지를 생성하는 비동기 이터레이터 |

---

## CaptureSession

### Connection 메서드

| 메서드 | 반환값 | 설명 |
|--------|---------|-------------|
| `conn.create_capture_session(end_user_id, collection_id, ws_connection_id, metadata)` | `CaptureSession` | 새 캡처 세션 생성 |
| `conn.get_capture_session(capture_session_id)` | `CaptureSession` | 기존 캡처 세션 조회 |
| `conn.generate_client_token()` | `str` | 클라이언트 측 인증 토큰 생성 |

### 캡처 세션 생성

```python
from pathlib import Path

ws_id = (Path.home() / ".local" / "state" / "videodb" / "videodb_ws_id").read_text().strip()

session = conn.create_capture_session(
    end_user_id="user-123",  # required
    collection_id="default",
    ws_connection_id=ws_id,
    metadata={"app": "my-app"},
)
print(f"Session ID: {session.id}")
```

> **참고:** `end_user_id`는 필수이며 캡처를 시작하는 사용자를 식별합니다. 테스트나 데모 목적으로는 어떤 고유 문자열 식별자도 사용 가능합니다 (예: `"demo-user"`, `"test-123"`).

### CaptureSession 속성

| 속성 | 타입 | 설명 |
|----------|------|-------------|
| `session.id` | `str` | 고유 캡처 세션 ID |

### CaptureSession 메서드

| 메서드 | 반환값 | 설명 |
|--------|---------|-------------|
| `session.get_rtstream(type)` | `list[RTStream]` | 타입별 RTStream 가져오기: `"mic"`, `"screen"`, 또는 `"system_audio"` |

### 클라이언트 토큰 생성

```python
token = conn.generate_client_token()
```

---

## CaptureClient

클라이언트는 사용자 기기에서 실행되며 권한, 채널 검색, 스트리밍을 처리합니다.

```python
from videodb.capture import CaptureClient

client = CaptureClient(client_token=token)
```

### CaptureClient 메서드

| 메서드 | 반환값 | 설명 |
|--------|---------|-------------|
| `await client.request_permission(type)` | `None` | 기기 권한 요청 (`"microphone"`, `"screen_capture"`) |
| `await client.list_channels()` | `Channels` | 사용 가능한 오디오/비디오 채널 검색 |
| `await client.start_capture_session(capture_session_id, channels, primary_video_channel_id)` | `None` | 선택된 채널 스트리밍 시작 |
| `await client.stop_capture()` | `None` | 캡처 세션 정상 중지 |
| `await client.shutdown()` | `None` | 클라이언트 리소스 정리 |

### 권한 요청

```python
await client.request_permission("microphone")
await client.request_permission("screen_capture")
```

### 세션 시작

```python
selected_channels = [c for c in [mic, display, system_audio] if c]
await client.start_capture_session(
    capture_session_id=session.id,
    channels=selected_channels,
    primary_video_channel_id=display.id if display else None,
)
```

### 세션 중지

```python
await client.stop_capture()
await client.shutdown()
```

---

## 채널

`client.list_channels()`가 반환합니다. 사용 가능한 기기를 타입별로 그룹화합니다.

```python
channels = await client.list_channels()
for ch in channels.all():
    print(f"  {ch.id} ({ch.type}): {ch.name}")

mic = channels.mics.default
display = channels.displays.default
system_audio = channels.system_audio.default
```

### 채널 그룹

| 속성 | 타입 | 설명 |
|----------|------|-------------|
| `channels.mics` | `ChannelGroup` | 사용 가능한 마이크 |
| `channels.displays` | `ChannelGroup` | 사용 가능한 화면 디스플레이 |
| `channels.system_audio` | `ChannelGroup` | 사용 가능한 시스템 오디오 소스 |

### ChannelGroup 메서드 및 속성

| 멤버 | 타입 | 설명 |
|--------|------|-------------|
| `group.default` | `Channel` | 그룹의 기본 채널 (또는 `None`) |
| `group.all()` | `list[Channel]` | 그룹의 모든 채널 |

### Channel 속성

| 속성 | 타입 | 설명 |
|----------|------|-------------|
| `ch.id` | `str` | 고유 채널 ID |
| `ch.type` | `str` | 채널 타입 (`"mic"`, `"display"`, `"system_audio"`) |
| `ch.name` | `str` | 사람이 읽을 수 있는 채널 이름 |
| `ch.store` | `bool` | 녹화 지속 여부 (저장하려면 `True`로 설정) |

`store = True` 없이는 스트림이 실시간으로 처리되지만 저장되지 않습니다.

---

## RTStream 및 AI 파이프라인

세션이 활성화된 후 `session.get_rtstream()`으로 RTStream 객체를 가져옵니다.

RTStream 메서드 (인덱싱, 전사, 알림, 배치 구성)에 대해서는 [rtstream-reference.md](rtstream-reference.md)를 참고하세요.

---

## 세션 생명주기

```
  create_capture_session()
          │
          v
  ┌───────────────┐
  │    created     │
  └───────┬───────┘
          │  client.start_capture_session()
          v
  ┌───────────────┐     WebSocket: capture_session.starting
  │   starting     │ ──> Capture channels connect
  └───────┬───────┘
          │
          v
  ┌───────────────┐     WebSocket: capture_session.active
  │    active      │ ──> Start AI pipelines
  └───────┬──────────────┐
          │              │
          │              v
          │      ┌───────────────┐     WebSocket: capture_session.failed
          │      │    failed      │ ──> Inspect error payload and retry setup
          │      └───────────────┘
          │      unrecoverable capture error
          │
          │  client.stop_capture()
          v
  ┌───────────────┐     WebSocket: capture_session.stopping
  │   stopping     │ ──> Finalize streams
  └───────┬───────┘
          │
          v
  ┌───────────────┐     WebSocket: capture_session.stopped
  │   stopped      │ ──> All streams finalized
  └───────┬───────┘
          │  (if store=True)
          v
  ┌───────────────┐     WebSocket: capture_session.exported
  │   exported     │ ──> Access video_id, stream_url, player_url
  └───────────────┘
```
