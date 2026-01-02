# TRD (Technical Requirements Document) Skill

PRD를 기반으로 기술적 구현 방법과 아키텍처를 정의하는 워크플로우.

## 핵심 원칙

**기술적 실현 가능성과 구현 방법에 집중한다.** 비즈니스 요구사항은 PRD에서 확정된 것을 따른다.

## TRD 작성 프로세스

### Phase 1: 아키텍처 설계

1. "시스템 아키텍처 패턴은 무엇으로 할까요?" (모놀리식/마이크로서비스/서버리스)
2. "주요 컴포넌트 간 통신 방식은?" (REST/GraphQL/gRPC/메시지 큐)
3. "데이터 흐름을 설명해주세요"

### Phase 2: 기술 스택 결정

1. "프론트엔드 기술 스택은?"
2. "백엔드 기술 스택은?"
3. "데이터베이스 선택은?" (RDB/NoSQL/시계열 등)
4. "인프라/클라우드 환경은?"

### Phase 3: API 설계

1. "주요 API 엔드포인트를 정의해주세요"
2. "인증/인가 방식은?"
3. "에러 처리 전략은?"
4. "API 버저닝 전략은?"

### Phase 4: 데이터 모델링

1. "핵심 엔티티는 무엇인가요?"
2. "엔티티 간 관계를 설명해주세요"
3. "인덱싱 전략은?"

### Phase 5: 비기능 요구사항 구현

1. "성능 최적화 전략은?" (캐싱, CDN, 로드밸런싱)
2. "보안 구현 방법은?" (암호화, 인증, 취약점 방지)
3. "모니터링/로깅 전략은?"
4. "백업/복구 전략은?"

## TRD 질문 규칙

- 기술적 의사결정의 근거를 명시한다
- 트레이드오프를 분석한다
- 확장성과 유지보수성을 고려한다
- 구체적인 버전과 라이브러리를 명시한다

## TRD 출력 형식

```markdown
# [제품명] TRD (Technical Requirements Document)

## 1. 개요

### 1.1 문서 목적
- PRD 참조: [PRD 링크]
- 기술적 범위:

### 1.2 기술 원칙
- 원칙 1:
- 원칙 2:

## 2. 시스템 아키텍처

### 2.1 아키텍처 개요
```
[아키텍처 다이어그램 - ASCII 또는 Mermaid]
```

### 2.2 아키텍처 결정 기록 (ADR)
| 결정 | 선택 | 대안 | 근거 |
|------|------|------|------|
| 아키텍처 패턴 | 모놀리식 | 마이크로서비스 | 초기 복잡도 최소화 |
| ... | ... | ... | ... |

### 2.3 컴포넌트 구성
| 컴포넌트 | 역할 | 기술 스택 |
|----------|------|-----------|
| Frontend | UI/UX | React, TypeScript |
| Backend | API 서버 | Node.js, Express |
| Database | 데이터 저장 | PostgreSQL |
| ... | ... | ... |

## 3. 기술 스택

### 3.1 Frontend
- Framework: React 18.x
- Language: TypeScript 5.x
- Styling: Tailwind CSS 3.x
- State Management: Zustand / Redux Toolkit
- 빌드 도구: Vite

### 3.2 Backend
- Runtime: Node.js 20 LTS
- Framework: Express / Fastify
- ORM: Prisma / TypeORM
- Validation: Zod / Joi

### 3.3 Database
- Primary: PostgreSQL 15
- Cache: Redis 7
- Search: Elasticsearch (필요시)

### 3.4 Infrastructure
- Cloud: AWS / GCP / Azure
- Container: Docker
- Orchestration: Kubernetes / ECS
- CI/CD: GitHub Actions

## 4. API 설계

### 4.1 API 스타일
- 스타일: REST / GraphQL
- 버저닝: URL 기반 (/api/v1)
- 인증: JWT Bearer Token

### 4.2 주요 엔드포인트
| Method | Endpoint | 설명 | 인증 |
|--------|----------|------|------|
| POST | /api/v1/auth/login | 로그인 | N |
| GET | /api/v1/users/me | 내 정보 조회 | Y |
| ... | ... | ... | ... |

### 4.3 에러 응답 형식
```json
{
  "error": {
    "code": "ERROR_CODE",
    "message": "Human readable message",
    "details": {}
  }
}
```

### 4.4 공통 응답 형식
```json
{
  "success": true,
  "data": {},
  "meta": {
    "page": 1,
    "limit": 20,
    "total": 100
  }
}
```

## 5. 데이터 모델

### 5.1 ERD
```
[ERD 다이어그램 - Mermaid 형식]
```

### 5.2 주요 엔티티
| 엔티티 | 설명 | 주요 필드 |
|--------|------|-----------|
| User | 사용자 | id, email, name, created_at |
| ... | ... | ... |

### 5.3 인덱스 전략
| 테이블 | 인덱스 | 타입 | 용도 |
|--------|--------|------|------|
| users | email | UNIQUE | 로그인 조회 |
| ... | ... | ... | ... |

## 6. 보안 설계

### 6.1 인증/인가
- 인증 방식: JWT + Refresh Token
- 토큰 유효기간: Access 15분, Refresh 7일
- 권한 모델: RBAC

### 6.2 데이터 보호
- 전송 암호화: TLS 1.3
- 저장 암호화: AES-256 (민감 데이터)
- 비밀번호: bcrypt (cost factor 12)

### 6.3 보안 체크리스트
- [ ] SQL Injection 방지 (Parameterized Query)
- [ ] XSS 방지 (Output Encoding)
- [ ] CSRF 방지 (Token)
- [ ] Rate Limiting

## 7. 성능 요구사항

### 7.1 성능 목표
| 지표 | 목표 | 측정 방법 |
|------|------|-----------|
| API 응답 시간 | < 200ms (p95) | APM |
| 페이지 로드 | < 3초 | Lighthouse |
| 동시 사용자 | 1,000명 | 부하 테스트 |

### 7.2 최적화 전략
- 캐싱: Redis (세션, API 응답)
- CDN: 정적 자산
- DB: 커넥션 풀링, 쿼리 최적화

## 8. 인프라 구성

### 8.1 환경 구성
| 환경 | 용도 | 구성 |
|------|------|------|
| Development | 개발 | 로컬 Docker |
| Staging | QA/테스트 | 축소 프로덕션 |
| Production | 서비스 | 고가용성 구성 |

### 8.2 배포 전략
- 방식: Blue-Green / Rolling
- 롤백: 자동 롤백 조건 정의

### 8.3 모니터링
- Metrics: Prometheus + Grafana
- Logging: ELK Stack / CloudWatch
- Alerting: PagerDuty / Slack

## 9. 테스트 전략

### 9.1 테스트 피라미드
| 레벨 | 도구 | 커버리지 목표 |
|------|------|---------------|
| Unit | Jest/Vitest | 80% |
| Integration | Supertest | 주요 흐름 |
| E2E | Playwright | 핵심 시나리오 |

### 9.2 테스트 환경
- CI 테스트: GitHub Actions
- 테스트 DB: Docker 컨테이너

## 10. 의존성 및 리스크

### 10.1 외부 의존성
| 서비스 | 용도 | 대안 |
|--------|------|------|
| AWS S3 | 파일 저장 | GCS, MinIO |
| ... | ... | ... |

### 10.2 기술적 리스크
| 리스크 | 영향 | 완화 방안 |
|--------|------|-----------|
| DB 병목 | 높음 | 읽기 복제본, 캐싱 |
| ... | ... | ... |
```

## 워크플로우 예시

```
사용자: "쇼핑몰 TRD 작성해줘"

Claude: "PRD 기반으로 기술 요구사항을 정리하겠습니다.

1. 시스템 아키텍처는 어떻게 구성할까요?
   - 모놀리식: 초기 빠른 개발, 단순한 배포
   - 마이크로서비스: 확장성, 독립 배포
   - 하이브리드: 핵심은 모놀리식, 특정 기능 분리"

사용자: "처음이라 모놀리식으로 시작하고 싶어요"

Claude: "좋은 선택입니다. 추가 질문입니다.

2. 백엔드 기술 스택 선호도가 있나요?
   - Node.js: JavaScript 생태계 활용
   - Python: 데이터 처리에 강점
   - Go: 고성능이 필요할 때"

... (인터뷰 계속) ...
```

## TRD 완료 후

TRD 확정 후 다음 단계 제안:

1. "TASKS로 구현 작업을 분해할까요?"
2. "특정 컴포넌트에 대해 더 상세한 설계가 필요한가요?"
3. "프로토타입 구현을 시작할까요?"
