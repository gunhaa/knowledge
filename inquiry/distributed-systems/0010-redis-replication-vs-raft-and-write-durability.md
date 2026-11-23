---
type: inquiry
created: 2026-11-23
verified: 2026-11-23
author: agent
model: claude-opus-5-5
---

# Redis 복제는 Raft가 아니다: 쓰기 응답 시점으로 본 Redis·DB·etcd, 그리고 정확성이 필요한 락

## 쉬운 설명

Redis는 숙제를 **받자마자 "받았어!"라고 대답**하고, 공책에 옮겨 적거나 친구에게 복사해 주는 건 나중에 한다. 그 사이에 넘어지면 받은 숙제 일부가 사라진다. etcd는 **반 친구 과반이 같이 적은 걸 확인한 뒤에야** 대답한다. 그래서 "받았다"고 한 건 사라지지 않는다. 사라지면 안 되는 약속(정확성이 필요한 락)은 Redis가 아니라 이렇게 확인하고 대답하는 쪽에 맡긴다.

## 일반 설명

### 1. Redis Cluster·Sentinel의 투표는 Raft가 아니다

Redis에도 과반 투표가 있다. Sentinel은 Sentinel끼리, Cluster는 마스터끼리 epoch마다 투표해 **장애 조치를 지휘할 리더나 승격할 레플리카**를 정한다. 겉모습은 Raft의 리더 선출과 비슷하다.

| | Raft (etcd) | Redis Sentinel / Cluster |
|---|---|---|
| 투표의 쓰임 | 리더 선출 **+ 모든 쓰기의 커밋** | **장애 조치 때 새 마스터 선출만** |
| 쓰기 확정 | 리더가 로그를 복제하고 **과반이 기록하면** 응답 | 마스터가 **자기 메모리에 반영하면** 바로 응답 |
| 레플리카 전파 | 커밋 전에 과반에 기록 | 응답 **후** 비동기 전파 |
| 리더 자격 | 커밋된 로그를 **반드시 가진** 노드만 당선 (로그 일치 규칙) | 복제 오프셋이 앞선 레플리카를 고르지만, 마스터의 마지막 쓰기를 못 받았어도 승격 가능 |
| 응답한 쓰기의 유실 | 과반이 살아 있으면 없음 | **장애 조치 때 있음** |

Raft의 핵심은 선출이 아니라 **로그 일치 규칙**(과반이 가진 로그만 커밋되고, 그 로그를 가진 노드만 리더가 된다)이다. Redis는 선출 부분만 비슷하게 쓰고 이 규칙은 없다. Jepsen 분석도 Redis에는 Raft의 term에 해당하는 구조가 없으며, 선형화하려면 Sentinel을 강한 일관성을 가진 조정자로 바꿔야 한다고 결론 냈다. Raft로 데이터까지 복제하려던 **Redis-Raft** 모듈은 별개 프로젝트이고 표준 배포 구성이 아니다.

`WAIT N`(레플리카 N대 수신 대기)은 유실 창을 줄이지만 합의는 아니다. 장애 조치 때 그 레플리카가 승격된다는 보장이 없고, 타임아웃이 나도 마스터 쓰기는 되돌리지 않는다.

복제·Sentinel·Cluster가 서로 어떤 관계인지(복제 위에 자동 장애 조치 = Sentinel, 거기에 샤딩 = Cluster)와 구성별 운영 포인트는 [Redis 장애 대응 — 세 구성의 관계](../../knowledge/database/0002-redis/05-failure-modes.md#3-복제--sentinel--cluster-세-구성의-관계)에 있다.

### 2. "응답을 언제 주는가"로 보면 차이가 보인다

"동기 복제냐 비동기 복제냐"만 보면 헷갈린다. **DB도 기본 복제는 비동기**이기 때문이다. 두 축으로 나눠 봐야 한다.

| | ① 응답 전 로컬 디스크 fsync | ② 레플리카 복제 | 응답 시점 |
|---|---|---|---|
| **Redis** | 기본 안 함. **기본 설정은 RDB(스냅샷)만** 켜져 있어 마지막 스냅샷 이후(수 분)가 유실 대상. AOF를 켜고(`appendonly yes`) `everysec`이면 최대 약 1초 | **항상 비동기**. `WAIT`/`WAITAOF`(7.2+)로 쓰기마다 선택해 기다림 | 메모리 반영 후 |
| **MySQL InnoDB** | **기본 함** (`innodb_flush_log_at_trx_commit=1`) | 기본 비동기. semi-sync·Group Replication으로 동기 | 로컬 fsync 후 |
| **PostgreSQL** | **기본 함** (`synchronous_commit=on`) | 기본 비동기. `synchronous_standby_names` 지정 시 동기 | 로컬 fsync 후 |
| **etcd** | 함 | **Raft 과반 기록** | 과반 기록 후 |

유실이 생기는 장면으로 정리하면:

| 장면 | Redis | DB (기본 설정) | etcd |
|---|---|---|---|
| 노드 1대 재시작 | 기본(RDB만)이면 마지막 스냅샷 이후, AOF `everysec`이면 약 1초분 유실 | 커밋된 것 남음 | 남음 |
| 장애 조치 (레플리카 승격) | 복제 안 된 쓰기 유실 | 비동기 레플리카면 마지막 일부 커밋 유실 | 과반 생존 시 유실 없음 |

Redis가 이렇게 설계된 이유는 응답 속도다. 쓰기마다 fsync와 과반 합의를 하면 수 ms 이상이 걸린다. 원래 용도가 캐시·세션·카운터처럼 **잃어도 다시 만들 수 있는 데이터**였기 때문에, 약간의 유실을 감수하고 1ms 미만 응답을 얻는 쪽을 기본값으로 골랐다. `appendfsync always`로 로컬 내구성은 올릴 수 있지만, 장애 조치에 합의가 없다는 점은 그대로라 etcd 수준의 보장은 되지 않는다.

### 3. 그래서 Redis 락은 정확성을 보장하지 못한다

1. A가 마스터에서 `SET lock NX PX 30000` → "OK"
2. 레플리카로 복제되기 전에 마스터가 죽는다.
3. 레플리카가 승격된다. 여기엔 락 키가 없다.
4. B도 `SET lock NX` → "OK". A와 B가 동시에 보유한다.

여기에 보유자가 GC로 멈춘 사이 TTL이 끝나는 문제까지 더해지면, Redis 락은 두 보유자를 구조적으로 막을 수 없다. 그래서 **효율용**(뚫려도 같은 일을 한 번 더 할 뿐인 곳)으로 분류된다 → [락 개론 — 효율용인가, 정확성용인가](../../knowledge/cs/0001-lock/03-tradeoffs.md#3-효율용인가-정확성용인가).

**정확성이 필요하면 보호 대상에 따라 고른다.**

| 보호 대상 | 수단 | 이유 |
|---|---|---|
| **DB 데이터** (재고, 결제, 선착순) | DB 트랜잭션·유니크 제약·조건부 `UPDATE` | 데이터와 같은 곳에서 원자적으로 막는다. 락 서비스를 하나 더 두지 않아도 된다 |
| **여러 시스템에 걸친 조정** (리더 선출, 외부 자원 하나를 한 곳만 쓰게) | **etcd·ZooKeeper + fencing token** | 락 기록이 과반 합의로 확정돼 장애 조치에도 사라지지 않는다 |

### 4. etcd는 무엇이고, Redis 락 대신 쓸 수 있나

etcd는 **작은 설정·메타데이터를 강한 일관성으로 저장하는 분산 key-value 저장소**다. CNCF 졸업 프로젝트이고, Kubernetes가 클러스터 상태를 전부 etcd에 둔다. 보통 3대나 5대로 Raft 클러스터를 만든다.

| | Redis 락 | etcd 락 (`concurrency.Mutex`) |
|---|---|---|
| 락이 유지되는 근거 | 마스터 1대의 메모리 | Raft 과반 기록 |
| 해제 보장 | TTL | 세션 lease (keepalive가 끊기면 만료) |
| fencing token | 기본 없음 (`INCR`, `RFencedLock`으로 따로) | **기본 제공**: 락 키의 생성 revision이 단조 증가 |
| 대기 | 재시도 또는 Pub/Sub | 앞 순번 키를 watch → 순서대로(공정) |
| 획득 비용 | 1ms 미만 | 수 ms~수십 ms (fsync + 과반 합의) |
| 처리량 | 매우 높음 | 낮음 (대량 락에는 맞지 않음) |
| 운영 | 이미 Redis가 있으면 추가 비용 거의 없음 | 3~5대 클러스터를 따로 운영, 디스크 지연에 민감 |

- **효율용**이면 Redis로 충분하다. etcd는 과하다.
- **정확성용 + 시스템 간 조정**이면 etcd가 맞다.
- etcd도 lease 기반이라, 보유자가 멈춘 사이 lease가 끝나면 이중 보유가 생길 수 있다. 차이는 revision이라는 토큰을 기본으로 준다는 점이므로, **보호 대상이 그 토큰을 검사해야** 실제로 안전해진다 → [락 개론 — Fencing token](../../knowledge/cs/0001-lock/02-overview.md#8-fencing-token).

---

### 관련 문서

- [Redis 장애 대응 — 복제 · Sentinel · Cluster](../../knowledge/database/0002-redis/05-failure-modes.md#3-복제--sentinel--cluster-세-구성의-관계)
- [Redis 장애 대응 — 데이터 유실과 스플릿 브레인](../../knowledge/database/0002-redis/05-failure-modes.md#4-데이터-유실과-스플릿-브레인)
- [Redis 락의 역사 — Redlock 논쟁](../../knowledge/database/0005-redis-lock/01-history.md#3-redlock-논쟁의-요지)
- [Redis 락 트레이드오프 — 대안 비교](../../knowledge/database/0005-redis-lock/03-tradeoffs.md#3-대안-비교)
- [락 개론 — 트레이드오프](../../knowledge/cs/0001-lock/03-tradeoffs.md)
- [고가용성: Replica와 Cluster는 무엇을 정의하나](./0009-high-availability-replication-vs-cluster-and-patterns.md)

### Sources

- [Redis cluster specification](https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/)
- [Valkey — Cluster specification](https://valkey.io/topics/cluster-spec/)
- [Jepsen: Redis redux (aphyr)](https://aphyr.com/posts/307-jepsen-redis-redux)
- [Jepsen: Redis-Raft 1b3fbf6](https://jepsen.io/analyses/redis-raft-1b3fbf6)
- [How Sentinel Leader Election Works in Redis — OneUptime](https://oneuptime.com/blog/post/2026-03-31-redis-how-sentinel-leader-election-works-in-redis/markdown)
- [Redis persistence demystified — antirez](https://oldblog.antirez.com/post/redis-persistence-demystified.html)
- [How to Use WAITAOF in Redis for AOF Sync Guarantee — OneUptime](https://oneuptime.com/blog/post/2026-03-31-redis-waitaof-aof-sync-guarantee/markdown)
- [MySQL Replication — MySQL Reference Manual](https://dev.mysql.com/doc/refman/8.4/en/replication.html)
- [PostgreSQL — Synchronous Replication](https://www.postgresql.org/docs/current/warm-standby.html#SYNCHRONOUS-REPLICATION)
- [go.etcd.io/etcd/client/v3/concurrency — pkg.go.dev](https://pkg.go.dev/go.etcd.io/etcd/client/v3/concurrency)
- [How to Build a Fenced Distributed Lock with etcd Leases and Revisions — OneUptime](https://oneuptime.com/blog/post/2026-09-09-etcd-fenced-lock-leases-revisions/markdown)
