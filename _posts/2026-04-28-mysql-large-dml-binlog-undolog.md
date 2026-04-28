---
title: "MySQL 대량 DML이 위험한 이유 — Binary Log와 Undo Log 관점"
date: 2026-04-28 18:00:00 +0900
categories: [Database, MySQL]
tags: [mysql, binary-log, undo-log, replication, bulk-dml]
toc: true
---

---

## 들어가며

[이전 글](/posts/mysql-rr-to-rc/)에서 상품 서비스가 RC를 채택한 이유를 다뤘다. 대량 재고를 청크 단위로 나눠서 병렬 처리한다는 이야기도 했다.

그런데 왜 굳이 청크로 나눠야 할까? 한 번에 100만 건을 UPDATE하면 안 되는 걸까?

이 질문에 답하려면 MySQL이 데이터를 변경할 때 **내부에서 어떤 일이 일어나는지** 알아야 한다. 핵심은 두 가지 로그다.

- **Binary Log**: 변경 사항을 기록해서 복제에 사용
- **Undo Log**: 변경 전 데이터를 보관해서 MVCC와 롤백에 사용

---

## Binary Log: 복제의 핵심

### Binary Log란

MySQL에서 데이터를 변경하는 모든 작업(INSERT, UPDATE, DELETE, DDL)은 **Binary Log**에 기록된다. SELECT 같은 조회는 기록하지 않는다.

Binary Log의 역할은 두 가지다.

1. **복제(Replication)**: Master의 변경 사항을 Slave로 전파
2. **복구(Point-in-Time Recovery)**: 백업 이후의 변경 사항을 재적용

### 기록 방식: ROW vs STATEMENT

Binary Log에는 변경 사항을 기록하는 포맷이 있다.

| 포맷 | 기록 내용 | 크기 | 안정성 |
|---|---|---|---|
| STATEMENT | SQL 문장 자체 | 작음 | 낮음 |
| ROW | 변경된 행의 실제 데이터 | 큼 | 높음 |
| MIXED | 상황에 따라 자동 선택 | 중간 | 높음 |

**ROW 포맷**이 실무에서 가장 많이 쓰인다. STATEMENT 포맷은 `NOW()` 같은 비결정적 함수에서 Master와 Slave의 값이 달라질 수 있기 때문이다.

```sql
-- STATEMENT 방식: SQL 문장을 기록
UPDATE product SET price = price * 1.1 WHERE category_id = 100;
-- → Slave에서 같은 SQL을 실행
-- → 비결정적 함수가 포함되면 결과가 달라질 수 있음

-- ROW 방식: 변경된 각 행의 전/후 데이터를 기록
-- → product_no=1: price 100 → 110
-- → product_no=2: price 200 → 220
-- → 정확한 값을 기록하므로 안전
```

### 대량 DML이 Binary Log에 미치는 영향

여기서 중요한 점이 있다. **Binary Log는 트랜잭션이 COMMIT될 때 한 번에 기록된다.**

트랜잭션이 진행되는 동안에는 메모리 캐시(`binlog_cache_size`, 기본 32KB)에 변경 내용을 쌓아두다가, COMMIT 시점에 Binary Log 파일에 쓴다. 캐시를 초과하면 임시 파일(디스크)을 사용한다.

```sql
-- 100만 건 UPDATE를 한 트랜잭션으로 실행하면

START TRANSACTION;

UPDATE product
   SET price = price * 1.1
 WHERE category_id = 100;
-- → 100만 건의 변경 데이터가 메모리/임시파일에 쌓임

COMMIT;
-- → 이 시점에 Binary Log에 한 번에 기록
-- → ROW 포맷이면 100만 행의 전/후 데이터가 모두 기록됨
```

**ROW 포맷에서 100만 건 UPDATE**가 발생하면, 각 행의 변경 전/후 데이터가 전부 기록된다. 수백 MB ~ 수 GB의 Binary Log가 한 번에 생성될 수 있다.

### 복제 지연(Replication Lag)

Binary Log는 **COMMIT 이후에** Slave로 전송된다. 대량 DML 트랜잭션은 COMMIT까지 오래 걸리고, COMMIT 후에도 Slave가 대량의 Binary Log를 재실행하는 데 시간이 걸린다.

```sql
-- Master: 100만 건 UPDATE (10분 소요)
--   ↓ COMMIT 후 Binary Log 전송
-- Slave: 100만 건 Binary Log 재실행 (10분+ 소요)
--   ↓
-- 총 복제 지연: 20분 이상
```

상품 서비스처럼 **읽기는 Slave로 보내는 구조**에서 복제 지연은 곧 데이터 불일치를 의미한다. Master에서 재고를 업데이트했는데, Slave에서 조회하면 이전 재고가 보일 수 있다.

---

## Undo Log: MVCC와 롤백의 핵심

### Undo Log란

[이전 글](/posts/mysql-rr-to-rc/)에서 MVCC를 설명하면서 "이전 버전을 Undo Log에 보관한다"고 했다. 좀 더 구체적으로 보자.

InnoDB가 데이터를 변경하면, **변경 전 데이터**를 Undo Log에 기록한다. 이 Undo Log는 두 가지 용도로 쓰인다.

1. **롤백(Rollback)**: 트랜잭션을 취소할 때 Undo Log를 역순으로 적용해서 원래 상태로 되돌림
2. **MVCC 읽기**: 다른 트랜잭션이 변경 전 데이터를 읽을 수 있도록 이전 버전을 제공

### History List Length

커밋된 트랜잭션의 Undo Log는 바로 삭제되지 않는다. 다른 트랜잭션이 MVCC 읽기를 위해 아직 참조할 수 있기 때문이다. 더 이상 아무도 참조하지 않게 되면 **Purge 스레드**가 Undo Log를 제거한다.

**History List Length(HLL)** 는 아직 Purge되지 않은 커밋된 트랜잭션의 Undo Log 수다. 정상 상태에서는 수천 이하이고, 이 값이 급증하면 Purge가 쌓이는 속도를 따라가지 못한다는 의미다.

### 대량 DML이 Undo Log에 미치는 영향

100만 건을 한 트랜잭션으로 UPDATE하면 어떤 일이 일어나는지 단계별로 보자.

**실행 중:**

```sql
-- 100만 건 UPDATE 진행 중 (아직 COMMIT 안 함)

-- → 100만 건의 변경 전 데이터가 Undo Log에 기록됨
-- → Undo Tablespace 크기 급증
-- → 이 트랜잭션이 끝날 때까지
--    다른 커밋된 트랜잭션의 Undo Log도 Purge 불가
```

**COMMIT 후:**

```sql
-- COMMIT 완료

-- → 그동안 쌓인 Undo Log를 Purge 스레드가 정리 시작
-- → History List Length가 높은 상태에서 서서히 감소
-- → Purge 완료까지 모든 SELECT가
--    더 많은 Undo 버전을 스캔해야 함
-- → 읽기 성능 저하
```

**핵심 문제**: HLL이 높으면 SELECT 시 InnoDB가 현재 버전에서 시작해서 Undo Log 체인을 따라가며 자신의 Read View에 맞는 버전을 찾아야 한다. 체인이 길수록 읽기가 느려진다.

---

## Binary Log + Undo Log 종합 비교

| 영향 | 한 트랜잭션에 대량 DML | 청크 분할 |
|---|---|---|
| Binary Log 크기 | 수백 MB 한 번에 기록 | 청크당 수 MB씩 분산 |
| 복제 지연 | 트랜잭션 종료까지 전송 안 됨 | 청크 COMMIT마다 즉시 전송 |
| Undo Log 팽창 | 전체 건수만큼 한번에 쌓임 | 청크 단위로 Purge 가능 |
| HLL 영향 | 장시간 급증 | 낮게 유지 |
| 롤백 시간 | 전체 롤백 (수십 분) | 해당 청크만 롤백 |
| Lock 점유 | 장시간 대량 row lock | 짧게 잡고 빠르게 해제 |

### 디스크 풀 위험

대량 INSERT나 UPDATE를 실행하면 Binary Log 파일이 급증한다. Binary Log는 지정된 주기에 따라 자동 삭제되지만, **복제에서 사용 중인 파일은 삭제할 수 없다.** 복제 지연이 길어지면 그만큼 오래된 Binary Log 파일이 유지되면서 디스크 사용량이 계속 증가한다.

---

## 상품 서비스의 대응: 청크 분할

[이전 글](/posts/mysql-rr-to-rc/)에서 소개한 청크 처리 패턴이 왜 필요한지 이제 명확하다.

```kotlin
// 대량 재고 upsert 흐름

suspend fun bulkUpsertStocks(
    stocks: List<StockModel>,
) {
    stocks
        .chunked(CHUNK_SIZE)           // N개씩 나누고
        .splitIntoEqualParts(PARALLEL) // M개 그룹으로 분배
        .map { group ->
            async {
                upsertStocks(group)    // 병렬 실행
            }
        }
        .awaitAll()
}
```

10만 건의 재고를 업데이트한다고 하면:

| | 한 번에 10만 건 | 1000건 × 100 청크 |
|---|---|---|
| 트랜잭션 크기 | 10만 건 | 1000건 |
| Binary Log | COMMIT 시 한 번에 기록 | 100번 나눠서 기록 |
| Undo Log | 10만 건 쌓인 후 Purge | 1000건씩 바로 Purge 가능 |
| 복제 지연 | 전체 완료 후 전송 | 청크마다 즉시 전송 |
| 실패 시 | 10만 건 전체 롤백 | 해당 청크만 재시도 |

### RC가 여기서도 도움이 되는 이유

이전 글에서 RC를 선택한 이유 중 하나가 Gap Lock 회피였다. Undo Log 관점에서 한 가지가 더 있다.

RR에서는 트랜잭션이 처음 만든 Read View를 계속 유지하기 때문에, 그 시점 이후에 커밋된 모든 Undo Log를 Purge할 수 없다. RC에서는 매 SELECT마다 새로운 Read View를 만들기 때문에, **Purge가 더 빨리 진행**될 수 있다.

---

## 정리

대량 DML을 한 트랜잭션으로 실행하면:

1. **Binary Log**: COMMIT까지 Slave로 전송되지 않아 복제 지연 발생, ROW 포맷에서는 로그 크기도 급증
2. **Undo Log**: 변경 전 데이터가 대량으로 쌓이고, Purge가 밀려서 읽기 성능 저하
3. **디스크**: Binary Log 파일 급증으로 디스크 풀 위험

대응은 단순하다.

- **청크 단위로 나눠서 각각 COMMIT**: Binary Log 분산, Undo Log 즉시 Purge 가능
- **RC 격리수준**: Purge가 빠르게 진행될 수 있도록
- **읽기/쓰기 트랜잭션 분리**: 쓰기 트랜잭션을 최대한 짧게 유지