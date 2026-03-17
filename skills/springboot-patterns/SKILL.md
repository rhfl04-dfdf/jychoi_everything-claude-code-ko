---
name: springboot-patterns
description: Spring Boot 아키텍처 패턴, REST API 설계, 계층형 서비스, 데이터 접근, 캐싱, 비동기 처리 및 로깅. Java Spring Boot 백엔드 작업 시 사용합니다.
origin: ECC
---

# Spring Boot 개발 패턴

확장 가능하고 프로덕션 수준의 서비스를 위한 Spring Boot 아키텍처 및 API 패턴입니다.

## 활성화 시점

- Spring MVC 또는 WebFlux로 REST API 구축 시
- controller -> service -> repository 레이어 구조화 시
- Spring Data JPA, 캐싱 또는 비동기 처리 설정 시
- validation, 예외 처리 또는 페이지네이션 추가 시
- dev/staging/production 환경별 프로필 설정 시
- Spring Events 또는 Kafka를 사용한 이벤트 기반 패턴 구현 시

## REST API 구조

```java
@RestController
@RequestMapping("/api/markets")
@Validated
class MarketController {
  private final MarketService marketService;

  MarketController(MarketService marketService) {
    this.marketService = marketService;
  }

  @GetMapping
  ResponseEntity<Page<MarketResponse>> list(
      @RequestParam(defaultValue = "0") int page,
      @RequestParam(defaultValue = "20") int size) {
    Page<Market> markets = marketService.list(PageRequest.of(page, size));
    return ResponseEntity.ok(markets.map(MarketResponse::from));
  }

  @PostMapping
  ResponseEntity<MarketResponse> create(@Valid @RequestBody CreateMarketRequest request) {
    Market market = marketService.create(request);
    return ResponseEntity.status(HttpStatus.CREATED).body(MarketResponse.from(market));
  }
}
```

## Repository 패턴 (Spring Data JPA)

```java
public interface MarketRepository extends JpaRepository<MarketEntity, Long> {
  @Query("select m from MarketEntity m where m.status = :status order by m.volume desc")
  List<MarketEntity> findActive(@Param("status") MarketStatus status, Pageable pageable);
}
```

## 트랜잭션이 있는 Service 레이어

```java
@Service
public class MarketService {
  private final MarketRepository repo;

  public MarketService(MarketRepository repo) {
    this.repo = repo;
  }

  @Transactional
  public Market create(CreateMarketRequest request) {
    MarketEntity entity = MarketEntity.from(request);
    MarketEntity saved = repo.save(entity);
    return Market.from(saved);
  }
}
```

## DTO와 Validation

```java
public record CreateMarketRequest(
    @NotBlank @Size(max = 200) String name,
    @NotBlank @Size(max = 2000) String description,
    @NotNull @FutureOrPresent Instant endDate,
    @NotEmpty List<@NotBlank String> categories) {}

public record MarketResponse(Long id, String name, MarketStatus status) {
  static MarketResponse from(Market market) {
    return new MarketResponse(market.id(), market.name(), market.status());
  }
}
```

## 예외 처리

```java
@ControllerAdvice
class GlobalExceptionHandler {
  @ExceptionHandler(MethodArgumentNotValidException.class)
  ResponseEntity<ApiError> handleValidation(MethodArgumentNotValidException ex) {
    String message = ex.getBindingResult().getFieldErrors().stream()
        .map(e -> e.getField() + ": " + e.getDefaultMessage())
        .collect(Collectors.joining(", "));
    return ResponseEntity.badRequest().body(ApiError.validation(message));
  }

  @ExceptionHandler(AccessDeniedException.class)
  ResponseEntity<ApiError> handleAccessDenied() {
    return ResponseEntity.status(HttpStatus.FORBIDDEN).body(ApiError.of("Forbidden"));
  }

  @ExceptionHandler(Exception.class)
  ResponseEntity<ApiError> handleGeneric(Exception ex) {
    // Log unexpected errors with stack traces
    return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
        .body(ApiError.of("Internal server error"));
  }
}
```

## 캐싱

설정 클래스에 `@EnableCaching`이 필요합니다.

```java
@Service
public class MarketCacheService {
  private final MarketRepository repo;

  public MarketCacheService(MarketRepository repo) {
    this.repo = repo;
  }

  @Cacheable(value = "market", key = "#id")
  public Market getById(Long id) {
    return repo.findById(id)
        .map(Market::from)
        .orElseThrow(() -> new EntityNotFoundException("Market not found"));
  }

  @CacheEvict(value = "market", key = "#id")
  public void evict(Long id) {}
}
```

## 비동기 처리

설정 클래스에 `@EnableAsync`가 필요합니다.

```java
@Service
public class NotificationService {
  @Async
  public CompletableFuture<Void> sendAsync(Notification notification) {
    // send email/SMS
    return CompletableFuture.completedFuture(null);
  }
}
```

## 로깅 (SLF4J)

```java
@Service
public class ReportService {
  private static final Logger log = LoggerFactory.getLogger(ReportService.class);

  public Report generate(Long marketId) {
    log.info("generate_report marketId={}", marketId);
    try {
      // logic
    } catch (Exception ex) {
      log.error("generate_report_failed marketId={}", marketId, ex);
      throw ex;
    }
    return new Report();
  }
}
```

## 미들웨어 / 필터

```java
@Component
public class RequestLoggingFilter extends OncePerRequestFilter {
  private static final Logger log = LoggerFactory.getLogger(RequestLoggingFilter.class);

  @Override
  protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
      FilterChain filterChain) throws ServletException, IOException {
    long start = System.currentTimeMillis();
    try {
      filterChain.doFilter(request, response);
    } finally {
      long duration = System.currentTimeMillis() - start;
      log.info("req method={} uri={} status={} durationMs={}",
          request.getMethod(), request.getRequestURI(), response.getStatus(), duration);
    }
  }
}
```

## 페이지네이션과 정렬

```java
PageRequest page = PageRequest.of(pageNumber, pageSize, Sort.by("createdAt").descending());
Page<Market> results = marketService.list(page);
```

## 오류 복원력 있는 외부 호출

```java
public <T> T withRetry(Supplier<T> supplier, int maxRetries) {
  int attempts = 0;
  while (true) {
    try {
      return supplier.get();
    } catch (Exception ex) {
      attempts++;
      if (attempts >= maxRetries) {
        throw ex;
      }
      try {
        Thread.sleep((long) Math.pow(2, attempts) * 100L);
      } catch (InterruptedException ie) {
        Thread.currentThread().interrupt();
        throw ex;
      }
    }
  }
}
```

## Rate Limiting (Filter + Bucket4j)

**보안 참고**: `X-Forwarded-For` 헤더는 클라이언트가 위조할 수 있으므로 기본적으로 신뢰할 수 없습니다.
다음 조건이 충족될 때만 forwarded 헤더를 사용하세요:
1. 앱이 신뢰할 수 있는 리버스 프록시(nginx, AWS ALB 등) 뒤에 있을 때
2. `ForwardedHeaderFilter`를 Bean으로 등록했을 때
3. application properties에서 `server.forward-headers-strategy=NATIVE` 또는 `FRAMEWORK`를 설정했을 때
4. 프록시가 `X-Forwarded-For` 헤더를 추가가 아닌 덮어쓰기하도록 설정되었을 때

`ForwardedHeaderFilter`가 올바르게 설정되면 `request.getRemoteAddr()`가 forwarded 헤더에서 올바른 클라이언트 IP를 자동으로 반환합니다. 이 설정 없이는 `request.getRemoteAddr()`를 직접 사용하세요. 이것은 직접 연결 IP를 반환하며, 유일하게 신뢰할 수 있는 값입니다.

```java
@Component
public class RateLimitFilter extends OncePerRequestFilter {
  private final Map<String, Bucket> buckets = new ConcurrentHashMap<>();

  /*
   * SECURITY: This filter uses request.getRemoteAddr() to identify clients for rate limiting.
   *
   * If your application is behind a reverse proxy (nginx, AWS ALB, etc.), you MUST configure
   * Spring to handle forwarded headers properly for accurate client IP detection:
   *
   * 1. Set server.forward-headers-strategy=NATIVE (for cloud platforms) or FRAMEWORK in
   *    application.properties/yaml
   * 2. If using FRAMEWORK strategy, register ForwardedHeaderFilter:
   *
   *    @Bean
   *    ForwardedHeaderFilter forwardedHeaderFilter() {
   *        return new ForwardedHeaderFilter();
   *    }
   *
   * 3. Ensure your proxy overwrites (not appends) the X-Forwarded-For header to prevent spoofing
   * 4. Configure server.tomcat.remoteip.trusted-proxies or equivalent for your container
   *
   * Without this configuration, request.getRemoteAddr() returns the proxy IP, not the client IP.
   * Do NOT read X-Forwarded-For directly—it is trivially spoofable without trusted proxy handling.
   */
  @Override
  protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
      FilterChain filterChain) throws ServletException, IOException {
    // Use getRemoteAddr() which returns the correct client IP when ForwardedHeaderFilter
    // is configured, or the direct connection IP otherwise. Never trust X-Forwarded-For
    // headers directly without proper proxy configuration.
    String clientIp = request.getRemoteAddr();

    Bucket bucket = buckets.computeIfAbsent(clientIp,
        k -> Bucket.builder()
            .addLimit(Bandwidth.classic(100, Refill.greedy(100, Duration.ofMinutes(1))))
            .build());

    if (bucket.tryConsume(1)) {
      filterChain.doFilter(request, response);
    } else {
      response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
    }
  }
}
```

## 백그라운드 작업

Spring의 `@Scheduled`를 사용하거나 큐(예: Kafka, SQS, RabbitMQ)와 통합하세요. 핸들러는 멱등성과 관측 가능성을 유지해야 합니다.

## 관측 가능성

- Logback 인코더를 통한 구조화된 로깅 (JSON)
- 메트릭: Micrometer + Prometheus/OTel
- 트레이싱: Micrometer Tracing (OpenTelemetry 또는 Brave 백엔드)

## 프로덕션 기본 설정

- 생성자 주입을 선호하고 필드 주입을 피하세요
- RFC 7807 오류를 위해 `spring.mvc.problemdetails.enabled=true` 활성화 (Spring Boot 3+)
- 워크로드에 맞게 HikariCP 풀 크기를 설정하고 타임아웃을 구성하세요
- 조회에는 `@Transactional(readOnly = true)` 사용
- `@NonNull`과 `Optional`을 적절히 사용하여 null 안전성을 확보하세요

**기억하세요**: controller는 얇게, service는 집중적으로, repository는 단순하게, 오류는 중앙에서 처리하세요. 유지보수성과 테스트 용이성을 위해 최적화하세요.
