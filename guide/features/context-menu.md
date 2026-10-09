# 컨텍스트 메뉴 (셀 우클릭)

셀 우클릭 시 커스텀 메뉴를 표시한다 — 타사 그리드 컨텍스트 메뉴에 해당한다.

## 설정

```ts
const grid = new GridCore({
  columns,
  data,
  contextMenu: [
    {
      id: "copy",
      label: "복사",
      onClick: (ctx) => navigator.clipboard.writeText(String(ctx.value ?? "")),
    },
    { id: "sep1", label: "", separator: true },
    {
      id: "delete",
      label: (ctx) => `${ctx.row.name} 삭제`,
      disabled: (ctx) => ctx.row.id === lockedId,
      onClick: (ctx) => grid.deleteRows([ctx.rowIndex]),
    },
    {
      id: "admin-only",
      label: "권한 변경",
      visible: (ctx) => isAdmin(ctx.row),
    },
  ],
});
```

어댑터는 `<DataGrid contextMenu={items} />` prop으로 전달한다. 외부
`grid` prop을 쓰는 제어 모드에서도 prop 변경 시 `grid.setContextMenu()`/
`setHeaderContextMenu()`로 동기화되므로 어느 모드든 동일하게 동작한다
(`null`을 넘기면 메뉴를 끈다).

### 팩토리 함수 형태

`contextMenu`/`headerContextMenu`에는 **배열 대신 `(grid) => items` 함수**도
넣을 수 있다. 메뉴를 열 때마다 그리드 인스턴스를 넘겨 호출하므로, 인스턴스가
필요한 기본 프리셋과 섞어 쓰기에 적합하다:

```ts
contextMenu: (grid) => [
  ...defaultContextMenuItems(grid),
  { id: "my", label: "내 메뉴", onClick: (ctx) => ... },
],
```

## 기본 메뉴 프리셋 — `defaultContextMenuItems`

타사 그리드 기본 우클릭 메뉴에 해당하는 항목을 코어가 제공한다:

```ts
import {
  defaultContextMenuItems,
  defaultHeaderContextMenuItems,
} from "@moda-grid/core";

new GridCore({
  columns,
  data,
  contextMenu: defaultContextMenuItems, // 함수를 그대로 넘겨도 됨
  headerContextMenu: defaultHeaderContextMenuItems,
});
```

`defaultContextMenuItems(grid)`가 반환하는 항목:

| id                            | 동작                                                                         |
| ----------------------------- | ---------------------------------------------------------------------------- |
| `undo` / `redo`               | 실행 취소/다시 실행 — 이력 없으면 disabled                                   |
| `copy`                        | 선택 영역/활성 셀을 TSV로 클립보드에 복사                                    |
| `copyHeaders`                 | 헤더 행 포함 복사                                                            |
| `paste`                       | 클립보드 TSV를 우클릭한 셀에 붙여넣기                                        |
| `insertAbove` / `insertBelow` | 우클릭한 행 위/아래에 새 행 삽입 (`addRows` + I 마킹)                        |
| `clear`                       | 선택 영역 내용 지우기 (`clearRange`)                                         |
| `duplicateRow`                | 선택 행(없으면 우클릭 행) 복제 — `duplicateRowsByIds`, 원본 뒤에 I 마킹 삽입 |
| `deleteRow`                   | 선택 행 삭제 — 다중 선택이면 라벨이 "선택한 N행 삭제"                        |
| `hideRow`                     | 선택 행(없으면 우클릭 행) 숨기기 — `setRowHidden`                            |
| `unhideRows`                  | 숨긴 행 전체 표시 — 숨긴 행이 있을 때만 표시됨                               |
| `exportCsv`                   | `exportToCsv()` 실행                                                         |
| `print`                       | `grid.print()` — 브라우저 인쇄                                               |

`defaultHeaderContextMenuItems(grid)`는 컬럼 숨기기·너비 자동 조정·
그룹화/해제·오름차순/내림차순/정렬 해제·모든 필터 지우기·컬럼 고정/해제
항목을 반환한다.

### 행 삽입 기본값 — `newRow` 옵션

`insertAbove`/`insertBelow`는 기본으로 빈 객체 `{}`를 넣는다.
`externalFilter`나 필수 컬럼이 있으면 빈 행이 필터에 걸려 삽입이
보이지 않을 수 있으므로, 두 번째 인자의 `newRow` 팩토리로 기본값을
지정한다:

```ts
contextMenu: (grid) =>
  defaultContextMenuItems<User>(grid, {
    newRow: () => ({ id: nextId++, name: "", age: 30, active: true }),
  }),
```

> 우클릭하면 코어가 해당 셀을 **활성 셀로 자동 지정**하므로 복사/지우기는
> 우클릭한 셀을 기준으로 동작한다.

## ContextMenuItem 필드

| 필드        | 타입                        | 설명                                          |
| ----------- | --------------------------- | --------------------------------------------- |
| `id`        | `string`                    | 고유 식별자                                   |
| `label`     | `string \| (ctx) => string` | 메뉴 라벨 — 함수면 행/셀 컨텍스트로 동적 생성 |
| `onClick`   | `(ctx) => void`             | 클릭 핸들러                                   |
| `visible`   | `(ctx) => boolean`          | false면 해당 셀에서 항목 숨김                 |
| `disabled`  | `(ctx) => boolean`          | true면 비활성 렌더링                          |
| `separator` | `boolean`                   | true면 구분선 (label/onClick 무시)            |

`ctx`(ContextMenuContext)에는 `row`, `rowIndex`(visibleData 기준),
`column`, `value`가 들어 있다.

## 코어 API

```ts
grid.openContextMenu(x, y, rowIndex, columnKey); // 표시 항목 없으면 false
grid.closeContextMenu();
grid.runContextMenuItem(id); // 실행 후 자동으로 닫힘
```

- `snapshot.contextMenu`가 `{ x, y, ctx, items }` 또는 `null` — 어댑터가
  viewport 좌표에 `role="menu"` 팝업을 렌더링한다.
- `openContextMenu`가 `false`를 반환하면(옵션 없음·행 없음·모든 항목 숨김)
  어댑터는 `preventDefault`를 하지 않아 브라우저 기본 메뉴가 열린다.
- 오버레이 클릭/우클릭 시 `closeContextMenu`로 닫힌다.

## 헤더 우클릭 메뉴

`headerContextMenu` 옵션으로 헤더 전용 메뉴를 정의할 수 있다 — `ctx.row`는
`null`, `ctx.rowIndex`는 `-1`이다. 숨기기/정렬/너비 자동조정 같은 컬럼
조작 메뉴에 적합하다. 자세한 예시는 [events.md](./events.md) 참고.
