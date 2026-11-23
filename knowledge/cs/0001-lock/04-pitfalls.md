---
type: knowledge
created: 2026-11-23
author: agent
model: claude-opus-5-5
---

# 락 사용 시 흔한 함정과 해결 방법

## 쉬운 설명

사람들이 락을 잘못 쓰는 방식은 어느 도구에서나 비슷하다. 다른 건물 문에는 맞지 않는 열쇠를 쓰고, 두 방 열쇠를 사람마다 다른 순서로 집어 서로 막히고, 열쇠를 쥔 채 오래 자리를 비우고, 문을 잠그기 전에 미리 확인한 내용을 믿는다. 그리고 "열쇠가 있으니 안전하다"고 생각하지만, 열쇠는 **열쇠를 쓰기로 약속한 사람만** 막는다.

## 일반 설명

> 이 문서는 솔루션을 가리지 않고 반복되는 실수를 다룬다. 솔루션 고유의 함정(Redis TTL·Redisson, InnoDB 락 범위, 가상 스레드 등)은 [Java 락 — 흔한 함정](../../java/0002-java-lock/04-pitfalls.md), [MySQL 락 — 흔한 함정](../../database/0004-mysql-lock/04-pitfalls.md), [Redis 락 — 흔한 함정](../../database/0005-redis-lock/04-pitfalls.md)에 있다. 대상 자체가 제 기능을 못 하는 경우는 [장애 유형](./05-failure-modes.md).

### 1. 층이 맞지 않는 락 (실수)

| 증상 | 원인 |
|---|---|
| 로컬·서버 1대에서는 정상, 운영(서버 N대)에서 중복 처리·재고 음수 | 프로세스 안 락(`synchronized`)으로 여러 서버 사이를 막으려 함 |
| Redis 락을 잡았는데도 DB 데이터가 틀어짐 | 보호 대상(DB)과 락(Redis)이 다른 곳 — 락이 뚫리는 순간 DB는 모름 |

- **해결**: 상호 배제가 필요한 범위(스레드 / 트랜잭션 / 서버)를 먼저 정하고 그 층의 락을 쓴다. 보호 대상이 DB면 DB 안에서 막는다 → [트레이드오프 — 어느 층에서 막을까](./03-tradeoffs.md#5-주요-선택지-어느-층에서-막을까).
- 솔루션별: [Java — Spring·JPA와 섞일 때](../../java/0002-java-lock/04-pitfalls.md#6-springjpa와-섞일-때).

### 2. 락 순서가 제각각 (실수)

```
요청 1: lock(A) → lock(B)
요청 2: lock(B) → lock(A)     → 서로 상대가 쥔 락을 기다림 = 데드락
```

- 스레드 락(계좌 이체), DB 행 락(두 행을 다른 순서로 UPDATE), 분산 락(여러 키) 모두 같은 원리다.
- **해결**: 여러 락은 **항상 같은 순서**(예: ID 오름차순)로 잡는다. 한 번에 다 잡을 수 없으면 `tryLock(timeout)`으로 실패 시 모두 놓고 백오프한다.
- 솔루션별: [MySQL — 데드락을 만드는 패턴](../../database/0004-mysql-lock/04-pitfalls.md#3-데드락을-만드는-패턴), [Java 장애 — 예방 설계](../../java/0002-java-lock/05-failure-modes.md#4-예방-설계).

### 3. 락을 쥔 채 느린 일을 함 (안티패턴)

- 락 안에서 외부 API 호출, 파일·네트워크 I/O, 사용자 입력 대기, 무거운 계산.
- **영향**: 처리량 상한 = 1 / 임계 구역 시간이라, 임계 구역이 100ms가 되면 그 락은 초당 10번이 한계다. 외부 호출이 멈추면 대기자가 쌓여 스레드 풀·커넥션 풀이 고갈된다.
- **해결**: 락 밖에서 준비하고, 락 안에서는 **확인과 쓰기만** 한다. 모든 I/O에 타임아웃을 둔다. DB에서는 트랜잭션 안에서 외부 호출을 하지 않는다.
- 솔루션별: [MySQL — 락을 오래 쥐는 실수](../../database/0004-mysql-lock/04-pitfalls.md#2-락을-오래-쥐는-실수).

### 4. 잠그기 전에 확인한 값을 믿음 (실수)

```
if (stock > 0) {        ← 락 없이 읽음
    lock()
    stock = stock - 1   ← 그 사이 0이 됐을 수 있음
    unlock()
}
```

- DB에서는 일반 `SELECT`(스냅샷)로 읽고 판단한 뒤 `UPDATE`하는 경우, Redis에서는 `GET` 후 `SET`, 락에서는 `GET` 후 `DEL`이 같은 실수다.
- **해결**: 확인과 실행을 **같은 락 안**에서 하거나, 하나의 원자 연산으로 합친다 (`UPDATE ... WHERE stock > 0`, `SELECT ... FOR UPDATE`, Lua 스크립트, CAS).
- 솔루션별: [Redis — `GET` 후 `DEL`](../../database/0005-redis-lock/04-pitfalls.md#3-get-후-del--비교와-삭제-사이의-틈-실수), [Java — 경쟁 조건](../../java/0002-java-lock/04-pitfalls.md#1-경쟁-조건-race-condition).

### 5. 해제 누락과 남의 락 해제 (실수)

| 실수 | 결과 | 해결 |
|---|---|---|
| 예외 경로에서 해제 안 함 | 락이 영원히 남음 (JVM) / TTL까지 막힘 (Redis) | `try-finally`, try-with-resources, 블록 구조(`synchronized`) |
| 소유 확인 없이 해제 | 내 락이 만료된 뒤 **남이 잡은 락**을 지움 | 해제 시 소유 토큰 비교를 원자적으로 |
| 획득하지 못했는데 `finally`에서 해제 | 예외 또는 남의 락 해제 | 획득 성공 여부를 확인한 뒤 해제 |

- 솔루션별: [Java — 락 해제·사용 실수](../../java/0002-java-lock/04-pitfalls.md#4-락-해제사용-실수), [Redis — 토큰 없이 해제](../../database/0005-redis-lock/04-pitfalls.md#2-토큰-없이-해제--남의-락을-지움-실수).

### 6. 락과 트랜잭션 경계가 어긋남 (실수)

```
@Transactional
void order() {
    lock()          ← JVM·Redis 락
    재고 차감 UPDATE
    unlock()        ← 여기서 풀림
}                   ← 커밋은 여기서 → 그 사이 다른 요청이 커밋 전 값을 읽음
```

- 원인: DB는 커밋까지 락을 쥐는데(2PL), 바깥 락은 블록 끝에서 풀린다 → [개요 — 2단계 락](./02-overview.md#10-2단계-락-2pl).
- **해결**: 바깥 락을 **트랜잭션 바깥**에서 잡고 커밋 뒤에 풀거나, DB 수준 제어(원자적 UPDATE, `FOR UPDATE`, `@Version`)만 쓴다.
- 솔루션별: [Redis — `@Transactional` 안에서 락 해제](../../database/0005-redis-lock/04-pitfalls.md#8-transactional-안에서-락-해제--커밋-전에-풀림-실수).

### 7. "락이 있으니 안전하다" (오해)

| 오해 | 실제 |
|---|---|
| 락이 데이터를 보호한다 | 락은 **락을 잡는 코드끼리만** 막는다. 이름 락(Redis 키, `GET_LOCK`)을 모르는 코드·배치·운영자 SQL은 그대로 데이터를 고친다 |
| 분산 락이면 동시 실행은 없다 | lease 기반 락은 보유자가 멈추면 **두 보유자**가 생긴다 → [개요 — 해제 보장과 lease](./02-overview.md#7-해제-보장과-lease) |
| 락이 조건만큼만 잠근다 | DB는 스캔한 범위·이웃 레코드까지 잠근다 → [MySQL 락 범위](../../database/0004-mysql-lock/02-overview.md#4-락-범위는-어떻게-정해지나-repeatable-read) |
| 안전한 연산 둘을 이으면 안전하다 | `containsKey` + `put`처럼 각각 원자적인 연산 둘은 합쳐서 원자적이지 않다 → [Java — 동시성 컬렉션의 함정](../../java/0002-java-lock/04-pitfalls.md#7-동시성-컬렉션의-함정) |

### 8. 단위를 잘못 고름 (안티패턴)

| 잘못 | 증상 | 해결 |
|---|---|---|
| 너무 굵게 (전역 락, 테이블 락, `lock:orders`) | 서로 무관한 요청까지 직렬화, 처리량이 서버 수와 무관 | 보호 대상 단위로 (`lock:order:42`, 행 락) |
| 너무 잘게 (필드마다, 키마다 여러 개) | 여러 락을 잡다 데드락, 락 관리 비용 | 함께 바뀌는 데이터는 한 락으로. 락 스트라이핑 |

- 솔루션별: [Redis — 락 키를 너무 굵게 또는 너무 가늘게](../../database/0005-redis-lock/04-pitfalls.md#11-락-키를-너무-굵게-또는-너무-가늘게-실수).

### 9. 대기 정책이 없음 (실수)

| 실수 | 결과 | 해결 |
|---|---|---|
| 타임아웃 없는 무한 대기 | 보유자가 멈추면 대기자 전체가 멈춤 | 획득 타임아웃, DB `innodb_lock_wait_timeout`을 API 타임아웃보다 짧게 |
| 실패 즉시 쉬지 않고 재시도 | 재시도 폭풍 → 락 저장소·DB 부하 | 지수 백오프 + 지터, 재시도 횟수 제한 |
| 낙관적 락을 고경합에 사용 | 재시도가 성공보다 많아짐 | 경합 지점은 비관적 락이나 큐로 직렬화 |

- 솔루션별: [Redis — 재시도를 쉬지 않고 돌림](../../database/0005-redis-lock/04-pitfalls.md#10-재시도를-쉬지-않고-돌림-안티패턴), [MySQL — 오류를 잘못 다루는 실수](../../database/0004-mysql-lock/04-pitfalls.md#4-오류를-잘못-다루는-실수).

### 10. 락이 필요 없는 곳에 락 (안티패턴)

- 원자 연산 하나로 되는 카운터·재고 차감에 락, 읽기 전용 데이터에 락, 같은 키를 한 작업자만 처리하면 되는 일에 분산 락.
- **해결**: [트레이드오프 — 어느 층에서 막을까](./03-tradeoffs.md#5-주요-선택지-어느-층에서-막을까)의 위쪽 질문(공유 제거 → 원자 연산)부터 확인한다.

### 11. 점검 체크리스트

- [ ] 상호 배제가 필요한 범위와 락의 층이 같은가
- [ ] 여러 락을 잡는 곳은 순서가 정해져 있는가
- [ ] 락 안에 I/O·외부 호출이 없는가, 있다면 타임아웃이 있는가
- [ ] 확인과 쓰기가 같은 락 안 또는 하나의 원자 연산인가
- [ ] 모든 경로에서 해제되고, 해제 전에 소유를 확인하는가
- [ ] 락 해제 시점이 트랜잭션 커밋 뒤인가
- [ ] 정확성이 필요하면 락 말고도 보호 대상 쪽 장치(제약, fencing, 멱등키)가 있는가
- [ ] 대기 타임아웃과 백오프가 있는가

## 관련 문서

- [락의 역사](./01-history.md)
- [락 개요와 공통 구조](./02-overview.md)
- [락 트레이드오프와 선택 기준](./03-tradeoffs.md)
- [락 장애 유형과 대응](./05-failure-modes.md)
- [Java 락 — 흔한 함정](../../java/0002-java-lock/04-pitfalls.md)
- [MySQL 락 — 흔한 함정](../../database/0004-mysql-lock/04-pitfalls.md)
- [Redis 락 — 흔한 함정](../../database/0005-redis-lock/04-pitfalls.md)

## 참고 자료

- [Martin Kleppmann — How to do distributed locking](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)
- [MySQL 8.4 Reference Manual — How to Minimize and Handle Deadlocks](https://dev.mysql.com/doc/refman/8.4/en/innodb-deadlocks-handling.html)
- [Java Concurrency — Oracle Tutorials: Liveness](https://docs.oracle.com/javase/tutorial/essential/concurrency/liveness.html)
