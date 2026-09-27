# IBSheet → moda-grid 마이그레이션

IBSheet에 익숙한 개발자를 위한 개념 매핑과 치트시트.

## TL;DR — 5줄 비교

```js
// IBSheet: 인스턴스가 DOM에 직접 렌더링
IBSheet.create({
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

### CDN/script 태그 환경 (IBSheet와 동일 패턴)

IBSheet처럼 빌드 도구 없이 `<script>` 태그만으로 쓰려면 `mountGrid`:

```html
<link rel="stylesheet" href="https://grid.modaolive.com/style.css" />
<script src="https://grid.modaolive.com/moda-grid.js"></script>
<div id="grid"></div>
<script>
  const m = ModaGrid.mountGrid(document.getElementById("grid"), {
    columns: [{ field: "name", header: "이름", filterable: true }],
    data: rows,
    events: { afterEdit: (e) => save(e) },   // IBSheet Events 대응
  });
  m.grid.exportToCsv();  // create() 반환 인스턴스와 동일한 위치
</script>
```

## 초기화 매핑

| IBSheet | moda-grid |
| ------- | --------- |
| `IBSheet.create({el, options, data})` | `<DataGrid columns={cols} data={rows} />` |
| `options.Cols: [{Header, Type, Name}]` | `columns: [{ field, header, cellEditor, width }]` |
| `options.Cfg` (SearchMode, Page…) | `DataGrid` props 또는 `useGridCore` 옵션 |
| `options.Events` | `events` prop → `{ cellClick, afterEdit, ... }` |
| 반환 `sheet` 인스턴스 | `onReady={(grid) => ...}` 또는 `useGridCore` |

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

| IBSheet Events | moda-grid `events` prop / `grid.on` |
| -------------- | ----------------------------------- |
| `onClick` | `cellClick` — `{ row, rowIndex, column, columnIndex, value }` |
| `onDblClick` | `cellDblClick` |
| `onEdit` | `afterEdit` — `{ row, column, oldValue, newValue }` |
| `onSelectRow` | `selectionChange` — `{ selectedIds }` |
| 드래그 범위 선택 (MouseSelect) | `selectionMode: "multi-cell"` — 셀 드래그/Shift+키로 범위 선택 |
| `onSort` | `sortChange` — `{ sortState }` |
| `onFilter` | `filterChange` — `{ filters }` |
| `onRowMove` | `rowReorder` — `{ fromIndex, toIndex, row }` |
| `onRowAdd` | `rowAdd` — `{ rows, index }` (addRows/duplicateRows) |
| `onRowDelete` | `rowDelete` — `{ ids }` (deleteRowsByIds — D 마킹 포함) |
| `onDataLoad` | `dataLoad` — `{ rows }` (setData 교체 후) |
| `onEditStart` (진입 시점) | `cellEditStart` — `{ row, rowIndex, column, columnIndex }` |
| 조합 상태 감시 | `grid.watch(selector, listener)` — 스냅샷 슬라이스 옵저버 (`events.md` 참조) |

```tsx
<DataGrid
  columns={cols} data={rows}
  events={{
    afterEdit: (e) => save(e.row.id, e.column.field, e.newValue),
    selectionChange: (e) => console.log(e.selectedIds),
  }}
/>
```

## 컬럼 타입 매핑

| IBSheet `Type` | moda-grid `ColumnDef` |
| -------------- | --------------------- |
| `Text` | 기본값 (`cellEditor` 생략) |
| `Int`/`Float` | `{ type: "number" }` 또는 `cellEditor: "number"` + `format: {kind:"number"}` |
| `Combo` | `{ cellEditor: "select", editorOptions: [...] }` |
| `MultiCombo` | `{ cellEditor: "multiselect", editorOptions: [...] }` |
| `CheckBox` | `{ cellEditor: "checkbox", headerCheckbox: true }` |
| `Radio` | `{ cellEditor: "radio", editorOptions: [...] }` |
| `Text`(MultiLine) | `{ cellEditor: "textarea", multiLine: true }` |
| `Date` | `{ cellEditor: "date" }` + `format: {kind:"date"}` |
| `Image`/`Button`/`Link`/`Progress` | `{ cellType: "image"\|"button"\|"link"\|"progress" }` |
| `Html` | `{ cellType: "html" }` — raw HTML, XSS 주의 |
| `AutoSum`/`Formula` | `{ formula: "price * qty" }` / `{ cumulative: "amount" }` |
| 읽기 전용 | `{ editable: false }` — `mg-cell-readonly` 스타일 자동 |
| `SaveName` | `field` (행 객체의 키) |

## 기능 매핑

| IBSheet 기능 | moda-grid |
| ------------ | --------- |
| 헤더 필터 (FilterMode) | `filterable: true` + `filterToggle` prop |
| AutoFilter (헤더 ▾ 필터) | `headerFilters` prop — 값 체크리스트 드롭다운 |
| 다중 정렬 | 헤더 클릭 + `Shift` 키 (자동) |
| 행 번호 컬럼 (Seq) | `rowNumbers` prop |
| 트리 (TreeMode) | `treeData: { getParentId }` / `{ childrenKey }` |
| 소계 (SubSum) | `groupSubtotals: true` + `setGroupBy()` |
| 피벗 | `pivot: { rows, columns, values }` / `pivotPanel` prop (드래그앤드랍 필드 패널) + `addPivotField`/`removePivotField`/`setPivotValueAgg` |
| 고정 행 (Sum 머리글 등) | `pinnedTopRows` / `pinnedBottomRows` |
| 셀 병합 (MergeSheet) | `merge: "row"\|"col"\|"both"` |
| Append Scroll | `appendScroll: { dataSource }` |
| 가상 스크롤 | `height` + `rowHeight` props |
| 상태 저장 | `grid.getState()` / `applyState()` |
| Excel/PDF보내기 | `exportToXlsx({styled})` / `exportToPdf()` |
| 컨텍스트 메뉴 | `contextMenu` / `headerContextMenu` props (배열 또는 팩토리 함수) |
| 기본 우클릭 메뉴 | `defaultContextMenuItems` / `defaultHeaderContextMenuItems` 프리셋 |
| 다국어 | `locale` prop (`enLocale` 프리셋) |
| 채우기 핸들 | 활성 셀 우하단 드래그 (자동, `fillRange` API) |
| 붙여넣기 행 확장 (EditExtend) | `pasteExtend: true` 옵션 |
| 찾기/바꾸기 | `grid.findCells()` / `grid.replaceAll()` |
| 그룹 패널 (GroupBar) | `groupPanel` prop — 그룹 칩 표시·해제 + **헤더 드래그로 그룹 추가** |
| 행 숨기기 (`setRowHidden`) | `grid.setRowHidden(ids, bool)` / `hiddenRowIds` prop |
| 헤더 높이 (`HeaderRowHeight`) | `headerHeight` 옵션/prop — 멀티레벨 헤더는 행 수로 균등 분할 |
| 행 줄무늬 (`Alternate`) | `striped` 옵션/prop + `grid.setStriped(bool)` — `--grid-stripe-bg` 변수 |
| 편집 진입 차단 (`OnBeforeEdit`) | `beforeEdit` 옵션/prop — `false` 반환 시 편집 취소 (체크박스 토글 포함) |
| 자동 행 높이 | `autoRowHeight` prop — multiLine 셀 기준 |
| 셀 노트 (Note) | `ColumnDef.note` — 코너 표시 + 툴팁 |

## 주요 API 메서드 매핑

| IBSheet | moda-grid |
| ------- | --------- |
| `sheet.getValue(r, c)` | `grid.getCellValueAt(r, c)` / `getCellValue(row, col)` |
| `sheet.setValue(r, c, v)` | `grid.setCellValue(r, c, v)` — 검증·이력·이벤트 포함 |
| `sheet.findText()` | `grid.findCells()` / `grid.findNext()` (순환 이동) |
| `sheet.replaceText()` | `grid.replaceAll(find, replace)` |
| `sheet.loadSearchData(json)` | `grid.setData(rows)` 또는 `data` prop 변경 |
| `sheet.getSaveJson()` | `grid.getChanges()` — `{created, updated, deleted}` |
| `sheet.addRow({row: i})` | `grid.addRows(row, index)` |
| `sheet.copyRows()` / 행 복제 | `grid.duplicateRows(rows)` — 원본 뒤 삽입 + I 마킹 |
| `sheet.setRowStatus(r, "I")` | `grid.addRows()` → 자동 I 마킹 |
| 셀 선택 후 Del | `grid.clearRange()` (어댑터에서 Delete/Backspace 자동) |
| Ctrl+X 잘라내기 | `grid.cutSelectionTsv()` — TSV 반환 + 지우기, Undo 1단위 |
| Shift+Space / Ctrl+Space | `grid.selectEntireRow()` / `selectEntireColumn()` — 행·열 전체 선택 |
| Ctrl+A 전체 선택 | `grid.selectAll()` — multi-cell/row는 전체 범위, single-cell은 전체 행 |
| 범위 드래그 이동 | `grid.moveRange(source, target)` |
| 자동 채우기 시리즈 (숫자/날짜 외삽) | `GridOptions.fillSeries: true` / `fillRange(…, {series:true})` |
| Ctrl+Enter 범위 입력 | `grid.fillActiveToSelection()` — 선택 범위 전체에 활성 셀 값 |
| `sheet.doSearch()` | (없음 — data prop 갱신) |
| `sheet.doSort(col)` | `grid.toggleSort(field)` |
| `sheet.setRowHeight(r, h)` | `grid.setRowHeight(id, px)` — `null`로 해제 |
| `sheet.print()` | `grid.print()` — `@media print` 스타일 포함 |
| 컬럼 속성 변경 | `grid.updateColumn(field, patch)` — 부분 갱신 |
| `Editable: 0` (전체 편집 잠금) | `GridOptions.editable: false` / `grid.setEditable(bool)` |
| `sheet.refresh()` | `grid.resetView()` — 정렬/필터/검색/페이지/선택 초기화 |
| `sheet.insertCol()` | `grid.addColumn(def, index)` |
| `sheet.removeCol()` | `grid.removeColumn(field)` — 정렬/필터도 함께 정리 |
| 행 높이 드래그 (AllowRowResizing) | `rowResizable` prop/옵션 — `rowNumbers` 필요 |
| 행 드래그 이동 | `col.rowDrag: true` 핸들 + `grid.moveRow(from, to)` / `rowReorder` 이벤트 |
| `sheet.showRow(r)` / `focusRow` | `grid.scrollToRow(rowIndex)` — 표시 인덱스 기준 |
| 다크 모드 / 테마 | `theme="dark"` prop/`mountGrid` 옵션 또는 `.grid-theme-dark` 클래스 — `--grid-*` CSS 변수로 커스텀 팔레트 |
| 키보드 이동 셀 화면 추적 | 자동 — `scrollRequest`가 행+열 좌표를 발행, 어댑터가 scrollIntoView |
| `sheet.setGroupBy(...)` / 그룹 해제 | `grid.setGroupBy(fields)` — 빈 배열로 해제 |
| `sheet.directDown2Excel()` | `grid.exportToXlsx({ filename })` |
| `sheet.dispose()` | 컴포넌트 언마운트 (자동) |

## 프레임워크별 미니멀 예시

```tsx
// React
import { DataGrid } from "@moda-grid/react";
import "@moda-grid/react/styles.css";
<DataGrid columns={cols} data={rows} height={480} rowHeight={37} />
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

- **IBSheet**: 그리드 인스턴스가 DOM과 데이터를 소유, 명령형 API 중심.
- **moda-grid**: 헤드리스 코어(`GridCore`) + 프레임워크 어댑터.
  데이터 소유는 앱(`data` prop), 그리드는 표시 상태(정렬·필터·편집)만 관리.
  행 데이터를 바꾸려면 `data` prop을 갈아끼우거나 `grid.setData()`를 호출.
