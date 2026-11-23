---
type: knowledge
created: 2026-11-23
author: agent
model: claude-opus-5-5
---

# 락의 역사

## 쉬운 설명

옛날 컴퓨터는 한 번에 한 가지 일만 해서 다툴 일이 없었다. 한 컴퓨터가 여러 일을 번갈아 하게 되자, 두 일이 같은 칠판에 동시에 글씨를 써서 글자가 뒤섞이는 문제가 생겼다. 그래서 "칠판을 쓰는 동안은 분필을 하나만 둔다"는 약속(락)이 생겼다. 그 뒤로 칠판이 데이터베이스가 되고, 다시 여러 건물(서버)에 흩어진 칠판이 되면서 약속도 그에 맞게 바뀌어 왔다.

## 일반 설명

> 이 세트는 Java·MySQL·Redis 락 세트가 공통으로 기대는 **개념**을 다룬다. 각 솔루션의 구현과 역사는 해당 세트에 있다 — [Java 락](../../java/0002-java-lock/01-history.md), [MySQL 락](../../database/0004-mysql-lock/01-history.md), [Redis 락](../../database/0005-redis-lock/01-history.md).

### 1. 등장 배경 (1960년대): 한 기계 안의 동시 실행

- 1960년대 **멀티프로그래밍**·시분할 운영체제가 등장하면서, 여러 프로세스가 메모리·장치 같은 자원을 함께 쓰게 됐다.
- 두 실행 흐름이 같은 데이터를 "읽고 → 고치고 → 쓰는" 사이에 끼어들면 결과가 틀어진다. 이것이 **경쟁 조건**(race condition)이다.
- **Dijkstra(1965)** 가 이 문제를 **상호 배제**(mutual exclusion) 문제로 정식화했다. "임계 구역(critical section)에는 한 번에 하나만 들어간다"는 조건과, 들어가려는 쪽이 영원히 기다리지 않아야 한다는 조건을 함께 요구했다. Lamport는 이 논문을 동시성 컴퓨터과학의 출발점으로 꼽는다.
- 같은 시기 Dijkstra는 **세마포어**(P/V 연산)를 내놓았다. 정수 하나로 "들어갈 수 있는 수"를 세는 장치이고, 1이면 상호 배제(뮤텍스), N이면 동시 사용 수 제한이 된다.

### 2. 주요 변천

| 시기 | 사건 | 풀려던 문제 | 지금 어디에 남았나 |
|---|---|---|---|
| 1965 | Dijkstra: 상호 배제 문제, 세마포어 | 공유 자원의 동시 수정 | OS 뮤텍스, Java `Semaphore` |
| 1971 | Coffman 등: 데드락 **4조건** 정리 | 서로 기다리다 영원히 멈춤 | 데드락 예방·감지 설계의 기준 → [장애 유형](./05-failure-modes.md#2-데드락) |
| 1973~74 | Brinch Hansen·Hoare: **모니터** | 세마포어는 짝(P/V)을 빼먹기 쉬움 → 락과 조건 대기를 언어 구조로 묶음 | Java `synchronized` + `wait/notify`, pthread mutex + condition |
| 1974 | Lamport: 빵집(bakery) 알고리즘 | 하드웨어 원자 연산 없이 상호 배제 | 이론적 의미. 실무는 CPU 원자 명령(CAS)을 쓴다 |
| 1975~76 | Gray 등: **락 단위(granularity)와 의도 락** | 테이블 전체 vs 행 하나 — 굵게 잠그면 느리고 잘게 잠그면 관리 비용 | InnoDB IS/IX 의도 락 |
| 1976 | Eswaran·Gray 등: **2단계 락(2PL)**, predicate lock | 트랜잭션 직렬화, 팬텀 | DB 락은 커밋까지 보유. predicate lock의 근사가 next-key lock |
| 1981 | Kung·Robinson: **낙관적 동시성 제어** | 충돌이 드물면 락 대기 자체가 낭비 | `@Version`, CAS, 조건부 UPDATE |
| 1978~81 | MVCC (Reed, Bernstein·Goodman) | 읽기가 쓰기를 막음 | InnoDB·PostgreSQL 일관 읽기 — 읽기는 락 없이 |
| 1989 | Gray·Cheriton: **lease** | 분산 환경에서 보유자가 죽으면 락이 영원히 남음 | TTL 락(Redis), etcd lease |
| 1991 | Herlihy: wait-free 동기화 | 락 보유자가 멈추면 모두 멈춤 | lock-free 자료구조, `ConcurrentHashMap`, Atomic |
| 1997 | Mars Pathfinder **우선순위 역전** 사고 | 낮은 우선순위 작업이 락을 쥐어 높은 우선순위 작업이 멈춤 | 우선순위 상속 → [장애 유형](./05-failure-modes.md#4-우선순위-역전) |
| 2006 | Google **Chubby** 락 서비스 | 여러 서버 사이 상호 배제, 리더 선출 | 세션·lease + sequencer(fencing) 개념 |
| 2008~14 | ZooKeeper 공개, etcd(2013)·Raft(2014) | Chubby의 오픈소스 대안, 합의 기반 조정 | 정확성용 분산 락·리더 선출 |
| 2016 | Kleppmann의 Redlock 비판과 antirez 반론 | TTL 락은 멈춘 보유자 문제로 상호 배제를 보장할 수 없다 | "분산 락은 **효율용**, 정확성은 **fencing·DB 제약**" 원칙 → [트레이드오프](./03-tradeoffs.md#3-효율용인가-정확성용인가) |

흐름을 한 줄로 줄이면: **한 기계의 스레드**(세마포어·모니터) → **한 DB의 트랜잭션**(2PL·의도 락·MVCC) → **여러 서버**(lease·합의·fencing). 층이 바뀔 때마다 새 문제가 하나씩 더해졌다.

| 층 | 더해진 문제 | 더해진 장치 |
|---|---|---|
| 스레드 | 동시 수정 | 상호 배제, 조건 대기 |
| DB 트랜잭션 | 여러 행에 걸친 일관성, 팬텀, 읽기 성능 | 2PL, 범위 잠금, MVCC |
| 분산 | 보유자가 죽거나 **멈춰도** 모름 | lease, 합의, fencing token |

### 3. 현재 상태 (2026년 11월)

- **바닥은 원자 연산이다.** 모든 층의 락은 결국 하나의 원자 연산 위에 선다 — CPU의 CAS, DB 내부 래치, Redis의 단일 스레드 명령 실행, etcd의 Raft 로그.
- **락을 덜 쓰는 쪽으로 간다.** MVCC(읽기 무락), 불변 객체, 키 파티셔닝(같은 키는 한 작업자만), 액터·단일 작성자 패턴, 원자적 조건부 UPDATE가 락보다 먼저 검토된다.
- **분산 락은 용도가 나뉘었다.** 중복 작업을 줄이는 효율용은 Redis, 정확성이 필요한 조정은 etcd·ZooKeeper와 fencing token, 데이터 정합성은 DB 제약·트랜잭션이 맡는다.
- 언어 쪽에서는 가상 스레드처럼 스레드가 매우 많아지는 모델이 퍼지면서, 락 보유 중 블로킹 비용과 공정성이 다시 중요해졌다 (Java는 [JEP 491](../../java/0002-java-lock/01-history.md#2-주요-변천)).

## 관련 문서

- [락 개요와 공통 구조](./02-overview.md)
- [락 트레이드오프와 선택 기준](./03-tradeoffs.md)
- [락 사용 시 흔한 함정과 해결 방법](./04-pitfalls.md)
- [락 장애 유형과 대응](./05-failure-modes.md)
- [Java 락의 역사](../../java/0002-java-lock/01-history.md)
- [MySQL 락의 역사](../../database/0004-mysql-lock/01-history.md)
- [Redis 락의 역사](../../database/0005-redis-lock/01-history.md)

## 참고 자료

- [Lamport — The Computer Science of Concurrency (Turing Lecture)](https://cacm.acm.org/research/turing-lecture-the-computer-science-of-concurrency)
- [Lampson — Programming Concurrent Systems](https://www.microsoft.com/en-us/research/wp-content/uploads/2023/03/Lampson-PCS-Word-reconciled-with-book.docx)
- [Gray, Lorie, Putzolu, Traiger — Granularity of Locks in a Large Shared Data Base (VLDB 1975)](https://www.cs.purdue.edu/homes/bb/cs542-20Spr/readings/cc/granularity-locks.pdf)
- [Eswaran, Gray, Lorie, Traiger — The Notions of Consistency and Predicate Locks in a Database System (CACM 1976)](https://www.csd.uoc.gr/~hy460/pdf/p624-eswaran.pdf)
- [Kung, Robinson — On Optimistic Methods for Concurrency Control (TODS 1981)](https://www.eecs.harvard.edu/~htk/publication/1981-tods-kung-robinson.pdf)
- [Monitors의 기원과 역사 (Swansea University)](https://cronfa.swan.ac.uk/Record/cronfa63086/Download/63086__27357__12ce44cad4f04658bad7fa33b776c39d.pdf)
- [Martin Kleppmann — How to do distributed locking (2016)](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)
