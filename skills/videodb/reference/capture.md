# 캡처 가이드

## 개요

VideoDB 캡처는 AI 처리 기능을 갖춘 실시간 화면 및 오디오 녹화를 지원합니다. 데스크탑 캡처는 현재 **macOS**만 지원합니다.

코드 수준 세부 정보 (SDK 메서드, 이벤트 구조, AI 파이프라인)는 [capture-reference.md](capture-reference.md)를 참고하세요.

## 빠른 시작

1. **WebSocket 리스너 시작**: `python scripts/ws_listener.py --clear &`
2. **캡처 코드 실행** (아래의 전체 캡처 워크플로 참고)
3. **이벤트 저장 위치**: `/tmp/videodb_events.jsonl`

---

## 전체 캡처 워크플로

웹훅이나 폴링이 필요하지 않습니다. WebSocket이 세션 생명주기를 포함한 모든 이벤트를 전달합니다.

> **중요:** `CaptureClient`는 캡처가 진행되는 동안 계속 실행되어야 합니다. 화면/오디오 데이터를 VideoDB로 스트리밍하는 로컬 레코더 바이너리를 실행합니다. `CaptureClient`를 생성한 Python 프로세스가 종료되면, 레코더 바이너리가 종료되고 캡처가 조용히 중단됩니다. 항상 캡처 코드를 **장기 실행 백그라운드 프로세스**로 실행하세요 (예: `nohup python capture_script.py &`). 명시적으로 중지할 때까지 유지하기 위해 신호 처리 (`asyncio.Event` + `SIGINT`/`SIGTERM`)를 사용하세요.

1. **WebSocket 리스너를 백그라운드에서 시작**합니다. `--clear` 플래그로 오래된 이벤트를 삭제합니다. WebSocket ID 파일이 생성될 때까지 기다립니다.

2. **WebSocket ID를 읽습니다**. 이 ID는 캡처 세션과 AI 파이프라인에 필요합니다.

3. **캡처 세션을 생성**하고 데스크탑 클라이언트용 클라이언트 토큰을 생성합니다.

4. **토큰으로 CaptureClient를 초기화**합니다. 마이크 및 화면 캡처 권한을 요청합니다.

5. **채널 목록을 확인하고 선택**합니다 (마이크, 디스플레이, 시스템 오디오). 동영상으로 저장하려는 채널에 `store = True`를 설정합니다.

6. **선택된 채널로 세션을 시작**합니다.

7. **`capture_session.active`를 확인**하여 세션이 활성화될 때까지 이벤트를 읽습니다. 이 이벤트에는 `rtstreams` 배열이 포함됩니다. 다른 스크립트가 읽을 수 있도록 세션 정보 (세션 ID, RTStream ID)를 파일에 저장합니다 (예: `/tmp/videodb_capture_info.json`).

8. **프로세스를 유지합니다.** `asyncio.Event`와 `SIGINT`/`SIGTERM` 신호 처리기를 사용하여 명시적으로 중지될 때까지 블록합니다. `kill $(cat /tmp/videodb_capture_pid)`로 나중에 프로세스를 중지할 수 있도록 PID 파일을 작성합니다 (예: `/tmp/videodb_capture_pid`). 재실행 시 항상 올바른 PID를 갖도록 PID 파일은 매번 덮어써야 합니다.

9. **별도의 명령/스크립트에서 AI 파이프라인을 시작**합니다. 저장된 세션 정보 파일에서 RTStream ID를 읽어 각 RTStream에 오디오 인덱싱과 시각 인덱싱을 적용합니다.

10. **사용 사례에 맞는 커스텀 이벤트 처리 로직을 작성**합니다 (별도의 명령/스크립트에서). 예시:
    - `visual_index`에서 "Slack"이 언급될 때 Slack 활동 기록
    - `audio_index` 이벤트 도착 시 토론 요약
    - `transcript`에 특정 키워드가 나타날 때 알림 트리거
    - 화면 설명에서 애플리케이션 사용량 추적

11. **완료 시 캡처를 중지**합니다 — 캡처 프로세스에 SIGTERM을 전송합니다. 신호 처리기에서 `client.stop_capture()` 및 `client.shutdown()`을 호출해야 합니다.

12. **내보내기를 기다립니다** — `capture_session.exported`가 나타날 때까지 이벤트를 읽습니다. 이 이벤트에는 `exported_video_id`, `stream_url`, `player_url`이 포함됩니다. 캡처 중지 후 몇 초가 걸릴 수 있습니다.

13. **내보내기 이벤트 수신 후 WebSocket 리스너를 중지**합니다. `kill $(cat /tmp/videodb_ws_pid)`로 정상 종료합니다.

---

## 종료 순서

모든 이벤트가 캡처되도록 올바른 종료 순서가 중요합니다:

1. **캡처 세션 중지** — `client.stop_capture()` 후 `client.shutdown()`
2. **내보내기 이벤트 대기** — `capture_session.exported`를 위해 `/tmp/videodb_events.jsonl` 폴링
3. **WebSocket 리스너 중지** — `kill $(cat /tmp/videodb_ws_pid)`

내보내기 이벤트를 수신하기 전에 WebSocket 리스너를 종료하지 마세요. 최종 동영상 URL을 놓칠 수 있습니다.

---

## 스크립트

| 스크립트 | 설명 |
|--------|-------------|
| `scripts/ws_listener.py` | WebSocket 이벤트 리스너 (JSONL로 덤프) |

### ws_listener.py 사용법

```bash
# Start listener in background (append to existing events)
python scripts/ws_listener.py &

# Start listener with clear (new session, clears old events)
python scripts/ws_listener.py --clear &

# Custom output directory
python scripts/ws_listener.py --clear /path/to/events &

# Stop the listener
kill $(cat /tmp/videodb_ws_pid)
```

**옵션:**
- `--clear`: 시작 전 이벤트 파일을 초기화합니다. 새 캡처 세션을 시작할 때 사용합니다.

**출력 파일:**
- `videodb_events.jsonl` - 모든 WebSocket 이벤트
- `videodb_ws_id` - WebSocket 연결 ID (`ws_connection_id` 파라미터용)
- `videodb_ws_pid` - 프로세스 ID (리스너 중지용)

**기능:**
- 연결 끊김 시 지수 백오프로 자동 재연결
- SIGINT/SIGTERM 시 정상 종료
- 간편한 프로세스 관리를 위한 PID 파일
- 연결 상태 로깅
