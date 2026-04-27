---
title: "MySQL RR 을 RC 로 바꾸는 이유"
date: 2026-04-24 18:00:00 +0900
categories: [Database, MySQL]
tags: [mysql, isolation-level, gap-lock, mvcc, read-committed]
toc: true
---

---

## 배경

MySQL InnoDB의 기본 격리수준은 REPEATABLE READ(RR)이다.
하지만 상품 서비스는 READ COMMITTED(RC)를 사용한다.

```yaml
# HikariCP 커넥션 초기화 시 RC 설정
spring:
  datasource:
    connection-init-sql: SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED
    always-send-set-isolation: false
```

왜 기본값을 바꿔서 쓰는 걸까? 이걸 이해하려면 두 격리수준이 **내부적으로 뭐가 다른지**부터 알아야 한다.

---

## MVCC: 두 격리수준의 공통 기반

InnoDB는 row를 UPDATE해도 이전 데이터를 덮어쓰지 않는다. **Undo Log**에 이전 버전을 보관한다.

```
[상품 테이블 - product_no=1]

현재 버전:    sale_price=200, trx_id=50
  ↓ undo log
이전 버전:    sale_price=100, trx_id=30
```

하나의 row에 여러 버전(Multi-Version)이 동시에 존재한다. 각 트랜잭션은 **Read View**를 만들어서 "내가 어느 시점까지의 데이터를 볼 수 있는지"를 결정한다. lock 없이 과거 버전을 읽으므로 읽기와 쓰기가 서로 블로킹하지 않는다.

RC와 RR의 차이는 **Read View를 언제 만드느냐** 딱 하나다.

| | RC | RR |
|---|---|---|
| Read View 생성 | 매 SELECT마다 새로 생성 | 트랜잭션 최초 SELECT 시 1번만 |
| 결과 | 항상 최신 커밋 데이터 | 트랜잭션 시작 시점 데이터 고정 |

```
TX-A 시작
SELECT sale_price → 100

TX-B: UPDATE sale_price = 200; COMMIT;

TX-A: SELECT sale_price
  RC → 200 (Read View 새로 만들어서 최신 반영)
  RR → 100 (처음 Read View 유지)
```

---

## 진짜 차이: Gap Lock

Read View 차이는 사실 실무에서 크게 문제되지 않는다. 상품 서비스에서 같은 트랜잭션 안에서 같은 상품을 두 번 읽으며 일관성을 보장해야 하는 시나리오는 거의 없기 때문이다.

**진짜 중요한 차이는 Gap Lock이다.**

RR에서 범위 조건으로 데이터를 잠그면, 기존 row뿐 아니라 **범위 사이의 빈 공간까지 lock**을 건다.

```sql
-- RR에서 이 쿼리를 실행하면
SELECT * FROM product WHERE mall_no = ? FOR UPDATE;

-- mall_no 범위 전체에 gap lock
-- → 해당 몰에 새 상품 INSERT 불가 (BLOCKED)
```

RC에서는 gap lock이 걸리지 않는다. 기존 row만 잠근다.

---

## 상품 서비스에서 이게 왜 중요한가

상품 서비스의 특성을 생각해보자.

**같은 몰 안에서 동시에 일어나는 일들:**
- 셀러가 새 상품을 등록한다 (INSERT)
- 다른 상품의 판매 상태를 변경한다 (UPDATE)
- 재고를 업데이트한다 (UPDATE)
- 상품 목록을 조회한다 (SELECT)

이게 전부 같은 몰 범위 안에서 동시에 빈번하게 발생한다.

만약 RR이었다면:

```
TX-A: SELECT * FROM product WHERE mall_no = ? FOR UPDATE;
  → 해당 몰 범위 전체 gap lock

TX-B: INSERT INTO product (mall_no, ...) VALUES (?, ...);
  → BLOCKED! (gap lock 때문에 대기)

TX-C: UPDATE product SET sale_status = 'SOLD_OUT' WHERE mall_no = ? AND product_no = ?;
  → 역시 대기 가능
```

셀러 입장에서는 "상품 등록이 왜 이렇게 느리지?" 가 된다.

RC에서는:

```
TX-A: SELECT * FROM product WHERE mall_no = ? FOR UPDATE;
  → 기존 row만 record lock

TX-B: INSERT INTO product (mall_no, ...) VALUES (?, ...);
  → OK! 바로 실행

TX-C: UPDATE product SET ... WHERE product_no = ?;
  → 해당 row에 lock이 잡혀있으면 대기, 아니면 바로 실행
```

---

## 실제 코드에서 보는 패턴

### 재고 업데이트: 읽기/쓰기 트랜잭션 분리

상품 서비스는 재고 업데이트 시 **읽기와 쓰기 트랜잭션을 분리**한다.

```kotlin
// 재고 업데이트 흐름
suspend fun updateStock(params: List<StockParam>, mallNo: Int) {

    // 1. 읽기 전용 트랜잭션 — Slave DB
    val options = readOnlyTransaction {
        optionRepository.findByOptionNos(params.map { it.optionNo })
    }

    // 2. 검증 (트랜잭션 밖에서)
    val validRequest = validate(options, params)

    // 3. 쓰기 트랜잭션 — Master DB
    if (validRequest.isNotEmpty()) {
        stockService.updateStocks(validRequest)
    }
}
```

읽기는 Slave로, 쓰기만 Master로 보낸다. 이 패턴에서 RC가 맞는 이유:
- 읽기 트랜잭션은 어차피 독립적 — RR의 스냅샷 일관성이 필요 없음
- 쓰기 트랜잭션은 짧게 — gap lock으로 다른 트랜잭션을 블로킹할 이유 없음

### 대량 재고 처리: 청크 + 병렬

```kotlin
// 대량 재고 upsert 흐름
suspend fun bulkUpsertStocks(stocks: List<StockModel>) {
    stocks
        .chunked(CHUNK_SIZE)             // N개씩 나누고
        .splitIntoEqualParts(PARALLEL)   // M개 그룹으로 분배
        .map { group ->
            async { upsertStocks(group) }  // 병렬 실행
        }
        .awaitAll()
}
```

N개씩 청크로 나눠서 M개의 코루틴이 동시에 재고를 업데이트한다. 이 상황에서 RR이었다면:
- M개의 동시 트랜잭션이 각각 범위 lock을 잡음
- 서로의 gap lock과 충돌 → **deadlock 가능성 급증**

RC에서는 각 트랜잭션이 실제로 업데이트하는 row만 잠그므로 병렬 처리가 안전하다.

### FOR UPDATE를 쓰지 않는 설계

상품 서비스는 `FOR UPDATE`를 사용하지 않는다. 대신:

1. **읽기 전용 트랜잭션**으로 데이터 조회 (lock 안 잡음)
2. **애플리케이션 레벨에서 검증**
3. **쓰기 트랜잭션**에서 업데이트 (필요한 row만 lock)

이 Optimistic한 접근 자체가 RC와 궁합이 맞는다. RR + FOR UPDATE 조합이 필요 없는 구조.

---

## connectionInitSql을 쓰는 이유

RC 설정 방법이 세 가지 있는데, `connectionInitSql`을 쓴다.

| 방법 | SET 실행 횟수 | 단점 |
|---|---|---|
| `@Transactional(isolation = RC)` | 매 트랜잭션마다 | ProxySQL에서 대량 라운드트립 → CPU 증가 |
| **`connectionInitSql`** | **커넥션 생성 시 1회** | **없음 (권장)** |
| MySQL 서버 기본값 변경 | 0회 | 서버 설정 변경 필요, 다른 서비스에 영향 |

HikariCP가 커넥션 풀에서 새 커넥션을 만들 때만 SET 명령을 실행한다. 커넥션 수가 수십~수백 개이므로 SET 횟수도 그 정도. `@Transactional` 방식은 쿼리 수만큼 SET이 날아가서 수천만 번이 될 수 있다.

---

## Phantom Read 엣지 케이스

RR이 phantom read를 "대부분" 방지한다고 하는데, 완전하지는 않다.

```
TX-A: SELECT * FROM product WHERE price BETWEEN 100 AND 200  →  3건
TX-B: INSERT INTO product(no=99, price=150); COMMIT;
TX-A: UPDATE product SET price=price WHERE no=99
  → UPDATE는 current read (최신 데이터) → B가 삽입한 row를 A의 스냅샷에 편입
TX-A: SELECT * FROM product WHERE price BETWEEN 100 AND 200  →  4건 (phantom!)
```

UPDATE/DELETE는 스냅샷이 아닌 **current read**를 하기 때문에, 다른 트랜잭션이 삽입한 row를 건드리면 그 row가 이후 SELECT에서도 보이게 된다. `SELECT ... FOR UPDATE`를 쓰면 gap lock으로 INSERT 자체를 막아서 완전 방지 가능하지만, 그건 동시성을 희생하는 것이다.

---

## 정리

| | RC (우리 선택) | RR (MySQL 기본) |
|---|---|---|
| Gap Lock | 없음 | 있음 |
| 동시 INSERT/UPDATE | 자유로움 | 범위 lock으로 블로킹 |
| Deadlock 가능성 | 낮음 | 높음 (특히 병렬 처리 시) |
| 스냅샷 일관성 | 없음 (매번 최신) | 있음 |
| 우리에게 필요한가? | 최신성 + 성능 | 거의 불필요 |

상품 서비스는:
- 같은 몰에서 등록/수정/조회가 동시에 빈번하게 발생
- 대량 재고를 청크 단위로 병렬 업데이트
- 읽기/쓰기 트랜잭션을 분리하는 Optimistic 패턴
- FOR UPDATE 없이 애플리케이션 레벨 검증

이 구조에서 RR의 gap lock은 순수한 오버헤드다. 그래서 상품 서비스는 RC를 채택해서 사용한다.