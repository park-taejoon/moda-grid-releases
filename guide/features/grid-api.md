# 그리드 편의 API (Grid Convenience API)

IBSheet·AG-Grid 같은 엔터프라이즈 그리드에서 매일 쓰는 조회·모델·스크롤
패턴을 `GridCore`에 동일하게 제공한다. 모두 기존 상태를 조합한 얇은
래퍼이므로 React/Vue/Vue2/Svelte/`mountGrid` 어디서든 동일하게 동작한다.

```ts
const { grid } = useGridCore({ columns, data }); // 또는 mountGrid().grid
```

## 행 조회 — 표시 행 vs 전체 행

"표시 행"은 필터·검색·정렬·페이징이 모두 반영된 결과다
(`snapshot.rows`/`visibleData`와 동일). 세 개의 카운트를 구분해 사용한다.

```ts
grid.getDisplayedRowCount(); // 현재 표시 행 수 (현재 페이지)
grid.getFilteredRowCount(); // 필터+검색 통과 후 전체 수 (페이징 이전)
grid.getTotalRowCount(); // 원본 데이터 전체 수 (필터 무관)

grid.getDisplayedRowAt(0); // 표시 순서 첫 행 → TData | undefined
grid.getDisplayedRowIndex("row-42"); // 행 ID → 표시 인덱스 (없으면 -1)

// 순회 — 표시 순서 vs 원본 순서
grid.forEachDisplayedRow((row, i) => console.log(i, row));
grid.forEachRow((row, i) => console.log(i, row));
```

"표시 인덱스"는 모두 같은 공간이다 — `snapshot.rows`(리프 행) 기준
인덱스로, 그룹 헤더·소계 행은 세지 않는다. `getDisplayedRowIndex`,
`getDisplayedRowAt`, `scrollToRow`, `setActiveCell`, `data-mg-row`
속성이 전부 이 공간을 공유하므로 서로 바로 넘겨 쓸 수 있다.
그룹화/트리 모드에서도 일관되며, 접힌 그룹·트리 안의 행은
`ensureRowVisible`이 조상을 펼친 뒤 이동한다.

## 컬럼 조회

```ts
grid.getColumn("age"); // ColumnDef | undefined — 숨김 컬럼도 조회 가능
grid.getColumnIndex("age"); // 표시 컬럼 인덱스 (visibleColumns 기준, 없거나 숨기면 -1)

grid.getColumns(); // 전체 컬럼 정의 — pinned 파티션 반영 렌더 순서
grid.getVisibleColumnDefs(); // 표시 컬럼만 (숨김 제외, DOM 순서)
grid.getColumnIds(); // field 목록 — 렌더 순서
grid.getHiddenColumnIds(); // 숨겨진 컬럼의 field 목록
```

## 행 탐색·셀 접근 단축

`forEachDisplayedRow`/`getCellValueAt`의 한 줄 버전 — id·predicate로
바로 찾고 읽는다.

```ts
grid.getAllRows(); // 원본 전체 (필터 무관, rawData 순서)
grid.getDisplayedRows(); // 표시 행 전체 (보이는 순서)
grid.getFirstRow(); // 표시 목록 첫 행
grid.getLastRow(); // 표시 목록 마지막 행
grid.getNextRow("42"); // id 행의 다음 표시 행
grid.getPrevRow("42"); // id 행의 이전 표시 행

grid.findRow((r) => r.name === "Bora"); // 조건 첫 매치
grid.findRows((r) => r.role === "admin"); // 조건 전체 매치
grid.findRowIndex((r) => r.age > 40); // 조건 첫 매치 인덱스 (-1)

// 셀 종합 조회 — { row, column, value, text, rowIndex, columnIndex }
const cell = grid.getCell("42", "name");
grid.getCellValueById("42", "name"); // 값만
grid.setCellValueById("42", "age", 40); // 쓰기 — 검증·Undo·afterEdit 동일
```

## 포커스 단축

`scrollToCell`/`scrollToRow` + `setActiveCell`을 한 번에.

```ts
grid.focusCell("42", "name"); // 스크롤 + 활성 셀 → boolean
grid.focusRow("42"); // 스크롤 + 첫 컬럼 활성 → boolean
```

## 삽입·변경분 단축

```ts
grid.insertRow(row); // 끝에 추가 (addRows와 동일)
grid.insertRow(row, "42"); // id=42 행 바로 앞에 삽입 → boolean

grid.getInsertedRows(); // I 상태 행
grid.getUpdatedRows(); // U 상태 행
grid.getDeletedRows(); // D 상태 행 (getChanges 분리 접근)
```

## 표시 행 집계 단축

`aggregationFn` 없이도 표시 행 기준 숫자 집계를 바로 얻는다.
컬럼의 값 해석(valueGetter 등)을 거쳐 숫자 변환 가능한 값만 모으고,
필터·검색·페이징이 적용된 표시 행만 계산한다.

```ts
grid.sumBy("age"); // 합계 — 숫자 없으면 null
grid.avgBy("age"); // 평균
grid.minBy("age"); // 최솟값
grid.maxBy("age"); // 최댓값
grid.countBy("age"); // 숫자 셀 수 (없으면 0)
```

## 정렬·필터·검색 모델 일괄 get/set

외부 UI(커스텀 필터 패널, URL 쿼리, 상태 바)와 그리드 상태를 동기화할 때
쓰는 모델 API. `getState`/`applyState`가 전체 레이아웃까지 저장한다면,
모델 API는 정렬/필터 조건만 가볍게 주고받는다.

```ts
// 읽기 — 복사본 반환
const sort = grid.getSortModel(); // SortSpec[] — priority 오름차순
const filter = grid.getFilterModel(); // FilterMap — { field: ColumnFilter }
const q = grid.getSearchText(); // 전역 검색어

// 쓰기 — 기존 조건을 모두 교체한다
grid.setSortModel([
  { columnKey: "role", direction: "asc", priority: 1 },
  { columnKey: "age", direction: "desc" }, // priority 생략 시 배열 순서
]);
grid.setFilterModel({
  role: { operator: "set", value: "", values: ["admin"] },
});
grid.setFilterModel(null); // 전체 해제
grid.setSortModel(null); // 정렬 해제
```

- `setSortModel`/`setFilterModel`은 각각 `sortChange`/`filterChange`
  이벤트를 발행하고, 서버 사이드 모드에서는 캐시를 비워 새 조건으로
  재요청한다. `setFilterModel`은 첫 페이지로 이동한다.
- `setFilterModel`에 아직 없는 컬럼의 필드를 넣어도 된다 — 컬럼이 나중에
  추가되면 그대로 적용된다.

## 행 선택 일괄 지정

```ts
grid.setRowSelection(["1", "3", "5"]); // ID 목록 일괄 선택
grid.setRowSelection(null); // 전체 해제 (clearSelection과 동일)
```

`isRowSelectable`이 거부하는 행과 존재하지 않는 ID는 자동으로 제외된다.
내용이 같으면 `selectionChange`를 재발행하지 않는다.

## 강제 갱신 — refreshCells

행 데이터를 그리드 밖에서 제자리 수정했거나, `formatter`/`valueGetter`가
참조하는 외부 상태가 바뀌었을 때 파이프라인을 재계산하고 다시 그린다.

```ts
row.price = fetchPrice(row.id);
grid.refreshCells(); // IBSheet Refresh / AG refreshCells 대응
```

데이터를 통째로 교체할 때는 `setData`가 더 적합하다(Undo 이력·행 상태 초기화).

## 수정분 수집 — getModifiedRows

`getChanges()`는 `{inserted, updated, deleted}` 세 배열로 분리해 반환한다.
저장 페이로드를 순회할 때는 flat 형태가 편하다.

```ts
for (const { row, status } of grid.getModifiedRows()) {
  // status: "I" | "U" | "D"
  await api.save(status, row);
}
```

## 스크롤 탐색

```ts
grid.ensureRowVisible("row-42"); // 행 ID 기준 — 없으면 false
grid.scrollToTop(); // 첫 표시 행
grid.scrollToBottom(); // 마지막 표시 행
```

`ensureRowVisible`은 ID로 리프 인덱스를 해석해 `scrollToRow`로 위임한다 —
정렬/필터/그룹화가 바뀌어도 ID 기준으로 안전하게 이동한다. 행이 접힌
그룹이나 트리 노드 안에 숨어 있으면 조상을 자동으로 펼친 뒤 이동한다
(그룹화는 행 값으로 조상 그룹 키를 재구성, 트리는 부모 체인을 따라감).

## 로딩 오버레이 — setLoading

코어 상태로 격상된 로딩 플래그. `snapshot.loading`이 true면 모든 어댑터가
그리드 위에 반투명 오버레이를 렌더링한다.

```ts
grid.setLoading(true); // IBSheet SetWaitImageVisible 대응
await fetchData();
grid.setLoading(false);
grid.isLoading(); // 현재 상태 조회
```

- `GridOptions.loading: true`로 초기 상태 지정 가능. 컴포넌트 props의
  `loading`과 OR로 합산되므로 선언형/명령형을 섞어 써도 된다.
- 서버 사이드 블록 로딩(`snapshot.serverSide.loading`)과는 별개의 수동 제어다.
- `mountGrid`에서는 `mounted.setLoading(on)`이 같은 코어 경로로 위임된다.

## IBSheet / AG-Grid 매핑 표

| IBSheet                    | AG-Grid                                         | moda-grid                                                                 |
| -------------------------- | ----------------------------------------------- | ------------------------------------------------------------------------- |
| `RowCount` / `TotalRows`   | `getDisplayedRowCount()`                        | `getDisplayedRowCount()` / `getFilteredRowCount()` / `getTotalRowCount()` |
| `GetRowData(row)`          | `getDisplayedRowAtIndex(i)`                     | `getDisplayedRowAt(i)` / `getRowById(id)`                                 |
| `GetRowIndexBy...`         | `getRowNode(id).rowIndex`                       | `getDisplayedRowIndex(id)`                                                |
| 노드 순회                  | `forEachNode` / `forEachNodeAfterFilterAndSort` | `forEachRow` / `forEachDisplayedRow`                                      |
| —                          | `columnApi.getColumn(key)`                      | `getColumn(field)` / `getColumnIndex(field)`                              |
| `GetSortCol`/`SetSortCol`  | `columnApi` 정렬 상태                           | `getSortModel()` / `setSortModel(specs)`                                  |
| —                          | `getFilterModel()` / `setFilterModel()`         | `getFilterModel()` / `setFilterModel(map)`                                |
| `SearchStr`                | `setGridOption('quickFilterText')`              | `setSearch(text)` / `getSearchText()`                                     |
| `SetSelectRow` / 전체 선택 | `setNodesSelected` / `selectAll`                | `setRowSelection(ids)` / `selectAllRows()`                                |
| `Refresh`                  | `refreshCells()` / `redrawRows()`               | `refreshCells()`                                                          |
| `GetSaveData`/`GetJson`    | 노드 순회 + `isRowSelected`                     | `getModifiedRows()` → `{row, status}[]`                                   |
| `MoveToRow`/`ShowRow`      | `ensureIndexVisible`/`ensureNodeVisible`        | `scrollToRow(i)` / `ensureRowVisible(id)`                                 |
| `SetFocus` + `ShowCell`    | `setFocusedCell` + `ensureNodeVisible`          | `focusCell(id, field)` / `focusRow(id)`                                   |
| 첫/마지막 행 이동          | `ensureIndexVisible(0/-1)`                      | `scrollToTop()` / `scrollToBottom()`                                      |
| `GetFirstRow`/`GetLastRow` | 첫/마지막 노드                                  | `getFirstRow()` / `getLastRow()`                                          |
| `GetNextRow`/`GetPrevRow`  | 이웃 노드                                       | `getNextRow(id)` / `getPrevRow(id)`                                       |
| `GetDataRows`/전체 행      | `forEachNode*`                                  | `getAllRows()` / `getDisplayedRows()`                                     |
| 조건부 행 검색             | `forEachNode` + 수동                            | `findRow` / `findRows` / `findRowIndex`                                   |
| `GetCellValue(Row, Col)`   | `getValue(colKey, rowNode)`                     | `getCell(id, field)` / `getCellValueById(id, field)`                      |
| `SetCellValue(Row, Col)`   | `setDataValue`                                  | `setCellValueById(id, field, v)`                                          |
| `GetTotalByCol`            | `forEachNode` + 수동                            | `sumBy` / `avgBy` / `minBy` / `maxBy` / `countBy`                         |
| 상태별 행 목록             | `forEachNode` + `is*`                           | `getInsertedRows` / `getUpdatedRows` / `getDeletedRows`                   |
| `SetWaitImageVisible`      | `setGridOption('loading')`                      | `setLoading(bool)` / `GridOptions.loading`                                |

## 어댑터별 접근

모든 메서드는 `GridCore` 인스턴스에 있다 — 플랫폼에 따라 인스턴스를 얻는
방법만 다르다.

```tsx
// React
const { grid, snapshot } = useGridCore({ columns, data });
grid.setLoading(true);

// Vue 3 — onReady prop 또는 template ref (defineExpose로 grid 노출)
<DataGrid :on-ready="(g) => (grid = g)" />

// Svelte 5 — bind:this → 컴포넌트 export
// vanilla
const m = mountGrid(el, { columns, data });
m.grid.forEachDisplayedRow((row) => console.log(row));
```

`snapshot.loading`은 어댑터가 자동으로 읽는다 — `setLoading`을 호출하면
어떤 프레임워크에서도 오버레이가 표시된다.
