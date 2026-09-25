# 테마 / 커스텀 스타일 (Theming)

모든 색상은 `--grid-*` CSS 변수 기반. 변수 재정의나 클래스 주입으로
그리드 전체 스타일을 제어한다.

## 다크 모드

그리드 또는 상위 요소에 `.grid-theme-dark` 클래스를 적용한다 (CSS 변수는
상속되므로 페이지 어디에 걸어도 된다):

```html
<body class="grid-theme-dark">   <!-- 페이지 전체 다크 -->
  <div class="grid-theme-dark">  <!-- 이 그리드만 다크 -->
```

클래스를 지정하지 않으면 `:root`의 라이트 팔레트가 기본 적용된다.

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
| `.mg-statusbar` / `.mg-colctl-panel` | 상태바 / 컬럼 관리 팝오버 |
| `.mg-row-drag-handle` / `.mg-drop-before` / `.mg-drop-after` | 행 드래그 |
| `.mg-empty` | 빈 그리드 행 |
