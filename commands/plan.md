---
description: 요구사항을 재확인하고, 위험을 평가하며, 단계별 구현 계획을 작성합니다. 코드를 건드리기 전에 사용자 확인을 기다립니다.
---

# Plan 커맨드

이 커맨드는 **planner** 에이전트를 호출하여 코드를 작성하기 전에 포괄적인 구현 계획을 만듭니다.

## 이 커맨드가 하는 일

1. **요구사항 재확인** - 무엇을 만들어야 하는지 명확히 합니다
2. **위험 식별** - 잠재적 이슈와 차단 요소를 도출합니다
3. **단계별 계획 작성** - 구현을 단계별로 분해합니다
4. **확인 대기** - 진행하기 전에 반드시 사용자 승인을 받아야 합니다

## 사용 시점

`/plan`을 사용해야 할 때:
- 새 기능을 시작할 때
- 중요한 아키텍처 변경을 할 때
- 복잡한 리팩토링 작업을 할 때
- 여러 파일/컴포넌트에 영향을 미칠 때
- 요구사항이 불명확하거나 모호할 때

## 작동 방식

planner 에이전트가 수행하는 작업:

1. 요청을 **분석**하고 요구사항을 명확한 용어로 재확인합니다
2. 구체적이고 실행 가능한 단계로 **분해**합니다
3. 컴포넌트 간 **의존성을 식별**합니다
4. **위험을 평가**하고 잠재적 차단 요소를 파악합니다
5. **복잡도를 추정**합니다 (High/Medium/Low)
6. 계획을 **제시**하고 명시적 확인을 **대기**합니다

## 사용 예시

```
User: /plan I need to add real-time notifications when markets resolve

Agent (planner):
# Implementation Plan: Real-Time Market Resolution Notifications

## Requirements Restatement
- Send notifications to users when markets they're watching resolve
- Support multiple notification channels (in-app, email, webhook)
- Ensure notifications are delivered reliably
- Include market outcome and user's position result

## Implementation Phases

### Phase 1: Database Schema
- Add notifications table with columns: id, user_id, market_id, type, status, created_at
- Add user_notification_preferences table for channel preferences
- Create indexes on user_id and market_id for performance

### Phase 2: Notification Service
- Create notification service in lib/notifications.ts
- Implement notification queue using BullMQ/Redis
- Add retry logic for failed deliveries
- Create notification templates

### Phase 3: Integration Points
- Hook into market resolution logic (when status changes to "resolved")
- Query all users with positions in market
- Enqueue notifications for each user

### Phase 4: Frontend Components
- Create NotificationBell component in header
- Add NotificationList modal
- Implement real-time updates via Supabase subscriptions
- Add notification preferences page

## Dependencies
- Redis (for queue)
- Email service (SendGrid/Resend)
- Supabase real-time subscriptions

## Risks
- HIGH: Email deliverability (SPF/DKIM required)
- MEDIUM: Performance with 1000+ users per market
- MEDIUM: Notification spam if markets resolve frequently
- LOW: Real-time subscription overhead

## Estimated Complexity: MEDIUM
- Backend: 4-6 hours
- Frontend: 3-4 hours
- Testing: 2-3 hours
- Total: 9-13 hours

**WAITING FOR CONFIRMATION**: Proceed with this plan? (yes/no/modify)
```

## 중요 참고 사항

**핵심**: planner 에이전트는 "yes"나 "proceed" 같은 긍정적 응답으로 명시적으로 계획을 확인하기 전까지 코드를 **절대 작성하지 않습니다.**

변경을 원하면 다음과 같이 응답하세요:
- "modify: [변경 사항]"
- "different approach: [대안]"
- "skip phase 2 and do phase 3 first"

## 다른 커맨드와의 연계

계획 수립 후:
- `/tdd`를 사용하여 테스트 주도 개발로 구현
- 빌드 에러 발생 시 `/build-fix` 사용
- 완성된 구현을 `/code-review`로 리뷰

## 관련 에이전트

이 커맨드는 다음 위치의 `planner` 에이전트를 호출합니다:
`~/.claude/agents/planner.md`
