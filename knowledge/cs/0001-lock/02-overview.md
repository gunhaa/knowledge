---
type: knowledge
created: 2026-11-23
author: agent
model: claude-opus-5-5
---

# 락 개요와 공통 구조

## 쉬운 설명

락은 **화장실 열쇠가 하나뿐인 가게**와 같다. 열쇠를 가진 사람만 들어가고, 나머지는 기다리거나 포기한다. 나올 때 열쇠를 돌려 놓아야 다음 사람이 쓴다. Java·MySQL·Redis의 락은 모두 이 구조이고, 다른 점은 **열쇠를 어디에 두나**(한 프로그램 안, DB 안, 여러 서버가 함께 보는 곳)와 **열쇠를 안 돌려주고 사라지면 어떻게 되돌려받나**뿐이다.

## 일반 설명

> 이 문서는 락이 **어떻게 동작하나**를 솔루션 공통으로 다룬다. 왜 이렇게 설계했고 언제 무엇을 고르는지는 [트레이드오프](./03-tradeoffs.md), 잘못 쓰는 패턴은 [흔한 함정](./04-pitfalls.md), 장애는 [장애 유형](./05-failure-modes.md)에 있다. 솔루션별 구현은 [Java 락](../../java/0002-java-lock/02-overview.md), [MySQL 락](../../database/0004-mysql-lock/02-overview.md), [Redis 락](../../database/0005-redis-lock/02-overview.md).

### 1. 락이 푸는 문제

여러 실행 흐름(스레드·트랜잭션·서버)이 같은 데이터를 **읽고 → 판단하고 → 쓰는** 사이에 끼어들면 결과가 틀어진다.

| 문제 | 예 | 왜 생기나 |
|---|---|---|
| **갱신 분실** (lost update) | 두 요청이 재고 10을 읽고 각각 9를 씀 → 9 (정답 8) | 읽기와 쓰기 사이에 다른 쓰기가 끼어듦 |
| **확인 후 실행** (check-then-act) | 쿠폰 잔여 1을 확인한 두 요청이 모두 발급 | 확인 결과가 실행 시점에 이미 틀림 |
| **불변식 위반** | 계좌 A 출금과 B 입금 사이를 다른 요청이 봄 → 총액이 잠시 틀림 | 여러 값을 함께 바꾸는 도중 상태가 보임 |
| **팬텀** | 범위를 확인한 뒤 그 범위에 새 행이 들어옴 | 없는 데이터는 잠글 대상이 없음 |
| **중복 실행** | 배치가 서버 3대에서 동시에 실행 | 누가 하는지 아무도 모름 |

락은 이 구간을 **임계 구역**(critical section)으로 묶어, 한 번에 하나만(또는 읽기끼리만) 들어가게 한다.

### 2. 기본 구조: 획득 → 임계 구역 → 해제

```
획득 시도 ─┬─ 비어 있음 → 획득 → [임계 구역] → 해제 → 대기자 하나를 깨움
          └─ 사용 중   → 대기(스핀·블록·큐) / 즉시 포기 / 타임아웃
```

좋은 락은 세 가지를 만족해야 한다 (Dijkstra의 조건).

| 성질 | 뜻 | 깨지면 |
|---|---|---|
| **상호 배제** | 임계 구역에 동시에 둘이 없다 | 경쟁 조건 |
| **진행** (progress) | 아무도 없으면 누군가는 들어간다 | 데드락·라이브락 |
| **한정 대기** (bounded waiting) | 기다리는 쪽이 영원히 밀리지 않는다 | 기아 |

### 3. 바닥에는 원자 연산이 있다

락 자체도 "비어 있나 확인 → 내 것으로 표시"라는 확인 후 실행이다. 이 두 단계를 **나눌 수 없게** 해 주는 무언가가 반드시 아래층에 있다.

| 층 | 락 | 그 아래의 원자 연산 |
|---|---|---|
| CPU·언어 | Java `synchronized`, `ReentrantLock` | CPU의 **CAS**(compare-and-swap) 명령 |
| DB | InnoDB 행 락 | 락 테이블을 보호하는 내부 **래치**(짧은 뮤텍스) |
| Redis | `SET key token NX PX` | Redis의 **단일 스레드 명령 실행** — 명령 하나는 끼어들 틈이 없다 |
| 합의 시스템 | etcd lease, ZooKeeper 순차 노드 | **합의 로그**(Raft·ZAB)의 순서 — 과반이 같은 순서에 동의 |

그래서 "락이 필요 없게 만드는" 방법도 같은 원자 연산을 직접 쓰는 것이다 — `AtomicLong.incrementAndGet()`, `UPDATE ... SET stock = stock - 1 WHERE stock > 0`, Redis `INCRBY`.

### 4. 모든 락이 공유하는 축

솔루션은 달라도 락은 아래 축의 조합이다. 이 표가 세 솔루션 세트를 읽는 지도다.

| 축 | 질문 | Java | MySQL (InnoDB) | Redis |
|---|---|---|---|---|
| **범위 (층)** | 누구 사이의 상호 배제인가 | 한 JVM의 스레드 | 한 DB의 트랜잭션 | 여러 서버·프로세스 |
| **모드** | 함께 가질 수 있나 | `ReadWriteLock`(읽기 공유/쓰기 배타) | S(공유)·X(배타) + 의도 락 IS/IX | 배타만 (`RReadWriteLock`은 Redisson 구현) |
| **단위** (granularity) | 무엇을 잠그나 | 객체 모니터, 락 객체 단위 | 테이블 / 인덱스 레코드 / gap(범위) | 키 이름 하나 (이름은 사용자가 정함) |
| **대기 방식** | 못 얻으면 | 스핀 후 park, `tryLock(timeout)` | 대기 큐, `NOWAIT`, `SKIP LOCKED`, 타임아웃 50초 | 즉시 실패, 재시도, Pub/Sub 알림 |
| **소유와 재진입** | 누가 쥐었나 | 스레드 소유, 재진입 가능 | 트랜잭션 소유 | 값에 넣은 **토큰**으로 소유 표시 |
| **해제 보장** | 보유자가 사라지면 | 블록 종료·`finally`. 프로세스가 죽으면 함께 사라짐 | 커밋·롤백, 세션 종료 시 롤백 | **TTL(lease) 만료** |
| **보유 기간** | 언제 푸나 | 임계 구역 끝 | **커밋까지**(2단계 락) | 명시적 해제 또는 TTL |
| **데드락 처리** | 서로 기다리면 | 감지만(스레드 덤프), 자동 해소 없음 | 대기 그래프로 감지 → 한쪽 롤백 | 없음 — TTL이 결국 풂 |

축마다 자세한 동작은 아래 절과 각 세트에 있다.

### 5. 모드: 공유(S)와 배타(X)

| 보유 \ 요청 | S (읽기) | X (쓰기) |
|---|---|---|
| S | ✓ 함께 가능 | ✗ |
| X | ✗ | ✗ |

- 읽기가 많고 쓰기가 드물면 S/X 구분으로 읽기를 병렬화한다.
- **의도 락**(intention lock): 큰 단위(테이블)에 "이 안의 작은 단위(행)를 잠글 것"이라고 표시해, 테이블 전체 락과의 충돌을 행을 다 뒤지지 않고 판정한다(Gray 1975). InnoDB의 IS/IX가 이것이다.

### 6. 단위: 무엇을 잠그나

| 단위 | 예 | 성질 |
|---|---|---|
| 전체 | 전역 뮤텍스, 테이블 락 | 단순, 데드락 없음, 병렬성 없음 |
| 객체·행 | `synchronized(account)`, InnoDB record lock | 병렬성 높음, 여러 개를 잡으면 순서 관리 필요 |
| 범위 | InnoDB gap·next-key lock | 아직 없는 데이터(팬텀)까지 막음 |
| 이름 | Redis 키, MySQL `GET_LOCK('name')` | 보호 대상과 락이 **약속으로만** 연결됨 |

**이름 락의 성질**: Redis 락이나 `GET_LOCK`은 데이터 자체가 아니라 **이름**을 잠근다. 같은 이름을 쓰기로 약속한 코드끼리만 서로를 막고, 약속을 모르는 코드는 그대로 데이터를 고칠 수 있다. 반대로 DB 행 락은 그 행을 건드리는 모든 잠금 문장을 막는다.

### 7. 해제 보장과 lease

보유자가 해제하지 못하고 사라지는 경우를 층마다 다르게 처리한다.

| 층 | 보유자가 죽으면 | 보유자가 **멈추면** (GC 정지, 네트워크 지연) |
|---|---|---|
| JVM | 프로세스와 함께 락도 사라짐 | 다른 스레드는 계속 기다림 — 상호 배제는 유지 |
| DB | 커넥션 종료 → 롤백 → 락 해제 | 트랜잭션이 열린 채 대기 — 상호 배제는 유지 |
| Redis·etcd | **TTL·lease 만료**로 해제 | **lease가 먼저 만료될 수 있음** → 다른 쪽이 획득 → **두 보유자** |

분산 락은 보유자가 "죽었는지 느린지"를 구분할 수 없다. 그래서 시간 제한(**lease**)으로 해제를 보장하고, 그 대가로 멈춘 보유자와 새 보유자가 겹칠 수 있다. 이것을 막는 장치가 fencing token이다.

### 8. Fencing token

락을 줄 때마다 **단조 증가 번호**를 함께 주고, **보호 대상(저장소)** 이 "이미 본 번호보다 작은 번호의 쓰기"를 거부한다.

```
A: 락 획득 → token 33 ─── (GC 정지 … lease 만료) ──────── write(33) → 거부 (이미 34를 봄)
B:                          락 획득 → token 34 ─ write(34) → 성공
```

- 번호의 출처: ZooKeeper `zxid`·순차 노드 번호, etcd `revision`, Redis `INCR`, DB의 `version` 컬럼.
- **보호 대상이 번호를 검사해야** 효과가 있다. 외부 API처럼 검사할 수 없는 대상은 멱등키로 대신한다.
- 낙관적 락의 `version` 컬럼은 사실상 fencing token이다 — 쓰기 시점에 "내가 본 버전이 아직 최신인가"를 저장소가 검사한다.

솔루션별 구현은 [Redis 락 — Fencing token](../../database/0005-redis-lock/02-overview.md#5-fencing-token).

### 9. 비관적 vs 낙관적 vs 락 없음

| 방식 | 동작 | 예 |
|---|---|---|
| **비관적** | 충돌할 것으로 보고 **먼저 잠근 뒤** 작업 | `synchronized`, `SELECT ... FOR UPDATE`, Redis 락 |
| **낙관적** | 잠그지 않고 작업한 뒤, **쓰는 순간 바뀌지 않았는지 검사**. 바뀌었으면 실패·재시도 | CAS, `@Version`, `UPDATE ... WHERE version = ?` |
| **MVCC** | 읽기는 과거 버전(스냅샷)을 봐서 쓰기와 안 막힘. 쓰기끼리는 여전히 락 | InnoDB·PostgreSQL 일반 `SELECT` |
| **락 없음** | 공유 자체를 없앰 | 불변 객체, 키 파티셔닝(같은 키는 한 작업자), 단일 작성자·액터 |

언제 무엇을 고르나는 [트레이드오프 — 대안 비교](./03-tradeoffs.md#4-대안-비교).

### 10. 2단계 락 (2PL)

DB 트랜잭션이 직렬화 가능한 결과를 내는 기본 규칙이다.

```
락 개수
  ▲        ┌───────┐
  │      ┌─┘       │
  │    ┌─┘         │ ← strict 2PL: 커밋·롤백 때 한꺼번에 해제
  │  ┌─┘           │
  └──┴─────────────┴──▶ 시간
     확장 단계      커밋
```

- **확장 단계**에서는 락을 얻기만 하고, **축소 단계**에 들어가면 새 락을 얻지 않는다.
- 실무 DB는 **strict 2PL**: 쓰기 락을 커밋까지 쥔다. 그래야 커밋 안 된 값을 남이 보고 쓰는 일이 없다.
- 결과: DB 락은 문장이 끝나도 풀리지 않고 **트랜잭션 길이만큼** 유지된다. 반면 Java 락은 2PL이 아니라 임계 구역이 끝나면 바로 풀린다 — 두 락을 섞을 때 경계가 어긋나는 원인이다 → [흔한 함정](./04-pitfalls.md#6-락과-트랜잭션-경계가-어긋남-실수).

## 관련 문서

- [락의 역사](./01-history.md)
- [락 트레이드오프와 선택 기준](./03-tradeoffs.md)
- [락 사용 시 흔한 함정과 해결 방법](./04-pitfalls.md)
- [락 장애 유형과 대응](./05-failure-modes.md)
- [Java 락 개요](../../java/0002-java-lock/02-overview.md)
- [MySQL 락 개요](../../database/0004-mysql-lock/02-overview.md)
- [Redis 락 개요](../../database/0005-redis-lock/02-overview.md)

## 참고 자료

- [Lamport — The Computer Science of Concurrency (Turing Lecture)](https://lamport.azurewebsites.net/pubs/turing.pdf)
- [Gray 등 — Granularity of Locks in a Large Shared Data Base](https://www.cs.purdue.edu/homes/bb/cs542-20Spr/readings/cc/granularity-locks.pdf)
- [Eswaran 등 — The Notions of Consistency and Predicate Locks in a Database System](https://www.csd.uoc.gr/~hy460/pdf/p624-eswaran.pdf)
- [Kung, Robinson — On Optimistic Methods for Concurrency Control](https://www.eecs.harvard.edu/~htk/publication/1981-tods-kung-robinson.pdf)
- [Martin Kleppmann — How to do distributed locking](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)
- [MySQL 8.4 Reference Manual — InnoDB Locking](https://dev.mysql.com/doc/refman/8.4/en/innodb-locking.html)
- [Redis Docs — Distributed Locks with Redis](https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/)
