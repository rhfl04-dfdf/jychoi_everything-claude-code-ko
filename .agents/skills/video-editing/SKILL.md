---
name: video-editing
description: 실제 푸티지를 컷팅, 구조화 및 보강하기 위한 AI 보조 비디오 편집 workflow입니다. 원본 캡처부터 FFmpeg, Remotion, ElevenLabs, fal.ai를 거쳐 Descript 또는 CapCut에서의 최종 마무리까지 전체 pipeline을 다룹니다. 사용자가 비디오를 편집하거나, 푸티지를 자르거나, 브이로그를 만들거나, 비디오 콘텐츠를 제작하고자 할 때 사용합니다.
origin: ECC
---

# 비디오 편집

실제 푸티지를 위한 AI 보조 편집입니다. 프롬프트로부터의 생성이 아닙니다. 기존 비디오를 빠르게 편집합니다.

## 활성화 시점

- 사용자가 비디오 푸티지를 편집, 컷팅 또는 구조화하고자 할 때
- 긴 녹화본을 숏폼 콘텐츠로 변환할 때
- 원본 캡처로부터 브이로그, 튜토리얼 또는 데모 비디오를 제작할 때
- 기존 비디오에 오버레이, 자막, 음악 또는 보이스오버를 추가할 때
- 다양한 플랫폼(YouTube, TikTok, Instagram)에 맞춰 비디오 reframing을 할 때
- 사용자가 "비디오 편집해줘", "이 푸티지 잘라줘", "브이로그 만들어줘" 또는 "비디오 workflow"라고 말할 때

## 핵심 논지

AI 비디오 편집은 전체 비디오를 만들어달라고 요청하는 대신 실제 푸티지를 압축, 구조화 및 보강하는 데 사용하기 시작할 때 유용합니다. 가치는 생성이 아니라 압축에 있습니다.

## Pipeline

```
Screen Studio / raw footage
  → Claude / Codex
  → FFmpeg
  → Remotion
  → ElevenLabs / fal.ai
  → Descript or CapCut
```

각 레이어는 특정 작업을 담당합니다. 레이어를 건너뛰지 마세요. 하나의 툴이 모든 것을 하도록 만들려고 하지 마세요.

## 레이어 1: 캡처 (Screen Studio / 원본 푸티지)

소스 자료 수집:
- **Screen Studio**: 앱 데모, 코딩 세션, 브라우저 workflow를 위한 정교한 화면 녹화
- **원본 카메라 푸티지**: 브이로그 푸티지, 인터뷰, 이벤트 녹화
- **VideoDB를 통한 데스크톱 캡처**: 실시간 컨텍스트를 포함한 세션 녹화 (`videodb` skill 참고)

출력: 정리를 위한 준비가 된 원본 파일들.

## 레이어 2: 조직화 (Claude / Codex)

Claude Code 또는 Codex를 사용하여 다음을 수행합니다:
- **트랜스크립트 작성 및 레이벨링**: 트랜스크립트 생성, 주제 및 테마 식별
- **구조 계획**: 남길 부분과 자를 부분 결정, 작동하는 순서 결정
- **불필요한 섹션 식별**: 일시 중지, 옆길로 새는 부분, 반복된 테이크 찾기
- **편집 결정 리스트 생성**: 컷을 위한 타임스탬프, 유지할 세그먼트
- **FFmpeg 및 Remotion 코드 스캐폴딩**: 명령 및 composition 생성

```
Example prompt:
"Here's the transcript of a 4-hour recording. Identify the 8 strongest segments
for a 24-minute vlog. Give me FFmpeg cut commands for each segment."
```

이 레이어는 최종적인 창의적 취향이 아니라 구조에 관한 것입니다.

## 레이어 3: 결정론적 컷 (FFmpeg)

FFmpeg는 지루하지만 중요한 작업인 분할, 트리밍, 연결 및 전처리를 처리합니다.

### 타임스탬프로 세그먼트 추출

```bash
ffmpeg -i raw.mp4 -ss 00:12:30 -to 00:15:45 -c copy segment_01.mp4
```

### 편집 결정 리스트에서 일괄 컷팅

```bash
#!/bin/bash
# cuts.txt: start,end,label
while IFS=, read -r start end label; do
  ffmpeg -i raw.mp4 -ss "$start" -to "$end" -c copy "segments/${label}.mp4"
done < cuts.txt
```

### 세그먼트 연결

```bash
# Create file list
for f in segments/*.mp4; do echo "file '$f'"; done > concat.txt
ffmpeg -f concat -safe 0 -i concat.txt -c copy assembled.mp4
```

### 더 빠른 편집을 위해 proxy 생성

```bash
ffmpeg -i raw.mp4 -vf "scale=960:-2" -c:v libx264 -preset ultrafast -crf 28 proxy.mp4
```

### 트랜스크립트 작성을 위해 오디오 추출

```bash
ffmpeg -i raw.mp4 -vn -acodec pcm_s16le -ar 16000 audio.wav
```

### 오디오 레벨 정규화

```bash
ffmpeg -i segment.mp4 -af loudnorm=I=-16:TP=-1.5:LRA=11 -c:v copy normalized.mp4
```

## 레이어 4: 프로그래밍 가능한 Composition (Remotion)

Remotion은 편집 문제를 구성 가능한 코드로 전환합니다. 기존 편집기에서 번거로운 작업에 사용하세요:

### Remotion을 사용할 때

- 오버레이: 텍스트, 이미지, 브랜딩, 하단 자막(lower thirds)
- 데이터 시각화: 차트, 통계, 애니메이션 숫자
- 모션 그래픽: 트랜지션, 설명 애니메이션
- 구성 가능한 씬: 비디오 전반에 걸쳐 재사용 가능한 템플릿
- 제품 데모: 주석이 달린 스크린샷, UI 하이라이트

### 기본 Remotion composition

```tsx
import { AbsoluteFill, Sequence, Video, useCurrentFrame } from "remotion";

export const VlogComposition: React.FC = () => {
  const frame = useCurrentFrame();

  return (
    <AbsoluteFill>
      {/* Main footage */}
      <Sequence from={0} durationInFrames={300}>
        <Video src="/segments/intro.mp4" />
      </Sequence>

      {/* Title overlay */}
      <Sequence from={30} durationInFrames={90}>
        <AbsoluteFill style={{
          justifyContent: "center",
          alignItems: "center",
        }}>
          <h1 style={{
            fontSize: 72,
            color: "white",
            textShadow: "2px 2px 8px rgba(0,0,0,0.8)",
          }}>
            The AI Editing Stack
          </h1>
        </AbsoluteFill>
      </Sequence>

      {/* Next segment */}
      <Sequence from={300} durationInFrames={450}>
        <Video src="/segments/demo.mp4" />
      </Sequence>
    </AbsoluteFill>
  );
};
```

### 렌더링 출력

```bash
npx remotion render src/index.ts VlogComposition output.mp4
```

상세한 패턴 및 API reference는 [Remotion 문서](https://www.remotion.dev/docs)를 확인하세요.

## 레이어 5: 생성된 에셋 (ElevenLabs / fal.ai)

필요한 것만 생성하세요. 전체 비디오를 생성하지 마세요.

### ElevenLabs를 사용한 보이스오버

```python
import os
import requests

resp = requests.post(
    f"https://api.elevenlabs.io/v1/text-to-speech/{voice_id}",
    headers={
        "xi-api-key": os.environ["ELEVENLABS_API_KEY"],
        "Content-Type": "application/json"
    },
    json={
        "text": "Your narration text here",
        "model_id": "eleven_turbo_v2_5",
        "voice_settings": {"stability": 0.5, "similarity_boost": 0.75}
    }
)
with open("voiceover.mp3", "wb") as f:
    f.write(resp.content)
```

### fal.ai를 사용한 음악 및 SFX

`fal-ai-media` skill을 다음 용도로 사용하세요:
- 배경 음악 생성
- 사운드 효과 (비디오-오디오 변환을 위한 ThinkSound 모델)
- 트랜지션 사운드

### fal.ai를 사용한 생성된 비주얼

존재하지 않는 인서트 샷, 썸네일 또는 b-roll에 사용하세요:
```
generate(model_name: "fal-ai/nano-banana-pro", input: {
  "prompt": "professional thumbnail for tech vlog, dark background, code on screen",
  "image_size": "landscape_16_9"
})
```

### VideoDB 생성형 오디오

VideoDB가 구성된 경우:
```python
voiceover = coll.generate_voice(text="Narration here", voice="alloy")
music = coll.generate_music(prompt="lo-fi background for coding vlog", duration=120)
sfx = coll.generate_sound_effect(prompt="subtle whoosh transition")
```

## 레이어 6: 최종 마무리 (Descript / CapCut)

마지막 레이어는 인간입니다. 기존 편집기를 다음 용도로 사용하세요:
- **페이싱**: 너무 빠르거나 느리게 느껴지는 컷 조정
- **캡션**: 자동 생성 후 수동으로 정리
- **컬러 그레이딩**: 기본 보정 및 분위기
- **최종 오디오 믹싱**: 목소리, 음악 및 SFX 레벨 균형 맞추기
- **내보내기**: 플랫폼별 형식 및 품질 설정

이곳은 취향이 살아있는 곳입니다. AI는 반복적인 작업을 제거합니다. 당신이 최종 결정을 내립니다.

## 소셜 미디어 Reframing

플랫폼마다 다른 종횡비가 필요합니다:

| 플랫폼 | 종횡비 | 해상도 |
|----------|-------------|------------|
| YouTube | 16:9 | 1920x1080 |
| TikTok / Reels | 9:16 | 1080x1920 |
| Instagram Feed | 1:1 | 1080x1080 |
| X / Twitter | 16:9 또는 1:1 | 1280x720 또는 720x720 |

### FFmpeg로 리프레임

```bash
# 16:9 to 9:16 (center crop)
ffmpeg -i input.mp4 -vf "crop=ih*9/16:ih,scale=1080:1920" vertical.mp4

# 16:9 to 1:1 (center crop)
ffmpeg -i input.mp4 -vf "crop=ih:ih,scale=1080:1080" square.mp4
```

### VideoDB로 리프레임

```python
# Smart reframe (AI-guided subject tracking)
reframed = video.reframe(start=0, end=60, target="vertical", mode=ReframeMode.smart)
```

## 씬 감지 및 자동 컷

### FFmpeg 씬 감지

```bash
# Detect scene changes (threshold 0.3 = moderate sensitivity)
ffmpeg -i input.mp4 -vf "select='gt(scene,0.3)',showinfo" -vsync vfr -f null - 2>&1 | grep showinfo
```

### 자동 컷을 위한 무음 감지

```bash
# Find silent segments (useful for cutting dead air)
ffmpeg -i input.mp4 -af silencedetect=noise=-30dB:d=2 -f null - 2>&1 | grep silence
```

### 하이라이트 추출

트랜스크립트 + 씬 타임스탬프 분석을 위해 Claude를 사용하세요:
```
"Given this transcript with timestamps and these scene change points,
identify the 5 most engaging 30-second clips for social media."
```

## 각 툴의 강점

| 툴 | 강점 | 약점 |
|------|----------|----------|
| Claude / Codex | 조직화, 계획, 코드 생성 | 창의적 취향 레이어가 아님 |
| FFmpeg | 결정론적 컷, 일괄 처리, 형식 변환 | 시각적 편집 UI 없음 |
| Remotion | 프로그래밍 가능한 오버레이, 구성 가능한 씬, 재사용 가능한 템플릿 | 개발자가 아닌 경우의 학습 곡선 |
| Screen Studio | 즉각적으로 정교한 화면 녹화 가능 | 화면 캡처만 가능 |
| ElevenLabs | 목소리, 내레이션, 음악, SFX | workflow의 중심이 아님 |
| Descript / CapCut | 최종 페이싱, 캡션, 마무리 | 수동적이며 자동화할 수 없음 |

## 주요 원칙

1. **생성하지 말고 편집하세요.** 이 workflow는 프롬프트에서 생성하는 것이 아니라 실제 푸티지를 자르는 용도입니다.
2. **스타일보다 구조입니다.** 시각적인 것을 건드리기 전에 레이어 2에서 스토리를 제대로 잡으세요.
3. **FFmpeg는 중추입니다.** 지루하지만 중요합니다. 긴 푸티지가 관리 가능해지는 단계입니다.
4. **반복성을 위한 Remotion.** 한 번 이상 수행할 작업이라면 Remotion 컴포넌트로 만드세요.
5. **선택적으로 생성하세요.** 모든 것이 아니라 존재하지 않는 에셋에 대해서만 AI 생성을 사용하세요.
6. **취향은 마지막 레이어입니다.** AI는 반복적인 작업을 처리합니다. 당신이 최종적인 창의적 결정을 내립니다.

## 관련 Skills

- `fal-ai-media` — AI 이미지, 비디오 및 오디오 생성
- `videodb` — 서버 측 비디오 처리, 인덱싱 및 스트리밍
- `content-engine` — 플랫폼 네이티브 콘텐츠 배포
