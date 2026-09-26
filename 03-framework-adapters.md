# 프레임워크 어댑터

세 어댑터는 모두 같은 계약을 구현한다: 코어의 `subscribe`/`getSnapshot`을
각 프레임워크의 반응형 시스템에 연결하고, 기본 `<table>` 렌더러를 제공한다.

## React — `@moda-grid/react`

### useGridCore

```tsx
import { useGridCore, DataGrid } from "@moda-grid/react";
import "@moda-grid/react/styles.css";

function UserTable({ users }: { users: User[] }) {
  const { grid, snapshot } = useGridCore({ columns, data: users });

  return (
    <>
      <input onChange={(e) => grid.setSearch(e.target.value)} />
      <DataGrid grid={grid} columns={columns} />
    </>
  );
}
```

- 내부적으로 `useSyncExternalStore` 사용 → 스냅샷 메모이제이션 필수
  (core가 보장)
- `options.data`/`options.columns` 참조가 바뀌면 `useEffect`로 그리드에 반영
- 기존 `Grid` 인스턴스를 외부에서 만들었다면 `useGridSnapshot(grid)`만으로
  구독 가능

### DataGrid

두 가지 사용 모드를 지원한다.

```tsx
// 1) 내부 인스턴스 모드 — columns/data만 넘기면 GridCore를 자체 생성·관리
<DataGrid columns={columns} data={users} height={480} rowHeight={37} />

// 2) 제어 모드 — 툴바 등에서 grid를 직접 써야 할 때
const { grid, snapshot } = useGridCore({ columns, data });
<DataGrid grid={grid} columns={columns} height={480} rowHeight={37} />
```

| prop | 타입 | 기본값 | 설명 |
| ---- | ---- | ------ | ---- |
| `columns` | `ReactColumnDef[]` | 필수 | 컬럼 정의 (`renderCell` 확장) |
| `data` | `TData[]` | — | 행 데이터 (`grid` 제어 모드면 생략 가능) |
| `grid` | `Grid<TData>` | — | 외부 GridCore. 생략 시 내부 생성 |
| `height` | `number` | — | 가상 스크롤 뷰포트 높이 (`rowHeight`와 함께) |
| `rowHeight` | `number` | — | 가상 스크롤 행 높이 |
| `overscan` | `number` | `5` | 가상 스크롤 여유분 |
| `resizable` | `boolean` | `true` | 헤더 리사이저 핸들 표시 |
| `serverSide` | `ServerSideOptions` | — | 서버 사이드 데이터 소스 (무한 스크롤 + 스켈레톤). 내부 GridCore 생성 모드에서만 적용 |
| `reorderable` | `boolean` | `true` | 헤더 DnD 컬럼 재배치 |
| `selectable` | `boolean` | `true` | 행 클릭 선택 토글 |
| `getRowId` | `(row) => string` | — | 행 ID 함수 |
| `columnController` | `boolean` | `false` | 우상단 컬럼 관리 도구(체크박스 팝오버 + 전체 선택/해제) 표시 |
| `filterToggle` | `boolean` | `false` | 우상단 필터 행 표시/숨김 버튼 (filterable 컬럼이 있을 때만 렌더) |
| `rowCheckboxes` | `boolean` | `false` | 행 체크박스 선택 컬럼 표시 (헤더에 전체선택 체크박스) |
| `statusBar` | `boolean` | `true` | 선택 영역 집계 상태바 표시 |
| `className` | `string` | — | 추가 클래스 |

`height` + `rowHeight`가 있으면 스크롤 컨테이너 + 상/하 스페이서 `<tr>`로
가상화되고 `snapshot.virtualRows`만 DOM에 그린다. 헤더는 sticky로 고정.

### ReactColumnDef.renderCell

```tsx
const columns: ReactColumnDef<User>[] = [
  { field: "name", header: "이름" },
  {
    field: "role",
    header: "역할",
    renderCell: (value, row, col) => (
      <span className={`badge role-${value}`}>{String(value)}</span>
    ),
  },
];
```

`renderCell`이 없는 컬럼은 `grid.getCellText()`(formatter 적용)로 렌더링한다.

## Vue 3 — `@moda-grid/vue`

### useGrid 컴포저블

```vue
<script setup lang="ts">
import { useGrid, DataGrid } from "@moda-grid/vue";
import "@moda-grid/vue/styles.css";

const { grid, state } = useGrid<User>({ columns, data: users });
// state: ShallowRef<GridSnapshot<User>>
</script>

<template>
  <input @input="grid.setSearch($event.target.value)" />
  <DataGrid :grid="grid" :columns="columns" :height="480" :row-height="37">
    <template #cell-role="{ value, row }">
      <span :class="`badge role-${value}`">{{ value }}</span>
    </template>
  </DataGrid>
</template>
```

- `shallowRef`로 스냅샷을 노출 — 스냅샷이 통째로 교체되는 구조라 deep
  반응형이 불필요. 외부에서 쓸 때는 읽기 전용으로 취급할 것
- 컴포넌트/이펙트 스코프 안에서 호출하면 `onScopeDispose`로 자동 구독 해제
- `useGridState(grid)`는 외부에서 만든 GridCore의 스냅샷만 구독할 때 사용

### DataGrid (`DataGrid.vue` — `<script setup>` SFC)

props는 React 어댑터와 동일하다 (`columns`, `data`, `grid`, `height`,
`rowHeight`, `overscan`, `selectable`, `getRowId`, `resizable`,
`reorderable`, `columnController`, `filterToggle`, `rowCheckboxes`, `statusBar`). `columns`/`data`만
넘기면 내부에서 GridCore를 생성하고, `grid`를 넘기면 외부 인스턴스를
구독한다. `height`+`rowHeight`로 가상 스크롤 활성화 — 스크롤 컨테이너 +
상/하 스페이서 `<tr>`, `@scroll → grid.handleScroll`.

### 스코프드 셀 슬롯

각 `<td>`는 `cell-{field}`라는 이름의 동적 슬롯을 노출한다:

```vue
<DataGrid :grid="grid" :columns="columns">
  <template #cell-role="{ value, row, column }">
    <span :class="`role-badge role-${value}`">{{ value }}</span>
  </template>
  <template #cell-email="{ value }">
    <a :href="`mailto:${value}`">{{ value }}</a>
  </template>
</DataGrid>
```

슬롯 props: `value`(원시 셀 값), `row`(행 객체), `column`(컬럼 정의).
슬롯이 없는 컬럼은 `getCellText()`(formatter 적용)로 렌더링한다.

## Svelte — `@moda-grid/svelte` (Svelte 5 runes)

### createGridStore / toGridStore

```svelte
<script lang="ts">
  import { createGridStore, DataGrid } from "@moda-grid/svelte";
  import "@moda-grid/svelte/styles.css";

  const store = createGridStore<User>({ columns, data: users });
  const { grid } = store;
</script>

<input oninput={(e) => grid.setSearch(e.currentTarget.value)} />
<DataGrid {store} />
```

- `readable`로 `GridCore`를 래핑 → `$store` 자동 구독으로 스냅샷 접근,
  액션은 `store.grid`로 호출
- `toGridStore(grid)` — 이미 생성된 GridCore 인스턴스도 동일하게 래핑 가능

### DataGrid (`DataGrid.svelte` — runes 모드)

props는 다른 어댑터와 동일하다 (`store`/`columns`/`data`/`height`/
`rowHeight`/`overscan`/`selectable`/`getRowId`/`resizable`/`reorderable`).
`store`가 없으면 내부에서
`createGridStore`로 생성한다. `height`+`rowHeight`로 가상 스크롤 활성화 —
스크롤 컨테이너 + 상/하 스페이서 `<tr>`, `onscroll → grid.handleScroll`.

### 커스텀 셀 — `{#snippet cell}`

Svelte 5의 named snippet으로 단일 `cell` snippet에 모든 컬럼의 커스텀
렌더링을 위임한다:

```svelte
<DataGrid {store} height={480} rowHeight={37}>
  {#snippet cell({ value, column, text })}
    {#if column.field === "role"}
      <span class="role-badge role-{value}">{value}</span>
    {:else}
      {text}
    {/if}
  {/snippet}
</DataGrid>
```

snippet 컨텍스트(`CellContext<TData>`): `value`(원시 값), `row`,
`column`(컬럼 정의), `text`(formatter 적용 문자열 — 폴백용).
`cell` snippet이 없으면 `getCellText()`로 기본 렌더링한다.

## Vue 2 — `@moda-grid/vue2`

> **주의**: Vue 2는 2023-12-31에 EOL이 됐다. 이 패키지는 마지막 마이너인
> **Vue 2.7**만 지원한다 — 2.7은 Composition API와 `<script setup>`을
> 내장해 `@vue/composition-api` 플러그인이 필요 없다.

### useGrid 컴포저블 (Vue 2.7)

```vue
<script setup lang="ts">
import { useGrid, DataGrid } from "@moda-grid/vue2";
import "@moda-grid/vue2/styles.css";

const { grid, state } = useGrid({ columns, data: users });
</script>

<template>
  <input @input="grid.setSearch($event.target.value)" />
  <DataGrid :grid="grid" :columns="columns" :height="480" :row-height="37">
    <template #cell-role="{ value }">
      <span :class="`badge role-${value}`">{{ value }}</span>
    </template>
  </DataGrid>
</template>
```

Vue 3 어댑터와 API가 동일하다 (`useGrid`/`useGridState`/`DataGrid` SFC +
`cell-{field}` 스코프드 슬롯 + 가상 스크롤). 차이점:

- props 선언이 타입 기반이 아닌 **런타임 선언**(`PropType`) — Vue 2.7
  컴파일러의 제약 때문.
- 템플릿에서 `??`/`?.`를 쓰지 않는다 — `||`와 `&&`로 대체.
- 앱 진입점이 `new Vue({ render }).$mount()` 형태.
- `vue-tsc`가 Vue 2를 지원하지 않으므로 dev-vue2의 타입체크는 `tsc`로
  `.ts` 파일만 검사하고, `.vue` 템플릿 안에서는 TS `as` 캐스트를
  피한다 (컴파일러가 표현식을 JS로 해석). 핸들러를 script에 분리할 것.

## DataGrid 공통 렌더링 동작

네 개 어댑터의 기본 `DataGrid`는 스냅샷을 그대로 렌더링한다:

- 헤더 클릭 → `grid.toggleSort(field, shiftKey)` — 일반 클릭은 단일 정렬,
  **Shift+클릭은 다중 정렬 조건 추가**. 2개 이상일 때 화살표 옆에
  `.mg-sort-priority` 숫자 배지로 우선순위 표시
- `filterable: true`인 컬럼이 있으면 헤더 아래 **필터 입력 행**을 자동 렌더링
  → 연산자 셀렉트(`col.filterType`의 `FILTER_OPERATORS_BY_TYPE`) + 값 입력
  (+ `inRange` 시 상한 입력). `grid.setFilter(field, ColumnFilter)` 호출
- 행 클릭 → `grid.toggleRowSelection(id)` (`selectable` prop으로 비활성 가능)
- 행이 없으면 `mg-empty` 행 표시
- 가상 스크롤은 4개 어댑터 모두 내장 (`height`+`rowHeight` props) —
  스크롤 컨테이너 + 상/하 스페이서로 `snapshot.virtualRows`만 렌더링
- **행 그룹화**: `snapshot.displayRows`가 있으면 그룹 헤더 행
  (`.mg-group-row`, 클릭 → `toggleGroupExpanded`, 첫 컬럼에 ▾/▸ 토글 +
  `field: value (N)` 라벨, 나머지 컬럼에 `aggregates` 집계 값)과 리프 행을
  섞어 렌더링. 리프 행은 `rowIndex`로 기존 선택/편집 인덱스와 연결되고
  `depth`만큼 첫 셀 들여쓰기. 가상 스크롤 병용 시 `displayRows.slice` 사용
- **컬럼 리사이즈** (`resizable` prop, 기본값 true) — 헤더 우측 경계의
  `.mg-resizer` 버튼을 드래그하면 `grid.setColumnWidth(field, w)` 호출.
  mousemove는 `requestAnimationFrame`으로 스로틀해 프레임당 최대 1회만
  코어에 반영하고, `resizable: false` 컬럼은 핸들 미표시. 포커스된
  리사이저에서 `←`/`→` 키로도 ±10px 조절 가능 (키보드 접근성).
- **컬럼 순서 변경** (`reorderable` prop, 기본값 true) — 헤더 셀에 HTML5
  DnD(`draggable` + dragstart/dragover/drop/dragend)를 연결해
  `grid.reorderColumn(draggedId, targetId)` 호출. 드롭 대상 셀은
  `.mg-col-dragover`로 표시된다.
- **컬럼 고정** (`ColumnDef.pinned` 또는 `grid.setColumnPinned`) — 고정
  컬럼이 하나라도 있으면 테이블을 `overflow-x: auto` 스크롤 컨테이너로
  감싸고, 고정 헤더/바디/필터/그룹 셀에 `position: sticky` +
  `snapshot.pinOffsets`의 `left`/`right` 오프셋을 적용한다. 렌더링 순서는
  코어가 정렬한 `visibleColumns` ([left | scrollable | right])를 그대로
  따른다. 고정/스크롤 영역 경계에는 `.mg-pin-left-edge`/`.mg-pin-right-edge`
  클래스로 `inset box-shadow` 구분선을 표시하고, `.mg-pinned` 셀은 불투명
  배경으로 스크롤 콘텐츠를 가린다. 헤더 고정 셀은 수직/수평 sticky가 겹치는
  최상단 z-index를 가진다. 고정 컬럼은 정확한 오프셋 계산을 위해 `width`
  지정을 권장한다 (미지정 시 `DEFAULT_COLUMN_WIDTH`).
- **셀 선택/키보드 내비** (`selectionMode` prop) — `<table role="grid"
  tabindex="0">`에 키 핸들러를 달고 `KeyboardEvent.key` → `CellNavigation`
  매핑 후 `grid.navigateCell(dir, shiftKey)` 호출. input/textarea/select
  안의 키 입력은 무시하고 `Escape`는 `clearCellSelection()`. 셀 클릭은
  `setActiveCell`. 활성 셀은 `.mg-cell-active`(아웃라인), 범위는
  `.mg-cell-selected`로 표시. 가상 스크롤 시 행 인덱스는
  `virtual.startIndex + i`로 보정하고, 스냅샷의 `virtual.scrollTop`을
  스크롤 컨테이너 DOM에 동기화한다 (scrollIntoView).
- **인라인 셀 편집** — 더블클릭/Enter/F2/문자 입력으로 `startEditing` 진입.
  편집 중 셀은 `.mg-cell-editing`으로 표시되고 에디터(`mg-editor`)로 교체
  렌더링: `cellEditor` 타입별 내장 편집기(`text`/`number`/`select`/`date`).
  에디터 내부 키는 `stopPropagation`으로 그리드 키 내비와 분리 — Enter
  저장+아래 이동, Tab 저장+좌/우 이동, Escape 취소. `editError`는
  `.mg-edit-error`로 셀 아래 표시하고 입력 테두리를 빨갛게 한다.

커스텀 셀 렌더링은 프레임워크 관용구를 따른다: React는 `renderCell` prop,
Vue/Vue2는 `#cell-{field}` 스코프드 슬롯, Svelte는 `{#snippet cell}`
(+ `column.field`로 분기) 방식.

커스텀 편집기도 같은 관용구를 쓴다 (`CellEditorContext` 전달): React는
`ReactColumnDef.renderEditor`, Vue/Vue2는 `#editor-{field}` 슬롯,
Svelte는 `{#snippet editor}`.

- **트리 데이터** (`treeData` prop) — `displayRows`의 리프 행을
  `depth`만큼 들여쓰기하고, 첫 컬럼 셀에 `hasChildren` 노드의 토글
  버튼(`.mg-tree-toggle`, `▾/▸`)을 렌더링해 `grid.toggleTreeExpanded(id)`를
  호출한다. 자식 없는 노드는 `.mg-tree-leaf` 스페이서로 정렬을 맞춘다.
  접힘/펼침은 `expandedRowKeys` 기반이며 가상 스크롤 높이에 반영된다.

- **행 드래그앤드롭** (`ColumnDef.rowDrag`) — 핸들 셀
  (`.mg-row-drag-handle`, HTML5 `draggable`)을 드래그하면
  `grid.beginRowDrag` → 각 행의 `dragover`에서 포인터 Y 위치로
  드롭 갭을 계산해 `grid.updateRowDropPosition` → `drop` 시
  `grid.endRowDrag(true)`로 커밋(`moveRow` + `rowReorder` 이벤트).
  드래그 중 행은 `.mg-row-dragging`(반투명), 갭 위치는
  `.mg-drop-before`/`.mg-drop-after` 파란 라인으로 표시한다.
  `Escape`/`dragend`는 `endRowDrag(false)`로 취소. 정렬·그룹화·서버
  모드에서는 핸들이 비활성(`.mg-disabled`)으로 표시된다.

- **서버 사이드/무한 스크롤** (`serverSide` prop) — `dataSource.getRows`
  결과가 도착하기 전 미로드 행(`undefined` 홀)은 `.mg-skeleton-row`로
  컬럼별 shimmer 바(`.mg-skeleton-bar`)를 렌더링한다. 고정 컬럼 스켈레톤도
  같은 sticky 오프셋을 적용해 정렬을 유지한다. `serverSide.loading` 동안
  스크롤 컨테이너 하단에 `.mg-loading` 인디케이터를 표시한다.

- **컬럼 관리 도구** (`columnController` prop, 기본값 false) — 그리드
  우상단에 `컬럼 ▾` 버튼을 오버레이하고, 클릭 시 체크박스 팝오버
  (`.mg-colctl-panel`)를 연다. 목록은 `snapshot.columns` 순서대로
  숨겨진 컬럼도 포함하고(`header ?? field` 라벨), 체크 변경은
  `grid.setColumnVisible(field, checked)`를 호출한다. 상단의
  전체 선택/전체 해제 버튼은 `grid.setAllColumnsVisible`로 일괄 토글하며
  전부 숨기는 것도 허용된다(빈 그리드). 패널은 오버레이 클릭이나
  `Escape`로 닫힌다.

- **필터 행 토글** (`filterToggle` prop, 기본값 false) — 우상단에
  `필터 ▾/▸` 버튼을 오버레이하고 `grid.setFilterRowVisible`로 필터 입력
  행을 표시/숨긴다. filterable 컬럼이 하나도 없으면 버튼은 렌더링되지
  않는다. 숨겨도 이미 적용된 필터 조건은 유지된다 — 표시 여부만 코어
  `snapshot.filterRowVisible`로 제어한다. `columnController`와 함께 쓰면
  같은 오버레이 영역에 나란히 배치된다.

- **테마/커스텀 클래스** — 모든 색상은 `--grid-*` CSS 변수 기반이며,
  `.grid-theme-dark` 클래스를 그리드(또는 상위 요소)에 적용하면 다크
  테마로 전환된다. `ColumnDef.cellClass`/`rowClass`와
  `GridOptions.rowClass`로 지정한 클래스는 `<td>`/`<tr>`에 합산되어
  조건부 스타일링이 가능하다 (코어 문서 "테마 시스템" 참고).

**클립보드/이력**: `Ctrl/Cmd+C` → `grid.getSelectionTsv()` 결과를
`navigator.clipboard.writeText`로 복사 (`Ctrl/Cmd+Shift+C`는 헤더 행 포함),
`Ctrl/Cmd+V` → `readText()` 후 `grid.pasteTsv(text)`로 붙여넣기 (TSV).
`Ctrl/Cmd+Z` → `grid.undo()`, `Ctrl/Cmd+Y`·`Ctrl/Cmd+Shift+Z` →
`grid.redo()`로 편집/붙여넣기 되돌리기. 입력 요소 내부에서는 기본 동작 유지.

**신규 상용 기능 렌더링** (4개 어댑터 공통):

- **행 체크박스** (`rowCheckboxes` prop) — 맨 앞 `mg-check-cell` 컬럼에
  헤더 전체선택(`grid.isAllSelected()`/`isSomeSelected()`/`toggleAllRows()`)
  + 행별 체크박스를 렌더링한다. 그룹/스켈레톤/고정 행에는 빈 셀로 정렬만 맞춘다.
- **컬럼 그룹 헤더** — 스냅샷 `headerGroups`가 있으면 헤더를 두 `<tr>`로
  렌더링한다: 그룹 셀은 `colspan`+`mg-colgroup`, 그룹 없는 컬럼은 `rowspan=2`.
- **Set 필터** — `filterType: 'set'` 컬럼은 연산자 셀렉트 대신
  `<details>` 팝오버의 값 체크리스트(`mg-setfilter`)를 표시한다.
  항목은 `grid.getUniqueValues(field)`, 토글은 `operator: 'set'` 필터로 갱신.
- **고정 행** — `snapshot.pinnedTopRows`/`pinnedBottomRows`를 tbody 맨 위/
  맨 아래에 `mg-pinned-top-row`/`mg-pinned-bottom-row`로 렌더링한다.
- **전체 총계** — `snapshot.grandTotals`가 있으면 `<tfoot>`의
  `mg-total-row`로 표시 (`formatAggregate` 포맷).
- **상태바** (`statusBar` prop, 기본값 true) — `snapshot.selectionAggregates`가
  있으면 그리드 하단 `mg-statusbar`에 셀 수/개수/합계/평균/최소/최대를 표시.

페이징 UI는 DataGrid에 포함하지 않고 앱에서 `grid.setPage()`로 구현한다
(dev 앱의 페이지 툴바 참고).

## 어댑터 공통 규칙

새 프레임워크 어댑터를 만들 때 지키는 최소 규칙:

1. **얇게 유지** — 상태 로직을 어댑터에 복제하지 않는다. 어댑터는 구독 연결
   + 렌더링만 한다.
2. **스냅샷 통째 전달** — 필드별로 ref/state를 나누지 않는다.
   `GridSnapshot`이 원자적이므로 torn state가 없다.
3. **styles.css는 core 재수출** — 각 패키지의 `styles.css`는
   `@import "@moda-grid/core/styles.css"` 한 줄로 유지한다.
4. `workspace:*`로 `@moda-grid/core`를 dependencies에, 프레임워크는
   `peerDependencies`에 둔다.
