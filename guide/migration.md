# 타사 엔터프라이즈 그리드 → moda-grid 마이그레이션

명령형 인스턴스 기반 엔터프라이즈 그리드에 익숙한 개발자를 위한
개념 매핑과 치트시트.

## TL;DR — 5줄 비교

```js
// 기존 엔터프라이즈 그리드: 인스턴스가 DOM에 직접 렌더링
const sheet = GridLib.create({
  el: "sheetDiv",
  options: { Cfg: {...}, Cols: [...], Events: { onClick: fn } },
  data: [...],
});
```

```tsx
// moda-grid: 프레임워크 컴포넌트 — 데이터는 props로 반응형 바인딩
<DataGrid
  columns={cols}
  data={rows}
  events={{ cellClick: fn }}
  onReady={(grid) => (gridRef = grid)}
/>
```

- `create()` → `<DataGrid>` 컴포넌트 (React/Vue3/Vue2/Svelte)
- `options.Events` → `events` prop (또는 `grid.on()`)
- 반환 인스턴스 → `onReady` 콜백 / React `useGridCore` / Vue `ref` + `expose`

### CDN/script 태그 환경 (레거시 그리드와 동일 패턴)

레거시 그리드처럼 빌드 도구 없이 `<script>` 태그만으로 쓰려면 `mountGrid`:

```html
<link rel="stylesheet" href="https://grid.modaolive.com/style.css" />
<script src="https://grid.modaolive.com/moda-grid.js"></script>
<div id="grid"></div>
<script>
  const m = ModaGrid.mountGrid(document.getElementById("grid"), {
    columns: [{ field: "name", header: "이름", filterable: true }],
    data: rows,
    events: { afterEdit: (e) => save(e) }, // 레거시 Events 옵션 대응
  });
  m.grid.exportToCsv(); // create() 반환 인스턴스와 동일한 위치
</script>
```

## 초기화 매핑

| 기존 엔터프라이즈 그리드                              | moda-grid                                         |
| ----------------------------------------------------- | ------------------------------------------------- |
| `GridLib.create({el, options, data})` (레거시 초기화) | `<DataGrid columns={cols} data={rows} />`         |
| `options.Cols: [{Header, Type, Name}]`                | `columns: [{ field, header, cellEditor, width }]` |
| `options.Cfg` (SearchMode, Page…)                     | `DataGrid` props 또는 `useGridCore` 옵션          |
| `options.Events`                                      | `events` prop → `{ cellClick, afterEdit, ... }`   |
| 반환 `sheet` 인스턴스                                 | `onReady={(grid) => ...}` 또는 `useGridCore`      |

### 인스턴스가 필요할 때 (props 모드)

```tsx
// React — onReady 또는 훅
<DataGrid columns={cols} data={rows} onReady={(g) => (gridRef.current = g)} />
const { grid, snapshot } = useGridCore({ columns, data });  // 제어 모드

<!-- Vue 3 — onReady prop 또는 템플릿 ref -->
<DataGrid :columns="cols" :data="rows" :on-ready="(g) => grid = g" />
<DataGrid ref="gridRef" />   // gridRef.grid 로 접근 (defineExpose)
```

## 이벤트 매핑

| 레거시 `Events` 옵션            | moda-grid `events` prop / `grid.on`                                                                          |
| ------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| `onClick`                       | `cellClick` — `{ row, rowIndex, column, columnIndex, value }`                                                |
| `onDblClick`                    | `cellDblClick`                                                                                               |
| `onEdit`                        | `afterEdit` — `{ row, column, oldValue, newValue }`                                                          |
| `onSelectRow`                   | `selectionChange` — `{ selectedIds }`                                                                        |
| 드래그 범위 선택 (MouseSelect)  | `selectionMode: "multi-cell"` — 셀 드래그/Shift+키로 범위 선택                                               |
| `onSort`                        | `sortChange` — `{ sortState }`                                                                               |
| `onFilter`                      | `filterChange` — `{ filters }`                                                                               |
| `onRowMove`                     | `rowReorder` — `{ fromIndex, toIndex, row }`                                                                 |
| `onRowAdd`                      | `rowAdd` — `{ rows, index }` (addRows/duplicateRows)                                                         |
| `onRowDelete`                   | `rowDelete` — `{ ids }` (deleteRowsByIds — D 마킹 포함)                                                      |
| `onDataLoad`                    | `dataLoad` — `{ rows }` (setData 교체 후)                                                                    |
| `onEditStart` (진입 시점)       | `cellEditStart` — `{ row, rowIndex, column, columnIndex }`                                                   |
| `onFocus` / `onFocusCell`       | `activeCellChange` — `{ row, rowIndex, column, columnIndex, previous }`                                      |
| `onPaste`                       | `dataPaste` — `{ applied, skipped, errors, start }`                                                          |
| `onChange` (모든 데이터 변경)   | `dataChange` — `{ source, edits? }` — 편집/붙여넣기/행 추가·삭제/undo·redo/setData 통합                      |
| `onColResize`                   | `columnResize` — `{ field, width }` (드래그 중 연속 발행)                                                    |
| `onColMove`                     | `columnMove` — `{ field, fromIndex, toIndex }`                                                               |
| 컬럼 표시/숨김                  | `columnVisible` — `{ field, visible }`                                                                       |
| `onPageChange`                  | `pageChange` — `{ pageIndex, pageSize }`                                                                     |
| `onExpand` (그룹/트리 펼침)     | `rowExpand` — `{ key, expanded }`                                                                            |
| 그룹화 기준 변경                | `groupChange` — `{ groupBy }`                                                                                |
| 피벗 설정 변경                  | `pivotChange` — `{ pivot }` (해제 시 null)                                                                   |
| `SetRowHeight`                  | `grid.setRowHeight(id, px)` + `rowResize` 이벤트                                                             |
| `onSearchEnd` / 서버 조회 완료  | `serverRequest`/`serverResponse`/`serverError` — 서버 사이드 모드 블록 요청 생명주기 (`server-side.md` 참조) |
| `findCheckedRow` / 선택 행 조회 | `grid.getSelectedRowData()` — 선택 행 데이터 배열                                                            |
| 체크박스 컬럼 체크 행 조회      | `grid.getCheckedRows(field?)` — `checkedValue` 기준, 선택과 별개                                             |
| 체크 설정                       | `grid.setRowChecked(id, checked, field?)` — ID 기준, 값 매핑 자동                                            |
| 조합 상태 감시                  | `grid.watch(selector, listener)` — 스냅샷 슬라이스 옵저버 (`events.md` 참조)                                 |

```tsx
<DataGrid
  columns={cols}
  data={rows}
  events={{
    afterEdit: (e) => save(e.row.id, e.column.field, e.newValue),
    selectionChange: (e) => console.log(e.selectedIds),
  }}
/>
```

## 컬럼 타입 매핑

| 레거시 컬럼 `Type`                 | moda-grid `ColumnDef`                                                        |
| ---------------------------------- | ---------------------------------------------------------------------------- |
| `Text`                             | 기본값 (`cellEditor` 생략)                                                   |
| `Int`/`Float`                      | `{ type: "number" }` 또는 `cellEditor: "number"` + `format: {kind:"number"}` |
| `Combo`                            | `{ cellEditor: "select", editorOptions: [...] }`                             |
| `MultiCombo`                       | `{ cellEditor: "multiselect", editorOptions: [...] }`                        |
| `CheckBox`                         | `{ cellEditor: "checkbox", headerCheckbox: true }`                           |
| `Radio`                            | `{ cellEditor: "radio", editorOptions: [...] }`                              |
| `Text`(MultiLine)                  | `{ cellEditor: "textarea", multiLine: true }`                                |
| `Date`                             | `{ cellEditor: "date" }` + `format: {kind:"date"}`                           |
| `Image`/`Button`/`Link`/`Progress` | `{ cellType: "image"\|"button"\|"link"\|"progress" }`                        |
| `Html`                             | `{ cellType: "html" }` — raw HTML, XSS 주의                                  |
| `AutoSum`/`Formula`                | `{ formula: "price * qty" }` / `{ cumulative: "amount" }`                    |
| 컬럼 이동 잠금                     | `movable: false` — `reorderable` 중에도 해당 컬럼 드래그 불가                |
| 읽기 전용                          | `{ editable: false }` — `mg-cell-readonly` 스타일 자동                       |
| 행 조건부 읽기 전용                | `{ editable: (row) => boolean }` — 행 데이터로 편집 가능 여부 결정           |
| `SaveName`                         | `field` (행 객체의 키)                                                       |
| `Align`/`HeaderAlign`              | `align` / `headerAlign` — left\|center\|right                                |
| 헤더 툴팁                          | `headerTooltip` — 헤더 마우스오버 시 title 툴팁                              |
| 입력값 파서                        | `valueParser: (text) => 저장값` — 편집 커밋/붙여넣기/가져오기에 적용         |

## 기능 매핑

| 레거시 그리드 기능                   | moda-grid                                                                                                                                        |
| ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| 헤더 필터 (FilterMode)               | `filterable: true` + `filterToggle` prop                                                                                                         |
| AutoFilter (헤더 ▾ 필터)             | `headerFilters` prop — 값 체크리스트 드롭다운                                                                                                    |
| 외부 조건 필터                       | `externalFilter(row)` 옵션/prop — 컬럼 필터와 별개로 행 제외                                                                                     |
| 정렬 후처리                          | `postSort(rows)` 옵션/prop — 정렬된 표시 행을 제자리 재배치                                                                                      |
| 다중 정렬                            | 헤더 클릭 + `Shift` 키 (자동)                                                                                                                    |
| 행 번호 컬럼 (Seq)                   | `rowNumbers` prop                                                                                                                                |
| 트리 (TreeMode)                      | `treeData: { getParentId }` / `{ childrenKey }`                                                                                                  |
| 소계 (SubSum)                        | `groupSubtotals: true` + `setGroupBy()`                                                                                                          |
| 피벗                                 | `pivot: { rows, columns, values }` / `pivotPanel` prop (드래그앤드랍 필드 패널) + `addPivotField`/`removePivotField`/`setPivotValueAgg`          |
| 고정 행 (Sum 머리글 등)              | `pinnedTopRows` / `pinnedBottomRows`                                                                                                             |
| 셀 병합 (MergeSheet)                 | `merge: "row"\|"col"\|"both"`                                                                                                                    |
| Append Scroll                        | `appendScroll: { dataSource }`                                                                                                                   |
| 가상 스크롤                          | `height` + `rowHeight` props                                                                                                                     |
| 페이지 크기 자동 (AutoPage)          | `autoPageSize` 옵션/prop — 뷰포트÷행 높이로 pageSize 계산, 리사이즈 재계산                                                                       |
| 내용 높이 레이아웃                   | `domLayout: "autoHeight"` 옵션/prop — 내부 스크롤 없이 행 수만큼 높이                                                                            |
| 상태 저장                            | `grid.getState()` / `applyState()`                                                                                                               |
| Excel/PDF보내기                      | `exportToXlsx({styled})` / `exportToPdf()`                                                                                                       |
| 컨텍스트 메뉴                        | `contextMenu` / `headerContextMenu` props (배열 또는 팩토리 함수)                                                                                |
| 기본 우클릭 메뉴                     | `defaultContextMenuItems` / `defaultHeaderContextMenuItems` 프리셋                                                                               |
| 다국어                               | `locale` prop (`enLocale` 프리셋)                                                                                                                |
| 채우기 핸들                          | 활성 셀 우하단 드래그 (자동, `fillRange` API)                                                                                                    |
| 붙여넣기 행 확장 (EditExtend)        | `pasteExtend: true` 옵션                                                                                                                         |
| 찾기/바꾸기                          | `grid.findCells()` / `grid.replaceAll()`                                                                                                         |
| 그룹 패널 (GroupBar)                 | `groupPanel` prop — 그룹 칩 표시·해제 + **헤더 드래그로 그룹 추가**                                                                              |
| 행 숨기기 (`setRowHidden`)           | `grid.setRowHidden(ids, bool)` / `hiddenRowIds` prop                                                                                             |
| 헤더 높이 (`HeaderRowHeight`)        | `headerHeight` 옵션/prop — 멀티레벨 헤더는 행 수로 균등 분할                                                                                     |
| 행 줄무늬 (`Alternate`)              | `striped` 옵션/prop + `grid.setStriped(bool)` — `--grid-stripe-bg` 변수                                                                          |
| 편집 진입 차단 (`OnBeforeEdit`)      | `beforeEdit` 옵션/prop — `false` 반환 시 편집 취소 (체크박스 토글 포함)                                                                          |
| 싱글클릭 편집 진입                   | `singleClickEdit` 옵션/prop — 클릭 한 번으로 편집 (cellType 셀 제외)                                                                             |
| 행 전체 편집 (`editType: "fullRow"`) | `editType: "fullRow"` 옵션/prop — 행의 모든 편집 셀 동시 편집, 커밋은 한 Undo 단위                                                               |
| 붙여넣기 전처리 (`OnBeforePaste`)    | `beforePaste` 옵션/prop — 문자열 반환 시 대체, `false` 시 취소                                                                                   |
| 복사 전처리                          | `beforeCopy` 옵션/prop — 문자열 반환 시 대체 TSV, `false` 시 복사 취소 (cut 구분)                                                                |
| 클립보드 구분자                      | `clipboardDelimiter` 옵션/prop — 복사·붙여넣기 셀 구분자 변경 (기본 탭, 탭 폴백 유지)                                                            |
| 마스터-디테일 (DetailBand)           | `detailRenderer`(React/vanilla) · `#detail` 슬롯(Vue) · `detail` snippet(Svelte) + `toggleRowDetail`/`setDetailExpanded`                         |
| 로딩 오버레이 (로딩 이미지)          | `loading` 옵션/prop + `grid.setLoading(bool)` — 코어 상태 기반 오버레이, `snapshot.loading`으로 노출. vanilla `mounted.setLoading`은 코어로 위임 |
| 자손 연쇄 선택 (TreeCheck)           | `treeData.selectsChildren: true` — 부모 체크박스가 자손 전체 선택/해제, 삼중 상태 표시                                                           |
| 종속 콤보 (Enum 체인)                | `editorOptions`에 함수 `(row) => string[]` — 다른 컬럼 값에 따라 행별 옵션 결정                                                                  |
| 검색 제외 컬럼                       | `searchable: false` 컬럼 옵션 — 전역 검색·찾기에서 제외. `getSearchText(row)`로 검색 텍스트 커스터마이즈                                         |
| 표시 문자열 복사                     | `copyFormatted` 옵션/prop 또는 `getSelectionTsv({ formatted: true })` — formatter/format 적용 텍스트                                             |
| 복수 범위 선택                       | Ctrl+드래그로 범위 추가 / `addCellRange` — 복사·삭제·집계는 모든 범위에 적용                                                                     |
| 행 조건부 선택                       | `isRowSelectable(row)` 옵션/prop — 불가 행은 클릭·체크박스·전체선택 모두 제외                                                                    |
| 자동 행 높이                         | `autoRowHeight` prop — multiLine 셀 기준                                                                                                         |
| 셀 노트 (Note)                       | `ColumnDef.note` — 코너 표시 + 툴팁                                                                                                              |

## 주요 API 메서드 매핑

| 기존 엔터프라이즈 그리드            | moda-grid                                                                                                                             |
| ----------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| `sheet.getValue(r, c)`              | `grid.getCellValueAt(r, c)` / `getCellValue(row, col)`                                                                                |
| `sheet.setValue(r, c, v)`           | `grid.setCellValue(r, c, v)` — 검증·이력·이벤트 포함                                                                                  |
| `sheet.getRowData(r)` / 행 조회     | `grid.getRowById(id)` — 숨김·필터 행 포함                                                                                             |
| `sheet.setRowData(r, {...})`        | `grid.updateRow(id, patch)` — 한 Undo 단위 + `dataChange`                                                                             |
| 일괄 행 변경 (add/update/remove)    | `grid.applyTransaction({ add, update, remove })`                                                                                      |
| `sheet.findText()`                  | `grid.findCells()` / `grid.findNext()` (순환 이동)                                                                                    |
| `sheet.replaceText()`               | `grid.replaceAll(find, replace)`                                                                                                      |
| `sheet.loadSearchData(json)`        | `grid.setData(rows)` 또는 `data` prop 변경                                                                                            |
| `sheet.getSaveJson()`               | `grid.getChanges()` — `{created, updated, deleted}`                                                                                   |
| 행 상태별 조회 필터                 | `grid.setRowStatusFilter(["I","U","D"])` — 변경분만 보기                                                                              |
| `sheet.addRow({row: i})`            | `grid.addRows(row, index)`                                                                                                            |
| `sheet.copyRows()` / 행 복제        | `grid.duplicateRows(rows)` — 원본 뒤 삽입 + I 마킹                                                                                    |
| `sheet.setRowStatus(r, "I")`        | `grid.addRows()` → 자동 I 마킹                                                                                                        |
| 셀 선택 후 Del                      | `grid.clearRange()` (어댑터에서 Delete/Backspace 자동)                                                                                |
| Ctrl+X 잘라내기                     | `grid.cutSelectionTsv()` — TSV 반환 + 지우기, Undo 1단위                                                                              |
| Shift+Space / Ctrl+Space            | `grid.selectEntireRow()` / `selectEntireColumn()` — 행·열 전체 선택                                                                   |
| Ctrl+A 전체 선택                    | `grid.selectAll()` — multi-cell/row는 전체 범위, single-cell은 전체 행                                                                |
| 범위 드래그 이동                    | `grid.moveRange(source, target)`                                                                                                      |
| 자동 채우기 시리즈 (숫자/날짜 외삽) | `GridOptions.fillSeries: true` / `fillRange(…, {series:true})`                                                                        |
| Ctrl+Enter 범위 입력                | `grid.fillActiveToSelection()` — 선택 범위 전체에 활성 셀 값                                                                          |
| `sheet.doSearch()`                  | (없음 — data prop 갱신)                                                                                                               |
| `sheet.doSort(col)`                 | `grid.toggleSort(field)`                                                                                                              |
| `sheet.setRowHeight(r, h)`          | `grid.setRowHeight(id, px)` — `null`로 해제                                                                                           |
| `sheet.print()`                     | `grid.print()` — `@media print` 스타일 포함                                                                                           |
| 컬럼 속성 변경                      | `grid.updateColumn(field, patch)` — 부분 갱신                                                                                         |
| `Editable: 0` (전체 편집 잠금)      | `GridOptions.editable: false` / `grid.setEditable(bool)`                                                                              |
| 변경 셀 플래시 (타사 변경 강조)     | `GridOptions.cellFlash: true` / `grid.setCellFlash(bool)` — `.mg-cell-flash` 클래스                                                   |
| `sheet.refresh()`                   | `grid.resetView()` — 정렬/필터/검색/페이지/선택 초기화                                                                                |
| `sheet.insertCol()`                 | `grid.addColumn(def, index)`                                                                                                          |
| 컬럼 공통 속성 일괄 지정            | `GridOptions.defaultColDef` — 모든 컬럼 기본값 병합                                                                                   |
| `sheet.removeCol()`                 | `grid.removeColumn(field)` — 정렬/필터도 함께 정리                                                                                    |
| 행 높이 드래그 (AllowRowResizing)   | `rowResizable` prop/옵션 — `rowNumbers` 필요                                                                                          |
| 행 드래그 이동                      | `col.rowDrag: true` 핸들 + `grid.moveRow(from, to)` / `rowReorder` 이벤트                                                             |
| `sheet.showRow(r)` / `focusRow`     | `grid.scrollToRow(rowIndex)` — 리프 인덱스(rows 기준). `grid.ensureRowVisible(id)` — 행 ID 기준, 접힌 그룹/트리·다른 페이지 자동 이동 |
| `showCell(r, c)` / `showColumn(c)`  | `grid.scrollToCell(r, c)` / `scrollToColumn(c)` — 수평 스크롤 포함                                                                    |
| 첫/마지막 행 이동                   | `grid.scrollToTop()` / `grid.scrollToBottom()`                                                                                        |
| 행 수 조회 (`RowCount`/`TotalRows`) | `grid.getDisplayedRowCount()` (현재 페이지) / `getFilteredRowCount()` (필터 후 전체) / `getTotalRowCount()` (원본)                    |
| 행 순회                             | `grid.forEachDisplayedRow(fn)` (표시 순서) / `forEachRow(fn)` (원본 순서)                                                             |
| 컬럼 조회                           | `grid.getColumn(field)` / `getColumnIndex(field)`                                                                                     |
| 정렬·필터 모델 일괄 적용            | `grid.getSortModel()`/`setSortModel(specs)` · `getFilterModel()`/`setFilterModel(map)` — getState보다 가벼운 조건만의 저장·복원       |
| 행 선택 일괄 지정                   | `grid.setRowSelection(ids)` — isRowSelectable·없는 ID 자동 제외, null은 전체 해제                                                     |
| 수정분 flat 수집 (`GetSaveData`)    | `grid.getModifiedRows()` → `{row, status}[]` (getChanges는 분리형 3배열)                                                              |
| 강제 재계산 (`Refresh`)             | `grid.refreshCells()` — 외부 제자리 수정·formatter 참조 상태 변경 후                                                                  |
| 다크 모드 / 테마                    | `theme="dark"` prop/`mountGrid` 옵션 또는 `.grid-theme-dark` 클래스 — `--grid-*` CSS 변수로 커스텀 팔레트                             |
| 키보드 이동 셀 화면 추적            | 자동 — `scrollRequest`가 행+열 좌표를 발행, 어댑터가 scrollIntoView                                                                   |
| `sheet.setGroupBy(...)` / 그룹 해제 | `grid.setGroupBy(fields)` — 빈 배열로 해제                                                                                            |
| `sheet.directDown2Excel()`          | `grid.exportToXlsx({ filename })`                                                                                                     |
| `sheet.dispose()`                   | 컴포넌트 언마운트 (자동)                                                                                                              |
| 행 조건부 스타일                    | `GridOptions.rowStyle` / `ColumnDef.rowStyle` — CSS 문자열·맵·함수, `<tr>`에 병합                                                     |
| 셀 커스텀 렌더러                    | React `renderCell` · Vue `#cell-{field}` 슬롯 · Svelte `cell` snippet · vanilla `cellRenderer`(DOM Node 반환)                         |

## 프레임워크별 미니멀 예시

```tsx
// React
import { DataGrid } from "@moda-grid/react";
import "@moda-grid/react/styles.css";
<DataGrid columns={cols} data={rows} height={480} rowHeight={37} />;
```

```vue
<!-- Vue 3 -->
<script setup>
import { DataGrid } from "@moda-grid/vue";
import "@moda-grid/vue/styles.css";
</script>
<DataGrid :columns="cols" :data="rows" :height="480" :row-height="37" />
```

```svelte
<!-- Svelte 5 -->
<script>
  import { DataGrid } from "@moda-grid/svelte";
  import "@moda-grid/svelte/styles.css";
</script>

<DataGrid columns={cols} data={rows} height={480} rowHeight={37} />
```

> **주의**: `styles.css` import는 필수다 — 없으면 레이아웃이 깨진다.

## 철학 차이

- **레거시 그리드**: 그리드 인스턴스가 DOM과 데이터를 소유, 명령형 API 중심.
- **moda-grid**: 헤드리스 코어(`GridCore`) + 프레임워크 어댑터.
  데이터 소유는 앱(`data` prop), 그리드는 표시 상태(정렬·필터·편집)만 관리.
  행 데이터를 바꾸려면 `data` prop을 갈아끼우거나 `grid.setData()`를 호출.
