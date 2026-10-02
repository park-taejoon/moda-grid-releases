# 인라인 셀 편집 (Editing) + Undo/Redo

더블클릭·키보드로 셀을 편집하고, 검증 후 저장한다. 편집/붙여넣기 이력은
Undo/Redo로 되돌릴 수 있다.

## 사용자 조작 (DataGrid 기본 동작)

| 조작                   | 동작                                                           |
| ---------------------- | -------------------------------------------------------------- |
| 셀 더블클릭            | 편집 진입 (`singleClickEdit: true`면 한 번 클릭)               |
| 활성 셀에서 Enter / F2 | 기존 값으로 편집 진입                                          |
| 활성 셀에서 문자 입력  | 입력값으로 교체하며 편집 진입                                  |
| Enter                  | 저장 + 아래 셀로 이동                                          |
| Tab / Shift+Tab        | 저장 + 오른쪽/왼쪽 이동 — 행 끝/처음에서 다음/이전 행으로 wrap |
| Escape                 | 취소 (저장 안 함)                                              |
| Ctrl/Cmd+Z             | Undo, Ctrl/Cmd+Y·Ctrl/Cmd+Shift+Z                              | Redo |

편집 중 셀은 `.mg-cell-editing`, 에러는 `.mg-edit-error`로 셀 아래 표시되고
입력 테두리가 빨간색이 된다. 에디터 내부 키는 `stopPropagation`으로
그리드 네비게이션과 분리된다.

저장 후 이동(Enter/Tab)이 성공하면 그리드 스크롤 컨테이너로 포커스가
복귀된다 — 편집기가 DOM에서 제거되면서 포커스가 `body`로 빠져 이후
방향키/Enter가 먹지 않는 문제를 막는다. 저장 실패(검증 오류) 시에는
편집 상태와 위치를 유지한다.

피커 계열 편집기(`select`·`date`·`radio`·`checkbox`)는 값을 고르는
순간 **즉시 저장되고 에디터가 닫힌다** — 다른 셀 클릭(blur)을 기다리지
않는다. `multiselect`(연속 선택 필요)·`text`·`number`·`textarea`는
Enter/Tab/blur로 저장한다. `editType:"fullRow"`에서는 모든 편집기가
보류 값만 갱신하고 행 커밋을 기다린다.

## 싱글클릭 편집 (`singleClickEdit`)

`singleClickEdit: true`를 켜면 셀 클릭 한 번으로 편집 모드에 들어간다
(타사 그리드 `singleClickEdit` 대응). 런타임에는 `grid.setSingleClickEdit(bool)`.

- `editable: false`·`editable` 함수가 거부한 셀은 클릭해도 진입하지 않는다.
- `cellType`(button/link/image/checkbox/progress/html) 셀은 자체 클릭
  동작이 있어 대상에서 제외된다. `cellEditor: "checkbox"`는 클릭 시
  즉시 토글하는 기존 동작을 따른다.
- 드래그 선택(click 억제)·Ctrl+드래그 복수 범위 선택과 공존한다 —
  드래그로 끝난 클릭은 편집을 시작하지 않는다.

## 행 전체 편집 (`editType: "fullRow"`)

기본 편집은 셀 단위(`"cell"`)다. `editType: "fullRow"`를 켜면 편집 진입
시 **행의 모든 편집 가능 셀이 동시에 에디터를 표시**한다 (타사 그리드
`editType:"fullRow"` 대응):

```ts
new GridCore({ columns, data, editType: "fullRow" });
grid.setEditType("fullRow"); // 런타임 전환 — 편집 중이면 취소 후 전환
```

| 항목    | 동작                                                                                                       |
| ------- | ---------------------------------------------------------------------------------------------------------- |
| 진입    | 더블클릭/F2/타이핑/`startEditing` — 행 전체가 편집 모드                                                    |
| 탭 이동 | Tab/Shift+Tab = **행 안** 다음·이전 편집 셀로 포커스 이동. 끝에서 커밋 후 이동                             |
| 커밋    | Enter·행 밖 클릭 — 행의 모든 보류 값을 **한 Undo 단위**로 적용                                             |
| 검증    | 셀 하나라도 실패하면 아무것도 쓰지 않고 첫 오류 셀로 포커스                                                |
| 취소    | Escape — 행의 모든 보류 값 폐기                                                                            |
| 제외    | `editable:false`·`editable` 함수 거부·수식 컬럼은 일반 셀로 표시. `cellEditor:"checkbox"`는 즉시 토글 유지 |

- 스냅샷: `editingValues`(field→보류 값 맵)·`editingErrors`(field→에러),
  `editingCell`은 행 안 포커스된 셀을 가리킨다.
- 셀별 에러는 해당 셀 에디터에 `mg-editor-error`/`mg-edit-error`로 표시된다.
- `getEditorContext`의 `ctx.fullRow`·`ctx.focused`로 커스텀 에디터가
  행 편집 중인지와 포커스 대상인지 구분할 수 있다.

## 컬럼 옵션

```ts
{
  field: "age",
  editable: true,                                // 기본값 true — 함수도 가능 (아래 참고)
  cellEditor: "number",                          // text|number|select|date|checkbox|multiselect|radio|textarea|custom
  editorOptions: ["admin", "editor"],            // select/multiselect 편집기 옵션
  // checkbox 편집기 전용:
  checkedValue: "Y", uncheckedValue: "N",        // 체크/해제 시 저장값 (기본 true/false)
  headerCheckbox: true,                          // 헤더에 전체 토글 체크박스 (삼중상태)
  valueSetter: (row, value) => { row.age = +value; },  // 기본: row[field]=value
  valueParser: (text) => Number(String(text).replace(/,/g, "")),  // 입력 문자열 → 저장값
  validate: (value, row) =>
    Number(value) >= 0 ? true : "0 이상이어야 합니다",  // false/문자열 → 저장 거부
  required: true,                                      // 빈 값 저장 거부 + 헤더 * 표시
}
```

`required`와 `validate`는 편집 저장뿐 아니라 `importCsv`·`pasteTsv`에도
같이 적용된다 — 같은 컬럼 정의가 모든 입력 경로의 검증 규칙이다.
빈 값은 `null`/`undefined`/빈 문자열/`NaN`으로 판정한다.

### editable 함수 — 행 조건부 편집

`editable`에 함수를 주면 **행 단위로 편집 가능 여부**를 결정한다
(타사 그리드 editable 콜백 대응):

```ts
{
  field: "role",
  editable: (row) => row.role !== "admin", // admin 행은 읽기 전용
}
```

- 편집 진입·붙여넣기·채우기·범위 이동·지우기·`setCellValue`·`updateRow`·
  `replaceAll`·체크박스 토글 등 **모든 쓰기 경로**에 적용된다
- false인 셀은 `mg-cell-readonly` 스타일이 붙고 체크박스는 disabled로 렌더된다
- 행 데이터를 바꾸면(예: 편집으로 role이 admin이 됨) 다음 렌더에서
  자동으로 읽기 전용으로 전환된다

### valueParser — 입력 문자열 변환

`valueParser`는 편집 커밋·`pasteTsv` 붙여넣기·`importCsv`/xlsx 가져오기에서
**입력 문자열을 저장값으로 변환**한다 (타사 그리드 valueParser 대응):

```ts
{
  field: "amount",
  cellEditor: "number",
  valueParser: (text) => Number(String(text).replace(/,/g, "")), // "1,234" → 1234
}
```

- 정의하면 내장 number/체크박스 변환보다 **우선**한다 — 파서가 반환한 값이 그대로 저장 경로로 간다
- 변환된 값이 `valueSetter`·`validate`에 전달된다 — 검증은 변환 후 값 기준
- select·checkbox 등 문자열이 아닌 값이 커밋되는 편집기에는 적용되지 않는다

## 코어 API

```ts
grid.startEditing(rowIndex, columnIndex, initial?); // 편집 진입 (editable:false면 false)
grid.updateEditValue(v);                            // 임시 입력값 갱신 (에러 해제)
grid.commitEditing();                               // 검증 → 저장. 실패 시 false + 편집 유지
grid.cancelEditing();                               // 저장 없이 종료
grid.isEditing(r, c); grid.isCellEditable(r, c);
grid.getEditorContext(r, c);                        // CellEditorContext (커스텀 에디터용)

grid.undo(); grid.redo();                           // 이력 되돌리기/다시 실행

// 전역 편집 잠금 (타사 그리드 Editable 대응)
new GridCore({ columns, data, editable: false });    // 전체 편집 불가
grid.setEditable(false);                             // 런타임 잠금 (편집 중이면 취소)
grid.isEditable();
grid.canUndo(); grid.canRedo();
```

스냅샷: `editingCell` / `editValue` / `editError` / `canUndo` / `canRedo`.

## 셀 변경 플래시 (`cellFlash`)

실시간 갱신 UI처럼 **외부에서 값이 바뀐 셀을 잠깐 강조**하고 싶을 때 쓴다:

```ts
new GridCore({ columns, data, cellFlash: true });
grid.setCellFlash(true); // 런타임 토글
```

- `setCellValue`/`updateRow`/`applyTransaction`/붙여넣기/채우기/바꾸기 등
  `dataChange` source가 `"edit"`이 아닌 변경에 적용된다 — 사용자가 직접
  칸에 치는 편집은 플래시하지 않는다(이미 시각적으로 명확하므로).
- 바뀐 셀에 `.mg-cell-flash` 클래스가 **600ms** 동안 붙고 자동 해제된다.
  연속 변경이 들어오면 마지막 변경 기준으로 타이머가 리셋된다.
- 스냅샷 `flashedCells`(ReadonlySet — `"rowId|field"` 키)로 커스텀 렌더러가
  직접 소비할 수도 있다. 색상은 `--grid-accent`를 따라간다.
- 어댑터: React/Vue/Svelte `cellFlash` prop, vanilla은 `MountOptions.cellFlash`.

## 체크박스 편집기 (`cellEditor: "checkbox"`)

편집 모드 없이 **클릭 즉시 토글**된다 — 셀에 체크박스가 상시 렌더링된다.

```ts
{ field: "active", cellEditor: "checkbox", checkedValue: "Y", uncheckedValue: "N", headerCheckbox: true }
```

- `checkedValue`/`uncheckedValue`로 저장값 지정 (기본 `true`/`false`,
  `"Y"`/`"N"` 같은 문자열도 가능).
- `headerCheckbox: true`면 헤더에 삼중 상태(전체/일부/없음) 체크박스가
  표시되고, 클릭 시 **필터링된 모든 편집 가능 행**을 일괄 체크/해제한다 —
  `editable: false`/`editable(row)`가 거부하는 행은 건너뛰며, 전체가 하나의
  Undo 단위로 기록된다. 삼중 상태도 편집 가능 행 기준으로 계산되므로
  읽기 전용 행만 미체크여도 헤더가 `일부` 상태로 멈추지 않는다.
  편집 가능 행이 하나도 없으면 헤더 체크박스는 `disabled`로 렌더링된다.
- `validate`/`required` 검증을 통과하지 못하면 토글이 거부된다.

```ts
grid.isCellChecked(row, col); // 셀의 체크 여부
grid.toggleCellChecked(row, col); // 즉시 토글 (검증 실패 시 false)
grid.setCellChecked(row, col, true); // 명시적 설정 (토글 아님)
grid.checkboxColumnState(col); // "all" | "some" | "none" — 편집 가능 행 기준
grid.isCheckboxColumnEditable(col); // 토글 가능한 편집 가능 행이 있는지
grid.toggleAllChecked(col); // 헤더 체크박스 토글과 동일

// 행 ID / 데이터 기준 API — 인덱스와 무관
grid.getCheckedRows(); // 체크된 행 데이터[] (첫 checkbox 컬럼 기준)
grid.getCheckedRows("done"); // 지정 필드 기준
grid.setRowChecked("42", true); // ID로 체크 설정 — checkedValue 자동 매핑
grid.setRowChecked("42", false, "done"); // 필드 지정 + 해제
```

- `getCheckedRows(field?)` — 체크박스 컬럼 값이 `checkedValue`인 행을
  반환한다. **행 선택(`selection`)과는 별개 개념**이다. `field` 생략 시 첫
  `cellEditor: "checkbox"` 컬럼. 숨김·필터된 행도 포함한다.
- `setRowChecked(id, checked, field?)` — `toggleCellChecked`의 결정적 버전.
  이미 같은 상태면 성공(true)으로 처리하고 이력을 남기지 않는다.
  field 지정 시 checkbox 에디터이거나 `checkedValue`/`uncheckedValue`가
  선언된 컬럼만 허용한다.

## 다중 선택 편집기 (`cellEditor: "multiselect"`)

`editorOptions` 중 복수 값을 체크박스 목록으로 선택하고 **배열로 저장**한다.
CSV/xlsx보내기는 `;` 구분 문자열로 직렬화되어 왕복이 보장된다
(`"a;b"` → `["a","b"]`).

```ts
{ field: "tags", cellEditor: "multiselect", editorOptions: ["긴급", "버그", "개선"] }
// row.tags = ["긴급", "버그"]
```

## 라디오 편집기 (`cellEditor: "radio"`)

`editorOptions`를 라디오 버튼 그룹으로 렌더링한다 — 옵션이 적을 때
select보다 한 클릭 빠르다 (타사 그리드 `Type:"Radio"` 대응). 선택 즉시
커밋되며, 이미 선택된 옵션을 다시 눌러도 행 상태가 `U`로 변하지 않는다.

```ts
{ field: "status", cellEditor: "radio", editorOptions: ["대기", "진행", "완료"] }
```

## 텍스트에어리어 + 멀티라인 (`cellEditor: "textarea"`)

여러 줄 텍스트 편집 — `multiLine: true`를 함께 주면 저장된 `\n`이 셀에도
줄바꿈으로 표시된다 (`white-space: pre-line`).

```ts
{ field: "memo", cellEditor: "textarea", multiLine: true }
```

키 조작: `Enter` = 줄바꿈, `Ctrl/Cmd+Enter` = 저장, `Tab` = 저장+이동,
`Esc` = 취소. React/Vue/Vue2/Svelte/`mountGrid` 모두 동일.

### 자동 행 높이 (`autoRowHeight`)

`multiLine` 셀의 줄 수에 맞춰 행 높이를 자동 확장한다:

```tsx
<DataGrid autoRowHeight columns={[{ field: "memo", multiLine: true }]} />
```

- 코어 `grid.getRowHeight(row)`가 컬럼 너비로 줄 수를 추정해 필요 높이를
  반환한다 — 어댑터가 `tr` 높이로 적용.
- `GridOptions.autoRowHeight` 또는 어댑터 `autoRowHeight` prop으로 설정,
  `grid.setAutoRowHeight(bool)`로 토글 가능.
- **가상 스크롤 모드에서는 무시**된다 — 가상 스크롤은 고정 행 높이가
  필요하기 때문.
- 줄 수는 추정치다 (글자당 ~8px 기준). 정확한 측정이 필요하면 직접 행
  높이를 계산해 CSS로 적용하거나 `setRowHeight`를 사용한다.

### 행별 높이 (`setRowHeight`)

특정 행만 높이를 지정한다 (타사 그리드 `setRowHeight` 대응):

```ts
grid.setRowHeight(grid.getRowId(row), 80); // px
grid.setRowHeight(id, null); // 해제 → 기본/autoRowHeight 계산
```

`getRowHeight`는 오버라이드를 `autoRowHeight` 추정보다 **우선 적용**한다.
어댑터/`mountGrid`는 `tr` 높이로 자동 반영. 가상 스크롤은 고정 높이를
쓰므로 비가상 모드에서만 적용된다.

### 행 높이 드래그 (`rowResizable`)

`rowResizable` prop/`mountGrid` 옵션을 주면 **행 번호 셀 하단**에 드래그
핸들(`.mg-row-resizer`)이 생긴다 — 엑셀/타사 그리드의 행 경계 드래그처럼
끌어서 행 높이를 조정한다 (`setRowHeight` 호출). `rowNumbers`가 필요하며
가상 스크롤에서는 표시되지 않는다:

```tsx
<DataGrid rowNumbers rowResizable ... />
mountGrid(el, { rowNumbers: true, rowResizable: true, ... });
```

## 텍스트 자동완성 (`editorOptions` + text 에디터)

기본 `text` 에디터에 `editorOptions`를 주면 `<datalist>` 자동완성이
연결된다 — 타사 그리드 Suggestion/ComboEdit 대응. 입력은 자유롭고 후보를
내려받아 고를 수 있다:

```ts
{ field: "dept", editorOptions: ["개발팀", "디자인팀", "기획팀"] }
// cellEditor 생략(text) + editorOptions → 자동완성 input
```

## 행별 종속 옵션 (`editorOptions` 함수)

`editorOptions`에 함수를 넘기면 **행 데이터로 옵션을 만든다** — 다른
컬럼 값에 따라 선택지가 달라지는 종속 콤보에 사용한다 (타사 그리드
종속 Enum/체인 콤보 대응):

```ts
// 활성 행은 전체 역할 선택 가능, 비활성 행은 viewer/operator만
{
  field: "role",
  cellEditor: "select",
  editorOptions: (row) =>
    row.active ? ROLES : ["viewer", "operator"],
}
```

- `select`·`multiselect`·`radio`·text(datalist) 에디터 모두 지원한다.
- 함수는 편집 시작 시 행에 대해 한 번 호출된다 — 다른 셀 값이 바뀌어도
  이미 열린 에디터의 옵션은 갱신되지 않는다.
- 함수가 예외를 던지면 빈 목록으로 폴백한다.
- 프로그래밍으로 해석하려면 `grid.getEditorOptions(column, row)` 사용.

## IME 조합 처리 (한글/일본어/중국어 입력)

모든 렌더러의 키 핸들러는 `isComposing`을 검사한다 — 한글 IME로 입력 중
**조합 확정 Enter가 편집 커밋·셀 이동·찾기 이동으로 오작동하지 않는다**:

- 인라인 에디터(text/number/select/textarea): 조합 중 Enter/Escape/Tab 무시
- 그리드 키 내비게이션: 조합 중 화살표/단축키 무시
- 찾기 바(`findBox`): 조합 확정 Enter가 다음 매치로 이동하지 않음

조합이 끝난 뒤의 키 입력은 정상 동작한다.

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
if (errors.length) return; // 서버 전송 중단
```

## 편집 진입 차단 — `beforeEdit`

행 상태·값에 따라 조건부로 편집을 막으려면 `beforeEdit` 훅을 사용한다
(타사 그리드 `OnBeforeEdit` 대응). `false`를 반환하면 해당 셀의 편집 진입이
취소된다 — `checkbox` 에디터의 클릭 토글에도 적용된다:

```tsx
<DataGrid
  columns={cols}
  data={rows}
  beforeEdit={({ row, column, rowStatus }) =>
    // 삭제 예정 행은 편집 불가, admin은 role 컬럼만 잠금
    rowStatus === "D" || (row.role === "admin" && column.field === "role")
      ? false
      : undefined
  }
/>
```

- ctx: `{ row, rowIndex, column, columnIndex, rowStatus }` — `rowIndex`/
  `columnIndex`는 표시(visibleData/visibleColumns) 기준이다.
- 런타임 교체는 `grid.setBeforeEdit(fn)` — prop 동기화 경로도 이를 사용한다.
- 프로그래밍적 쓰기(`setCellValue`, `pasteTsv`, 채우기)에는 적용되지 않는다 —
  편집 UI 진입만 차단한다. 데이터 무결성은 `validate`/`validateChanges`로.

## 커스텀 에디터

`cellEditor: 'custom'`이면 `getEditorContext(r, c)`가
`CellEditorContext`(`{ row, column, value, editValue, error, setValue,
commit, cancel }`)를 제공한다. 어댑터별 연결:

| 프레임워크 | 확장점                             |
| ---------- | ---------------------------------- |
| React      | `ReactColumnDef.renderEditor(ctx)` |
| Vue / Vue2 | `#editor-{field}` 슬롯             |
| Svelte     | `{#snippet editor}`                |
| vanilla    | `ColumnDef.editorRenderer(ctx)`    |

- React `renderEditor`와 vanilla `editorRenderer`는 지정되면 cellEditor
  종류와 무관하게 우선 적용된다. Vue 슬롯은 해당 field 컬럼만 덮어쓰고,
  Svelte `editor` 스니펫은 `cellEditor: 'custom'` 컬럼에만 적용된다 —
  다른 컬럼은 내장 에디터로 폴백한다.
- 반환/렌더된 노드에서 `ctx.setValue`로 임시 값을 갱신하고
  `ctx.commit()`/`ctx.cancel()`로 저장·취소한다.

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
