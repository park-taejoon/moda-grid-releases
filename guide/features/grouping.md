# 행 그룹화 + 집계 (Row Grouping)

파이프라인 결과를 컬럼 값 기준 계층 구조로 변환한다. 그룹 헤더 행에
펼침/접힘과 집계 값이 표시된다.

## 코어 API

```ts
grid.setGroupBy(["dept", "role"]); // 다단계 그룹화 기준 필드
grid.setGroupBy(null); // 해제
grid.toggleGroupExpanded(key); // 그룹 펼침/접힘
grid.expandAllGroups();
grid.collapseAllGroups();
grid.isGroupExpanded(key);
```

## 집계

`aggregationFn`이 선언된 컬럼만 그룹별로 집계된다:

```ts
{ field: "salary", header: "연봉", aggregationFn: "sum" }
// 'sum' | 'avg' | 'min' | 'max' | 'count' | 'first' | 'last'
```

- `first`/`last`는 정렬 순서 기준 첫/마지막 행의 **원시 값**을 반환한다 —
  상태·최신 메모 같은 비숫자 컬럼 집계에 쓴다.
- 그룹 노드의 `aggregates[field]`에 저장 (`count`=행 수, `sum`/`avg`/
  `min`/`max`는 숫자 변환 가능한 값만 합산).

### 커스텀 집계 함수

함수를 직접 전달하면 행 전체에 접근해 어떤 집계든 만들 수 있다:

```ts
{
  field: "progress",
  aggregationFn: ({ values }) =>
    `${values.filter((v) => Number(v) >= 80).length}명 우수`,
},
// 가중평균·중복 제거처럼 행이 필요하면 rows/field도 쓴다
{ field: "score",
  aggregationFn: ({ rows, field }) =>
    rows.reduce((a, r) => a + Number(r[field]) * r.weight, 0) },
```

- 파라미터: `{ values, rows, field }` — `values`는 해당 컬럼의 원시 값 목록
  (행 순서), `rows`는 집계 대상 행 전체.
- 반환값은 문자열을 포함해 무엇이든 가능 — `formatAggregate`가 표시한다.
- 함수에서 예외가 나면 `null`로 집계된다 (셀이 비어 보임).
- 피벗 측정값의 `agg`에도 동일하게 쓸 수 있다 — 단 패널 UI의 집계 셀렉트는
  내장 함수만 노출하므로 함수는 코드로 지정한다.
- 표시 포맷은 `formatAggregate()` 헬퍼 (소수 2자리 반올림).
- `aggregationFn` 컬럼이 하나라도 있으면 `snapshot.grandTotals`에
  전체 총계가 계산되어 `<tfoot>`에 표시된다 ([pinned-rows.md](./pinned-rows.md)).

## 사용자 조작 (DataGrid 기본 동작)

`snapshot.displayRows`가 null이 아니면 그룹 모드로 렌더링된다:

- 그룹 헤더 행(`.mg-group-row`): 클릭 → `toggleGroupExpanded`, 첫 컬럼에
  `▾/▸` 토글 + `field: value (N)` 라벨, 나머지 컬럼에 집계 값.
- 리프 행: `depth`만큼 첫 셀 들여쓰기. `rowIndex`는 `visibleData` 기준이라
  기존 선택/편집 인덱스와 호환된다.
- 소계 행(`.mg-subtotal-row`): `groupSubtotals` 옵션 시 펼쳐진 그룹 끝에
  표시 — 첫 컬럼에 "소계: field: value" 라벨, 집계 컬럼에 소계 값.
- 가상 스크롤 병용 시 `displayRows.slice(startIndex, endIndex)`를 사용하고
  `virtual.totalHeight`는 그룹 헤더를 포함한 평탄 행 수 기준이다.

## 그룹 패널 (`groupPanel`)

그리드 상단에 그룹 상태 바를 표시한다 — 어댑터 prop과 `mountGrid` 옵션
모두 지원:

```tsx
<DataGrid groupPanel />; // React / Vue / Svelte 동일
mountGrid(el, { columns, data, groupPanel: true });
```

- 그룹된 컬럼이 칩으로 나열되고 **칩 클릭 시 해당 그룹이 해제**된다.
- **컬럼 헤더를 패널로 드래그해 드롭하면 그룹에 추가**된다
  (타사 그리드 그룹 패널과 동일) — 이미 그룹된 필드는 무시되고, 드롭 중에는
  `mg-drop-target` 하이라이트가 표시된다. 헤더 드래그는
  `application/x-mg-column` 데이터 타입을 사용한다.
- 그룹이 없으면 `locale.groupPanelEmpty` 안내 문구가 표시된다.
- 우클릭 메뉴로도 추가 가능 — 헤더 우클릭 →
  `defaultHeaderContextMenuItems`의 "컬럼으로 그룹화"/"그룹 해제" 항목
  또는 `setGroupBy` API.

## 동작 규칙

- `GroupNode`의 `key`는 전체 경로를 JSON 직렬화한 고유 문자열.
- `expandedRowKeys: Set<string>` — 펼친 그룹 키 집합. **새 그룹은 기본
  펼침**, 사용자가 접은 그룹은 `setData` 후에도 접힘 유지.
- 로컬 페이징과는 병용되지 않고, 서버 사이드 모드에서는 적용되지 않는다.
- 행 드래그는 그룹 모드에서 자동 비활성된다.

## 스냅샷 필드

`snapshot.groupBy` / `snapshot.expandedRowKeys` / `snapshot.displayRows`
(`GroupNode | LeafDisplayRow` 평탄 목록, 그룹 모드가 아니면 `null`) /
`snapshot.grandTotals`.

## 순수 함수 (커스텀 렌더링용)

```ts
import {
  buildGroupTree,
  flattenGroupTree,
  collectGroupKeys,
  aggregateRows,
  formatAggregate,
} from "@moda-grid/core";
```

## 그룹 소계 행 (groupSubtotals)

```ts
new GridCore({ columns, data, groupSubtotals: true });
// 어댑터: <DataGrid groupSubtotals />
```

`setGroupBy`로 그룹화하면 **펼쳐진 각 그룹의 마지막**에 소계 행이 붙는다.
집계는 그룹 헤더와 같은 `aggregates`를 공유하므로 `aggregationFn`이 선언된
컬럼에만 값이 표시된다. 접힌 그룹에는 소계가 없다 — 헤더 행의 집계가
그 역할을 한다. 타입은 `SubtotalDisplayRow` (`type: "subtotal"`).
