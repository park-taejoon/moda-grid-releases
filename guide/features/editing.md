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
  cellEditor: "number",                          // text|number|select|date|checkbox|multiselect|custom
  editorOptions: ["admin", "editor"],            // select/multiselect 편집기 옵션
  // checkbox 편집기 전용:
  checkedValue: "Y", uncheckedValue: "N",        // 체크/해제 시 저장값 (기본 true/false)
  headerCheckbox: true,                          // 헤더에 전체 토글 체크박스 (삼중상태)
  valueSetter: (row, value) => { row.age = +value; },  // 기본: row[field]=value
  validate: (value, row) =>
    Number(value) >= 0 ? true : "0 이상이어야 합니다",  // false/문자열 → 저장 거부
  required: true,                                      // 빈 값 저장 거부 + 헤더 * 표시
}
```

`required`와 `validate`는 편집 저장뿐 아니라 `importCsv`·`pasteTsv`에도
같이 적용된다 — 같은 컬럼 정의가 모든 입력 경로의 검증 규칙이다.
빈 값은 `null`/`undefined`/빈 문자열/`NaN`으로 판정한다.

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

## 체크박스 편집기 (`cellEditor: "checkbox"`)

편집 모드 없이 **클릭 즉시 토글**된다 — 셀에 체크박스가 상시 렌더링된다.

```ts
{ field: "active", cellEditor: "checkbox", checkedValue: "Y", uncheckedValue: "N", headerCheckbox: true }
```

- `checkedValue`/`uncheckedValue`로 저장값 지정 (기본 `true`/`false`,
  `"Y"`/`"N"` 같은 문자열도 가능).
- `headerCheckbox: true`면 헤더에 삼중 상태(전체/일부/없음) 체크박스가
  표시되고, 클릭 시 **필터링된 모든 행**을 일괄 체크/해제한다 — 전체가
  하나의 Undo 단위로 기록된다.
- `validate`/`required` 검증을 통과하지 못하면 토글이 거부된다.

```ts
grid.isCellChecked(row, col);            // 셀의 체크 여부
grid.toggleCellChecked(row, col);        // 즉시 토글 (검증 실패 시 false)
grid.checkboxColumnState(col);           // "all" | "some" | "none"
grid.toggleAllChecked(col);              // 헤더 체크박스 토글과 동일
```

## 다중 선택 편집기 (`cellEditor: "multiselect"`)

`editorOptions` 중 복수 값을 체크박스 목록으로 선택하고 **배열로 저장**한다.
CSV/xlsx보내기는 `;` 구분 문자열로 직렬화되어 왕복이 보장된다
(`"a;b"` → `["a","b"]`).

```ts
{ field: "tags", cellEditor: "multiselect", editorOptions: ["긴급", "버그", "개선"] }
// row.tags = ["긴급", "버그"]
```


## 커밋 순서

`commitEditing()`은:

1. `cellEditor === 'number'`이면 입력을 `Number`로 변환
2. `validateCellValue(row, col, value)` — `required`(빈 값 거부) →
   `column.validate` 순서로 검사. 실패 시 `editError`에 메시지 기록,
   편집 유지, `false` 반환
3. `valueSetter` 호출(없으면 `row[field] = value`) 후 `notify()`로 파이프라인 재계산

## 저장 전 검증 — validateChanges

변경된 행(I/U)만 모아 검사하려면 행 상태 추적 API를 사용한다
([row-state.md](./row-state.md) 참고). 변경 행에서 검증에 실패한 셀은
`mg-cell-invalid` 클래스 + title 툴팁으로 표시된다:

```ts
const errors = grid.validateChanges(); // [{ row, rowId, field, message }]
if (errors.length) return;             // 서버 전송 중단
```

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

## 읽기 전용 셀 표시

`editable: false`이거나 `formula`가 있는 셀은 어댑터가 자동으로
`mg-cell-readonly` 클래스를 부여해 연한 배경으로 구분한다.
`--grid-readonly-bg` / `--grid-readonly-color` CSS 변수로 색을 바꿀 수 있다.
