# Vue 3 가이드 — `@moda-grid/vue`

Vue >= 3.3 지원. `<script setup>` Composition API 기반.

## 설치

```bash
npm install @moda-grid/vue   # peer: vue >= 3.3
```

## 기본 사용

```vue
<script setup lang="ts">
import { DataGrid, useGrid } from "@moda-grid/vue";
import "@moda-grid/vue/styles.css";       // 필수 — 공통 스타일
import type { ColumnDef } from "@moda-grid/vue";

interface User { id: number; name: string; role: string; age: number; }

const columns: ColumnDef<User>[] = [
  { field: "id", header: "ID", width: 60 },
  { field: "name", header: "이름", filterable: true },
  { field: "role", header: "역할" },
  { field: "age", header: "나이", filterable: true, filterType: "number" },
];

const { grid, state } = useGrid<User>({ columns, data: users });
// state: ShallowRef<GridSnapshot<User>> — 템플릿에서 state.xxx 로 접근
</script>

<template>
  <input placeholder="검색…" @input="grid.setSearch($event.target.value)" />
  <button @click="grid.exportToCsv({ filename: 'users.csv' })">CSV보내기</button>

  <DataGrid
    :grid="grid"
    :columns="columns"
    :height="480"
    :row-height="37"
    selection-mode="multi-cell"
    column-controller
    row-checkboxes
  >
    <template #cell-role="{ value }">
      <span :class="`badge role-${value}`">{{ value }}</span>
    </template>
  </DataGrid>
</template>
```

## 두 가지 모드

```vue
<!-- 1) 내부 생성 모드 -->
<DataGrid :columns="columns" :data="users" :height="480" :row-height="37" />

<!-- 2) 제어 모드 — 외부 grid 인스턴스 -->
<DataGrid :grid="grid" :columns="columns" :height="480" :row-height="37" />
```

`serverSide`/`treeData`는 내부 생성 모드 prop 또는 `useGrid` 옵션으로 전달한다.

## Props

React 어댑터와 동일하다 (kebab-case로 전달): `columns` `data` `grid`
`height` `row-height` `overscan` `resizable` `reorderable` `selectable`
`selection-mode` `server-side` `tree-data` `column-controller`
`row-checkboxes` `status-bar` `get-row-id` `class-name`.
전체 목록: [react.md의 props 표](./react.md#props)와 동일.

## Composables

| 함수 | 반환 | 용도 |
| ---- | ---- | ---- |
| `useGrid(options)` | `{ grid, state, unsubscribe }` | GridCore 생성 + 스냅샷 `ShallowRef` |
| `useGridState(grid)` | `{ state, unsubscribe }` | 외부 GridCore의 스냅샷만 구독 |

- 컴포넌트/이펙트 스코프 안에서 호출하면 `onScopeDispose`로 자동 구독 해제.
- `state`는 `ShallowRef` — 스냅샷이 통째로 교체되는 구조라 deep 반응형이
  불필요하다. 읽기 전용으로 취급할 것.

## 커스텀 셀 / 에디터 슬롯

각 `<td>`는 `cell-{field}` 동적 슬롯, 각 에디터는 `editor-{field}` 슬롯을
노출한다:

```vue
<DataGrid :grid="grid" :columns="columns">
  <template #cell-role="{ value, row, column }">
    <span :class="`badge role-${value}`">{{ value }}</span>
  </template>
  <template #cell-email="{ value }">
    <a :href="`mailto:${value}`">{{ value }}</a>
  </template>

  <!-- 커스텀 편집기 — ctx: CellEditorContext -->
  <template #editor-role="ctx">
    <select :value="ctx.value" @change="ctx.setValue($event.target.value)">
      <option value="admin">admin</option>
    </select>
  </template>
</DataGrid>
```

- `#cell-{field}` 슬롯 props: `{ value, row, column }` — `value`는 원시 값.
- 슬롯이 없는 컬럼은 `getCellText()`(formatter 적용)로 렌더링한다.

## 스냅샷으로 보조 UI 만들기

```vue
<button :disabled="state.pageIndex === 0" @click="grid.setPage(state.pageIndex - 1)">
  이전
</button>
<span>{{ state.filteredRowCount }} / {{ state.totalRowCount }}행</span>
```

## 자주 묻는 것

- **props가 안 먹는다** — kebab-case로 전달했는지 확인 (`:row-height`,
  `selection-mode`).
- **`state`를 바꾸고 싶다** — 하지 말 것. 변경은 항상 `grid.*()` 액션으로.
- **외부에서 만든 그리드** — `useGridState(grid)`로 스냅샷만 구독.

기능별 상세는 [features/](../features/) 디렉토리 참고.
