# 인라인 셀 편집 (Editing) + Undo/Redo

더블클릭·키보드로 셀을 편집하고, 검증 후 저장한다. 편집/붙여넣기 이력은
Undo/Redo로 되돌릴 수 있다.

## 사용자 조작 (DataGrid 기본 동작)

| 조작 | 동작 |
| ---- | ---- |
| 셀 더블클릭 | 편집 진입 |
| 활성 셀에서 Enter / F2 | 기존 값으로 편집 진입 |
| 활성 셀에서 문자 입력 | 입력값으로 교체하며 편집 진입 |
| Enter | 저장 + 아래 셀로 이동 |
| Tab / Shift+Tab | 저장 + 오른쪽/왼쪽 이동 |
| Escape | 취소 (저장 안 함) |
| Ctrl/Cmd+Z | Undo, Ctrl/Cmd+Y·Ctrl/Cmd+Shift+Z | Redo |

편집 중 셀은 `.mg-cell-editing`, 에러는 `.mg-edit-error`로 셀 아래 표시되고
입력 테두리가 빨간색이 된다. 에디터 내부 키는 `stopPropagation`으로
그리드 네비게이션과 분리된다.

## 컬럼 옵션

```ts
{
  field: "age",
  editable: true,                                // 기본값 true
  cellEditor: "number",                          // text|number|select|date|custom
  editorOptions: ["admin", "editor"],            // select 편집기 옵션
  valueSetter: (row, value) => { row.age = +value; },  // 기본: row[field]=value
  validate: (value, row) =>
    Number(value) >= 0 ? true : "0 이상이어야 합니다",  // false/문자열 → 저장 거부
}
```

## 코어 API

```ts
grid.startEditing(rowIndex, columnIndex, initial?); // 편집 진입 (editable:false면 false)
grid.updateEditValue(v);                            // 임시 입력값 갱신 (에러 해제)
grid.commitEditing();                               // 검증 → 저장. 실패 시 false + 편집 유지
grid.cancelEditing();                               // 저장 없이 종료
grid.isEditing(r, c); grid.isCellEditable(r, c);
grid.getEditorContext(r, c);                        // CellEditorContext (커스텀 에디터용)

grid.undo(); grid.redo();                           // 이력 되돌리기/다시 실행
grid.canUndo(); grid.canRedo();
```

스냅샷: `editingCell` / `editValue` / `editError` / `canUndo` / `canRedo`.

## 커밋 순서

`commitEditing()`은:

1. `cellEditor === 'number'`이면 입력을 `Number`로 변환
2. `column.validate(value, row)` — `true`가 아니면 `editError`에 메시지 기록,
   편집 유지, `false` 반환
3. `valueSetter` 호출(없으면 `row[field] = value`) 후 `notify()`로 파이프라인 재계산

## 커스텀 에디터

`cellEditor: 'custom'`이면 `getEditorContext(r, c)`가
`CellEditorContext`(`{ row, column, value, editValue, error, setValue,
commit, cancel }`)를 제공한다. 어댑터별 연결:

| 프레임워크 | 확장점 |
| ---------- | ------ |
| React | `ReactColumnDef.renderEditor(ctx)` |
| Vue / Vue2 | `#editor-{field}` 슬롯 |
| Svelte | `{#snippet editor}` |

예시는 각 플랫폼 가이드 참고.

## Undo/Redo

- `commitEditing()`과 `pasteTsv()`가 이력에 기록된다 — 붙여넣기는 여러 셀이
  바뀌어도 **1개 단위**.
- 되돌리기도 `valueSetter` 경로로 `oldValue`/`newValue`를 재적용한다.
- `GridOptions.undoLimit`(기본값 100)으로 깊이 제한. `0`이면 기록 자체를 끈다.
- `setData` 호출 시 이력은 초기화된다.
- 버튼 UI는 `snapshot.canUndo`/`canRedo`로 disabled 상태를 연동한다.
