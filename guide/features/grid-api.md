# 그리드 편의 API (Grid Convenience API)

상용 엔터프라이즈 그리드에서 매일 쓰는 조회·모델·스크롤
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

### 무음 쓰기 (`silent`)

프로그램이 값을 되돌리거나 동기화할 때 이벤트 루프를 피하고 싶다면
`{ silent: true }`를 넘긴다 — `afterEdit`/`dataChange` 이벤트, Undo
이력, cellFlash 없이 값과 행 상태(U/pristine 추적)만 갱신된다:

```ts
grid.setCellValueById("42", "age", 40, { silent: true });
grid.setCellValue(0, 1, "x", { silent: true });
grid.updateRow("42", { age: 40, name: "x" }, { silent: true });

// 범위·체크박스 쓰기도 같은 옵션을 받는다
grid.setRangeValues(range, values, { silent: true });
grid.clearRange(range, { silent: true });
grid.moveRange(src, target, { silent: true });
grid.fillRange(src, dst, { series: true, silent: true });
grid.setCellChecked(row, col, true, { silent: true });
grid.setRowChecked("42", true, undefined, { silent: true });
grid.setCheckedRowIds(["1", "3"], true, undefined, { silent: true });
grid.toggleAllChecked(col, { silent: true });
```

유저 편집과 프로그램 쓰기를 리스너에서 구분하려면 silent 대신
`afterEdit`/`dataChange` 이벤트의 `source` 필드를 본다 — 유저 편집은
`"edit"`, API 쓰기는 `"api"`다 (`events.md` 참조).

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

## 행 상태·선택·체크 단축

```ts
// 행 상태 수동 지정 — I/U/D 마킹 또는 null로 해제
grid.setRowStatus("42", "U"); // 서버 저장 전 상태 강제 조정 → boolean
grid.setRowStatus("42", null); // 상태 해제

// 선택 행 ID / 일괄 삭제 — getSelectedRowData의 ID 버전
grid.getSelectedRowIds(); // ["1","4",...] (rawData 순서)
grid.deleteSelectedRows(); // 선택 행을 일괄 D 마킹 → 처리 행 수

// checkbox 컬럼 ID 일괄 조작 — setRowChecked의 복수 버전
grid.getCheckedRowIds(); // 체크된 행 ID 목록
grid.setCheckedRowIds(["1", "3"]); // 일괄 체크 → 적용 수
grid.setCheckedRowIds(["1"], false); // 일괄 해제
```

## 편집·트리·컬럼 레이아웃 단축

```ts
// 행 ID + 필드로 바로 편집 진입 — focusCell + startEditing 합성
grid.startEditingById("42", "name"); // → boolean

// 트리 노드 명시적 펼침/접힘 — toggleTreeExpanded의 결정적 버전
grid.isTreeExpanded("1"); // 현재 펼침 여부
grid.setTreeExpanded("1", false); // 접기 (같은 상태면 무시)

// 컬럼 순서 일괄 지정 — 나열 컬럼이 순서대로 앞에 배치,
// 나머지는 기존 상대 순서 유지
grid.setColumnOrder(["role"]); // role 컬럼을 맨 앞으로

// 컬럼 너비 저장/복원 — getState 없이 너비만 주고받을 때
const widths = grid.getColumnWidths(); // { [field]: px } 실효 너비
grid.setColumnWidths({ name: 240 }); // 일괄 지정 → 적용된 컬럼 수
```

## 페이지·표시 여부·변경 되돌리기

```ts
// 페이지 상태 조회 — 페이저 UI를 직접 그릴 때
grid.getPageIndex(); // 현재 페이지 (0-base)
grid.getPageSize(); // 0이면 페이징 해제
grid.getPageCount(); // 전체 페이지 수
grid.setPageSize(50); // 크기만 변경 — 현재 페이지는 범위 내에서 유지

// 트리 깊이·일괄 펼침 — setTreeExpandLevel의 네이밍 단축
grid.getRowDepth("8"); // 루트=0, 비트리/미존재는 -1
grid.expandAllTree(); // 전체 펼치기
grid.collapseAllTree(); // 전체 접기

// 표시 여부 — 필터·페이징·숨김·트리/그룹 접힘까지 반영
grid.isRowDisplayed("8"); // 화면 표시 목록에 있으면 true

// 변경 되돌리기 — commitChanges 없이 폐기
// I 행은 제거, D 행은 복원, U 행은 편집 전 값으로 복귀 → 처리 행 수
grid.discardRowChanges(grid.getSelectedRowIds());
```

## 활성/편집 셀·선택·순서·고정/그룹 탐색

```ts
// 활성/편집 셀 조회 — 리프 인덱스 + 컬럼 field + 행 ID를 한 번에 반환
grid.getActiveCell(); // {rowIndex, columnIndex, columnKey, rowId} | null
grid.getEditingCell(); // 편집 중이 아니면 null

// 결정적 선택 — toggleRowSelection의 명시적 버전
grid.setRowSelected("42", true); // isRowSelectable 거부·미존재 행은 false

// 표시 행 ID 목록 / 순서 이동 — ID 기반 API의 입력으로 바로 사용
grid.getDisplayedRowIds(); // ["1","4",...] (표시 순서)
grid.moveRowById("42", 0); // 표시 목록 맨 앞으로 → boolean

// 컬럼 표시/고정 상태 — setColumnVisible/setColumnPinned의 조회 버전
grid.isColumnVisible("age"); // 숨김이면 false (없는 field도 false)
grid.getPinnedColumnIds(); // ["id","progress"] — 표시 순서
grid.getPinnedColumnIds("left"); // 한쪽만
grid.getColumnPinned("id"); // "left" | "right" | null

// 그룹 행 토글 — displayRows의 GroupNode.key를 그대로 사용
const g = grid.getSnapshot().displayRows?.find((d) => d.type === "group");
if (g?.type === "group") {
  grid.isGroupExpanded(g.key); // 현재 펼침 여부
  grid.setGroupExpanded(g.key, false); // 결정적 접기
}

// 트리 부모/자식 ID — getRowDepth와 함께 트리 탐색에 사용
grid.getParentRowId("8"); // 부모 ID 또는 null
grid.getChildRowIds("1"); // 직계 자식 ID 목록
```

## 고정 행·셀 오버라이드·모델 조회

```ts
// 행 ID 기준 상/하단 고정 — 한 행은 한쪽에만 고정 (반대쪽은 자동 해제)
grid.pinRow("42", "top"); // → boolean (없는 ID는 false)
grid.unpinRow("42"); // 어느 쪽이든 해제
grid.getPinnedRowIds("top"); // 고정 행 ID 목록
grid.getPinnedRows("top"); // 행 객체 복사본 — 이펙트에서 요약 행과 병합용

// 수동 셀 에러 — 서버 저장 결과 등 외부 검증 에러의 셀 표시.
// 컬럼 validate보다 우선, null로 해제하면 원래 검증으로 복귀
grid.setCellError("42", "name", "중복된 이름입니다");
grid.setCellError("42", "name", null);

// 수동 셀 노트 — ColumnDef.note의 행 단위 오버라이드
grid.setCellNote("42", "name", "확인 필요");

// 활성/편집 셀의 행 객체 — getActiveCell/getEditingCell의 행 버전
grid.getActiveRow(); // TData | null
grid.getEditingRow(); // 비편집 시 null

// 단일 컬럼 모델 조회 — setFilter/setColumnWidth의 읽기 버전
grid.getColumnFilter("role"); // ColumnFilter | null (복사본)
grid.getColumnWidth("name"); // 실효 px 또는 null

// 현재 그룹핑 컬럼 목록 — setGroupBy의 읽기 버전
grid.getGroupBy(); // ["role"] (비그룹이면 [])

// 실제 DOM 렌더 행 범위 — 가상 스크롤이면 뷰포트 슬라이스
grid.getViewportRowRange(); // {start, end} | null (end 미포함)
```

## 위치·정렬·변경분·원본 값 조회

```ts
// 원본 데이터(setData 순서) 기준 위치 — 표시 순서와 별개
grid.getRowByIndex(0); // 첫 원본 행
grid.indexOfRow("42"); // 원본 내 인덱스, 없으면 -1
// 표시 컬럼 순서 ↔ 필드 상호 변환 (재배치·숨김 반영)
grid.getColumnIndex("name"); // 1
grid.getFieldAt(1); // "name"

// 현재 정렬 상태 — 커스텀 정렬 표시기용
grid.getSortState(); // SortSpec[] 복사본 (우선순위순)
grid.getSortDirection("name"); // "asc" | "desc" | null

// 행 상태 조건자 — getRowState의 단축
grid.isRowAdded("42"); // I
grid.isRowModified("42"); // U
grid.isRowDeleted("42"); // D

// 변경 필드·원본 값 — "무엇이 바뀌었나" 검토용
// 원본으로 되돌린 필드는 목록에서 제외된다 (현재 값과 실시간 비교)
grid.getChangedFields("42"); // ["name", "age"]
grid.getOriginalValue("42", "name"); // 최초 로드 시점 값
grid.getOriginalRow("42"); // 원본 값이 적용된 행 사본

// 개별 행 높이 오버라이드 맵 — 레이아웃 저장용
grid.getRowHeights(); // {"42": 48}
```

## 숨김 증분·일괄 삽입·범위 선택·셀 플래시

```ts
// 숨김 증분 제어 — setHiddenRows의 합집합/차집합 버전
grid.hideRowsByIds(["7", "9"]); // 새로 숨긴 행 수 반환
grid.showRowsByIds(["7"]); // 다시 표시된 행 수 반환
grid.getHiddenRows(); // 숨겨진 행 객체 배열 (rawData 순서)

// 여러 행 일괄 삽입 — insertRow의 일괄 버전 (I 상태 마킹)
grid.insertRows(rows, "42"); // "42" 앞에 삽입 → 삽입 수 반환
grid.insertRows(rows); // beforeId 생략 시 끝에 추가

// 선택 상태 판정·범위 값 읽기 — 커스텀 렌더러/외부 UI용
grid.isCellSelected(0, 1); // 활성 셀이거나 선택 범위 안이면 true
grid.getRangeValues(); // 활성 범위의 2차원 값 배열, 범위 없으면 null
grid.getRangeValues({ startRow: 0, startCol: 1, endRow: 3, endCol: 2 });

// 행 ID + 컬럼 필드로 범위 선택 — 표시 좌표를 몰라도 된다
grid.selectRangeByIds("1", "name", "9", "role"); // 실패 시 false

// 수동 셀 플래시 — 데이터 변경 없이 잠깐 강조 (600ms 자동 해제)
grid.flashCells(["42"], ["age"]); // 실제로 플래시된 셀 수 반환
grid.flashRows(["42", "43"]); // 행 전체 — 모든 표시 컬럼
```

- `hideRowsByIds`는 존재하지 않는 ID를 무시하고, `showRowsByIds`는 숨겨져
  있지 않은 ID를 무시한다. 둘 다 실제로 바뀐 행 수를 반환한다.
- `selectRangeByIds`는 어느 좌표든 표시 목록(필터·페이징·숨김 반영)에
  없으면 `false`를 반환하고 기존 선택을 바꾸지 않는다.
- `flashCells`는 `cellFlash` prop과 무관하게 동작한다 — 자동 플래시를
  꺼둔 그리드에서도 수동 강조가 가능하다.

## 편집 값·범위 쓰기·행 이동·행 TSV·그룹/트리 탐색

```ts
// 편집 중 보류 값 — 커밋 전 임시 값을 읽는다
grid.getEditValues(); // cell 편집: {name: "임시"} / fullRow: 행 전체 맵 / 비편집: null
grid.isRowEditing("42"); // 해당 행이 지금 편집 중이면 true

// 선택 상태 검사 — 외부 툴바·보내기용
grid.getSelectedCells(); // [{rowIndex, columnIndex, rowId, field, value}, ...]

// 범위에 2차원 값 기록 — 붙여넣기와 같은 규칙 (editable·검증·valueSetter)
// 전체가 한 Undo 단위로 기록된다
grid.setRangeValues(
  { startRow: 0, startCol: 1, endRow: 1, endCol: 2 },
  [
    ["범위1", "r1@x.io"],
    ["범위2", "r2@x.io"],
  ],
); // 실제로 쓰인 셀 수 반환

// 행 목록을 TSV로 — 범위 선택이 아닌 "행" 단위 복사
grid.getRowsTsv(["1", "3"]); // 표시 컬럼 순서 TSV, 대상 없으면 null
grid.getRowsTsv(); // 생략 시 선택된 행
grid.getRowsTsv(["1"], { includeHeaders: true, formatted: true });

// 여러 행 일괄 이동 — moveRowById의 벡터 버전 (지정 순서 유지)
grid.moveRowsByIds(["1", "2"], 999); // 맨 뒤로 → 실제 이동 수 반환

// 그룹/트리 탐색 — 노드 키로 리프 행 조회
grid.getGroupRows("operator"); // displayRows의 GroupNode.key → 리프 행 배열
grid.getChildRows("8"); // getChildRowIds의 행 객체 버전
grid.addTreeChild("1", { id: 99, parentId: 1, ... }); // 트리 자식 삽입
```

- `getEditValues`는 cell 편집이면 해당 셀의 `{field: value}` 한 엔트리,
  `editType: "fullRow"`면 행 전체 `field → value` 맵을 반환한다
  (입력 중인 미커밋 값 포함). 반환값은 복사본이다.
- `setRangeValues`의 좌표는 `getRangeValues`와 같은 표시 행/표시 컬럼
  기준이다. `values`가 범위보다 짧으면 주어진 셀만 쓰고 길면 잘라낸다.
  읽기 전용·formula·검증 실패 셀은 건너뛰고, 값이 실제로 바뀐 셀만
  이력에 남는다.
- `getRowsTsv`의 구분자는 `clipboardDelimiter` 옵션을 따른다.
  `beforeCopy` 훅은 적용하지 않는다(훅이 필요하면 `getSelectionTsv`).
- `addTreeChild`는 두 트리 모드를 모두 지원한다. nested(childrenKey)
  모드에서는 부모의 children 배열에, flat(getParentId) 모드에서는
  부모 필드가 지정된 행을 부모 바로 뒤에 삽입한다. `parentId`가
  `null`이면 루트 끝에 추가되고, 삽입된 행은 I 상태로 마킹된다.

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
grid.refreshCells(); // 상용 그리드의 Refresh/refreshCells 대응
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
grid.setLoading(true); // 상용 그리드의 로딩 표시 대응
await fetchData();
grid.setLoading(false);
grid.isLoading(); // 현재 상태 조회
```

- `GridOptions.loading: true`로 초기 상태 지정 가능. 컴포넌트 props의
  `loading`과 OR로 합산되므로 선언형/명령형을 섞어 써도 된다.
- 서버 사이드 블록 로딩(`snapshot.serverSide.loading`)과는 별개의 수동 제어다.
- `mountGrid`에서는 `mounted.setLoading(on)`이 같은 코어 경로로 위임된다.

## 상용 그리드 매핑 표

| 레거시 그리드 (`sheet.*`)      | 대표적 그리드 라이브러리                           | moda-grid                                                                 |
| ------------------------------ | -------------------------------------------------- | ------------------------------------------------------------------------- |
| `RowCount` / `TotalRows`       | `getDisplayedRowCount()`                           | `getDisplayedRowCount()` / `getFilteredRowCount()` / `getTotalRowCount()` |
| `GetRowData(row)`              | `getDisplayedRowAtIndex(i)`                        | `getDisplayedRowAt(i)` / `getRowById(id)`                                 |
| `GetRowIndexBy...`             | `getRowNode(id).rowIndex`                          | `getDisplayedRowIndex(id)`                                                |
| 노드 순회                      | `forEachNode` / `forEachNodeAfterFilterAndSort`    | `forEachRow` / `forEachDisplayedRow`                                      |
| —                              | `columnApi.getColumn(key)`                         | `getColumn(field)` / `getColumnIndex(field)`                              |
| `GetSortCol`/`SetSortCol`      | `columnApi` 정렬 상태                              | `getSortModel()` / `setSortModel(specs)`                                  |
| —                              | `getFilterModel()` / `setFilterModel()`            | `getFilterModel()` / `setFilterModel(map)`                                |
| `SearchStr`                    | `setGridOption('quickFilterText')`                 | `setSearch(text)` / `getSearchText()`                                     |
| `SetSelectRow` / 전체 선택     | `setNodesSelected` / `selectAll`                   | `setRowSelection(ids)` / `selectAllRows()`                                |
| `Refresh`                      | `refreshCells()` / `redrawRows()`                  | `refreshCells()`                                                          |
| `GetSaveData`/`GetJson`        | 노드 순회 + `isRowSelected`                        | `getModifiedRows()` → `{row, status}[]`                                   |
| `MoveToRow`/`ShowRow`          | `ensureIndexVisible`/`ensureNodeVisible`           | `scrollToRow(i)` / `ensureRowVisible(id)`                                 |
| `SetFocus` + `ShowCell`        | `setFocusedCell` + `ensureNodeVisible`             | `focusCell(id, field)` / `focusRow(id)`                                   |
| 첫/마지막 행 이동              | `ensureIndexVisible(0/-1)`                         | `scrollToTop()` / `scrollToBottom()`                                      |
| `GetFirstRow`/`GetLastRow`     | 첫/마지막 노드                                     | `getFirstRow()` / `getLastRow()`                                          |
| `GetNextRow`/`GetPrevRow`      | 이웃 노드                                          | `getNextRow(id)` / `getPrevRow(id)`                                       |
| `GetDataRows`/전체 행          | `forEachNode*`                                     | `getAllRows()` / `getDisplayedRows()`                                     |
| 조건부 행 검색                 | `forEachNode` + 수동                               | `findRow` / `findRows` / `findRowIndex`                                   |
| `GetCellValue(Row, Col)`       | `getValue(colKey, rowNode)`                        | `getCell(id, field)` / `getCellValueById(id, field)`                      |
| `SetCellValue(Row, Col)`       | `setDataValue`                                     | `setCellValueById(id, field, v)`                                          |
| `GetTotalByCol`                | `forEachNode` + 수동                               | `sumBy` / `avgBy` / `minBy` / `maxBy` / `countBy`                         |
| 상태별 행 목록                 | `forEachNode` + `is*`                              | `getInsertedRows` / `getUpdatedRows` / `getDeletedRows`                   |
| `GetRowStatus`/`SetRowStatus`  | 노드 데이터 수동                                   | `getRowState(row)` / `setRowStatus(id, status)`                           |
| 선택 행 ID 목록                | `getSelectedRows().map(id)`                        | `getSelectedRowIds()`                                                     |
| `DeleteSelectedRows`           | `applyTransaction({remove})`                       | `deleteSelectedRows()`                                                    |
| `FindCheckedRow`               | 체크박스 노드 필터                                 | `getCheckedRowIds(field?)` / `setCheckedRowIds(ids, checked?)`            |
| 셀 편집 진입                   | `startEditingCell` + 포커스                        | `startEditingById(id, field)`                                             |
| `SetColOrder`/컬럼 이동        | `columnApi.moveColumns`                            | `setColumnOrder(fields)` / `reorderColumn(dragged, target)`               |
| `SetColWidth` 반복             | `columnApi.setColumnWidths`                        | `getColumnWidths()` / `setColumnWidths(map)`                              |
| `GetRowExpand`/`SetRowExpand`  | 트리 노드 `expanded`                               | `isTreeExpanded(id)` / `setTreeExpanded(id, bool)`                        |
| `GetCurrentPage`/`SetPageSize` | `paginationGetCurrentPage`/`paginationSetPageSize` | `getPageIndex()` / `setPageSize(n)` / `getPageCount()`                    |
| 트리 일괄 펼침/접기            | `expandAll`/`collapseAll` (rowModel)               | `expandAllTree()` / `collapseAllTree()`                                   |
| `GetRowDepth`                  | 노드 `level`                                       | `getRowDepth(id)`                                                         |
| `IsVisible`                    | 노드 `displayed`                                   | `isRowDisplayed(id)` — 필터·페이징·숨김·접힘 반영                         |
| `DiscardData`/`Undo`           | `applyTransaction` 역연산                          | `discardRowChanges(ids)` — I 제거·D 복원·U 원본 복귀                      |
| `GetFocusRow`/`GetFocusCol`    | `getFocusedCell()`                                 | `getActiveCell()` — 인덱스+컬럼키+행 ID 한 번에                           |
| `GetEditRow`                   | `getEditingCells()`                                | `getEditingCell()` — 비편집 시 null                                       |
| `SetRowSelected`               | `setSelected(bool)`/`selectAll`                    | `setRowSelected(id, bool)` — 결정적 지정                                  |
| 행 순서 이동                   | `moveRowNode`                                      | `moveRowById(id, toIndex)` — 표시 목록 기준                               |
| `GetColHidden`                 | `columnApi.getColumnState`                         | `isColumnVisible(field)`                                                  |
| 고정 컬럼 조회                 | `getColumnState` `pinned`                          | `getPinnedColumnIds(side?)` / `getColumnPinned(field)`                    |
| `SetGroupRowExpanded`          | `setExpanded(bool)` (그룹 노드)                    | `setGroupExpanded(key, bool)` / `isGroupExpanded(key)`                    |
| 트리 부모/자식                 | 노드 `parent`/`childrenAfterGroup`                 | `getParentRowId(id)` / `getChildRowIds(id)`                               |
| 행 고정                        | `setPinnedTopRowData`/`setPinnedBottomRowData`     | `pinRow(id, side)` / `unpinRow(id)` / `getPinnedRowIds` / `getPinnedRows` |
| 셀 에러 수동 표시              | `setCellValue` + `valid` 플래그                    | `setCellError(id, field, msg)` — 서버 검증 결과 표시                      |
| 셀 노트 오버라이드             | —                                                  | `setCellNote(id, field, note)`                                            |
| `GetFocusRow` 데이터           | `getFocusedCell`의 노드                            | `getActiveRow()` / `getEditingRow()`                                      |
| `GetTopRow`/렌더 범위          | `getFirst/LastDisplayedRowIndex`                   | `getViewportRowRange()` → `{start, end}`                                  |
| 그룹 컬럼 조회                 | `getRowGroupColumns`                               | `getGroupBy()`                                                            |
| 단일 컬럼 필터/너비            | `getFilterModel`/`getActualWidth`                  | `getColumnFilter(field)` / `getColumnWidth(field)`                        |
| 원본 행 위치                   | `getRowIndex` (데이터)                             | `indexOfRow(id)` / `getRowByIndex(i)`                                     |
| 컬럼 위치 ↔ 필드               | `getColumn` / `colId`                              | `getColumnIndex(field)` / `getFieldAt(i)`                                 |
| 정렬 상태 조회                 | `getSortModel`/`getSortState`                      | `getSortState()` / `getSortDirection(field)`                              |
| 행 상태 조건자                 | 노드 `rowPinned`/`data` 비교                       | `isRowAdded` / `isRowModified` / `isRowDeleted(id)`                       |
| 변경 필드·원본 값              | —                                                  | `getChangedFields(id)` / `getOriginalValue` / `getOriginalRow`            |
| 개별 행 높이 조회              | 노드 `rowHeight`                                   | `getRowHeights()` → `{id: px}`                                            |
| 행 숨기기/표시 증분            | —                                                  | `hideRowsByIds(ids)` / `showRowsByIds(ids)` / `getHiddenRows()`           |
| `DataInsert` 일괄              | `applyTransaction({add, addIndex})`                | `insertRows(rows, beforeId?)`                                             |
| 선택 셀 판정·범위 값           | `getCellRangeSelections`                           | `isCellSelected(r,c)` / `getRangeValues(range?)`                          |
| `SelectCell` 범위              | `setCellSelection` / `addCellRange`                | `selectRangeByIds(startId, field, endId?, endField?)`                     |
| —                              | `flashCells`                                       | `flashCells(ids, fields?)` / `flashRows(ids)` — 600ms 자동 해제           |
| `GetEditValues`                | —                                                  | `getEditValues()` / `isRowEditing(id)` — 커밋 전 임시 값·편집 여부        |
| 범위 값 쓰기                   | `getCellRangeSelections` + `setCellValue`          | `setRangeValues(range, values)` — 한 Undo 단위                            |
| 선택 셀 목록                   | `getCellRangeSelections`                           | `getSelectedCells()` → `{rowIndex, columnIndex, rowId, field, value}[]`   |
| 행 목록 TSV                    | `getDataAsClipboard`류                             | `getRowsTsv(ids?, {includeHeaders?, formatted?})`                         |
| 행 일괄 이동                   | `applyTransaction` + 행 인덱스 재배치              | `moveRowsByIds(ids, toIndex)` — 지정 순서 유지                            |
| 그룹 리프 행 조회              | 노드 `childrenAfterGroup` 순회                     | `getGroupRows(groupKey)` — GroupNode.key로 리프 행 배열                   |
| `ChildAdd`                     | 노드 `children` 수동 삽입                          | `getChildRows(id)` / `addTreeChild(parentId, row, index?)`                |
| `SetWaitImageVisible`          | `setGridOption('loading')`                         | `setLoading(bool)` / `GridOptions.loading`                                |

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
