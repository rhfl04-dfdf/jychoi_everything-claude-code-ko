# 검색 및 인덱싱 가이드

검색을 통해 자연어 쿼리, 정확한 키워드, 또는 시각적 장면 설명으로 동영상 내 특정 순간을 찾을 수 있습니다.

## 사전 조건

동영상은 검색 전에 **인덱싱**되어야 합니다. 인덱싱은 동영상 유형당 한 번만 수행하면 됩니다.

## 인덱싱

### 음성 단어 인덱스

시맨틱 및 키워드 검색을 위해 동영상의 전사된 음성 콘텐츠를 인덱싱합니다:

```python
video = coll.get_video(video_id)

# force=True makes indexing idempotent — skips if already indexed
video.index_spoken_words(force=True)
```

오디오 트랙을 전사하고 음성 콘텐츠에 대한 검색 가능한 인덱스를 구축합니다. 시맨틱 검색과 키워드 검색에 필요합니다.

**파라미터:**

| 파라미터 | 타입 | 기본값 | 설명 |
|-----------|------|---------|-------------|
| `language_code` | `str\|None` | `None` | 동영상의 언어 코드 |
| `segmentation_type` | `SegmentationType` | `SegmentationType.sentence` | 분할 타입 (`sentence` 또는 `llm`) |
| `force` | `bool` | `False` | 이미 인덱싱된 경우 건너뛰려면 `True`로 설정 ("already exists" 오류 방지) |
| `callback_url` | `str\|None` | `None` | 비동기 알림용 웹훅 URL |

### 장면 인덱스

장면의 AI 설명을 생성하여 시각적 콘텐츠를 인덱싱합니다. 음성 단어 인덱싱과 마찬가지로, 장면 인덱스가 이미 존재하면 오류를 발생시킵니다. 오류 메시지에서 기존 `scene_index_id`를 추출하세요.

```python
import re
from videodb import SceneExtractionType

try:
    scene_index_id = video.index_scenes(
        extraction_type=SceneExtractionType.shot_based,
        prompt="Describe the visual content, objects, actions, and setting in this scene.",
    )
except Exception as e:
    match = re.search(r"id\s+([a-f0-9]+)", str(e))
    if match:
        scene_index_id = match.group(1)
    else:
        raise
```

**추출 타입:**

| 타입 | 설명 | 적합 |
|------|-------------|----------|
| `SceneExtractionType.shot_based` | 시각적 샷 경계에서 분할 | 일반 목적, 액션 콘텐츠 |
| `SceneExtractionType.time_based` | 고정 간격으로 분할 | 균일 샘플링, 긴 정적 콘텐츠 |
| `SceneExtractionType.transcript` | 전사 구간 기반으로 분할 | 음성 기반 장면 경계 |

**`time_based` 파라미터:**

```python
video.index_scenes(
    extraction_type=SceneExtractionType.time_based,
    extraction_config={"time": 5, "select_frames": ["first", "last"]},
    prompt="Describe what is happening in this scene.",
)
```

## 검색 타입

### 시맨틱 검색

음성 콘텐츠에 대해 자연어 쿼리를 매칭합니다:

```python
from videodb import SearchType

results = video.search(
    query="explaining the benefits of machine learning",
    search_type=SearchType.semantic,
)
```

쿼리와 시맨틱으로 일치하는 음성 콘텐츠의 순위가 매겨진 구간을 반환합니다.

### 키워드 검색

전사된 음성에서 정확한 용어를 매칭합니다:

```python
results = video.search(
    query="artificial intelligence",
    search_type=SearchType.keyword,
)
```

정확한 키워드나 구문이 포함된 구간을 반환합니다.

### 장면 검색

인덱싱된 장면 설명에 대해 시각적 콘텐츠 쿼리를 매칭합니다. 사전에 `index_scenes()` 호출이 필요합니다.

`index_scenes()`는 `scene_index_id`를 반환합니다. 특정 장면 인덱스를 대상으로 하려면 `video.search()`에 전달하세요 (특히 동영상에 여러 장면 인덱스가 있을 때 중요합니다):

```python
from videodb import SearchType, IndexType
from videodb.exceptions import InvalidRequestError

# Search using semantic search against the scene index.
# Use score_threshold to filter low-relevance noise (recommended: 0.3+).
try:
    results = video.search(
        query="person writing on a whiteboard",
        search_type=SearchType.semantic,
        index_type=IndexType.scene,
        scene_index_id=scene_index_id,
        score_threshold=0.3,
    )
    shots = results.get_shots()
except InvalidRequestError as e:
    if "No results found" in str(e):
        shots = []
    else:
        raise
```

**중요 참고사항:**

- `index_type=IndexType.scene`과 함께 `SearchType.semantic`을 사용하세요 — 이것이 가장 신뢰할 수 있는 조합이며 모든 플랜에서 작동합니다.
- `SearchType.scene`은 존재하지만 모든 플랜 (예: 무료 티어)에서 이용 가능하지 않을 수 있습니다. `IndexType.scene`과 함께 `SearchType.semantic`을 선호하세요.
- `scene_index_id` 파라미터는 선택사항입니다. 생략하면 동영상의 모든 장면 인덱스에서 검색이 실행됩니다. 특정 인덱스를 대상으로 하려면 전달하세요.
- 동영상당 여러 장면 인덱스를 생성 (다른 프롬프트나 추출 타입 사용)하고 `scene_index_id`로 독립적으로 검색할 수 있습니다.

### 메타데이터 필터링을 통한 장면 검색

커스텀 메타데이터로 장면을 인덱싱할 때 시맨틱 검색과 메타데이터 필터를 결합할 수 있습니다:

```python
from videodb import SearchType, IndexType

results = video.search(
    query="a skillful chasing scene",
    search_type=SearchType.semantic,
    index_type=IndexType.scene,
    scene_index_id=scene_index_id,
    filter=[{"camera_view": "road_ahead"}, {"action_type": "chasing"}],
)
```

커스텀 메타데이터 인덱싱 및 필터링 검색의 전체 예시는 [scene_level_metadata_indexing cookbook](https://github.com/video-db/videodb-cookbook/blob/main/quickstart/scene_level_metadata_indexing.ipynb)을 참고하세요.

## 결과 처리

### Shot 가져오기

개별 결과 구간에 접근합니다:

```python
results = video.search("your query")

for shot in results.get_shots():
    print(f"Video: {shot.video_id}")
    print(f"Start: {shot.start:.2f}s")
    print(f"End: {shot.end:.2f}s")
    print(f"Text: {shot.text}")
    print("---")
```

### 컴파일된 결과 재생

모든 매칭 구간을 단일 컴파일 동영상으로 스트리밍합니다:

```python
results = video.search("your query")
stream_url = results.compile()
results.play()  # opens compiled stream in browser
```

### 클립 추출

특정 결과 구간을 다운로드하거나 스트리밍합니다:

```python
for shot in results.get_shots():
    stream_url = shot.generate_stream()
    print(f"Clip: {stream_url}")
```

## 컬렉션 전체 검색

컬렉션의 모든 동영상에서 검색합니다:

```python
coll = conn.get_collection()

# Search across all videos in the collection
results = coll.search(
    query="product demo",
    search_type=SearchType.semantic,
)

for shot in results.get_shots():
    print(f"Video: {shot.video_id} [{shot.start:.1f}s - {shot.end:.1f}s]")
```

> **참고:** 컬렉션 수준 검색은 `SearchType.semantic`만 지원합니다. `coll.search()`에 `SearchType.keyword` 또는 `SearchType.scene`을 사용하면 `NotImplementedError`가 발생합니다. 키워드 또는 장면 검색은 개별 동영상에서 `video.search()`를 사용하세요.

## 검색 + 컴파일

인덱싱, 검색, 매칭 구간을 단일 재생 가능한 스트림으로 컴파일합니다:

```python
video.index_spoken_words(force=True)
results = video.search(query="your query", search_type=SearchType.semantic)
stream_url = results.compile()
print(stream_url)
```

## 팁

- **한 번 인덱싱, 여러 번 검색**: 인덱싱이 비용이 많이 드는 작업입니다. 한 번 인덱싱하면 검색은 빠릅니다.
- **인덱스 타입 결합**: 동영상에 대해 모든 검색 타입을 활성화하려면 음성 단어와 장면 모두 인덱싱하세요.
- **쿼리 개선**: 시맨틱 검색은 단일 키워드보다 설명적인 자연어 구문에서 가장 잘 작동합니다.
- **정확도를 위해 키워드 검색 사용**: 정확한 용어 매칭이 필요할 때 키워드 검색으로 시맨틱 드리프트를 방지합니다.
- **"No results found" 처리**: `video.search()`는 일치하는 결과가 없으면 `InvalidRequestError`를 발생시킵니다. 항상 검색 호출을 try/except로 감싸고 `"No results found"`를 빈 결과 집합으로 처리하세요.
- **장면 검색 노이즈 필터링**: 시맨틱 장면 검색은 모호한 쿼리에서 낮은 관련성 결과를 반환할 수 있습니다. 노이즈를 필터링하려면 `score_threshold=0.3` (이상)을 사용하세요.
- **멱등적 인덱싱**: 안전하게 재인덱싱하려면 `index_spoken_words(force=True)` 사용. `index_scenes()`에는 `force` 파라미터가 없습니다 — try/except로 감싸고 `re.search(r"id\s+([a-f0-9]+)", str(e))`로 오류 메시지에서 기존 `scene_index_id`를 추출하세요.
