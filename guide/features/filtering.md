# 필터 / 검색 (Filtering & Search)

컬럼별 연산자 필터, 전역 검색, 엑셀식 Set 필터를 제공한다.

## 사용자 조작 (DataGrid 기본 동작)

`filterable: true`인 컬럼이 있으면 헤더 아래 **필터 입력 행**이 자동 렌더링된다:

- `filterType`에 따른 연산자 셀렉트 + 값 입력 (+ `inRange` 시 상한 입력)
- `filterType: "set"`이면 `<details>` 팝오버의 값 체크리스트(`.mg-setfilter`)

## 코어 API

```ts
grid.setFilter("name", "hana");                    // 문자열 → contains
grid.setFilter("age", { operator: "greaterThan", value: "30" });
grid.setFilter("age", { operator: "inRange", value: "20", valueTo: "40" });
grid.setFilter("role", { operator: "set", values: ["admin"], value: "" });
grid.setFilter("name", null);                      // 해제
grid.clearFilters();                               // 전체 해제

grid.setSearch("hana");                            // 전역 검색
```

필터/검색 변경 시 `pageIndex`는 0으로 리셋된다. 복수 컬럼 필터는 **AND** 결합.

## 컬럼 옵션

```ts
{ field: "name", filterable: true }                           // 필터 행 표시
{ field: "age",  filterable: true, filterType: "number" }     // 연산자 타입
{ field: "role", filterable: true, filterType: "set" }        // 값 체크리스트
{ field: "x",    filterPredicate: (value, row, filter) => boolean }  // 커스텀 판별
```

## 연산자 목록 (`filterType`별)

| 타입 | 연산자 | 의미 |
| ---- | ------ | ---- |
| `text` (기본) | `contains` `equals` `startsWith` `endsWith` | 문자열 비교 (대소문자 무시) |
| `number` | `equals` `greaterThan` `lessThan` `inRange` | `Number()` 변환, `inRange`는 `value~valueTo` |
| `date` | `equals` `before` `after` | "일" 단위 비교 (`equals`=같은 날) |
| `set` | `set` | `values` 배열에 포함된 값만 통과 |

- `equals`는 스마트 판별: 양쪽 숫자면 숫자 비교, 양쪽 날짜면 같은 날 비교,
  아니면 문자열 동일 비교.
- `value`/`valueTo`가 모두 빈 조건은 항상 통과 — 연산자만 선택된 상태에서
  모든 행이 사라지지 않는다.
- 타입별 연산자 상수: `FILTER_OPERATORS_BY_TYPE`, `DEFAULT_OPERATOR_BY_TYPE`,
  `FILTER_OPERATOR_LABELS`.
- 순수 평가 함수 `matchFilterValue(value, filter)`도 export된다.

## Set 필터 (값 체크리스트)

```ts
grid.getUniqueValues("role");  // ["admin", "editor", ...] — 체크리스트 항목
grid.setFilter("role", { operator: "set", value: "", values: ["admin"] });
grid.setFilter("role", null);  // 전체 선택과 동일 = 해제
```

- 비교는 셀 값의 문자열 표현(`String(value)`) 기준.
- `values`가 고유값 전체를 포함하면 필터 해제와 동일.
- 어댑터는 `(전체 선택)` → `setFilter(field, null)`, 개별 토글 → `values`
  갱신 후 `operator: "set"`으로 재지정한다.

## 서버 사이드 모드

`serverSide` 사용 시 필터/검색은 `filterModel: Record<field, ColumnFilter>`로
`getRows`에 전달된다 — 서버가 필터링한다. [server-side.md](./server-side.md) 참고.

## 관련 스냅샷 필드

`snapshot.filters: Record<field, ColumnFilter>` / `snapshot.searchText` /
`snapshot.filteredRowCount` (페이징 전 수) / `snapshot.totalRowCount` (필터 전 수).
