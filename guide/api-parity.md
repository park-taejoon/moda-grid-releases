# 렌더러 API 파리티 매트릭스

5개 렌더러(React / Vue 3 / Vue 2.7 / Svelte 5 / `mountGrid`)는 모두 같은
`GridCore` 위에서 동작한다. prop·옵션 이름과 동작은 최대한 동일하게 유지하고,
프레임워크 관용구에 따라 표현 채널만 달라진다.

> 이 표는 **코드에서 추출한 실제 표면**을 기준으로 한다. 새 기능을 추가하면
> 이 문서도 함께 갱신한다 (AGENTS.md "기능 추가 시" 규칙).

## 진입점·라이프사이클

|                  | React                           | Vue 3                    | Vue 2.7                  | Svelte 5                        | Vanilla                  |
| ---------------- | ------------------------------- | ------------------------ | ------------------------ | ------------------------------- | ------------------------ |
| 엔트리           | `<DataGrid>`                    | `<DataGrid>`             | `<DataGrid>`             | `<DataGrid>`                    | `mountGrid(el, options)` |
| 외부 그리드 주입 | `grid` prop                     | `grid` prop              | `grid` prop              | `store` prop (`GridStore`)      | `grid` 옵션              |
| 코어 접근 훅     | `useGridCore`/`useGridSnapshot` | `useGrid`/`useGridState` | `useGrid`/`useGridState` | `createGridStore`/`toGridStore` | `mounted.grid`           |
| 스냅샷 구독      | `useGridSnapshot`               | `useGridState`           | `useGridState`           | `$store` (Svelte store)         | `grid.subscribe`         |
| 준비 콜백        | `onReady`                       | `onReady`                | `onReady`                | `onReady`                       | — (동기 반환으로 불필요) |
| 선언형 이벤트    | `events` prop                   | `events` prop            | `events` prop            | `events` prop                   | `events` 옵션            |
| 해제             | 언마운트                        | 언마운트                 | 언마운트                 | 언마운트                        | `mounted.destroy()`      |

`events` 계약은 5개 모두 `GridEventHandlers<TData>` — 마운트 시 `grid.on`으로
구독하고 해제 시 구독을 뗀다. `GridCore` 생성자의 `GridOptions.events` 경로도
있지만(그리드 수명까지 유지), 렌더러 표면의 `events`는 마운트 수명을 따른다.

## 공통 prop·옵션 (5개 렌더러 동일 이름)

| 카테고리      | prop/옵션                                                                                                                                                          |
| ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 데이터/컬럼   | `columns` · `defaultColDef` · `data` · `getRowId`                                                                                                                  |
| 레이아웃      | `height` · `rowHeight` · `overscan` · `virtualColumns` · `domLayout` · `autoPageSize` · `dir` · `headerHeight` · `autoRowHeight` · `striped`                       |
| 선택          | `selectable` · `selectionMode` · `hiddenRowIds`                                                                                                                    |
| 편집/클립보드 | `editType` · `singleClickEdit` · `beforeEdit` · `beforePaste` · `beforeCopy` · `pasteExtend` · `copyFormatted` · `clipboardDelimiter` · `fillSeries` · `cellFlash` |
| 모드          | `serverSide` · `appendScroll` · `treeData` · `pivot` · `groupSubtotals`                                                                                            |
| 메뉴/훅       | `contextMenu` · `headerContextMenu` · `isRowSelectable` · `externalFilter` · `postSort` · `isFullWidthRow` · `animateRows`                                         |
| 내장 UI       | `rowNumbers` · `rowCheckboxes` · `rowStatus` · `statusBar` · `columnController` · `filterToggle` · `groupPanel` · `findBox` · `headerFilters` · `pivotPanel`       |
| 헤더/행 조작  | `resizable` · `reorderable` · `rowResizable` · `rowDragAcceptExternal`                                                                                             |
| 스타일/기타   | `theme` · `density` · `locale` · `loading` · `events`                                                                                                              |

## 커스텀 렌더 채널 (프레임워크별 표현 방식)

기능은 같고 선언 방식만 다르다. 컬럼 단위와 행 단위로 나뉜다.

| 렌더 지점               | React                         | Vue 3 / Vue 2.7        | Svelte 5                | Vanilla `mountGrid`                                |
| ----------------------- | ----------------------------- | ---------------------- | ----------------------- | -------------------------------------------------- |
| 셀                      | `ReactColumnDef.renderCell`   | `#cell-{field}` 슬롯   | `cell` 스니펫 prop      | `ColumnDef.cellRenderer` (Node)                    |
| 에디터                  | `ReactColumnDef.renderEditor` | `#editor-{field}` 슬롯 | `editor` 스니펫 prop    | `ColumnDef.editorRenderer` (Node)                  |
| 헤더                    | `ReactColumnDef.renderHeader` | `#header-{field}` 슬롯 | `header` 스니펫 prop    | `ColumnDef.headerRenderer` (Node)                  |
| 행 상세 (master-detail) | `detailRenderer` prop         | `#detail` 슬롯         | `detail` 스니펫 prop    | `detailRenderer` 옵션                              |
| 전체 너비 행            | `renderFullRow` prop          | `#fullwidth` 슬롯      | `fullwidth` 스니펫 prop | `isFullWidthRow` + `GridOptions.fullWidthRenderer` |
| 상세 행 높이            | `detailHeight` prop           | `detail-height` prop   | `detailHeight` prop     | `detailHeight` 옵션                                |

코어 `ColumnDef`의 `cellRenderer`/`editorRenderer`/`headerRenderer`는 DOM
`Node`를 반환하는 vanilla 전용 채널이다 — 어댑터는 각자 프레임워크 렌더 결과물
(ReactNode/슬롯/스니펫)을 쓰므로 이 필드를 해석하지 않는다.

## vanilla 전용 내장 UI

`mountGrid`는 프레임워크 없이 완성형 그리드를 제공하므로 어댑터에 없는 내장
컨트롤을 옵션으로 연다:

| 옵션              | 동작                        | 어댑터 대응 경로                    |
| ----------------- | --------------------------- | ----------------------------------- |
| `searchBox`       | 상단 전역 검색 입력         | 앱이 직접 입력창 → `grid.setSearch` |
| `pager`           | 하단 페이저/행수            | 앱이 직접 페이저 → `grid.setPage`   |
| `pageSizeOptions` | 페이저의 페이지 크기 셀렉트 | 앱이 직접 셀렉트 → `grid.setPage`   |

## prop 미노출 — `GridOptions` 전용 필드

아래는 어댑터 prop에 없고 `MountOptions`(또는 외부 `grid`/`store` 주입)로만
도달한다. 어댑터에서 필요하면 외부에서 `new GridCore(...)`를 만들어 넘긴다.

`pageSize`(초기 페이지 크기) · `rowClass` · `rowStyle` · `undoLimit` ·
`pinnedTopRows`/`pinnedBottomRows` · `columnGroups` · `editable` ·
`filterRowVisible` · `virtualScroll`(내부 계산) · `features`(슬림/플러그인 구성)

## 타입·값 수출 표면

어댑터 index 재수출 — 4개 어댑터가 동일 집합을 제공한다.

| 수출                                                                                               | React | Vue 3 | Vue 2.7 | Svelte | 용도                      |
| -------------------------------------------------------------------------------------------------- | :---: | :---: | :-----: | :----: | ------------------------- |
| `GridEventHandlers` · `GridEventMap` · `GridListener` · `Unsubscribe`                              |   ✓   |   ✓   |    ✓    |   ✓    | `events`/수동 구독 타입   |
| `ContextMenuItem`                                                                                  |   ✓   |   ✓   |    ✓    |   ✓    | 커스텀 컨텍스트 메뉴 정의 |
| `VirtualScrollState`                                                                               |   ✓   |   ✓   |    ✓    |   ✓    | 가상 스크롤 상태 타입     |
| `defineColumns` (값)                                                                               |   ✓   |   ✓   |    ✓    |   ✓    | 컬럼 배열 타입 추론 헬퍼  |
| `buildPivot` · `matchFilterValue` (값)                                                             |   ✓   |   ✓   |    ✓    |   ✓    | 데이터 가공 유틸          |
| `defaultContextMenuItems` · `defaultHeaderContextMenuItems` · `enLocale` · `gridThemePresets` (값) |   ✓   |   ✓   |    ✓    |   ✓    | 내장 프리셋               |

렌더러 고유 수출:

- **React** — `useGridCore` · `useGridSnapshot` · `UseGridCoreResult` ·
  `ReactColumnDef`(renderCell/renderEditor/renderHeader 확장)
- **Vue 3 / Vue 2.7** — `useGrid` · `useGridState` · `UseGridReturn`
- **Svelte** — `createGridStore` · `toGridStore` · `GridStore` · `CellContext`
- **Vanilla** — `@moda-grid/core`가 직접 수출: `mountGrid` · `MountOptions` ·
  `MountedGrid` · `GridCore` · 전체 `types/` 표면
- **공용 DOM 계약** — `adapterDom.ts`(`pinPartition`/`pinClasses`/
  `pinStyleProps`/`describeCell`/`camelizeStyles`/`stylesToCssText`/
  `resolveRowDrop`/`startColumnResize`/`startRowResize`/
  `startFillDrag`/`resizeKeyDelta`/`trackPointerDrag`/`handleGridKeydown`/
  `editorInputValue`/`cellPosFromPoint`/`dropRatioForY` 등)는 5개 렌더러
  내부 구현이 공유하는 인프라 수출이다 — 어댑터 index가 재수출하지는 않고,
  커스텀 렌더러/확장을 만드는 경우 코어에서 직접 import해 같은 계약을 재사용한다.
  리프 셀의 클래스/스타일/title/병합 span은 `describeCell`이 단일 계산한다 —
  새 셀 표현을 추가할 때는 5개 템플릿이 아니라 여기에 둔다

## 알려진 차이 (갭 / 의도)

| 항목                            | 상태 | 내용                                                                                     |
| ------------------------------- | ---- | ---------------------------------------------------------------------------------------- |
| `className` (React/vanilla만)   | 의도 | Vue/Svelte는 컴포넌트 `class` 속성 상속으로 커버                                         |
| Svelte 외부 그리드 `store` 명명 | 의도 | `grid`가 아니라 반응형 `GridStore` 래퍼를 받는다 — `$store` 구독과 일관                  |
| mountGrid `MountOptions` 유니온 | 의도 | `grid` 지정 시 GridOptions 필드가 optional로 풀린다 — 어댑터처럼 그리드 자체 구성이 우선 |
| `pageSize` 초기값 prop          | 제한 | 어댑터 prop 없음 — 외부 `grid`/`store`의 `GridOptions.pageSize` 또는 `setPage`로 지정    |

## 규칙

- 새 기능(prop/옵션/수출 타입)을 추가하면 이 표를 같은 커밋에 갱신한다.
- 렌더러별 지원이 다르면 "의도"와 "갭"을 구분해 기록한다 — 갭은 구현 후보로
  TODO.md에도 남긴다.
