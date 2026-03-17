---
name: claude-api
description: Python 및 TypeScript를 위한 Anthropic Claude API 패턴. Messages API, streaming, tool use, vision, extended thinking, batches, prompt caching 및 Claude Agent SDK를 다룹니다. Claude API 또는 Anthropic SDK를 사용하여 애플리케이션을 구축할 때 사용합니다.
origin: ECC
---

# Claude API

Anthropic Claude API 및 SDK를 사용하여 애플리케이션을 구축합니다.

## 활성화 시점

- Claude API를 호출하는 애플리케이션을 빌드할 때
- 코드가 `anthropic` (Python) 또는 `@anthropic-ai/sdk` (TypeScript)를 import할 때
- 사용자가 Claude API 패턴, tool use, streaming 또는 vision에 대해 물어볼 때
- Claude Agent SDK로 agent 워크플로우를 구현할 때
- API 비용, 토큰 사용량 또는 latency를 최적화할 때

## 모델 선택

| Model | ID | 용도 |
|-------|-----|----------|
| Opus 4.1 | `claude-opus-4-1` | 복잡한 추론, 아키텍처, 연구 |
| Sonnet 4 | `claude-sonnet-4-0` | 균형 잡힌 코딩, 대부분의 개발 작업 |
| Haiku 3.5 | `claude-3-5-haiku-latest` | 빠른 응답, 대량 작업, 비용 민감 |

깊은 추론(Opus)이나 속도/비용 최적화(Haiku)가 필요한 작업이 아니라면 Sonnet 4를 기본으로 사용하세요. 프로덕션 환경에서는 alias보다 고정된 snapshot ID를 권장합니다.

## Python SDK

### 설치

```bash
pip install anthropic
```

### 기본 메시지

```python
import anthropic

client = anthropic.Anthropic()  # reads ANTHROPIC_API_KEY from env

message = client.messages.create(
    model="claude-sonnet-4-0",
    max_tokens=1024,
    messages=[
        {"role": "user", "content": "Explain async/await in Python"}
    ]
)
print(message.content[0].text)
```

### Streaming

```python
with client.messages.stream(
    model="claude-sonnet-4-0",
    max_tokens=1024,
    messages=[{"role": "user", "content": "Write a haiku about coding"}]
) as stream:
    for text in stream.text_stream:
        print(text, end="", flush=True)
```

### System Prompt

```python
message = client.messages.create(
    model="claude-sonnet-4-0",
    max_tokens=1024,
    system="You are a senior Python developer. Be concise.",
    messages=[{"role": "user", "content": "Review this function"}]
)
```

## TypeScript SDK

### 설치

```bash
npm install @anthropic-ai/sdk
```

### 기본 메시지

```typescript
import Anthropic from "@anthropic-ai/sdk";

const client = new Anthropic(); // reads ANTHROPIC_API_KEY from env

const message = await client.messages.create({
  model: "claude-sonnet-4-0",
  max_tokens: 1024,
  messages: [
    { role: "user", content: "Explain async/await in TypeScript" }
  ],
});
console.log(message.content[0].text);
```

### Streaming

```typescript
const stream = client.messages.stream({
  model: "claude-sonnet-4-0",
  max_tokens: 1024,
  messages: [{ role: "user", content: "Write a haiku" }],
});

for await (const event of stream) {
  if (event.type === "content_block_delta" && event.delta.type === "text_delta") {
    process.stdout.write(event.delta.text);
  }
}
```

## Tool Use

tool을 정의하고 Claude가 호출하게 하세요:

```python
tools = [
    {
        "name": "get_weather",
        "description": "Get current weather for a location",
        "input_schema": {
            "type": "object",
            "properties": {
                "location": {"type": "string", "description": "City name"},
                "unit": {"type": "string", "enum": ["celsius", "fahrenheit"]}
            },
            "required": ["location"]
        }
    }
]

message = client.messages.create(
    model="claude-sonnet-4-0",
    max_tokens=1024,
    tools=tools,
    messages=[{"role": "user", "content": "What's the weather in SF?"}]
)

# Handle tool use response
for block in message.content:
    if block.type == "tool_use":
        # Execute the tool with block.input
        result = get_weather(**block.input)
        # Send result back
        follow_up = client.messages.create(
            model="claude-sonnet-4-0",
            max_tokens=1024,
            tools=tools,
            messages=[
                {"role": "user", "content": "What's the weather in SF?"},
                {"role": "assistant", "content": message.content},
                {"role": "user", "content": [
                    {"type": "tool_result", "tool_use_id": block.id, "content": str(result)}
                ]}
            ]
        )
```

## Vision

분석을 위해 이미지를 전송하세요:

```python
import base64

with open("diagram.png", "rb") as f:
    image_data = base64.standard_b64encode(f.read()).decode("utf-8")

message = client.messages.create(
    model="claude-sonnet-4-0",
    max_tokens=1024,
    messages=[{
        "role": "user",
        "content": [
            {"type": "image", "source": {"type": "base64", "media_type": "image/png", "data": image_data}},
            {"type": "text", "text": "Describe this diagram"}
        ]
    }]
)
```

## Extended Thinking

복잡한 추론 작업을 위해:

```python
message = client.messages.create(
    model="claude-sonnet-4-0",
    max_tokens=16000,
    thinking={
        "type": "enabled",
        "budget_tokens": 10000
    },
    messages=[{"role": "user", "content": "Solve this math problem step by step..."}]
)

for block in message.content:
    if block.type == "thinking":
        print(f"Thinking: {block.thinking}")
    elif block.type == "text":
        print(f"Answer: {block.text}")
```

## Prompt Caching

비용을 절감하기 위해 대규모 system prompt 또는 context를 캐싱하세요:

```python
message = client.messages.create(
    model="claude-sonnet-4-0",
    max_tokens=1024,
    system=[
        {"type": "text", "text": large_system_prompt, "cache_control": {"type": "ephemeral"}}
    ],
    messages=[{"role": "user", "content": "Question about the cached context"}]
)
# Check cache usage
print(f"Cache read: {message.usage.cache_read_input_tokens}")
print(f"Cache creation: {message.usage.cache_creation_input_tokens}")
```

## Batches API

50% 비용 절감으로 대량의 데이터를 비동기 처리하세요:

```python
import time

batch = client.messages.batches.create(
    requests=[
        {
            "custom_id": f"request-{i}",
            "params": {
                "model": "claude-sonnet-4-0",
                "max_tokens": 1024,
                "messages": [{"role": "user", "content": prompt}]
            }
        }
        for i, prompt in enumerate(prompts)
    ]
)

# Poll for completion
while True:
    status = client.messages.batches.retrieve(batch.id)
    if status.processing_status == "ended":
        break
    time.sleep(30)

# Get results
for result in client.messages.batches.results(batch.id):
    print(result.result.message.content[0].text)
```

## Claude Agent SDK

다단계 agent를 구축하세요:

```python
# Note: Agent SDK API surface may change — check official docs
import anthropic

# Define tools as functions
tools = [{
    "name": "search_codebase",
    "description": "Search the codebase for relevant code",
    "input_schema": {
        "type": "object",
        "properties": {"query": {"type": "string"}},
        "required": ["query"]
    }
}]

# Run an agentic loop with tool use
client = anthropic.Anthropic()
messages = [{"role": "user", "content": "Review the auth module for security issues"}]

while True:
    response = client.messages.create(
        model="claude-sonnet-4-0",
        max_tokens=4096,
        tools=tools,
        messages=messages,
    )
    if response.stop_reason == "end_turn":
        break
    # Handle tool calls and continue the loop
    messages.append({"role": "assistant", "content": response.content})
    # ... execute tools and append tool_result messages
```

## 비용 최적화

| 전략 | 절감 | 사용 시점 |
|----------|---------|-------------|
| Prompt caching | 캐싱된 토큰에 대해 최대 90% | 반복되는 system prompt 또는 context |
| Batches API | 50% | 시간에 민감하지 않은 대량 처리 |
| Haiku instead of Sonnet | 약 75% | 간단한 작업, 분류, 추출 |
| Shorter max_tokens | 가변적 | 출력이 짧을 것으로 예상될 때 |
| Streaming | 없음 (동일 비용) | 더 나은 UX, 동일 가격 |

## Error Handling

```python
import time

from anthropic import APIError, RateLimitError, APIConnectionError

try:
    message = client.messages.create(...)
except RateLimitError:
    # Back off and retry
    time.sleep(60)
except APIConnectionError:
    # Network issue, retry with backoff
    pass
except APIError as e:
    print(f"API error {e.status_code}: {e.message}")
```

## 환경 설정

```bash
# Required
export ANTHROPIC_API_KEY="your-api-key-here"

# Optional: set default model
export ANTHROPIC_MODEL="claude-sonnet-4-0"
```

API key를 하드코딩하지 마세요. 항상 환경 변수를 사용하세요.
