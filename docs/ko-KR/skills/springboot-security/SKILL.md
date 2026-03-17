---
name: springboot-security
description: Java Spring Boot 서비스에서의 authn/authz, validation, CSRF, secrets, headers, rate limiting, 그리고 dependency security를 위한 Spring Security best practices.
origin: ECC
---

# Spring Boot Security Review

auth 추가, input 처리, endpoint 생성 또는 secrets를 다룰 때 사용하세요.

## 활성화 시점

- authentication 추가 (JWT, OAuth2, session-based)
- authorization 구현 ( @PreAuthorize, role-based access)
- user input validation (Bean Validation, custom validators)
- CORS, CSRF 또는 security headers 설정
- secrets 관리 (Vault, environment variables)
- rate limiting 또는 brute-force 방어 추가
- CVE를 위한 dependency 스캐닝

## Authentication

- stateless JWT 또는 revocation list가 포함된 opaque token을 권장함
- session에 대해 `httpOnly`, `Secure`, `SameSite=Strict` cookie를 사용함
- `OncePerRequestFilter` 또는 resource server를 통해 token을 validate함

```java
 @everything-claude-code/schemas/install-components.schema.json
public class JwtAuthFilter extends OncePerRequestFilter {
  private final JwtService jwtService;

  public JwtAuthFilter(JwtService jwtService) {
    this.jwtService = jwtService;
  }

  @Override
  protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
      FilterChain chain) throws ServletException, IOException {
    String header = request.getHeader(HttpHeaders.AUTHORIZATION);
    if (header != null && header.startsWith("Bearer ")) {
      String token = header.substring(7);
      Authentication auth = jwtService.authenticate(token);
      SecurityContextHolder.getContext().setAuthentication(auth);
    }
    chain.doFilter(request, response);
  }
}
```

## Authorization

- method security 활성화: ` @EnableMethodSecurity`
- ` @PreAuthorize("hasRole('ADMIN')")` 또는 ` @PreAuthorize(" @authz.canEdit(#id)")`를 사용함
- 기본적으로 deny 설정; 필요한 scope만 노출함

```java
 @RestController @RequestMapping("/api/admin")
public class AdminController {

  @PreAuthorize("hasRole('ADMIN')")
  @GetMapping("/users")
  public List<UserDto> listUsers() {
    return userService.findAll();
  }

  @PreAuthorize(" @authz.isOwner(#id, authentication)")
  @DeleteMapping("/users/{id}")
  public ResponseEntity<Void> deleteUser( @PathVariable Long id) {
    userService.delete(id);
    return ResponseEntity.noContent().build();
  }
}
```

## Input Validation

- controller에서 ` @Valid`와 함께 Bean Validation을 사용함
- DTO에 constraint 적용: ` @NotBlank`, ` @Email`, ` @Size`, custom validators
- 렌더링 전 whitelist를 사용하여 모든 HTML을 sanitize함

```java
// BAD: No validation
 @PostMapping("/users")
public User createUser( @RequestBody UserDto dto) {
  return userService.create(dto);
}

// GOOD: Validated DTO
public record CreateUserDto(
    @NotBlank @Size(max = 100) String name,
    @NotBlank @Email String email,
    @NotNull @everything-claude-code/skills/videodb/reference/streaming.md(0) @Max(150) Integer age
) {}

 @PostMapping("/users")
public ResponseEntity<UserDto> createUser( @everything-claude-code/tests/ci/validators.test.js @RequestBody CreateUserDto dto) {
  return ResponseEntity.status(HttpStatus.CREATED)
      .body(userService.create(dto));
}
```

## SQL Injection 방지

- Spring Data repository 또는 parameterized query를 사용함
- native query의 경우 `:param` 바인딩을 사용하고, 절대 문자열을 결합하지 말 것

```java
// BAD: String concatenation in native query
 @Query(value = "SELECT * FROM users WHERE name = '" + name + "'", nativeQuery = true)

// GOOD: Parameterized native query
 @Query(value = "SELECT * FROM users WHERE name = :name", nativeQuery = true)
List<User> findByName( @Param("name") String name);

// GOOD: Spring Data derived query (auto-parameterized)
List<User> findByEmailAndActiveTrue(String email);
```

## Password Encoding

- 항상 BCrypt 또는 Argon2로 password를 해싱함 — 절대 plaintext로 저장하지 말 것
- 수동 해싱 대신 `PasswordEncoder` bean을 사용함

```java
 @Bean
public PasswordEncoder passwordEncoder() {
  return new BCryptPasswordEncoder(12); // cost factor 12
}

// In service
public User register(CreateUserDto dto) {
  String hashedPassword = passwordEncoder.encode(dto.password());
  return userRepository.save(new User(dto.email(), hashedPassword));
}
```

## CSRF Protection

- 브라우저 session 앱의 경우 CSRF를 활성화 상태로 유지하고, form/header에 token을 포함함
- Bearer token을 사용하는 순수 API의 경우 CSRF를 비활성화하고 stateless auth에 의존함

```java
http
  .csrf(csrf -> csrf.disable())
  .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS));
```

## Secrets Management

- 소스에 secrets를 두지 말 것; env 또는 vault에서 로드함
- `application.yml`에 credential을 두지 말 것; placeholder를 사용함
- token 및 DB credential을 정기적으로 rotate함

```yaml
# BAD: Hardcoded in application.yml
spring:
  datasource:
    password: mySecretPassword123

# GOOD: Environment variable placeholder
spring:
  datasource:
    password: ${DB_PASSWORD}

# GOOD: Spring Cloud Vault integration
spring:
  cloud:
    vault:
      uri: https://vault.example.com
      token: ${VAULT_TOKEN}
```

## Security Headers

```java
http
  .headers(headers -> headers
    .contentSecurityPolicy(csp -> csp
      .policyDirectives("default-src 'self'"))
    .frameOptions(HeadersConfigurer.FrameOptionsConfig::sameOrigin)
    .xssProtection(Customizer.withDefaults())
    .referrerPolicy(rp -> rp.policy(ReferrerPolicyHeaderWriter.ReferrerPolicy.NO_REFERRER)));
```

## CORS 설정

- CORS를 controller별이 아닌 security filter 레벨에서 설정함
- 허용된 origin을 제한함 — 프로덕션에서 절대 `*`를 사용하지 말 것

```java
 @Bean
public CorsConfigurationSource corsConfigurationSource() {
  CorsConfiguration config = new CorsConfiguration();
  config.setAllowedOrigins(List.of("https://app.example.com"));
  config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
  config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
  config.setAllowCredentials(true);
  config.setMaxAge(3600L);

  UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
  source.registerCorsConfiguration("/api/**", config);
  return source;
}

// In SecurityFilterChain:
http.cors(cors -> cors.configurationSource(corsConfigurationSource()));
```

## Rate Limiting

- 비용이 많이 드는 endpoint에 Bucket4j 또는 gateway 레벨의 제한을 적용함
- 트래픽 폭주 시 로그 및 알람을 생성하고, retry hint와 함께 429를 반환함

```java
// Using Bucket4j for per-endpoint rate limiting
 @everything-claude-code/schemas/install-components.schema.json
public class RateLimitFilter extends OncePerRequestFilter {
  private final Map<String, Bucket> buckets = new ConcurrentHashMap<>();

  private Bucket createBucket() {
    return Bucket.builder()
        .addLimit(Bandwidth.classic(100, Refill.intervally(100, Duration.ofMinutes(1))))
        .build();
  }

  @Override
  protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
      FilterChain chain) throws ServletException, IOException {
    String clientIp = request.getRemoteAddr();
    Bucket bucket = buckets.computeIfAbsent(clientIp, k -> createBucket());

    if (bucket.tryConsume(1)) {
      chain.doFilter(request, response);
    } else {
      response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
      response.getWriter().write("{\"error\": \"Rate limit exceeded\"}");
    }
  }
}
```

## Dependency Security

- CI에서 OWASP Dependency Check / Snyk를 실행함
- Spring Boot 및 Spring Security를 지원되는 버전으로 유지함
- 알려진 CVE가 있는 경우 build를 실패 처리함

## Logging 및 PII

- secrets, tokens, passwords 또는 전체 PAN 데이터를 절대 로깅하지 말 것
- 민감한 필드를 redact 처리하고, structured JSON logging을 사용함

## File Upload

- size, content type 및 확장자를 validate함
- web root 외부에 저장하고, 필요한 경우 스캔함

## 배포 전 체크리스트

- [ ] auth token이 정상적으로 validate되고 만료되는지 확인
- [ ] 모든 민감한 path에 authorization guard가 설정되었는지 확인
- [ ] 모든 input이 validate되고 sanitize되었는지 확인
- [ ] 문자열 결합 방식의 SQL이 없는지 확인
- [ ] 앱 유형에 맞는 CSRF 설정 확인
- [ ] secrets가 외부화되었으며 commit되지 않았는지 확인
- [ ] security headers가 설정되었는지 확인
- [ ] API에 rate limiting이 적용되었는지 확인
- [ ] dependency가 스캔되었으며 최신 상태인지 확인
- [ ] 로그에 민감한 데이터가 없는지 확인

**기억하세요**: 기본적으로 거부(Deny by default), input validation, 최소 권한(least privilege), 그리고 설정에 의한 보안(secure-by-configuration)을 최우선으로 하세요.

--- Content from referenced files ---

Content from @everything-claude-code/schemas/install-components.schema.json:
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "ECC Install Components",
  "type": "object",
  "additionalProperties": false,
  "required": [
    "version",
    "components"
  ],
  "properties": {
    "version": {
      "type": "integer",
      "minimum": 1
    },
    "components": {
      "type": "array",
      "items": {
        "type": "object",
        "additionalProperties": false,
        "required": [
          "id",
          "family",
          "description",
          "modules"
        ],
        "properties": {
          "id": {
            "type": "string",
            "pattern": "^(baseline|lang|framework|capability):[a-z0-9-]+$"
          },
          "family": {
            "type": "string",
            "enum": [
              "baseline",
              "language",
              "framework",
              "capability"
            ]
          },
          "description": {
            "type": "string",
            "minLength": 1
          },
          "modules": {
            "type": "array",
            "minItems": 1,
            "items": {
              "type": "string",
              "pattern": "^[a-z0-9-]+$"
            }
          }
        }
      }
    }
  }
}

Content from @everything-claude-code/skills/videodb/reference/streaming.md:
# Streaming & Playback

VideoDB는 필요에 따라 스트림을 생성하여 모든 표준 비디오 플레이어에서 즉시 재생되는 HLS 호환 URL을 반환합니다. 렌더링 시간이나 export 대기 시간이 없습니다 - 편집, 검색 및 composition이 즉시 스트리밍됩니다.

## Prerequisites

스트림을 생성하려면 먼저 비디오가 collection에 **업로드되어야 합니다**. 검색 기반 스트림의 경우 비디오가 **인덱싱**(spoken words 및/또는 scenes)되어야 합니다. 인덱싱에 대한 자세한 내용은 [search.md](search.md)를 참조하세요.

## Core Concepts

### Stream Generation

VideoDB의 모든 비디오, 검색 결과 및 timeline은 **stream URL**을 생성할 수 있습니다. 이 URL은 필요에 따라 컴파일되는 HLS (HTTP Live Streaming) manifest를 가리킵니다.

```python
# 비디오에서 생성
stream_url = video.generate_stream()

# timeline에서 생성
stream_url = timeline.generate_stream()

# 검색 결과에서 컴파일
stream_url = results.compile()
```

## 단일 비디오 스트리밍

### 기본 재생

```python
import videodb

conn = videodb.connect()
coll = conn.get_collection()
video = coll.get_video("your-video-id")

# stream URL 생성
stream_url = video.generate_stream()
print(f"Stream: {stream_url}")

# 기본 브라우저에서 열기
video.play()
```

### 자막 포함

```python
# 먼저 인덱싱 및 자막 추가
video.index_spoken_words(force=True)
stream_url = video.add_subtitle()

# 반환된 URL에는 이미 자막이 포함되어 있습니다
print(f"Subtitled stream: {stream_url}")
```

### 특정 세그먼트

timestamp 범위의 timeline을 전달하여 비디오의 일부만 스트리밍합니다:

```python
# 10-30초 및 60-90초 스트리밍
stream_url = video.generate_stream(timeline=[(10, 30), (60, 90)])
print(f"Segment stream: {stream_url}")
```

## Timeline Compositions 스트리밍

multi-asset composition을 빌드하고 실시간으로 스트리밍합니다:

```python
import videodb
from videodb.timeline import Timeline
from videodb.asset import VideoAsset, AudioAsset, ImageAsset, TextAsset, TextStyle

conn = videodb.connect()
coll = conn.get_collection()

video = coll.get_video(video_id)
music = coll.get_audio(music_id)

timeline = Timeline(conn)

# 메인 비디오 콘텐츠
timeline.add_inline(VideoAsset(asset_id=video.id))

# 배경 음악 오버레이 (0초에서 시작)
timeline.add_overlay(0, AudioAsset(asset_id=music.id))

# 시작 부분에 텍스트 오버레이
timeline.add_overlay(0, TextAsset(
    text="Live Demo",
    duration=3,
    style=TextStyle(fontsize=48, fontcolor="white", boxcolor="#000000"),
))

# 구성된 스트림 생성
stream_url = timeline.generate_stream()
print(f"Composed stream: {stream_url}")
```

**중요:** `add_inline()`은 `VideoAsset`만 허용합니다. `AudioAsset`, `ImageAsset`, `TextAsset`에는 `add_overlay()`를 사용하세요.

자세한 timeline 편집은 [editor.md](editor.md)를 참조하세요.

## 검색 결과 스트리밍

검색 결과를 일치하는 모든 세그먼트의 단일 스트림으로 컴파일합니다:

```python
from videodb import SearchType
from videodb.exceptions import InvalidRequestError

video.index_spoken_words(force=True)
try:
    results = video.search("key announcement", search_type=SearchType.semantic)

    # 모든 일치하는 샷을 하나의 스트림으로 컴파일
    stream_url = results.compile()
    print(f"Search results stream: {stream_url}")

    # 또는 직접 재생
    results.play()
except InvalidRequestError as exc:
    if "No results found" in str(exc):
        print("No matching announcement segments were found.")
    else:
        raise
```

### 개별 검색 결과 스트리밍

```python
from videodb.exceptions import InvalidRequestError

try:
    results = video.search("product demo", search_type=SearchType.semantic)
    for i, shot in enumerate(results.get_shots()):
        stream_url = shot.generate_stream()
        print(f"Hit {i+1} [{shot.start:.1f}s-{shot.end:.1f}s]: {stream_url}")
except InvalidRequestError as exc:
    if "No results found" in str(exc):
        print("No product demo segments matched the query.")
    else:
        raise
```

## 오디오 재생

오디오 콘텐츠에 대한 서명된 재생 URL을 가져옵니다:

```python
audio = coll.get_audio(audio_id)
playback_url = audio.generate_url()
print(f"Audio URL: {playback_url}")
```

## 전체 워크플로 예시

### Search-to-Stream 파이프라인

검색, timeline 구성 및 스트리밍을 하나의 워크플로로 결합합니다:

```python
import videodb
from videodb import SearchType
from videodb.exceptions import InvalidRequestError
from videodb.timeline import Timeline
from videodb.asset import VideoAsset, TextAsset, TextStyle

conn = videodb.connect()
coll = conn.get_collection()
video = coll.get_video("your-video-id")

video.index_spoken_words(force=True)

# 주요 순간 검색
queries = ["introduction", "main demo", "Q&A"]
timeline = Timeline(conn)
timeline_offset = 0.0

for query in queries:
    try:
        results = video.search(query, search_type=SearchType.semantic)
        shots = results.get_shots()
    except InvalidRequestError as exc:
        if "No results found" in str(exc):
            shots = []
        else:
            raise

    if not shots:
        continue

    # 컴파일된 timeline에서 이 배치가 시작되는 위치에 섹션 레이블 추가
    timeline.add_overlay(timeline_offset, TextAsset(
        text=query.title(),
        duration=2,
        style=TextStyle(fontsize=36, fontcolor="white", boxcolor="#222222"),
    ))

    for shot in shots:
        timeline.add_inline(
            VideoAsset(asset_id=shot.video_id, start=shot.start, end=shot.end)
        )
        timeline_offset += shot.end - shot.start

stream_url = timeline.generate_stream()
print(f"Dynamic compilation: {stream_url}")
```

### Multi-Video 스트림

서로 다른 비디오의 클립을 단일 스트림으로 결합합니다:

```python
import videodb
from videodb.timeline import Timeline
from videodb.asset import VideoAsset

conn = videodb.connect()
coll = conn.get_collection()

video_clips = [
    {"id": "vid_001", "start": 0, "end": 15},
    {"id": "vid_002", "start": 10, "end": 30},
    {"id": "vid_003", "start": 5, "end": 25},
]

timeline = Timeline(conn)
for clip in video_clips:
    timeline.add_inline(
        VideoAsset(asset_id=clip["id"], start=clip["start"], end=clip["end"])
    )

stream_url = timeline.generate_stream()
print(f"Multi-video stream: {stream_url}")
```

### 조건부 스트림 조립

검색 가용성에 따라 동적으로 스트림을 빌드합니다:

```python
import videodb
from videodb import SearchType
from videodb.exceptions import InvalidRequestError
from videodb.timeline import Timeline
from videodb.asset import VideoAsset, TextAsset, TextStyle

conn = videodb.connect()
coll = conn.get_collection()
video = coll.get_video("your-video-id")

video.index_spoken_words(force=True)

timeline = Timeline(conn)

# 특정 콘텐츠를 찾으려고 시도하고, 실패하면 전체 비디오로 대체합니다
topics = ["opening remarks", "technical deep dive", "closing"]

found_any = False
timeline_offset = 0.0
for topic in topics:
    try:
        results = video.search(topic, search_type=SearchType.semantic)
        shots = results.get_shots()
    except InvalidRequestError as exc:
        if "No results found" in str(exc):
            shots = []
        else:
            raise

    if shots:
        found_any = True
        timeline.add_overlay(timeline_offset, TextAsset(
            text=topic.title(),
            duration=2,
            style=TextStyle(fontsize=32, fontcolor="white", boxcolor="#1a1a2e"),
        ))
        for shot in shots:
            timeline.add_inline(
                VideoAsset(asset_id=shot.video_id, start=shot.start, end=shot.end)
            )
            timeline_offset += shot.end - shot.start

if found_any:
    stream_url = timeline.generate_stream()
    print(f"Curated stream: {stream_url}")
else:
    # 전체 비디오 스트림으로 대체
    stream_url = video.generate_stream()
    print(f"Full video stream: {stream_url}")
```

### 라이브 이벤트 Recap

이벤트 녹화물을 여러 섹션이 있는 스트리밍 가능한 recap으로 처리합니다:

```python
import videodb
from videodb import SearchType
from videodb.exceptions import InvalidRequestError
from videodb.timeline import Timeline
from videodb.asset import VideoAsset, AudioAsset, ImageAsset, TextAsset, TextStyle

conn = videodb.connect()
coll = conn.get_collection()

# 이벤트 녹화 업로드
event = coll.upload(url="https://example.com/event-recording.mp4")
event.index_spoken_words(force=True)

# 배경 음악 생성
music = coll.generate_music(
    prompt="upbeat corporate background music",
    duration=120,
)

# 제목 이미지 생성
title_img = coll.generate_image(
    prompt="modern event recap title card, dark background, professional",
    aspect_ratio="16:9",
)

# recap timeline 빌드
timeline = Timeline(conn)
timeline_offset = 0.0

# 검색을 통한 메인 비디오 세그먼트
try:
    keynote = event.search("keynote announcement", search_type=SearchType.semantic)
    keynote_shots = keynote.get_shots()[:5]
except InvalidRequestError as exc:
    if "No results found" in str(exc):
        keynote_shots = []
    else:
        raise
if keynote_shots:
    keynote_start = timeline_offset
    for shot in keynote_shots:
        timeline.add_inline(
            VideoAsset(asset_id=shot.video_id, start=shot.start, end=shot.end)
        )
        timeline_offset += shot.end - shot.start
else:
    keynote_start = None

try:
    demo = event.search("product demo", search_type=SearchType.semantic)
    demo_shots = demo.get_shots()[:5]
except InvalidRequestError as exc:
    if "No results found" in str(exc):
        demo_shots = []
    else:
        raise
if demo_shots:
    demo_start = timeline_offset
    for shot in demo_shots:
        timeline.add_inline(
            VideoAsset(asset_id=shot.video_id, start=shot.start, end=shot.end)
        )
        timeline_offset += shot.end - shot.start
else:
    demo_start = None

# 제목 카드 이미지 오버레이
timeline.add_overlay(0, ImageAsset(
    asset_id=title_img.id, width=100, height=100, x=80, y=20, duration=5
))

# 정확한 timeline 오프셋에 섹션 레이블 오버레이
if keynote_start is not None:
    timeline.add_overlay(max(5, keynote_start), TextAsset(
        text="Keynote Highlights",
        duration=3,
        style=TextStyle(fontsize=40, fontcolor="white", boxcolor="#0d1117"),
    ))
if demo_start is not None:
    timeline.add_overlay(max(5, demo_start), TextAsset(
        text="Demo Highlights",
        duration=3,
        style=TextStyle(fontsize=36, fontcolor="white", boxcolor="#0d1117"),
    ))

# 배경 음악 오버레이
timeline.add_overlay(0, AudioAsset(
    asset_id=music.id, fade_in_duration=3
))

# 최종 recap 스트리밍
stream_url = timeline.generate_stream()
print(f"Event recap: {stream_url}")
```

---

## 팁

- **HLS 호환성**: stream URL은 HLS manifest (`.m3u8`)를 반환합니다. Safari에서는 기본적으로 작동하며, 다른 브라우저에서는 hls.js 또는 유사한 라이브러리를 통해 작동합니다.
- **On-demand 컴파일**: 스트림은 요청 시 서버 측에서 컴파일됩니다. 첫 번째 재생 시 짧은 컴파일 지연이 있을 수 있으며, 동일한 구성의 후속 재생은 캐시됩니다.
- **캐싱**: 인자 없이 `video.generate_stream()`을 두 번 호출하면 재컴파일하는 대신 캐시된 stream URL을 반환합니다.
- **세그먼트 스트림**: `video.generate_stream(timeline=[(start, end)])`는 전체 `Timeline` 객체를 빌드하지 않고 특정 클립을 스트리밍하는 가장 빠른 방법입니다.
- **Inline vs overlay**: `add_inline()`은 `VideoAsset`만 허용하며 메인 트랙에 자산을 순차적으로 배치합니다. `add_overlay()`는 `AudioAsset`, `ImageAsset`, `TextAsset`을 허용하며 지정된 시작 시간에 레이어링합니다.
- **TextStyle 기본값**: `TextStyle`의 기본값은 `font='Sans'`, `fontcolor='black'`입니다. 텍스트의 배경색에는 `bgcolor`가 아닌 `boxcolor`를 사용하세요.
- **생성 기능과 결합**: `coll.generate_music(prompt, duration)` 및 `coll.generate_image(prompt, aspect_ratio)`를 사용하여 timeline 구성을 위한 자산을 생성하세요.
- **재생**: `.play()`는 시스템 기본 브라우저에서 stream URL을 엽니다. 프로그래밍 방식으로 사용하려면 URL 문자열을 직접 작업하세요.

Content from @everything-claude-code/tests/ci/validators.test.js:
[생략: 1000줄 이상의 JavaScript 테스트 코드]
