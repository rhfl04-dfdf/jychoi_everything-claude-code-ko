---
name: content-hash-cache-pattern
description: SHA-256 content hash를 사용하여 비용이 많이 드는 파일 처리 결과를 캐싱합니다 — 경로에 독립적이며, 자동 무효화 및 service layer 분리가 특징입니다.
origin: ECC
---

# Content-Hash 파일 캐시 패턴

SHA-256 content hash를 캐시 키로 사용하여 비용이 많이 드는 파일 처리 결과(PDF 파싱, 텍스트 추출, 이미지 분석)를 캐싱합니다. 경로 기반 캐싱과 달리, 이 방식은 파일 이동/이름 변경에도 유지되며 내용이 변경되면 자동으로 무효화됩니다.

## 활성화 시점

- 파일 처리 파이프라인 구축 시 (PDF, 이미지, 텍스트 추출)
- 처리 비용이 높고 동일한 파일이 반복적으로 처리되는 경우
- --cache/--no-cache CLI 옵션이 필요한 경우
- 기존의 순수 함수를 수정하지 않고 캐싱을 추가하고 싶은 경우

## 핵심 패턴

### 1. Content-Hash 기반 캐시 키

파일 경로가 아닌 파일 내용(content)을 캐시 키로 사용합니다:

```python
import hashlib
from pathlib import Path

_HASH_CHUNK_SIZE = 65536  # 64KB chunks for large files

def compute_file_hash(path: Path) -> str:
    """SHA-256 of file contents (chunked for large files)."""
    if not path.is_file():
        raise FileNotFoundError(f"File not found: {path}")
    sha256 = hashlib.sha256()
    with open(path, "rb") as f:
        while True:
            chunk = f.read(_HASH_CHUNK_SIZE)
            if not chunk:
                break
            sha256.update(chunk)
    return sha256.hexdigest()
```

**왜 content hash인가?** 파일 이름 변경/이동 = 캐시 히트. 내용 변경 = 자동 무효화. 인덱스 파일 불필요.

### 2. 캐시 엔트리를 위한 Frozen Dataclass

```python
from dataclasses import dataclass

@dataclass(frozen=True, slots=True)
class CacheEntry:
    file_hash: str
    source_path: str
    document: ExtractedDocument  # The cached result
```

### 3. 파일 기반 캐시 스토리지

각 캐시 엔트리는 `{hash}.json`으로 저장됩니다 — 해시를 통한 O(1) 조회, 인덱스 파일 불필요.

```python
import json
from typing import Any

def write_cache(cache_dir: Path, entry: CacheEntry) -> None:
    cache_dir.mkdir(parents=True, exist_ok=True)
    cache_file = cache_dir / f"{entry.file_hash}.json"
    data = serialize_entry(entry)
    cache_file.write_text(json.dumps(data, ensure_ascii=False), encoding="utf-8")

def read_cache(cache_dir: Path, file_hash: str) -> CacheEntry | None:
    cache_file = cache_dir / f"{file_hash}.json"
    if not cache_file.is_file():
        return None
    try:
        raw = cache_file.read_text(encoding="utf-8")
        data = json.loads(raw)
        return deserialize_entry(data)
    except (json.JSONDecodeError, ValueError, KeyError):
        return None  # Treat corruption as cache miss
```

### 4. Service Layer Wrapper (SRP)

처리 함수를 순수하게 유지하세요. 캐싱을 별도의 service layer로 추가합니다.

```python
def extract_with_cache(
    file_path: Path,
    *,
    cache_enabled: bool = True,
    cache_dir: Path = Path(".cache"),
) -> ExtractedDocument:
    """Service layer: cache check -> extraction -> cache write."""
    if not cache_enabled:
        return extract_text(file_path)  # Pure function, no cache knowledge

    file_hash = compute_file_hash(file_path)

    # Check cache
    cached = read_cache(cache_dir, file_hash)
    if cached is not None:
        logger.info("Cache hit: %s (hash=%s)", file_path.name, file_hash[:12])
        return cached.document

    # Cache miss -> extract -> store
    logger.info("Cache miss: %s (hash=%s)", file_path.name, file_hash[:12])
    doc = extract_text(file_path)
    entry = CacheEntry(file_hash=file_hash, source_path=str(file_path), document=doc)
    write_cache(cache_dir, entry)
    return doc
```

## 주요 설계 결정 사항

| 결정 사항 | 근거 |
|----------|-----------|
| SHA-256 content hash | 경로 독립적, 내용 변경 시 자동 무효화 |
| `{hash}.json` 파일 명명 | O(1) 조회, 인덱스 파일 불필요 |
| Service layer wrapper | SRP: 추출 기능은 순수하게 유지되고 캐시는 별도의 관심사로 분리됨 |
| 수동 JSON 직렬화 | frozen dataclass 직렬화에 대한 완전한 제어 |
| 손상 시 `None` 반환 | 성능 저하를 최소화(Graceful degradation)하며 다음 실행 시 재처리 |
| `cache_dir.mkdir(parents=True)` | 첫 쓰기 시 디렉토리 지연 생성 |

## Best Practices

- **경로가 아닌 내용을 해싱하세요** — 경로는 변경되지만 내용의 정체성은 변하지 않습니다.
- **해싱 시 큰 파일은 청크 단위로 처리하세요** — 전체 파일을 메모리에 로드하는 것을 피합니다.
- **처리 함수를 순수하게 유지하세요** — 캐싱에 대해 전혀 알지 못해야 합니다.
- **디버깅을 위해 단축된 해시와 함께 캐시 히트/미스를 로그로 남기세요**
- **손상을 우아하게 처리하세요** — 잘못된 캐시 엔트리는 미스로 취급하고 절대 크래시가 발생하지 않도록 합니다.

## Anti-Patterns

```python
# BAD: Path-based caching (breaks on file move/rename)
cache = {"/path/to/file.pdf": result}

# BAD: Adding cache logic inside the processing function (SRP violation)
def extract_text(path, *, cache_enabled=False, cache_dir=None):
    if cache_enabled:  # Now this function has two responsibilities
        ...

# BAD: Using dataclasses.asdict() with nested frozen dataclasses
# (can cause issues with complex nested types)
data = dataclasses.asdict(entry)  # Use manual serialization instead
```

## 사용해야 할 때

- 파일 처리 파이프라인 (PDF 파싱, OCR, 텍스트 추출, 이미지 분석)
- --cache/--no-cache 옵션이 유용한 CLI 도구
- 여러 실행에 걸쳐 동일한 파일이 나타나는 배치 처리
- 기존 순수 함수를 수정하지 않고 캐싱을 추가할 때

## 사용하지 말아야 할 때

- 항상 최신 상태여야 하는 데이터 (실시간 피드)
- 캐시 엔트리가 매우 큰 경우 (대신 스트리밍 고려)
- 파일 내용 이외의 매개변수에 의존하는 결과 (예: 다른 추출 설정)
