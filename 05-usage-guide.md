# moda-grid 사용법 가이드

프레임워크별 임포트 방법과 전체 기능 명세. **새 기능이 추가될 때마다
이 문서의 "기능 명세" 섹션에 항목을 추가한다.**

- CDN 배포: <https://grid.modaolive.com>
  - `https://grid.modaolive.com/moda-grid.js` — 코어 IIFE 번들 (`window.ModaGrid`)
  - `https://grid.modaolive.com/style.css` — 공통 스타일

---

## 공통 개념

모든 어댑터는 헤드리스 코어(`@moda-grid/core`의 `GridCore`)를 감싸는
얇은 렌더링 레이어다. 어댑터를 쓰지 않고 코어만 직접 사용해도 된다.

### ColumnDef — 컬럼 정의

```ts
interface ColumnDef<TData> {
  field: keyof TData & string;             // 컬럼 키 (필수)
  header?: string;                         // 헤더 텍스트 (기본값: field)
  width?: number;                          // 고정 너비(px)
  minWidth?: number;                       // 리사이즈 최소 (기본 40)
  maxWidth?: number;                       // 리사이즈 최대
  resizable?: boolean;                     // 기본값 true
  sortable?: boolean;                      // 기본값 true
  visible?: boolean;                       // 기본값 true
  pinned?: "left" | "right" | null;        // 좌/우 고정
  filterable?: boolean;                    // 헤더 필터 입력 (기본값 false)
  filterType?: "text"|"number"|"date"|"set"; // 연산자 목록 (set=값 체크리스트)
  group?: string;                          // 컬럼 그룹 ID — 다단계 헤더
  comparator?: (a, b) => number;           // 커스텀 정렬 비교
  valueGetter?: (row) => unknown;          // 셀 값 추출 (기본값 row[field])
  formatter?: (value, row) => string;      // 표시 문자열 변환
  filterPredicate?: (value, row, filter) => boolean; // 커스텀 필터 판별
  aggregationFn?: "sum"|"avg"|"min"|"max"|"count";   // 그룹 행 집계
  editable?: boolean;                      // 기본값 true
  cellEditor?: "text"|"number"|"select"|"date"|"custom";
  editorOptions?: readonly string[];       // select 옵션
  valueSetter?: (row, value) => void;      // 기본값 row[field] = value
  validate?: (value, row) => boolean | string;
  rowDrag?: boolean;                       // 이 컬럼에 행 드래그 핸들
  cellClass?: string | ((p: CellClassParams) => string);   // 셀 클래스
  rowClass?: string | ((p: RowClassParams) => string);     // 행 클래스(합산)
}
```

### GridOptions — 그리드 생성 옵션

```ts
interface GridOptions<TData> {
  columns: ColumnDef<TData>[];
  data: TData[];
  getRowId?: (row) => string;              // 기본값 row.id ?? 자동 부여
  pageSize?: number;                       // 0이면 페이징 없음 (기본값 0)
  virtualScroll?: { rowHeight, viewportHeight, overscan? };
  selectionMode?: "single-cell" | "multi-cell" | "row";
  serverSide?: ServerSideOptions<TData>;   // 서버 사이드 + 무한 스크롤
  treeData?: TreeDataOptions<TData>;       // 계층형 데이터
  rowClass?: string | ((p: RowClassParams) => string); // 전체 행 클래스
  undoLimit?: number;                      // 편집 이력 깊이 (기본값 100, 0=비활성)
  pinnedTopRows?: TData[];                 // 상단 고정 행
  pinnedBottomRows?: TData[];              // 하단 고정 행
  columnGroups?: ColumnGroupDef[];         // 컬럼 그룹 헤더 라벨 ({ id, header })
}
```

---

## 순수 JavaScript (CDN)

CDN 번들은 **헤드리스 코어만** 포함한다 — DOM 렌더링은 직접 작성한다.
`GridCore`/`createGrid` 외에 스냅샷 조회·액션 API는 모두 `grid` 객체로
제공된다.

```html
<link rel="stylesheet" href="https://grid.modaolive.com/style.css" />
<script src="https://grid.modaolive.com/moda-grid.js"></script>

<input id="search" placeholder="검색…" />
<div id="app"></div>

<script>
  const { GridCore } = window.ModaGrid;

  const columns = [
    { field: "name", header: "이름" },
    { field: "role", header: "역할" },
    { field: "age", header: "나이", filterType: "number" },
  ];
  const users = [
    { id: 1, name: "Hana", role: "admin", age: 42 },
    { id: 2, name: "Daeho", role: "editor", age: 36 },
  ];

  const grid = new GridCore({ columns, data: users });

  function render() {
    const snap = grid.getSnapshot();
    document.getElementById("app").innerHTML = `
      <table class="mg-table">
        <thead><tr>
          ${snap.visibleColumns
            .map((c, i) => `<th data-field="${c.field}">${c.header ?? c.field}</th>`)
            .join("")}
        </tr></thead>
        <tbody>
          ${snap.rows
            .map(
              (row) => `<tr>${snap.visibleColumns
                .map((c) => `<td>${grid.getCellText(row, c)}</td>`)
                .join("")}</tr>`,
            )
            .join("")}
        </tbody>
      </table>`;
  }

  // 상태 변경 시 리렌더
  grid.subscribe(render);

  // 헤더 클릭 → 정렬 (Shift+클릭 시 다중 정렬)
  document.getElementById("app").addEventListener("click", (e) => {
    const th = e.target.closest("th[data-field]");
    if (th) grid.toggleSort(th.dataset.field, e.shiftKey);
  });
  document.getElementById("search").addEventListener("input", (e) => {
    grid.setSearch(e.target.value);
  });

  render();
</script>
```

`window.ModaGrid`는 core의 모든 export를 노출한다 (`GridCore`,
`createGrid`, `defineColumns`, `matchFilterValue`, `buildCsv` 등).

---

## React — `@moda-grid/react`

```tsx
import { DataGrid, useGridCore } from "@moda-grid/react";
import "@moda-grid/react/styles.css";

const columns: ReactColumnDef<User>[] = [
  { field: "id", header: "ID", width: 60 },
  { field: "name", header: "이름", filterable: true },
  {
    field: "role",
    header: "역할",
    // 커스텀 셀 렌더러 (React 전용 ColumnDef 확장)
    renderCell: (value) => <span className={`badge role-${value}`}>{value}</span>,
    // 커스텀 에디터 (선택)
    // renderEditor: (ctx) => <select value={ctx.value} onChange={...} />,
  },
];

function App() {
  const { grid, snapshot } = useGridCore({ columns, data: users });

  return (
    <>
      <input onChange={(e) => grid.setSearch(e.target.value)} />
      <button onClick={() => grid.exportToCsv({ filename: "users.csv" })}>
        CSV보내기
      </button>
      <DataGrid
        grid={grid}              // 외부 GridCore (생략 시 내부 생성)
        columns={columns}
        height={480}             // 가상 스크롤 뷰포트
        rowHeight={37}           // 가상 스크롤 행 높이
        selectionMode="multi-cell"
        columnController         // 우상단 컬럼 관리 팝오버
      />
    </>
  );
}
```

### props

| prop | 타입 | 기본값 | 설명 |
| ---- | ---- | ------ | ---- |
| `columns` | `ReactColumnDef[]` | 필수 | 컬럼 정의 (`renderCell`/`renderEditor` 확장) |
| `data` | `TData[]` | — | 행 데이터 (`grid` 제어 모드면 생략 가능) |
| `grid` | `Grid<TData>` | — | 외부 GridCore. 생략 시 내부 생성 |
| `height` / `rowHeight` | `number` | — | 가상 스크롤 (함께 지정) |
| `overscan` | `number` | `5` | 가상 스크롤 여유분 |
| `resizable` / `reorderable` / `selectable` | `boolean` | `true` | 리사이즈/재배치/행 클릭 선택 |
| `selectionMode` | `SelectionMode` | `single-cell` | 셀 선택 모드 |
| `serverSide` | `ServerSideOptions` | — | 서버 데이터 소스 (내부 생성 모드만) |
| `treeData` | `TreeDataOptions` | — | 트리 데이터 (내부 생성 모드만) |
| `columnController` | `boolean` | `false` | 컬럼 관리 팝오버 |
| `rowCheckboxes` | `boolean` | `false` | 행 체크박스 선택 컬럼 (헤더 전체선택 포함) |
| `statusBar` | `boolean` | `true` | 선택 영역 집계 상태바 |
| `getRowId` | `(row) => string` | — | 행 ID 함수 |
| `className` | `string` | — | 추가 클래스 |

---

## Vue 3 — `@moda-grid/vue`

```vue
<script setup lang="ts">
import { DataGrid, useGrid } from "@moda-grid/vue";
import "@moda-grid/vue/styles.css";

const { grid, state } = useGrid<User>({ columns, data: users });
</script>

<template>
  <input @input="grid.setSearch($event.target.value)" />
  <DataGrid
    :grid="grid"
    :columns="columns"
    :height="480"
    :row-height="37"
    selection-mode="multi-cell"
    column-controller
  >
    <!-- 컬럼별 커스텀 셀 슬롯 -->
    <template #cell-role="{ value, row }">
      <span :class="`badge role-${value}`">{{ value }}</span>
    </template>
    <!-- 커스텀 편집기 슬롯: ctx는 CellEditorContext -->
    <template #editor-role="ctx">
      <select :value="ctx.value" @change="ctx.setValue($event.target.value)" />
    </template>
  </DataGrid>
</template>
```

- `useGrid` → `{ grid: GridCore, state: ShallowRef<GridSnapshot> }`
- props는 React와 동일 (kebab-case로 전달)
- 슬롯: `#cell-{field}` = `{ value, row, column }`, `#editor-{field}` = `CellEditorContext`
- `useGridState(grid)` — 외부 GridCore의 스냅샷만 구독

## Vue 2.7 — `@moda-grid/vue2`

Composition API(`<script setup>`)로 Vue 3과 동일하게 사용한다:

```vue
<script setup lang="ts">
import { DataGrid, useGrid } from "@moda-grid/vue2";
import "@moda-grid/vue2/styles.css";

const { grid, state } = useGrid<User>({ columns, data: users });
</script>
```

props/슬롯/동작은 Vue 3 어댑터와 동일. props 선언이 런타임(`PropType`)
방식이며 `vite-plugin-vue2` 기반 앱에서 동작한다.

## Svelte — `@moda-grid/svelte` (Svelte 5 runes)

```svelte
<script lang="ts">
  import { createGridStore, DataGrid } from "@moda-grid/svelte";
  import "@moda-grid/svelte/styles.css";

  const store = createGridStore<User>({ columns, data: users });
  const { grid } = store;
</script>

<input oninput={(e) => grid.setSearch(e.currentTarget.value)} />

<DataGrid
  {store}                <!-- store 생략 시 columns/data로 내부 생성 -->
  columns={userColumns}
  height={480}
  rowHeight={37}
  selectionMode="multi-cell"
  columnController
>
  {#snippet cell({ value, row, column, text })}
    {#if column.field === "role"}
      <span class="badge role-{value}">{value}</span>
    {:else}
      {text}
    {/if}
  {/snippet}
</DataGrid>
```

- `createGridStore` → `GridStore` (Svelte store, `$store` 자동 구독으로 스냅샷 접근)
- `cell` snippet: `{ value, row, column, text }` — `text`는 formatter 적용 폴백
- `editor` snippet: `CellEditorContext` — 커스텀 편집기
- `toGridStore(grid)` — 외부 GridCore를 스토어로 래핑

---

## 기능 명세

### 정렬 / 다중 정렬

| 사용법 | 동작 |
| ------ | ---- |
| 헤더 클릭 | `grid.toggleSort(field)` — asc → desc → 해제 |
| Shift+헤더 클릭 | `grid.toggleSort(field, true)` — 조건 추가 (priority 부여) |
| `grid.setSort(field, dir)` | 단일 조건으로 교체 |
| `grid.clearSorts()` | 전체 해제 |
| `sortable: false` | 해당 컬럼 정렬 비활성 |

기본 비교: `Intl.Collator(numeric: true)` — 숫자/날짜 자연 비교.
`comparator`로 컬럼별 교체 가능. 스냅샷: `sort`(첫 조건), `sortState`(전체 배열).

### 필터 / 검색

| 사용법 | 동작 |
| ------ | ---- |
| `filterable: true` | 헤더 아래 연산자+입력 필터 행 표시 |
| `grid.setFilter(field, filter \| string)` | 컬럼 필터 설정 (문자열이면 contains) |
| `grid.clearFilters()` | 모든 필터 해제 |
| `grid.setSearch(text)` | 전역 검색 (표시 컬럼 전체, 대소문자 무시) |
| `filterType: "number" \| "date"` | 타입별 연산자 목록 |
| `filterType: "set"` | 값 체크리스트 필터 — `grid.getUniqueValues(field)`로 항목 구성, `operator: "set"` + `values`로 지정 |
| `filterPredicate` | 커스텀 판별 함수 |

복수 필터는 AND 결합. 연산자: text=`contains/equals/startsWith/endsWith`,
number=`equals/greaterThan/lessThan/inRange`, date=`equals/before/after`,
set=`values` 배열 포함 여부.

### 페이징

`grid.setPage(pageIndex, pageSize)` — `pageSize: 0`이면 비활성.
이전/다음 이동은 `setPage(snapshot.pageIndex ± 1)`로 구현한다.
페이징 UI는 어댑터에 없으므로 앱에서 `snapshot.pageIndex`/`pageCount`로
구현한다.

### 선택 (행/셀/범위)

| 사용법 | 동작 |
| ------ | ---- |
| `selectionMode` prop | `single-cell` / `multi-cell` / `row` |
| 행 클릭 | `toggleRowSelection(id)` (`selectable: false`로 비활성) |
| 셀 클릭 | `setActiveCell(row, col)` — 활성 셀 아웃라인 |
| 방향키 / Tab / Shift+방향키 | 셀 네비게이션 / 범위 확장 |
| `grid.setCellRange({start,end})` | 범위 선택 직접 지정 |
| `rowCheckboxes` prop | 행 체크박스 컬럼 — 헤더 전체선택 (`toggleAllRows`/`isAllSelected`/`isSomeSelected`) |
| `snapshot.selectedRowIds` / `selectedRange` | 선택 상태 |
| `snapshot.selectionAggregates` | 선택 범위 집계 `{cells,count,sum,avg,min,max}` — `statusBar` prop으로 표시 |

### 가상 스크롤

`height` + `rowHeight` props로 활성화. `snapshot.virtual`에
`startIndex/endIndex/totalHeight`가 계산되고 스페이서 행으로 렌더링된다.
그룹화·트리 모드에서는 평탄화된 `displayRows` 기준으로 작동.

### 컬럼 조작

| 기능 | API / UI |
| ---- | -------- |
| 너비 리사이즈 | `resizable` prop (기본 on) — 헤더 경계 드래그, `setColumnWidth(field, px)` |
| 순서 변경 | `reorderable` prop (기본 on) — 헤더 드래그, `reorderColumn(from, to)` |
| 고정 | `pinned: "left"\|"right"` — sticky + 경계 그림자, `setColumnPinned` |
| 표시/숨김 | `visible: false`, `setColumnVisible(field, bool)` |
| 전체 표시/숨김 | `setAllColumnsVisible(bool)` |
| 레이아웃 초기화 | `resetColumnLayout()` |
| 자동 너비 | `autoSizeColumn(field, measureText?)` / `autoSizeAllColumns(measureText?)` — 내용 기준 |
| 그룹 헤더 | `ColumnDef.group` + `GridOptions.columnGroups` — `snapshot.headerGroups`의 2단 헤더 |
| 컬럼 관리 UI | `columnController` prop — 우상단 `컬럼 ▾` 팝오버 (체크박스 + 전체 선택/해제) |

### 인라인 편집

- 셀 더블클릭 또는 활성 셀에서 문자 입력 → 편집 진입 (`editable: false`면 거부)
- `cellEditor`: `text` / `number` / `select` / `date` / `custom`
- `validate(value, row)` → `false`/`문자열`이면 저장 거부 + 에러 표시
- `valueSetter`로 커스텀 쓰기, `Enter` 저장 / `Esc` 취소
- React `renderEditor`, Vue `#editor-{field}`, Svelte `editor` snippet으로 커스텀 편집기

### 편집 이력 (Undo/Redo)

- `Ctrl/Cmd+Z` — `grid.undo()`, `Ctrl/Cmd+Y`·`Ctrl/Cmd+Shift+Z` — `grid.redo()`
- 인라인 편집 커밋과 TSV 붙여넣기가 기록된다 (붙여넣기는 1개 단위)
- `undoLimit` 옵션으로 깊이 제한 (기본값 100, `0`이면 이력 비활성)
- `setData` 시 이력 초기화. `snapshot.canUndo`/`canRedo`로 버튼 상태 연동

### 고정 행 / 총계 / 상태바

- `setPinnedTopRows(rows)` / `setPinnedBottomRows(rows)` — 상/하단 고정 행
  (`GridOptions.pinnedTopRows`/`pinnedBottomRows`로 초기 지정도 가능)
- `aggregationFn` 컬럼이 있으면 `snapshot.grandTotals`가 `<tfoot>` 총계 행으로 표시
- `snapshot.selectionAggregates` → `statusBar` prop(기본 on)으로 하단 상태바 표시

### 상태 저장/복원

```ts
localStorage.setItem("grid", JSON.stringify(grid.getState()));
grid.applyState(JSON.parse(localStorage.getItem("grid")!));
```

정렬/필터/검색/페이징/그룹화/선택모드/컬럼 레이아웃(너비·순서·표시·고정)을
직렬화 가능한 `GridPersistedState`로 저장·복원한다.

### 행 그룹화 + 집계

```ts
grid.setGroupBy(["role", "name"]);   // 다단계 그룹화
grid.toggleGroupExpanded(key);       // 접기/펼치기
grid.expandAllGroups();              // 전체 펼치기
grid.collapseAllGroups();            // 전체 접기
```

- 그룹 행에 `aggregationFn` 지정 컬럼의 집계 값 표시
- `snapshot.displayRows` — 그룹 헤더 + 리프가 평탄화된 목록
- 접힘 상태는 데이터 변경 후에도 유지 (신규 그룹은 자동 펼침)

### 트리 데이터

```ts
// flat parentId
treeData: { getParentId: (row) => row.parentId }
// nested children
treeData: { childrenKey: "children" }   // 기본값 'children'
```

첫 컬럼에 토글 화살표(`.mg-tree-toggle`) + depth 들여쓰기. 펼침 상태는
행 ID 기반 `expandedRowKeys`. 자식은 로컬 필터/정렬 대상이 아니다.

### CSV보내기

```ts
grid.exportToCsv({
  filename: "users.csv",        // 브라우저 다운로드 파일명
  visibleColumnsOnly: true,     // 숨김 컬럼 제외 (기본값)
  selectedRowsOnly: false,      // 선택 행만
});
```

- 현재 표시 상태(필터/정렬/그룹 펼침) 그대로 출력
- `,` `"` 줄바꿈 자동 이스케이프, UTF-8 BOM 포함 (엑셀 한글 대응)
- 서버 모드에서는 로드된 행만 출력

### 클립보드 (Copy & Paste)

- `Ctrl/Cmd+C` — 선택 범위(없으면 활성 셀)를 TSV로 복사
- `Ctrl/Cmd+Shift+C` — 헤더 행 포함 복사 (`getSelectionTsv({ includeHeaders: true })`)
- `Ctrl/Cmd+V` — 활성 셀부터 TSV 순차 붙여넣기 (Undo 1단위로 기록됨)
- `editable: false` 컬럼은 건너뜀, `validate` 실패 시 에러 기록
- `number` 편집기 컬럼은 자동 형변환
- 코어 API: `getSelectionTsv(options?)` / `pasteTsv(tsv, start?)` → `PasteResult`

### 서버 사이드 + 무한 스크롤

```ts
serverSide: {
  cacheBlockSize: 100,
  dataSource: {
    async getRows({ startRow, endRow, sortModel, filterModel }) {
      const res = await fetch(`/api/rows?start=${startRow}&end=${endRow}`);
      return res.json();  // { rows, lastRowIndex? }
    },
  },
}
```

- 뷰포트가 미로드 블록에 도달하면 `getRows` 자동 호출
- 미로드 행은 스켈레톤(`.mg-skeleton-row`) 렌더링
- 정렬/필터 변경 시 캐시 폐기 + 재요청 (stale 응답은 자동 폐기)
- `snapshot.serverSide.loading` 동안 로딩 인디케이터 표시
- 로컬 페이징/그룹화와 병용 불가

### 행 드래그앤드롭

- `rowDrag: true` 컬럼 셀에 `⠿` 핸들 표시 — 드래그로 행 이동
- 드롭 위치에 파란 가이드 라인, `Escape`로 취소
- `grid.moveRow(from, to)` / `grid.on("rowReorder", cb)` 이벤트
- 정렬·그룹화·트리·서버 모드에서는 자동 비활성 (`.mg-disabled`)

### 테마 / 커스텀 스타일

- 모든 색상은 `--grid-*` CSS 변수로 정의 (`styles.css` 상단)
- `.grid-theme-dark` 클래스를 그리드/상위 요소에 적용 → 다크 모드
- 커스텀 테마: `.my-theme { --grid-primary-color: #9333ea; }` 식으로 변수 재정의
- `ColumnDef.cellClass` / `rowClass`, `GridOptions.rowClass`로 조건부 클래스
  (함수 형태는 `{value,row,rowIndex,column}`/`{row,rowIndex}` 파라미터)

---

## 업데이트 규칙

새 기능이 추가되면 이 문서에 반드시 기록한다:

1. 해당하는 "기능 명세" 표에 행 추가 (또는 새 섹션)
2. 프레임워크별 차이가 있으면 해당 프레임워크 섹션에도 기록
3. ColumnDef/GridOptions 타입 변경 시 상단 스키마 블록도 갱신
