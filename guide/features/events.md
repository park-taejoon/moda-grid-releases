# 이벤트 / 행 번호 / 채우기 핸들 / 헤더 메뉴

## 타입 이벤트 (`grid.on`)

IBSheet의 onClick/onDblClick/onEdit 계열에 해당하는 타입 안전 이벤트:

```ts
grid.on("cellClick", (e) => {
  // { row, rowIndex, column, columnIndex, value }
});
grid.on("cellDblClick", (e) => { /* 편집 진입 직전 */ });
grid.on("afterEdit", (e) => {
  // { row, rowIndex, column, oldValue, newValue } — 값이 실제로 바뀐 경우만
});
grid.on("selectionChange", (e) => {
  // { selectedIds: string[] }
});
grid.on("sortChange", (e) => {
  // { sortState: SortSpec[] } — 다중 정렬 조건 전체
});
grid.on("filterChange", (e) => {
  // { filters: FilterMap } — 컬럼 필터·전역 검색 변경 모두
});
grid.on("rowReorder", (e) => { /* { fromIndex, toIndex, row } */ });
grid.on("rowAdd", (e) => {
  // { rows, index } — addRows/duplicateRows 후 (IBSheet OnRowAdd 대응)
});
grid.on("rowDelete", (e) => {
  // { ids } — deleteRowsByIds로 D 마킹/제거된 행 (IBSheet OnRowDelete 대응)
});
grid.on("dataLoad", (e) => {
  // { rows } — setData로 원본 데이터 교체 후 (IBSheet OnDataLoad 대응)
});
grid.on("cellEditStart", (e) => {
  // { row, rowIndex, column, columnIndex } — 편집 진입 성공 직후
  // (beforeEdit를 통과하고 editingCell이 설정된 시점)
});
```

- 어댑터의 셀 클릭/더블클릭이 `cellClick`/`cellDblClick`을 발행한다.
- `afterEdit`은 `commitEditing`·체크박스 토글에서 실제 값 변경 시에만 발행.
- 반환값은 구독 해제 함수: `const off = grid.on("cellClick", cb); off();`

## 상태 옵저버 (`grid.watch`)

`on`이 개별 액션 이벤트라면, `watch`는 **스냅샷의 특정 값이 바뀔 때만**
호출되는 옵저버다 — 셀렉터 기반으로 관심 있는 슬라이스만 감시한다:

```ts
// 선택된 행이 바뀔 때만 실행 (정렬·스크롤 등 다른 변경엔 무반응)
const un = grid.watch(
  (s) => s.selectedRowIds,
  (ids, prev) => console.log("선택", ids.size, "이전", prev.size),
);

// 행 상태 맵 — U/I/D가 달라질 때
grid.watch((s) => s.rowStates, (states) => markDirty(states));

// 활성 셀 위치
grid.watch((s) => s.activeCell, (cell) => highlight(cell));

un(); // 구독 해제
```

- **얕은 비교**: 원시값·객체는 `Object.is`, 배열·Set·Map은 요소 단위 비교.
  내용이 같으면 스냅샷이 새로 생성돼도 호출되지 않는다.
- `{ immediate: true }` — 등록 즉시 현재 값으로 한 번 호출.
- `watch`는 `subscribe` 위의 얇은 래퍼라 렌더러 무관하게 코어에 동작한다.

```ts
grid.watch((s) => s.editingCell, (c) => log(c), { immediate: true });
```

## 편집 여부 판정 규칙 (U 마킹)

행 상태(`U`)와 `afterEdit`는 **저장 값이 실제로 바뀐 경우에만** 기록된다:

- 셀 클릭·더블클릭·드래그 선택만으로는 U가 되지 않는다.
- 에디터를 열고 값을 바꾸지 않은 채 커밋(Enter/blur/다른 셀 클릭)해도 무시된다.
- select/multiselect에서 원래 값과 같은 옵션을 고르거나, 빈 선택을
  커밋해도 기록되지 않는다.
- `valueSetter`를 쓰는 컬럼은 setter 실행 후 저장 값을 다시 비교한다.

### 선언형 구독 — `events` prop (권장)

`grid.on` 대신 선언형 맵으로 넘길 수 있다 — IBSheet의 `options.Events`에 해당:

```tsx
// 어댑터 prop — 마운트 시 구독, 언마운트 시 자동 해제
<DataGrid columns={cols} data={rows}
  events={{ afterEdit: (e) => save(e.row.id, e.newValue) }} />

// 코어 옵션 — 생성 시 구독
new GridCore({ columns, data, events: { sortChange: (e) => ... } });
```

### 인스턴스 접근 — `onReady`

props 모드(내부 GridCore)에서도 인스턴스를 받아 API를 호출할 수 있다:

```tsx
<DataGrid columns={cols} data={rows} onReady={(grid) => (myGrid = grid)} />
```

Vue 3는 `<DataGrid ref="g" />` 템플릿 ref의 `g.grid`로도 접근 가능
(`defineExpose`). Vue 2도 동일하게 expose된다.

## 행 번호 컬럼 (`rowNumbers`)

IBSheet의 행번호(Seq) 컬럼 — 맨 왼쪽에 표시 순번을 렌더링한다:

```tsx
<DataGrid columns={cols} data={data} rowNumbers />   // React
<DataGrid :row-numbers="true" />                      // Vue
```

- 표시 순서 기준 1부터 시작 — 정렬/필터 후 재번호.
- 그룹/소계/pinned/합계 행에는 빈 셀로 정렬 유지.
- `.mg-rownum-cell` 클래스로 스타일 커스터마이즈 가능.

## 채우기 핸들 (Fill handle)

활성 셀 우하단의 작은 사각형(`.mg-fill-handle`)을 드래그하면 소스 범위의
값을 드래그 방향으로 **타일링 복사**한다 — 엑셀 채우기 핸들 방식:

- 소스 범위가 여러 행/열이면 패턴이 반복된다.
- `editable: false`·수식 컬럼은 채우기 대상에서 제외.
- 채운 셀은 편집 이력·행 상태(U)·Undo에 정상 기록된다.
- 드래그 완료 후 선택 범위가 채운 영역 전체로 확장된다.

```ts
// 코어 직접 호출도 가능
const filled = grid.fillRange(
  { startRow: 0, startCol: 0, endRow: 0, endCol: 0 },  // 소스
  { startRow: 0, startCol: 0, endRow: 5, endCol: 0 },  // 대상(소스 포함)
);
```

키보드 단축키도 지원한다 — 선택 범위에서 `Ctrl/Cmd+D`는 첫 행을 아래로,
`Ctrl/Cmd+R`은 첫 컬럼을 오른쪽으로 채운다 (`fillDown`/`fillRight`,
엑셀과 동일).

### 채우기 시리즈 (`fillSeries`)

`fillSeries: true` 옵션을 켜면 엑셀 자동 채우기처럼 소스 라인의 패턴을
외삽한다 — 한 축으로만 채울 때 적용된다:

- 소스 값이 **모두 숫자** → 등차 수열 (`10,20` → `30,40,…`; 단일 값은 +1씩)
- 소스 값이 **모두 Date** → 같은 간격의 날짜 수열 (단일 값은 +1일)
- 그 외 값 → 일반 타일링 복사로 되돌아간다

`fillRange(source, target, { series })` 인자로 호출 단위로도 제어할 수
있고, `setFillSeries(bool)`로 실행 중 토글할 수 있다. Ctrl+D/R과
Ctrl+Enter(`fillActiveToSelection`)는 엑셀과 마찬가지로 항상 복사다.

## 내용 지우기 (Delete/Backspace)

그리드에 포커스된 상태에서 `Delete`/`Backspace`를 누르면 **선택 범위
(없으면 활성 셀)의 내용을 지운다** — 엑셀/IBSheet와 동일한 동작.

- `editable: false`·수식 컬럼은 건너뛴다.
- `checkbox` 컬럼은 `uncheckedValue`로, 나머지는 `null`로 지운다.
- 지우기는 검증을 거치지 않는다 — `required` 컬럼을 지우면 이후
  `validateChanges()`에서 검출된다.
- 변경은 Undo 1단위·행 상태(U)에 기록된다.

```ts
// 코어 직접 호출 — 반환값은 실제로 지워진 셀 수
grid.clearRange();                                   // 선택 범위/활성 셀
grid.clearRange({ startRow: 0, startCol: 0, endRow: 2, endCol: 3 });
```

## 범위 이동 (`moveRange`)

셀 범위의 값을 다른 위치로 **이동**한다 — 소스를 읽어 타겟 앵커에 같은
크기로 쓰고 소스를 지운다 (엑셀 범위 드래그 이동):

```ts
grid.moveRange(
  { startRow: 0, startCol: 0, endRow: 0, endCol: 1 },  // 소스 A0:B0
  { rowIndex: 3, columnIndex: 0 },                     // 타겟 앵커 A3
);
```

- 편집 불가/수식 컬럼·검증 실패 셀은 건너뛴다.
- 소스와 타겟이 겹치면 타겟에 실제로 쓰인 셀은 지우지 않는다.
- 이동+지우기 전체가 Undo 1단위다.

## 헤더 컨텍스트 메뉴 (`headerContextMenu`)

셀 메뉴(`contextMenu`)와 별도로 **헤더 우클릭 메뉴**를 정의한다.
`ctx.row`는 `null`, `ctx.rowIndex`는 `-1`이다:

```ts
new GridCore({
  columns, data,
  headerContextMenu: [
    { id: "hide", label: (ctx) => `${ctx.column.header} 숨기기`,
      onClick: (ctx) => grid.setColumnVisible(ctx.column.field, false) },
    { id: "sort-asc", label: "오름차순",
      onClick: (ctx) => grid.setSort(ctx.column.field, "asc") },
    { id: "autofit", label: "너비 자동",
      onClick: (ctx) => grid.autoSizeColumn(ctx.column.field) },
  ],
});
```

어댑터는 `<DataGrid headerContextMenu={items} />`로 전달한다.
메뉴 팝업은 셀 메뉴와 같은 `snapshot.contextMenu` 경로로 렌더링된다.

## 행으로 스크롤 — `scrollToRow`

표시 인덱스 기준으로 해당 행이 보이도록 스크롤한다 (IBSheet `showRow` 대응):

```ts
grid.scrollToRow(150);              // 150번째 표시 행으로 이동
grid.scrollToRow(match.rowIndex);   // findNext/findCells 결과와 조합
```

- **가상 스크롤**: 코어가 `scrollTop`을 직접 보정 — 어댑터의 스크롤
  동기화가 DOM에 반영한다. 이미 보이는 행은 이동하지 않는다.
- **비가상 스크롤**: `snapshot.scrollRequest`(`{rowIndex, seq}`)를 발행 —
  어댑터가 `td[data-mg-row]`로 `scrollIntoView({block:"nearest"})`한다.
- 범위 밖 인덱스는 무시한다.
