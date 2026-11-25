---
type: knowledge
created: 2026-11-19
author: agent
model: claude-opus-5-5
---

# MySQL 락 개요와 동작 방식

## 쉬운 설명

MySQL(InnoDB)은 **책장의 책(행)이 아니라 색인 카드(인덱스)에 팻말을 붙여서** 잠근다. 카드 한 장에 붙이면 그 행만 잠기고(record lock), 카드 사이 빈칸에 붙이면 **그 자리에 새 카드가 못 끼어든다**(gap lock). 그냥 읽는 사람은 팻말을 보지 않고 **사진(스냅샷)** 을 본다. 그래서 읽기는 막히지 않고, 고치려는 사람끼리만 줄을 선다.

## 일반 설명

> 이 문서는 **어떻게 동작하나**만 다룬다. 왜 이렇게 설계했고 언제 무엇을 고르는지는 [트레이드오프](./03-tradeoffs.md), 잘못 쓰는 패턴은 [흔한 함정](./04-pitfalls.md), 장애는 [장애 유형](./05-failure-modes.md)에 있다. 솔루션을 가리지 않는 공통 개념(모드·단위·해제 보장·fencing·2단계 락)은 [락 개론](../../cs/0001-lock/02-overview.md)에 있다.

### 1. 락이 해결하는 문제

| 문제 | 예 | 쓰는 락 |
|---|---|---|
| **갱신 분실**(lost update) | 두 요청이 재고 10을 읽고 각각 9로 씀 | 행 배타 락(X) |
| **팬텀** | 범위를 읽고 판단하는 동안 그 범위에 새 행이 들어옴 | gap / next-key lock |
| **스키마 변경 중 접근** | 트랜잭션이 쓰는 테이블을 `ALTER`가 바꿈 | 메타데이터 락(MDL) |
| **백업 일관성** | 백업 중 DDL로 파일 구조가 바뀜 | 백업 락 |

### 2. 락의 층

```
서버 계층 ─┬─ 전역/백업 락      FLUSH TABLES WITH READ LOCK, LOCK INSTANCE FOR BACKUP
          ├─ 메타데이터 락(MDL)  모든 테이블 접근(DML=공유, DDL=배타)
          └─ 사용자 락           GET_LOCK('name', timeout)
InnoDB ───┬─ 테이블 락          IS / IX(의도 락), AUTO-INC, LOCK TABLES
          └─ 행 락(인덱스 레코드) record / gap / next-key / insert intention
```

- **MDL은 서버 계층**, 행 락은 **InnoDB 계층**이다. 둘은 서로 모르고 따로 대기한다. 그래서 `ALTER TABLE`이 MDL을 기다리며 뒤 쿼리를 막는 문제는 InnoDB 락 뷰(`data_locks`)에 보이지 않는다.
- **의도 락(IS/IX)**: 행에 S/X 락을 걸기 전에 테이블에 "나 이 안의 행을 잠글 거야"라고 표시한다. 행 락끼리는 막지 않고, `LOCK TABLES ... WRITE` 같은 테이블 전체 락과의 충돌만 빠르게 판정하는 용도다.

### 3. 행 락: 존재하는 인덱스 레코드에만 붙는다

InnoDB 행 락의 출발점은 하나다. **락은 실제로 존재하는 인덱스 레코드에만 붙일 수 있다.**

- 락은 "어느 페이지의 몇 번째 레코드"로 기록된다(페이지마다 레코드 번호 비트맵). `13 ≤ x ≤ 17` 같은 **조건 자체는 기록할 방법이 없다.**
- 빈칸(gap)도 독립된 대상이 아니다. "레코드 R **앞의** 빈칸"으로 표현해 **R에 붙인다.** 마지막 레코드 뒤 빈칸은 가상 레코드 **supremum**(+∞)에 붙인다.
- 그래서 어떤 락이든 경계는 **이웃한 실제 레코드**로 정해진다. 조건(술어) 자체를 잠그는 방식(predicate lock)과 다른 점이고, InnoDB는 SPATIAL 인덱스에서만 predicate lock을 쓴다. 이렇게 설계한 이유와 대가는 [트레이드오프](./03-tradeoffs.md).

인덱스 값이 `10, 15, 20`일 때 붙일 수 있는 락은 다음 넷뿐이다.

```
 -∞ ──( gap )── 10 ──( gap )── 15 ──( gap )── 20 ──( gap )── +∞(supremum)
                    └── 15에 붙는 next-key (10,15] ──┘
```

| 락 | 잠그는 대상 | 예 (15에 붙임) | 막는 것 | `data_locks.lock_mode` |
|---|---|---|---|---|
| **record lock** | 레코드 하나 | `[15]` | 15의 수정·삭제·잠금 읽기 | `X,REC_NOT_GAP` |
| **gap lock** | 레코드 **앞** 빈칸 (레코드 자체는 아님) | `(10,15)` | 11~14 INSERT만 | `X,GAP` |
| **next-key lock** | record + 그 앞 gap | `(10,15]` | 위 둘 다 | `X` |
| **insert intention** | INSERT 직전 "이 gap에 넣겠다"는 표시 | — | (gap lock과 충돌해 대기) | `X,INSERT_INTENTION` |

- RR 잠금 읽기의 기본 단위는 **next-key lock**이고, 안전할 때만 record lock이나 gap lock으로 줄어든다(4절).
- **gap lock끼리는 충돌하지 않는다.** 두 트랜잭션이 같은 gap에 gap lock을 동시에 가질 수 있다. gap lock이 막는 것은 **insert intention lock**(INSERT)뿐이다. 존재하지 않는 행에 `FOR UPDATE`를 건 두 트랜잭션이 둘 다 성공한 뒤 서로의 INSERT를 막아 데드락이 나는 이유다.
- 공유(S)와 배타(X): S끼리는 함께 가질 수 있고, X는 S·X 모두와 충돌한다.

호환성 (기존 락 → 요청 락):

| 요청 \ 보유 | record S | record X | gap(S/X) | next-key X |
|---|---|---|---|---|
| record S | ✓ | ✗ | ✓ | ✗ |
| record X | ✗ | ✗ | ✓ | ✗ |
| gap | ✓ | ✓ | ✓ | ✓ |
| insert intention | ✓ | ✓ | **✗** | **✗** |

### 4. 락 범위는 어떻게 정해지나 (REPEATABLE READ)

목표는 팬텀 방지다. 같은 잠금 읽기를 다시 했을 때 결과가 같으려면 **조건을 만족할 수 있는 빈자리 전체**에 INSERT를 막아야 한다. 하지만 3절처럼 락은 실제 레코드에만 붙으므로, InnoDB는 조건을 잠그는 대신 **인덱스를 스캔하며 만난 레코드에 락을 붙인다.** 범위는 아래 세 규칙으로 정해진다.

1. **스캔한 레코드마다 next-key lock.** 조건이 아니라 스캔 기준이라, 조건에서 탈락한 행도 스캔했으면 잠긴다. 쓸 인덱스가 없으면 PK 전체를 스캔하므로 모든 레코드가 잠긴다.
2. **조건을 벗어난 첫 레코드에서 멈추되, 그 앞 빈칸까지 잠근다.** 조건을 만족하는 값이 그 빈칸에 새로 들어올 수 있기 때문이다. 멈출 레코드가 없으면 supremum까지 간다.
3. **안전할 때만 줄인다.**
   - 유니크 인덱스 + 모든 키 컬럼 등치 + 값 있음 → **record lock만.** 같은 값이 새로 들어올 수 없으니 빈칸을 막을 필요가 없다. 복합 유니크의 일부 컬럼만 쓰면 범위로 취급한다.
   - 등치인데 값 없음 → **그 자리 gap lock만.** 잠글 레코드가 없다.
   - 멈춘 레코드 → PK는 8.0.18부터 **gap lock만**(레코드 자체는 안 잠금). 그 이전 버전과 보조 인덱스는 멈춘 레코드까지 next-key로 잠그기도 한다.

결과적으로 락 범위는 **조건이 아니라 조건 양옆의 실제 레코드**까지 넓어진다.

```
id > 12 AND id < 18 FOR UPDATE   (PK, 인덱스 값 10, 15, 20)

조건:          12 <─────────── x ───────────> 18
인덱스:   10 ──( gap )── 15 ──( gap )── 20
락:          └─ (10,15] ─┘└─ (15,20) ─┘
             15에 next-key  20에 gap (규칙 2·3)

→ 조건은 13~17인데 INSERT는 11~19가 막힌다.
  (10,15) 빈칸은 15에 붙은 하나의 단위라 "13부터"로 자를 수 없다.
```

같은 데이터에서 조건별로 정리하면 (PK, 8.0.18+):

| 조건 | 걸리는 락 | INSERT가 막히는 값 | 왜 |
|---|---|---|---|
| `id = 15` | `[15]` record | 없음 | 규칙 3: 유니크 등치 + 값 있음 |
| `id = 17` | `(15,20)` gap | 16~19 | 규칙 3: 값 없음 → 17이 들어갈 빈칸(20 앞)만 |
| `id > 12 AND id < 18` | `(10,15]` + `(15,20)` gap | 11~19 | 위 그림 |
| `id > 15` | `(15,20]`, `(20,+∞]` | 16 이상 **전부** | 규칙 2: 멈출 레코드가 없어 supremum까지 |
| `id < 15` | `(-∞,10]` + `(10,15)` gap | 14 이하 **전부** | 규칙 2: 맨 앞부터 15 앞까지 |
| 인덱스 없는 컬럼 조건 | 모든 레코드 next-key + supremum | **전부** | 규칙 1: PK 전체 스캔 |

인덱스 종류별 요약 (`= 15`, `> 15`·`< 15` 기준):

| 인덱스 | 등치 조건 | 범위 조건 |
|---|---|---|
| **클러스터드 (PK)** | • 값 있음: **record lock**만<br>• 값 없음: 그 자리 **gap lock** | 스캔한 레코드 **next-key** + 멈춘 레코드 앞 **gap** |
| **유니크 보조** | • 값 있음: 보조키 **record lock** + 매칭 PK **record lock**<br>• 값 없음: 그 자리 **gap lock** | 스캔한 보조키 **next-key**(멈춘 레코드 포함될 수 있음) + 매칭 PK **record lock** |
| **비유니크 보조** | • 값 있음: 모든 `15`에 **next-key** + 다음 레코드 앞 **gap** + 매칭 PK **record lock** (같은 값이 앞뒤로 또 들어올 수 있어 양쪽 빈칸을 막는다)<br>• 값 없음: 그 자리 **gap lock** | 스캔한 보조키 **next-key**(멈춘 레코드 포함) + 매칭 PK **record lock** |
| **인덱스 없음** | PK 전체 스캔 → 모든 레코드 **next-key** + supremum = 사실상 테이블 전체 | 등치와 같다 |

- 데이터가 듬성듬성하면 넓게, 촘촘하면 좁게 잠긴다. 같은 쿼리라도 운영 DB와 개발 DB에서 잠기는 범위가 다르다.
- 이 범위 때문에 생기는 대기·데드락은 [흔한 함정](./04-pitfalls.md), 쓰기 문장(INSERT 중복 키, `REPLACE` 등)의 락은 [RR에서 next-key lock이 걸리는 경우](../../../inquiry/database/0016-mysql-rr-next-key-lock-cases.md), 격리 수준별 차이는 6절.

### 5. 읽기의 두 종류

| | 일관 읽기 (consistent read) | 잠금 읽기 (locking read) |
|---|---|---|
| 문장 | 일반 `SELECT` | `SELECT ... FOR SHARE` / `FOR UPDATE`, `UPDATE`, `DELETE`, `INSERT` |
| 보는 데이터 | MVCC **스냅샷** (undo log로 과거 버전 재구성) | **최신 커밋 버전** |
| 락 | 없음 | S 또는 X + (격리 수준에 따라) gap |
| 막히나 | 안 막힘 | 다른 쓰기와 충돌하면 대기 |

같은 트랜잭션 안에서 일반 `SELECT`와 `SELECT ... FOR UPDATE`가 **서로 다른 값을 볼 수 있다.** 앞은 스냅샷, 뒤는 최신값이다. 스냅샷과 gap lock은 별개의 장치다 → [gap lock은 스냅샷 시점에 걸리지 않는다](../../../inquiry/database/0003-mysql-gap-lock-next-key-lock-mechanism-and-terms.md). 스냅샷이 undo로 옛 버전을 재구성하는 과정은 [InnoDB MVCC는 어떻게 동작하나](../../../inquiry/database/0017-mysql-innodb-mvcc-mechanism.md).

### 6. 격리 수준에 따른 차이

| 격리 수준 | 일반 SELECT | 잠금 문장의 행 락 |
|---|---|---|
| READ COMMITTED | 문장마다 새 스냅샷 | **record lock만**. gap lock은 외래 키·중복 키 검사에서만. 조건에 안 맞는 행의 락은 평가 직후 해제 |
| **REPEATABLE READ (기본)** | 트랜잭션 첫 읽기 스냅샷 | **next-key lock** |
| SERIALIZABLE | (autocommit 꺼짐 시) `FOR SHARE`로 승격 | next-key lock |

자세한 내용은 [next-key lock은 RR에서만 걸리나](../../../inquiry/database/0004-mysql-next-key-lock-scope-and-postgres-visibility-check.md).

### 7. 기다림과 데드락 처리

```
락 요청 ─▶ 충돌 없음 ─▶ 즉시 획득
       └▶ 충돌 ─▶ 대기열에 들어감 (CATS: 많은 트랜잭션을 막고 있는 쪽에 먼저 부여)
                 ├─ 대기 그래프에 사이클 → 데드락 감지 → 작은 트랜잭션을 롤백 (ERROR 1213)
                 └─ innodb_lock_wait_timeout(기본 50초) 초과 → 해당 문장만 롤백 (ERROR 1205)
```

- **데드락**: 감지 즉시 희생자를 고른다. 삽입·수정·삭제한 행 수가 적은 트랜잭션이 희생된다. 희생된 트랜잭션은 **전체 롤백**된다.
- **락 대기 타임아웃**: 기본으로 **문장만** 롤백되고 트랜잭션은 열려 있다. `innodb_rollback_on_timeout=ON`이면 트랜잭션 전체가 롤백된다.
- 락은 **커밋·롤백 때 한꺼번에 풀린다**(2단계 락). 문장이 끝나도 풀리지 않는다.

### 8. 기본 사용법

```sql
-- 비관적 락: 읽고 바로 고칠 행
START TRANSACTION;
SELECT stock FROM product WHERE id = 42 FOR UPDATE;
UPDATE product SET stock = stock - 1 WHERE id = 42;
COMMIT;

-- 기다리지 않기
SELECT * FROM seat WHERE id = 7 FOR UPDATE NOWAIT;        -- 잠겨 있으면 즉시 ERROR 3572

-- 잠긴 행은 건너뛰기 (작업 큐)
SELECT id FROM job WHERE status = 'READY'
ORDER BY id LIMIT 10 FOR UPDATE SKIP LOCKED;

-- 조인 중 특정 테이블만 잠그기
SELECT * FROM orders o JOIN customer c ON ...
WHERE o.id = 1 FOR UPDATE OF o;

-- 이름 붙은 사용자 락 (세션 단위)
SELECT GET_LOCK('daily-batch', 0);   -- 1=획득, 0=실패
SELECT RELEASE_LOCK('daily-batch');
```

### 9. 관측

```sql
-- 현재 보유·대기 중인 행 락
SELECT engine_transaction_id, object_name, index_name, lock_type, lock_mode, lock_status, lock_data
FROM performance_schema.data_locks;

-- 누가 누구를 막고 있나 (blocking_pid를 KILL 대상으로)
SELECT * FROM sys.innodb_lock_waits\G

-- 메타데이터 락 대기
SELECT * FROM sys.schema_table_lock_waits\G

-- 최근 데드락 1건
SHOW ENGINE INNODB STATUS\G   -- LATEST DETECTED DEADLOCK
```

## 관련 문서

- [락 개론 (공통 개념)](../../cs/0001-lock/02-overview.md)
- [MySQL 락의 역사](./01-history.md)
- [MySQL 락 트레이드오프와 선택 기준](./03-tradeoffs.md)
- [MySQL 락: 흔한 함정과 해결 방법](./04-pitfalls.md)
- [MySQL 락 장애 유형과 대응](./05-failure-modes.md)
- [MySQL 개요 — 트랜잭션과 격리 수준](../0001-mysql/02-overview.md#4-트랜잭션과-격리-수준)
- [gap lock·next-key lock 용어 정리](../../../inquiry/database/0003-mysql-gap-lock-next-key-lock-mechanism-and-terms.md)
- [next-key lock은 RR에서만 걸리나](../../../inquiry/database/0004-mysql-next-key-lock-scope-and-postgres-visibility-check.md)
- [RR에서 next-key lock이 걸리는 경우](../../../inquiry/database/0016-mysql-rr-next-key-lock-cases.md)
- [Java 락 개요](../../java/0002-java-lock/02-overview.md)

## 참고 자료

- [MySQL 8.4 Reference Manual — InnoDB Locking](https://dev.mysql.com/doc/refman/8.4/en/innodb-locking.html)
- [MySQL 8.4 Reference Manual — Locks Set by Different SQL Statements in InnoDB](https://dev.mysql.com/doc/refman/8.4/en/innodb-locks-set.html)
- [MySQL 8.4 Reference Manual — Locking Reads](https://dev.mysql.com/doc/refman/8.4/en/innodb-locking-reads.html)
- [MySQL 8.4 Reference Manual — Deadlock Detection](https://dev.mysql.com/doc/refman/8.4/en/innodb-deadlock-detection.html)
- [MySQL Manual — The metadata_locks Table](https://dev.mysql.com/doc/mysql/en/performance-schema-metadata-locks-table.html)
- [MySQL 8.0 Reference Manual — Transaction Scheduling](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-scheduling.html)
- [A Comprehensive (and Animated) Guide to InnoDB Locking](https://jahfer.com/posts/innodb-locks/)
