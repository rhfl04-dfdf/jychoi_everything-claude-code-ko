---
name: springboot-security
description: Java Spring Boot 서비스의 인증/인가, validation, CSRF, 시크릿, 헤더, rate limiting 및 의존성 보안에 대한 Spring Security 모범 사례입니다.
origin: ECC
---

# Spring Boot 보안 리뷰

인증 추가, 입력 처리, 엔드포인트 생성 또는 시크릿 관리 시 사용합니다.

## 활성화 시점

- 인증 추가 시 (JWT, OAuth2, 세션 기반)
- 인가 구현 시 (@PreAuthorize, 역할 기반 접근 제어)
- 사용자 입력 검증 시 (Bean Validation, 커스텀 validator)
- CORS, CSRF 또는 보안 헤더 설정 시
- 시크릿 관리 시 (Vault, 환경 변수)
- rate limiting 또는 무차별 대입 공격 방지 추가 시
- 의존성 CVE 스캔 시

## 인증

- 상태 비저장 JWT 또는 해지 목록이 있는 opaque 토큰 선호
- 세션에는 `httpOnly`, `Secure`, `SameSite=Strict` 쿠키 사용
- `OncePerRequestFilter` 또는 resource server로 토큰 검증

```java
@Component
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

## 인가

- 메서드 보안 활성화: `@EnableMethodSecurity`
- `@PreAuthorize("hasRole('ADMIN')")` 또는 `@PreAuthorize("@authz.canEdit(#id)")` 사용
- 기본적으로 거부하고 필요한 범위만 노출

```java
@RestController
@RequestMapping("/api/admin")
public class AdminController {

  @PreAuthorize("hasRole('ADMIN')")
  @GetMapping("/users")
  public List<UserDto> listUsers() {
    return userService.findAll();
  }

  @PreAuthorize("@authz.isOwner(#id, authentication)")
  @DeleteMapping("/users/{id}")
  public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
    userService.delete(id);
    return ResponseEntity.noContent().build();
  }
}
```

## 입력 검증

- controller에서 `@Valid`를 사용한 Bean Validation 적용
- DTO에 제약조건 적용: `@NotBlank`, `@Email`, `@Size`, 커스텀 validator
- 렌더링 전에 화이트리스트로 HTML을 정제

```java
// BAD: No validation
@PostMapping("/users")
public User createUser(@RequestBody UserDto dto) {
  return userService.create(dto);
}

// GOOD: Validated DTO
public record CreateUserDto(
    @NotBlank @Size(max = 100) String name,
    @NotBlank @Email String email,
    @NotNull @Min(0) @Max(150) Integer age
) {}

@PostMapping("/users")
public ResponseEntity<UserDto> createUser(@Valid @RequestBody CreateUserDto dto) {
  return ResponseEntity.status(HttpStatus.CREATED)
      .body(userService.create(dto));
}
```

## SQL Injection 방지

- Spring Data repository 또는 파라미터화된 쿼리 사용
- native 쿼리에서는 `:param` 바인딩 사용; 문자열 연결 금지

```java
// BAD: String concatenation in native query
@Query(value = "SELECT * FROM users WHERE name = '" + name + "'", nativeQuery = true)

// GOOD: Parameterized native query
@Query(value = "SELECT * FROM users WHERE name = :name", nativeQuery = true)
List<User> findByName(@Param("name") String name);

// GOOD: Spring Data derived query (auto-parameterized)
List<User> findByEmailAndActiveTrue(String email);
```

## 비밀번호 인코딩

- 항상 BCrypt 또는 Argon2로 비밀번호를 해시 -- 평문 저장 금지
- 수동 해싱 대신 `PasswordEncoder` Bean 사용

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

## CSRF 보호

- 브라우저 세션 앱의 경우 CSRF를 활성화 상태로 유지하고 폼/헤더에 토큰 포함
- Bearer 토큰을 사용하는 순수 API의 경우 CSRF를 비활성화하고 상태 비저장 인증에 의존

```java
http
  .csrf(csrf -> csrf.disable())
  .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS));
```

## 시크릿 관리

- 소스 코드에 시크릿 포함 금지; 환경 변수 또는 vault에서 로드
- `application.yml`에 자격 증명을 넣지 않고 플레이스홀더 사용
- 토큰 및 DB 자격 증명을 정기적으로 교체

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

## 보안 헤더

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

- controller별이 아닌 security filter 수준에서 CORS 설정
- 허용된 origin을 제한 -- 프로덕션에서 `*` 사용 금지

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

- 비용이 높은 엔드포인트에 Bucket4j 또는 게이트웨이 수준의 제한 적용
- 버스트를 로깅하고 알림 전송; 재시도 힌트와 함께 429 반환

```java
// Using Bucket4j for per-endpoint rate limiting
@Component
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

## 의존성 보안

- CI에서 OWASP Dependency Check / Snyk 실행
- Spring Boot 및 Spring Security를 지원되는 버전으로 유지
- 알려진 CVE 발견 시 빌드 실패 처리

## 로깅과 PII

- 시크릿, 토큰, 비밀번호 또는 전체 PAN 데이터를 절대 로깅하지 않음
- 민감한 필드를 마스킹하고 구조화된 JSON 로깅 사용

## 파일 업로드

- 크기, 콘텐츠 타입, 확장자 검증
- 웹 루트 외부에 저장; 필요시 스캔

## 릴리스 전 체크리스트

- [ ] 인증 토큰이 올바르게 검증되고 만료되는지 확인
- [ ] 모든 민감한 경로에 인가 가드 적용
- [ ] 모든 입력이 검증되고 정제됨
- [ ] 문자열 연결 SQL 없음
- [ ] 앱 유형에 맞는 CSRF 설정
- [ ] 시크릿이 외부화되어 있고 커밋되지 않음
- [ ] 보안 헤더 설정 완료
- [ ] API에 rate limiting 적용
- [ ] 의존성 스캔 완료 및 최신 상태
- [ ] 로그에 민감한 데이터 없음

**기억하세요**: 기본적으로 거부, 입력 검증, 최소 권한, 설정 기반 보안 우선입니다.
