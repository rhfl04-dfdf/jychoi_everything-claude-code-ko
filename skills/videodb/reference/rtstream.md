# RTStream 가이드

## 개요

RTStream은 라이브 동영상 스트림 (RTSP/RTMP) 및 데스크탑 캡처 세션의 실시간 수집을 지원합니다. 연결 후 라이브 소스의 콘텐츠를 녹화, 인덱싱, 검색, 내보내기할 수 있습니다.

코드 수준 세부 정보 (SDK 메서드, 파라미터, 예시)는 [rtstream-reference.md](rtstream-reference.md)를 참고하세요.

## 사용 사례

- **보안 및 모니터링**: RTSP 카메라 연결, 이벤트 감지, 알림 트리거
- **라이브 방송**: RTMP 스트림 수집, 실시간 인덱싱, 즉시 검색 지원
- **회의 녹화**: 데스크탑 화면 및 오디오 캡처, 실시간 전사, 녹화 내보내기
- **이벤트 처리**: 라이브 피드 모니터링, AI 분석 실행, 감지된 콘텐츠에 반응

## 빠른 시작

1. **라이브 스트림 연결** (RTSP/RTMP URL) 또는 캡처 세션에서 RTStream 가져오기

2. **수집 시작**하여 라이브 콘텐츠 녹화 시작

3. **AI 파이프라인 시작**으로 실시간 인덱싱 (오디오, 시각, 전사)

4. **WebSocket으로 이벤트 모니터링**하여 라이브 AI 결과 및 알림 수신

5. **수집 중지**

6. **동영상으로 내보내기** 하여 영구 저장 및 추가 처리

7. **녹화 검색**으로 특정 순간 찾기

## RTStream 소스

### RTSP/RTMP 스트림에서

라이브 동영상 소스에 직접 연결합니다:

```python
rtstream = coll.connect_rtstream(
    url="rtmp://your-stream-server/live/stream-key",
    name="My Live Stream",
)
```

### 캡처 세션에서

데스크탑 캡처에서 RTStream 가져오기 (마이크, 화면, 시스템 오디오):

```python
session = conn.get_capture_session(session_id)

mics = session.get_rtstream("mic")
displays = session.get_rtstream("screen")
system_audios = session.get_rtstream("system_audio")
```

캡처 세션 워크플로는 [capture.md](capture.md)를 참고하세요.

---

## 스크립트

| 스크립트 | 설명 |
|--------|-------------|
| `scripts/ws_listener.py` | 실시간 AI 결과를 위한 WebSocket 이벤트 리스너 |
