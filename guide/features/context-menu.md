# 컨텍스트 메뉴 (셀 우클릭)

셀 우클릭 시 커스텀 메뉴를 표시한다 — IBSheet 컨텍스트 메뉴에 해당한다.

## 설정

```ts
const grid = new GridCore({
  columns,
  data,
  contextMenu: [
    { id: "copy", label: "복사", onClick: (ctx) => navigator.clipboard.writeText(String(ctx.value ?? "")) },
    { id: "sep1", label: "", separator: true },
    {
      id: "delete",
      label: (ctx) => `${ctx.row.name} 삭제`,
      disabled: (ctx) => ctx.row.id === lockedId,
      onClick: (ctx) => grid.deleteRows([ctx.rowIndex]),
    },
    { id: "admin-only", label: "권한 변경", visible: (ctx) => isAdmin(ctx.row) },
  ],
});
```

어댑터는 `<DataGrid contextMenu={items} />` prop으로 전달한다 (내부
GridCore 생성 모드에서만 적용 — 제어 모드는 `new GridCore({ ..., contextMenu })`).

## ContextMenuItem 필드

| 필드 | 타입 | 설명 |
| ---- | ---- | ---- |
| `id` | `string` | 고유 식별자 |
| `label` | `string \| (ctx) => string` | 메뉴 라벨 — 함수면 행/셀 컨텍스트로 동적 생성 |
| `onClick` | `(ctx) => void` | 클릭 핸들러 |
| `visible` | `(ctx) => boolean` | false면 해당 셀에서 항목 숨김 |
| `disabled` | `(ctx) => boolean` | true면 비활성 렌더링 |
| `separator` | `boolean` | true면 구분선 (label/onClick 무시) |

`ctx`(ContextMenuContext)에는 `row`, `rowIndex`(visibleData 기준),
`column`, `value`가 들어 있다.

## 코어 API

```ts
grid.openContextMenu(x, y, rowIndex, columnKey); // 표시 항목 없으면 false
grid.closeContextMenu();
grid.runContextMenuItem(id);                     // 실행 후 자동으로 닫힘
```

- `snapshot.contextMenu`가 `{ x, y, ctx, items }` 또는 `null` — 어댑터가
  viewport 좌표에 `role="menu"` 팝업을 렌더링한다.
- `openContextMenu`가 `false`를 반환하면(옵션 없음·행 없음·모든 항목 숨김)
  어댑터는 `preventDefault`를 하지 않아 브라우저 기본 메뉴가 열린다.
- 오버레이 클릭/우클릭 시 `closeContextMenu`로 닫힌다.
