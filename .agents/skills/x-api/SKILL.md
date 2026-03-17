---
name: x-api
description: 트윗, thread 게시, timeline 읽기, 검색 및 analytics를 위한 X/Twitter API 통합입니다. OAuth 인증 패턴, rate limits 및 플랫폼 네이티브 콘텐츠 게시를 다룹니다. 사용자가 프로그래밍 방식으로 X와 상호작용하기를 원할 때 사용하십시오.
origin: ECC
---

# X API

포스팅, 읽기, 검색 및 analytics를 위한 X(Twitter)와의 프로그래밍 방식 상호작용.

## 활성화 시점

- 사용자가 프로그래밍 방식으로 tweet이나 thread를 게시하길 원할 때
- X에서 timeline, mention 또는 사용자 데이터를 읽을 때
- 콘텐츠, 트렌드 또는 대화를 위해 X를 검색할 때
- X 통합 또는 bot을 구축할 때
- Analytics 및 engagement 추적
- 사용자가 "post to X", "tweet", "X API" 또는 "Twitter API"라고 말할 때

## 인증

### OAuth 2.0 Bearer Token (App-Only)

가장 적합한 용도: 읽기 위주의 작업, 검색, 공개 데이터.

```bash
# Environment setup
export X_BEARER_TOKEN="your-bearer-token"
```

```python
import os
import requests

bearer = os.environ["X_BEARER_TOKEN"]
headers = {"Authorization": f"Bearer {bearer}"}

# Search recent tweets
resp = requests.get(
    "https://api.x.com/2/tweets/search/recent",
    headers=headers,
    params={"query": "claude code", "max_results": 10}
)
tweets = resp.json()
```

### OAuth 1.0a (User Context)

필수 용도: tweet 게시, 계정 관리, DM.

```bash
# Environment setup — source before use
export X_API_KEY="your-api-key"
export X_API_SECRET="your-api-secret"
export X_ACCESS_TOKEN="your-access-token"
export X_ACCESS_SECRET="your-access-secret"
```

```python
import os
from requests_oauthlib import OAuth1Session

oauth = OAuth1Session(
    os.environ["X_API_KEY"],
    client_secret=os.environ["X_API_SECRET"],
    resource_owner_key=os.environ["X_ACCESS_TOKEN"],
    resource_owner_secret=os.environ["X_ACCESS_SECRET"],
)
```

## 핵심 작업

### Tweet 게시

```python
resp = oauth.post(
    "https://api.x.com/2/tweets",
    json={"text": "Hello from Claude Code"}
)
resp.raise_for_status()
tweet_id = resp.json()["data"]["id"]
```

### Thread 게시

```python
def post_thread(oauth, tweets: list[str]) -> list[str]:
    ids = []
    reply_to = None
    for text in tweets:
        payload = {"text": text}
        if reply_to:
            payload["reply"] = {"in_reply_to_tweet_id": reply_to}
        resp = oauth.post("https://api.x.com/2/tweets", json=payload)
        resp.raise_for_status()
        tweet_id = resp.json()["data"]["id"]
        ids.append(tweet_id)
        reply_to = tweet_id
    return ids
```

### 사용자 Timeline 읽기

```python
resp = requests.get(
    f"https://api.x.com/2/users/{user_id}/tweets",
    headers=headers,
    params={
        "max_results": 10,
        "tweet.fields": "created_at,public_metrics",
    }
)
```

### Tweet 검색

```python
resp = requests.get(
    "https://api.x.com/2/tweets/search/recent",
    headers=headers,
    params={
        "query": "from:affaanmustafa -is:retweet",
        "max_results": 10,
        "tweet.fields": "public_metrics,created_at",
    }
)
```

### Username으로 사용자 조회

```python
resp = requests.get(
    "https://api.x.com/2/users/by/username/affaanmustafa",
    headers=headers,
    params={"user.fields": "public_metrics,description,created_at"}
)
```

### 미디어 업로드 및 게시

```python
# Media upload uses v1.1 endpoint

# Step 1: Upload media
media_resp = oauth.post(
    "https://upload.twitter.com/1.1/media/upload.json",
    files={"media": open("image.png", "rb")}
)
media_id = media_resp.json()["media_id_string"]

# Step 2: Post with media
resp = oauth.post(
    "https://api.x.com/2/tweets",
    json={"text": "Check this out", "media": {"media_ids": [media_id]}}
)
```

## Rate Limits

X API rate limits는 endpoint, 인증 방법 및 계정 등급에 따라 다르며 시간이 지남에 따라 변경됩니다. 항상 다음을 준수하십시오:
- 가정을 하드코딩하기 전에 현재 X developer 문서를 확인하십시오
- 런타임에 `x-rate-limit-remaining` 및 `x-rate-limit-reset` 헤더를 읽으십시오
- 코드의 정적 테이블에 의존하는 대신 자동으로 back off 하십시오

```python
import time

remaining = int(resp.headers.get("x-rate-limit-remaining", 0))
if remaining < 5:
    reset = int(resp.headers.get("x-rate-limit-reset", 0))
    wait = max(0, reset - int(time.time()))
    print(f"Rate limit approaching. Resets in {wait}s")
```

## Error Handling

```python
resp = oauth.post("https://api.x.com/2/tweets", json={"text": content})
if resp.status_code == 201:
    return resp.json()["data"]["id"]
elif resp.status_code == 429:
    reset = int(resp.headers["x-rate-limit-reset"])
    raise Exception(f"Rate limited. Resets at {reset}")
elif resp.status_code == 403:
    raise Exception(f"Forbidden: {resp.json().get('detail', 'check permissions')}")
else:
    raise Exception(f"X API error {resp.status_code}: {resp.text}")
```

## 보안

- **토큰을 절대 하드코딩하지 마십시오.** 환경 변수나 `.env` 파일을 사용하십시오.
- **.env 파일을 절대 commit하지 마십시오.** `.gitignore`에 추가하십시오.
- 노출된 경우 **토큰을 교체(Rotate)하십시오.** developer.x.com에서 재생성하십시오.
- 쓰기 권한이 필요하지 않은 경우 **읽기 전용 토큰을 사용하십시오.**
- **OAuth secret을 안전하게 저장하십시오** — 소스 코드나 로그에 저장하지 마십시오.

## Content Engine과의 통합

`content-engine` skill을 사용하여 플랫폼 네이티브 콘텐츠를 생성한 다음, X API를 통해 게시하십시오:
1. `content-engine`으로 콘텐츠 생성 (X 플랫폼 형식)
2. 길이 검증 (단일 tweet의 경우 280자)
3. 위의 패턴을 사용하여 X API를 통해 게시
4. `public_metrics`를 통해 engagement 추적

## 관련 Skills

- `content-engine` — X를 위한 플랫폼 네이티브 콘텐츠 생성
- `crosspost` — X, LinkedIn 및 기타 플랫폼에 콘텐츠 배포
