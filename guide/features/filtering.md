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
grid.setFilter("name", "hana");                    // 문자열 → contains
grid.setFilter("age", { operator: "greaterThan", value: "30" });
grid.setFilter("age", { operator: "inRange", value: "20", valueTo: "40" });
grid.setFilter("role", { operator: "set", values: ["admin"], value: "" });
grid.setFilter("name", null);                      // 해제
grid.clearFilters();                               // 전체 해제

grid.setSearch("hana");                            // 전역 검색

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

## 헤더 필터 드롭다운 (`headerFilters`)

필터 행과 별개로, `filterable` 컬럼 헤더에 `▾` 버튼을 달아
엑셀 autofilter식 **값 체크리스트 드롭다운**을 연다:

```tsx
// React/Vue3/Vue2/Svelte — prop
<DataGrid columns={cols} data={rows} headerFilters />
```

```ts
// vanilla
mountGrid(el, { columns, data, headerFilters: true });
```

- 클릭 시 해당 컬럼의 고유 값 체크리스트(`getUniqueValues`)가 버튼 아래
  팝오버로 열린다 — `(전체 선택)` + 값별 체크박스.
- 토글은 즉시 `operator: "set"` 필터로 반영되고, 모든 값이 선택되면
  `setFilter(field, null)`로 자동 해제된다.
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
  columns, data,
  externalFilter: (row) => row.age >= 30, // false면 행을 숨긴다
});
grid.setExternalFilter(fn);    // 런타임 교체 — filterChange 이벤트 발행
grid.setExternalFilter(null);  // 해제
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
