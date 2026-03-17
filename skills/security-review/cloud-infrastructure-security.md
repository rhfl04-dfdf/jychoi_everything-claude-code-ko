| name | description |
|------|-------------|
| cloud-infrastructure-security | 클라우드 플랫폼 배포, 인프라 구성, IAM 정책 관리, 로깅/모니터링 설정, CI/CD 파이프라인 구현 시 이 skill을 사용하세요. 모범 사례에 맞춘 클라우드 보안 체크리스트를 제공합니다. |

# 클라우드 및 인프라 보안 Skill

이 skill은 클라우드 인프라, CI/CD 파이프라인, 배포 구성이 보안 모범 사례를 따르고 업계 표준을 준수하도록 보장합니다.

## 활성화 시점

- 클라우드 플랫폼에 애플리케이션 배포 시 (AWS, Vercel, Railway, Cloudflare)
- IAM 역할 및 권한 구성 시
- CI/CD 파이프라인 설정 시
- 코드형 인프라 구현 시 (Terraform, CloudFormation)
- 로깅 및 모니터링 구성 시
- 클라우드 환경에서 시크릿 관리 시
- CDN 및 엣지 보안 설정 시
- 재해 복구 및 백업 전략 구현 시

## 클라우드 보안 체크리스트

### 1. IAM 및 접근 제어

#### 최소 권한 원칙

```yaml
# ✅ CORRECT: Minimal permissions
iam_role:
  permissions:
    - s3:GetObject  # Only read access
    - s3:ListBucket
  resources:
    - arn:aws:s3:::my-bucket/*  # Specific bucket only

# ❌ WRONG: Overly broad permissions
iam_role:
  permissions:
    - s3:*  # All S3 actions
  resources:
    - "*"  # All resources
```

#### 다단계 인증 (MFA)

```bash
# ALWAYS enable MFA for root/admin accounts
aws iam enable-mfa-device \
  --user-name admin \
  --serial-number arn:aws:iam::123456789:mfa/admin \
  --authentication-code1 123456 \
  --authentication-code2 789012
```

#### 검증 단계

- [ ] 프로덕션에서 루트 계정 미사용
- [ ] 모든 권한 있는 계정에 MFA 활성화
- [ ] 서비스 계정은 장기 자격 증명 대신 역할 사용
- [ ] IAM 정책이 최소 권한 원칙 준수
- [ ] 정기적인 접근 권한 검토 수행
- [ ] 미사용 자격 증명 교체 또는 제거

### 2. 시크릿 관리

#### 클라우드 시크릿 매니저

```typescript
// ✅ CORRECT: Use cloud secrets manager
import { SecretsManager } from '@aws-sdk/client-secrets-manager';

const client = new SecretsManager({ region: 'us-east-1' });
const secret = await client.getSecretValue({ SecretId: 'prod/api-key' });
const apiKey = JSON.parse(secret.SecretString).key;

// ❌ WRONG: Hardcoded or in environment variables only
const apiKey = process.env.API_KEY; // Not rotated, not audited
```

#### 시크릿 교체

```bash
# Set up automatic rotation for database credentials
aws secretsmanager rotate-secret \
  --secret-id prod/db-password \
  --rotation-lambda-arn arn:aws:lambda:region:account:function:rotate \
  --rotation-rules AutomaticallyAfterDays=30
```

#### 검증 단계

- [ ] 모든 시크릿을 클라우드 시크릿 매니저에 저장 (AWS Secrets Manager, Vercel Secrets)
- [ ] 데이터베이스 자격 증명에 자동 교체 활성화
- [ ] API 키를 최소 분기마다 교체
- [ ] 코드, 로그, 오류 메시지에 시크릿 미포함
- [ ] 시크릿 접근에 대한 감사 로깅 활성화

### 3. 네트워크 보안

#### VPC 및 방화벽 구성

```terraform
# ✅ CORRECT: Restricted security group
resource "aws_security_group" "app" {
  name = "app-sg"
  
  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["10.0.0.0/16"]  # Internal VPC only
  }
  
  egress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  # Only HTTPS outbound
  }
}

# ❌ WRONG: Open to the internet
resource "aws_security_group" "bad" {
  ingress {
    from_port   = 0
    to_port     = 65535
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  # All ports, all IPs!
  }
}
```

#### 검증 단계

- [ ] 데이터베이스가 공개 접근 불가
- [ ] SSH/RDP 포트가 VPN/배스천으로만 제한
- [ ] 보안 그룹이 최소 권한 원칙 준수
- [ ] 네트워크 ACL 구성됨
- [ ] VPC 흐름 로그 활성화

### 4. 로깅 및 모니터링

#### CloudWatch/로깅 구성

```typescript
// ✅ CORRECT: Comprehensive logging
import { CloudWatchLogsClient, CreateLogStreamCommand } from '@aws-sdk/client-cloudwatch-logs';

const logSecurityEvent = async (event: SecurityEvent) => {
  await cloudwatch.putLogEvents({
    logGroupName: '/aws/security/events',
    logStreamName: 'authentication',
    logEvents: [{
      timestamp: Date.now(),
      message: JSON.stringify({
        type: event.type,
        userId: event.userId,
        ip: event.ip,
        result: event.result,
        // Never log sensitive data
      })
    }]
  });
};
```

#### 검증 단계

- [ ] 모든 서비스에 CloudWatch/로깅 활성화
- [ ] 인증 실패 시도 기록
- [ ] 관리자 작업 감사
- [ ] 로그 보존 기간 구성 (준수를 위해 90일 이상)
- [ ] 의심스러운 활동에 대한 알림 구성
- [ ] 로그 중앙화 및 변조 방지

### 5. CI/CD 파이프라인 보안

#### 안전한 파이프라인 구성

```yaml
# ✅ CORRECT: Secure GitHub Actions workflow
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read  # Minimal permissions
      
    steps:
      - uses: actions/checkout@v4
      
      # Scan for secrets
      - name: Secret scanning
        uses: trufflesecurity/trufflehog@main
        
      # Dependency audit
      - name: Audit dependencies
        run: npm audit --audit-level=high
        
      # Use OIDC, not long-lived tokens
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/GitHubActionsRole
          aws-region: us-east-1
```

#### 공급망 보안

```json
// package.json - Use lock files and integrity checks
{
  "scripts": {
    "install": "npm ci",  // Use ci for reproducible builds
    "audit": "npm audit --audit-level=moderate",
    "check": "npm outdated"
  }
}
```

#### 검증 단계

- [ ] 장기 자격 증명 대신 OIDC 사용
- [ ] 파이프라인에 시크릿 스캔 포함
- [ ] 의존성 취약점 스캔
- [ ] 컨테이너 이미지 스캔 (해당되는 경우)
- [ ] 브랜치 보호 규칙 시행
- [ ] 병합 전 코드 리뷰 필수
- [ ] 서명된 커밋 강제

### 6. Cloudflare 및 CDN 보안

#### Cloudflare 보안 구성

```typescript
// ✅ CORRECT: Cloudflare Workers with security headers
export default {
  async fetch(request: Request): Promise<Response> {
    const response = await fetch(request);
    
    // Add security headers
    const headers = new Headers(response.headers);
    headers.set('X-Frame-Options', 'DENY');
    headers.set('X-Content-Type-Options', 'nosniff');
    headers.set('Referrer-Policy', 'strict-origin-when-cross-origin');
    headers.set('Permissions-Policy', 'geolocation=(), microphone=()');
    
    return new Response(response.body, {
      status: response.status,
      headers
    });
  }
};
```

#### WAF 규칙

```bash
# Enable Cloudflare WAF managed rules
# - OWASP Core Ruleset
# - Cloudflare Managed Ruleset
# - Rate limiting rules
# - Bot protection
```

#### 검증 단계

- [ ] OWASP 규칙이 포함된 WAF 활성화
- [ ] 속도 제한 구성
- [ ] 봇 보호 활성화
- [ ] DDoS 보호 활성화
- [ ] 보안 헤더 구성
- [ ] SSL/TLS 엄격 모드 활성화

### 7. 백업 및 재해 복구

#### 자동 백업

```terraform
# ✅ CORRECT: Automated RDS backups
resource "aws_db_instance" "main" {
  allocated_storage     = 20
  engine               = "postgres"
  
  backup_retention_period = 30  # 30 days retention
  backup_window          = "03:00-04:00"
  maintenance_window     = "mon:04:00-mon:05:00"
  
  enabled_cloudwatch_logs_exports = ["postgresql"]
  
  deletion_protection = true  # Prevent accidental deletion
}
```

#### 검증 단계

- [ ] 자동 일일 백업 구성
- [ ] 백업 보존 기간이 준수 요건 충족
- [ ] 특정 시점 복구 활성화
- [ ] 분기마다 백업 테스트 수행
- [ ] 재해 복구 계획 문서화
- [ ] RPO 및 RTO 정의 및 테스트 완료

## 프로덕션 클라우드 배포 전 보안 체크리스트

프로덕션 클라우드 배포 전 반드시 확인:

- [ ] **IAM**: 루트 계정 미사용, MFA 활성화, 최소 권한 정책
- [ ] **시크릿**: 모든 시크릿을 교체 기능이 있는 클라우드 시크릿 매니저에 저장
- [ ] **네트워크**: 보안 그룹 제한, 공개 데이터베이스 없음
- [ ] **로깅**: CloudWatch/로깅이 보존 기간과 함께 활성화
- [ ] **모니터링**: 이상 징후에 대한 알림 구성
- [ ] **CI/CD**: OIDC 인증, 시크릿 스캔, 의존성 감사
- [ ] **CDN/WAF**: OWASP 규칙이 있는 Cloudflare WAF 활성화
- [ ] **암호화**: 저장 데이터 및 전송 중 데이터 암호화
- [ ] **백업**: 복구 테스트가 완료된 자동 백업
- [ ] **준수**: GDPR/HIPAA 요건 충족 (해당되는 경우)
- [ ] **문서화**: 인프라 문서화, 런북 작성
- [ ] **사고 대응**: 보안 사고 계획 수립

## 일반적인 클라우드 보안 잘못된 구성

### S3 버킷 노출

```bash
# ❌ WRONG: Public bucket
aws s3api put-bucket-acl --bucket my-bucket --acl public-read

# ✅ CORRECT: Private bucket with specific access
aws s3api put-bucket-acl --bucket my-bucket --acl private
aws s3api put-bucket-policy --bucket my-bucket --policy file://policy.json
```

### RDS 공개 접근

```terraform
# ❌ WRONG
resource "aws_db_instance" "bad" {
  publicly_accessible = true  # NEVER do this!
}

# ✅ CORRECT
resource "aws_db_instance" "good" {
  publicly_accessible = false
  vpc_security_group_ids = [aws_security_group.db.id]
}
```

## 리소스

- [AWS 보안 모범 사례](https://aws.amazon.com/security/best-practices/)
- [CIS AWS Foundations Benchmark](https://www.cisecurity.org/benchmark/amazon_web_services)
- [Cloudflare 보안 문서](https://developers.cloudflare.com/security/)
- [OWASP 클라우드 보안](https://owasp.org/www-project-cloud-security/)
- [Terraform 보안 모범 사례](https://www.terraform.io/docs/cloud/guides/recommended-practices/)

**기억하세요**: 클라우드 잘못된 구성은 데이터 침해의 주요 원인입니다. 단 하나의 노출된 S3 버킷이나 지나치게 허용적인 IAM 정책이 전체 인프라를 위험에 빠뜨릴 수 있습니다. 항상 최소 권한 원칙과 심층 방어를 따르세요.
