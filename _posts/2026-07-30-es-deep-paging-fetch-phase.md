---
title: "Elasticsearch 딥 페이징이 CPU를 태우고 있을 때 - fetch phase는 왜 비싼가"
date: 2026-07-30 10:00:00 +0900
categories: [Elasticsearch, Backend]
tags: [elasticsearch, deep-paging, search-after, pagination, fetch-phase, performance, backend]
toc: true
---

> 상품 검색의 깊은 페이징(deep paging) 루프가 공용 Elasticsearch 클러스터의 CPU를 90% 넘게 튀게 만들고 있었다. 원인을 추적해 보니 "쓰지도 않는 데이터를 매번 압축 해제하고 파싱하는" 구조였다. `_source`를 끄는 한 줄로 검색 시간의 대부분을 차지하던 fetch phase를 걷어낸 변경을 뜯어보며, 그 원인과 원리를 정리한다.
>
> *직접 구현한 변경이 아니라 팀에 올라온 PR을 리뷰하며 파고든 학습 기록입니다.*

---

## 1. 증상: 공용 클러스터 CPU 90%+ 스파이크

여러 서비스가 함께 쓰는 공용 Elasticsearch 클러스터의 CPU가 간헐적으로 90%를 넘겼다. 공용 클러스터라, 특정 서비스가 CPU를 태우면 클러스터에 올라탄 다른 서비스까지 같이 느려진다.

먼저 지금 이 순간 무슨 쿼리가 오래 돌고 있는지부터 잡아야 했다. `_cat/tasks`로 실행 중인 태스크를 포착했다.

```
indices[products]  140.4ms
{
  "from": 0, "size": 10000,
  "_source": { "includes": ["id"] },
  "sort": [{ "id": { "order": "desc" } }],
  "track_total_hits": 2147483647,
  "search_after": [123456789],
  "query": { ... 검색 조건 ... }
}
```

`from:0, size:10000, search_after:[...]` — 만 건씩 커서로 앞으로 걸어가는 전형적인 **딥 페이징 루프**였다. 그리고 이 요청 모양은 이 서비스의 딥 페이징 처리 로직(아래에서 설명할 `getAfterKeyWhenOverMaximumPage`)과 정확히 일치했다.

같은 시점의 fetch 태스크도 같이 잡혔다. (샤드별 fetch 건수를 합치면 10,000건이다.)

```
indices:data/read/search[phase/fetch/id]  120.4ms  size[3323]
indices:data/read/search[phase/fetch/id]  120.1ms  size[3330]
```

시간을 쪼개보면 이렇게 나뉜다.

```
전체 140.4ms
├─ query phase   약  20ms  (14%)
└─ fetch phase   약 120ms  (86%)   ← 여기가 범인
```

인덱스 통계로도 이 인덱스는 fetch가 자기 검색 시간의 **약 60%**를 차지하고 있었다. 즉 "검색"이라고 부르는 시간의 절반 이상이 실제로는 fetch에서 새고 있었다.

---

## 2. 배경 지식: Elasticsearch 검색은 2단계다

왜 fetch가 이렇게 비싼지 이해하려면 ES 검색이 두 단계로 나뉜다는 걸 알아야 한다.

```
                    검색 요청 1건
                         │
                         ▼
   ┌───────────────────────────────────────────────┐
   │  query phase — "무엇이 매칭되고, 어떤 순서인가"    │
   │    · 조건에 맞는 문서 매칭 + 정렬(sort) 계산       │
   │    · 정렬 값은 doc values(컬럼형 저장소)에서 읽음   │   저렴  ≈ 14%
   │    · 산출물: 문서 ID + 정렬 키  (본문은 아직 안 읽음) │
   └───────────────────────────────────────────────┘
                         │  상위 문서 ID 목록
                         ▼
   ┌───────────────────────────────────────────────┐
   │  fetch phase — "실제 문서 내용을 가져온다"          │
   │    · _source를 디스크에서 읽기                    │
   │    · 압축 해제 → JSON 파싱                        │   비쌈  ≈ 86%  ← 범인
   │    · includes 필터로 필요한 필드만 추출            │
   └───────────────────────────────────────────────┘
                         │
                         ▼
                        응답
```

### query phase — "무엇이 매칭되고, 어떤 순서인가"

- 각 샤드가 쿼리 조건에 맞는 문서를 찾고, **정렬(sort)** 을 계산한다.
- 정렬 값은 `_source`(원본 JSON)가 아니라 **doc values**(컬럼형 저장소)에서 읽는다. (숫자·날짜·keyword 등 doc values를 갖는 필드 기준. `text` 필드 정렬은 fielddata를 쓰지만, 이 글의 정렬 키는 숫자형 `id`라 doc values로 처리된다.)
- 결과로 나오는 건 문서 ID와 정렬 키뿐. 문서 본문은 아직 안 읽는다.

### fetch phase — "실제 문서 내용을 가져온다"

- query phase에서 추린 문서 ID들에 대해 **저장된 `_source`를 디스크에서 읽어 → 압축 해제 → JSON 파싱** 한다.
- 그다음 `_source` 필터(`includes`)를 적용해 필요한 필드만 남긴다.

여기서 오해하기 쉬운 지점이 있다.

> **`_source: { includes: ["id"] }`는 서버 CPU를 줄여주지 않는다.**

`includes`는 **응답으로 내보내는 양(네트워크 전송량)만** 줄인다. 서버는 여전히 문서 **전체**를 압축 해제하고 파싱한 다음, 그중 `id`만 골라서 버린다. 압축 해제 + 파싱이라는 비싼 일은 이미 다 해버린 뒤다.

---

## 3. 진짜 원인: 쓰지도 않는 데이터를 매 회차 파싱하고 있었다

문제의 루프를 보자. `from + size`가 10,000을 넘는 깊은 페이지를 처리하려고, 10,000건씩 `search_after`로 앞으로 걸어가면서 **다음 커서로 쓸 정렬 키만** 뽑아내는 코드다.

> ES는 `from + size`가 `index.max_result_window`(기본 **10,000**)를 넘으면 검색을 거부한다. 그래서 그 너머의 페이지를 조회하려면 커서(`search_after`)로 원하는 위치까지 앞으로 걸어가야 한다.

```kotlin
// 변경 전
private suspend fun getAfterKeyWhenOverMaximumPage(request: ProductSearchRequest): List<FieldValue> {
    val sortOptions = SortBuilder.getSort(request.sort)
    val sourceConfig = SourceConfig.of { source -> source.filter { it.includes(request.returnValue) } }

    var currentSortKeys = mutableListOf<FieldValue>()
    var maxRequestCount = request.offset / MAXIMUM_PAGING_COUNT   // 10,000
    val remainder = request.offset % MAXIMUM_PAGING_COUNT
    if (remainder > 0) maxRequestCount += 1

    for (i in 1..maxRequestCount) {
        val loopRequest = SearchRequest.Builder()
            .index(PRODUCT_ALIAS)
            .trackTotalHits { it.enabled(true) }   // 매 회차 전체 건수 정확히 카운트
            .source(sourceConfig)                  // 매 회차 _source 10,000건 파싱
            .from(0)
            .size(MAXIMUM_PAGING_COUNT)
            .sort(sortOptions)
            .query(request.toEsQuery())
            .apply {
                if (currentSortKeys.isNotEmpty()) this.searchAfter(currentSortKeys)
            }.build()

        val result = esClient.search(loopRequest, ProductSearchResult::class.java).join()

        if (i == 1 && request.offset > result.hits().total()!!.value()) break

        currentSortKeys = /* result.hits().hits().last().sort() 로 다음 커서 계산 */
    }
    return currentSortKeys
}
```

이 루프가 실제로 사용하는 값은 딱 하나, **`hit.sort()`(다음 커서로 쓸 정렬 키)** 뿐이다. `_source`(문서 본문)는 단 한 번도 읽지 않는다.

그런데 매 회차 10,000건의 `_source`를 디스크에서 읽어 압축 해제하고 파싱하고 있었다. **쓰지도 않을 데이터를 위해 검색 시간의 86%를 태우고 있었던 것.**

---

## 4. 해결 (1): `_source`를 꺼서 fetch phase를 통째로 스킵

정렬 키는 query phase 산출물이라 `_source`가 없어도 hit에 실려 온다. 그러니 fetch phase 자체가 필요 없다.

```kotlin
// 변경 후
.trackTotalHits { it.enabled(i == 1) }
.source { it.fetch(false) }              // 변경 전: .source(sourceConfig)
```

`_source: false`로 두면 ES는 fetch phase에서 **문서 본문을 아예 읽지 않는다.** 압축 해제도, JSON 파싱도 없다. 그런데도 `result.hits().hits().last().sort()`는 그대로 동작한다 — 정렬 값은 query phase에서 doc values로 계산되어 hit 메타데이터에 붙어 오기 때문이다.

즉 커서 로직은 그대로 둔 채, 지배적 비용(86%)만 사라진다.

> ⚠️ 주의: 루프가 끝난 뒤 **실제 데이터를 조회하는 최종 쿼리**는 `_source`가 진짜로 필요하다. 그래서 그 쿼리는 그대로 둔다. `_source`를 끄는 건 "커서만 걸어가는 중간 루프"에 한정한 최적화다.

---

## 5. 해결 (2): 전체 건수 카운팅은 첫 회차에만

`track_total_hits: true`는 ES가 **매칭되는 전체 문서 수를 끝까지 정확히 세게** 만든다. 정확한 총건수를 얻기 위해, 상위 문서만 추리고 조기에 멈추는 최적화(early termination)를 포기하는 것이다. 공짜가 아니다.

이 루프에서 전체 건수(`total()`)를 실제로 읽는 코드는 첫 회차의 이 한 줄뿐이다.

```kotlin
if (i == 1 && request.offset > result.hits().total()!!.value()) break
```

2회차부터는 아무도 `total()`을 읽지 않는다 (`hit.sort()`만 쓴다). 같은 쿼리라 총건수도 회차 간 변하지 않는다. 그래서 **첫 회차에만** 카운팅을 켠다.

```kotlin
.trackTotalHits { it.enabled(i == 1) }
```

여기서 "왜 첫 회차는 꼭 정확해야 하나"가 중요하다. `track_total_hits`를 기본값으로 두면 ES는 카운트를 **10,000에서 잘라서** `value=10000, relation="gte"`만 돌려준다. 그런데 이 루프는 **항상 offset > 10,000인 딥 페이징 구간**에서만 돈다. 잘린 값을 쓰면:

```
offset(예: 12,000) > total.value(잘린 10,000)  →  항상 true  →  무조건 break
```

→ **모든 딥 페이징이 "페이지 없음"으로 오판되어 빈 결과**가 나와버린다. 그래서 break 판정을 하는 첫 회차만큼은 정확한 전체 건수가 반드시 필요하다.

부가 효과도 있다. `i == 1 &&` 단락 평가 덕분에 `.total()!!` 역참조는 카운팅이 켜진 첫 회차에서만 일어나므로, 2회차 이후 `total()`이 null이어도 NPE가 나지 않는다. `enabled(i == 1)`과 `i == 1 &&` 가드가 정확히 세트로 맞물린다.

| 회차 | 정확한 total 필요? | trackTotalHits |
|---|---|---|
| `i == 1` | 필요 (break 판정 — 잘리면 오판) | `enabled(true)` |
| `i >= 2` | 불필요 (아무도 안 읽음) | `enabled(false)` → 카운팅 비용 제거 + early termination 허용 |

---

## 6. 해결 (3): offset 상한을 두고 search_after로 유도

(1)과 (2)로 **회차당 비용**은 크게 줄였다. 하지만 루프 **횟수 자체**는 여전히 `offset / 10,000`에 선형으로 비례한다. offset이 깊어질수록 순차 ES 왕복이 계속 늘어난다는 구조적 한계는 그대로다.

그래서 이 변경은 offset에 상한(50,000 = 루프 최대 5회)을 두고, 그 이상은 커서 기반 `search_after`로 유도한다.

```kotlin
override suspend fun searchProducts(request: ProductSearchRequest): ProductSearchResponse {
    var offset = request.offset
    var currentSortKeys = emptyList<FieldValue>()

    if (!request.searchAfter.isNullOrBlank()) {         // 커서를 들고 온 요청
        offset = 0
        currentSortKeys = request.searchAfter.split(",").map { FieldValue.of(it) }
    }

    // 딥 페이징 구간(10,000 초과)이고, 커서도 없는 경우에만 진입
    if ((request.offset + request.size > MAXIMUM_PAGING_COUNT) && currentSortKeys.isEmpty()) {
        // offset이 깊어질수록 10,000건 루프가 선형으로 늘어나므로 상한을 두고 searchAfter로 유도한다
        if (request.offset > MAXIMUM_SEARCH_AFTER_REQUIRED_OFFSET) {   // 50_000
            throw ExceededMaxOffsetException(MAXIMUM_SEARCH_AFTER_REQUIRED_OFFSET)
        }
        currentSortKeys = getAfterKeyWhenOverMaximumPage(request)
        offset = 0
        if (currentSortKeys.isEmpty()) return ProductSearchResponse()
    }
    // ... 이하 실제 데이터 조회 (여기선 _source 필요) ...
}
```

이 검증 위치가 설계의 핵심이다.

- **바깥 `if`의 의미**: `offset + size > 10,000`(= 이 페이지가 딥 페이징 구간) **AND** `currentSortKeys.isEmpty()`(= 호출자가 커서를 안 줌). 커서를 준 요청은 위 블록에서 `currentSortKeys`가 채워지므로 이 분기를 아예 안 탄다.
- **상한이 이 분기 안, 루프 호출 직전에** 놓여 있다. 그래서 상한은 **비싼 경로에만** 걸리고, 그 **탈출구가 정확히 `search_after` 커서 경로**(offset을 0으로 리셋하고 단일 쿼리로 끝나는 싼 경로)다.

즉 "막기"와 "유도"가 한 지점에서 동시에 성립한다. offset로 깊게 파고드는 요청은 막고, 그 대안(커서)을 쓰면 애초에 이 분기를 건너뛰어 O(1)로 처리된다.

> **"유도"는 자동 전환이 아니라 안내다.** 서버가 커서로 갈아타 주는 게 아니라, `throw`로 막고 메시지로 "searchAfter 쓰세요"라고 알릴 뿐 — 실제 전환은 **클라이언트 몫**이다. 그래서 클라이언트가 커서 페이징을 구현하지 않으면 유도가 아니라 그냥 "막힘"이 된다. 상한을 도입할 땐 에러 노출·전환 방식을 클라이언트와 함께 합의해야 한다.

값으로 확인하면 (페이지당 30건 기준):

| 요청 | offset | 바깥 if | offset > 50,000? | 결과 |
|---|---|---|---|---|
| 1페이지 | 0 | 안 탐 (30 ≤ 10,000) | — | 일반 from/size 쿼리 |
| 400페이지 | 11,970 | 진입 | ❌ | 루프 약 2회 |
| 2000페이지 | 59,970 | 진입 | ✅ | `ExceededMaxOffsetException` |
| search_after 커서 요청 | 0으로 리셋 | 안 탐 (커서 있음) | — | 단일 쿼리 (면제) |

그리고 검색 요청 생성 지점이 여러 곳이어도 전부 이 `searchProducts`로 수렴하고, 비싼 루프로 분기하는 조건문도 하나뿐이라 **초크포인트 한 곳**에 검증이 자리한다. 신규 호출자가 추가돼도 자동으로 걸린다.

에러는 별도 코드/메시지로 등록되고, 다국어 로케일 메시지도 함께 추가됐다.

```kotlin
class ExceededMaxOffsetException(maxOffset: Int) :
    SearchException(SearchErrorCode.EXCEEDED_MAX_OFFSET, arrayOf(maxOffset))
```

```properties
search.error.exceededMaxOffset=조회 가능한 최대 위치({0})를 초과했습니다. searchAfter 파라미터를 사용해 주세요.
```

---

## 7. 정리: 세 변경이 겨냥한 비용축

세 가지 변경이 각각 다른 비용축을 겨냥한다. 서로 중복이 없다.

| 변경 | 겨냥한 비용 | 효과 |
|---|---|---|
| `_source(false)` | **회차당** fetch phase (압축 해제 + 파싱) | 검색 시간의 86% 제거 (본체) |
| `trackTotalHits(i == 1)` | **회차당** 전체 카운팅 + early termination 차단 | 부수 절감 |
| `offset` 상한 + search_after 유도 | **루프 횟수**(offset에 선형) | 최악 케이스 백스톱 |

---

## 8. 배운 점

1. **`_source` 필터링(`includes`)은 CPU 최적화가 아니다.** 전송량만 줄일 뿐, 서버는 문서 전체를 이미 파싱한다. CPU를 줄이려면 아예 `_source: false`로 fetch phase를 끄거나, 정렬/집계처럼 doc values로 끝나는 경로로 설계해야 한다.
2. **정렬 값은 `_source` 없이도 온다.** 커서(`search_after`)만 필요한 구간이라면 문서 본문을 읽을 이유가 없다.
3. **`track_total_hits`는 공짜가 아니다.** 총건수를 실제로 쓰는 곳이 어딘지 확인하고, 필요한 회차에만 켜라. 단, 딥 페이징 구간에서는 잘린 카운트(기본 10,000 cap)가 오판을 일으킬 수 있으니 "정확값이 필요한 첫 판정"만큼은 켜둬야 한다.
4. **offset 기반 딥 페이징은 근본적으로 O(offset)이다.** 상한을 두고 `search_after` 커서 페이징으로 유도하는 게 정석이다. 그리고 그 상한 검증은 "비싼 경로로 분기하기 직전, 커서 경로는 면제되는 지점"에 두면 막기와 유도가 자연스럽게 한 곳에서 성립한다.