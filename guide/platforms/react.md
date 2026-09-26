# React 가이드 — `@moda-grid/react`

## 설치

```bash
npm install @moda-grid/react   # peer: react, react-dom
```

> 모노레포 소스에서 사용 시: `"@moda-grid/react": "workspace:*"` —
> exports가 소스를 가리켜 빌드 없이 HMR로 동작한다.

## 기본 사용

```tsx
import { DataGrid, useGridCore } from "@moda-grid/react";
import "@moda-grid/react/styles.css";   // 필수 — 공통 스타일
import type { ReactColumnDef } from "@moda-grid/react";

interface User { id: number; name: string; role: string; age: number; }

const columns: ReactColumnDef<User>[] = [
  { field: "id", header: "ID", width: 60 },
  { field: "name", header: "이름", filterable: true },
  {
    field: "role",
    header: "역할",
    renderCell: (value) => (
      <span className={`badge role-${value}`}>{String(value)}</span>
    ),
  },
  { field: "age", header: "나이", filterable: true, filterType: "number" },
];

function App({ users }: { users: User[] }) {
  const { grid, snapshot } = useGridCore({ columns, data: users });

  return (
    <>
      <input
        placeholder="검색…"
        onChange={(e) => grid.setSearch(e.target.value)}
      />
      <button onClick={() => grid.exportToCsv({ filename: "users.csv" })}>
        CSV보내기
      </button>
      <DataGrid
        grid={grid}
        columns={columns}
        height={480}
        rowHeight={37}
        selectionMode="multi-cell"
        columnController
        rowCheckboxes
      />
    </>
  );
}
```

## DataGrid 두 가지 모드

```tsx
// 1) 내부 인스턴스 모드 — GridCore를 컴포넌트가 생성/관리
<DataGrid columns={columns} data={users} height={480} rowHeight={37} />

// 2) 제어 모드 — 툴바/버튼에서 grid API를 써야 할 때
const { grid, snapshot } = useGridCore({ columns, data: users });
<DataGrid grid={grid} columns={columns} height={480} rowHeight={37} />
```

`serverSide`, `treeData` 옵션은 **내부 생성 모드에서만** prop으로 받는다.
제어 모드에서는 `useGridCore({ ..., serverSide })` 옵션으로 넘긴다.

## Props

| prop | 타입 | 기본값 | 설명 |
| ---- | ---- | ------ | ---- |
| `columns` | `ReactColumnDef[]` | 필수 | 컬럼 정의 (`renderCell`/`renderEditor` 확장) |
| `data` | `TData[]` | — | 행 데이터 (`grid` 제어 모드면 생략 가능) |
| `grid` | `Grid<TData>` | — | 외부 GridCore. 생략 시 내부 생성 |
| `height` / `rowHeight` | `number` | — | 가상 스크롤 (함께 지정) |
| `overscan` | `number` | `5` | 가상 스크롤 여유분 |
| `resizable` | `boolean` | `true` | 컬럼 리사이즈 핸들 |
| `reorderable` | `boolean` | `true` | 헤더 DnD 재배치 |
| `selectable` | `boolean` | `true` | 행 클릭 선택 |
| `selectionMode` | `SelectionMode` | `single-cell` | `single-cell`/`multi-cell`/`row` |
| `serverSide` | `ServerSideOptions` | — | 무한 스크롤 데이터 소스 |
| `treeData` | `TreeDataOptions` | — | 계층 데이터 |
| `columnController` | `boolean` | `false` | 우상단 컬럼 관리 팝오버 |
| `filterToggle` | `boolean` | `false` | 우상단 필터 행 표시/숨김 버튼 |
| `rowCheckboxes` | `boolean` | `false` | 행 체크박스 + 헤더 전체선택 |
| `statusBar` | `boolean` | `true` | 선택 범위 집계 상태바 |
| `getRowId` | `(row) => string` | — | 행 ID 함수 |
| `className` | `string` | — | 추가 클래스 |

## Hooks

| 훅 | 반환 | 용도 |
| -- | ---- | ---- |
| `useGridCore(options)` | `{ grid, snapshot }` | GridCore 생성 + 스냅샷 구독. `options.data`/`columns` 참조 변경 시 자동 반영 |
| `useGridSnapshot(grid)` | `GridSnapshot` | 외부에서 만든 그리드만 구독 |

## 커스텀 셀 / 에디터

```tsx
const columns: ReactColumnDef<User>[] = [
  {
    field: "role",
    header: "역할",
    // 커스텀 셀 렌더러
    renderCell: (value, row, col) => <b>{String(value)}</b>,
    // 커스텀 편집기 — ctx: CellEditorContext
    cellEditor: "custom",
    renderEditor: (ctx) => (
      <select
        value={String(ctx.editValue ?? ctx.value)}
        onChange={(e) => ctx.setValue(e.target.value)}
        onKeyDown={(e) => e.stopPropagation()}
      >
        <option value="admin">admin</option>
        <option value="editor">editor</option>
      </select>
    ),
  },
];
```

`renderCell`/`renderEditor`가 없으면 `grid.getCellText()`(formatter 적용)와
내장 에디터(`text`/`number`/`select`/`date`)가 사용된다.

## 스냅샷으로 보조 UI 만들기

```tsx
const { grid, snapshot } = useGridCore({ columns, data });
// snapshot.rows, snapshot.pageIndex/pageCount, snapshot.selectedRowIds,
// snapshot.selectionAggregates, snapshot.serverSide.loading …

<button
  disabled={snapshot.pageIndex === 0}
  onClick={() => grid.setPage(snapshot.pageIndex - 1)}
>
  이전
</button>
```

## 자주 묻는 것

- **리렌더가 안 된다** — `grid` 메서드 호출 후 스냅샷을 `useGridSnapshot`/
  `useGridCore`로 읽고 있는지 확인. 스냅샷은 메모이즈되므로 같은 참조면
  리렌더되지 않는다(정상 동작).
- **데이터가 안 바뀐다** — `useGridCore`는 `options.data` **참조** 변경을
  감지한다. 같은 배열을 mutate하면 `grid.setData(newArray)`를 직접 호출할 것.
- **StrictMode** — `useSyncExternalStore` 기반이라 이중 마운트에 안전하다.

기능별 상세는 [features/](../features/) 디렉토리, 코어 API 전체는
[02-core-api.md](../../02-core-api.md) 참고.
