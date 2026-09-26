# Svelte 가이드 — `@moda-grid/svelte`

Svelte 5 (runes) 전용.

## 설치

```bash
npm install @moda-grid/svelte   # peer: svelte ^5
```

## 기본 사용

```svelte
<script lang="ts">
  import { createGridStore, DataGrid } from "@moda-grid/svelte";
  import "@moda-grid/svelte/styles.css";      // 필수 — 공통 스타일
  import type { ColumnDef } from "@moda-grid/svelte";

  interface User { id: number; name: string; role: string; age: number; }

  const columns: ColumnDef<User>[] = [
    { field: "id", header: "ID", width: 60 },
    { field: "name", header: "이름", filterable: true },
    { field: "role", header: "역할" },
    { field: "age", header: "나이", filterable: true, filterType: "number" },
  ];

  const store = createGridStore<User>({ columns, data: users });
  const { grid } = store;   // 액션 호출용
  // $store === GridSnapshot<User> (스토어 자동 구독)
</script>

<input placeholder="검색…" oninput={(e) => grid.setSearch(e.currentTarget.value)} />
<button onclick={() => grid.exportToCsv({ filename: "users.csv" })}>CSV보내기</button>

<DataGrid
  {store}
  height={480}
  rowHeight={37}
  selectionMode="multi-cell"
  columnController
  rowCheckboxes
/>
```

## 두 가지 모드

```svelte
<!-- 1) 내부 생성 모드 -->
<DataGrid {columns} data={users} height={480} rowHeight={37} />

<!-- 2) 스토어 모드 — 외부에서 생성한 store -->
<DataGrid {store} height={480} rowHeight={37} />
```

## API

| 함수 | 반환 | 용도 |
| ---- | ---- | ---- |
| `createGridStore(options)` | `GridStore` | GridCore 생성 + `readable` 래핑 |
| `toGridStore(grid)` | `GridStore` | 기존 GridCore를 스토어로 래핑 |

`GridStore` = `Readable<GridSnapshot>` + `{ grid: Grid }`.

- `$store`로 스냅샷 접근 (자동 구독/해제).
- 액션은 항상 `store.grid.*()`로 호출.

Props는 다른 어댑터와 동일: `store`/`columns`/`data`/`height`/`rowHeight`/
`overscan`/`selectable`/`resizable`/`reorderable`/`columnController`/
`filterToggle`/`rowCheckboxes`/`statusBar`/`getRowId`/`className`/`selectionMode`.

## 커스텀 셀 / 에디터 — `{#snippet}`

Svelte 5의 named snippet으로 위임한다:

```svelte
<DataGrid {store} height={480} rowHeight={37}>
  <!-- 모든 컬럼 공용 셀 스니펫 — column.field로 분기 -->
  {#snippet cell({ value, row, column, text })}
    {#if column.field === "role"}
      <span class="badge role-{value}">{value}</span>
    {:else}
      {text}   <!-- formatter 적용 폴백 문자열 -->
    {/if}
  {/snippet}

  <!-- 커스텀 편집기 — ctx: CellEditorContext -->
  {#snippet editor(ctx)}
    <select
      value={String(ctx.editValue ?? ctx.value ?? "")}
      onchange={(e) => ctx.setValue(e.currentTarget.value)}
    >
      <option value="admin">admin</option>
    </select>
  {/snippet}
</DataGrid>
```

- `cell` snippet 컨텍스트(`CellContext<TData>`): `{ value, row, column, text }`.
- snippet이 없으면 `getCellText()`(formatter 적용)로 기본 렌더링.

## 스냅샷으로 보조 UI 만들기

```svelte
<button disabled={$store.pageIndex === 0} onclick={() => grid.setPage($store.pageIndex - 1)}>
  이전
</button>
<span>{$store.filteredRowCount} / {$store.totalRowCount}행</span>
```

## 자주 묻는 것

- **`$store` vs `store.grid`** — 읽기는 `$store`, 쓰기(액션)는 `store.grid`.
- **타입 추론이 안 된다** — `createGridStore<User>`에 제네릭을 명시하고,
  커스텀 컴포넌트 래핑 시 `generics` 속성을 사용할 것.

기능별 상세는 [features/](../features/) 디렉토리 참고.
