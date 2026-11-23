---
type: inquiry
created: 2026-11-23
author: agent
model: claude-opus-5-5
---

# 서버를 프로덕션에 배포하기 전에 하는 테스트 방법

## 쉬운 설명

새 다리를 개통하기 전에는 부품 하나하나를 검사하고(단위 테스트), 부품을 이어 붙여 보고(통합 테스트), 똑같은 다리를 시험장에 지어 트럭을 잔뜩 올려 보고(스테이징·부하 테스트), 마지막으로 차선 하나만 먼저 열어 본다(카나리). 앞 단계일수록 싸고 빠르게 많이 검사하고, 뒤로 갈수록 실제와 비슷하지만 비싸고 느리다.

## 일반 설명

### 1. 전체 흐름

```mermaid
flowchart LR
    c["코드·커밋<br/>정적 분석, 단위"] --> ci["CI<br/>통합, 계약, 보안 스캔"]
    ci --> st["스테이징<br/>E2E, 마이그레이션 리허설, DAST"]
    st --> pf["성능·안정성<br/>부하, 내구, 카오스"]
    pf --> pr["배포 직전·직후<br/>섀도, 카나리, 스모크"]
```

- 배포 전 테스트는 **알려진 위험**을 걸러 내고, 카나리·모니터링은 **운영에서만 드러나는 위험**을 확인한다. 둘은 대체 관계가 아니다. "운영에서 확인하면 된다"며 배포 전 테스트를 건너뛰는 것은 대표적인 안티패턴이다.
- 테스트는 **피라미드** 비중으로 둔다. 아래(단위)는 빠르고 많이, 위(E2E)는 느리고 적게. E2E가 많으면 느리고 자주 흔들리는(flaky) 테스트가 쌓여 결과를 아무도 믿지 않게 된다.

### 2. 코드·커밋 단계 (몇 초~몇 분)

| 테스트 | 확인하는 것 | 도구 예 |
|---|---|---|
| 정적 분석·린트 | 코드 스타일, 흔한 버그 패턴, null 위험 | SonarQube, ErrorProne, ESLint, ktlint |
| **단위 테스트** | 함수·클래스 하나의 로직. 외부 의존성은 가짜(mock)로 대체 | JUnit 5, Kotest, Mockito, pytest |
| 커버리지 | 테스트가 지나가지 않은 코드 | JaCoCo |
| 뮤테이션 테스트 (선택) | 코드를 일부러 바꿨을 때 테스트가 실패하는지, 즉 테스트가 제 역할을 하는지 | PIT |

### 3. CI 단계 (몇 분~수십 분)

| 테스트 | 확인하는 것 | 도구 예 |
|---|---|---|
| **통합 테스트** | 실제 DB·Redis·Kafka와 붙여 쿼리, 트랜잭션, 직렬화가 맞는지 | **Testcontainers**, `@SpringBootTest` |
| API 테스트 | 엔드포인트별 상태 코드, 응답 형식, 오류 응답 | MockMvc, REST Assured |
| **계약 테스트** | 서비스 사이 API 모양이 맞는지. 두 서비스를 함께 띄우지 않고 검증 | Pact, Spring Cloud Contract |
| 아키텍처 테스트 | 계층 규칙 위반 (예: domain이 infrastructure 참조) | ArchUnit |
| 보안 스캔 | 취약한 라이브러리(SCA), 코드 취약점(SAST), 비밀값 커밋 | Dependabot, Trivy, Semgrep, gitleaks |
| 이미지 스캔 | 컨테이너 베이스 이미지 취약점 | Trivy, Grype |

- 통합 테스트에 H2 같은 인메모리 DB를 쓰면 운영 DB 전용 문법, 마이그레이션 문법, 락 동작 차이를 놓친다. 수백 개의 H2 테스트를 통과한 서비스가 첫 배포에서 PostgreSQL 전용 마이그레이션 문법 때문에 실패한 사례가 있다. **운영과 같은 DB 버전을 Testcontainers로** 띄우는 것이 표준이다.
- 외부 응답 전체를 기준 파일과 비교하는 방식은 [골든 테스트](./0006-golden-test-golden-master-testing.md), "정답을 어떻게 아나"의 문제는 [테스트 오라클](./0007-test-oracle-meaning-and-oracle-problem.md) 참고.

### 4. 스테이징 단계 (운영과 같은 구성의 환경)

| 테스트 | 확인하는 것 | 도구 예 |
|---|---|---|
| **E2E 테스트** | 로그인 → 주문 → 결제 같은 사용자 흐름 전체 | Playwright, Karate, Postman/Newman |
| 회귀 테스트 | 기존 기능이 깨지지 않았는지 (위 묶음 전체 재실행) | CI 파이프라인 |
| **DB 마이그레이션 리허설** | 운영 크기 데이터 복사본에서 변경 시간, 락, 롤백 가능 여부 | Flyway, Liquibase, gh-ost, pt-online-schema-change |
| 설정·시크릿 검증 | 환경 변수, 연결 문자열, 권한(IAM)이 환경마다 맞는지 | 기동 시 health check |
| DAST·침투 테스트 | 실행 중인 서버를 외부에서 공격 (인증 우회, 인젝션) | OWASP ZAP, Burp Suite |
| UAT (사용자 인수 테스트) | 기획·현업이 요구사항대로 동작하는지 확인 | 수동 |

- 스테이징이 운영과 다를수록(버전, 설정, 데이터 크기, 네트워크 구성) 여기서 통과한 결과의 가치가 떨어진다.
- 락 범위는 데이터 분포에 따라 달라지므로, 소량 데이터의 개발 DB에서는 락 대기·데드락이 잘 드러나지 않는다 ([MySQL 락 개요 — 락 범위는 어떻게 정해지나](../../knowledge/database/0004-mysql-lock/02-overview.md)). 운영 크기 데이터로 리허설하는 이유다.

### 5. 성능·안정성 테스트

| 테스트 | 질문 | 방식 |
|---|---|---|
| **부하 (Load)** | 예상 트래픽에서 응답 시간·오류율이 목표(SLO) 안인가? | 평소 피크의 1~1.5배를 일정 시간 |
| 스트레스 | 어디까지 버티고, 넘으면 어떻게 무너지나? | 한계까지 계속 증가 |
| 스파이크 | 갑자기 몰리는 트래픽(이벤트, 푸시 직후)을 견디나? | 순간 몇 배 투입 |
| **내구 (Soak)** | 몇 시간~며칠 돌려도 메모리 누수·커넥션 고갈·디스크 증가가 없나? | 보통 부하로 장시간 |
| 용량 | 데이터가 1년 뒤 크기가 되어도 쿼리가 버티나? | 대량 데이터 사전 적재 |

- 도구: k6, Gatling, JMeter, Locust, nGrinder.
- 지연 시간을 정확히 보려면 고정 요청률 방식(wrk2, k6 `constant-arrival-rate`)을 쓴다. 응답을 기다린 뒤 다음 요청을 보내는 방식은 느려진 구간의 요청을 덜 보내 지연을 과소 측정한다 (coordinated omission, [wrk vs wrk2](./0002-wrk-vs-wrk2-difference.md)).
- 내구 테스트 자세한 내용은 [장기 테스트(soak)](./0001-long-term-test-soak-endurance-testing.md).

| 장애 테스트 | 확인하는 것 | 도구 예 |
|---|---|---|
| 카오스 테스트 | DB·외부 API가 죽거나 느려질 때 타임아웃·재시도·서킷 브레이커가 동작하나 | Chaos Mesh, Toxiproxy, AWS FIS |
| 장애 조치 (Failover) | 주 DB가 죽으면 복제본으로 넘어가고 유실이 없나 | 수동 리허설 |
| 백업 복구 (DR) | 백업에서 실제로 복구되나, 얼마나 걸리나 | 정기 복구 훈련 |

### 6. 배포 직전·직후 (운영 환경에서 제한적으로)

| 방법 | 확인하는 것 | 주의 |
|---|---|---|
| **섀도 트래픽 (미러링)** | 운영 요청을 복제해 새 버전에도 보내고 응답 비교. 사용자 영향 없음 | 쓰기 경로(DB 저장, 메일, 결제)를 막거나 격리하지 않으면 운영 데이터가 오염된다 |
| **카나리 배포** | 트래픽 1~5%만 새 버전으로 보내 오류율·지연 비교 | 자동 롤백 기준을 미리 정한다 |
| 블루-그린 배포 | 새 환경을 통째로 띄운 뒤 트래픽 전환 | 되돌리기 쉽지만 신·구 버전이 같은 DB 스키마에서 동작해야 한다 |
| 기능 플래그 | 배포와 기능 공개를 분리, 일부 사용자에게만 켜기 | 다 쓴 플래그는 정리한다 |
| **스모크 테스트** | 배포 직후 핵심 API 몇 개가 정상인지 | 자동화하고 실패 시 즉시 롤백 |
| 합성 모니터링 | 가짜 사용자가 주기적으로 핵심 흐름 호출 | 테스트 트래픽이 통계에 섞이지 않게 표시 |

### 7. 배포 전 체크리스트

- [ ] 단위·통합·계약 테스트 통과, 커버리지 기준 충족
- [ ] 보안 스캔 Critical·High 0건
- [ ] 스테이징 E2E·회귀 통과
- [ ] 마이그레이션을 운영 크기 데이터로 리허설, 롤백 스크립트 준비
- [ ] 부하 테스트 결과가 SLO 안 (예: p99 < 300ms, 오류율 < 0.1%)
- [ ] 타임아웃·재시도·서킷 브레이커를 장애 주입으로 확인
- [ ] 모니터링 대시보드·알림 준비 (새 기능 지표 포함)
- [ ] 카나리 롤백 기준과 담당자 지정
- [ ] 배포 후 스모크 테스트 자동화

### 8. 규모별 최소 구성

| 규모 | 최소 구성 |
|---|---|
| 개인·초기 서비스 | 단위 테스트, Testcontainers 통합 테스트, 배포 후 스모크 테스트 |
| 팀 서비스 | 위 + 계약 테스트, 스테이징 E2E, 부하 테스트, 보안 스캔 |
| 대규모·금융 | 위 + 카오스 테스트, 섀도 트래픽, 카나리 자동 롤백, DR 훈련, 침투 테스트 |

## 출처

- [Docker — Shift-Left Testing with Testcontainers](https://www.docker.com/blog/shift-left-testing-with-testcontainers/)
- [DEV — Real infrastructure testing with Testcontainers](https://dev.to/ximanta/stop-lying-to-your-tests-real-infrastructure-testing-with-testcontainers-spring-boot-4-pl9)
- [The New Stack — Microservice integration testing a pain? Try shadow testing](https://thenewstack.io/microservice-integration-testing-a-pain-try-shadow-testing/)
- [OneUptime — Shadow deployments: mirror production traffic](https://oneuptime.com/blog/post/2026-02-09-shadow-deployments-mirror-production-traffic/view)
- [InfoQ — Microservices traffic mirroring with Istio](https://www.infoq.com/articles/microservices-traffic-mirroring-istio-vpc)
- [BEE-15006 — Testing in Production](https://alivedise.github.io/backend-engineering-essentials/testing/testing-in-production)
- [DEV — Feature flags, canary releases, observability-driven testing](https://dev.to/therizwansaleem/testing-in-production-feature-flags-canary-releases-and-observability-driven-testing-1lo5)
- [Software Test Pilot — Microservices testing strategy](https://softwaretestpilot.com/blog/api-testing/microservices-testing-strategy)
- [ACCELQ — Types of testing](https://accelq.com/blog/types-of-testing/)
- [Ardura — Are your pre-deployment tests thorough enough?](https://ardura.consulting/blog/are-your-pre-deployment-tests-thorough-enough/)
- [N-iX — Software testing best practices](https://n-ix.com/software-testing-best-practices)
- [Microsoft — Types of tests](https://learn.microsoft.com/en-us/dynamics365/guidance/implementation-guide/testing-strategy-test-types)
