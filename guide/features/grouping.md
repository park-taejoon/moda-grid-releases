# 행 그룹화 + 집계 (Row Grouping)

파이프라인 결과를 컬럼 값 기준 계층 구조로 변환한다. 그룹 헤더 행에
펼침/접힘과 집계 값이 표시된다.

## 코어 API

```ts
grid.setGroupBy(["dept", "role"]);   // 다단계 그룹화 기준 필드
grid.setGroupBy(null);               // 해제
grid.toggleGroupExpanded(key);       // 그룹 펼침/접힘
grid.expandAllGroups();
grid.collapseAllGroups();
grid.isGroupExpanded(key);
```

## 집계

`aggregationFn`이 선언된 컬럼만 그룹별로 집계된다:

```ts
{ field: "salary", header: "연봉", aggregationFn: "sum" }
// 'sum' | 'avg' | 'min' | 'max' | 'count'
```

- 그룹 노드의 `aggregates[field]`에 저장 (`count`=행 수, 나머지는 숫자 변환
  가능한 값만 합산).
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
import { buildGroupTree, flattenGroupTree, collectGroupKeys,
         aggregateRows, formatAggregate } from "@moda-grid/core";
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
