---
name: fal-ai-media
description: fal.ai MCP를 통한 통합 미디어 생성 — 이미지, 비디오, 오디오. 텍스트-이미지 (Nano Banana), 텍스트/이미지-비디오 (Seedance, Kling, Veo 3), 텍스트-음성 (CSM-1B), 비디오-오디오 (ThinkSound)를 지원합니다. 사용자가 AI로 이미지, 비디오, 오디오를 생성하고 싶을 때 사용합니다.
origin: ECC
---

# fal.ai 미디어 생성

MCP를 통해 fal.ai 모델을 사용하여 이미지, 비디오, 오디오를 생성합니다.

## 활성화 시점

- 사용자가 텍스트 프롬프트로 이미지를 생성하고 싶을 때
- 텍스트 또는 이미지로 비디오를 만들 때
- 음성, 음악, 효과음을 생성할 때
- 모든 미디어 생성 작업
- 사용자가 "이미지 생성", "비디오 만들기", "텍스트-음성", "썸네일 만들기" 등과 유사하게 말할 때

## MCP 요구사항

fal.ai MCP 서버가 설정되어 있어야 합니다. `~/.claude.json`에 추가:

```json
"fal-ai": {
  "command": "npx",
  "args": ["-y", "fal-ai-mcp-server"],
  "env": { "FAL_KEY": "YOUR_FAL_KEY_HERE" }
}
```

API 키는 [fal.ai](https://fal.ai)에서 발급받으세요.

## MCP 도구

fal.ai MCP가 제공하는 도구:
- `search` — 키워드로 사용 가능한 모델 검색
- `find` — 모델 세부 정보와 파라미터 확인
- `generate` — 파라미터로 모델 실행
- `result` — 비동기 생성 상태 확인
- `status` — 작업 상태 확인
- `cancel` — 실행 중인 작업 취소
- `estimate_cost` — 생성 비용 추정
- `models` — 인기 모델 목록
- `upload` — 입력으로 사용할 파일 업로드

---

## 이미지 생성

### Nano Banana 2 (빠른 속도)
최적 용도: 빠른 반복, 초안 작업, 텍스트-이미지, 이미지 편집.

```
generate(
  model_name: "fal-ai/nano-banana-2",
  input: {
    "prompt": "a futuristic cityscape at sunset, cyberpunk style",
    "image_size": "landscape_16_9",
    "num_images": 1,
    "seed": 42
  }
)
```

### Nano Banana Pro (고화질)
최적 용도: 프로덕션 이미지, 사실주의, 타이포그래피, 상세한 프롬프트.

```
generate(
  model_name: "fal-ai/nano-banana-pro",
  input: {
    "prompt": "professional product photo of wireless headphones on marble surface, studio lighting",
    "image_size": "square",
    "num_images": 1,
    "guidance_scale": 7.5
  }
)
```

### 공통 이미지 파라미터

| 파라미터 | 타입 | 옵션 | 비고 |
|-------|------|---------|-------|
| `prompt` | string | 필수 | 원하는 내용 설명 |
| `image_size` | string | `square`, `portrait_4_3`, `landscape_16_9`, `portrait_16_9`, `landscape_4_3` | 화면 비율 |
| `num_images` | number | 1-4 | 생성할 이미지 수 |
| `seed` | number | 임의 정수 | 재현성 |
| `guidance_scale` | number | 1-20 | 프롬프트를 얼마나 따를지 (높을수록 문자 그대로 해석) |

### 이미지 편집
인페인팅, 아웃페인팅, 스타일 전환에 입력 이미지와 함께 Nano Banana 2 사용:

```
# First upload the source image
upload(file_path: "/path/to/image.png")

# Then generate with image input
generate(
  model_name: "fal-ai/nano-banana-2",
  input: {
    "prompt": "same scene but in watercolor style",
    "image_url": "<uploaded_url>",
    "image_size": "landscape_16_9"
  }
)
```

---

## 비디오 생성

### Seedance 1.0 Pro (ByteDance)
최적 용도: 텍스트-비디오, 높은 모션 품질의 이미지-비디오.

```
generate(
  model_name: "fal-ai/seedance-1-0-pro",
  input: {
    "prompt": "a drone flyover of a mountain lake at golden hour, cinematic",
    "duration": "5s",
    "aspect_ratio": "16:9",
    "seed": 42
  }
)
```

### Kling Video v3 Pro
최적 용도: 기본 오디오 생성이 포함된 텍스트/이미지-비디오.

```
generate(
  model_name: "fal-ai/kling-video/v3/pro",
  input: {
    "prompt": "ocean waves crashing on a rocky coast, dramatic clouds",
    "duration": "5s",
    "aspect_ratio": "16:9"
  }
)
```

### Veo 3 (Google DeepMind)
최적 용도: 생성된 사운드가 포함된 비디오, 높은 시각적 품질.

```
generate(
  model_name: "fal-ai/veo-3",
  input: {
    "prompt": "a bustling Tokyo street market at night, neon signs, crowd noise",
    "aspect_ratio": "16:9"
  }
)
```

### 이미지-비디오
기존 이미지에서 시작:

```
generate(
  model_name: "fal-ai/seedance-1-0-pro",
  input: {
    "prompt": "camera slowly zooms out, gentle wind moves the trees",
    "image_url": "<uploaded_image_url>",
    "duration": "5s"
  }
)
```

### 비디오 파라미터

| 파라미터 | 타입 | 옵션 | 비고 |
|-------|------|---------|-------|
| `prompt` | string | 필수 | 비디오 내용 설명 |
| `duration` | string | `"5s"`, `"10s"` | 비디오 길이 |
| `aspect_ratio` | string | `"16:9"`, `"9:16"`, `"1:1"` | 프레임 비율 |
| `seed` | number | 임의 정수 | 재현성 |
| `image_url` | string | URL | 이미지-비디오의 소스 이미지 |

---

## 오디오 생성

### CSM-1B (대화형 음성)
자연스러운 대화체 품질의 텍스트-음성 변환.

```
generate(
  model_name: "fal-ai/csm-1b",
  input: {
    "text": "Hello, welcome to the demo. Let me show you how this works.",
    "speaker_id": 0
  }
)
```

### ThinkSound (비디오-오디오)
비디오 콘텐츠에서 어울리는 오디오 생성.

```
generate(
  model_name: "fal-ai/thinksound",
  input: {
    "video_url": "<video_url>",
    "prompt": "ambient forest sounds with birds chirping"
  }
)
```

### ElevenLabs (API 직접 사용, MCP 불필요)
전문적인 음성 합성에는 ElevenLabs를 직접 사용:

```python
import os
import requests

resp = requests.post(
    "https://api.elevenlabs.io/v1/text-to-speech/<voice_id>",
    headers={
        "xi-api-key": os.environ["ELEVENLABS_API_KEY"],
        "Content-Type": "application/json"
    },
    json={
        "text": "Your text here",
        "model_id": "eleven_turbo_v2_5",
        "voice_settings": {"stability": 0.5, "similarity_boost": 0.75}
    }
)
with open("output.mp3", "wb") as f:
    f.write(resp.content)
```

### VideoDB 생성형 오디오
VideoDB가 설정된 경우 생성형 오디오 사용:

```python
# Voice generation
audio = coll.generate_voice(text="Your narration here", voice="alloy")

# Music generation
music = coll.generate_music(prompt="upbeat electronic background music", duration=30)

# Sound effects
sfx = coll.generate_sound_effect(prompt="thunder crack followed by rain")
```

---

## 비용 추정

생성 전 예상 비용 확인:

```
estimate_cost(model_name: "fal-ai/nano-banana-pro", input: {...})
```

## 모델 탐색

특정 작업에 맞는 모델 찾기:

```
search(query: "text to video")
find(model_name: "fal-ai/seedance-1-0-pro")
models()
```

## 팁

- 프롬프트 반복 시 재현 가능한 결과를 위해 `seed` 사용
- 프롬프트 반복에는 저렴한 모델 (Nano Banana 2)로 시작하고, 최종 결과물에는 Pro로 전환
- 비디오는 프롬프트를 설명적이지만 간결하게 유지 — 모션과 장면에 집중
- 이미지-비디오가 순수 텍스트-비디오보다 더 제어된 결과 생성
- 비용이 높은 비디오 생성 전 `estimate_cost` 확인

## 관련 스킬

- `videodb` — 비디오 처리, 편집, 스트리밍
- `video-editing` — AI 기반 비디오 편집 워크플로우
- `content-engine` — 소셜 플랫폼 콘텐츠 제작
