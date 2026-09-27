# 테마 / 커스텀 스타일 (Theming)

모든 색상은 `--grid-*` CSS 변수 기반. 변수 재정의나 클래스 주입으로
그리드 전체 스타일을 제어한다.

## 다크 모드

### `theme` prop — 그리드 단위 테마 (권장)

각 렌더러의 `theme` prop/`mountGrid` 옵션으로 그리드 자체에 테마를 적용한다 —
상위 요소 클래스 없이 그리드 하나만 다크로 둘 수 있다:

```tsx
// 어댑터 (React/Vue3/Vue2/Svelte)
<DataGrid columns={cols} data={rows} theme="dark" />

// mountGrid
mountGrid(el, { columns, data, theme: "dark" });
```

- `"dark"` → 루트에 `.grid-theme-dark`, `"light"` → `.grid-theme-light` 적용
  (다크 페이지 안의 라이트 그리드도 가능).
- 생략하면 클래스를 추가하지 않는다 — 상위 요소/`:root` 팔레트를 상속.

### 클래스 직접 적용

그리드 또는 상위 요소에 `.grid-theme-dark` 클래스를 적용한다 (CSS 변수는
상속되므로 페이지 어디에 걸어도 된다):

```html
<body class="grid-theme-dark">   <!-- 페이지 전체 다크 -->
  <div class="grid-theme-dark">  <!-- 이 그리드만 다크 -->
```

클래스를 지정하지 않으면 `:root`의 라이트 팔레트가 기본 적용된다.

## 행 줄무늬 — `striped`

zebra 줄무늬(짝수 표시 행 배경)를 켠다 — 타사 그리드 `Alternate` 스타일 대응:

```tsx
<DataGrid columns={cols} data={rows} striped />   // 어댑터 prop
mountGrid(el, { columns, data, striped: true }); // mountGrid 옵션
grid.setStriped(true);                            // 런타임 토글
```

- `tbody tr:nth-child(even)`에 `--grid-stripe-bg`를 적용한다 — 정렬/필터/
  그룹으로 표시 순서가 바뀌어도 화면 기준 짝수 행에 일관 적용.
- hover·선택·I/U/D 상태·고정 행·find 하이라이트는 줄무늬보다 우선한다.
- 고정(pinned) 셀에는 **불투명** `--grid-stripe-pinned-bg`를 사용한다 —
  반투명을 쓰면 sticky 아래로 스크롤 콘텐츠가 비치는 잔상이 생긴다.
- 색상은 `--grid-stripe-bg` / `--grid-stripe-pinned-bg` 변수로 재정의 가능
  (다크 테마 값 내장).

### 네이티브 컨트롤 — `color-scheme`

테마 클래스는 `color-scheme: light|dark`를 함께 지정한다 — 그리드 내부의
네이티브 스크롤바·checkbox·select 화살표 등도 테마에 맞춰 렌더링된다.

## CSS 변수

```css
:root, .grid-theme-light {
  --grid-bg-color: #ffffff;
  --grid-border-color: #e2e2e2;
  --grid-header-bg: #f7f7f8;
  --grid-text-color: #222222;
  --grid-row-hover-bg: #f4f6ff;
  --grid-row-selected-bg: #e5ecff;
  --grid-primary-color: #2563eb;
  /* 행 상태 색도 변수: --grid-status-i/u/d-color, --grid-status-i/u/d-bg */
  /* 입력/패널/스켈레톤/고정 경계 등 파생 변수 — styles.css 참고 */
}
```

커스텀 테마는 같은 이름의 변수만 덮어쓴다:

```css
.my-theme {
  --grid-primary-color: #9333ea;
  --grid-row-selected-bg: #f3e8ff;
}
```

## 조건부 스타일 — cellClass / rowClass

```ts
const columns = [
  {
    field: "age",
    // 셀 파라미터 { value, row, rowIndex, column } → 조건부 클래스
    cellClass: ({ value }) => (Number(value) >= 40 ? "cell-senior" : ""),
  },
  {
    field: "role",
    // 행 파라미터 { row, rowIndex } — 이 컬럼이 속한 모든 행에 적용
    rowClass: ({ row }) => (row.role === "admin" ? "row-admin" : ""),
  },
];

// 모든 행에 공통 적용 — GridOptions.rowClass
createGrid({
  columns, data,
  rowClass: ({ rowIndex }) => (rowIndex % 2 ? "row-odd" : ""),
});
```

```css
.cell-senior { font-weight: 600; color: #b45309; }
.row-admin   { background: #fff7ed; }
.row-odd     { background: #fafafa; }
```

- `rowIndex`는 `visibleData` 기준 — 필터/정렬 후 인덱스가 함수에 전달된다.
- `GridOptions.rowClass`와 모든 컬럼의 `rowClass`는 **합산**되어 `<tr>`에 적용.
- 함수가 빈 문자열/`null`/`undefined`를 반환하면 클래스를 추가하지 않는다.
- 클래스 값은 문자열 상수도 가능: `cellClass: "text-right"`.
- 커스텀 렌더러(비-어댑터)는 `grid.getRowClass(row, i)` /
  `grid.getCellClass(row, i, col)`로 해석 결과를 얻는다.

## 인라인 스타일 — cellStyle

클래스 없이 셀 하나만 직접 색을 바꿀 때 (타사 그리드 조건부 색상 대응).
CSS 문자열, kebab-case 속성 맵, 또는 셀 파라미터를 받는 함수를 지정한다:

```ts
{
  field: "age",
  cellStyle: ({ value }) =>
    Number(value) >= 40 ? { "background-color": "#fee2e2", color: "#b91c1c" } : null,
}
// 또는 CSS 문자열: cellStyle: "background-color: #fee2e2; color: #b91c1c"
```

- 반환값은 `Record<string, string>`(kebab-case) 또는 CSS 문자열 —
  코어가 문자열을 맵으로 파싱해 어댑터가 `<td style>`에 병합한다.
- 커스텀 렌더러는 `grid.getCellStyle(row, rowIndex, col)`로 맵을 얻는다.

## 인라인 스타일 — rowStyle

`rowClass`의 인라인 스타일 버전 — 행(`<tr>`) 전체에 스타일을 직접 적용한다.
CSS 문자열/kebab-case 맵/행 파라미터 함수 모두 지원:

```ts
{
  field: "role",
  rowStyle: ({ row }) =>
    row.role === "admin" ? { "background-color": "rgba(99,102,241,.15)" } : null,
}

// 모든 행 공통 — GridOptions.rowStyle
createGrid({
  columns, data,
  rowStyle: "border-top: 2px solid #e5e7eb",
});
```

- `GridOptions.rowStyle`과 모든 컬럼의 `rowStyle`이 **병합**되어 `<tr>`에 적용 —
  같은 속성이면 나중(컬럼) 쪽이 우선한다.
- 커스텀 렌더러는 `grid.getRowStyle(row, rowIndex)`로 맵을 얻는다.
- `<tr>`에 적용되므로 `background-color`는 셀 배경이 투명할 때만 보인다.

## 주요 구조 클래스 (커스텀 스타일링 대상)

| 클래스 | 대상 |
| ------ | ---- |
| `.mg-table` | 그리드 테이블 |
| `.mg-group-row` | 그룹 헤더 행 |
| `.mg-cell-active` / `.mg-cell-selected` | 활성 셀 / 선택 범위 |
| `.mg-cell-editing` / `.mg-edit-error` | 편집 중 셀 / 에러 |
| `.mg-pinned` / `.mg-pin-left-edge` / `.mg-pin-right-edge` | 고정 컬럼 |
| `.mg-pinned-top-row` / `.mg-pinned-bottom-row` | 고정 행 |
| `.mg-total-row` | 총계 `<tfoot>` 행 |
| `.mg-skeleton-row` / `.mg-loading` | 서버 모드 스켈레톤/로딩 |
| `.mg-loading-host` / `.mg-loading-overlay` | 로딩 오버레이 활성 루트 / 오버레이 (`loading` 옵션·prop) |
| `.mg-statusbar` / `.mg-colctl-panel` | 상태바 / 컬럼 관리 팝오버 |
| `.mg-row-drag-handle` / `.mg-drop-before` / `.mg-drop-after` | 행 드래그 |
| `.mg-empty` | 빈 그리드 행 |
| `.mg-striped` | 줄무늬 활성 루트 (`striped` 옵션) |
