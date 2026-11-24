---
type: knowledge
created: 2026-11-24
verified: 2026-11-24
author: agent
model: claude-opus-5-5
---

# 원장의 역사

## 쉬운 설명

옛날 상인들은 돈이 어디서 와서 어디로 갔는지 헷갈려 손해를 봤다. 그래서 거래 하나를 **두 칸에 같은 금액으로** 적기 시작했다. 한쪽에 "현금이 나갔다", 다른 쪽에 "물건이 들어왔다"를 적으면 양쪽 합이 항상 같아야 하므로, 틀린 곳이 바로 드러난다. 700년쯤 지나 컴퓨터가 장부를 맡게 됐지만, 개발자들은 결국 같은 결론에 다시 도착했다. **지우지 말고 덧붙여라, 그리고 양쪽 합을 맞춰라.**

## 일반 설명

> 이 세트는 원장을 **수량·가치가 옮겨 다니는 모든 도메인**(돈, 포인트, 재고, 크레딧)의 정합성 구조로 다룬다. 동시성 관점에서는 [락 세트](../../cs/0001-lock/01-history.md)가 "어떻게 막나"를, 이 세트가 "막는 게 실패해도 결과가 맞으려면"을 맡는다. 용어는 [개요 2절](./02-overview.md#2-용어).

### 1. 등장 배경 (14세기): 거래가 많아진 상인

- 지중해 무역이 커지면서 상인은 여러 도시·동업자·외상 거래를 동시에 관리해야 했다. 한 줄 메모식 장부로는 "돈이 왜 이만큼 남았나"를 설명할 수 없었다.
- 거래 하나를 차변·대변 두 쪽에 같은 금액으로 적는 **복식부기**가 이탈리아 상업 도시에서 자리 잡았다. 논란 없이 남아 있는 가장 이른 예는 **1340년 제노바 시 재무관(Massari)의 장부**다. 그보다 이른 토스카나 장부를 근거로 기원을 더 앞으로 보는 연구도 있다.
- **Pacioli(1494)**가 『산술 총서』(*Summa de Arithmetica*)에서 베네치아식 복식부기를 설명했다. 메모장 → 분개장(journal) → 원장(ledger) 세 권을 쓰는 방식이다. 처음 쓴 사람이 아니라 **처음 널리 인쇄해 퍼뜨린 사람**이다.

이때 정해진 규칙이 지금도 그대로 쓰인다.

| 규칙 | 풀려던 문제 | 지금의 대응물 |
|---|---|---|
| 거래마다 차변 합 = 대변 합 | 빠지거나 잘못 적은 금액 찾기 | 거래별 검사, 시산표 → [개요 4절](./02-overview.md#4-기록-범위-잔액만-단식부기-복식부기) |
| 지우지 않고 반대 기록으로 정정 | 사후 조작, 근거 소실 | 덧붙이기만 하는 기입 테이블, 반대 거래 → [개요 3절](./02-overview.md#3-구조와-불변성) |
| 분개장(시간순) → 원장(계정별) | 시간순 기록과 계정별 잔액을 함께 보기 | 거래 테이블 + 계정별 기입 |

### 2. 주요 변천

| 시기 | 사건 | 풀려던 문제 | 지금 어디에 남았나 |
|---|---|---|---|
| 1340 | 제노바 Massari 장부 | 공공 재정의 추적 | 논란 없는 가장 이른 복식부기 장부 |
| 1494 | Pacioli, 『산술 총서』 | 복식부기의 표준화·전파 | 회계 교육의 출발점 |
| 2005 | Fowler, *Accounting Entry*·*Event Sourcing* 패턴 | 상태만 저장하면 "왜 이 값인가"를 잃음 | 잔액 = 기입 합, 정정 = 반대 거래(Reversal) |
| 2015 | Helland, "Immutability Changes Everything" (CIDR) | 분산 시스템에서 갱신이 일관성 문제를 만듦 | "회계사는 지우개를 쓰지 않는다", "진실은 로그이고 DB는 그 일부의 캐시" |
| 2019 | Square, *Books* (불변 복식부기 원장 서비스) 공개 | 결제 서비스마다 제각각인 잔액 관리 | 사내 공용 원장 서비스 |
| 2020 | TigerBeetle 시작 (Coil 내부 프로젝트) | 범용 DB 위 원장의 핫 계정·처리량 한계 | 복식부기 전용 DB, 2단계 전송 |
| 2021 | Modern Treasury Ledgers 출시 | 원장을 직접 만드는 비용 | 원장 API, `lock_version` 낙관적 락 |
| 2024 | Stripe *Ledger*, Uber *LedgerStore* 공개 | 하루 수십억 건 규모의 돈 흐름 검증 | 불변 이벤트 로그 + 복식부기, 대규모 인덱스 |
| 2025 | Amazon QLDB 지원 종료 (7월 31일) | — | 전용 "불변 DB" 상품보다 범용 DB + 원장 설계로 |

흐름을 한 줄로 줄이면: **종이 장부의 규칙**(복식부기, 지우지 않기) → **소프트웨어 패턴으로 재발견**(이벤트 소싱, 불변성) → **대규모 서비스와 전용 시스템**(Stripe, TigerBeetle). 규칙은 700년 동안 거의 그대로이고, 바뀐 것은 처리량과 동시성이다.

### 3. 현재 상태 (2026년 11월)

| 방식 | 대표 | 특징 |
|---|---|---|
| 범용 RDBMS에 직접 설계 | MySQL, PostgreSQL | 가장 흔하다. 제약·트랜잭션으로 정합성 → [트레이드오프 3절](./03-tradeoffs.md#3-동시성-제어-분산-락이-뚫려도-정확하게) |
| 사내 원장 서비스 | Stripe Ledger(하루 약 50억 이벤트), Square Books, Uber LedgerStore | 결제 규모 회사가 공용 플랫폼으로 운영 |
| 전용 원장 DB | TigerBeetle | 계정과 전송 두 가지만 저장. 128비트 정수 금액, 한 코어 + 묶음 처리. 2025년 Jepsen 검증에서 충돌·질의 버그가 보고되어 수정됨 |
| 오픈소스 원장 | Formance Ledger | 거래를 Numscript DSL로 기술 |
| 원장 API 서비스 | Modern Treasury Ledgers | 확정·대기·사용 가능 잔액, `lock_version`, 멱등키 |

- **덧붙이기만 + 복식부기가 기본값이다.** 공개된 대규모 원장은 모두 이 조합을 쓴다. Stripe는 정정도 이전 작업을 되돌리고 다시 처리하는 방식으로 하고, Modern Treasury는 확정된 거래를 불변으로 둔다.
- **전용 "불변 DB" 상품은 물러났다.** QLDB 종료 때 AWS는 Aurora PostgreSQL을 대안으로 안내했다. 원장은 저장소 종류보다 **설계 규칙**의 문제라는 쪽으로 정리되고 있다.
- **남은 과제는 핫 계정이다.** Uber는 2026년 한 계정의 처리량을 초당 3~4건에서 30건 이상으로 올리면서 계정 나누기를 버리고 250ms 묶음 처리를 골랐다 → [트레이드오프 3.3](./03-tradeoffs.md#33-핫-계정).

## 관련 문서

- [원장 개요와 동작 방식](./02-overview.md)
- [원장 트레이드오프와 선택 기준](./03-tradeoffs.md)
- [원장 사용 시 흔한 함정과 해결 방법](./04-pitfalls.md)
- [원장 장애 유형과 대응](./05-failure-modes.md)
- [락의 역사](../../cs/0001-lock/01-history.md)

## 참고 자료

- [Encyclopedia.com — Accounting and Bookkeeping](https://www.encyclopedia.com/history/encyclopedias-almanacs-transcripts-and-maps/accounting-and-bookkeeping)
- [De Computis — Cartullarium massariorum (1340) 재검토 (2025)](https://www.upo.es/revistas/index.php/decomputis/en/article/view/11927)
- [Chicago Booth Review — The birth of modern accounting and finance](https://www.chicagobooth.edu/review/birth-modern-accounting-finance)
- [Martin Fowler — Accounting Entry (2005)](https://martinfowler.com/eaaDev/AccountingEntry.html)
- [Martin Fowler — Event Sourcing (2005)](https://martinfowler.com/eaaDev/EventSourcing.html)
- [Pat Helland — Immutability Changes Everything (CACM 2016, CIDR 2015)](https://cacm.acm.org/practice/immutability-changes-everything)
- [Square — Books, an immutable double-entry accounting database service (2019)](https://developer.squareup.com/blog/books-an-immutable-double-entry-accounting-database-service/)
- [Modern Treasury — Why we built Ledgers (2021)](https://moderntreasury.com/journal/why-we-built-ledgers)
- [TigerBeetle — Company](https://www.tigerbeetle.com/company)
- [Stripe — Ledger: Stripe's system for tracking and validating money movement (2024)](https://stripe.dev/blog/ledger-stripe-system-for-tracking-and-validating-money-movement)
- [Uber — How LedgerStore Supports Trillions of Indexes (2024)](https://www.uber.com/us/en/blog/how-ledgerstore-supports-trillions-of-indexes/)
- [Jepsen — TigerBeetle 0.16.11 (2025)](https://jepsen.io/analyses/tigerbeetle-0.16.11)
- [AWS — Amazon QLDB end of support](https://docs.aws.amazon.com/general/latest/gr/full_shutdown_services.html)
- [Uber — Building High Throughput Payment Account Processing (2026)](https://www.uber.com/us/en/blog/high-throughput-processing/)
- [Formance Ledger (GitHub)](https://github.com/formancehq/ledger)
