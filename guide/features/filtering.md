# 필터 / 검색 (Filtering & Search)

컬럼별 연산자 필터, 전역 검색, 엑셀식 Set 필터를 제공한다.

## 사용자 조작 (DataGrid 기본 동작)

`filterable: true`인 컬럼이 있으면 헤더 아래 **필터 입력 행**이 자동 렌더링된다:

- `filterType`에 따른 연산자 셀렉트 + 값 입력 (+ `inRange` 시 상한 입력)
- `filterType: "set"`이면 `<details>` 팝오버의 값 체크리스트(`.mg-setfilter`)

필터 행의 표시/숨김은 `filterToggle` prop(4개 어댑터 공통)으로 우상단
`필터 ▾/▸` 버튼을 켤 수 있다. 숨겨도 설정된 필터 조건은 계속 적용된다.

## 코어 API

```ts
grid.setFilter("name", "hana"); // 문자열 → contains
grid.setFilter("age", { operator: "greaterThan", value: "30" });
grid.setFilter("age", { operator: "inRange", value: "20", valueTo: "40" });
grid.setFilter("role", { operator: "set", values: ["admin"], value: "" });
grid.setFilter("name", null); // 해제
grid.clearFilters(); // 전체 해제

grid.setSearch("hana"); // 전역 검색

// 필터 입력 행 표시/숨김 (UI 표시 상태 — 필터 조건과 무관)
grid.setFilterRowVisible(false);
grid.toggleFilterRow();
new GridCore({ columns, data, filterRowVisible: false }); // 초기 숨김

// 뷰 일괄 초기화 — 정렬·필터·검색·페이지·행/셀 선택 모두 해제
// (데이터·컬럼 레이아웃·그룹화는 유지)
grid.resetView();
```

필터/검색 변경 시 `pageIndex`는 0으로 리셋된다. 복수 컬럼 필터는 **AND** 결합.

## 컬럼 옵션

```ts
{ field: "name", filterable: true }                           // 필터 행 표시
{ field: "age",  filterable: true, filterType: "number" }     // 연산자 타입
{ field: "role", filterable: true, filterType: "set" }        // 값 체크리스트
{ field: "x",    filterPredicate: (value, row, filter) => boolean }  // 커스텀 판별
{ field: "id",   searchable: false }                          // 전역 검색·찾기 제외
{ field: "code", getSearchText: (row) => row.internalCode }   // 표시값 대신 다른 텍스트로 검색
```

`searchable: false`는 **컬럼 필터와 무관**하다 — `setSearch` 전역 검색과
`findCells`/`replaceAll` 찾기에서만 제외된다. `getSearchText`는
포맷된 표시값 대신 검색 대상 문자열을 바꾼다 (예: 코드 컬럼을
원시 코드로 검색). 두 옵션 모두 컬럼 필터·정렬에는 영향을 주지 않는다.

## 연산자 목록 (`filterType`별)

| 타입          | 연산자                                      | 의미                                         |
| ------------- | ------------------------------------------- | -------------------------------------------- |
| `text` (기본) | `contains` `equals` `startsWith` `endsWith` | 문자열 비교 (대소문자 무시)                  |
| `number`      | `equals` `greaterThan` `lessThan` `inRange` | `Number()` 변환, `inRange`는 `value~valueTo` |
| `date`        | `equals` `before` `after`                   | "일" 단위 비교 (`equals`=같은 날)            |
| `set`         | `set`                                       | `values` 배열에 포함된 값만 통과             |

- `equals`는 스마트 판별: 양쪽 숫자면 숫자 비교, 양쪽 날짜면 같은 날 비교,
  아니면 문자열 동일 비교.
- `value`/`valueTo`가 모두 빈 조건은 항상 통과 — 연산자만 선택된 상태에서
  모든 행이 사라지지 않는다.
- 타입별 연산자 상수: `FILTER_OPERATORS_BY_TYPE`, `DEFAULT_OPERATOR_BY_TYPE`,
  `FILTER_OPERATOR_LABELS`.
- 순수 평가 함수 `matchFilterValue(value, filter)`도 export된다.

## Set 필터 (값 체크리스트)

```ts
grid.getUniqueValues("role"); // ["admin", "editor", ...] — 체크리스트 항목
grid.setFilter("role", { operator: "set", value: "", values: ["admin"] });
grid.setFilter("role", null); // 해제
```

- 비교는 셀 값의 문자열 표현(`String(value)`) 기준.
- `values`가 고유값 전체를 포함하면 체크리스트는 비활성(필터 없음과 동일).

## 고급 필터 빌더 — 조건식 + 체크리스트 AND/OR 결합

`ColumnFilter`는 **조건식**(operator + value/valueTo)과 **값 체크리스트**
(`values`)를 같은 모델 안에서 `join`으로 결합할 수 있다:

```ts
// 역할이 "admin"을 포함(조건) 하면서 값이 admin/editor 중 하나(체크리스트)
grid.setFilter("role", {
  operator: "contains",
  value: "admin",
  values: ["admin", "editor"],
  join: "and", // 기본값 — 조건과 체크리스트를 모두 만족
});

// OR — 조건 또는 체크리스트 둘 중 하나만 만족해도 통과
grid.setFilter("age", {
  operator: "greaterThan",
  value: "30",
  values: ["22"],
  join: "or",
});
```

규칙:

- `join`은 조건식과 체크리스트가 **둘 다 활성**일 때만 의미가 있다.
  조건식(value/valueTo 모두 공백)이나 체크리스트(빈 `values`)가 비어 있으면
  해당 부분은 없는 것으로 평가된다.
- `operator: "set"`이면 `values`만이 조건이다 — `join`은 무시된다.
- 컬럼 간 결합은 기존과 동일하게 항상 AND다.
- `filterPredicate`가 있는 컬럼은 결합 필터 전체를 받아 커스텀 판정한다.
- `getFilterModel()`/`setFilterModel()`/`getColumnFilter()`는 `values` 배열
  까지 깊은 복사로 주고받는다 — 반환값을 변형해도 내부 상태가 오염되지 않는다.

렌더러의 필터 빌더 UI가 쓰는 공용 헬퍼도 export된다:

```ts
import { buildColumnFilter, normalizeColumnFilter } from "@moda-grid/core";

// FilterBuilderInput { operator, value, valueTo, values, join } → ColumnFilter|null
buildColumnFilter({
  operator: "contains",
  value: "a",
  values: ["x"],
  join: "or",
});
// 조건·체크리스트 모두 비활성이면 null — setFilter(field, null)로 해제
buildColumnFilter({ operator: "contains", value: "" }); // → null
// 체크리스트만 있으면 레거시 { operator: "set" } 형태로 낸다
buildColumnFilter({ values: ["admin"] }); // → { operator: "set", value: "", values: ["admin"] }
```

## 헤더 필터 드롭다운 (`headerFilters`) — 필터 빌더 UI

필터 행과 별개로, `filterable` 컬럼 헤더에 `▾` 버튼을 달아 **필터 빌더
드롭다운**을 연다:

```tsx
// React/Vue3/Vue2/Svelte — prop
<DataGrid columns={cols} data={rows} headerFilters />
```

```ts
// vanilla
mountGrid(el, { columns, data, headerFilters: true });
```

드롭다운 구성 (`filterType: "set"`이 아닌 컬럼 기준):

1. **조건 섹션** — 연산자 셀렉트(`.mg-filter-cond-op`) + 비교 값 입력
   (`.mg-filter-cond-val`), `inRange`면 상한 입력(`.mg-filter-to`) 추가.
   값 입력은 blur/Enter에서 반영된다.
2. **결합 토글** — 조건식 ↔ 체크리스트를 `그리고(AND)`/`또는(OR)`로 묶는
   라디오(`.mg-filter-join-radio`). 로케일 키: `filterAnd`/`filterOr`.
3. **값 체크리스트** — `(전체 선택)` + `getUniqueValues` 값별 체크박스
   (`.mg-colctl-item`). 조건식과 독립 상태로 유지된다.

- `filterType: "set"` 컬럼은 체크리스트만 표시된다.
- 조건·체크리스트 모두 비어 있으면 필터는 자동 해제된다.
- 활성 필터가 있는 컬럼의 버튼은 강조 색(`mg-active`)으로 표시된다.
- 한 번에 하나만 열리며, 오버레이 클릭·`Esc`·버튼 재클릭으로 닫힌다.
- `filterRowVisible: false`로 필터 행을 숨겨도 헤더 드롭다운으로 필터할 수
  있다 — 빠른 필터 UX에 적합.

## 외부 필터 (`externalFilter`)

컬럼 필터·검색·행 상태 필터와 **AND로 조합되는 사용자 조건** — 그리드 바깥
UI(체크박스/기간 선택/권한 규칙 등)로 표시 범위를 제어할 때 쓴다
(타사 그리드 `externalFilter` 대응):

```ts
new GridCore({
  columns,
  data,
  externalFilter: (row) => row.age >= 30, // false면 행을 숨긴다
});
grid.setExternalFilter(fn); // 런타임 교체 — filterChange 이벤트 발행
grid.setExternalFilter(null); // 해제
```

- 컬럼 필터/검색 이후 마지막 단계에 적용 — 결과는 `filteredRowCount`·집계·
  내보내기에도 반영된다.
- 어댑터에서는 `externalFilter`/`external-filter` prop으로 전달하고
  반응형 교체도 지원한다.

## 서버 사이드 모드

`serverSide` 사용 시 필터/검색은 `filterModel: Record<field, ColumnFilter>`로
`getRows`에 전달된다 — 서버가 필터링한다. [server-side.md](./server-side.md) 참고.

## 관련 스냅샷 필드

`snapshot.filters: Record<field, ColumnFilter>` / `snapshot.searchText` /
`snapshot.filterRowVisible` (필터 행 표시 여부 — `getState`/`applyState`에도
포함) / `snapshot.filteredRowCount` (페이징 전 수) / `snapshot.totalRowCount`
(필터 전 수).
