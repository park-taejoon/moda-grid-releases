# 정렬 / 다중 정렬 (Sorting)

헤더 클릭으로 정렬하고, Shift+클릭으로 다중 정렬 조건을 추가한다.

## 사용자 조작 (DataGrid 기본 동작)

| 조작 | 동작 |
| ---- | ---- |
| 헤더 클릭 | `asc → desc → 해제` 순환 (기존 조건 전부 교체) |
| Shift + 헤더 클릭 | 다중 정렬 조건 추가/순환/제거 — 화살표 옆에 우선순위 배지(`.mg-sort-priority`) 표시 |

## 코어 API

```ts
grid.toggleSort("age");          // 단일 정렬 (기존 조건 교체)
grid.toggleSort("age", true);    // additive — 다중 정렬에 추가
grid.setSort("name", "desc");    // 단일 조건으로 교체 ("asc" | "desc" | null)
grid.clearSorts();               // 전체 해제
```

## 다중 정렬 규칙

`snapshot.sortState: SortSpec[]` — `{ columnKey, direction, priority }`.
`priority` 순으로 체인 비교한다 (앞 조건이 같으면 다음 조건으로).

```ts
grid.toggleSort("role");         // priority 1
grid.toggleSort("age", true);    // priority 2
grid.toggleSort("age", true);    // → desc
grid.toggleSort("age", true);    // → 조건 제거, priority 재정규화
```

`snapshot.sort`는 첫 번째 조건만 노출해 단일 정렬 코드와 호환된다.

## 컬럼 옵션

```ts
{ field: "age", sortable: false }              // 정렬 비활성
{ field: "name", comparator: (a, b) => ... }   // 커스텀 비교 (a,b는 행)
```

- 기본 비교는 `Intl.Collator(numeric: true)` — 숫자/날짜/문자열 자연 비교.
- 정렬은 파이프라인의 필터/검색 **이후**에 적용된다.

## 서버 사이드 모드

`serverSide` 사용 시 로컬 정렬은 건너뛰고 `sortModel: SortSpec[]`이
`dataSource.getRows` 파라미터로 전달된다 — 서버가 정렬을 수행한다.
변경 시 캐시를 폐기하고 재요청한다. [server-side.md](./server-side.md) 참고.

## 관련 스냅샷 필드

`snapshot.sort` / `snapshot.sortState` — 현재 정렬 조건.
