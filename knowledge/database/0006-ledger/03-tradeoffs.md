---
type: knowledge
created: 2026-11-24
author: agent
model: claude-opus-5-5
---

# 원장 트레이드오프와 선택 기준

## 쉬운 설명

기입장은 **지우개를 포기**했다. 틀린 줄도 지우지 못하고 반대 줄을 하나 더 쓴다. 그리고 내 것이 아닌 곳까지 칸을 만들어 적느라 **쓸 줄이 두 배**가 됐다. 대신 두 사람이 같은 쪽을 보고 동시에 다음 줄을 쓰려 하면, 같은 줄 번호에 두 줄을 쓸 수 없어서 한 명은 다시 써야 한다. 기다리는 줄(락) 대신 줄 번호가 순서를 정한다.

## 일반 설명

> 구조와 용어는 [개요](./02-overview.md)([용어](./02-overview.md#2-용어)), 고른 뒤의 실수는 [흔한 함정](./04-pitfalls.md), 장애는 [장애 유형](./05-failure-modes.md). 이 문서는 "왜 이렇게 만들었고, 대가는 무엇이고, 언제 무엇을 고르나"만 다룬다.

### 1. 핵심 설계 선택: 얻은 것과 포기한 것

| 설계 | 얻은 것 | 포기한 것 |
|---|---|---|
| **덧붙이기만** (원장의 전제) | 이력이 곧 근거. 어느 시점 잔액이든 다시 계산할 수 있다. 감사·분쟁 대응 | 저장 공간이 계속 는다. 정정 한 번에 거래 2개(반대 거래 + 올바른 거래) |
| **복식부기** | 거래마다 차변 합 = 대변 합 검사. 시스템 계정 잔액으로 외부와 대사 | 기입 수가 두 배. 시스템 계정이 모든 거래에 끼어 **핫 계정**이 된다 |
| **차변/대변 + 양수 금액** | 시스템 계정의 증감 방향이 회계와 같아 헷갈리지 않는다 | 잔액을 구할 때 정상잔액을 알아야 한다. 부호(+/−) 한 열보다 계산이 한 단계 많다 |
| **기입에 거래 후 잔액** | 현재 잔액이 행 하나. 음수 금지를 `CHECK`로 | 한 계정의 쓰기가 **직렬화**된다. 마지막 기입을 알아야 다음 기입을 쓴다 |
| **대기 → 확정** | 외부 결과를 기다리는 동안에도 잔액을 정확히 잡는다(홀드) | 상태 전이·만료 처리가 필요하다. 대기가 쌓이면 사용 가능 잔액이 묶인다 |

거래마다 차변 합 = 대변 합이라는 규칙은 여러 행에 걸친 조건이라 **`CHECK`로 걸 수 없다.** 애플리케이션이 거래 단위로 검사하고, 주기적으로 시산표(전체 차변 합 = 대변 합)로 확인한다.

### 2. 단식부기 vs 복식부기

| | 단식부기 | 복식부기 |
|---|---|---|
| 거래 하나의 기입 | 고객 계정만 (입출금 1행, 이체 2행) | 관련 계정 모두 (입출금도 2행 이상) |
| 바깥(은행, PG)의 기록 | 거래의 속성(`source`, `target`) | 시스템 계정의 기입 |
| 외부와 대사 | 거래를 따로 집계해 비교 | 자금·청산 계정 잔액을 바로 비교 |
| 값이 새거나 생기는 오류 | 탐지 수단 없음 | 시산표가 맞지 않으면 드러남 |
| 수수료·환불·부분 정산 | 속성이 늘어나며 복잡해진다 | 계정 하나 더 추가 |
| 비용 | 적다 | 기입 두 배, 핫 계정 대응 |

**고르는 기준**: 외부 기관과 돈·수량을 주고받고 그 결과를 맞춰 봐야 하면 복식부기다. 결제·지갑·마켓플레이스는 처음부터 복식부기로 시작한다. 나중에 바꾸려면 과거 거래의 시스템 계정 기입을 모두 다시 만들어야 한다 → [흔한 함정 5절](./04-pitfalls.md#5-시스템-계정-없이-시작). 외부와 맞출 일이 없는 내부 수량(게임 내 재화 등)이라면 단식부기로도 충분하다.

### 3. 동시성 제어: 분산 락이 뚫려도 정확하게

분산 락은 장애 중 이중 보유를 막지 못한다 → [락 장애 유형 — 고가용성 구성](../../cs/0001-lock/05-failure-modes.md#8-락-서비스-중단과-고가용성-구성). 그래서 정확성은 DB가 맡는다. 원리는 **"확인하고 쓰기"를 DB의 쓰기 한 번 안에 넣는 것**이다. 판단의 근거가 제약으로 쓰기 안에 들어 있으면, 두 보유자가 같은 판단을 해도 DB가 하나만 통과시킨다.

#### 3.1 예: 복식부기 원장에 순번 (MySQL)

```sql
CREATE TABLE account (
  id             BIGINT       PRIMARY KEY,
  kind           VARCHAR(16)  NOT NULL,       -- customer | funds | clearing | revenue | expense | suspense
  normal_balance ENUM('debit','credit') NOT NULL,
  serialized     BOOLEAN      NOT NULL        -- true: 음수 금지. 순번·거래 후 잔액으로 직렬화
);

CREATE TABLE tx (
  id              BIGINT      PRIMARY KEY,
  type            VARCHAR(16) NOT NULL,       -- deposit | withdraw | transfer | reversal ...
  idempotency_key VARCHAR(64) NOT NULL UNIQUE,
  reverses_tx_id  BIGINT      NULL,
  created_at      DATETIME(6) NOT NULL,       -- 조회·감사용. 순서·충돌 판정에 쓰지 않는다
  FOREIGN KEY (reverses_tx_id) REFERENCES tx(id)
);

CREATE TABLE entry (
  id            BIGINT        AUTO_INCREMENT PRIMARY KEY,
  tx_id         BIGINT        NOT NULL,
  account_id    BIGINT        NOT NULL,
  direction     ENUM('debit','credit') NOT NULL,
  amount        DECIMAL(19,2) NOT NULL CHECK (amount > 0),
  seq           BIGINT        NULL,           -- 직렬화 계정만: 그 계정의 마지막 기입 + 1
  balance_after DECIMAL(19,2) NULL CHECK (balance_after >= 0),   -- 정상잔액 방향. CHECK는 8.0.16+
  UNIQUE (account_id, seq),                   -- 같은 상태를 보고 쓴 요청은 같은 순번으로 부딪힌다
  CHECK ((seq IS NULL) = (balance_after IS NULL)),
  FOREIGN KEY (tx_id) REFERENCES tx(id),
  FOREIGN KEY (account_id) REFERENCES account(id)
);
```

- **고객 계정**(`serialized = true`)의 기입만 순번과 거래 후 잔액을 가진다. `UNIQUE`는 NULL끼리 충돌하지 않으므로, 시스템 계정 기입(`seq = NULL`)은 직렬화되지 않는다. 핫 계정 문제가 스키마에서 풀린다(3.3).
- 순번은 발급받지 않고 마지막 기입에서 **계산한다**.

```sql
-- 지갑 A에서 100 출금
-- ① 마지막 기입 읽기 (락 없는 일반 SELECT로 충분)
SELECT seq, balance_after FROM entry
 WHERE account_id = :wallet_a AND seq IS NOT NULL
 ORDER BY seq DESC LIMIT 1;                                   -- 예: 순번 7, 잔액 500

-- ② 한 트랜잭션에 거래 1행 + 기입 2행
INSERT INTO tx (id, type, idempotency_key, created_at) VALUES (:tx, 'withdraw', :key, NOW(6));
INSERT INTO entry (tx_id, account_id, direction, amount, seq, balance_after) VALUES
  (:tx, :wallet_a, 'debit',  100, 7 + 1, 500 - 100),           -- 누가 먼저 8을 썼으면 중복 키 → ①부터
  (:tx, :bank,     'credit', 100, NULL,  NULL);                -- 시스템 계정: 직렬화하지 않음
```

```
A: Redis 락 획득 → 마지막 기입 읽음 (순번 7) ─ GC 정지 ────────── 깨어남 → 순번 8 INSERT → 중복 키 → 다시 읽고 재시도
                                                (TTL 만료)
B:                                              Redis 락 획득 → 마지막 기입 읽음 (순번 7) → 순번 8 INSERT → 성공
```

채번 테이블·`AUTO_INCREMENT`·시각으로 번호를 받으면 동시 요청이 서로 다른 번호를 받아 둘 다 성공한다. 같은 상태를 본 요청이 **같은 번호**를 만들어야 충돌이 잡힌다 → [흔한 함정 2절](./04-pitfalls.md#2-충돌을-못-잡는-번호). `entry.id`의 `AUTO_INCREMENT`는 행 식별용일 뿐 충돌 판정에 쓰지 않는다.

| 막을 것 | 장치 |
|---|---|
| 같은 잔액을 보고 둘 다 출금 | `UNIQUE (account_id, seq)` |
| 같은 거래 두 번 | `tx.idempotency_key` 유니크 |
| 잔액 음수 | `CHECK (balance_after >= 0)` |
| 한쪽 기입만 반영 | 거래와 기입을 한 트랜잭션에 |
| 기입 수정·삭제 | 권한 + 트리거 → [개요 3.1](./02-overview.md#31-확정된-기입은-고치지-않는다) |

- **오류별 처리**: 중복 키(1062)와 데드락(1213)은 마지막 기입을 다시 읽고 재시도한다. 멱등키 충돌은 "이미 처리됨"이다.
- **고객 계정 두 개가 끼는 이체는 `account_id` 오름차순으로 기입을 넣는다** (보내는 쪽·받는 쪽과 무관). 반대 방향 이체끼리의 데드락을 줄이는 효율 규칙이고, 어겨도 InnoDB가 한쪽을 롤백하므로 잔액은 틀어지지 않는다.
- **①과 ②를 `INSERT ... SELECT` 한 문장으로 합치지 않는다.** RR에서 원본 행에 공유 락이 걸려 동시 요청끼리 대기·데드락이 생긴다.
- **외부 호출이 끼면** 대기로 먼저 커밋하고, 외부 결과(거래 ID를 외부 멱등키로)에 따라 확정·취소한다 → [개요 7절](./02-overview.md#7-상태-대기--확정).

분산 락이 뚫리면 결과는 데이터 손상이 아니라 **한쪽의 재시도**다. 남는 것은 정확성 밖의 문제다.

| 남는 것 | 대응 |
|---|---|
| 한 고객 계정에 거래가 몰려 재시도 폭증 | 분산 락·계정별 큐로 줄을 세운다 (효율용) → 3.3 |
| 옛 값으로 재시도해 같은 충돌 반복 | 재시도마다 다시 읽기, 상한·백오프·지터 |
| 장애 조치 때 커밋된 기입 유실 | 반동기·Group Replication → [MySQL 락 — 장애 조치](../0004-mysql-lock/05-failure-modes.md#3-장애-조치failover). 정확성은 **DB의 제약과 내구성** 둘 다에 기댄다 |

#### 3.2 방식 비교

| 방식 | 충돌을 잡는 곳 | 얻은 것 | 포기한 것 |
|---|---|---|---|
| **순번 유니크** (3.1) | 기입 `(account, seq)` 중복 키 | 락 대기 없음. 이력과 충돌 검사가 한 테이블 | 계정별 직렬화, 재시도 코드 |
| **계정 행 버전** (낙관적 락) | `UPDATE account ... WHERE version = :v` 영향 행 0 | 잔액 캐시를 계정 행에 함께 둔다. Modern Treasury가 `lock_version`으로 제공 | 기입과 계정 행 두 곳을 함께 갱신 |
| **계정 행 잠금** (`SELECT ... FOR UPDATE`) | DB 행 락 대기 | 재시도가 거의 없다 | 대기·데드락. 락은 효율, 정확성은 제약이 맡는다 |
| **단일 작성자** (계정별 큐, 전용 DB) | 쓰기 순서 자체 | 충돌이 없다. TigerBeetle은 한 코어가 묶음으로 처리 | 큐·전용 시스템 운영 |

#### 3.3 핫 계정

복식부기에서 시스템 계정(회사 은행 계좌, PG 미수금, 수수료)은 **거의 모든 거래**에 들어간다. 이 계정까지 직렬화하면 전체 처리량이 그 계정 하나에 묶인다. 한 계정의 쓰기가 직렬화되면 커밋당 1ms(fsync 포함)일 때 그 계정은 **초당 약 1,000건**이 상한이다 → [MySQL 락 — 수치로 보는 비용](../0004-mysql-lock/03-tradeoffs.md#2-수치로-보는-비용).

| 대응 | 내용 | 대가 |
|---|---|---|
| **잔액 규칙 없는 계정은 직렬화하지 않는다** | 시스템 계정은 `seq = NULL`로 덧붙이고 잔액은 합계로 구한다 (3.1 스키마) | 그 계정의 현재 잔액 읽기가 느리다 |
| 계정 나누기 | 한 논리 계정을 하위 계정 N개로 나눠 쓰고 잔액은 합으로 읽는다 | 음수 금지를 하위 계정마다 걸어야 해서, 합은 충분한데 한 하위 계정이 모자라 거절되는 일이 생긴다 |
| 묶어서 쓰기 | 짧은 시간 동안 모은 거래를 계정당 한 번 읽고 한 번 쓴다 | 묶는 시간만큼 지연이 는다 |

사례:
- **Uber (2026)**: 결제 계정 하나의 처리량이 초당 3~4건에 묶여 있었다. 계정 나누기는 버리고, 계정별로 **250ms 동안 모은 작업**을 한 번 읽고 낙관적 락으로 한 번에 쓰게 바꿔 초당 30건 이상(작업당 8~20ms)으로 올렸다.
- **TigerBeetle**: 한 코어·한 리더가 요청을 묶음(요청당 최대 8,190건)으로 처리한다. 충돌이 없어 핫 계정이 처리량을 떨어뜨리지 않지만, 노드를 늘려도 처리량은 늘지 않는다(가용성만 는다).

### 4. 잔액을 어떻게 얻나

| 방식 | 읽기 | 쓰기·동시성 | 맞는 계정 |
|---|---|---|---|
| **매번 합계** | 기입 수에 비례해 느려진다 | 덧붙이기뿐. 음수 금지 검사를 하면 합계와 쓰기 사이에 다른 쓰기가 끼어든다 | 잔액 규칙 없는 시스템 계정, 보고용 |
| **거래 후 잔액** | 마지막 기입 1행 | 계정별 직렬화. 충돌은 순번으로 잡는다(3.1) | 음수 금지 고객 계정 |
| **계정 행 잔액 캐시** | 계정 행 1개 | 조건부 `UPDATE`나 버전 비교. 캐시와 기입 합이 어긋날 수 있다 → [장애 유형 5절](./05-failure-modes.md#5-잔액-캐시-어긋남) | 계정 행을 이미 갱신하는 시스템 |
| **스냅샷 + 이후 합계** | 스냅샷 + 그 뒤 기입 합 | 덧붙이기뿐. 스냅샷 작업이 따로 돈다 | 기입이 아주 많고 읽기가 드문 계정 |

한 원장 안에서 계정 종류마다 다른 방식을 쓰는 것이 보통이다. 고객 계정은 거래 후 잔액, 시스템 계정은 합계나 스냅샷.

### 5. 구성 선택

| 선택지 | 맞는 곳 | 대가 |
|---|---|---|
| **범용 RDBMS** (MySQL, PostgreSQL) | 대부분. 제약·트랜잭션·운영 경험이 이미 있다 | 핫 계정 상한, 스키마·재시도를 직접 설계 |
| **전용 원장 DB** (TigerBeetle) | 초당 수만 건 이상. 복식부기·2단계 전송이 기본 기능 | 범용 질의 불가, 계정·전송 모델에 맞춰야 한다 |
| **원장 API 서비스** (Modern Treasury Ledgers 등) | 직접 운영하기 부담스러운 결제·핀테크 | 외부 의존, 비용, 지연 |

고가용성은 원장을 담은 DB 층의 구성(반동기, Group Replication, Raft 기반 저장소)을 따른다. 분산 락 층과 DB 층의 역할 분리는 [락 장애 유형 — 권장 구조](../../cs/0001-lock/05-failure-modes.md#8-락-서비스-중단과-고가용성-구성).

### 6. 쓰지 말아야 할 때

| 상황 | 이유 | 대안 |
|---|---|---|
| 옮겨지는 양이 없는 규칙 (이메일 중복, 좌석 1인) | 보존할 양도, 쌓을 이력도 없다 | 유니크 제약 |
| 근사치로 충분한 카운터 (조회수, 좋아요) | 이력이 근거가 될 일이 없다. 핫 계정만 생긴다 | 캐시 카운터, 주기적 합산 |
| 최신 값만 의미 있는 데이터 (설정, 프로필) | 이력이 필요 없다 | 일반 `UPDATE`, 필요하면 변경 로그 |

### 7. 선택 체크리스트

- 옮겨지는 양이 있고, 잔액 규칙이 있고, 이력이 근거가 되는가? 셋 다 아니면 원장은 과하다.
- 외부 기관과 주고받는가? 그렇다면 **복식부기**, 그리고 시스템 계정 목록부터 설계한다.
- 기입 테이블에 `UPDATE`·`DELETE`를 **DB 차원에서** 막았는가?
- 어느 계정이 음수 금지인가? 그 계정만 직렬화하고 나머지는 덧붙이기만.
- 외부 호출이 끼는가? 그렇다면 **대기 → 확정** 상태.
- 고객 계정 하나에 몰리는 쓰기가 초당 수백 건을 넘는가? 넘으면 묶음 처리·전용 DB.

## 관련 문서

- [원장의 역사](./01-history.md)
- [원장 개요와 동작 방식](./02-overview.md)
- [원장 사용 시 흔한 함정과 해결 방법](./04-pitfalls.md)
- [원장 장애 유형과 대응](./05-failure-modes.md)
- [락 트레이드오프 — 효율용인가 정확성용인가](../../cs/0001-lock/03-tradeoffs.md#3-효율용인가-정확성용인가)
- [MySQL 락 트레이드오프와 선택 기준](../0004-mysql-lock/03-tradeoffs.md)

## 참고 자료

- [Modern Treasury — Handle Concurrency](https://docs.moderntreasury.com/ledgers/docs/handle-concurrency)
- [Modern Treasury — How to Scale a Ledger, Part VI](https://www.moderntreasury.com/journal/how-to-scale-a-ledger-part-vi)
- [TigerBeetle — Performance](https://docs.tigerbeetle.com/concepts/performance/)
- [Uber — Building High Throughput Payment Account Processing (2026)](https://www.uber.com/us/en/blog/high-throughput-processing/)
- [MySQL 8.0 Reference Manual — CHECK Constraints](https://dev.mysql.com/doc/refman/8.0/en/create-table-check-constraints.html)
- [Martin Kleppmann — How to do distributed locking (2016)](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)
